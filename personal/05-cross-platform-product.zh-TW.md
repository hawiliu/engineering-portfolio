# 跨平台行動產品

[← 作品集索引](../README.zh-TW.md) · [English version](05-cross-platform-product.md)

![React](https://img.shields.io/badge/React-61DAFB?logo=react&logoColor=white) ![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?logo=typescript&logoColor=white) ![Capacitor](https://img.shields.io/badge/Capacitor-119EFF?logo=capacitor&logoColor=white) ![Firebase](https://img.shields.io/badge/Firebase-FFCA28?logo=firebase&logoColor=white) ![App Store: live](https://img.shields.io/badge/App%20Store-live-0D96F6?logo=appstore&logoColor=white)

> 用 React 加 Capacitor 做的物理遊戲，獨力上架 iOS App Store，含內購、廣告、認證與 Firebase 後端。一份程式碼、三個目標，而 Web 端沒有原生能力，所以降級發生在服務邊界上而不是遊戲裡。

## 1. 專案綱要

| 項目 | 內容 |
|---|---|
| **類型** | 商業行動產品 · 一份程式碼出 iOS + Android + Web |
| **角色** | 獨力完成，含商店送審 |
| **期間** | 2025-12-04 至 2026-01-03 · **183 個 commit 裡有 180 個落在 2025 年 12 月** |
| **規模** | 36 個 TypeScript 模組 · 客戶端與伺服器合計約 9,700 行 |
| **目的** | 把完整的商業發行週期從頭走到尾 |
| **狀態** | 已上架 App Store,**目前仍在架上**（[App Store](https://apps.apple.com/us/app/roundevolution/id6756927084)） |
| **原始碼** | 私有 · [landing page](https://ha516.com/RoundEvolution/index.html) |

## 2. 技術棧

| 層 | 技術 |
|---|---|
| **客戶端** | React · TypeScript · Vite · `matter-js` 物理 · `lucide-react` |
| **原生外殼** | `@capacitor/core` · `@capacitor/ios` · `@capacitor/android` |
| **裝置 API** | `@capacitor/preferences` · `@capacitor/haptics` · `@capacitor/network` · `@capacitor/status-bar` · `@capacitor/app` |
| **認證** | `@capacitor-firebase/authentication` · `firebase` · 預設匿名登入 |
| **後端** | Firebase Firestore · Node cron 做排行榜聚合 |
| **金流** | `@revenuecat/purchases-capacitor` · `@capgo/capacitor-admob` |
| **設定安全** | `docs/debug/` 與 `docs/production/` 兩份 `google-services.json`，都受版控 |

## 3. 架構

```text
src/
├── App.tsx                    最上層做一次平台判斷，之後不再做
├── components/                EvolutionGame · UIOverlay · 7 個 modal · 教學層
├── services/
│   ├── storage/
│   │   ├── IStorageService.ts        介面
│   │   ├── LocalStorageService.ts    Web — 瀏覽器 localStorage
│   │   ├── NativeStorageService.ts   原生 — @capacitor/preferences
│   │   └── StorageManager.ts         singleton；只挑一次實作
│   ├── authService.ts         原生與 Web 的認證流程收在同一組 API 後面
│   ├── purchaseService.ts     RevenueCat
│   ├── cloudService.ts        Firestore 同步
│   └── leaderboard.ts         7 個反正規化查詢欄位
├── config/firebase.ts
└── types/user.ts
```

**五項能力因平台而異（儲存、認證、購買、廣告、雲端同步），而每一項都只有一道接縫。**

## 4. 做了哪些事

| 做法 | 用什麼做 | 解決什麼問題 |
|---|---|---|
| **儲存收在介面後面** | `IStorageService` + 兩份實作 + singleton manager | 平台判斷只出現一次；灑在元件裡等於在每個呼叫點重複同一個決定 |
| **反正規化的排行榜 schema** | 送出當下就寫入 7 個可查詢欄位 | Firestore 按文件讀取計費，所以問題是一個榜碰幾份文件，不是查詢快不快 |
| **排行榜聚合改版** | Node cron 預先聚合 | 讀取從 **N×50 降到 N×1** |
| **匿名優先的認證** | `signInAnonymously` | 不必註冊就能送分數，帳號可以之後再綁 |
| **用 RevenueCat 而非直接接商店 API** | 一次整合、兩個商店 | 權益狀態在重裝後仍存在，而且不需要我自己的伺服器 |
| **debug 與 production 設定分離** | 兩份受版控的 `google-services.json` | 「debug build 指向正式專案」是那種只會犯一次、然後用設計去防的錯 |
| **每個目標一個 build 指令** | 平台同步、設定注入、資產生成都串在一個指令後面 | 讓發行變成一個指令而不是一份檢查清單 |

## 5. 關鍵實作細節

<details>
<summary><b>降級住在介面後面，不住在遊戲裡</b></summary>

儲存是最清楚的例子。一個介面、兩份實作，加上一個負責挑選的 manager:

```ts
export interface IStorageService {
    initialize(): Promise<void>;
    saveUserData(data: UserData): Promise<void>;
    loadUserData(): Promise<UserData>;
    saveSettings(settings: any): Promise<void>;
    loadSettings(): Promise<any>;
    clearAllData(): Promise<void>;
}
```

```ts
private constructor() {
    if (Capacitor.isNativePlatform()) {
        this.localService = new NativeStorageService();   // @capacitor/preferences
    } else {
        this.localService = new LocalStorageService();    // 瀏覽器 localStorage
    }
}
```

**平台判斷只出現一次，在 manager 的建構子裡。** 它上面的所有東西呼叫 `saveUserData`，從來不問自己跑在什麼平台上。另一種做法，把 `if (Capacitor.isNativePlatform())` 灑在各個元件裡，等於在每個呼叫點重複同一個決定，而每一次重複都是一個「Web 路徑會被忘記」的地方。

同樣的形狀覆蓋五項因平台而異的能力：儲存、認證、購買、廣告與雲端同步。`services/` 底下是 `authService`、`cloudService`、`leaderboard`、`purchaseService` 與 storage 模組，每一個都是那道接縫。

> **誠實註記：** 五項都遵守服務邊界這條規則，但確實有一個平台判斷落在遊戲元件裡面。規則是設計；那個判斷是一個沒有遵守它的地方。

</details>

<details>
<summary><b>排行榜在寫入時就反正規化，讓讀取變便宜</b></summary>

每一筆送出的分數都帶七個事先選好的可查詢欄位，而服務裡逐一寫明每個欄位開啟了什麼查詢：

```text
1. themeId          → where('themeId', '==', 'ocean')
2. hasUsedItems     → where('hasUsedItems', '==', false)   // 無道具排行榜
3. maxLevelReached  → where('maxLevelReached', '>=', 9)
4. platform         → where('platform', '==', 'android')
5. timestamp        → where('timestamp', '>=', startOfDay) // 每日／每週榜
6. score            → where('score', '>=', 5000)
7. maxCombo         → where('maxCombo', '>=', 10)
```

**Firestore 是按文件讀取次數計費的**，所以成本問題不是「查詢快不快」，而是「它碰到幾份文件」。在送出當下就寫進這些欄位，意味著一個依主題、依平台或依時間窗的排行榜是一次索引查詢，而不是把整個 collection 抓下來再過濾。聚合改版把讀取從 **N×50 降到 N×1**。

這是一個後端成本決定改寫了產品功能的例子：**存在的那些排行榜，是寫入 schema 讓它們變便宜的那些。**

認證預設是匿名的（`signInAnonymously`）所以不必註冊就能送出分數，帳號可以之後再綁。

</details>

<details>
<summary><b>發行前的接線工作才是真正的內容</b></summary>

| 面向 | 套件 |
|---|---|
| 原生外殼，iOS + Android | `@capacitor/core`、`@capacitor/ios`、`@capacitor/android` |
| 認證 | `@capacitor-firebase/authentication`、`firebase` |
| 購買 | `@revenuecat/purchases-capacitor` |
| 廣告 | `@capgo/capacitor-admob` |
| 裝置 | `@capacitor/haptics`、`@capacitor/network`、`@capacitor/status-bar`、`@capacitor/app` |
| 儲存 | `@capacitor/preferences` |
| 遊戲 | `matter-js`（物理）、`react`、`react-dom`、`lucide-react` |

兩份 `google-services.json` 分別受版控在 `docs/debug/` 與 `docs/production/`，因為「debug build 指向正式 Firebase 專案」是那種你只會犯一次、然後開始用設計去防的錯。

用 RevenueCat 而不是直接接 StoreKit 與 Play Billing：一次整合、兩個商店，而且權益狀態在重裝後仍然存在，不需要我自己的伺服器。這個專案最後三個 commit 是把 IAP 識別碼分平台設定，那既是常見的順序，也是 §4 結束在那裡的原因。

</details>

## 6. 它後來怎麼了

183 個 commit,**其中 180 個落在 2025 年 12 月**，幾乎每天都動，一天 8 到 16 個。最後三個在 2026 年 1 月初，是金流。然後就沒有了。

它上架了，而且到現在還在架上。停下來的是開發，不是產品。**這裡的工程含量比這份作品集其他系統低；它有的是另一種東西：一個走完的發行週期，包含那些只會發生一次的部分**：商店審核、每個平台不同的 IAP 識別碼，以及一個活得比自己的 commit 歷史還久的上架紀錄。

它也是這份作品集裡唯一可以直接下載來用的東西：[RoundEvolution on the App Store](https://apps.apple.com/us/app/roundevolution/id6756927084)。

## 7. 限制

- **完全沒有測試。** 物理遊戲迴圈是確定性的、測得起來，而它沒有被測試。
- **有一個平台判斷落在遊戲元件裡**（降級那一段有註明）那是規則沒有被遵守，不是規則錯了。
- **沒有我自己的伺服器。** 狀態放在 Firebase 與 RevenueCat，所以成本與可用性的樣貌是他們的。
- **Web 端從未做過負載測試**，而它正是能力最少、出錯方式最多的那一個。
- **原始碼是私有的。** 商店頁面公開、build 可以下載，但它背後的程式碼不公開。

## 8. 參考

| 項目 | 內容 |
|---|---|
| **客戶端** | React、TypeScript、`matter-js` 物理、Vite |
| **原生外殼** | Capacitor 從一份程式碼出 iOS 與 Android;Web 是第三個目標，沒有原生外掛 |
| **後端** | Firebase（Firestore、匿名認證）、Node cron 做排行榜聚合 |
| **金流** | RevenueCat 內購、AdMob |
| **降級** | `IStorageService` 帶原生與瀏覽器兩份實作，在一個 singleton manager 裡挑選一次；認證、購買、廣告與雲端同步用同一道接縫 |
| **發行** | 每個目標一個 build 指令，平台同步、設定注入與資產生成都串在後面；debug 與 production 的 Firebase 設定分開受版控 |

**不在本文件內：** 遊戲設計與機制、各供應商的設定識別字與 bundle ID、商店上架文案，以及任何營收或安裝數字。

---

[作品集索引](../README.zh-TW.md)

---
