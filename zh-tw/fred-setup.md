# FRED API 申請指南

本文件說明如何申請 FRED API key，這是 `econ-sentinel` skill 的唯一外部依賴。

FRED（Federal Reserve Bank of St. Louis Economic Data）是美聯儲聖路易斯分行維護的免費經濟數據庫，提供超過 800,000 筆官方經濟時間序列數據，包含 CPI、PCE、非農、失業率、Fed 利率等所有受監控指標。

---

## Step 1：申請 FRED API Key

1. 前往 [https://fredaccount.stlouisfed.org/login/secure/](https://fredaccount.stlouisfed.org/login/secure/)
2. 點擊 **Create New Account** 建立帳號（免費）
3. 登入後，前往 [https://fredaccount.stlouisfed.org/apikeys](https://fredaccount.stlouisfed.org/apikeys)
4. 點擊 **Request API Key**
5. 填寫用途說明（例如：`Personal use for economic data monitoring`）
6. 點擊 **Request API Key**，key 會立即顯示

> API key 格式為 32 位英數字串，例如：`abcdef1234567890abcdef1234567890`

---

## Step 2：設定環境變數

```bash
export FRED_API_KEY='your_fred_api_key_here'
export SENTINEL_TIMEZONE='Asia/Taipei'  # 改成你的時區
```

永久生效（加入 `~/.bashrc` 或 `~/.zshrc`）：

```bash
echo "export FRED_API_KEY='your_fred_api_key_here'" >> ~/.bashrc
echo "export SENTINEL_TIMEZONE='Asia/Taipei'" >> ~/.bashrc
source ~/.bashrc
```

---

## Step 3：驗證 API Key

```bash
curl "https://api.stlouisfed.org/fred/series?series_id=CPIAUCSL&api_key=$FRED_API_KEY&file_type=json"
```

回傳 JSON 且包含 `"title": "Consumer Price Index"` 即代表 key 有效。

---

## API 使用限制

| 項目 | 限制 |
|------|------|
| 費用 | 完全免費 |
| 每日請求上限 | 120 次／IP（一般個人使用遠低於此）|
| 數據更新頻率 | 官方發布後數分鐘內更新 |

econ-sentinel 的自排程設計每次觸發約消耗 2–3 次 API call，每月總消耗遠低於限制。

---

## 常見問題

| 問題 | 解決方式 |
|------|----------|
| API key 申請後沒有立即生效 | 通常幾分鐘內生效，若超過 30 分鐘請重新整理頁面 |
| 回傳 `{"error_code":400,"error_message":"Bad Request"}` | 確認 `FRED_API_KEY` 環境變數已正確 export |
| 回傳 `{"error_code":429}` | 已達每日請求上限，等待 24 小時後自動重置 |
