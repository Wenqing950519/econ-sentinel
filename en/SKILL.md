---
name: econ-sentinel
description: >
  An economic data monitoring skill that tracks FRED's official release schedule and sends
  instant Telegram notifications when key indicators are published — including CPI, PCE,
  Non-Farm Payrolls, Unemployment Rate, and FOMC rate decisions.
  Features a self-scheduling loop design: no fixed-frequency polling, triggers precisely
  at official release times.
  Trigger keywords: "set up macro monitor", "macro sentinel", "install economic data alerts", "econ-sentinel".
version: 1.0.0
metadata:
  openclaw:
    requires:
      env:
        - FRED_API_KEY
        - SENTINEL_TIMEZONE
      bins: []
    primaryEnv: FRED_API_KEY
    emoji: "📡"
    homepage: https://github.com/Wenqing950519/econ-sentinel
---

# SKILL: Econ Sentinel

## Overview

Automatically monitors FRED (Federal Reserve Bank of St. Louis Economic Data) for official release schedules, sending instant Telegram notifications when the following key indicators are published:

| Indicator | FRED Series ID |
|-----------|----------------|
| CPI (Consumer Price Index) | `CPIAUCSL` |
| Core CPI | `CPILFESL` |
| PCE (Personal Consumption Expenditures Price Index) | `PCEPI` |
| Core PCE | `PCEPILFE` |
| Non-Farm Payrolls | `PAYEMS` |
| Unemployment Rate | `UNRATE` |
| Fed Funds Rate | `FEDFUNDS` |

Uses a **self-scheduling loop** design: triggers precisely at official release times rather than polling at fixed intervals, minimizing API usage.

---

## Environment Variables

| Variable | Description | Required |
|----------|-------------|----------|
| `FRED_API_KEY` | FRED API key (see `fred-setup.md`) | ✅ |
| `SENTINEL_TIMEZONE` | Your local timezone (affects notification time display) | ✅ |

---

## First-Time Setup

### Pre-Flight Check (runs on every activation)

Before any action, check whether `schedule.json` exists and contains a valid queue:

```
if schedule.json does not exist OR queue is empty:
    → run Bootstrap (Steps 2–4 below)
else:
    → display current schedule and resume normal operation
```

This means the user never needs to run a separate initialization command — the skill self-initializes on first use and self-heals if the schedule file is missing or corrupted.

### Step 1: Collect Required Variables

Ask the user:

```
Econ Sentinel needs the following to get started:

1. Your FRED_API_KEY (see fred-setup.md for how to get one)
2. Your timezone (e.g. Asia/Taipei, America/New_York)

Don't have a FRED API key yet? Please complete fred-setup.md first, then come back to continue.
```

Once the user provides both values, **immediately proceed through Steps 2–4 without waiting for further input**. The skill will be fully running by the end of Step 4 — no additional activation command needed.

### Step 2: Bootstrap — Fetch Initial Release Schedule

Call the FRED Releases API to fetch the next **2** upcoming release dates for monitored indicators:

```
GET https://api.stlouisfed.org/fred/releases/dates?api_key={FRED_API_KEY}&realtime_start={YYYY-MM-DD}&realtime_end={YYYY-12-31}&limit=50&sort_order=asc&file_type=json&include_release_dates_with_no_data=true&offset=0
```

Filter results to only include Series IDs in the monitored list. Take the next 2 entries and write to `schedule.json`:

```json
{
  "queue": [
    {
      "series_id": "CPIAUCSL",
      "release_date": "2026-03-12",
      "release_time_et": "08:30"
    },
    {
      "series_id": "PAYEMS",
      "release_date": "2026-04-04",
      "release_time_et": "08:30"
    }
  ],
  "last_updated": "2026-03-12T00:00:00Z"
}
```

> Note: FRED's `releases/dates` endpoint returns release **dates** only, not specific times. Release time is hardcoded as `08:30 ET` — this is the standard time for all major US economic indicators (CPI, PCE, Non-Farm Payrolls, Unemployment Rate, FOMC minutes).
```

### Step 3: Set Cron Job

Schedule a cron job based on the first entry in `schedule.json`.

Timezone handling: FRED release times are in US Eastern Time (ET). Use the `America/New_York` timezone when setting the cron — the system will automatically handle Daylight Saving Time (EDT/EST) transitions without manual conversion.

`SENTINEL_TIMEZONE` controls how times are **displayed** in Telegram notifications (converted to the user's local time). Cron scheduling always uses `America/New_York` regardless of the user's timezone.

### Step 4: Confirm Setup

Send a confirmation message to Telegram:

```
📡 Econ Sentinel is active

Monitoring: CPI / Core CPI / PCE / Core PCE / Non-Farm Payrolls / Unemployment Rate / Fed Rate

📅 Upcoming releases:
• CPIAUCSL — 03/12 08:30 ET
• PAYEMS — 04/04 08:30 ET

You'll be notified when data drops.
```

---

## Trigger Flow (Each Cron Execution)

### Step 1: Fetch Latest Data

```
GET https://api.stlouisfed.org/fred/series/observations?series_id={SERIES_ID}&api_key={FRED_API_KEY}&sort_order=desc&limit=3&file_type=json
```

FRED returns observations in ascending order by date. With `sort_order=desc&limit=3`, the array is:
- `observations[0]` = latest actual value (current release)
- `observations[1]` = previous value
- `observations[2]` = the one before that (for trend reference if needed)

### Step 2: Fetch Consensus Estimate

FRED does not provide market consensus estimates. Fetch via web_search:

```
Search: "{indicator name} forecast consensus {release month} {year}"
Source priority: Investing.com → Trading Economics → Bloomberg
```

> ⚠️ Consensus estimates are sourced from web search, not official data. Different sources may vary slightly — treat as reference only.
> If the search fails, the forecast field will show `N/A` and the notification will still be sent.

### Step 3: Determine Bullish / Bearish Signal

Logic below is evaluated from a **risk assets / Fed rate-cut headroom** perspective:

| Indicator | Bullish | Bearish |
|-----------|---------|---------|
| CPI / PCE | Actual < Forecast (inflation cooling) | Actual > Forecast (inflation hot) |
| Non-Farm Payrolls | Actual < Forecast (labor cooling, more Fed room) | Actual > Forecast (labor overheating) |
| Unemployment Rate | Actual > Forecast (labor cooling) | Actual < Forecast (labor overheating) |
| Fed Funds Rate | Hold or cut | Hike or hawkish statement |

Actual ≈ Forecast (delta < 0.1%) → ⚪ Neutral

### Step 4: Format Telegram Notification

```
📡 [{Indicator Name}] Data Released

Actual:   {actual}
Forecast: {forecast}
Previous: {previous}

Signal: 🟢 Bullish / 🔴 Bearish / ⚪ Neutral
{One to two sentences on what this print means for markets and what to watch next.}

📅 Next release: {next indicator name} — {MM/DD HH:MM ET}

— Econ Sentinel {MM/DD HH:MM}
```

Example output:
```
📡 [CPI YoY] Data Released

Actual:   3.1%
Forecast: 3.2%
Previous: 3.4%

Signal: 🟢 Bullish
Inflation came in below expectations for the second straight month, widening the Fed's room to cut.
Watch for a dovish shift in tone at the 3/20 FOMC statement.

📅 Next release: Non-Farm Payrolls — 04/04 08:30 ET

— Econ Sentinel 03/12 08:35
```

### Step 5: Update Schedule (Self-Scheduling Loop)

After sending the notification:

1. Remove the triggered event from `schedule.json` queue
2. Call FRED API to fetch the next upcoming release for monitored indicators
3. Append it to the queue, keeping 2 pending events at all times
4. Update the cron job to match the new first entry

---

## Manual Commands

Users can trigger the following at any time:

| Command | Action |
|---------|--------|
| "start econ sentinel" | First-time activation — runs Bootstrap and sets up cron |
| "show schedule" / "next release" | List current `schedule.json` queue |
| "fetch CPI now" | Immediately run the CPI trigger flow and send notification |
| "reset schedule" | Re-run Bootstrap (Step 2) and update cron |
| "stop monitoring" | Remove all related cron jobs and clear `schedule.json` |

---

## Error Handling

| Situation | Response |
|-----------|----------|
| FRED API error | Send: `⚠️ FRED API call failed. Please check that your FRED_API_KEY is valid.` |
| Forecast search failed | Show `N/A` for forecast field, send notification anyway |
| `schedule.json` corrupted | Re-run Bootstrap flow |
| Cron setup failed | Send warning and prompt user to run "reset schedule" |

---

## Notes

- FRED release times are in **US Eastern Time (ET)**. Use `America/New_York` for cron scheduling to handle DST automatically
- `schedule.json` is stored in the skill's workspace directory — do not delete it manually
- Consensus estimates are sourced from web search and are for reference only
- This skill does not provide investment advice — all signals are technical descriptions only
