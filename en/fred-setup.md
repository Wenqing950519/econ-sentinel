# FRED API Setup Guide

This document explains how to obtain a FRED API key — the only external dependency for `econ-sentinel`.

FRED (Federal Reserve Bank of St. Louis Economic Data) is a free economic database maintained by the St. Louis Fed, providing over 800,000 official economic time series including CPI, PCE, Non-Farm Payrolls, Unemployment Rate, and Fed Funds Rate.

---

## Step 1: Request a FRED API Key

1. Go to [https://fredaccount.stlouisfed.org/login/secure/](https://fredaccount.stlouisfed.org/login/secure/)
2. Click **Create New Account** (free)
3. After logging in, go to [https://fredaccount.stlouisfed.org/apikeys](https://fredaccount.stlouisfed.org/apikeys)
4. Click **Request API Key**
5. Enter a purpose description (e.g. `Personal use for economic data monitoring`)
6. Click **Request API Key** — your key will appear immediately

> API keys are 32-character alphanumeric strings, e.g. `abcdef1234567890abcdef1234567890`

---

## Step 2: Set Environment Variables

```bash
export FRED_API_KEY='your_fred_api_key_here'
export SENTINEL_TIMEZONE='Asia/Taipei'  # change to your timezone
```

To persist across sessions, add to `~/.bashrc` or `~/.zshrc`:

```bash
echo "export FRED_API_KEY='your_fred_api_key_here'" >> ~/.bashrc
echo "export SENTINEL_TIMEZONE='Asia/Taipei'" >> ~/.bashrc
source ~/.bashrc
```

---

## Step 3: Verify Your API Key

```bash
curl "https://api.stlouisfed.org/fred/series?series_id=CPIAUCSL&api_key=$FRED_API_KEY&file_type=json"
```

A successful response will include `"title": "Consumer Price Index"` in the JSON output.

---

## API Usage Limits

| Item | Limit |
|------|-------|
| Cost | Free |
| Daily request limit | 120 requests/IP (typical personal usage is far below this) |
| Data update frequency | Updated within minutes of official release |

Macro Sentinel's self-scheduling design uses approximately 2–3 API calls per trigger event, well within the daily limit for personal use.

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Key not working immediately after creation | Usually activates within minutes — wait up to 30 minutes and retry |
| `{"error_code":400,"error_message":"Bad Request"}` | Confirm `FRED_API_KEY` is correctly exported as an environment variable |
| `{"error_code":429}` | Daily request limit reached — resets automatically after 24 hours |
