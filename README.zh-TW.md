# 工程作品集：系統與架構

**Liu Che Wei** · 台灣桃園
[LinkedIn](https://www.linkedin.com/in/chewei-liu-429a85174/) · [github.com/hawiliu](https://github.com/hawiliu) · `hawiliu@gmail.com`

![C#](https://img.shields.io/badge/C%23-239120?logo=csharp&logoColor=white) ![Python](https://img.shields.io/badge/Python-3776AB?logo=python&logoColor=white) ![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?logo=typescript&logoColor=white) ![.NET](https://img.shields.io/badge/.NET-512BD4?logo=dotnet&logoColor=white) ![FastAPI](https://img.shields.io/badge/FastAPI-009688?logo=fastapi&logoColor=white) ![Vue](https://img.shields.io/badge/Vue-4FC08D?logo=vuedotjs&logoColor=white) ![PostgreSQL](https://img.shields.io/badge/PostgreSQL-4169E1?logo=postgresql&logoColor=white) ![Docker](https://img.shields.io/badge/Docker-2496ED?logo=docker&logoColor=white) ![systems](https://img.shields.io/badge/systems-6-informational)

六個系統，都是用自己的時間與設備做的。下面每一則是一分鐘的摘要，後面連過去的完整文件是十五分鐘的工程紀錄。**不含原始碼**，除了 P6 那四個公開 repository。

[English version](README.md)

## 索引

| # | 系統 | 規模 | 狀態 |
|---|---|---|---|
| **P1** | 自主研究平台 | 61 模組 · 30,800 行 · 612 commit | 運行中，連續 111 天 |
| **P2** | 事件驅動交易引擎 | 164 模組 · 34,000 行 · 72 份規格 | 僅紙上交易 |
| **P3** | 量化研究平台 | 130 模組 · 26,800 行 · 84 commit | 已部署 |
| **P4** | 多角色 TCP 遊戲伺服器 | 202 個 C# 檔 · 40,700 行 · 12 專案 | 分派完整，結算留 stub |
| **P5** | 跨平台行動產品 | 36 模組 · 9,700 行 · 183 commit | 已上架，仍在架上 |
| **P6** | C# 桌面小工具 | 4 個公開 repo · 3,700 行 | GitHub 上公開 |

## 系統

### P1 · 自主研究平台

**61 個 Python 模組 · 約 30,800 行 · 111 天內 612 個 commit · 無人看管運行中**

一條常駐、無人監督的研究管線：產生假說，在每日硬額度下把外部評估花在排序最高的候選上，提交通過驗證閘的結果。搜尋層是 bandit（UCB 與 Thompson）、取最大值回傳的 MCTS/UCT 家族帳本，以及遺傳演算法。漏斗依成本排序：本地硬閘門先擋、bandit 排序，最後才花配額做模擬。五個 LLM 供應商收在單一模組後面，帶跨行程斷路器與明文的降級契約。兩個 SQLite 庫分存結果與研究紀錄，FastAPI 加 Vue 儀表板讀同一組資料。

`Python` · `SQLite` · `FastAPI` · `Vue` · bandit、MCTS/UCT、遺傳演算法 · 跨行程斷路器
[完整文件 →](personal/01-alpha-research-platform.zh-TW.md)

### P2 · 事件驅動交易引擎

**164 個 Python 模組 · 約 34,000 行 · 72 份設計文件（約 29,750 行）· 僅紙上交易**

永續合約的回測與前推模擬引擎，僅紙上交易。訊號層是 OHLCV 上的純函式，放在一個引擎從不繞過的 `Executor` 介面後面，所以回測與實盤同源。逐根事件迴圈、執行模擬器、walk-forward 驗證，以及依流動性分級的滑價模型。164 個模組裡有 126 個是偵測器，其餘是部位大小、市況偵測、訊號 arbitrator 與回測機制。書面規格 72 份、約 29,750 行，篇幅與實作相當。

`Python` · `ccxt` · `parquet` · walk-forward 驗證
[完整文件 →](personal/02-trading-engine.zh-TW.md)

### P3 · 量化研究平台

**87 個 Python + 43 個 TypeScript 模組 · 約 26,800 行 · 84 個 commit · 已部署在反向代理後**

FastAPI 加 Next.js 的全端量化研究平台，部署在帶認證的反向代理後。TimescaleDB 擷取行情，React Flow 節點圖編輯策略，向量化回測，APScheduler 每 15 分鐘跑一輪。研究端有一個 LLM 因子挖掘迴圈，模型產生的程式碼在 AST 受限沙箱裡執行；另有一個負控制端點，把隨機訊號送進完全相同的回測與門檻。訊號計算收斂成單一路徑，由逐根相等測試守住。

`FastAPI` · `Next.js` · `TimescaleDB` · `Redis` · `LightGBM` · `React Flow`
[完整文件 →](personal/03-quant-platform.zh-TW.md)

### P4 · 多角色 TCP 遊戲伺服器（.NET 10）

**202 個 C# 檔 · 12 個專案共約 40,700 行 · 分派完整，結算留 stub**

建在 .NET 10 Generic Host 上的多角色網路伺服器，12 專案 solution，依賴圖無環。三個 `BackgroundService` 角色共用同一個 DI 容器與同一個訊息層，所以 39 個型別化訊息定義只寫一次。連線層跑在裸 `Socket` 上，手寫長度前綴 framing 與串流重組，大端序讀取收在 `BinaryReader` 子類別裡。28 個 handler 以屬性掃描註冊，1,240 行的分派對應表。伺服器本體 6,351 行佔整個 solution 的 16%，工具佔 57%。

`C#` · `.NET` · generic host · 屬性式分派 · `ConcurrentDictionary`
[完整文件 →](personal/04-server-emulator.zh-TW.md)

### P5 · 跨平台行動產品

**36 個 TypeScript 模組 · 約 9,700 行 · 183 個 commit · 已上架 App Store，目前仍在架上**

React 加 Capacitor 的物理遊戲，一份程式碼出 iOS、Android 與 Web，獨力完成到商店送審。五項因平台而異的能力（儲存、認證、購買、廣告、雲端同步）各收在一道服務介面後面，平台判斷只做一次。後端是 Firebase Firestore 加一個 Node cron 排行榜聚合，排行榜在寫入時就反正規化。金流走 RevenueCat 一次整合兩個商店，廣告走 AdMob。目前仍在 App Store 上架：[RoundEvolution](https://apps.apple.com/us/app/roundevolution/id6756927084)。

`React` · `Capacitor` · `matter-js` · `Firebase` · `Node`
[完整文件 →](personal/05-cross-platform-product.zh-TW.md)

### P6 · C# 桌面小工具

**4 個公開 repository · 約 3,700 行 · 50 個 commit · 2019 至 2026**

四個公開 repository，橫跨 2019 到 2026。CryptoWidget 是 Avalonia 加 .NET 8 的跨平台價格與損益小工具，MVVM 分層、ccxt 接交易所、三語言本地化，MIT 授權。TWjpRunner 用 WMI 從執行中的行程取回命令列，再以提權重新啟動。imageZoom 是批次圖片縮放工具，2026 年從 .NET Framework 搬到 .NET 10。aiueo 是 2019 年的假名練習程式。

這是這份作品集裡唯一能讀的原始碼。其他每一份都要求讀者信任它，這四個不要求。

[CryptoWidget](https://github.com/hawiliu/CryptoWidget) · [imageZoom](https://github.com/hawiliu/imageZoom) · [aiueo](https://github.com/hawiliu/aiueo) · [TWjpRunner](https://github.com/hawiliu/TWjpRunner)
[完整文件 →](personal/06-csharp-desktop-tools.zh-TW.md)

## 技術索引

可搜尋，用來確認某項技術我有沒有碰過。

| 領域 | 出現在 |
|---|---|
| **語言** | **C# / .NET**(P4, P6)· Python(P1–P3)· TypeScript(P3, P5)· SQL(P1–P3) |
| **LLM 與 ML** | 多供應商編排、斷路器、結構化輸出（P1）· 沙箱中執行模型產生的程式碼（P3）· scikit-learn、LightGBM、random forest(P3) |
| **搜尋與統計** | bandit、取最大值回傳的 MCTS/UCT、遺傳演算法（P1）· walk-forward 驗證（P2）· 對齊換手率的負控制、t 統計量門檻（P3） |
| **後端與 API** | FastAPI(P1, P3) |
| **即時** | 事件驅動模擬迴圈（P2）· WebSocket 行情（P3） |
| **前端與桌面** | Vue 3(P1)· Next.js、React Flow(P3)· React、Capacitor(P5)· **Avalonia、MVVM**(P6)· Windows Forms(P4, P6) |
| **資料** | **TimescaleDB**(P3)· SQLite 含 WAL(P1, P2)· Redis(P3)· parquet 快取（P2）· 文件資料庫（P5） |
| **基礎設施** | Docker 與 Compose(P3)· 帶認證的反向代理（P3）· APScheduler(P3) |
| **伺服器與連線處理** | 以屬性註冊的 handler 分派（P4）· 39 個型別化訊息定義（P4）· accept 迴圈、逐連線緩衝、整條 async(P4)· 行程內維運主控台（P4）· WMI 行程檢視（P6） |
| **發行與維運** | App store 送審、逐平台 IAP(P5)· MIT 授權公開發行加三語言 i18n(P6) |

## 這裡沒有什麼

受雇期間的工作不在這裡說明，寫在履歷上，面談時可以談。六個系統裡有五個維持私有，其中一個是因為仍在商業運作中，面談階段可以安排讀取權限。

**例外是 P6。** 那四個 repository 是公開的，現在就能讀。它們同時也是這裡規模最小的系統。
