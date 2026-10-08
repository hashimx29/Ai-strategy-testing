## Prescreen Strategy

*(Info is also available in PRESCREEN.md)*

### 1. What prescreen IS, in one sentence

Prescreen is an extremely fast, in-memory stress test of a candidate strategy's raw entry concept (stripped of all entry filters, session rules, and volatility checks), run through your exact production pipeline (`execution_costs` + `position_manager`) and evaluated against a 100-run Monte Carlo random baseline, to instantly eliminate bad ideas before spending ~5 minutes setting up a full logged backtest.

**It is not a loose approximation or a cheap cost model.** It ALWAYS uses your full execution modules — including realistic spreads, slippage, commissions, position sizing, and daily trade limiters — ensuring zero bias and zero inflated results.

### 2. Why prescreen exists — the super-fast workflow

Writing, configuring, and executing a full backtest with custom entry filters, complex trade management rules, and trade-by-trade log generation takes around 5 minutes per strategy. Running full backtests on unproven concepts is a massive waste of time when most raw ideas lack a basic statistical edge.

Prescreen acts as an immediate high-speed filter:

```
+-----------------------------------------------------------------+
|                          RAW CONCEPT                             |
|     (Isolate raw trigger e.g., FVG + Sweep, zero entry filters)  |
+-----------------------------------------------------------------+
                            |
                            v
+-----------------------------------------------------------------+
|                     PRESCREEN EVALUATION                         |
|   • Standardized 5-bar / ATR exit engine                         |
|   • Full execution_costs & position_manager modules (In-Memory)  |
|   • 100-run Monte Carlo random entry comparison                  |
+-----------------------------------------------------------------+
                            |
              +-------------+-------------+
              |                           |
              v                           v
    [ z < -0.25: REJECT ]        [ z >= -0.25: PASS ]
    Instantly discard in          Advance to Full Backtest
    seconds (saves 5 mins)        (Build filters & run 5-min log)
```

By testing the raw concept alone, prescreen answers one question in seconds: **"Does this entry trigger possess any inherent statistical edge over random noise?"** If a raw signal cannot beat or closely match a random baseline under full market friction, adding filters later will not save it.

### 3. The modules involved — ALWAYS present in prescreen

Prescreen must always use your standard production modules to ensure results are never unrealistically optimistic or biased.

- `execution_costs` → simulates one trade's cost/fill under full adverse market friction
- `position_manager` → enforces daily trade limits, hard risk caps, and computes final stats

Both modules are executed in full, unmodified, during prescreen:

| Step | Function Called | What It Does | Same in Full Backtest? |
|---|---|---|---|
| 1 | `pm.request_signal_slot(pair_id, date)` | Per-pair-per-day trade limiter | Identical |
| 2 | `pm.calculate_position_size(entry, sl, requested_risk_pct)` | Hard-capped 1% equity risk sizing | Identical |
| 3 | `engine.execute_trade(...)` (execution_costs) | Adverse slippage, bid/ask spread, dual commissions, gap logic | Identical |
| 4 | `pm.log_trade_from_execution(...)` | Accumulates TradeRecord and updates equity in memory | Identical |
| 5 | `pm.finalize()` | Computes the overall PerformanceSummary | Identical (in-memory return) |

### 4. Raw Signal Isolation vs. Full Backtest

The key distinction between prescreening and full backtesting is what goes into the simulation and where the output goes:

```python
# PositionManager handling for prescreening vs backtesting
summary = compute_summary(self.config.starting_equity, self._trades)
if self.config.mode != "backtester":
    return summary          # <-- Prescreen path: Return memory summary instantly.
                             #     No file I/O, no console formatting.

# Full Backtest path (takes ~5 mins to construct & write):
self._log_path_written = log_writer.write_trade_log(...)
self._print_summary(summary, self._log_path_written)
return summary
```

| Dimension | Prescreen Mode (`mode="prescreener"`) | Full Backtest Mode (`mode="backtester"`) |
|---|---|---|
| Entry Logic | Raw Concept Only (no session filters, no volatility checks, no confluences) | Fully Filtered Strategy (all regime, time, and indicator filters active) |
| Exit Logic | Generic Benchmark Engine (fixed 5-bar hold or standard ATR stop/target) | Strategy-Specific Exits (trailing stops, structural targets, partial closes) |
| Execution Costs | FULL (`execution_costs` active with spread, slippage, fees) | FULL (`execution_costs` active with spread, slippage, fees) |
| Risk & Sizing | FULL (`position_manager` 1% hard cap active) | FULL (`position_manager` 1% hard cap active) |
| Output / I/O | Pure in-memory `PerformanceSummary` object (instant) | Full `.txt` trade-by-trade log written to disk (~5 min build time) |

### 5. Daily / Per-Pair Trade Gating (`position_manager`)

`pm.request_signal_slot(pair_id, date)` calls `DailyTradeLimiter.try_reserve((pair_id, date))`.

- **Strict per-pair isolation:** each pair receives an independent 1-trade-per-day limit.
- **Signal reservation:** slots are locked at the moment of signal generation to prevent over-trading on high-frequency days.
- Prescreen enforces this identical limit to ensure high-frequency raw signals do not obscure real-world execution constraints.

### 6. Real Execution Costs — NO Simplifications Allowed

Prescreen NEVER shortcuts trade costs. Every raw signal evaluated in prescreen passes directly through `execution_costs.ExecutionEngine`:

- **Slippage:** 2x standard adverse multiplier (5x during news events).
- **Spread:** Charged separately on top of slippage-adjusted prices.
- **Commission:** Deducted on both entry AND exit independently.
- **Fixed Costs:** Standard per-trade transaction fees applied at entry.
- **Gap-Through-Stop:** Fills executed at adverse bar open if price gaps past stop loss.
- **SL/TP Ambiguity:** Stop Loss always takes priority if both SL and TP fall within the same bar's high-low range.

**Crucial Rule:** If a raw concept cannot survive real execution costs during prescreen, it is fundamentally flawed. Never disable or lighten `execution_costs` during prescreen.

### 7. Position Sizing & Risk Management

Identical to full backtesting, `pm.calculate_position_size()` enforces fixed-fractional sizing with a strict upper bound:

$$\text{Risk Amount} = \text{Equity} \times \min(\text{Requested Risk}, 0.01)$$

Effective risk per trade is hard-capped at 1.0% (`HARD_MAX_RISK_PCT = 0.01`). Prescreen mode never relaxes, bypasses, or adjusts this boundary.

### 8. Statistical Benchmarking: Monte Carlo Null & Z-Score Gating

To determine if a raw signal holds genuine promise without relying on raw profitability alone, prescreen compares the strategy's raw performance against a 100-run Monte Carlo random-entry baseline using the exact same generic exit mechanics.

**The Z-Score Formula:**

$$z = \frac{\mu_{\text{strategy}} - \mu_{\text{random}}}{\sigma_{\text{random}}}$$

Where:
- $\mu_{\text{strategy}}$ = mean return of the raw strategy entries.
- $\mu_{\text{random}}$ = mean return across 100 random-entry Monte Carlo runs.
- $\sigma_{\text{random}}$ = standard deviation of the Monte Carlo random returns.

**The Prescreen Pass/Fail Decision Matrix**

Because raw concepts lack entry filters, their raw mean return will often be slightly negative due to natural market friction and spread. The z-score evaluates relative performance over pure noise:

| z-Score Result | Classification | Prescreen Outcome | Action |
|---|---|---|---|
| z ≥ 0.0 | Outperforms or matches random noise | PASS | Advance to full backtest design |
| -0.25 ≤ z < 0.0 | Slight negative edge vs. random (within tolerance) | PASS | Advance to full backtest design |
| z < -0.25 | Significantly worse than random noise | FAIL | Instantly discard concept |

### 9. Mistakes to Avoid

- **NEVER** run full strategy entry filters during prescreen. Prescreen is designed to test raw signal triggers (e.g., bare FVG or sweep) in seconds. Save filter optimization for full backtests.
- **NEVER** simplify or bypass `execution_costs` or `position_manager`. Removing spreads, slippage, or daily limits introduces positive bias and renders prescreen useless.
- **NEVER** add arbitrary "fudge factors" or allowance constants to the z-score. The formula z ≥ -0.25 naturally allows for raw concept friction without needing custom parameter shifts.
- **NEVER** skip prescreen and jump straight to full backtests. Spending 5 minutes setting up a full backtest for a concept with z < -0.25 wastes developer and compute time.
- **NEVER** assume a positive z-score guarantees net positive returns. A raw concept with z ≥ 0.0 simply proves that the signal contains structural edge over noise. Filters added in the full backtest phase refine that edge into net profitability.

### 10. Quick Reference: Running a High-Speed Prescreen Pass

```python
import numpy as np

# 1. Setup production PositionManager in prescreen mode
config = PositionManagerConfig(
    strategy_name="raw_fvg_sweep_concept",
    strategy_number=101,
    mode="prescreener"  # Memory-only execution, zero disk I/O
)
pm = PositionManager(config)

# 2. Execute Raw Concept Signals (NO Entry Filters Active)
for signal in raw_candidate_signals:
    pair_id = pm.derive_pair_id(signal.filename)
    if not pm.request_signal_slot(pair_id, signal.date):
        continue

    sizing = pm.calculate_position_size(signal.entry, signal.sl)

    # ALWAYS use full execution_costs engine for real market friction
    exec_result = engine.execute_trade(...)
    pm.log_trade_from_execution(pair_id, signal.date, signal.direction, exec_result, sizing)

raw_summary = pm.finalize()

# 3. Calculate Z-Score against 100-Run Monte Carlo Null Baseline
strat_mean = raw_summary.average_monthly_return_pct  # PerformanceSummary has no per-trade field; this is the real field name
rand_mean = np.mean(mc_random_returns)
rand_std = np.std(mc_random_returns, ddof=1)

z_score = (strat_mean - rand_mean) / rand_std if rand_std > 0 else 0.0

# 4. Enforce Prescreen Gate
Z_MIN_THRESHOLD = -0.25

if z_score >= Z_MIN_THRESHOLD:
    print(f"PASS [z={z_score:.2f}]: Proceeding to 5-minute Full Filtered Backtest.")
    # Proceed to build session/volatility filters & run mode="backtester"
else:
    print(f"FAIL [z={z_score:.2f}]: Raw concept rejected. Discarding immediately.")
```

---
