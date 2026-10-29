# VaR and Expected Shortfall Calculator for Stocks

A Jupyter notebook that downloads historical price data from Yahoo Finance and calculates one-day Value at Risk (VaR) and Expected Shortfall (ES) for a stock using the historical simulation method.

## Table of Contents

- [Overview](#overview)
- [Background](#background)
- [Features](#features)
- [Requirements](#requirements)
- [Installation](#installation)
- [Usage](#usage)
- [How It Works](#how-it-works)
- [Interpreting the Results](#interpreting-the-results)
- [Customization](#customization)
- [Limitations](#limitations)
- [Possible Extensions](#possible-extensions)
- [Disclaimer](#disclaimer)

## Overview

This notebook answers a simple question: how much can a stock lose on a bad day? It pulls roughly 20 years of daily adjusted closing prices for a chosen ticker, converts them to daily log returns, and estimates the typical worst-case loss at a chosen confidence level.

## Background

**Value at Risk (VaR)** is the loss threshold that a position should not exceed on most days. A 95% one-day VaR of 2% means that on about 95% of trading days, losses should stay below 2%. The remaining 5% of days can be worse, and VaR says nothing about how much worse.

**Expected Shortfall (ES)**, also called Conditional VaR, fills that gap. It is the average loss on the days when losses exceed VaR. A 95% ES of 3% means that on the worst 5% of days, the average loss is 3%.

Regulators, including the Basel III/FRTB framework, have moved toward ES because it captures the severity of tail losses.

## Features

- Downloads adjusted price history from Yahoo Finance with no API key required
- Computes daily log returns
- Calculates VaR and ES at 95% and 99% confidence levels
- Plots the return distribution with VaR and ES marked
- Easy to switch to any ticker by changing one variable

## Requirements

- Python 3.9 or newer
- Jupyter Notebook or JupyterLab
- Internet connection (for Yahoo Finance)

Python packages:

- `yfinance`
- `pandas`
- `numpy`
- `scipy` (optional, only needed for parametric methods)
- `matplotlib`

## Installation

1. Clone or download this repository, or create a new notebook.
2. Install the dependencies:

```bash
pip install yfinance pandas numpy matplotlib
```

3. Launch Jupyter:

```bash
jupyter notebook
```

## Usage

The notebook is organized into five cells that run in order.

| Cell | Purpose |
|------|---------|
| 1 | Import libraries |
| 2 | Download price data from Yahoo Finance |
| 3 | Compute daily log returns and summary statistics |
| 4 | Calculate VaR and ES at 95% and 99% |
| 5 | Plot the return distribution with VaR and ES marked |

To analyze a different stock, change the `TICKER` variable in Cell 2 and run the notebook from Cell 2 onward. Use Yahoo Finance symbols, for example `MSFT`, `RELIANCE.NS` for NSE-listed stocks, or `^NSEI` for an index.

## How It Works

1. **Download.** `yfinance` retrieves daily prices between `START` and `END`. `auto_adjust=True` corrects for splits and dividends.
2. **Returns.** Daily log returns are computed as ln(P_t / P_{t-1}).
3. **VaR.** The VaR at confidence level c is the negative of the (1 − c) percentile of the returns.
4. **ES.** The ES is the negative of the average of all returns at or below the VaR cutoff.
5. **Plot.** A histogram of returns is drawn with vertical lines at −VaR and −ES.

## Interpreting the Results

Suppose the output shows a 95% one-day VaR of 2.5% and a 95% ES of 3.8%. This means:

- On roughly 95% of trading days, the stock should not lose more than 2.5%.
- On the worst 5% of days, the average loss is about 3.8%.

Results are reported as positive numbers representing losses. Multiply by the position size in currency to get a dollar figure, for example a ₹10,00,000 position with a 2.5% VaR has a one-day VaR of ₹25,000.

## Customization

- **Date range:** edit `START` and `END` in Cell 2. Shorter windows (for example the last 2 years) reflect current risk regimes.
- **Horizon:** for an N-day horizon, multiply the one-day figures by √N. This assumes returns are independent and identically distributed.
- **Confidence level:** add values to the list in Cell 4, such as 0.975.
- **Multiple stocks:** wrap Cells 2 to 4 in a loop over a list of tickers.

## Limitations

- **Historical method assumes the past is representative.** Extreme events not present in the sample will be missed.
- **Equal weighting of all days.** A 20-year window mixes calm and crisis periods. Recent volatility may differ substantially from the long-run average.
- **Square-root-of-time scaling** for multi-day horizons is an approximation.
- **Small tail sample at 99%.** With about 5,000 observations, the 99% ES rests on only around 50 data points, so it is noisy.
- **No backtesting** is included. Checking how often losses exceed VaR is recommended before relying on the numbers.

## Possible Extensions

- Parametric (normal) and Monte Carlo methods for comparison
- GARCH(1,1) with Student-t innovations to capture volatility clustering
- Rolling-window VaR to track how risk changes over time
- Backtesting with Kupiec's proportion-of-failures test
- Portfolio VaR using covariance across several stocks

## Disclaimer

This project is for educational purposes only and is not investment advice. Risk models depend on their assumptions, and historical data does not guarantee future results. Verify any figures before using them for trading or risk management decisions.
