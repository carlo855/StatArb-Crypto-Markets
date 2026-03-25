# Statistical Arbitrage in Cryptocurrencies

A quantitative trading research notebook that builds, optimizes, and evaluates a market-neutral statistical arbitrage strategy across the cryptocurrency universe. The strategy combines mean reversion and momentum signals with portfolio-level risk management, tested on out-of-sample data with frozen parameters.

## Strategy Overview

The system trades ~200 USDT-paired tokens on Binance at 12-hour frequency, combining two uncorrelated alpha signals into a single portfolio with volatility targeting and drawdown management.

### Mean Reversion

Captures short-term reversals using a multi-lookback ensemble (1, 2, 4, 8 periods). Signals are generated when a coin's return exceeds an adaptive rolling percentile threshold, then decay linearly over a configurable holding period to reduce turnover. Key design choices:

- **Adaptive thresholds** — rolling percentile of absolute returns (rather than a fixed cutoff) so the signal adjusts to changing volatility regimes
- **BTC regime filter** — positions are fully zeroed when BTC's short-term SMA is below its long-term SMA, avoiding mean-reversion trades during broad market crashes
- **Inverse BTC vol scaling** — exposure scales up in calm markets and down in volatile ones, the opposite of naive vol-scaling (which would chase noise)
- **Per-coin position caps** — 10% max gross per coin to prevent concentration

Parameters were selected via grid search over percentile threshold, holding period, and lookback combinations.

### Momentum

Z-score cross-sectional momentum based on the divergence between short- and long-window rolling mean returns, normalized by rolling standard deviation. Positions are sized by `tanh(z-score)` for smooth, bounded weights and filtered by a rolling volatility screen.

- **Winsorized z-scores** — clipped at ±3 before `tanh` to limit outlier influence
- **Volatility filter** — coins whose rolling vol exceeds a threshold multiple of their longer-term average vol are excluded, reducing exposure during blow-ups
- **Execution lag** — configurable (default 1 bar vs. original 2) to test sensitivity to fill assumptions

Parameters were selected via Bayesian optimization (200 iterations, Gaussian process with Expected Improvement), with a drawdown penalty in the objective to discourage fragile parameter regions.

### Portfolio Construction

The two strategies are combined using expanding-window optimal weights with covariance shrinkage (30% shrinkage toward diagonal). Weights are clamped at ±80% to prevent extreme tilts. Several combination methods are compared (fixed mean-variance, equal-volatility, Sharpe-ratio weighted, expanding optimal) and the best in-sample method is selected.

Risk overlays applied to the combined portfolio:

- **Volatility targeting** — EWMA(60) realized vol is scaled to a 15% annualized target
- **Drawdown deleveraging** — exposure is cut by 50% when the portfolio enters a drawdown exceeding 10%

## Data

- **Source**: Binance US via `python-binance`
- **Universe**: All USDT-paired tokens, filtered to the top 80% by median daily volume (removes illiquid tails)
- **Frequency**: 12-hour bars
- **Training period**: June 2021 – December 2024
- **Test period**: January 2025 – June 2026
- **BTC excluded** from the tradeable universe but used as a regime/vol signal

## Out-of-Sample Methodology

All parameters (signal lookbacks, thresholds, momentum windows, portfolio weights) are frozen after in-sample optimization. The test set uses the same liquid coin universe identified during training. Vol targeting and drawdown management use only data available at each point in time (no future information).

## Known Limitations & Open Questions

This is a research notebook, not a production system. Several issues remain:

- **Mean reversion OOS Sharpe is implausibly high (~6).** This likely reflects survivorship bias in the coin universe — tokens that delisted or crashed to zero during the test period may be excluded from the Binance API response, inflating returns for the mean-reversion strategy that buys dips. A more rigorous backtest would need a point-in-time universe with dead coins included.
- **Momentum degrades significantly out of sample (Sharpe ~ -2).** The momentum signal was heavily optimized in-sample (200 Bayesian iterations across 4 parameters), making it the most overfit component. The negative OOS Sharpe suggests the optimized parameters captured noise rather than structure.
- **Transaction costs are fixed at 20 bps.** No sensitivity analysis across cost assumptions (e.g., 10/30/50 bps) is included, though this would be straightforward to add.
- **No statistical significance testing on alpha.** The OLS regression against BTC is computed in-sample but alpha t-statistics and confidence intervals are not reported. For a strategy with this much parameter optimization, a bootstrap or block-bootstrap significance test would be more appropriate than standard OLS inference.
- **Single cost model.** The 20 bps flat fee ignores market impact, slippage, and the fact that taker/maker fees vary across exchanges and volume tiers.

## Dependencies

```
numpy
pandas
python-binance
matplotlib
statsmodels
scikit-optimize
```

## Usage

The notebook pulls live data from Binance on each run. You'll need a Binance US account (no API key required for public market data via `python-binance`). Run cells sequentially — later sections depend on objects created earlier (parameter grids, optimized weights, etc.).

```bash
pip install numpy pandas python-binance matplotlib statsmodels scikit-optimize
jupyter notebook Statistical_Arbitrage_in_Cryptocurrencies.ipynb
```
