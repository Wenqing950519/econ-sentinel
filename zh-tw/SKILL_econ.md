---
name: econ-sentinel
description: >
  總經數據監控 skill，自動追蹤 FRED 官方發布時間表，在 CPI、PCE、非農、失業率、FOMC 等關鍵數據公布時，
  即時推送實際值／預期值對比與利多利空簡評到 Telegram。
  採自排程循環設計，不做固定頻率輪詢，精準在發布時間觸發。
  觸發關鍵字：「設定總經監控」、「macro sentinel」、「安裝經濟數據通知」、「econ-sentinel」。
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

# SKILL：Econ Sentinel 經濟數據哨兵

## 概述

自動監控 FRED（美聯儲聖路易斯分行）官方發布時間表，在以下關鍵指標公布時即時推送通知到 Telegram：

| 指標 | FRED Series ID |
|------|----------------|
| CPI（消費者物價指數） | `CPIAUCSL` |
| Core CPI（核心 CPI） | `CPILFESL` |
| PCE（個人消費支出物價） | `PCEPI` |
| Core PCE | `PCEPILFE` |
| 非農就業人數 | `PAYEMS` |
| 失業率 | `UNRATE` |
| Fed Funds Rate（聯邦基金利率） | `FEDFUNDS` |

採**自排程循環**設計：根據 FRED 官方發布時間精準觸發，不做固定頻率輪詢，降低 API 消耗。

---

## 環境變數

| 變數 | 說明 | 必填 |
|------|------|------|
| `FRED_API_KEY` | FRED API 金鑰（見 `fred-setup.md`）| ✅ |
| `SENTINEL_TIMEZONE` | 用戶所在時區（影響通知時間顯示）| ✅ |

---

## 首次安裝流程

### Step 1：收集必要變數

詢問用戶：

```
Econ Sentinel 需要以下資訊：

1. 你的 FRED_API_KEY（申請方式見 fred-setup.md）
2. 你的時區（例如 Asia/Taipei、America/New_York）

還沒有 FRED API key 嗎？請先參考 fred-setup.md 完成申請，再回來繼續。
```

等待用戶提供後繼續。

### Step 2：Bootstrap — 抓取初始發布時間表

呼叫 FRED Releases API，抓取接下來最近的 **2 個**受監控指標發布事件：

```
GET https://api.stlouisfed.org/fred/releases/dates?api_key={FRED_API_KEY}&realtime_start={YYYY-MM-DD}&realtime_end={YYYY-12-31}&limit=50&sort_order=asc&file_type=json&include_release_dates_with_no_data=true&offset=0
```

從回應中篩選出屬於受監控 Series ID 的發布紀錄，取最近 2 筆，寫入 `schedule.json`：

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

> 注意：FRED `releases/dates` 端點只回傳發布**日期**，不含具體時間。發布時間硬編碼為 `08:30 ET`——這是美國所有主要經濟指標（CPI、PCE、非農、失業率、FOMC 紀要）的標準發布時間。
```

### Step 3：設置 Cron

依 `schedule.json` 中第一筆事件的發布時間，設置對應的 cron job。

時間換算規則：FRED 公布時間通常為美東時間（ET），cron 設置時使用 `America/New_York` 時區，由系統自動處理夏令時（EDT/EST）切換，不需手動換算。

`SENTINEL_TIMEZONE` 控制 Telegram 通知中的**時間顯示**（換算為用戶本地時間）。Cron 排程無論用戶時區為何，一律使用 `America/New_York`。

### Step 4：確認安裝完成

發送確認訊息到 Telegram：

```
📡 Econ Sentinel 啟動完成

監控指標：CPI / Core CPI / PCE / Core PCE / 非農 / 失業率 / Fed 利率

📅 接下來的發布排程：
• CPIAUCSL — 03/12 08:30 ET
• PAYEMS — 04/04 08:30 ET

數據公布時會即時通知你。
```

---

## 觸發流程（每次 Cron 執行）

### Step 1：抓取最新數據

```
GET https://api.stlouisfed.org/fred/series/observations?series_id={SERIES_ID}&api_key={FRED_API_KEY}&sort_order=desc&limit=3&file_type=json
```

FRED 回傳的 observations 以日期升序排列。使用 `sort_order=desc&limit=3` 後，陣列順序為：
- `observations[0]` = 最新實際值（本次發布）
- `observations[1]` = 前值
- `observations[2]` = 更前一期（需要趨勢參考時使用）

### Step 2：取得預期值

FRED 本身不提供市場預期值。預期值透過 web_search 取得：

```
搜尋："{指標名稱} forecast consensus {發布月份} {年份}"
來源優先順序：Investing.com → Trading Economics → Bloomberg
```

> ⚠️ 預期值來自網路搜尋，非官方數據，不同來源可能略有差異，僅供參考。
> 若搜尋失敗，預期值欄位顯示 `N/A`，仍正常發送通知。

### Step 3：判斷利多 / 利空

依以下邏輯判斷（以**風險資產／Fed 降息空間**角度評估）：

| 指標 | 利多條件 | 利空條件 |
|------|----------|----------|
| CPI / PCE | 實際值 < 預期值（通膨降溫）| 實際值 > 預期值（通膨升溫）|
| 非農 | 實際值 < 預期值（就業降溫，Fed 有降息空間）| 實際值 > 預期值（就業過熱）|
| 失業率 | 實際值 > 預期值（就業降溫）| 實際值 < 預期值（就業過熱）|
| Fed 利率 | 維持或下調 | 升息或鷹派聲明 |

實際值 ≈ 預期值（誤差 < 0.1%）→ ⚪ 中性

### Step 4：格式化 Telegram 通知

```
📡 [{指標名稱}] 數據公布

實際值：{actual}
預期值：{forecast}
前值：{previous}

判斷：🟢 利多 / 🔴 利空 / ⚪ 中性
{一到兩句話說明這個數字對市場的含義，以及下一步值得觀察的方向}

📅 下一個數據：{下一個指標名稱} — {MM/DD HH:MM ET}

— Econ Sentinel {MM/DD HH:MM}
```

範例輸出：
```
📡 [CPI YoY] 數據公布

實際值：3.1%
預期值：3.2%
前值：3.4%

判斷：🟢 利多
通膨連續第二個月低於預期，Fed 降息空間進一步打開。
下一步關注 3/20 FOMC 聲明措辭是否轉鴿。

📅 下一個數據：非農就業 — 04/04 08:30 ET

— Econ Sentinel 03/12 08:35
```

### Step 5：更新排程（自排程循環）

通知發送後，立刻執行以下動作：

1. 將剛觸發的事件從 `schedule.json` 的 `queue` 移除
2. 呼叫 FRED API 抓取下一個受監控指標的發布時間
3. 補入 `queue`，維持 queue 中始終有 2 筆待觸發事件
4. 依新的第一筆事件時間更新 cron job

---

## 手動指令

用戶可隨時觸發以下操作：

| 指令 | 動作 |
|------|------|
| 「查看排程」/ 「下次發布」 | 列出 `schedule.json` 的 queue 內容 |
| 「手動抓 CPI」 | 立刻執行 CPI 的觸發流程並發送通知 |
| 「重設排程」 | 重新執行 Bootstrap（Step 2）並更新 cron |
| 「停止監控」 | 移除所有相關 cron job，清空 schedule.json |

---

## 錯誤處理

| 情況 | 處理方式 |
|------|----------|
| FRED API 回傳錯誤 | 發送：`⚠️ FRED API 呼叫失敗，請確認 FRED_API_KEY 是否有效。` |
| 預期值搜尋失敗 | 預期值欄位顯示 `N/A`，仍發送通知 |
| schedule.json 損毀 | 重新執行 Bootstrap 流程 |
| cron 設置失敗 | 發送警告並提示用戶手動執行「重設排程」|

---

## 注意事項

- FRED 發布時間以**美東時間（ET）**為準，cron 換算時需注意夏令時（EDT = UTC-4，EST = UTC-5）
- `schedule.json` 存放於 skill 的 workspace 目錄下，不得手動刪除
- 預期值來自網路搜尋，非官方數據，僅供參考
- 本 skill 不提供投資建議，所有判斷僅為技術性描述
