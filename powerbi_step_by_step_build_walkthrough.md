# Power BI Build Walkthrough for Retention Cohort + Ongage Contact Activity

This guide walks through how to build the Power BI model for the two source files:

- `retention_users.csv`
- `Retention_Contact_Activity.csv`

It is written for **Power BI Desktop** and assumes you want a clean semantic model that supports the retention-cohort analysis we designed earlier.

---

## 1) What you are building

You are not building one flat imported table. You are building a small semantic model with a clear grain:

- **DimContact** = one row per retained email/contact
- **FactContactActivity** = one row per contact activity event from Ongage
- **DimDate** = one row per calendar date
- **DimCampaign** = one row per campaign/mailing
- optional helper tables for event type, source groups, or KPI bands later

That model lets you answer questions like:

- Which acquisition sources created the most active retained users?
- Which campaigns generate the highest open and click depth inside the retained cohort?
- Which cohorts are fading after initial sends?
- Which campaigns are inflating sends without generating clicks?
- Where do soft bounces cluster by source, date, or campaign?

---

## 2) What is in your files

### `retention_users.csv`
Current observed structure:

- `email`
- `utm_source`
- `utm_medium`
- `utm_campaign`
- `o_event`
- `sub_date`
- `ocx_created_date`
- `ongage_status`

Observed characteristics:

- 2,312 rows
- 2,312 unique emails
- zero duplicate emails in the file
- `utm_campaign` is blank for all rows in the current sample

### `Retention_Contact_Activity.csv`
Current observed structure:

- `email`
- `type`
- `mailing_id`
- `mailing_name`
- `email_message_id`
- `mailing_type`
- `mailing_schedule`
- `timestamp`
- `action_timestamp_rounded`
- `esp_id`
- `connection_id`
- `link_id`
- `data`
- `days_passed`
- `ip`
- `country_code`
- `browser`
- `os`
- `ocx_bounce_reason`
- `bot`

Observed characteristics:

- 14,368 activity rows
- 2,312 unique emails
- 14 unique campaigns (`mailing_id`)
- event types currently present:
  - `send`
  - `open`
  - `click`
  - `soft_bounce`

This means your first model should prioritize:

- source quality
- cohort maturity
- engagement conversion
- click-through quality
- soft-bounce risk

It should **not** overemphasize unsubscribe or complaint reporting unless those event types are added later.

---

## 3) Before opening Power BI

Put both CSV files in a stable local folder. Do not move them after building the report unless you will update the source path.

Suggested folder:

```text
C:\BI\Retention_Ongage\source\
```

Recommended files:

```text
C:\BI\Retention_Ongage\source\retention_users.csv
C:\BI\Retention_Ongage\source\Retention_Contact_Activity.csv
```

---

## 4) Open Power BI Desktop and import the two sources

### Step 4.1: Launch Power BI Desktop
Open **Power BI Desktop**.

### Step 4.2: Get the first file
- Home ribbon → **Get data** → **Text/CSV**
- Select `retention_users.csv`
- Click **Transform Data**

Do **not** click Load yet. You want Power Query first.

### Step 4.3: Get the second file
Inside Power Query:
- Home → **New Source** → **Text/CSV**
- Select `Retention_Contact_Activity.csv`
- Click **OK**

Now both tables should appear in the left Queries pane.

Recommended initial query names:

- `retention_users_raw`
- `retention_contact_activity_raw`

Then duplicate/transform them into model-ready queries.

---

## 5) Build the contact dimension in Power Query

Right-click `retention_users_raw` → **Reference**.
Rename the new query to:

```text
DimContact
```

### Step 5.1: Keep only the columns you need
Keep:

- `email`
- `utm_source`
- `utm_medium`
- `utm_campaign`
- `o_event`
- `sub_date`
- `ocx_created_date`
- `ongage_status`

### Step 5.2: Clean the email key
Transform `email`:

- Transform → Format → **Trim**
- Transform → Format → **Clean**
- Transform → Format → **Lowercase**

This is critical because `email` is your join key.

### Step 5.3: Fix blanks and normalize text
Suggested replacements:

- `utm_campaign` blank or null → `'(none)'`
- `utm_medium` blank or null → `'(unknown)'`
- `utm_source` blank or null → `'(unknown)'`
- `o_event` blank or null → `'(unknown)'`
- `ongage_status` blank or null → `'(unknown)'`

### Step 5.4: Convert date columns
Convert these columns to **Date/Time** first, then optionally derive Date:

- `sub_date`
- `ocx_created_date`

If Power BI misreads them as text:
- select the column
- Transform → **Data Type** → Date/Time
- if needed, use **Using Locale**

Then create derived date-only columns:

- `SubDate` from `sub_date`
- `CreatedDate` from `ocx_created_date`

Use Add Column → Date → Date Only.

### Step 5.5: Add contact classification columns
Create the following helper columns in Power Query.

#### SourceBucket
Use `utm_source` to group sites into business-friendly buckets.
Example logic:

- eatingwell.com → EatingWell
- verywellhealth.com → Verywell Health
- verywellmind.com → Verywell Mind
- everydayhealth.com → Everyday Health
- health.com → Health.com
- else → Other Source

#### MediumBucket
Normalize `utm_medium` values into a short set, for example:

- email
- paid
- referral
- unknown

#### ContactAgeDays
Add a custom column:

```powerquery
Duration.Days(Date.From(DateTime.LocalNow()) - [SubDate])
```

#### ContactAgeBand
Band the days into lifecycle buckets:

- 0–7 days
- 8–14 days
- 15–30 days
- 31–60 days
- 61+ days

#### AcquisitionMonth
Add month start from `SubDate`.

### Step 5.6: Remove duplicates as a safety step
Even though the file currently has unique emails:
- select `email`
- Home → **Remove Rows** → **Remove Duplicates**

### Step 5.7: Set final data types
Recommended types:

- `email` = Text
- `utm_source` = Text
- `utm_medium` = Text
- `utm_campaign` = Text
- `o_event` = Text
- `ongage_status` = Text
- `SubDate` = Date
- `CreatedDate` = Date
- `ContactAgeDays` = Whole Number
- `ContactAgeBand` = Text
- `AcquisitionMonth` = Date

---

## 6) Build the contact activity fact table in Power Query

Right-click `retention_contact_activity_raw` → **Reference**.
Rename it:

```text
FactContactActivity
```

### Step 6.1: Keep the needed columns
Keep:

- `email`
- `type`
- `mailing_id`
- `mailing_name`
- `email_message_id`
- `mailing_type`
- `mailing_schedule`
- `timestamp`
- `action_timestamp_rounded`
- `esp_id`
- `connection_id`
- `link_id`
- `days_passed`
- `ip`
- `browser`
- `ocx_bounce_reason`
- `bot`

You can keep `data`, `country_code`, `os` if you plan deeper analysis later, but they are low-value in the current sample.

### Step 6.2: Clean the join key
Apply to `email`:

- Trim
- Clean
- Lowercase

### Step 6.3: Normalize event type
Transform `type` to lowercase.
Then optionally create a friendly label column:

- send → Send
- open → Open
- click → Click
- soft_bounce → Soft Bounce

### Step 6.4: Convert timestamps
Convert:

- `timestamp` → Date/Time
- `action_timestamp_rounded` → Date/Time
- `mailing_schedule` → Date/Time

Then create:

- `ActivityDate` = date only from `timestamp`
- `ActivityMonth` = month start from `ActivityDate`
- `ActivityHour` = hour of `timestamp`

### Step 6.5: Normalize bot flag
Create `BotFlag`:

- if `bot` = true / 1 / yes → 1
- else → 0

If the source uses text values, normalize them to a clean numeric flag.

### Step 6.6: Normalize bounce reason
Create `BounceReasonClean`:

- if `ocx_bounce_reason` is null → `'(none)'`
- else keep text trimmed

### Step 6.7: Clean campaign text
For `mailing_name` and `mailing_type`:

- Trim
- Clean

If campaign names contain trailing spaces, fixing them now prevents accidental duplicate campaign categories.

### Step 6.8: Create row-level event flags
Add custom columns:

- `IsSend` = if `[type] = "send"` then 1 else 0
- `IsOpen` = if `[type] = "open"` then 1 else 0
- `IsClick` = if `[type] = "click"` then 1 else 0
- `IsSoftBounce` = if `[type] = "soft_bounce"` then 1 else 0

These make DAX simpler and faster to write.

### Step 6.9: Set final data types
Recommended:

- `email` = Text
- `type` = Text
- `mailing_id` = Whole Number
- `mailing_name` = Text
- `email_message_id` = Whole Number
- `mailing_type` = Text
- `mailing_schedule` = Date/Time
- `timestamp` = Date/Time
- `action_timestamp_rounded` = Date/Time
- `ActivityDate` = Date
- `ActivityMonth` = Date
- `ActivityHour` = Whole Number
- `esp_id` = Whole Number
- `connection_id` = Whole Number
- `link_id` = Whole Number
- `days_passed` = Decimal Number
- `BotFlag` = Whole Number
- `BounceReasonClean` = Text
- `IsSend` = Whole Number
- `IsOpen` = Whole Number
- `IsClick` = Whole Number
- `IsSoftBounce` = Whole Number

---

## 7) Build the campaign dimension in Power Query

Reference `FactContactActivity` and rename the query:

```text
DimCampaign
```

### Step 7.1: Keep only campaign columns
Keep:

- `mailing_id`
- `mailing_name`
- `mailing_type`
- `mailing_schedule`
- `esp_id`
- `connection_id`

### Step 7.2: Remove duplicates
Use `mailing_id` as the natural key.
- select `mailing_id`
- Remove Duplicates

### Step 7.3: Derive campaign fields
Create:

- `MailingScheduleDate` = date only from `mailing_schedule`
- `CampaignMonth` = month start from `MailingScheduleDate`
- `CampaignDisplayName` = maybe `mailing_name` unless blank then `Campaign #<mailing_id>`

### Step 7.4: Set data types
- `mailing_id` = Whole Number
- `mailing_name` = Text
- `mailing_type` = Text
- `mailing_schedule` = Date/Time
- `MailingScheduleDate` = Date
- `CampaignMonth` = Date
- `esp_id` = Whole Number
- `connection_id` = Whole Number

---

## 8) Create a date table

You can create a date dimension in DAX after loading, which is usually the easiest route here.

For now, in Power Query, just confirm your fact tables have good date columns:

- `DimContact[SubDate]`
- `DimContact[CreatedDate]`
- `FactContactActivity[ActivityDate]`
- `DimCampaign[MailingScheduleDate]`

Then click **Close & Apply**.

---

## 9) Create the date dimension in DAX

After data loads, go to **Modeling** → **New Table** and create:

```DAX
DimDate =
VAR MinDate =
    MINX(
        UNION(
            SELECTCOLUMNS(DimContact, "d", DimContact[SubDate]),
            SELECTCOLUMNS(FactContactActivity, "d", FactContactActivity[ActivityDate]),
            SELECTCOLUMNS(DimCampaign, "d", DimCampaign[MailingScheduleDate])
        ),
        [d]
    )
VAR MaxDate =
    MAXX(
        UNION(
            SELECTCOLUMNS(DimContact, "d", DimContact[SubDate]),
            SELECTCOLUMNS(FactContactActivity, "d", FactContactActivity[ActivityDate]),
            SELECTCOLUMNS(DimCampaign, "d", DimCampaign[MailingScheduleDate])
        ),
        [d]
    )
RETURN
ADDCOLUMNS(
    CALENDAR(MinDate, MaxDate),
    "Year", YEAR([Date]),
    "Month Number", MONTH([Date]),
    "Month Name", FORMAT([Date], "MMMM"),
    "Month Short", FORMAT([Date], "MMM"),
    "YearMonth", FORMAT([Date], "YYYY-MM"),
    "Quarter", "Q" & FORMAT([Date], "Q"),
    "Weekday Number", WEEKDAY([Date], 2),
    "Weekday", FORMAT([Date], "dddd"),
    "Month Start", DATE(YEAR([Date]), MONTH([Date]), 1)
)
```

Then in Table tools:

- **Mark as date table**
- choose `DimDate[Date]`

Create a sort rule:

- Sort `Month Name` by `Month Number`
- Sort `Month Short` by `Month Number`

---

## 10) Create the relationships in Model view

Open **Model view**.

You want these relationships:

### Core relationships

1. `DimContact[email]` → `FactContactActivity[email]`
   - One-to-many
   - single direction from DimContact to FactContactActivity
   - active

2. `DimCampaign[mailing_id]` → `FactContactActivity[mailing_id]`
   - One-to-many
   - single direction from DimCampaign to FactContactActivity
   - active

3. `DimDate[Date]` → `FactContactActivity[ActivityDate]`
   - One-to-many
   - single direction
   - active

### Optional inactive relationships

4. `DimDate[Date]` → `DimContact[SubDate]`
   - One-to-many
   - inactive

5. `DimDate[Date]` → `DimCampaign[MailingScheduleDate]`
   - One-to-many
   - inactive

Why inactive for those last two?
Because your primary reporting time axis should usually be **ActivityDate**. You can activate the other date paths inside measures with `USERELATIONSHIP()` when needed.

### Relationship settings to avoid

Do not use:

- many-to-many here
- bidirectional filtering unless you have a very specific need
- automatic snowflaking before phase 2

This is a classic small star schema and should stay simple.

---

## 11) Hide technical columns before reporting

In Model view, hide columns that users should not drag into visuals directly.

### Hide in `FactContactActivity`
Usually hide:

- `email_message_id`
- `esp_id`
- `connection_id`
- `ip`
- `data`
- raw `bot`
- raw `ocx_bounce_reason`
- raw timestamp columns you replaced with cleaned date fields
- row-level event flags if you only want measure use

### Hide in `DimContact`
Usually hide:

- raw datetime columns if you created cleaner date-only fields

### Keep visible
Keep visible the business fields:

- source
- status
- cohort fields
- mailing name
- mailing type
- date fields users will actually slice by

---

## 12) Create the base DAX measures

Go to Modeling → **New measure**.

### Volume measures

```DAX
Contacts = DISTINCTCOUNT(DimContact[email])
```

```DAX
Activity Rows = COUNTROWS(FactContactActivity)
```

```DAX
Campaigns = DISTINCTCOUNT(DimCampaign[mailing_id])
```

### Event totals

```DAX
Total Sends = SUM(FactContactActivity[IsSend])
```

```DAX
Total Opens = SUM(FactContactActivity[IsOpen])
```

```DAX
Total Clicks = SUM(FactContactActivity[IsClick])
```

```DAX
Total Soft Bounces = SUM(FactContactActivity[IsSoftBounce])
```

### Unique engaged contacts

```DAX
Unique Sent Contacts =
CALCULATE(
    DISTINCTCOUNT(FactContactActivity[email]),
    FactContactActivity[type] = "send"
)
```

```DAX
Unique Openers =
CALCULATE(
    DISTINCTCOUNT(FactContactActivity[email]),
    FactContactActivity[type] = "open"
)
```

```DAX
Unique Clickers =
CALCULATE(
    DISTINCTCOUNT(FactContactActivity[email]),
    FactContactActivity[type] = "click"
)
```

```DAX
Unique Soft Bounce Contacts =
CALCULATE(
    DISTINCTCOUNT(FactContactActivity[email]),
    FactContactActivity[type] = "soft_bounce"
)
```

### Rate measures

```DAX
Open Rate = DIVIDE([Total Opens], [Total Sends])
```

```DAX
CTR = DIVIDE([Total Clicks], [Total Sends])
```

```DAX
CTOR = DIVIDE([Total Clicks], [Total Opens])
```

```DAX
Soft Bounce Rate = DIVIDE([Total Soft Bounces], [Total Sends])
```

### Unique rate measures

```DAX
Unique Open Rate = DIVIDE([Unique Openers], [Unique Sent Contacts])
```

```DAX
Unique Click Rate = DIVIDE([Unique Clickers], [Unique Sent Contacts])
```

```DAX
Unique CTOR = DIVIDE([Unique Clickers], [Unique Openers])
```

### Engagement depth measures

```DAX
Avg Opens per Opener = DIVIDE([Total Opens], [Unique Openers])
```

```DAX
Avg Clicks per Clicker = DIVIDE([Total Clicks], [Unique Clickers])
```

### Efficiency measures

```DAX
Clicks per 1000 Sends = [CTR] * 1000
```

```DAX
Opens per 1000 Sends = [Open Rate] * 1000
```

### Non-clicking opener measure

```DAX
Opened Not Clicked = [Unique Openers] - [Unique Clickers]
```

```DAX
Opened Not Clicked Rate = DIVIDE([Opened Not Clicked], [Unique Openers])
```

---

## 13) Create cohort-aware measures

These help you analyze the retained users file against campaign performance.

### Contacts by sub date

```DAX
Contacts by SubDate =
CALCULATE(
    [Contacts],
    USERELATIONSHIP(DimDate[Date], DimContact[SubDate])
)
```

### Contacts by created date

```DAX
Contacts by CreatedDate =
CALCULATE(
    [Contacts],
    USERELATIONSHIP(DimDate[Date], DimContact[CreatedDate])
)
```

### Campaigns by schedule date

```DAX
Campaigns by ScheduleDate =
CALCULATE(
    [Campaigns],
    USERELATIONSHIP(DimDate[Date], DimCampaign[MailingScheduleDate])
)
```

### Cohort openers

```DAX
Cohort Openers = [Unique Openers]
```

This measure itself is simple, but it becomes powerful when sliced by:

- `DimContact[AcquisitionMonth]`
- `DimContact[SourceBucket]`
- `DimContact[ContactAgeBand]`

### Clickers per cohort contact

```DAX
Clickers per Contact = DIVIDE([Unique Clickers], [Contacts])
```

### Sends per contact

```DAX
Sends per Contact = DIVIDE([Total Sends], [Contacts])
```

### Opens per contact

```DAX
Opens per Contact = DIVIDE([Total Opens], [Contacts])
```

---

## 14) Build the first report page to validate the model

Before creating the polished dashboard, make a simple QA page called:

```text
00_Model_QA
```

Add these visuals:

### Card row
- Contacts
- Activity Rows
- Campaigns
- Total Sends
- Total Opens
- Total Clicks
- Total Soft Bounces

### Table 1: Source summary
Rows:
- `DimContact[utm_source]`

Values:
- Contacts
- Unique Sent Contacts
- Unique Openers
- Unique Clickers
- Unique Open Rate
- Unique Click Rate
- Unique CTOR
- Soft Bounce Rate

### Table 2: Campaign summary
Rows:
- `DimCampaign[mailing_name]`

Values:
- Total Sends
- Total Opens
- Total Clicks
- Open Rate
- CTR
- CTOR
- Soft Bounce Rate

### Table 3: Date QA
Rows:
- `DimDate[Date]`

Values:
- Activity Rows
- Total Sends
- Total Opens
- Total Clicks

This page is just to verify that the model works and the numbers look plausible.

---

## 15) Recommended report build order after the QA page

Build in this order:

1. `00_Model_QA`
2. `01_Executive Overview`
3. `02_Source Quality`
4. `03_Cohort Retention and Maturity`
5. `04_Campaign Performance`
6. `05_Deliverability Risk`
7. `06_Contact Drillthrough`

That order reduces debugging pain because you validate fundamentals before styling.

---

## 16) What to put on the Executive Overview page first

Once the model is stable, create these visuals first:

### KPI cards
- Contacts
- Campaigns
- Total Sends
- Unique Open Rate
- Unique Click Rate
- Unique CTOR
- Soft Bounce Rate

### Trend chart
Line chart:
- X-axis = `DimDate[Date]`
- Y-axis = Total Sends, Total Opens, Total Clicks

### Source comparison
Clustered bar chart:
- Y-axis = `DimContact[SourceBucket]`
- X-axis = Unique Click Rate
- Tooltips = Contacts, Unique Open Rate, Unique CTOR, Soft Bounce Rate

### Campaign leaderboard
Table or matrix:
- Campaign name
- Sends
- Opens
- Clicks
- Open rate
- CTR
- CTOR
- Soft bounce rate

---

## 17) Important slicers to add globally

Place these on the left or top panel:

- `DimDate[Date]`
- `DimContact[SourceBucket]`
- `DimContact[ContactAgeBand]`
- `DimCampaign[mailing_name]`
- `DimCampaign[mailing_type]`
- `FactContactActivity[type]` only on QA pages if needed

Do not overload the page with too many slicers at first.

---

## 18) Format the measures correctly

Use these formatting rules:

- Counts = Whole number with thousands separator
- Rates = Percentage with 1 or 2 decimals
- Per-contact metrics = Decimal with 2 decimals
- Per-1000 metrics = Whole number or 1 decimal

Examples:

- Open Rate → Percentage, 1 decimal
- CTR → Percentage, 2 decimals
- Unique CTOR → Percentage, 1 decimal
- Sends per Contact → Decimal, 2 decimals

---

## 19) Create a measure table so the model stays clean

Create a blank DAX table:

```DAX
_Measures = { BLANK() }
```

Move all measures into this table.
Then hide the single placeholder column.

This keeps the field list much cleaner once the report grows.

---

## 20) Validation checklist before styling

Before doing layout and colors, validate all of this:

### Contact table checks
- Contacts = 2,312
- No duplicate emails in `DimContact`
- Source counts match the CSV

### Activity table checks
- Activity rows = 14,368
- Event totals match CSV
- 14 campaigns appear
- send/open/click/soft_bounce counts match imported data

### Relationship checks
- Slicing by source changes campaign metrics
- Slicing by campaign changes event totals
- Slicing by date affects activity metrics correctly
- Using a cohort measure with `USERELATIONSHIP` affects contact-by-sub-date charts correctly

### Sanity checks
- Clicks should never exceed opens in the same filtered slice in this dataset
- Soft bounce rate should stay small relative to sends
- Blank campaign names should be handled

---

## 21) Common issues and how to fix them

### Problem: relationships create ambiguous filtering
Fix:
- keep only one active date relationship to the fact
- use inactive relationships for other dates
- avoid bi-directional relationships

### Problem: email keys do not match
Fix:
- lowercase + trim + clean in both queries
- verify there are no leading/trailing spaces

### Problem: date slicers affect the wrong metrics
Fix:
- remember the active relationship is likely `DimDate` to `FactContactActivity[ActivityDate]`
- use `USERELATIONSHIP()` for sub-date and schedule-date measures

### Problem: rates look inflated
Fix:
- separate total-event rates from unique-contact rates
- avoid mixing `COUNTROWS` with `DISTINCTCOUNT(email)` in the same business definition without intent

### Problem: campaign duplicates appear
Fix:
- trim campaign names
- deduplicate `DimCampaign` on `mailing_id`, not campaign name

---

## 22) What to build after the base model works

After the core model is stable, add:

### A contact drillthrough page
Fields:
- email
- source
- sub date
- campaign history
- event history
- total opens
- total clicks
- soft bounce history

### A source benchmarking page
Compare source quality using:
- contacts
- unique opener rate
- unique click rate
- clicks per contact
- soft bounce rate

### A campaign fatigue page
Focus on:
- sends per contact
- opens per contact
- clicks per contact
- opened-not-clicked share

### A deliverability page
Focus on:
- soft bounce rate by campaign
- soft bounce rate by source
- bounce reasons
- activity by browser or network fields if later useful

---

## 23) Publish only after the Desktop model is stable

Once the PBIX is validated:

- save the file locally
- publish to the Power BI Service
- configure refresh if the CSV paths are stable or move to a managed source later

For early-stage work, it is often easier to keep the first version in Desktop until all logic is verified.

---

## 24) Recommended naming standard inside the model

### Queries / tables
- `DimContact`
- `FactContactActivity`
- `DimCampaign`
- `DimDate`
- `_Measures`

### Measures
Prefix by category if you want very strong organization:

- `Vol | Contacts`
- `Vol | Total Sends`
- `Rate | Open Rate`
- `Rate | CTR`
- `Rate | Soft Bounce Rate`
- `Depth | Avg Opens per Opener`
- `Cohort | Contacts by SubDate`

This becomes very helpful when the field pane grows.

---

## 25) Your best first-version model summary

For version 1, keep it exactly like this:

### Tables
- `DimContact`
- `FactContactActivity`
- `DimCampaign`
- `DimDate`
- `_Measures`

### Active relationships
- `DimContact[email]` → `FactContactActivity[email]`
- `DimCampaign[mailing_id]` → `FactContactActivity[mailing_id]`
- `DimDate[Date]` → `FactContactActivity[ActivityDate]`

### Inactive relationships
- `DimDate[Date]` → `DimContact[SubDate]`
- `DimDate[Date]` → `DimCampaign[MailingScheduleDate]`

### First measures
- Contacts
- Activity Rows
- Campaigns
- Total Sends
- Total Opens
- Total Clicks
- Total Soft Bounces
- Unique Openers
- Unique Clickers
- Open Rate
- CTR
- CTOR
- Soft Bounce Rate
- Opened Not Clicked
- Contacts by SubDate

That is enough to produce a strong first dashboard without overengineering the model.

---

## 26) What I recommend you do next inside Power BI

Follow this exact execution order:

1. Import both CSVs
2. Build `DimContact`
3. Build `FactContactActivity`
4. Build `DimCampaign`
5. Load the model
6. Create `DimDate`
7. Create relationships
8. Hide technical columns
9. Create the base measures
10. Build `00_Model_QA`
11. Validate counts against the raw CSVs
12. Start building the executive and source pages

If you stick to that order, you will avoid most of the common Power BI modeling mistakes.
