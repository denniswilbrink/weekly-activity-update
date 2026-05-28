# weekly-activity-update

A [Cortex Code](https://docs.snowflake.com/en/user-guide/cortex-code/cortex-code) skill that refreshes the **Weekly Activity Matrix** dashboard — a heatmap showing Use Cases Created and Customer Meetings per week for two commercial sales teams.

## What it does

Queries Snowflake for the latest activity data and rewrites a local HTML dashboard file with fresh numbers. The dashboard covers:

- **Mark Gorelik's team** — 9 AEs, Senior Commercial Sales Manager
- **Jihed Alya Feddaoui's team** — 7 AEs, Commercial Sales Manager

Each team gets two side-by-side heatmap panels:
1. **Use Cases Created per Week** — color-coded from gray (0) to dark green (8+)
2. **Customer Meetings per Week** — color-coded from red (0) to dark green (11+)

## Dashboard Preview

The output is a self-contained HTML file with:
- Color-coded heatmap cells per AE per week
- AVG/WK column with annotations for new hires and PTO weeks
- Partial week indicators (`*`) with data-lag notes
- Key Observations section summarizing both teams

## Installation

1. Copy the `weekly-activity-update/` folder into your Cortex Code skills directory:
   ```
   ~/.snowflake/cortex/skills/weekly-activity-update/
   ```
2. Restart Cortex Code — the skill will be auto-discovered.

## Usage

In Cortex Code chat:
```
run weekly activity update
```
or
```
refresh the activity matrix dashboard
```

Cortex Code will:
1. Determine which Q2 FY27 weeks have data
2. Query use cases created and meetings per AE per week
3. Rewrite the HTML dashboard with fresh data
4. Open the result in the browser

## Data Sources

All queries run against `SALES.RAVEN` in Snowflake:

| Table/View | Used For |
|-----------|---------|
| `SDA_USE_CASE_VIEW` | Use cases created (GO_LIVE_DATE) |
| `SDA_USER_ACCOUNT_ACTIVITY_MEETING_CALL_EMAIL_VIEW` | Customer meetings (ACTIVITY_TYPE = 'MEETING') |
| `RAVEN_EMPLOYEE_WITH_REPORTING_CHAIN` | Team roster, manager hierarchy |

## Fiscal Calendar

Q2 FY27 = May 1 – Jul 31, 2026. Weeks start on Monday (W1 = May 4).

## Configuration

Edit `SKILL.md` to update:
- Manager emails (`mark.gorelik@snowflake.com`, `jihedalya.feddaoui@snowflake.com`)
- Output file path
- Snowflake connection name
- Fiscal quarter date range (update each quarter)

## Known Issues

- **Júlia Lucas Ruiz** may show 0 meetings due to an accent character (`ú`) in her name causing a case-insensitive match failure in Salesforce activity records.
- **W4+ partial weeks** may show lower-than-expected numbers due to data ingestion lag.
