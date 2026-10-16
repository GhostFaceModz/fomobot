# GhostFace FOMOBot: Staged Plan (v2)

> PAPER TRADING ONLY right now. Not financial advice. No live trading code exists, and no wallet is connected.

## v1 post-mortem, 2026-10-16 (archived in trades_v1.db)
- v1 won only about 18% of trades. Its stops were checked every 15s, so rugs gapped straight through them (FTR closed at -99.9%).
- **Changes in v2:**
  - **Rug checks:** rejects mint or freeze authority still set (Solana RPC), top-10 holders above 25% (pool and burn accounts excluded), and liquidity down more than 15% over 1h.
  - **Stricter entries:** 5m volume at least 2x the hourly average, 5m buys/sells at least 1.3, and h1 change between +2% and +60%.
  - **Faster, tighter exits:** checked every 5s. Sell 1/3 at +10%, 1/3 at +20%, and trail the rest by 8%. Stop loss -7%, time stop at 45 minutes if P/L is within ±3%, max hold 3h. Sells everything if pool liquidity falls more than 15% below entry.

## Phase 0: Paper testing (current: v2, fresh $100 wallets x 8)
- 8 strategy variants, each starting with a virtual $100. They all trade on the same DexScreener/GeckoTerminal data.
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
- **Stop loss:** −7%, checked every 5s. **Time stop:** 45 minutes if P/L is flat (±3%). **Max hold:** 3h.
- **Scale-out:** 1/3 at +10%, 1/3 at +20%. Trail the rest by 8%, with the floor at breakeven after fees.
- **Rug filters:** mint and freeze authority must be revoked. Top 10 holders ≤ 25% (pool and burn accounts excluded). Liquidity must not have fallen more than 15% over 1h, and a liquidity pull after entry triggers an immediate exit.
- A stop is not a guarantee: a rug can gap through it within one 5s poll. Position size is the real protection.
- **Daily loss limit:** −5% of equity stops new entries until 00:00 UTC.
- **Kill switch:** a `KILL` file in the bot directory (or a drawdown > 20% from peak, or 3 API/tx failures in a row) stops all new entries and flattens positions. A human must restart the bot.
- **Hot wallet:** top it up manually and keep it small. Sweep profits out regularly. Keep keys out of the repo.

## Metrics
- **Per variant:** equity curve, closed trades, win rate, average win and average loss, expectancy, profit factor, max drawdown, average hold time, and exit-reason mix.
- **Signals:** entry signals are logged with each trade (in SQLite `trades.signals`). Use them to check which filters actually predict winners.

## How the evolution works (v2)
- Evolution runs every 6h. A variant is only judged once it has **at least 20 closed positions**.
- **Score:** expectancy after fees, meaning the mean P/L % per closed position with all partial sells combined. Equity is not used.
- If at least two variants qualify, the worst one is retired: its positions close at the last price. The best one is cloned with Gaussian mutations (σ ≈ 10%) on ~40% of its parameters, kept within safe bounds. The clone gets a fresh $100.
- If fewer than two qualify, nothing changes. This avoids selecting on noise.
- Before going live, still require 100+ trades with a positive bootstrap lower bound. A rank alone is not enough.
