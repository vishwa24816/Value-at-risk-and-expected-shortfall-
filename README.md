# Stock Risk Engine: VaR & Expected Shortfall (Historical, Variance-Covariance, Monte Carlo)

A 10-cell Jupyter notebook that downloads daily (1d) prices for a stock and estimates **1-day Value at Risk (VaR)** and **Expected Shortfall (ES / CVaR)** using three methods, then **backtests** them to pick the best one.

| Method | Idea | Distribution assumption |
|---|---|---|
| Historical Simulation | Take the empirical percentile of past returns | None (non-parametric) |
| Variance-Covariance (Parametric) | Use mean and std dev with a normal quantile | Normal |
| Monte Carlo | Simulate many random returns from fitted parameters | Normal (swappable for Student-t) |

---

## 1. Quick Start

```bash
pip install numpy pandas yfinance matplotlib scipy jupyter
jupyter notebook
```

Create a new notebook and paste each code block into its own cell (Cell 1 to Cell 10), then run top to bottom.

**Requirements:** Python 3.9+, internet access (for Yahoo Finance).

---

## 2. Configuration

All settings live in **Cell 2**:

| Parameter | Default | Meaning |
|---|---|---|
| `TICKER` | `"RELIANCE.NS"` | Yahoo Finance symbol (`.NS` = NSE, `.BO` = BSE, no suffix = US) |
| `START`, `END` | `2018-01-01`, `2026-11-01` | Price history window |
| `CL` | `0.99` | Confidence level (use `0.95` for 95% VaR) |
| `WINDOW` | `250` | Rolling look-back (about 1 trading year) used in the backtest |
| `N_SIMS` | `100_000` | Monte Carlo paths for the full-sample estimate |

---

## 3. Cell-by-Cell Walkthrough

| Cell | Purpose | Output |
|---|---|---|
| 1 | Imports | None |
| 2 | Set parameters, download adjusted daily closes | Last 5 prices |
| 3 | Compute daily log returns, print summary stats, skew and excess kurtosis | Descriptive stats |
| 4 | Historical VaR and ES | `h_var`, `h_es` |
| 5 | Variance-covariance VaR and ES | `p_var`, `p_es` |
| 6 | Monte Carlo VaR and ES | `m_var`, `m_es` |
| 7 | Rolling backtest: each day's VaR uses only the previous `WINDOW` days | DataFrame `bt` |
| 8 | Kupiec POF test per method | DataFrame `results` |
| 9 | Select best method, print final VaR/ES table | `best`, `final` |
| 10 | Plot returns vs. VaR lines and return histogram | Charts |

---

## 4. Methodology

### 4.1 Returns
Daily log returns: `r_t = ln(P_t / P_{t-1})`, using dividend/split-adjusted closes (`auto_adjust=True`).

### 4.2 Historical VaR / ES
- **VaR** = negative of the `(1 - CL)` percentile of returns.
- **ES** = average loss on days when the return is at or below `-VaR`.
- No distributional assumption; captures fat tails, but only as much as the sample contains.

### 4.3 Variance-Covariance VaR / ES
With sample mean `μ`, sample std dev `σ`, and `z = Φ⁻¹(1 - CL)`:

```
VaR = -(μ + z·σ)
ES  = -(μ - σ·φ(z) / (1 - CL))
```

Fast and closed-form, but assumes normality, so it usually **understates tail risk** for equities.

> For a single stock the "covariance" reduces to plain variance. For a portfolio you would replace `σ` with `sqrt(wᵀΣw)`, where `Σ` is the covariance matrix of asset returns and `w` the weights.

### 4.4 Monte Carlo VaR / ES
Draw `N_SIMS` one-day returns from `Normal(μ, σ)` (seeded for reproducibility), then take the percentile (VaR) and tail mean (ES), as in the historical method but on simulated data.

With a normal draw this converges to the variance-covariance answer; its value comes from swapping in fatter-tailed distributions (see Section 7).

### 4.5 Backtest
For each day `t` after the first `WINDOW` days, VaR is computed from the prior `WINDOW` returns only (no look-ahead). A **breach** occurs when the realized return is worse than `-VaR`. At a 99% CL, you expect roughly 1% breaches.

### 4.6 Kupiec Proportion-of-Failures (POF) Test
Tests whether the observed breach rate is statistically consistent with `1 - CL`:

- Null hypothesis: breach probability equals `1 - CL`.
- Likelihood-ratio statistic follows `χ²(1)`.
- **p-value > 0.05** means the model is not rejected.

### 4.7 "Best" Method Rule
1. Keep methods with Kupiec p-value > 0.05.
2. Among those, pick the one whose breach rate is closest to `1 - CL`.
3. If none pass, fall back to the smallest gap across all methods.

---

## 5. Interpreting Results

- **VaR of 2.5% at 99%:** on 99% of days you expect to lose less than 2.5%. On the worst 1% of days, you expect to lose at least that.
- **ES** is always at least VaR. It answers "when VaR is breached, how bad is the average loss?"
- **Too many breaches** means the model underestimates risk (typical of normal-based methods in fat-tailed markets).
- **Too few breaches** means the model is overly conservative (capital is tied up unnecessarily).
- Compare `final` (full-sample numbers) to `results` (backtest quality); the best-calibrated model is not always the one with the largest VaR.

To convert to rupee or dollar terms: `VaR_amount = VaR_pct × position_value`.

---

## 6. Output Objects

| Variable | Type | Contents |
|---|---|---|
| `prices` | Series | Adjusted daily closes |
| `returns` | Series | Daily log returns |
| `bt` | DataFrame | Daily realized return and rolling VaR per method |
| `results` | DataFrame | Breaches, breach rate, Kupiec p-value, gap |
| `final` | DataFrame | Full-sample VaR and ES per method |
| `best` | str | Name of the selected method |

---

## 7. Extensions

**Fat-tailed Monte Carlo (Student-t):** in Cell 6, replace the normal draw with:

```python
params = stats.t.fit(r)
sims = stats.t.rvs(*params, size=n, random_state=seed)
```

**Other ideas**
- **Multi-day VaR:** scale by `√h` (an approximation valid under i.i.d. returns) or simulate `h`-day paths.
- **EWMA / GARCH volatility** for volatility clustering (`arch` package).
- **Filtered Historical Simulation:** standardize returns by GARCH volatility, then resample.
- **Portfolio VaR:** download multiple tickers, build the covariance matrix, and use correlated multivariate draws (Cholesky).
- **ES backtesting** (Acerbi-Szekely) to validate ES, not just VaR.
- **Christoffersen test** to check breach independence (clustering), complementing Kupiec.

---

## 8. Limitations

- **ES is not backtested** here; only VaR is validated via Kupiec.
- **Kupiec checks frequency, not clustering.** A model can pass while breaches bunch together in crises.
- **Stationarity assumption:** past returns are assumed representative of the near future.
- **Window sensitivity:** `WINDOW = 250` gives only about 2-3 tail observations at 99%, so rolling historical VaR is noisy. Consider 500+ days or 95% CL.
- **Log vs. simple returns:** VaR is expressed on log returns; the difference is negligible for 1-day horizons.
- **Data quality:** Yahoo Finance data can have gaps, bad ticks, or corporate-action errors. Verify before relying on results.
- **Illiquidity, gaps, and event risk** (earnings, circuit limits) are not modeled.

---

## 9. Troubleshooting

| Issue | Fix |
|---|---|
| Empty DataFrame from `yf.download` | Check ticker suffix (`.NS` / `.BO`), dates, and internet connection |
| `prices` is a DataFrame, not a Series | Newer `yfinance` returns multi-index columns; `.squeeze()` handles one ticker, so for multiple tickers select the column explicitly |
| Cell 7 is slow | Reduce `n` in the rolling Monte Carlo call (e.g. 5,000) or shorten the date range |
| Kupiec p-value is `NaN` or breaches are 0 | Sample too short for the chosen CL; use a longer history or lower CL |
| Different results on each run | Monte Carlo uses a fixed seed (`42`); changing it will change estimates slightly |

---

## 10. Disclaimer

This notebook is for education and research. It is not financial advice, and VaR/ES are model estimates that can fail in extreme markets. Validate any risk model before using it for real capital or regulatory purposes.
