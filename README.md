# Quantitative Research on MOEX

Personal research project: systematic search for alpha on the Moscow Exchange using publicly available data. The focus is on strategies that survive realistic transaction costs (Tinkoff Trader tariff: 0.05% equities, 0.04% futures) and can be implemented with a small starting capital.

**Data sources:** MOEX ISS API (free, no auth), MOEX AlgoPack (5-min microstructure data, local cache), Tinkoff Invest API (fundamentals, order execution).

**Infrastructure:** three live bots on a VPS  Gap Fade Si, Dividend Run-Up, and Momentum  collecting real paper-trading statistics since June 2026.

---

## Notebooks

**01_data_collection.** Setting up data pipelines: MOEX ISS candle API, Tinkoff SDK, caching layer for 100+ tickers. 

**02_strategy_research.** Exploratory anomaly search on Si, BR, RI futures: calendar effects, lead-lag relationships, mean-reversion, gap statistics. Calendar effects absent (ANOVA p > 0.98); hourly autocorrelation on Si = -0.44; identified overnight gap fade as the main hypothesis.

**03_walk_forward.** Walk-forward validation of Gap Fade Si (fade the morning gap > 0.5% relative to the previous evening close). OOS: 180 trades, win rate 87.8%, WF efficiency 124%; no overfitting detected; bot deployed to VPS.

**04_paper_trading.** Manual paper trading infrastructure via Tinkoff Sandbox and monitoring tools for the gap fade bot. Sandbox environment verified, trade logging established, transition criteria defined (20+ trades, WR > 70%).

**05_intraday_momentum.** Do stocks that rally in the first hour continue through the day? Tested on 100 MOEX stocks across multiple signal windows. No momentum effect (Pearson ~0.001, p > 0.5 in all configurations); discovered a weak but consistent -0.08%/day afternoon drift across the whole market.

**06_momentum_12_1.** Classic cross-sectional momentum (Jegadeesh-Titman 1993): buy 12-month winners, skip last month. The 12-month lookback is statistically significant (p < 0.04); shorter windows are not. Long-only has no alpha vs the benchmark; long-short with an MXI futures hedge gives Sharpe ~0.6.

**07_dividend_gap.** Pre-ex-date run-up: buy 20-60 days before the record date, exit on the last day with dividend rights. 245 events 2022-2026, average alpha +8.3% per holding period, t-stat 7.0, win rate 68%. Bot running live.

**08_short_term_reversal.** Weekly mean reversion: do last week's losers recover the following week? No reversal on MOEX - the effect runs in the opposite direction (weekly winners continue, Sharpe 0.52 for top-3). Extreme drops above 10% do show a reversal signal (Sharpe 1.09) but with only 39 observations over four years.

**09_buy_the_panic.** Market-wide mean reversion: buy after IMOEX drops more than 1.5-3%. After IMOEX falls below -3%, the market continues falling for 3-5 days (p = 0.004). Best backtest Sharpe 0.28 with 70% max drawdown. Strategy abandoned.

**10_value_momentum.** Combined factor model: rank stocks by momentum (12-1) and value (P/B from Tinkoff API). Value alone is weak on MOEX (sanctions discount distorts P/B multiples). A 50/50 blend slightly improves some configurations and is included as an optional parameter in the momentum bot.

**11_strategies_backtest.** Academic replication: five published strategies tested together with walk-forward validation and alpha tests vs IMOEX. Best new finding is Frog in the Pan (information discreteness filter, p = 0.006). Time-series momentum and demeaned momentum are both stable. GLM momentum fails due to insufficient data history.

**12_order_flow_imbalance.** Does order flow imbalance in the first 30 minutes predict the next 1-2 hours? AlgoPack data downloaded locally (48 tickers, 6.8M rows, 2023-2026). Signal design and IC framework built; full backtest is the next research step.

**13_giga_momentum.** Out-of-sample replication of GigaMomentum (GoAlgo 2023 winner): stochastic oscillator combined with order book imbalance rank, daily rebalancing. OOS Sharpe 1.03 (p = 0.043), bootstrap at 100th percentile confirming no overfitting. Fails at Tinkoff's 0.10% RT cost (Sharpe drops to 0.63, p = 0.21). Regime analysis shows the 2023 bull recovery drove most of the gross return.

**14_decline_reversal_eda.** EDA of individual stock behavior after sustained 10-day declines of -7% to -20%. Forward returns are statistically significant at -7% to -15% thresholds over 5-7 day horizons. OFI is the strongest predictor of subsequent reversals (p < 0.0001). Win rate is 55% after a decline exceeding 10%.

**15_reversal_backtest.** Full backtest of the reversal strategy calibrated in notebook 14: enter when OFI begins recovering after a sustained decline, exit after 5 days. Base configuration gives Sharpe -0.13. Best configuration (decline below -15%, realized vol 25-60%, N = 5 positions): IS Sharpe 1.85, OOS Sharpe 0.84 on only 29 trades (p = 0.53). WF efficiency 0.45.

**16_reversal_ml.** ML filter (Logistic Regression and LightGBM) to separate profitable from losing reversal setups. LGBM OOS AUC 0.569. LogReg top-50% filter raises OOS Sharpe to 1.27 (58 trades, p = 0.18) - promising but not conclusive at the current sample size.

**17_futures_alpha.** Three futures strategies tested: overnight gap fade on Si, BR, MX, and RI; MX vs RI pairs trading using the log-spread stationarity; Si calendar spread as a CBR rate expectations indicator. Gap fade IC is not significant on any instrument. MX/RI pairs trading gives Sharpe 1.16-1.19 at entry threshold z = 1.5-2.0 (p < 0.05, half-life 1.1 days). Si calendar spread currently implies a CBR rate of 15.1% vs the actual 14.5%.

---

## What is running live

Three paper-trading bots on a VPS , all using Tinkoff Invest API in sandbox mode:

- **Gap Fade Si** - trades the overnight gap on SiU6 when gap exceeds 0.4%; systemd service, email notifications
- **Dividend Run-Up** - daily scan for upcoming ex-dates, 15 positions open as of June 2026; systemd service
- **Momentum 12-1** - monthly rebalancing across 41 configurations including Frog in the Pan, inverse-vol weighting, and demeaned momentum; 
