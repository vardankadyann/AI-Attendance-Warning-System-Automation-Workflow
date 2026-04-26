# AI-Powered Attendance & Warning System
## Technical Documentation — Growify HR Automation
**Candidate Submission | 48-Hour Challenge**

---

## Architecture Overview

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                         DATA FLOW — END TO END                               │
└──────────────────────────────────────────────────────────────────────────────┘

  HR Manager
  uploads Excel
      │
      ▼
┌─────────────┐    POST multipart/form-data
│  Webhook    │◄──────────────────────────────  Biometric Excel Export (.xlsx)
│  Trigger    │
└──────┬──────┘
       │
       ▼
┌─────────────────────┐
│  Extract Spreadsheet │  Parses each row → { EmployeeID, Name, Date, Time, Status }
│  File (n8n node)    │
└──────────┬──────────┘
           │
           ▼
┌──────────────────────────────────────────────────────────┐
│              Code Node — Aggregate & Transform           │
│                                                          │
│  For each EmployeeID + Date:                             │
│  ┌─────────────────────────────────────────────────────┐ │
│  │  All IN punches  → sort ascending  → take first     │ │
│  │  All OUT punches → sort descending → take last      │ │
│  │  If CheckIn > 11:00 AM → lateFlag = YES             │ │
│  │  If no OUT punches found → missingOut = true        │ │
│  └─────────────────────────────────────────────────────┘ │
└────────────┬─────────────────────────────────────────────┘
             │
     ┌───────┼──────────────────────┐
     ▼       ▼                      ▼
┌─────────┐ ┌──────────────────┐  ┌──────────────────────┐
│ Google  │ │  Filter: Only    │  │  Filter: Missing OUT  │
│ Sheets  │ │  Late Arrivals   │  │  Punch Only           │
│ Write   │ │  (lateFlag=YES)  │  └─────────┬────────────┘
│ Daily   │ └────────┬─────────┘            │
│ Records │          │                      ▼
└─────────┘          │             ┌─────────────────────┐
                     │             │  Gmail Alert to HR  │
                     ▼             │  (data quality flag)│
            ┌────────────────┐     └─────────────────────┘
            │  Google Sheets │
            │  Read Monthly  │
            │  Late Counter  │
            └───────┬────────┘
                    │
                    ▼
            ┌────────────────┐
            │  Code Node     │
            │  Calculate New │
            │  Strike Count  │
            └───────┬────────┘
                    │
                    ▼
            ┌────────────────┐
            │  Google Sheets │
            │  Update Monthly│
            │  Late Counter  │
            └───────┬────────┘
                    │
                    ▼
            ┌────────────────┐
            │  Switch Node   │  Routes by strikeLevel (1, 2, or 3)
            └──┬─────┬────┬──┘
               │     │    │
        Strike1│     │Strike2    │Strike3
               ▼     ▼    ▼
        ┌─────────┐ ┌─────────┐ ┌──────────────────────────────┐
        │ AI Agent│ │ AI Agent│ │  Google Calendar             │
        │ Friendly│ │  Firm   │ │  Create 5 PM Meeting Event   │
        │ Reminder│ │ Warning │ └───────────────┬──────────────┘
        └────┬────┘ └────┬────┘                 │
             │           │         ┌────────────▼──────────────┐
             ▼           ▼         │  AI Agent — Final Warning │
        ┌─────────┐ ┌─────────┐    │  (with calendar link)     │
        │  Gmail  │ │  Gmail  │    └────────────┬──────────────┘
        │  Send   │ │  Send   │                 ▼
        │ Strike1 │ │ Strike2 │         ┌────────────────┐
        │  Email  │ │  Email  │         │  Gmail — Send  │
        └─────────┘ └─────────┘         │  Strike3 Email │
                                        └────────────────┘
```

---

## Database / Sheet Structure

### Structure 1 — Daily Processed Records (tab: `DailyRecords`)

| Column        | Type    | Description                                       |
|---------------|---------|---------------------------------------------------|
| Employee ID   | String  | Unique identifier (e.g. EMP-001)                 |
| Name          | String  | Full employee name                                |
| Date          | Date    | YYYY-MM-DD format                                 |
| Check-In      | String  | Earliest IN punch (HH:MM AM/PM) or "MISSING"     |
| Check-Out     | String  | Latest OUT punch (HH:MM AM/PM) or "MISSING"      |
| Late Flag     | String  | YES / NO                                          |
| Missing OUT   | Boolean | TRUE if no OUT punch found                        |

**Sample data:**

| Employee ID | Name       | Date       | Check-In   | Check-Out  | Late Flag | Missing OUT |
|-------------|------------|------------|------------|------------|-----------|-------------|
| EMP-001     | Jane Doe   | 2025-07-01 | 11:15 AM   | 06:00 PM   | YES       | FALSE       |
| EMP-002     | John Smith | 2025-07-01 | 09:45 AM   | 05:30 PM   | NO        | FALSE       |
| EMP-003     | Riya Patel | 2025-07-02 | 11:32 AM   | MISSING    | YES       | TRUE        |

**Unique key:** `Employee ID` + `Date` (enforced via appendOrUpdate matching)

---

### Structure 2 — Monthly Late Counter (tab: `MonthlyLateCounter`)

| Column                    | Type   | Description                                      |
|---------------------------|--------|--------------------------------------------------|
| Employee ID               | String | Unique identifier                                |
| Name                      | String | Full employee name                               |
| Current Monthly Late Count| Number | Cumulative late count for the month              |
| Last Warning Date         | Date   | Date the most recent warning email was sent      |
| Month/Year                | String | e.g. "July 2025" (resets each new month)        |

**Sample data:**

| Employee ID | Name       | Current Monthly Late Count | Last Warning Date | Month/Year |
|-------------|------------|----------------------------|-------------------|------------|
| EMP-001     | Jane Doe   | 3                          | 2025-07-10        | July 2025  |
| EMP-002     | John Smith | 1                          | 2025-07-05        | July 2025  |

**Unique key:** `Employee ID` + `Month/Year`

**Auto-reset:** Each new month creates a fresh row — old months are preserved for historical audit.

---

## Edge Case Handling

### Missing OUT Punch

**Detection:** In the `Aggregate & Transform Punches` code node, if `outPunches.length === 0` after grouping, the record gets `missingOut: true` and `checkOut: "MISSING"`.

**Behavior:**
1. The daily record is still written to Google Sheets with `Check-Out = MISSING` and `Missing OUT = TRUE`.
2. A separate filter node (`Filter — Missing OUT Punch Only`) catches these records and triggers an immediate Gmail alert to `hr@growify.in` with the employee name, ID, date, and check-in time.
3. The employee's late flag is still evaluated based on their check-in time only — a missing OUT punch does not block the late detection pipeline.
4. HR can manually update the Check-Out column after verifying the physical register or contacting the employee.

### Other Edge Cases Handled

| Edge Case | Handling |
|-----------|----------|
| Employee punches IN multiple times | Only earliest IN is used as Check-In |
| Employee punches OUT multiple times | Only latest OUT is used as Check-Out |
| Midnight shifts (OUT after 12 AM) | Time is parsed as 24h; date grouping uses the IN punch date |
| First-ever late for an employee | `appendOrUpdate` creates a new row with count = 1 |
| Strike count exceeds 3 | `Math.min(newCount, 3)` caps routing at Strike 3 treatment |
| Malformed date/time formats | `parseTime()` handles AM/PM, 24h, and with/without seconds |
| Missing Employee ID in row | Row is silently skipped in the transform node |
| Employee email unknown | Email derived as `{employeeId}@growify.in` by default |

---

## Scalability Plan — 500 Employees, 5 Locations

### Throughput

**Current capacity:** n8n Cloud handles ~10,000 executions/month on the Pro plan. For 500 employees × 25 working days = 12,500 daily records/month. Upgrade to the Enterprise plan or self-host n8n on a VPS (2 vCPU, 4 GB RAM) to handle unlimited executions with no throttle.

**Batch processing:** Instead of processing one employee at a time through the pipeline, the Code node already runs `runOnceForAllItems` — meaning all 500 employees are grouped in one execution pass. The Google Sheets write uses `appendOrUpdate` which performs batch writes. No changes needed here.

**Google Sheets API rate limit:** The Sheets API allows 300 requests/minute per project. With 500 employees writing 2 rows (daily record + monthly counter update), we hit 1,000 writes per upload. Introduce a `Split In Batches` node before the Sheets write nodes (batch size: 50) with a 1-second delay to stay safely within rate limits.

### Data Partitioning

Add a `Location` field to both sheet structures and the input Excel. Route processing through a Switch node keyed on `Location` before writing to Sheets. Use separate Google Sheets tabs per location (`DailyRecords_Delhi`, `DailyRecords_Mumbai`, etc.) or a single sheet with a Location column and filtered views per HR manager.

For heavier workloads (500+ employees), replace Google Sheets with a PostgreSQL database. The schema maps directly:

```sql
-- Daily records
CREATE TABLE daily_records (
  employee_id   VARCHAR(20),
  name          VARCHAR(100),
  date          DATE,
  check_in      TIME,
  check_out     TIME,
  late_flag     BOOLEAN,
  missing_out   BOOLEAN,
  location      VARCHAR(50),
  PRIMARY KEY (employee_id, date)
);

-- Monthly late counter
CREATE TABLE monthly_late_counter (
  employee_id     VARCHAR(20),
  name            VARCHAR(100),
  month_year      VARCHAR(20),
  late_count      INTEGER DEFAULT 0,
  last_warning    DATE,
  location        VARCHAR(50),
  PRIMARY KEY (employee_id, month_year)
);
```

Use the n8n Postgres node (already supports credentials + query templates) — swap all Google Sheets nodes with Postgres nodes. Indexes on `(employee_id, date)` and `(location, month_year)` ensure sub-10ms lookups at 500+ employees.

### Location-Based Rules

Add a `location` field to the webhook payload and propagate it through the pipeline. Use a Switch node after transformation to apply per-location late thresholds (e.g., Mumbai office: 10:30 AM due to local traffic patterns) stored in a `LocationConfig` tab in Google Sheets or a config table in Postgres. The AI prompts dynamically incorporate the location-specific policy so emails reference the correct check-in time for each office.

---

## AI Prompt Engineering Notes

All three warning emails use **zero-shot prompting** with rich contextual constraints:
- Employee name, ID, date, check-in time, and strike count are injected directly.
- Tone instructions escalate explicitly: "friendly" → "firm" → "formal and unambiguous."
- The model is instructed to avoid generic placeholders and output only the email body.
- Temperature is set to 0.7 — creative enough to feel human-written, grounded enough to stay professional.
- `max_tokens: 800` prevents runaway responses while allowing 4–5 paragraphs comfortably.

---

*Submitted by candidate | Growify Technical Assessment | 48-Hour Challenge*
