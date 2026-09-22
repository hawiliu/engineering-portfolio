# 工程作品集：系統與架構

**Liu Che Wei** · 台灣桃園
[LinkedIn](https://www.linkedin.com/in/chewei-liu-429a85174/) · [github.com/hawiliu](https://github.com/hawiliu) · `hawiliu@gmail.com`

![C#](https://img.shields.io/badge/C%23-239120?logo=csharp&logoColor=white) ![Python](https://img.shields.io/badge/Python-3776AB?logo=python&logoColor=white) ![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?logo=typescript&logoColor=white) ![.NET](https://img.shields.io/badge/.NET-512BD4?logo=dotnet&logoColor=white) ![FastAPI](https://img.shields.io/badge/FastAPI-009688?logo=fastapi&logoColor=white) ![Vue](https://img.shields.io/badge/Vue-4FC08D?logo=vuedotjs&logoColor=white) ![PostgreSQL](https://img.shields.io/badge/PostgreSQL-4169E1?logo=postgresql&logoColor=white) ![Docker](https://img.shields.io/badge/Docker-2496ED?logo=docker&logoColor=white) ![systems](https://img.shields.io/badge/systems-11-informational)

十一個系統：五個在受雇期間完成，六個是我自己的。下面每一則是一分鐘的摘要，後面連過去的完整文件是十五分鐘的工程紀錄。**不含原始碼**，除了 P6 那四個公開 repository。

[English version](README.md)

## 索引

| # | 系統 | 規模 | 狀態 |
|---|---|---|---|
| **E1** | 串流影像理解 | 43 天 286 個 commit，單人 | 服役中 |
| **E2** | 即時 AI 影像牆 | 10 個 release tag，單人 | 已交付，ARM64 離線 |
| **E3** | 協作規劃平台 | 236 個 commit 中的 116 個，四人 | 開發中 |
| **E4** | RAG 客服助理 | 1,461 行攝取管線 | 原型，未上線 |
| **E5** | 負載預測管線 | notebook 轉可配置套件 | 重構完成，模型未驗證 |
| **P1** | 自主研究平台 | 61 模組 · 30,800 行 · 612 commit | 運行中，連續 111 天 |
| **P2** | 事件驅動交易引擎 | 164 模組 · 34,000 行 · 72 份規格 | 僅紙上交易 |
| **P3** | 量化研究平台 | 130 模組 · 26,800 行 · 84 commit | 已部署 |
| **P4** | 遊戲伺服器模擬 | 202 個 C# 檔 · 40,700 行 · 12 專案 | 派送完整，結算留樁 |
| **P5** | 跨平台行動產品 | 36 模組 · 9,700 行 · 183 commit | 已上架，仍在架上 |
| **P6** | C# 桌面小工具 | 4 個公開 repo · 3,700 行 | GitHub 上公開 |

## 個人系統

六個用自己的時間與設備做的系統，給真實數字。

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

### P4 · 遊戲伺服器模擬

**202 個 C# 檔 · 12 個專案共約 40,700 行 · 派送完整，結算留樁**

建在 .NET 10 Generic Host 上的多角色網路伺服器，12 專案 solution，依賴圖無環。三個 `BackgroundService` 角色共用同一個 DI 容器與同一個訊息層，所以 39 個型別化訊息定義只寫一次。連線層跑在裸 `Socket` 上，手寫長度前綴 framing 與串流重組，大端序讀取收在 `BinaryReader` 子類別裡。28 個 handler 以屬性掃描註冊，1,240 行的派送對應表。伺服器本體 6,351 行佔整個 solution 的 16%，工具佔 57%。

`C#` · `.NET` · generic host · 屬性式派送 · `ConcurrentDictionary`
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

## 受雇期間的系統

五個在受雇期間完成的生產與研發系統，只談架構與推理。

### E1 · 即時串流影像理解

**43 天 286 個 commit，單人 · 服役中**

自架平台，以視覺語言模型持續分析即時與錄製影像，讓模型判斷現場在發生什麼，而不是在規則引擎裡窮舉異常。傳統電腦視覺並行處理 VLM 不夠快或不夠精確的任務：人員偵測、屬性辨識、人流計數、虛擬絆線與區域佔用。每串流一條工作執行緒負責取像、降採樣與取樣決策，GPU 推論收在單一 worker 後面序列化。唯一的逐串流設定是一段自由文字的場景描述，換場地不必改程式。

`Python` · `FastAPI` · `Vue 3` · 本地 VLM 推論 · `OpenVINO` · `Docker` · WSL2 GPU 直通
[完整文件 →](employer/01-streaming-video-understanding.zh-TW.md)

### E2 · 即時 AI 監控影像牆

**九路攝影機 · 10 個 release tag · ARM64 離線交付**

真實資料驅動的大螢幕維運牆。外部 AI 監控系統依一份我無權更動的契約推送偵測事件；本系統驗證並持久化它們、即時扇出到每一面連線中的顯示端，並在旁邊渲染九路影像牆。分發走手寫的 Server-Sent Events broadcaster，多路通道加週期性保活流量；影像走 RTSP 轉 WebRTC。以離線 ARM64 封裝交付，跨十個標記版本。

`TypeScript` · `Fastify` · `PostgreSQL` · `Server-Sent Events` · `Vue 3` · `WebRTC` · `Docker`
[完整文件 →](employer/02-realtime-video-wall.zh-TW.md)

### E3 · 多租戶協作規劃平台

**236 個 commit 中的 116 個 · 四人團隊 · 開發中**

企業任務與計畫產品，從單一 codebase 出貨成多種商業組態：兩種功能層級、多品牌、多種認證模式，全程支援即時協作。三條互相獨立的建置期變異軸，靠反射式架構測試而不是 code review 維持正交。前端的 API 客戶端由 OpenAPI 生成，即時層是 SignalR，部署路徑含離線單機 IIS。

`C#` · `.NET` · `PostgreSQL` · `Vue 3` · `Nuxt` · `SignalR` · OpenAPI 生成 · `IIS`
[完整文件 →](employer/03-collaborative-planning-platform.zh-TW.md)

### E4 · RAG 客服助理

**1,461 行攝取管線 · 六個模型家族、四種量化策略 · 原型，未上線**

針對某領域專用 SaaS 產品的檢索增強客服助理，原型與評估階段的工作。離線文件攝取管線把長篇操作手冊、常見問題與錯誤代碼參考轉成檢索導向的知識庫，含中英 OCR、表格抽取與 LLM 重組。知識庫與問題集經人工策展，模型本地託管，前面掛一層防護：共享密鑰認證、回答前分類、輸出過濾。評估涵蓋六個模型家族、四種量化策略、五個服務執行環境、兩個作業系統。

`Python` · 本地 LLM 服務 · 文件處理 · OCR · prompt engineering · `FastAPI`
[完整文件 →](employer/04-rag-support-assistant.zh-TW.md)

### E5 · GPU 加速負載預測管線

**notebook 轉可配置套件 · 重構完成，模型未驗證**

設備遙測與歷史氣象融合後預測下一小時負載，從探索性 notebook 重構成可配置套件。管線分成載入、準備、特徵工程、建模、呈現五個階段，每個階段各檢查一次 GPU 可用性，在沒有 GPU 的機器上透明退到 pandas 與 scikit-learn。標的改成逐小時差分並把當前值排除在特徵之外，切分依時序，自動化的資料品質與洩漏閘接在特徵工程之後。

`Python` · `RAPIDS` · `scikit-learn` · 時序特徵工程 · `SQL Server`
[完整文件 →](employer/05-load-forecasting-pipeline.zh-TW.md)

## 技術索引

可搜尋，用來確認某項技術我有沒有碰過。

| 領域 | 出現在 |
|---|---|
| **語言** | **C# / .NET**(E3, P4, P6)· Python(E1, E4, E5, P1–P3)· TypeScript(E2, P3, P5)· SQL(E3, E5, P1–P3) |
| **LLM 與 ML** | 本地 LLM 服務，vLLM / TGI / Ollama / llama.cpp(E4)· 視覺語言推論（E1）· RAG 與文件攝取（E4）· 多供應商編排、斷路器、結構化輸出（P1）· 沙箱中執行模型產生的程式碼（P3）· scikit-learn、LightGBM、random forest(P3)· RAPIDS、時序特徵工程（E5） |
| **搜尋與統計** | bandit、取最大值回傳的 MCTS/UCT、遺傳演算法（P1）· walk-forward 驗證（P2）· 對齊換手率的負控制、t 統計量門檻（P3） |
| **後端與 API** | FastAPI(E1, E4, P1, P3)· Fastify(E2)· ASP.NET(E3)· OpenAPI 驅動生成（E3） |
| **即時** | Server-Sent Events(E2)· SignalR(E3)· WebRTC(E2)· 事件驅動模擬迴圈（P2）· WebSocket 行情（P3） |
| **前端與桌面** | Vue 3(E1, E2, E3, P1)· Nuxt(E3)· Next.js、React Flow(P3)· React、Capacitor(P5)· **Avalonia、MVVM**(P6)· Windows Forms(P4, P6) |
| **資料** | PostgreSQL(E2, E3)· **TimescaleDB**(P3)· SQL Server(E5)· SQLite 含 WAL(P1, P2)· Redis(P3)· parquet 快取（P2）· 文件資料庫（P5） |
| **基礎設施** | Docker 與 Compose(E1, E2, E4, P3)· IIS(E3)· 帶認證的 Caddy 反向代理（P3）· ARM64 離線交付（E2）· GPU 直通與 container toolkit(E1, E4)· APScheduler(P3) |
| **伺服器與連線處理** | 以屬性註冊的 handler 派送（P4）· 39 個型別化訊息定義（P4）· accept 迴圈、逐連線緩衝、整條 async(P4)· 行程內維運主控台（P4）· WMI 行程檢視（P6） |
| **工業與協定** | Modbus TCP/RTU、OPC UA、BACnet，數千台設備輪詢（較早期的工作，見 CV） |
| **發行與維運** | App store 送審、逐平台 IAP(P5)· MIT 授權公開發行加三語言 i18n(P6)· 10 個 release tag(E2)· build 階段的隱私掃描（E1–E3 的工具） |

## 這裡沒有什麼

五個受雇期間的系統，實作屬於雇主。六個個人系統裡有五個維持私有，其中數個仍在商業運作中，面談階段可以安排讀取權限。

**例外是 P6。** 那四個 repository 是公開的，現在就能讀。它們同時也是這裡最小的東西，而那份文件直說了這件事，沒有繞過它。
