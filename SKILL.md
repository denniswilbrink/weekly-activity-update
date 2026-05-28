---
name: weekly-activity-update
description: "Refresh the Weekly Activity Matrix dashboard for Mark Gorelik and Jihed Feddaoui's teams. Queries Snowflake for the latest go-lives (use cases created) and customer meetings per week, then rewrites the HTML heatmap file. Use when: weekly activity matrix, weekly update dashboard, refresh activity heatmap, update go-lives and meetings. Triggers: weekly activity update, activity matrix, refresh the dashboard, update the heatmap."
---

# Weekly Activity Matrix — Update Skill

Refreshes `/Users/dwilbrink/.snowflake/cortex/playground/workspace/weekly_activity_matrix.html`
with fresh go-live and meeting counts for both teams.

## Config

| Parameter | Value |
|-----------|-------|
| Output file | `/Users/dwilbrink/.snowflake/cortex/playground/workspace/weekly_activity_matrix.html` |
| Connection | `SFCOGSOPS-SNOWHOUSE_AWS_US_WEST_2` |
| Mark's manager email | `mark.gorelik@snowflake.com` |
| Jihed's manager email | `jihedalya.feddaoui@snowflake.com` |
| Schema | `SALES.RAVEN` |

## Fiscal Calendar

Q2 FY27 = May 1 – Jul 31, 2026. Weeks start on Monday:

| Week | Date Range | Notes |
|------|-----------|-------|
| W1 | May 4–10 | |
| W2 | May 11–17 | |
| W3 | May 18–24 | |
| W4 | May 25–31 | |
| W5 | Jun 1–7 | |
| W6 | Jun 8–14 | |
| W7 | Jun 15–21 | |
| W8 | Jun 22–28 | |
| W9 | Jun 29–Jul 5 | |
| W10 | Jul 6–12 | |
| W11 | Jul 13–19 | |
| W12 | Jul 20–26 | |
| W13 | Jul 27–31 | partial |

Include all weeks from W1 up through the most recent completed week (or current partial week).
Mark partial weeks with `*` in the column header.

## Workflow

### Step 1: Determine Current Weeks

Calculate which weeks have data:
- All full weeks from W1 (May 4) up to the week containing `CURRENT_DATE()`.
- The current in-progress week is partial — mark it with `*` and note data lag.
- Update the dashboard header date range accordingly (e.g., "May 1–Jun 14").

### Step 2: Query Use Cases Created (Go-Lives)

Run for **each manager** (Mark + Jihed):

```sql
SELECT
    DATE_TRUNC('week', uc.GO_LIVE_DATE) AS week_start,
    e.EMPLOYEE_NAME AS ae_name,
    COUNT(*) AS go_lives
FROM SALES.RAVEN.SDA_USE_CASE_VIEW uc
JOIN SALES.RAVEN.RAVEN_EMPLOYEE_WITH_REPORTING_CHAIN e
    ON UPPER(uc.OWNER_EMAIL) = UPPER(e.PRIMARY_WORK_EMAIL)
WHERE uc.GO_LIVE_DATE >= '2026-05-01'
  AND uc.GO_LIVE_DATE <= CURRENT_DATE()
  AND UPPER(e.MANAGER_EMAIL) = UPPER('<MANAGER_EMAIL>')
  AND e.IS_ACTIVE = TRUE
GROUP BY week_start, e.EMPLOYEE_NAME
ORDER BY week_start, go_lives DESC;
```

Replace `<MANAGER_EMAIL>` with each manager's email.

### Step 3: Query Customer Meetings

Run for **each manager** (Mark + Jihed):

```sql
SELECT
    DATE_TRUNC('week', a.ACTIVITY_DATE) AS week_start,
    a.OWNER AS ae_name,
    COUNT(*) AS meetings
FROM SALES.RAVEN.SDA_USER_ACCOUNT_ACTIVITY_MEETING_CALL_EMAIL_VIEW a
WHERE a.ACTIVITY_DATE >= '2026-05-04'
  AND a.ACTIVITY_DATE <= CURRENT_DATE()
  AND a.ACTIVITY_TYPE = 'MEETING'
  AND UPPER(a.OWNER) IN (
      SELECT UPPER(EMPLOYEE_NAME)
      FROM SALES.RAVEN.RAVEN_EMPLOYEE_WITH_REPORTING_CHAIN
      WHERE UPPER(MANAGER_EMAIL) = UPPER('<MANAGER_EMAIL>')
        AND IS_ACTIVE = TRUE
  )
GROUP BY week_start, a.OWNER
ORDER BY week_start, meetings DESC;
```

### Step 4: Fetch Team Rosters

Query both rosters to get the complete AE list (in case an AE has zero activity in all weeks and won't appear in the aggregated results):

```sql
SELECT EMPLOYEE_NAME, PRIMARY_WORK_EMAIL
FROM SALES.RAVEN.RAVEN_EMPLOYEE_WITH_REPORTING_CHAIN
WHERE UPPER(MANAGER_EMAIL) = UPPER('<MANAGER_EMAIL>')
  AND IS_ACTIVE = TRUE
ORDER BY EMPLOYEE_NAME;
```

### Step 5: Build the Per-AE Data Tables

For each AE on each team, pivot the query results into a per-week grid:
- Weeks with no data = 0.
- Avg/wk = total / number of active weeks for that AE.
- If an AE was not on the team for W1 (new hire), mark W1 with `—` and reduce the denominator.
- If an AE logged 0 meetings in a week, show `PTO?` in the meetings panel if surrounded by active weeks — otherwise show `0`.
- Known issue: **Júlia Lucas Ruiz** may show 0 meetings due to a name accent mismatch in Salesforce. Preserve the footnote noting this.

**Color classes for use cases created (go-lives):**

| Value | Class |
|-------|-------|
| 0 | `gl-0` |
| 1–2 | `gl-1` / `gl-2` |
| 3–4 | `gl-3` |
| 5–7 | `gl-5p` |
| 8+ | `gl-8p` |

**Color classes for meetings:**

| Value | Class |
|-------|-------|
| PTO? | `mt-pto` |
| 0 | `mt-0` |
| 1 | `mt-1` |
| 2 | `mt-2` |
| 3 | `mt-3` |
| 4 | `mt-4` |
| 5 | `mt-5` |
| 6 | `mt-6` |
| 7 | `mt-7` |
| 8 | `mt-8` |
| 9 | `mt-9` |
| 10 | `mt-10` |
| 11+ | `mt-11p` |

Sort each panel: go-lives panel by total desc; meetings panel by avg/wk desc.

### Step 6: Add New Week Columns if Needed

If weeks have passed since the last run:
- Add new `<th>` columns for the new week(s) in both panels of both teams.
- Add corresponding `<td>` cells for every AE row.
- Update the dashboard header date range.
- Update the info bar week list.

### Step 7: Rewrite the HTML File

Update the existing file at the configured output path. Do NOT recreate the full file from scratch — use targeted edits to replace only the table `<tbody>` content and headers.

Key sections to update:
1. Header `<span class="header-title">` — update date range
2. Info bar — update week list if new weeks added
3. Mark's go-lives `<tbody>` — replace all `<tr>` rows
4. Mark's meetings `<tbody>` — replace all `<tr>` rows
5. Jihed's go-lives `<tbody>` — replace all `<tr>` rows
6. Jihed's meetings `<tbody>` — replace all `<tr>` rows
7. Key Observations — update bullets with fresh highlights

### Step 8: Open and Verify

Open the file in the browser:
```
file:///Users/dwilbrink/.snowflake/cortex/playground/workspace/weekly_activity_matrix.html
```

Confirm both team sections render correctly with the new data.

## Stopping Points

- ✋ After Step 1: Confirm the week range before querying (especially if today is the start of a new week and data may be sparse).

## Notes

- `SDA_USE_CASE_VIEW.GO_LIVE_DATE` is used as a proxy for "use cases created" — the panel title in the HTML is "Use Cases Created per Week".
- W4* and later partial weeks may show lower numbers due to data lag. Always annotate partial weeks with `*`.
- The Avg/wk footnote `(3 wks)` should only appear when fewer than 4 full weeks have data for that AE.
- Jie Shen (Mark's team) joined in W2 — mark W1 with `—` and calculate avg over active weeks only.
