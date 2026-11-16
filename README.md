# Engineering Portfolio: Systems & Architecture

**Liu Che Wei** · Taoyuan, Taiwan
[LinkedIn](https://www.linkedin.com/in/chewei-liu-429a85174/) · [github.com/hawiliu](https://github.com/hawiliu) · `hawiliu@gmail.com`

![C#](https://img.shields.io/badge/C%23-239120?logo=csharp&logoColor=white) ![Python](https://img.shields.io/badge/Python-3776AB?logo=python&logoColor=white) ![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?logo=typescript&logoColor=white) ![.NET](https://img.shields.io/badge/.NET-512BD4?logo=dotnet&logoColor=white) ![FastAPI](https://img.shields.io/badge/FastAPI-009688?logo=fastapi&logoColor=white) ![Vue](https://img.shields.io/badge/Vue-4FC08D?logo=vuedotjs&logoColor=white) ![PostgreSQL](https://img.shields.io/badge/PostgreSQL-4169E1?logo=postgresql&logoColor=white) ![Docker](https://img.shields.io/badge/Docker-2496ED?logo=docker&logoColor=white) ![systems](https://img.shields.io/badge/systems-6-informational)

Six systems, built on my own time and infrastructure. Each entry below is a one-minute summary; the full write-up behind it is a fifteen-minute engineering document. **No source code**, except the four public repositories in P6.

[繁體中文版](README.zh-TW.md)

## Index

| # | System | Scale | State |
|---|---|---|---|
| **P1** | Autonomous research platform | 61 modules · 30.8k LOC · 612 commits | Running, 111-day streak |
| **P2** | Event-driven trading engine | 164 modules · 34k LOC · 72 specs | Paper trading only |
| **P3** | Quant research platform | 130 modules · 26.8k LOC · 84 commits | Deployed |
| **P4** | Multi-role TCP game server | 202 C# files · 40.7k LOC · 12 projects | Dispatch complete, resolution stubbed |
| **P5** | Cross-platform mobile product | 36 modules · 9.7k LOC · 183 commits | Shipped, live on the App Store |
| **P6** | C# desktop tools | 4 public repos · 3.7k LOC | Public on GitHub |

## Systems

### P1 · Autonomous Research Platform

**61 Python modules · ~30,800 LOC · 612 commits over 111 days · running unattended**

A long-running, unsupervised research pipeline: it generates hypotheses, spends a hard daily quota of external evaluations on the highest-ranked candidates, and submits what clears a validation gate. The search layer is a bandit (UCB and Thompson), an MCTS/UCT family ledger that backs up the maximum, and a genetic algorithm. The funnel is ordered by cost: local hard gates first, then bandit ranking, and only then the quota-consuming simulations. Five LLM providers sit behind a single module with cross-process circuit breakers and an explicit degradation contract. Two SQLite stores hold results and the research record, and a FastAPI and Vue dashboard reads the same databases.

`Python` · `SQLite` · `FastAPI` · `Vue` · bandits, MCTS/UCT, genetic algorithms · cross-process circuit breakers
[Full write-up →](personal/01-alpha-research-platform.md)

### P2 · Event-Driven Trading Engine

**164 Python modules · ~34,000 LOC · 72 design documents (~29,750 lines) · paper trading only**

A backtest and forward-simulation engine for perpetual futures, paper trading only. The signal layer is pure functions over OHLCV behind an `Executor` interface the engine never bypasses, so backtest and live share one path. A bar-by-bar event loop, an execution simulator, walk-forward validation, and a slippage model tiered by liquidity. Of 164 modules, 126 are detectors; the rest are position sizing, regime detection, the signal arbitrator and the backtest machinery. The written specification runs to 72 documents and about 29,750 lines, roughly the size of the implementation.

`Python` · `ccxt` · `parquet` · walk-forward validation
[Full write-up →](personal/02-trading-engine.md)

### P3 · Quant Research Platform

**87 Python + 43 TypeScript modules · ~26,800 LOC · 84 commits · deployed behind a reverse proxy**

A full-stack quant research platform in FastAPI and Next.js, deployed behind an authenticated reverse proxy. TimescaleDB ingests market data, a React Flow node graph edits strategies, backtests run vectorised, and APScheduler drives a cycle every 15 minutes. The research side carries an LLM factor-mining loop whose model-generated code runs in an AST-restricted sandbox, plus a negative-control endpoint that feeds random signals through the identical backtest and gate. Signal computation is collapsed into one path, held there by a bar-by-bar equality test.

`FastAPI` · `Next.js` · `TimescaleDB` · `Redis` · `LightGBM` · `React Flow`
[Full write-up →](personal/03-quant-platform.md)

### P4 · Multi-Role TCP Game Server (.NET 10)

**202 C# files · ~40,700 LOC across 12 projects · dispatch complete, resolution stubbed**

A multi-role network server on the .NET 10 Generic Host: a twelve-project solution with an acyclic dependency graph. Three `BackgroundService` roles share one DI container and one message layer, so 39 typed message definitions are written once rather than duplicated. The transport runs on a raw `Socket` with hand-written length-prefixed framing and stream reassembly, and big-endian reads are confined to a `BinaryReader` subclass. 28 handlers register by attribute scan against a 1,240-line dispatch map. The server itself is 6,351 lines, 16% of the solution; the tooling is 57%.

`C#` · `.NET` · generic host · attribute-based dispatch · `ConcurrentDictionary`
[Full write-up →](personal/04-server-emulator.md)

### P5 · Cross-Platform Mobile Product

**36 TypeScript modules · ~9,700 LOC · 183 commits · shipped to the App Store, still listed**

A physics game in React and Capacitor, one codebase to iOS, Android and web, taken solo through store submission. Five capabilities that differ by target (storage, auth, purchases, ads, cloud sync) each sit behind one service interface, so the platform check happens once. The backend is Firebase Firestore with a Node cron job aggregating the leaderboard, which is denormalised on write. Monetisation goes through RevenueCat for both stores and AdMob for advertising. It is still listed on the App Store: [RoundEvolution](https://apps.apple.com/us/app/roundevolution/id6756927084).

`React` · `Capacitor` · `matter-js` · `Firebase` · `Node`
[Full write-up →](personal/05-cross-platform-product.md)

### P6 · C# Desktop Tools

**4 public repositories · ~3,700 LOC · 50 commits · 2019 to 2026**

Four public repositories spanning 2019 to 2026. CryptoWidget is a cross-platform price and P&L widget in Avalonia on .NET 8, with MVVM layering, ccxt for exchange access and three-language localisation, under MIT. TWjpRunner recovers a running process's command line through WMI and relaunches it elevated. imageZoom is a batch image resizer, moved from .NET Framework to .NET 10 in 2026. aiueo is a kana drilling program from 2019.

*These four repositories are the only readable code in this portfolio. Everything else asks to be taken on trust; these do not.*

[CryptoWidget](https://github.com/hawiliu/CryptoWidget) · [imageZoom](https://github.com/hawiliu/imageZoom) · [aiueo](https://github.com/hawiliu/aiueo) · [TWjpRunner](https://github.com/hawiliu/TWjpRunner)
[Full write-up →](personal/06-csharp-desktop-tools.md)

## Technology index

Searchable, so you can see whether I have touched a thing.

| Area | Where |
|---|---|
| **Languages** | **C# / .NET** (P4, P6) · Python (P1–P3) · TypeScript (P3, P5) · SQL (P1–P3) |
| **LLM & ML** | Multi-provider orchestration, circuit breakers, structured output (P1) · model-generated code in a sandbox (P3) · scikit-learn, LightGBM, random forest (P3) |
| **Search & statistics** | Multi-armed bandits, MCTS/UCT with max-backpropagation, genetic algorithms (P1) · walk-forward validation (P2) · negative control with turnover matching, t-statistic gating (P3) |
| **Backend & APIs** | FastAPI (P1, P3) |
| **Real-time** | Event-driven simulation loops (P2) · WebSocket market data (P3) |
| **Frontend & desktop** | Vue 3 (P1) · Next.js, React Flow (P3) · React, Capacitor (P5) · **Avalonia, MVVM** (P6) · Windows Forms (P4, P6) |
| **Data** | **TimescaleDB** (P3) · SQLite incl. WAL (P1, P2) · Redis (P3) · parquet caching (P2) · document database (P5) |
| **Infrastructure** | Docker and Compose (P3) · authenticated reverse proxy (P3) · APScheduler (P3) |
| **Server & connection handling** | Attribute-registered handler dispatch (P4) · 39 typed message definitions (P4) · accept loop, per-connection buffers, async throughout (P4) · in-process operator console (P4) · WMI process inspection (P6) |
| **Release & operations** | App Store submission, IAP per platform (P5) · MIT-licensed public release with three-language i18n (P6) |

## What is not here

Work done for my employer is not described here; it is covered in my CV and can be discussed in interviews. Five of the six systems remain private, one of them because it is commercially live; read access can be arranged for hiring conversations.

**The exception is P6.** Those four repositories are public and can be read now. They are also the smallest systems here.
