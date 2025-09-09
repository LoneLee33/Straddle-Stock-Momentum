# Straddle-Stock-Momentum — Stock-Momentum Baseline for Option Momentum Research

## Overview

The repo conducts a **cross-sectional momentum backtest** using the typical **12-1 stock-return signal**. The code is a **clean reference** and **handy scaffold** for the eventual **straddle-return momentum** on **delta-neutral straddles**.

**Monthly—on third Friday-aligned calendars**, we estimate each ticker's **12–1 momentum** from monthly returns and **disregard the most recent month**. We categorize tickers into **quantiles** or pick **top-N / bottom-N**, build **equal- or value-weighted portfolios**, **leave in place for one month**, and **validate out of sample**. We keep track of **long–short P&L**, **hit rate**, **turnover**, and **drawdowns**.

**Modular notebooks.** **Daily prices add up to months.** **Signals** can be **winosored** and **universe-filtered**. **Portfolio construction** is its own step, so you can **readily switch in a new signal**. **there is a companion notebook** that reuses the same calendar and portfolio template at the **options level**, checking **stop-loss** and **profit-taking** rules for **straddles** using **daily mid prices**.

---

## Repository layout

```
Straddle-Stock-Momentum/
├─ data/
│  ├─ Top20_2023_with_OLHC/           # option OHLC + quotes by ticker (CSV per ticker)
│  ├─ stock_data_25_v2.csv            # daily stock panel used to build monthly returns
│  └─ (derived) All_Options_With_OHLC.csv
├─ notebooks/
│  ├─ stock_momentum.ipynb            # end‑to‑end stock‑momentum baseline
│  └─ stop_loss_profit_booking.ipynb  # straddle daily P&L + stop/profit logic
└─ README.md
```

---

## Environment

```bash
python -m venv .venv
source .venv/bin/activate
pip install pandas numpy matplotlib
# (add jupyter and nbformat if you want to run notebooks interactively)
```

---

## Data inputs

### 1) Daily stocks panel (data/stock\_data\_25\_v2.csv)

Required columns (case‑sensitive):

* date (datetime), ticker, permno
* adj\_ret (daily total return), adj\_prc (adj close)

### 2) Option OHLC + quotes (data/Top20\_2023\_with\_OLHC/\*\_Options\_With\_OHLC.csv)

Used for the straddle extension and the stop‑loss/profit‑booking notebook. Key columns used in the code:

* Contract identifiers: ticker, secid (or sid), cp\_flag, exdate or expirationDate, adjStrikePrice
* Quotes and greeks: adj\_best\_bid, adj\_best\_offer, delta, plus OHLC fields

Note: The loader concatenates all \*\_Options\_With\_OHLC.csv into All\_Options\_With\_OHLC.csv. There is light ticker normalization in the notebooks (for example, MARA1→MARA, AVGO1→AVGO, IPOE→SOFI). If a name is missing in the stock panel (for example, NIO noted in comments), it will be absent from the portfolio.

---

## Evaluation calendar (Third Fridays)

We align all month buckets to third‑Friday boundaries (standard option expiry convention). In stock\_momentum.ipynb:

* A helper constructs a list or Series of third Fridays (with holiday adjustments when needed).
* Each daily stock return is mapped to the next third Friday via np.searchsorted, so a month’s return is the product of (1 + adj\_ret) from one third‑Friday to the next.

This yields a panel of monthly stock returns:

```
[ticker, date, monthly_return]
```

---

## Signal: 12‑1 stock momentum

We implement the standard 12‑1 lookback:

* For each (ticker, month\_t) we collect the prior 11 months t−12 … t−2 (skip t−1).
* Compute the mean of available monthly returns in that window.
* Require at least 8 months present; otherwise set momentum to NaN.

Notebook functions you’ll see:

* LBwin12 — dict mapping (ticker, month\_t) → list of 11 prior (ticker, month) keys
* returnlookup(rt, monthly\_stock\_returns) — fetches a prior month’s return
* lbwinr(monthly\_stock\_returns) — returns both the list of prior‑window returns and the MOM dict

The resulting dataframe:

```
MOMs_df: [ticker, date, MOM]
```
---

## Portfolio formation

Two equivalent approaches are illustrated; pick one for your analysis.

### A) Quantile sort (Q = 3, 5, 10)

1. Within each month, rank by MOM and cut into Q buckets (for example, deciles).
2. Compute next‑month equal‑weight returns for each bucket.
3. Long‑short = top − bottom.
4. Summary metrics: average monthly return, volatility, Sharpe, t‑stat, cumulative return.

### B) Top‑N / Bottom‑N (robust to missing next‑month data)

The notebook provides helpers:

```python
get_sorted_tickers(month_df, reverse=True)  # highest first
pick_available_tickers(ticker_list, this_month, next_month, returns_df, n=3)
```

This ensures each pick has returns in both the current and next month before inclusion. Compute long‑short as the average next‑month return of the top‑N minus bottom‑N.

The notebook shows performance tables for Q = 3 and Q = 10 (see LSReturns\_Q3 and LSReturns\_Q10) with Sharpe and t‑statistics.

---

## How to run (notebooks)

1. Open notebooks/stock\_momentum.ipynb and run top‑to‑bottom:

   * Concatenate option files (optional, for later straddle work)
   * Load stock\_data\_25\_v2.csv
   * Build third‑Friday calendar → aggregate monthly stock returns
   * Construct 12‑1 momentum (lbwinr) → MOMs\_df
   * Form portfolios (quantiles or top‑N/bottom‑N) → long‑short series
   * Compute Sharpe and t‑stat and plot cumulative return

2. Open notebooks/stop\_loss\_profit\_booking.ipynb (optional, straddle add‑on):

   * Create daily mid quotes: mid = (adj\_best\_bid + adj\_best\_offer) / 2
   * Pivot calls and puts by contract and merge with delta‑neutral weights
   * Build daily straddle value paths (entry → exit)
   * Apply profit‑booking and stop‑loss triggers (see below)

---

## Profit booking and stop‑loss (straddles)

The second notebook demonstrates rule‑based exits for delta‑neutral straddles. Key ideas:

* Entry: on an evaluation date, buy a straddle using mid prices and delta‑neutral weights w\_C, w\_P.
* Daily value: straddle\_value\_t = w\_C \* mid\_call\_t + w\_P \* mid\_put\_t.
* Profit‑booking trigger (example): exit the first day combined value ≥ entry \* (1 + profit\_pct), for profit\_pct in {0.5, 0.8, 1.0}.
* Stop‑loss trigger (example): exit the first day combined value ≤ entry \* (1 − loss\_pct).
* Fallback: if neither trigger hits, hold to expiry, value equals the weighted sum of intrinsic values at expiration.

Notebook utility (example):

```python
profit_booking_50 = profit_booking_trigger(LS_straddles_daily, 0.50)
profit_booking_80 = profit_booking_trigger(LS_straddles_daily, 0.80)
profit_booking_100 = profit_booking_trigger(LS_straddles_daily, 1.00)
```

The outputs include ticker, entry\_date, exit\_date, exit\_value, and a stop\_type label (for example, none, profit, loss, both).

Once you build a monthly panel of straddle returns, you can replace the stock‑return momentum with straddle‑return momentum by feeding those returns into the same 12‑1 window code (LBwin12 → lbwinr). Everything else (sorting, holding, evaluation) remains identical.

---

## Outputs (examples)

* Stock\_MOM\_\*.csv — momentum values by ticker × month
* long\_short\_returns.csv — monthly long‑short series (if you export it)
* perf\_summary.csv — mean, volatility, Sharpe, t‑stat (see final notebook cells)
* cum\_return.png — cumulative long‑short performance plot

---

## Configuration knobs

* Quantiles or N picks: Q in {3, 5, 10} or n for top/bottom selection (default n = 3 in helper)
* Lookback window: fixed 12‑1; require ≥ 8 valid months
* Date map: third‑Friday schedule (editable in the helper)
* (Straddles) delta filter for calls in examples: 0.25 ≤ delta ≤ 0.75

---

## Reproducibility

* Sort by ticker, date before rolling operations
* Fix random seeds
* Sanity‑check that this\_month and next\_month returns exist for all selected names

---

