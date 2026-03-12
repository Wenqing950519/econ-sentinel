[English](#english) | [繁體中文](#繁體中文)

---

<a name="english"></a>

# 📡 econ-sentinel

> There's no shortage of apps or websites for economic data. But when the numbers drop — is it bullish or bearish? You still have to figure that out yourself, or manually feed it to an AI.
>
> This skill lets your AI agent handle everything end-to-end: from fetching the data to delivering the verdict, straight to your Telegram the moment it's released.

A self-scheduling OpenClaw skill that monitors FRED's official release calendar and sends instant Telegram alerts for key US economic indicators — with actual vs. forecast comparison and a bullish/bearish signal included.

---

## Features

**📊 Monitors 7 Key Indicators**
CPI, Core CPI, PCE, Core PCE, Non-Farm Payrolls, Unemployment Rate, and Fed Funds Rate — all sourced directly from FRED's official API.

**⚡ Event-Driven, Not Polling**
Uses a self-scheduling loop: after each release, the skill automatically fetches the next upcoming event and reschedules itself. No fixed-interval polling, minimal API usage.

**🟢 Instant Signal**
Every notification includes actual value, forecast, previous value, and a bullish/bearish judgment with a one-line analysis — no manual interpretation needed.

---

## How It Looks

```
📡 [CPI YoY] Data Released

Actual:   3.1%
Forecast: 3.2%
Previous: 3.4%

Signal: 🟢 Bullish
Inflation came in below expectations for the second straight month,
widening the Fed's room to cut. Watch for a dovish shift at the 3/20 FOMC.

📅 Next release: Non-Farm Payrolls — 04/04 08:30 ET

— Econ Sentinel 03/12 08:35
```

---

## Architecture

```
econ-sentinel/
├── en/
│   ├── SKILL.md          # Main skill logic
│   └── fred-setup.md     # FRED API key setup guide
└── zh-tw/
    ├── SKILL.md
    └── fred-setup.md
```

---

## Quick Start

1. Get a free FRED API key — see [`en/fred-setup.md`](en/fred-setup.md) for the full walkthrough (takes about 5 minutes)
2. Add this repo to your OpenClaw installation
3. Tell your agent: `set up macro monitor`
4. Follow the guided setup — provide your API key and timezone, done

---

## Requirements

- [OpenClaw](https://github.com/openclaw/openclaw)
- FRED API key (free — see [`en/fred-setup.md`](en/fred-setup.md))
- Telegram bot connected to your OpenClaw agent

---

<a name="繁體中文"></a>

# 📡 econ-sentinel

> 市面上不缺總經數據的 app 或網站，但數據出來之後——是利多還是利空？還是得自己判斷，或是手動丟給 AI 解讀。
>
> 這套 skill 讓 AI agent 從數據抓取到分析一條龍完成，數據公布的瞬間，結論直接送到你的 Telegram。

一套自排程的 OpenClaw skill，監控 FRED 官方發布時間表，在關鍵美國經濟指標公布時即時推送 Telegram 通知——附上實際值與預期值對比、利多利空判斷。

---

## 功能

**📊 監控 7 大指標**
CPI、核心 CPI、PCE、核心 PCE、非農就業、失業率、Fed 利率——全部直接從 FRED 官方 API 抓取。

**⚡ 事件觸發，非固定輪詢**
採自排程循環設計：每次數據公布後，skill 自動抓取下一個發布時間並重新排程。不做固定頻率輪詢，API 消耗最小化。

**🟢 即時判斷**
每則通知包含實際值、預期值、前值，以及利多利空判斷與一句簡評——不需要手動解讀。

---

## 輸出範例

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

---

## 架構

```
econ-sentinel/
├── en/
│   ├── SKILL.md          # 主 skill 邏輯
│   └── fred-setup.md     # FRED API key 申請指南
└── zh-tw/
    ├── SKILL.md
    └── fred-setup.md
```

---

## 快速開始

1. 申請免費的 FRED API key——完整流程見 [`zh-tw/fred-setup.md`](zh-tw/fred-setup.md)（約 5 分鐘）
2. 將本 repo 加入你的 OpenClaw
3. 告訴你的 agent：「設定總經監控」
4. 跟著引導完成設定——填入 API key 與時區，完成

---

## 環境需求

- [OpenClaw](https://github.com/openclaw/openclaw)
- FRED API key（免費——見 [`zh-tw/fred-setup.md`](zh-tw/fred-setup.md)）
- 連接到你的 OpenClaw agent 的 Telegram bot
