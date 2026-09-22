# 事件驅動交易引擎

[← 作品集索引](../README.zh-TW.md) · [English version](02-trading-engine.md)

![Python](https://img.shields.io/badge/Python-3776AB?logo=python&logoColor=white) ![ccxt](https://img.shields.io/badge/ccxt-000000) ![pandas](https://img.shields.io/badge/pandas-150458?logo=pandas&logoColor=white) ![NumPy](https://img.shields.io/badge/NumPy-013243?logo=numpy&logoColor=white) ![paper trading only](https://img.shields.io/badge/paper%20trading%20only-yellow)

> 給加密永續合約用的回測與前推模擬引擎，建造目標是讓測試架構無法美化策略。每一個預設值都指向悲觀那一側，而結果是：它被造出來要測的東西，大部分沒有活過交易成本。

## 1. 專案綱要

| 項目 | 內容 |
|---|---|
| **類型** | 研究引擎 · 回測 + 紙上交易前推 runner |
| **角色** | 獨力完成 |
| **期間** | 2026-04-27 至 2026-09-01 |
| **規模** | 164 個 Python 模組 · 約 34,000 行 · 179 個 commit |
| **規格** | 72 份設計與研究文件，約 29,750 行 |
| **狀態** | 僅紙上交易 · **不下任何真實委託** |
| **原始碼** | 私有 · 面談階段可安排讀取權限 |

## 2. 技術棧

| 層 | 技術 |
|---|---|
| **語言** | Python |
| **市場資料** | `ccxt` ≥4.2 · `pyarrow` ≥15 parquet 快取 |
| **分析** | `pandas` ≥2.1 · `numpy` ≥1.26 · `scipy` ≥1.11 · `matplotlib` |
| **設定** | `PyYAML` · `python-dateutil` |
| **執行模型** | 次根開盤成交 · 逆向滑價 · 兩腿 taker 費率 · 影線雙穿時採「先打停損」的悲觀約定 |
| **成本模型** | 依流動性分級的滑價 + 每萬美元名目的線性市場衝擊 |
| **驗證** | 滾動 walk-forward，參數掃描限縮在訓練切片，凍結後才套上測試切片 |
| **前推模式** | 約 3,300 行紙上 runner，只取市場資料，維護虛擬投資組合 |

## 3. 架構

```text
crypto_strategy/
├── signals/      126 檔 — 偵測器
├── alpha/         10 檔 — 橫斷面因子（成交量 z-score、反轉、區間回歸、複合）
├── engine/         5 檔 — loop.py (991) · arbitrator.py (585) · regime_detector.py (222)
├── execution/      4 檔 — simulator.py (207) · exit_policy.py (542)
├── backtest/       4 檔 — walk_forward.py (232) · metrics.py (254) · runner.py (171)
├── risk/           4 檔 — 部位大小
├── data/           4 檔 — ccxt 擷取 + parquet 快取
└── reports/        2 檔
```

**164 個模組裡有 126 個是偵測器，其餘全部是阻止那些偵測器騙我的機制。** 那個比例就是這個專案。

## 4. 做了哪些事

| 做法 | 用什麼做 | 解決什麼問題 |
|---|---|---|
| **次根開盤成交** | 訊號在 T 根收盤 → T+1 開盤價加逆向滑價成交 | 用訊號那根的收盤價成交，等於假設一張在過去就下好的單 |
| **先打停損的悲觀約定** | K 棒內區間檢查，影線雙穿時假設先打停損 | 假設先打停利，會在最難重建的那些 K 棒上把虧損變成獲利 |
| **依流動性分級且帶線性衝擊的滑價** | rank ≤10 與 >10 兩級，`+ impact_per_10k × (notional / 10_000)` | 固定滑價常數是讓高換手策略看起來可行的最有效單一手段 |
| **凍結參數的 walk-forward** | 只在訓練切片掃描，凍結後套上未見過的測試切片，再串接 | 參數掃描正是產生過擬合的東西，所以它絕不能看到測試資料 |
| **同類別訊號收斂** | `max + 0.15 × (N-1)`，上限 `+0.5` | 高度相關的偵測器相加，等於把同一份證據重複計算、製造信心 |
| **市況硬性排除** | 多單訊號在 TRENDING_DOWN 永不觸發，不論整體強度 | 懲罰可以被足夠強的 ensemble 買掉，而這一條不應該被買掉 |
| **自帶出處的規則** | 程式碼註解引用 `signal_catalog.md` 條款編號與 `review.md` 決定 ID | 72 份規格文件只有在程式碼說得出它實作哪一條時才有用 |
| **v1/v2 並存** | Arbitrator 忽略沒有 v2 強度的訊號 | 對 126 個訊號模組來一次一刀切，那不是遷移，是停機 |

## 5. 關鍵實作細節

<details>
<summary><b>執行模擬器是回測說謊的地方</b></summary>

三個行為，每一個都選了悲觀的那邊：

```python
"""Backtest execution simulator.

1. **Entry (next-bar-open fill).** Bar T closes -> signal generated ->
   order placed -> filled at bar T+1's open price plus slippage in the
   adverse direction. This models the latency between your strategy seeing
   a bar close and your order actually hitting the exchange.

2. **Stop / TP intra-bar checks.** On each subsequent bar we ask: did this
   bar's range cross the stop or TP? If both crossed in the same bar
   (which happens when a candle wicks both sides), we conservatively
   assume the stop hit first — this is the pessimistic-fill convention.

3. **Fees & slippage.** Taker fee on both legs. Slippage tier depends on
   the symbol's liquidity rank (set when the simulator is constructed).
"""
```

**三個各自封住一種「數字免費變好看」的方式。** 用訊號那根的收盤價成交，等於假設一張在過去就下好的單。K 棒的影線同時穿過兩個價位時假設先打到停利，等於在最難重建的那些 K 棒上把虧損變成獲利。兩腿都算 maker 費率，等於假設市價單拿得到它拿不到的成交。

滑價不是常數。它依流動性分級，而且隨部位大小成長：

```python
def _slippage_for(self, symbol: str, notional: float) -> float:
    rank = self.ranks.get(symbol, 25)
    if rank <= 10:
        base, impact_per_10k = self.slip_top10, self.impact_top10
    else:
        base, impact_per_10k = self.slip_top50, self.impact_top50
    # Linear impact: notional / 10_000 → number of $10k units → impact bps.
    return base + impact_per_10k * (notional / 10_000.0)

def _apply_slippage(self, price, side, symbol, is_entry, notional):
    slip = self._slippage_for(symbol, notional)
    # Adverse direction: entry long -> pay up; entry short -> sell low;
    # exit long -> sell low; exit short -> buy high.
    if is_entry:
        return price * (1 + slip) if side is Side.LONG else price * (1 - slip)
    return price * (1 - slip) if side is Side.LONG else price * (1 + slip)
```

**一個固定的滑價常數，是讓高換手策略看起來可行的最有效單一手段**，因為交易成本不再隨著你交易多少而放大。每一萬美元名目金額的線性衝擊讓「做大」變貴，而沒有排名的標的預設 rank 25，也就是落進比較差的那一級而不是比較好的那一級。**預設值刻意指向悲觀那一側。**

</details>

<details>
<summary><b>Walk-forward 在參數看到測試資料之前就凍結它們</b></summary>

```python
"""Walk-Forward Analysis.

1. Split the full date range into rolling (train, test) window pairs.
2. For every window, run a parameter sweep over the train slice and pick
   the params with the best in-sample avg_R.
3. Apply those frozen params to the (unseen) test slice.
4. Concatenate all test slices — that is the "out-of-sample" track record.

If the OOS expectancy stays positive and the chosen params are reasonably
stable across windows, the edge generalizes. If OOS collapses to zero or
goes negative, the in-sample +0.020R was fit to the training noise.
"""
```

參數掃描正是產生過擬合的東西，所以掃描**只在訓練切片上跑**，而它選出的參數在測試切片被碰到之前就凍結。紀錄是所有測試切片的串接，不是任何單一窗口。

那段 docstring 用這個專案裡的一個真實數字，寫明它在監視什麼失效：一個樣本內 **+0.020R**，在樣本外崩掉，就代表它是被訓練雜訊擬合出來的。**把預期的失效模式寫進工具裡**，是讓一個負面結果讀起來像結果而不是像 bug 的原因。

</details>

<details>
<summary><b>72 份規格文件，以及自帶出處的規則</b></summary>

Confluence arbitrator 是 585 行，實作一份書面訊號目錄裡的一個編號小節，而程式碼裡每一條規則都引用了產生它的條款與審查決定：

```python
"""v2 Confluence Arbitrator — implementation of signal_catalog.md §3.5.

- §3.3 rule (a) — Hard-opposite drop: opposing-direction signals at strength
  ≥ 3.0 on the same bar drop both sides.
- §3.3 rule (b) — Same-category collapse: within one category, take max +
  0.15 × (N-1) capped at +0.5. Avoids double-counting near-correlated
  detectors (review.md P0 #3).
- §3.3 rule (c) — Cross-category boost: ≥3 different categories agreeing
  contribute +0.5 per extra category (cap +1.0).
- §3.3 rule (d) — Filter signals only have *negative* effect. Never positive.
- §3.5 step 6 — **HARD DROP on regime against** (review.md P0 #2): a long
  signal in TRENDING_DOWN regime never fires, regardless of ensemble strength.
"""
```

**規則 (b) 是統計上最要緊的那一條。** 同一個類別裡的數個偵測器彼此高度相關，把它們的強度相加等於把同一份證據重複計算，製造出來的是信心而不是資訊。收斂成「取最大值加一個有上限的加成」，就是 ensemble 與回音的差別。

**訊號目錄 §3.5 step 6 是硬性排除而不是權重**，因為懲罰可以被足夠強的整體強度買掉，而這一條不應該被買掉。

Arbitrator 也會忽略任何沒有 v2 強度的訊號，這讓 v1 與 v2 兩套訊號在遷移期並存，而不需要對 126 個訊號模組來一次一刀切。

</details>

<details>
<summary><b>佈局</b></summary>

`crypto_strategy/` 是 164 個模組，而它的分布就是對這個專案最誠實的描述：

| 套件 | 檔數 | 是什麼 |
|---|---|---|
| `signals/` | **126** | 偵測器 |
| `alpha/` | 10 | 橫斷面因子（成交量 z-score、報酬反轉、區間回歸、複合） |
| `engine/` | 5 | 991 行的主迴圈、585 行的 arbitrator、市況偵測 |
| `execution/` | 4 | 模擬器、出場策略（542 行） |
| `backtest/` | 4 | walk-forward、指標、runner |
| `risk`、`data`、`reports` | 12 | 部位大小、ccxt 擷取與 parquet 快取、輸出 |

**164 個模組裡有 126 個是偵測器，其餘全部是阻止那些偵測器騙我的機制。** 那個比例就是這個專案。

資料經 `ccxt` 取得，用 `pyarrow` 快取成 parquet，所以重跑不會重新下載，而同一個窗口的兩次執行比較的是同一批 K 棒。

</details>

## 6. 它實際上得出什麼結論

從初始 commit 開始，這個 repo 自己的實驗記錄：

| 日期 | 記下的結論 |
|---|---|
| 2026-04-27 | 把標的池擴到 33 個 symbol **傷害**了 edge(`exp-019`) |
| 2026-04-30 | 對存活組態做逐月 walk-forward:39 個月裡 27 個月為正，p < 0.01 |
| 2026-05 | 訊號品質 ML 與波動度 regime 閘，當成研究工具做：**結論全為負** |
| 2026-06 | dollar bars 與 volume profile 均值回歸：兩者皆 **KILL**，成本牆沒被打破 |

接著是兩個月沒有 commit，而讓專案重啟的是把同一套引擎指向另一個市場，不是一個更好的想法。

**引擎能用；它被造出來要測的東西，大部分沒有活過交易成本。** 那正是一個誠實的成本模型會產出的東西，也是執行模擬器被放在第一個講的原因。

## 7. 限制

- **它不下真單。** 前推 runner 只為市場資料連線，對收盤的即時 K 棒跑生產用的 ensemble，並維護一個虛擬投資組合。每一個訊號、委託、成交與權益變動都寫入磁碟記錄。**不下任何真實委託。**
- **這個引擎裡沒有 deflated Sharpe 或 PBO。** walk-forward 在這裡，但能對 best-of-N 結果做扣減的多重比較校正不在，所以一個存活下來的組態是「沒有明顯被擬合」而不是「已證明可泛化」。
- **那個 0.020R 是樣本內數字。** 它在本文件裡只以「walk-forward 存在是為了抓什麼」的例子出現。
- **流動性排名是靜態的**，在模擬器建構時設定，所以一個在回測期間內深度改變過的標的會被算在錯誤的級距。
- **最後一個 commit 是 2026-09-01。** 這個引擎已經沒有在擴充了。

## 8. 參考

| 項目 | 內容 |
|---|---|
| **執行環境** | Python,`ccxt` ≥4.2 · `pandas` ≥2.1 · `numpy` ≥1.26 · `pyarrow` ≥15（parquet 快取）· `scipy` ≥1.11 · `PyYAML` · `matplotlib` |
| **執行模型** | 次根開盤成交、逆向滑價、兩腿 taker 費率、影線雙穿時採「先打停損」的悲觀約定、依流動性分級的滑價並帶每萬美元名目的線性衝擊 |
| **驗證** | 滾動 walk-forward，參數掃描限縮在訓練切片、凍結後才套上測試切片；紀錄是串接起來的測試切片 |
| **訊號層** | 126 個偵測器模組，v2 訊號帶 catalog ID、1–5 強度、factors、zone 與 regime fit |
| **仲裁** | 來自書面目錄的六條編號規則：反向硬排除、同類別收斂、跨類別加成、filter 只能有負向效果、強度修飾、市況硬性排除 |
| **前推模式** | 約 3,300 行的紙上交易 runner，帶虛擬投資組合，只取市場資料 |
| **規格** | 72 份文件、約 29,750 行，程式碼引用條款編號與審查決定 |

**不在本文件內：** 策略本身、偵測器邏輯、門檻與調校常數。

**取得方式。** 實作是私有的。面談階段可以安排讀取權限。

---

[作品集索引](../README.zh-TW.md)

---
