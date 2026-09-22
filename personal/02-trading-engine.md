# Event-Driven Trading Engine

[← Portfolio index](../README.md) · [繁體中文版](02-trading-engine.zh-TW.md)

![Python](https://img.shields.io/badge/Python-3776AB?logo=python&logoColor=white) ![ccxt](https://img.shields.io/badge/ccxt-000000) ![pandas](https://img.shields.io/badge/pandas-150458?logo=pandas&logoColor=white) ![NumPy](https://img.shields.io/badge/NumPy-013243?logo=numpy&logoColor=white) ![paper trading only](https://img.shields.io/badge/paper%20trading%20only-yellow)

> A backtest and forward-simulation engine for crypto perpetual futures, built so the test harness cannot flatter the strategy. Every default errs on the pessimistic side, and the result is that most of what it was built to test did not survive transaction costs.

## 1. At a glance

| Item | Detail |
|---|---|
| **Type** | Research engine · backtest + paper-trading forward runner |
| **Role** | Sole author |
| **Period** | 2026-04-27 to 2026-09-01 |
| **Size** | 164 Python modules · ~34,000 LOC · 179 commits |
| **Specification** | 72 design and research documents, ~29,750 lines |
| **State** | Paper trading only · **no real orders are placed** |
| **Source** | Private · read access can be arranged for hiring conversations |

## 2. Tech stack

| Layer | Technology |
|---|---|
| **Language** | Python |
| **Market data** | `ccxt` ≥4.2 · `pyarrow` ≥15 parquet cache |
| **Analysis** | `pandas` ≥2.1 · `numpy` ≥1.26 · `scipy` ≥1.11 · `matplotlib` |
| **Config** | `PyYAML` · `python-dateutil` |
| **Execution model** | Next-bar-open fill · adverse slippage · taker fees both legs · pessimistic stop-first on wick-through bars |
| **Cost model** | Liquidity-tiered slippage + linear market impact per $10k notional |
| **Validation** | Rolling walk-forward, parameter sweep confined to the train slice, frozen params on test |
| **Forward mode** | ~3,300-line paper runner, market data only, virtual portfolio |

## 3. Architecture

```text
crypto_strategy/
├── signals/      126 files — the detectors
├── alpha/         10 files — cross-sectional factors (volume z-score, reversal, range revert, composite)
├── engine/         5 files — loop.py (991) · arbitrator.py (585) · regime_detector.py (222)
├── execution/      4 files — simulator.py (207) · exit_policy.py (542)
├── backtest/       4 files — walk_forward.py (232) · metrics.py (254) · runner.py (171)
├── risk/           4 files — position sizing
├── data/           4 files — ccxt fetch + parquet cache
└── reports/        2 files
```

**126 of 164 modules are detectors; everything else is the machinery that stops those detectors from fooling me.** That ratio is the project.

## 4. What was built

| Mechanism | Built with | Problem it solves |
|---|---|---|
| **Next-bar-open fill** | Signal on bar T close → fill at T+1 open + adverse slippage | Filling at the signal bar's close assumes an order placed in the past |
| **Pessimistic stop-first convention** | Intra-bar range check, stop assumed first on wick-through | Assuming the target hit first converts losses into wins on exactly the hardest bars to reconstruct |
| **Liquidity-tiered slippage with linear impact** | Rank ≤10 vs >10 tiers, `+ impact_per_10k × (notional / 10_000)` | A flat slippage constant is the single most effective way to make a high-turnover strategy look viable |
| **Walk-forward with frozen parameters** | Sweep on train slice only, apply frozen to unseen test, concatenate test slices | The sweep is what produces overfitting, so it must never see the test data |
| **Same-category signal collapse** | `max + 0.15 × (N-1)` capped at `+0.5` | Near-correlated detectors summed together count the same evidence repeatedly and manufacture confidence |
| **Hard regime drop** | Long signal in TRENDING_DOWN never fires, regardless of ensemble strength | A penalty is negotiable against a strong enough ensemble; this one should not be |
| **Rules citing their own provenance** | Code comments cite `signal_catalog.md` clause numbers and `review.md` decision IDs | 72 specification documents are only useful if the code says which clause it implements |
| **v1/v2 coexistence** | Arbitrator ignores signals without a v2 strength | Migrating 126 signal modules on a flag day is not a migration, it is an outage |

## 5. Key implementation details

<details>
<summary><b>The execution simulator is where a backtest lies</b></summary>

Three behaviours, all of them the pessimistic choice:

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

**Each of the three closes a specific way the numbers improve for free.** Filling at the signal bar's close assumes an order placed in the past. Assuming the target was hit first when a wick crossed both levels converts a loss into a win on exactly the bars that are hardest to reconstruct. Charging maker fees on both legs assumes fills that a market order does not get.

Slippage is not a constant. It is tiered by liquidity and grows with size:

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

**A flat slippage constant is the single most effective way to make a high-turnover strategy look viable**, because the cost of trading stops scaling with how much you trade. Linear impact per $10k of notional makes size expensive, and an unranked symbol defaults to rank 25, which puts it in the worse tier rather than the better one. Defaults err on the pessimistic side on purpose.

</details>

<details>
<summary><b>Walk-forward freezes parameters before they see test data</b></summary>

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

The sweep is what produces overfitting, so the sweep runs **only on the train slice**, and the parameters it picks are frozen before the test slice is touched. The concatenation of test slices, not any single window, is the record.

The docstring names the failure it is watching for with a real number from this project: an in-sample **+0.020R** that collapses out of sample was fitted to training noise. Writing the expected failure mode into the tool is what makes a negative result readable as a result rather than as a bug.

</details>

<details>
<summary><b>72 specification documents, and rules that carry their own provenance</b></summary>

The confluence arbitrator is 585 lines implementing a numbered section of a written signal catalogue, and every rule in the code cites the clause and the review decision that produced it:

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

**Rule (b) is the one that matters statistically.** Several detectors in one category are near-correlated, so summing their strengths counts the same evidence repeatedly and manufactures confidence. Collapsing to the max plus a capped bonus is the difference between an ensemble and an echo.

**The catalogue’s §3.5 step 6 is a hard drop rather than a weighting** because a penalty is negotiable against a strong enough ensemble, and this one should not be.

The arbitrator also ignores any signal without a v2 strength, which let the v1 and v2 signal sets coexist during migration instead of requiring a flag day across 126 signal modules.

</details>

<details>
<summary><b>Layout</b></summary>

`crypto_strategy/` is 164 modules. The distribution is the honest description of the project:

| Package | Files | What it is |
|---|---|---|
| `signals/` | **126** | The detectors |
| `alpha/` | 10 | Cross-sectional factors (volume z-score, returns reversal, range revert, composite) |
| `engine/` | 5 | The 991-line loop, the 585-line arbitrator, regime detection |
| `execution/` | 4 | Simulator, exit policy (542 lines) |
| `backtest/` | 4 | Walk-forward, metrics, runner |
| `risk`, `data`, `reports` | 12 | Sizing, ccxt fetch with parquet cache, output |

**126 of 164 modules are detectors and everything else is the machinery that stops those detectors from fooling me.** That ratio is the project.

Data is fetched through `ccxt` and cached as parquet via `pyarrow`, so a re-run does not re-download and two runs over the same window are comparing the same bars.

</details>

## 6. What it actually concluded

The repository's own experiment log, from the initial commit onward:

| Date | Logged conclusion |
|---|---|
| 2026-04-27 | Expanding the universe to 33 symbols **hurt** the edge (`exp-019`) |
| 2026-04-30 | Monthly walk-forward on the surviving configuration: 27 of 39 months positive, p < 0.01 |
| 2026-05 | Signal-quality ML and a volatility-regime gate, built as research tools: **all conclusions negative** |
| 2026-06 | Dollar bars and volume-profile mean reversion: both **KILL**, the cost wall held |

Then two months with no commits, and what restarted the project was pointing the same engine at a different market rather than a better idea.

**The engine works; most of what it was built to test did not survive transaction costs.** That is what an honest cost model produces, and it is the reason the execution simulator is the first thing described.

## 7. Limitations

- **It does not place real orders.** The forward runner connects for market data only, runs the production ensemble against live bars as they close, and maintains a virtual portfolio. Every signal, order, fill and equity tick is recorded to disk. **No real orders are placed.**
- **No deflated Sharpe or PBO in this engine.** Walk-forward is here; the multiple-comparison correction that would deflate a best-of-N result is not, so a surviving configuration is "not obviously fitted" rather than "shown to generalise".
- **The 0.020R figure is in-sample.** It appears in this document only as the example of what walk-forward exists to catch.
- **Liquidity ranks are static**, set when the simulator is constructed, so a symbol whose depth changed over the backtest window is charged the wrong tier.
- **Last commit 2026-09-01.** The engine is not being extended.

## 8. Reference

| Item | Detail |
|---|---|
| **Runtime** | Python, `ccxt` ≥4.2 · `pandas` ≥2.1 · `numpy` ≥1.26 · `pyarrow` ≥15 (parquet cache) · `scipy` ≥1.11 · `PyYAML` · `matplotlib` |
| **Execution model** | Next-bar-open fill, adverse slippage, taker fees both legs, pessimistic stop-first convention on wick-through bars, liquidity-tiered slippage with linear impact per $10k notional |
| **Validation** | Rolling walk-forward with parameter sweep confined to the train slice and frozen params on test; concatenated test slices as the record |
| **Signal layer** | 126 detector modules, v2 signals carrying a catalogue ID, 1–5 strength, factors, zone and regime fit |
| **Arbitration** | Six numbered rules from a written catalogue: hard-opposite drop, same-category collapse, cross-category boost, negative-only filters, strength modifiers, hard regime drop |
| **Forward mode** | ~3,300-line paper-trading runner with a virtual portfolio, market data only |
| **Specification** | 72 documents, ~29,750 lines, with code citing clause numbers and review decisions |

**Not in this document:** the strategies themselves, detector logic, thresholds and tuning constants.

**Availability.** The implementation is private. Read access can be arranged for hiring conversations.

---

[Portfolio index](../README.md)

---
