# Power BI — Retention Campaign Dashboard: Data Model & Design Spec (v2)

> **Brand**: Health Brief (HB)
> **ESP**: Ongage
> **Flow**: L0 Lead Nurturing → L1 Subscriber Upconversion
> **Data Period**: 2026-02-06 → ongoing
> **Last Updated**: 2026-03-13

---

## 1. Source Data Overview

### 1.1 Table: `RetentionUsers` (retention_users_005.csv)

| Column | Type | Description |
|---|---|---|
| `email` | Text (PK) | Contact email address |
| `utm_campaign` | Text | Acquisition campaign (mostly blank) |
| `utm_source` | Text | Originating website/source (e.g., eatingwell.com, verywellhealth.com) |
| `utm_medium` | Text | Always `retention_s3` |
| `sub_level` | Text | Current subscription tier: L0 (lead) or L1 (subscriber) |
| `o_event` | Text | Always `retention` |
| `sub_date` | DateTime | Date the contact was added/subscribed |
| `ocx_created_date` | DateTime | Date the contact record was created in Ongage |
| `ongage_status` | Text | Current status: Active, Unsubscribed, or Complained |

**Key Stats** (as of 2026-03-13):
- 2,960 total contacts
- 2,959 L0 / 1 L1
- 2,805 Active / 135 Unsubscribed / 20 Complained
- ~490–496 leads per weekly cohort (6 cohorts: Feb 6, 13, 19, 26, Mar 5, 12)
- Top sources: eatingwell.com (1,059), verywellhealth.com (685), everydayhealth.com (211)

### 1.2 Table: `ContactActivity` (Retention_Contact_Activity.csv)

| Column | Type | Description |
|---|---|---|
| `email` | Text (FK → RetentionUsers) | Contact email |
| `type` | Text | Activity type: `send`, `open`, `click`, `soft_bounce` |
| `mailing_id` | Integer | Unique mailing identifier |
| `mailing_name` | Text | Mailing name (e.g., HB_PF_20260306) |
| `email_message_id` | Integer | Message template ID |
| `mailing_type` | Text | Always `campaign` |
| `mailing_schedule` | DateTime | Scheduled send time |
| `timestamp` | DateTime | Actual event timestamp |
| `action_timestamp_rounded` | DateTime | Event timestamp rounded to the hour |
| `esp_id` | Integer | ESP identifier |
| `connection_id` | Integer | Connection/sending domain ID |
| `link_id` | BigInt | Clicked link identifier |
| `data` | Text | Link URL (for clicks) |
| `days_passed` | Decimal | Days between schedule and event |
| `ip` | Text | Event IP address |
| `country_code` | Text | Geo location |
| `browser` | Text | Browser/client (e.g., Apple Proxy) |
| `os` | Text | Operating system |
| `ocx_bounce_reason` | Text | Bounce error message |
| `bot` | Text | Bot flag: Yes/No (critical for click filtering) |

**Key Stats**:
- 14,368 activity rows across 2,312 unique contacts
- 9,411 sends / 4,642 opens / 140 clicks (90 non-bot, 50 bot) / 175 soft bounces
- 52 unique real (non-bot) clickers
- 1,192 unique openers
- 648 contacts in users table with NO activity (newer cohort or suppressed before send)

---

## 2. Data Model (Star Schema)

```
┌──────────────────┐       ┌─────────────────────┐
│  DIM_Contact     │───1:N─│  FACT_Activity       │
│  (RetentionUsers)│       │  (ContactActivity)   │
└────────┬─────────┘       └──────────┬──────────┘
         │                            │
         │                   ┌────────┴────────┐
         │                   │  DIM_Date        │
         │                   │  (Calendar)      │
         │                   └─────────────────┘
         │
    ┌────┴─────────┐
    │  DIM_Source   │
    │  (Derived)   │
    └──────────────┘
```

### 2.1 Relationships

| From Table | From Column | To Table | To Column | Cardinality | Direction |
|---|---|---|---|---|---|
| FACT_Activity | email | DIM_Contact | email | Many-to-One | Both |
| FACT_Activity | action_date (calculated) | DIM_Date | Date | Many-to-One | Single |
| DIM_Contact | utm_source | DIM_Source | source_name | Many-to-One | Single |

### 2.2 Calculated Columns

#### On `DIM_Contact`:

```dax
// Cohort week based on sub_date
CohortWeek =
    FORMAT(RetentionUsers[sub_date], "YYYY-WW")

// Days since subscription
DaysSinceSub =
    DATEDIFF(RetentionUsers[sub_date], TODAY(), DAY)

// Days to first open
DaysToFirstOpen =
    VAR FirstOpen =
        CALCULATE(
            MIN(ContactActivity[timestamp]),
            ContactActivity[type] = "open"
        )
    RETURN IF(ISBLANK(FirstOpen), BLANK(),
        DATEDIFF(RetentionUsers[sub_date], FirstOpen, DAY))

// Days to first click (non-bot)
DaysToFirstClick =
    VAR FirstClick =
        CALCULATE(
            MIN(ContactActivity[timestamp]),
            ContactActivity[type] = "click",
            ContactActivity[bot] <> "Yes"
        )
    RETURN IF(ISBLANK(FirstClick), BLANK(),
        DATEDIFF(RetentionUsers[sub_date], FirstClick, DAY))

// Days to L1 conversion (for contacts that converted)
DaysToL1 =
    IF(RetentionUsers[sub_level] = "L1",
        DATEDIFF(RetentionUsers[sub_date], RetentionUsers[ocx_created_date], DAY),
        BLANK())

// Sends before first open
SendsBeforeFirstOpen =
    VAR FirstOpenTime =
        CALCULATE(
            MIN(ContactActivity[timestamp]),
            ContactActivity[type] = "open"
        )
    RETURN
        IF(ISBLANK(FirstOpenTime),
            CALCULATE(COUNTROWS(ContactActivity), ContactActivity[type] = "send"),
            CALCULATE(
                COUNTROWS(ContactActivity),
                ContactActivity[type] = "send",
                ContactActivity[timestamp] < FirstOpenTime
            )
        )

// Flow stage assignment
FlowStage =
    VAR TotalSends = CALCULATE(COUNTROWS(ContactActivity), ContactActivity[type] = "send")
    VAR HasOpened = CALCULATE(COUNTROWS(ContactActivity), ContactActivity[type] = "open") > 0
    VAR HasClicked = CALCULATE(COUNTROWS(ContactActivity), ContactActivity[type] = "click", ContactActivity[bot] <> "Yes") > 0
    VAR IsUnsubOrComplained = RetentionUsers[ongage_status] IN {"Unsubscribed", "Complained"}
    RETURN
        SWITCH(TRUE(),
            HasClicked, "L0 C30 — Converted",
            RetentionUsers[sub_level] = "L1", "L1 — Subscriber",
            IsUnsubOrComplained, "Suppressed — " & RetentionUsers[ongage_status],
            TotalSends = 0, "L0 S0 — Fresh Lead",
            TotalSends <= 14 && NOT HasOpened, "L0 S1-14 -O — Non-Opener",
            TotalSends <= 14 && HasOpened, "L0 S1-14 +O — Opener (Pre-Gate)",
            HasOpened && TotalSends <= 60, "L0 O30 S<60x — Engaged Lead",
            TotalSends > 14 && NOT HasOpened, "L0 S15-21 Hail Mary",
            "L0 Active"
        )

// Engagement tier
EngagementTier =
    VAR Opens = CALCULATE(COUNTROWS(ContactActivity), ContactActivity[type] = "open")
    VAR Clicks = CALCULATE(COUNTROWS(ContactActivity), ContactActivity[type] = "click", ContactActivity[bot] <> "Yes")
    RETURN
        SWITCH(TRUE(),
            Clicks > 0, "Clicker",
            Opens >= 3, "Multi-Opener",
            Opens > 0, "Single Opener",
            TRUE(), "Non-Engager"
        )
```

#### On `FACT_Activity`:

```dax
// Extract date for calendar relationship
ActionDate = DATE(
    YEAR(ContactActivity[action_timestamp_rounded]),
    MONTH(ContactActivity[action_timestamp_rounded]),
    DAY(ContactActivity[action_timestamp_rounded])
)

// Flag non-bot events
IsRealEngagement =
    IF(ContactActivity[bot] = "Yes", FALSE(), TRUE())

// Hour of day for timing analysis
HourOfDay = HOUR(ContactActivity[timestamp])
```

### 2.3 DIM_Date (Calendar Table)

```dax
DIM_Date =
    ADDCOLUMNS(
        CALENDARAUTO(),
        "Year", YEAR([Date]),
        "Month", FORMAT([Date], "MMMM"),
        "MonthNum", MONTH([Date]),
        "WeekNum", WEEKNUM([Date]),
        "WeekDay", FORMAT([Date], "dddd"),
        "WeekDayNum", WEEKDAY([Date]),
        "DayInFlow", DATEDIFF(DATE(2026, 2, 6), [Date], DAY)
    )
```

### 2.4 DIM_Source (Derived Table)

```dax
DIM_Source =
    SUMMARIZE(
        RetentionUsers,
        RetentionUsers[utm_source],
        "source_name", RetentionUsers[utm_source],
        "contact_count", COUNTROWS(RetentionUsers),
        "source_group",
            SWITCH(TRUE(),
                RetentionUsers[utm_source] = "", "Direct / Unknown",
                CONTAINSSTRING(RetentionUsers[utm_source], "eatingwell"), "EatingWell",
                CONTAINSSTRING(RetentionUsers[utm_source], "verywell"), "VeryWell Network",
                CONTAINSSTRING(RetentionUsers[utm_source], "everydayhealth"), "Everyday Health",
                CONTAINSSTRING(RetentionUsers[utm_source], "health.com"), "Health.com",
                CONTAINSSTRING(RetentionUsers[utm_source], "mindbodygreen"), "MindBodyGreen",
                "Other"
            )
    )
```

---

## 3. DAX Measures

### 3.1 Core Volume Metrics

```dax
// Total contacts
Total Contacts = COUNTROWS(RetentionUsers)

// Active contacts
Active Contacts =
    CALCULATE(COUNTROWS(RetentionUsers), RetentionUsers[ongage_status] = "Active")

// Total sends
Total Sends =
    CALCULATE(COUNTROWS(ContactActivity), ContactActivity[type] = "send")

// Total opens
Total Opens =
    CALCULATE(COUNTROWS(ContactActivity), ContactActivity[type] = "open")

// Unique openers
Unique Openers =
    CALCULATE(
        DISTINCTCOUNT(ContactActivity[email]),
        ContactActivity[type] = "open"
    )

// Total clicks (non-bot only)
Total Real Clicks =
    CALCULATE(
        COUNTROWS(ContactActivity),
        ContactActivity[type] = "click",
        ContactActivity[bot] <> "Yes"
    )

// Unique real clickers
Unique Real Clickers =
    CALCULATE(
        DISTINCTCOUNT(ContactActivity[email]),
        ContactActivity[type] = "click",
        ContactActivity[bot] <> "Yes"
    )

// Total bounces
Total Bounces =
    CALCULATE(COUNTROWS(ContactActivity), ContactActivity[type] = "soft_bounce")

// Unique bounced contacts
Unique Bounced Contacts =
    CALCULATE(
        DISTINCTCOUNT(ContactActivity[email]),
        ContactActivity[type] = "soft_bounce"
    )
```

### 3.2 Rate Metrics

```dax
// Open rate (unique)
Open Rate =
    DIVIDE([Unique Openers],
        CALCULATE(DISTINCTCOUNT(ContactActivity[email]), ContactActivity[type] = "send"),
        0)

// Click rate (unique, non-bot)
Click Rate =
    DIVIDE([Unique Real Clickers],
        CALCULATE(DISTINCTCOUNT(ContactActivity[email]), ContactActivity[type] = "send"),
        0)

// Click-to-Open rate
CTOR = DIVIDE([Unique Real Clickers], [Unique Openers], 0)

// Bounce rate
Bounce Rate = DIVIDE([Total Bounces], [Total Sends], 0)

// Complaint rate
Complaint Rate =
    DIVIDE(
        CALCULATE(COUNTROWS(RetentionUsers), RetentionUsers[ongage_status] = "Complained"),
        [Total Contacts],
        0)

// Unsubscribe rate
Unsubscribe Rate =
    DIVIDE(
        CALCULATE(COUNTROWS(RetentionUsers), RetentionUsers[ongage_status] = "Unsubscribed"),
        [Total Contacts],
        0)
```

### 3.3 Additional Client-Requested Metrics

```dax
// === COMPLAINTS & UNSUBSCRIBES ===

// Complaints count
Complaints Count =
    CALCULATE(COUNTROWS(RetentionUsers), RetentionUsers[ongage_status] = "Complained")

// Unsubscribes count
Unsubscribes Count =
    CALCULATE(COUNTROWS(RetentionUsers), RetentionUsers[ongage_status] = "Unsubscribed")

// Complaints per source
Complaints by Source =
    CALCULATE(
        COUNTROWS(RetentionUsers),
        RetentionUsers[ongage_status] = "Complained"
    )
    // Use with utm_source on axis/rows

// Complaint rate per source
Complaint Rate by Source =
    DIVIDE(
        CALCULATE(COUNTROWS(RetentionUsers), RetentionUsers[ongage_status] = "Complained"),
        COUNTROWS(RetentionUsers),
        0)

// === SENDS BEFORE FIRST OPEN ===

// Average sends before first open
Avg Sends Before Open =
    AVERAGEX(
        FILTER(RetentionUsers, RetentionUsers[SendsBeforeFirstOpen] > 0),
        RetentionUsers[SendsBeforeFirstOpen]
    )

// === CONVERSION TIMING ===

// Average days to L1 conversion
Avg Days to L1 =
    AVERAGEX(
        FILTER(RetentionUsers, RetentionUsers[sub_level] = "L1"),
        RetentionUsers[DaysToL1]
    )

// L1 conversion count
L1 Conversions =
    CALCULATE(COUNTROWS(RetentionUsers), RetentionUsers[sub_level] = "L1")

// L1 conversion rate
L1 Conversion Rate = DIVIDE([L1 Conversions], [Total Contacts], 0)

// === REMOVALS / SUPPRESSION ===

// Removed contacts (unsubscribed + complained)
Removed Contacts =
    CALCULATE(
        COUNTROWS(RetentionUsers),
        RetentionUsers[ongage_status] IN {"Unsubscribed", "Complained"}
    )

// Average days before removal
Avg Days Before Removal =
    AVERAGEX(
        FILTER(RetentionUsers, RetentionUsers[ongage_status] IN {"Unsubscribed", "Complained"}),
        DATEDIFF(RetentionUsers[sub_date], TODAY(), DAY)
    )

// === UPLEVELLING ===

// Contacts who moved to L1 on HB
L1 on HB =
    CALCULATE(COUNTROWS(RetentionUsers), RetentionUsers[sub_level] = "L1")

// Average time to uplevel (days)
Avg Time to Uplevel = [Avg Days to L1]

// === CROSS-POLLINATION ===
// Note: SA subscription data would need to be joined from an additional source table.
// Placeholder measure:
// SA Subscribers = CALCULATE(COUNTROWS(CrossPollination), CrossPollination[brand] = "SA")
// Ad Clickers = CALCULATE(COUNTROWS(CrossPollination), CrossPollination[event] = "ad_click")
```

### 3.4 Flow Stage Measures

```dax
// Contacts at each flow stage
Contacts in Stage =
    COUNTROWS(RetentionUsers)
    // Use with RetentionUsers[FlowStage] on axis

// Stage conversion rate (from previous stage)
Stage Conversion Rate =
    VAR CurrentStageCount = COUNTROWS(RetentionUsers)
    VAR PreviousStage =
        SWITCH(
            SELECTEDVALUE(RetentionUsers[FlowStage]),
            "L0 O30 S<60x — Engaged Lead",
                CALCULATE(COUNTROWS(RetentionUsers),
                    RetentionUsers[FlowStage] IN {"L0 S1-14 +O — Opener (Pre-Gate)", "L0 O30 S<60x — Engaged Lead", "L0 C30 — Converted", "L1 — Subscriber"}),
            "L0 C30 — Converted",
                CALCULATE(COUNTROWS(RetentionUsers),
                    RetentionUsers[FlowStage] IN {"L0 O30 S<60x — Engaged Lead", "L0 C30 — Converted", "L1 — Subscriber"}),
            [Total Contacts]
        )
    RETURN DIVIDE(CurrentStageCount, PreviousStage, 0)

// Day 14 gate pass rate
Day14 Gate Pass Rate =
    DIVIDE(
        CALCULATE(COUNTROWS(RetentionUsers),
            RetentionUsers[FlowStage] IN {
                "L0 S1-14 +O — Opener (Pre-Gate)",
                "L0 O30 S<60x — Engaged Lead",
                "L0 C30 — Converted",
                "L1 — Subscriber"
            }),
        CALCULATE(COUNTROWS(RetentionUsers),
            RetentionUsers[DaysSinceSub] >= 14),
        0)

// Suppression rate
Suppression Rate =
    DIVIDE(
        CALCULATE(COUNTROWS(RetentionUsers),
            LEFT(RetentionUsers[FlowStage], 10) = "Suppressed"),
        [Total Contacts],
        0)
```

### 3.5 Cohort Measures

```dax
// Cohort open rate
Cohort Open Rate =
    DIVIDE(
        CALCULATE(DISTINCTCOUNT(ContactActivity[email]), ContactActivity[type] = "open"),
        CALCULATE(DISTINCTCOUNT(ContactActivity[email]), ContactActivity[type] = "send"),
        0)

// Cohort click rate
Cohort Click Rate =
    DIVIDE(
        CALCULATE(DISTINCTCOUNT(ContactActivity[email]), ContactActivity[type] = "click", ContactActivity[bot] <> "Yes"),
        CALCULATE(DISTINCTCOUNT(ContactActivity[email]), ContactActivity[type] = "send"),
        0)
```

---

## 4. Dashboard Pages & Visual Recommendations

### Page 1: Executive Overview

| Visual | Type | Metrics | Reasoning |
|---|---|---|---|
| Total Contacts | **Card** | `Total Contacts` (2,960) | At-a-glance KPI |
| Active Contacts | **Card** | `Active Contacts` (2,805) | At-a-glance KPI |
| Open Rate | **Card** with conditional color | `Open Rate` | Green >20%, Yellow 10-20%, Red <10% |
| Click Rate | **Card** with conditional color | `Click Rate` | Key conversion indicator |
| L1 Conversions | **Card** | `L1 Conversions` | Primary success metric |
| Complaint Rate | **Card** with conditional color | `Complaint Rate` | Red if >0.1% |
| Engagement Funnel | **Funnel Chart** | Sends → Opens → Clicks → L1 | Shows drop-off at each stage |
| Daily Activity Trend | **Line Chart** (multi-series) | Sends, Opens, Clicks by date | Trend analysis over time |
| Status Breakdown | **Donut Chart** | Contacts by `ongage_status` | Quick status distribution |

### Page 2: Retention Flow Visualization

| Visual | Type | Metrics | Reasoning |
|---|---|---|---|
| Flow Funnel | **Funnel Chart** or **Custom Sankey** (Deneb/HTML visual) | Volume at each stage: Fresh Lead → Daily Sends → Day 14 Gate → Engaged/Hail Mary → Converted/Suppressed | Maps directly to the retention flow diagram; shows volume and drop-off at each decision point |
| Stage Distribution | **Stacked Bar Chart** | Contacts by `FlowStage`, colored by status | Shows current population across all stages |
| Day 14 Gate Results | **100% Stacked Bar** | Opened vs Not Opened at Day 14 | Critical decision point visualization |
| Conversion Waterfall | **Waterfall Chart** | Starting contacts → additions/losses at each stage → final subscribers | Shows net flow through the system |
| Time in Stage | **Box Plot** (Deneb) or **Bar Chart** | Average days spent in each stage | Identifies bottlenecks |
| Stage-to-Stage Transition Matrix | **Matrix Table** with conditional formatting | From-stage (rows) × To-stage (columns) with counts | Detailed transition tracking |

**Sankey Diagram Implementation (Deneb Custom Visual)**:

```json
{
  "description": "Retention Flow Sankey",
  "mark": "rect",
  "encoding": {
    "stages": ["L0 Fresh", "Daily Sends", "Day 14 Gate", "Opened Track", "Non-Opener Track", "Engaged Lead", "Hail Mary", "Converted (L1)", "Suppressed"]
  }
}
```

> **Recommended approach**: Use the **Deneb** custom visual with Vega-Lite to create a Sankey/alluvial diagram. Alternatively, use the **xViz Sankey** or **Power BI Decomposition Tree** as simpler alternatives.

### Page 3: Source Analysis & Complaints

| Visual | Type | Metrics | Reasoning |
|---|---|---|---|
| Contacts by Source | **Horizontal Bar Chart** | Contact count by `utm_source` | Ranked source comparison |
| Complaint Rate by Source | **Clustered Bar Chart** | Complaint count & rate by `utm_source` | Identifies problematic sources |
| Unsubscribe Rate by Source | **Clustered Bar Chart** | Unsubscribe count & rate by source | Source quality assessment |
| Source Performance Matrix | **Matrix Table** | Source × (Contacts, Opens, Clicks, Complaints, Unsubs) | Comprehensive source scorecard |
| Source Quality Scatter | **Scatter Plot** | X: Volume, Y: Open Rate, Size: Complaint Rate | Identifies high-volume problem sources |
| Source Trend | **Line Chart** | New contacts by source over cohort weeks | Acquisition trend |

### Page 4: Engagement Deep-Dive

| Visual | Type | Metrics | Reasoning |
|---|---|---|---|
| Sends Before First Open | **Histogram** (bin chart) | Distribution of `SendsBeforeFirstOpen` | Understand how many touches needed |
| Avg Sends Before Open | **Card** | `Avg Sends Before Open` | Quick reference |
| Open Timing Heatmap | **Matrix** with conditional formatting | Day of week × Hour of day → open count | Optimize send timing |
| Engagement Tier Distribution | **Donut Chart** | Contacts by `EngagementTier` | Segment size overview |
| Campaign Performance | **Table** | Mailing name × sends, opens, clicks, open rate, click rate | Per-campaign breakdown |
| Bot vs Real Click Breakdown | **Stacked Bar** | Bot clicks vs Real clicks per campaign | Data quality check |
| Days to First Open | **Histogram** | Distribution of `DaysToFirstOpen` | Engagement velocity |

### Page 5: Cohort Analysis

| Visual | Type | Metrics | Reasoning |
|---|---|---|---|
| Cohort Retention Grid | **Matrix** with conditional formatting | Cohort week (rows) × Days since sub (columns) → Open rate | Classic cohort retention heatmap |
| Cohort Conversion | **Grouped Bar Chart** | L1 conversion rate by cohort | Trend in conversion quality |
| Cohort Size | **Column Chart** | Contacts per cohort week | Volume consistency check |
| Cohort Engagement Comparison | **Line Chart** (multi-series) | Open rate by cohort over time | Compare cohort performance |

### Page 6: Bounce & Deliverability

| Visual | Type | Metrics | Reasoning |
|---|---|---|---|
| Bounce Rate | **Card** | `Bounce Rate` | KPI card |
| Bounce Reasons | **Horizontal Bar** | Top bounce reasons by count | Identify deliverability issues |
| Bounce by Campaign | **Clustered Bar** | Bounce count by mailing name | Find problematic sends |
| Bounced Contact Impact | **Card** | Unique bounced contacts | Scale of deliverability problem |

---

## 5. Retention Flow Visualization — Detailed Spec

### 5.1 Flow Stages (from Flow_of_retention_Lead.jpeg)

```
Stage 0: L0 S0 — Fresh Lead (~500/day intake)
    │
    ▼
Stage 1: Daily Email Sends (Days 1–14, all L0 leads)
    │
    ├── DAY 14 GATE: "Has the lead opened yet?"
    │
    ├── YES (OPENED) ──▶ Stage 2A: L0 O30 S<60x — Engaged Lead
    │                         │    (Opened in last 30 days, daily sends, up to 60 sends total)
    │                         │
    │                         ├── Final 7-Send Window (notification header warning)
    │                         │
    │                         ├── CLICK at any point ──▶ Stage 3: L0 C30 — Converted
    │                         │                              │
    │                         │                              ▼
    │                         │                         L1 Upconversion → Welcome Email → SUBSCRIBER ✓
    │                         │
    │                         └── 60 sends, no click ──▶ SUPPRESSED (60 sends, no click)
    │
    └── NO (NOT OPENED) ──▶ Stage 2B: L0 S1-14 -O — Non-Opener
                                │
                                ▼
                           Stage 2C: L0 S15-21 -S7 -O — Hail Mary
                                │    (7 sends at reduced frequency, ~weekly, ~7 weeks)
                                │
                                └── No engagement ──▶ SUPPRESSED (Final)
```

### 5.2 Recommended Visuals for Flow Representation

#### Option A: Sankey Diagram (Deneb + Vega-Lite) — **Recommended**

Best for showing volume flow between stages with proportional width:

```
[Fresh Leads: 2960] ─────────▶ [Daily Sends: 2312] ─────▶ [Day 14 Gate]
                                                              │
                                              ┌───────────────┤
                                              ▼               ▼
                                     [Engaged: 1192]   [Non-Opener: 1120]
                                         │                    │
                                    ┌────┤               ┌────┤
                                    ▼    ▼               ▼    ▼
                              [Converted] [Suppressed] [Hail Mary] [Suppressed]
                                  [52]       [~1140]              [~1120]
```

**Data Preparation for Sankey**:

Create a `FlowTransitions` table in Power Query:

```
| From | To | Count |
|---|---|---|
| Fresh Lead | Daily Sends | 2312 |
| Fresh Lead | No Activity | 648 |
| Daily Sends | Opened (Day 14) | 1192 |
| Daily Sends | Not Opened (Day 14) | 1120 |
| Opened | Engaged Lead | ~1140 |
| Opened | Converted (Click) | 52 |
| Engaged Lead | Suppressed (60 sends) | TBD |
| Engaged Lead | Still Active | TBD |
| Not Opened | Hail Mary | 1120 |
| Hail Mary | Suppressed (Final) | TBD |
```

#### Option B: Funnel Chart (Built-in)

Simpler, shows progressive drop-off:

```
All Leads ▎████████████████████████████████████▎ 2,960
Sent To   ▎██████████████████████████████▎       2,312
Opened    ▎████████████████▎                     1,192
Clicked   ▎█▎                                      52
L1        ▎▏                                         1
```

#### Option C: Custom HTML Visual

Embed an interactive D3.js/HTML visual directly in Power BI using the HTML Content visual. This can replicate the exact layout from Flow_of_retention_Lead.jpeg with live data.

### 5.3 KPIs to Overlay on Flow Visual

| Stage Transition | KPI | Current Value |
|---|---|---|
| Fresh → Sent | Activation Rate | 78.1% (2312/2960) |
| Sent → Opened | Day 14 Open Rate | 51.6% (1192/2312) |
| Opened → Clicked | Click-to-Open Rate | 4.4% (52/1192) |
| Clicked → L1 | Conversion Rate | 1.9% (1/52) |
| Overall | End-to-End Conversion | 0.03% (1/2960) |
| Non-Opener → Hail Mary | Hail Mary Pool | 48.4% (1120/2312) |
| Any → Unsub/Complained | Attrition Rate | 5.2% (155/2960) |

---

## 6. Filters & Slicers

| Slicer | Field | Type |
|---|---|---|
| Date Range | DIM_Date[Date] | Date range slider |
| Cohort Week | RetentionUsers[CohortWeek] | Dropdown multi-select |
| Source | RetentionUsers[utm_source] | Dropdown multi-select |
| Status | RetentionUsers[ongage_status] | Buttons |
| Sub Level | RetentionUsers[sub_level] | Buttons |
| Flow Stage | RetentionUsers[FlowStage] | Dropdown multi-select |
| Engagement Tier | RetentionUsers[EngagementTier] | Buttons |
| Campaign | ContactActivity[mailing_name] | Dropdown |
| Bot Filter | ContactActivity[bot] | Toggle (default: exclude bots) |

---

## 7. Power Query Transformations

### 7.1 RetentionUsers

```powerquery
let
    Source = Csv.Document(File.Contents("retention_users_005.csv"), [Delimiter=",", Encoding=65001]),
    PromotedHeaders = Table.PromoteHeaders(Source),
    TypedColumns = Table.TransformColumnTypes(PromotedHeaders, {
        {"email", type text},
        {"utm_campaign", type text},
        {"utm_source", type text},
        {"utm_medium", type text},
        {"sub_level", type text},
        {"o_event", type text},
        {"sub_date", type datetime},
        {"ocx_created_date", type datetime},
        {"ongage_status", type text}
    }),
    CleanedSource = Table.ReplaceValue(TypedColumns, "", "Direct / Unknown", Replacer.ReplaceValue, {"utm_source"}),
    AddedCohort = Table.AddColumn(CleanedSource, "CohortDate", each Date.From([sub_date]), type date)
in
    AddedCohort
```

### 7.2 ContactActivity

```powerquery
let
    Source = Csv.Document(File.Contents("Retention_Contact_Activity.csv"), [Delimiter=",", Encoding=65001]),
    PromotedHeaders = Table.PromoteHeaders(Source),
    TypedColumns = Table.TransformColumnTypes(PromotedHeaders, {
        {"email", type text},
        {"type", type text},
        {"mailing_id", Int64.Type},
        {"mailing_name", type text},
        {"timestamp", type datetime},
        {"action_timestamp_rounded", type datetime},
        {"days_passed", type number},
        {"bot", type text}
    }),
    AddedActionDate = Table.AddColumn(TypedColumns, "ActionDate", each Date.From([action_timestamp_rounded]), type date),
    FilteredBotClicks = Table.AddColumn(AddedActionDate, "IsRealEngagement",
        each if [bot] = "Yes" then false else true, type logical)
in
    FilteredBotClicks
```

---

## 8. Notes & Considerations

1. **Bot Filtering**: 36% of clicks are bot-generated. Always filter by `bot <> "Yes"` for click metrics. Consider showing both raw and filtered in a toggle.

2. **Cross-Pollination Data Gap**: The client requested metrics for SA subscriptions and Ad clicks. These require additional data sources not present in the current CSVs. Create placeholder visuals with "Data Pending" labels.

3. **Removal Timing**: The current data doesn't explicitly track removal date. Use `ocx_created_date` vs current status as a proxy, or request an additional export with status change timestamps from Ongage.

4. **Cohort Maturity**: As of March 13, 2026, the oldest cohort (Feb 6) has only ~5 weeks of data, and the newest (Mar 12) has 1 day. Cohort analysis will become more meaningful as data accumulates.

5. **Send Cadence**: Most contacts have exactly 4 sends (the 4 daily HB_PF campaigns from Mar 6–9), indicating the current dataset captures a narrow window. The flow assumes up to 60 sends over ~60 days, so longitudinal tracking will require ongoing data refreshes.

6. **Incremental Refresh**: Set up incremental refresh on `ContactActivity` partitioned by `ActionDate` to handle growing data volume efficiently.
