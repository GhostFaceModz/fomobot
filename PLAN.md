# GhostFace FOMOBot: Staged Plan

> PAPER TRADING ONLY right now. Not financial advice. No live trading code exists, and no wallet is connected.

## Phase 0: Paper testing (current)
- 5 strategy variants, each starting with a virtual $100. They all trade on the same DexScreener/GeckoTerminal data.
- Fees plus slippage are modelled as 1.5% per side, so a round trip costs about 3%.
- **To move to Phase 1, at least one variant must have all of these:**
  - **100+ closed trades** (exits), spread over at least 7 days and different market conditions.
  - **Positive expectancy after fees:** mean P/L per trade > 0. A 95% bootstrap lower bound near or above 0 is preferred.
  - **Profit factor > 1.3.** Max drawdown < 25% of equity.
  - **Not carried by one trade:** the result stays positive with the best trade removed.
  - **Parameters stable:** the variant (or its clones) stays near the top across 2+ evolution cycles.
- A forward-test also counts: freeze the winning parameter set as a non-evolving variant and require it to stay positive for another 50 trades.

## Phase 1: Tiny live size (not built yet)
- **Execution:** Jupiter Swap API (quote, then swap), signed by a **dedicated hot wallet** that holds only the risk budget. Never use a main wallet, and never connect one.
- **Budget:** e.g. $50–100 total. $2–5 per trade, max 2 open positions.
- **Run paper in parallel** on the same signals and compare live fills with paper fills to measure real slippage and fail rates.
- **Real-world checks:** set slippageBps limits, use priority fees, and check for honeypots/freeze authority (e.g. a mint/freeze-authority check). Simulate the sell route before buying.
- **Exit condition:** after 50 live trades, live expectancy must be at least 50% of paper expectancy. Otherwise go back to Phase 0.

## Phase 2: Scale
- Increase size in steps (x2 at a time) only after each step shows 50+ trades with positive expectancy.
- **Cap position size** at ≤1% of pool liquidity to limit price impact.
- Re-check slippage on every step. Edge in thin memecoin books shrinks quickly as size grows.

## Risk rules (all phases)
- **Per-trade size:** ~6% of equity in paper, clamped to $5–7. In live trading, ≤5% of the hot-wallet balance.
- **Max open positions:** 3 (paper), 2 (early live).
- **Stop loss:** about −10%. **Max hold:** 6h. **Trailing stop:** after the half take-profit, with a floor at entry.
- **Daily loss limit:** −5% of equity stops new entries until 00:00 UTC.
- **Kill switch:** a `KILL` file in the bot directory (or a drawdown > 20% from peak, or 3 API/tx failures in a row) stops all new entries and flattens positions. A human must restart the bot.
- **Hot wallet:** top it up manually and keep it small. Sweep profits out regularly. Keep keys out of the repo.

## Metrics
- **Per variant:** equity curve, closed trades, win rate, average win and average loss, expectancy, profit factor, max drawdown, average hold time, and exit-reason mix.
- **Signals:** entry signals are logged with each trade (in SQLite `trades.signals`). Use them to check which filters actually predict winners.

## How the evolution works
- Every 4h the bot ranks the live variants by equity.
- If the gap is meaningful (≥ $0.50) and the best variant has at least 2 exits, the worst variant is retired: its positions are closed at the last seen price. The best variant is cloned with Gaussian mutations (σ ≈ 10%) on a random half of its parameters, kept within safe bounds, and the clone gets a fresh $100.
- Otherwise only the worst variant's parameters are mutated.
- **Caveat:** ranking by equity on few trades is noisy. Before going live, judge by expectancy over 100+ trades, not by rank.
