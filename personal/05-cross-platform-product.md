# Cross-Platform Mobile Product

[← Portfolio index](../README.md) · [繁體中文版](05-cross-platform-product.zh-TW.md)

![React](https://img.shields.io/badge/React-61DAFB?logo=react&logoColor=white) ![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?logo=typescript&logoColor=white) ![Capacitor](https://img.shields.io/badge/Capacitor-119EFF?logo=capacitor&logoColor=white) ![Firebase](https://img.shields.io/badge/Firebase-FFCA28?logo=firebase&logoColor=white) ![App Store: live](https://img.shields.io/badge/App%20Store-live-0D96F6?logo=appstore&logoColor=white)

> A React and Capacitor physics game shipped solo to the iOS App Store with in-app purchases, advertising, auth and a Firebase backend. One codebase, three targets, and the web target has no native capabilities, so degradation happens at a service boundary rather than in the game.

## 1. At a glance

| Item | Detail |
|---|---|
| **Type** | Commercial mobile product · iOS + Android + web from one codebase |
| **Role** | Sole author, including store submission |
| **Period** | 2025-12-04 to 2026-01-03 · **180 of 183 commits inside December 2025** |
| **Size** | 36 TypeScript modules · ~9,700 LOC across client and server |
| **Purpose** | Work through a complete commercial release cycle end to end |
| **State** | Shipped to the App Store, **still listed** ([App Store](https://apps.apple.com/us/app/roundevolution/id6756927084)) |
| **Source** | Private · [landing page](https://ha516.com/RoundEvolution/index.html) |

## 2. Tech stack

| Layer | Technology |
|---|---|
| **Client** | React · TypeScript · Vite · `matter-js` physics · `lucide-react` |
| **Native shell** | `@capacitor/core` · `@capacitor/ios` · `@capacitor/android` |
| **Device APIs** | `@capacitor/preferences` · `@capacitor/haptics` · `@capacitor/network` · `@capacitor/status-bar` · `@capacitor/app` |
| **Auth** | `@capacitor-firebase/authentication` · `firebase` · anonymous sign-in by default |
| **Backend** | Firebase Firestore · Node cron for leaderboard aggregation |
| **Monetisation** | `@revenuecat/purchases-capacitor` · `@capgo/capacitor-admob` |
| **Config safety** | Separate `google-services.json` under `docs/debug/` and `docs/production/`, both version-controlled |

## 3. Architecture

```text
src/
├── App.tsx                    one platform check at the top, then never again
├── components/                EvolutionGame · UIOverlay · 7 modals · tutorial overlays
├── services/
│   ├── storage/
│   │   ├── IStorageService.ts        the interface
│   │   ├── LocalStorageService.ts    web — browser localStorage
│   │   ├── NativeStorageService.ts   native — @capacitor/preferences
│   │   └── StorageManager.ts         singleton; picks the implementation once
│   ├── authService.ts         native vs web auth flows behind one API
│   ├── purchaseService.ts     RevenueCat
│   ├── cloudService.ts        Firestore sync
│   └── leaderboard.ts         7 denormalised query fields
├── config/firebase.ts
└── types/user.ts
```

**Five capabilities differ by target (storage, auth, purchases, ads, cloud sync), and each has exactly one seam.**

## 4. What was built

| Mechanism | Built with | Problem it solves |
|---|---|---|
| **Storage behind an interface** | `IStorageService` + two implementations + singleton manager | The platform test appears once; sprinkling `isNativePlatform()` through components repeats the same decision at every call site |
| **Denormalised leaderboard schema** | 7 queryable fields written at submission time | Firestore bills per document read, so the question is how many documents a board touches, not how fast the query is |
| **Leaderboard aggregation redesign** | Node cron pre-aggregation | Reads went from **N×50 to N×1** |
| **Anonymous-first auth** | `signInAnonymously` | A score can be submitted without an account, and an account attached later |
| **RevenueCat over raw store APIs** | One integration, two stores | Entitlement state survives a reinstall without a server of my own |
| **Split debug/production Firebase configs** | Two version-controlled `google-services.json` | A debug build pointing at the production project is a mistake you make once, then design against |
| **Per-target build command** | Platform sync, config injection and asset generation chained behind one command | A release becomes a command rather than a checklist |

## 5. Key implementation details

<details>
<summary><b>Degradation lives behind an interface, not in the game</b></summary>

Storage is the clearest case. There is one interface, two implementations, and a manager that picks:

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
        this.localService = new LocalStorageService();    // browser localStorage
    }
}
```

**The platform test appears once, in the manager's constructor.** Everything above it calls `saveUserData` and never asks what it is running on. The alternative, `if (Capacitor.isNativePlatform())` sprinkled through the components, is the same decision repeated at every call site, and every repetition is a place where the web path can be forgotten.

The same shape covers the five capabilities that differ by target: storage, authentication, purchases, advertising and cloud sync. `services/` holds `authService`, `cloudService`, `leaderboard`, `purchaseService` and the storage module, each of them the seam.

> **Honest note:** all five follow the service-boundary rule, but one platform check did land inside the game components. The rule is the design; that check is a place that did not follow it.

</details>

<details>
<summary><b>The leaderboard is denormalised on write so reads are cheap</b></summary>

Every submitted score carries seven queryable fields chosen in advance, documented in the service with the query each one enables:

```text
1. themeId          → where('themeId', '==', 'ocean')
2. hasUsedItems     → where('hasUsedItems', '==', false)   // no-item leaderboard
3. maxLevelReached  → where('maxLevelReached', '>=', 9)
4. platform         → where('platform', '==', 'android')
5. timestamp        → where('timestamp', '>=', startOfDay) // daily / weekly boards
6. score            → where('score', '>=', 5000)
7. maxCombo         → where('maxCombo', '>=', 10)
```

**Firestore bills per document read**, so the cost question is not "is the query fast" but "how many documents does it touch". Writing these fields at submission time means a themed, platform-scoped or time-windowed board is one indexed query rather than a fetch-and-filter over the collection. The aggregation redesign took reads from **N×50 to N×1**.

That is a backend cost decision changing a product feature: the boards that exist are the ones the write schema made cheap.

Auth is anonymous by default (`signInAnonymously`) so a score can be submitted without an account, and an account can be attached later.

</details>

<details>
<summary><b>Release plumbing is the actual content</b></summary>

| Concern | Package |
|---|---|
| Native shell, iOS + Android | `@capacitor/core`, `@capacitor/ios`, `@capacitor/android` |
| Auth | `@capacitor-firebase/authentication`, `firebase` |
| Purchases | `@revenuecat/purchases-capacitor` |
| Advertising | `@capgo/capacitor-admob` |
| Device | `@capacitor/haptics`, `@capacitor/network`, `@capacitor/status-bar`, `@capacitor/app` |
| Storage | `@capacitor/preferences` |
| Game | `matter-js` (physics), `react`, `react-dom`, `lucide-react` |

Two `google-services.json` files are version-controlled under `docs/debug/` and `docs/production/`, because a debug build pointing at the production Firebase project is a mistake you make exactly once and then design against.

RevenueCat rather than raw StoreKit and Play Billing: one integration, two stores, and entitlement state that survives a reinstall without a server of my own. The last three commits of the project are the IAP identifiers being separated per platform, which is both the usual order and the reason §4 ends where it does.

</details>

## 6. What happened to it

183 commits, **180 of them inside December 2025**, near-daily at 8 to 16 a day. The final three, in early January 2026, are monetisation. Then nothing.

It shipped, and it is still on the store. Development stopped; the product did not. **The engineering content here is smaller than in the other systems in this portfolio; what it has instead is a finished release cycle, including the parts that only happen once**: store review, per-platform IAP identifiers, and a listing that outlived its own commit history.

It is also the only thing in this portfolio you can download and use: [RoundEvolution on the App Store](https://apps.apple.com/us/app/roundevolution/id6756927084).

## 7. Limitations

- **No tests at all.** The physics game loop is deterministic enough to test and was not tested.
- **One platform check inside the game components** (noted in the degradation section), which is the rule not being followed rather than the rule being wrong.
- **No server of my own.** Firebase and RevenueCat hold the state, so the cost and availability profile is theirs.
- **The web target was never load-tested**, and it is the one with the fewest capabilities and the most ways to be wrong.
- **The source is private.** The store listing is public and the build is downloadable, but the code behind it is not.

## 8. Reference

| Item | Detail |
|---|---|
| **Client** | React, TypeScript, `matter-js` physics, Vite |
| **Native shell** | Capacitor for iOS and Android from one codebase; web as a third target with no native plugins |
| **Backend** | Firebase (Firestore, anonymous auth), Node cron for leaderboard aggregation |
| **Monetisation** | RevenueCat in-app purchases, AdMob |
| **Degradation** | `IStorageService` with native and browser implementations chosen once in a singleton manager; the same seam for auth, purchases, ads and cloud sync |
| **Release** | Per-target build command with platform sync, configuration injection and asset generation chained behind it; separate debug and production Firebase configs under version control |

**Not in this document:** game design and mechanics, vendor configuration identifiers and bundle IDs, store listing copy, and any revenue or install figures.

---

[Portfolio index](../README.md)

---
