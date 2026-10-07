# Market Bars: Types, Statistical Properties, and How to Handle Their Problems

A complete guide to time bars, tick bars, volume bars, dollar bars, imbalance/run bars, and price-based bars. For each one you get: how it is formed, what is good, what is bad, how it affects **serial correlation, heteroskedasticity, non-normality and fat tails**, when to use it, and how to manage its issues.

Much of the framework comes from Marcos López de Prado, *Advances in Financial Machine Learning* (2018), Chapter 2, and from the older "market activity clock" literature (Clark 1973; Ané & Geman 2000).

> **Honest note:** Everything below about which bar is "better" describes **general tendencies** seen in research, mostly on liquid instruments such as futures and large stocks. It is not a guarantee. Always test on your own data (code for this is in Part 5).

---

## Table of Contents

1. [Core concepts](#part-1-core-concepts)
2. [Why the choice of bar matters](#part-2-why-the-choice-of-bar-matters)
3. [Each bar type in detail](#part-3-each-bar-type-in-detail)
4. [Comparison tables](#part-4-comparison-tables)
5. [Code: build bars and measure their properties](#part-5-code-build-bars-and-measure-their-properties)
6. [Toolbox: how to manage each problem](#part-6-toolbox-how-to-manage-each-problem)
7. [Decision guide and checklist](#part-7-decision-guide-and-checklist)
8. [Glossary](#glossary)

---

# Part 1: Core Concepts

## 1.1 Tick, bar, OHLCV

- A **tick** is one raw trade: `(timestamp, price, quantity)`.
- A **bar** is a summary of many ticks: **O**pen (first price), **H**igh, **L**ow, **C**lose (last price), **V**olume.
- Bar types differ only in **what event closes a bar and starts the next one**.

## 1.2 The variable we analyze: per-bar return

Each bar produces one return:

```
r_t = (Close_t - Close_{t-1}) / Close_{t-1}        (simple return)
r_t = ln(Close_t / Close_{t-1})                     (log return, often preferred)
```

The result is a sequence `r_1, r_2, ..., r_T`, **one number per bar**. With 1-minute bars it is the 1-minute return; with daily bars it is the daily return; with dollar bars it is the return per dollar bar. All the statistical problems below are properties of this sequence.

## 1.3 IID: what statistics and ML want

**IID = Independent and Identically Distributed.**

| Part | Meaning | What breaks it |
|---|---|---|
| **Independent** | Knowing one return tells you nothing about another | Serial correlation |
| **Identically distributed** | Every return comes from the same distribution (same mean, same variance) | Heteroskedasticity, regime changes |

Most classical statistics (t-tests, standard errors, linear regression inference) and many ML validation methods assume something close to IID. When data is not IID, models look more accurate than they are and backtests overfit.

## 1.3b Stationarity (related idea)

A series is (weakly) **stationary** if its mean, variance and autocorrelation do not change over time. Prices are not stationary; returns are *closer* to stationary but still show changing variance. Many models need stationarity.

## 1.4 Serial correlation (autocorrelation)

**Meaning:** the return in one bar is correlated with the return in an earlier bar. "Serial" means "in sequence"; "auto" means the series is correlated with itself.

**How it is calculated (lag k):**

```
rho_k = Cov(r_t, r_{t-k}) / Var(r_t)
```

You line up the series against a copy of itself shifted by k bars and take the ordinary correlation. The result is between -1 and +1. You compute it for many lags (1, 2, 3, ...) and plot the values: this is the **ACF plot**.

**Worked example.** Returns (%): `0.3, 0.5, 0.2, -0.1, -0.4, -0.2`

| Previous (x) | Current (y) |
|---|---|
| 0.3 | 0.5 |
| 0.5 | 0.2 |
| 0.2 | -0.1 |
| -0.1 | -0.4 |
| -0.4 | -0.2 |

Correlation of the two columns = **+0.70** (a toy example; real returns are much weaker).

**How to read it**

| Value | Meaning |
|---|---|
| Near 0 | Past bar says nothing about the next (what IID wants) |
| Positive | Momentum: direction tends to continue |
| Negative | Mean reversion / bounce: direction tends to flip |

**Noise band:** with T bars, values within about `±1.96 / sqrt(T)` are indistinguishable from zero. For 10,000 bars that is about ±0.02.

**Why it appears in financial bars**
- **Bid-ask bounce:** trades alternate between bid and ask prices, creating small *negative* correlation between consecutive returns at high frequency.
- **Order splitting:** large orders are executed in slices over time, so the same direction repeats across consecutive bars (*positive* correlation).
- **Volatility clustering:** *sizes* of moves are correlated (see 1.5).

**Why it matters:** correlated rows carry overlapping information. 10,000 bars may be worth only a few thousand independent observations. Standard errors come out too small and results look too good.

### Rolling serial correlation

A single ρ₁ for the whole sample is an *average* that assumes the relationship is stable. It may flip sign between regimes and cancel out to ~0. To check this, compute it over a rolling window:

```python
rolling_ac1 = r.rolling(W).corr(r.shift(1))
```

Trade-off: small W reacts fast but is very noisy (noise band is `±1.96/sqrt(W)`: ±0.20 at W=100, ±0.09 at W=500, ±0.04 at W=2000). Real return autocorrelation is usually only 0.02 to 0.10, so use hundreds to thousands of bars and always draw the noise band.

## 1.5 Heteroskedasticity (changing variance) and volatility clustering

**Meaning:** the variance of the distribution that generates each bar's return is not constant over time.

```
Ideal (homoskedastic):      r_t ~ D(mu, sigma^2)        same sigma^2 for every bar
Real (heteroskedastic):     r_t ~ D(mu, sigma_t^2)      sigma_t^2 changes with t
```

- The distribution does **not** have to be normal. The common mental model is `r_t ~ N(0, sigma_t^2)` with a changing `sigma_t`.
- You only see **one number per bar**, not the distribution, so you infer `sigma_t` from rolling standard deviation or from the size of recent returns.

**Example with 5-minute time bars:**

| Time of day | Typical 5-min move |
|---|---|
| Open (9:30 to 10:00) | about ±0.40% |
| Lunch | about ±0.05% |
| Close | about ±0.30% |

A 0.2% move is a huge event at lunch and ordinary at the open.

**Volatility clustering:** calm periods follow calm periods; turbulent follows turbulent. This shows up as **serial correlation in squared (or absolute) returns**, even when raw returns have almost none:

```python
(r**2).autocorr(lag=1)     # clearly positive => volatility clustering
```

**Why it matters:** high-volatility bars dominate the loss function and the errors, and tests that assume constant variance give wrong confidence intervals.

## 1.6 Non-normality, skewness, kurtosis, fat tails

**Normal distribution** (bell curve): symmetric (skew = 0), kurtosis = 3 (**excess kurtosis = 0**).

| Term | Meaning | What you see in returns |
|---|---|---|
| **Skewness** | Asymmetry | Often negative (crashes larger than rallies), varies by asset |
| **Kurtosis / fat tails** | Extreme moves are more frequent than the normal predicts | Excess kurtosis well above 0 |
| **Non-normality** | Distribution shape differs from the bell curve | Tall narrow peak, fat tails |

A 5-standard-deviation daily move should essentially never happen under a normal distribution, yet in markets they occur regularly.

**Why heteroskedasticity creates fat tails:** if each bar is normal but with its own variance, then piling all returns into one histogram *mixes* normals with different spreads. A mixture of different-variance normals has a tall peak and fat tails. This is the "mixture of distributions" idea.

**Why it matters:** Gaussian-based risk measures (basic VaR, normal-error tests) underestimate the chance of large losses.

## 1.7 How the problems connect

```
Information arrives at an uneven pace
            +
Bars are cut on a fixed clock (or other non-activity rule)
            |
            +--> different amount of activity per bar
            |        --> different variance per bar        = HETEROSKEDASTICITY
            |        --> mixing variances in one histogram = FAT TAILS / NON-NORMALITY
            |
            +--> overlapping / clustered activity across bars = SERIAL CORRELATION
                 (including serial correlation of squared returns = vol clustering)
```

---

# Part 2: Why the Choice of Bar Matters

**Core idea (market activity clock):** the variance of a price change over an interval is roughly proportional to *how much trading and information* happened in that interval, not to how many minutes passed. A quiet lunch minute and a busy opening minute contain very different amounts of information.

- If you sample **by the clock** (time bars), each bar holds a very different amount of activity, so variances differ and the pooled distribution is fat-tailed.
- If you sample **by activity** (ticks, volume, dollars traded), each bar holds a similar amount of activity, so variances are more similar and returns look closer to IID and normal. Research (e.g., Ané & Geman 2000 using the number of trades as the clock, and López de Prado's tests on S&P futures) supports this tendency.

**Important limits of this idea**
- It *reduces* the problems; it does not *eliminate* them. Volatility per unit of activity still changes (news days, price impact, regime shifts).
- Activity bars give **variable duration**. That is fine for ML, but you lose a fixed calendar grid, which makes cross-asset alignment and calendar reporting harder.
- The tick/volume/dollar choice itself matters (see below).

---

# Part 3: Each Bar Type in Detail

For each bar: formation, good, bad, effect on the four properties, when to use, how to manage.

---

## 3.1 Time Bars

### How formed
Fixed clock interval (1 min, 5 min, 1 hour, 1 day). All trades in the interval are summarized. A new bar starts when the clock ticks over.

```
bar_id = floor(timestamp / interval)
```

### Advantages
- Simplest, supported by every platform, library and data vendor
- Same grid for all assets, so easy to align, compare and build portfolios
- Natural for calendar-driven things: daily P&L, risk reports, earnings, macro data
- Low data volume (no tick data needed if you get ready-made bars)
- Easy to backtest and to explain

### Disadvantages
- **Ignores market activity:** oversamples quiet periods (many near-identical bars), undersamples busy ones (lots of information squashed into one bar)
- Strong intraday seasonality (U-shaped volatility: high open, low midday, higher close)
- Empty or meaningless bars in illiquid periods (overnight, lunch, thin stocks)
- Easier for others to game (algorithms that trade at the minute or hour mark)
- Overnight and weekend gaps produce huge returns that look like outliers

### Effect on statistical properties

| Property | Effect | Why |
|---|---|---|
| Serial correlation of returns | **Weak but can be significant.** Often slightly negative at very high frequency (bid-ask bounce), sometimes positive from order splitting | Microstructure noise |
| Serial correlation of squared returns | **Strong and slow to decay** | Volatility clustering plus intraday seasonality |
| Heteroskedasticity | **Strongest of the common bars** | Different activity per bar, intraday U-shape |
| Non-normality | **High** | Mixing quiet and busy bars |
| Fat tails | **High excess kurtosis**, especially at short intervals | Same cause |

Note: kurtosis tends to fall as the interval gets longer (daily or weekly returns are closer to normal than 1-minute returns), but fat tails and volatility clustering remain.

### When to use
- Charting, reporting, daily/weekly strategies, risk and P&L
- Portfolio construction and cross-asset work needing a common calendar
- Macro/event strategies tied to scheduled releases
- A quick baseline before trying other bars

### How to manage the issues
- Use **longer intervals** (reduces microstructure noise and kurtosis, at the cost of fewer observations)
- **Remove intraday seasonality:** divide each return by the average absolute return for that time of day
- **Scale returns by rolling or EWMA volatility** (`r_t / sigma_t`) so the series is closer to identically distributed
- Use **GARCH-type models** or **volatility targeting**
- Drop or separately handle the **opening/closing auctions and overnight gaps**
- Use mid-price instead of last-trade price to reduce bid-ask bounce
- Use robust or HAC (Newey-West) standard errors; use purged cross-validation in ML

---

## 3.2 Tick Bars

### How formed
Count trades. When the count reaches N (for example 1,000 trades), close the bar and reset the counter.

```
bar_id = floor(trade_number / N)
```

### Advantages
- Sampling follows activity: more bars when the market is busy, fewer when quiet
- Returns are generally closer to IID and normal than time bars
- No empty bars
- Simple to compute (just a counter)

### Disadvantages
- **Order fragmentation:** one big order can appear as 1 trade or 500 trades depending on the exchange, the execution algorithm, or how the vendor aggregates fills. Tick count then does not reflect how much really traded.
- A 1-share trade counts the same as a 10,000-share trade
- Number of trades per day changes over the years (more algorithmic trading, smaller lots), so a fixed N drifts
- Trade counts differ between venues and data vendors, so results are hard to reproduce
- Susceptible to manipulation (spamming small trades creates bars)

### Effect on statistical properties

| Property | Effect | Why |
|---|---|---|
| Serial correlation | **Reduced** vs time bars, but bid-ask bounce is still present at small N; order-splitting artifacts can remain | Sampling per trade |
| Heteroskedasticity | **Reduced** but not removed | Trades differ in size; fragmentation adds noise |
| Non-normality | **Better** than time bars | More even activity per bar |
| Fat tails | **Lower** than time bars, still present | Big trades and news still create outliers |

### When to use
- Instruments where trade size is fairly uniform
- Quick activity-based sampling when you lack reliable volume data
- Microstructure studies where the number of trades is the object of interest

### How to manage
- Choose **N adaptively** (e.g., target a number of bars per day using a rolling average of daily trade counts)
- Be careful with vendor aggregation; check that fills are not split or merged inconsistently
- Use a **larger N** to reduce bid-ask bounce
- Filter odd lots, bad prints and auction trades consistently
- Prefer volume or dollar bars when trade sizes vary a lot

---

## 3.3 Volume Bars

### How formed
Sum the traded quantity (shares or contracts). When cumulative volume reaches V (for example 50,000 shares), close the bar.

```
bar_id = floor(cumulative_volume / V)
```

### Advantages
- **Fixes order fragmentation:** it does not matter whether 10,000 shares traded in 1 print or 100 prints
- Better statistical properties than tick bars in most studies
- Directly tied to a quantity that relates to information flow and liquidity
- Good for futures, where contract counts are stable and meaningful

### Disadvantages
- **Ignores price level:** 10,000 shares at $10 is $100k; at $500 it is $5M. The "same" bar means very different economic size.
- **Corporate actions:** splits change share counts (a 2-for-1 split doubles volume per bar if not adjusted); buybacks and issuance slowly change turnover
- Over long histories, volume per day drifts (liquidity changes), so bars per day change
- Cross-asset comparison is awkward (different share prices)
- For futures, contract rolls and changing contract specs need care

### Effect on statistical properties

| Property | Effect | Why |
|---|---|---|
| Serial correlation | **Lower** than time and tick bars | Equal traded quantity per bar |
| Heteroskedasticity | **Lower** than time/tick over short spans; can reappear over long spans when price level or liquidity shifts | Notional size of a bar changes with price |
| Non-normality | **Better** than time/tick | More even activity |
| Fat tails | **Reduced**; still present | News and regime changes |

### When to use
- Futures and single instruments over a **short to medium** history
- Fixed-lot instruments
- When you want fragmentation-robust sampling and have reliable volume data

### How to manage
- **Adjust for splits** (or use dollar bars)
- Set **V adaptively**: `V = rolling average daily volume / target bars per day`
- Calculate the threshold from **past data only** (shift by one day) to avoid look-ahead bias
- For futures, build bars on **back-adjusted, roll-aware** series
- If you compare many assets, normalize V per asset (e.g., fraction of average daily volume)

---

## 3.4 Dollar Bars (Value Bars)

### How formed
For every trade compute `price * quantity`. Accumulate it; when the total reaches D (for example $5 million), close the bar.

```
bar_id = floor(cumulative(price * quantity) / D)
```

### Advantages
- **Robust to price changes, splits, buybacks and issuance** because it samples the *money exchanged*
- Stable number of bars per day over long histories (when threshold is adaptive)
- Typically the **best statistical properties** among the standard bars in López de Prado's tests (lowest serial correlation, most stable variance, returns closest to normal)
- Good for comparing assets and for ML pipelines; each bar represents a similar economic amount of trading
- Works well for stocks whose price changed a lot over time

### Disadvantages
- **Threshold must still be updated:** a fixed $5M means something very different in 2010 and 2026 (market grew). Fixed thresholds drift.
- Requires **tick-level data**: large, sometimes expensive
- Futures and FX need **contract multipliers and currency conversions** handled correctly
- Dollar value can mix different currencies or quote conventions if not careful
- Still variable duration, so calendar alignment is harder
- Not perfect: volatility per dollar traded still varies (news, regime changes)

### Effect on statistical properties

| Property | Effect | Why |
|---|---|---|
| Serial correlation | **Usually lowest** among standard bars | Equal economic activity per bar |
| Heteroskedasticity | **Lowest** among standard bars, but **not zero** | Volatility per unit of dollar volume still changes |
| Non-normality | **Closest to normal** among standard bars | Variance mixture is reduced |
| Fat tails | **Smallest**, but excess kurtosis is usually still > 0 | Jumps and news |

### When to use
- **Default choice for quant research and ML** on a single asset or a cross-section
- Long histories with changing prices
- When you want comparable sampling across assets and time

### How to manage
- Use an **adaptive threshold**: `D_t = rolling mean of daily dollar volume (past 20 days, shifted) / target bars per day`
- Choose bars per day deliberately (for example 20 to 100) and check the distribution of bars per day
- Handle multipliers (futures), FX conversion and currency consistently
- Apply **volatility scaling** on top (`r_t / sigma_t`) for the remaining heteroskedasticity
- Decide how to treat **overnight/session boundaries** (let bars span them, or reset at session start); see the data-hygiene notes in Part 6

---

## 3.5 Information-Driven Bars: Imbalance Bars and Run Bars

These do **not** try to equalize activity. They try to sample when **new information probably arrived**, detected as one-sided trading.

### Step 1: classify each trade as buy or sell (tick rule)

```
b_t = +1   if price_t > price_{t-1}
b_t = -1   if price_t < price_{t-1}
b_t = b_{t-1}   if price_t == price_{t-1}     (repeat previous sign)
```

(This is an approximation of the trade's **aggressor** side. If your data provides the true aggressor flag, use that instead.)

### Imbalance bars (tick, volume, dollar versions)

- Define the signed flow: `b_t` (tick imbalance), `b_t * qty_t` (volume imbalance), or `b_t * price_t * qty_t` (dollar imbalance)
- Accumulate: `theta_T = sum of signed flow in the current bar`
- Close the bar when `|theta_T|` exceeds the expected imbalance: `E[T] * |E[b]|`
- `E[T]` (expected bar length) and `E[b]` (expected signed flow per trade) are estimated with an exponentially weighted moving average (EWMA) of past bars

**Intuition:** if buys and sells are balanced, the imbalance stays near zero and the bar keeps going. If informed traders push one direction, the imbalance grows fast and the bar closes early.

### Run bars

- Track the **longest run** (sum of the dominant side) rather than net imbalance
- Designed to detect large orders **sliced into many pieces and executed over time**
- Close when the run exceeds its expected value

### Advantages
- Samples **when information probably arrived**, so bars carry signal about informed trading
- Useful for microstructure research and short-horizon prediction
- Bar frequency adapts to order-flow toxicity (buying or selling pressure)
- Run bars specifically detect sliced "iceberg" executions

### Disadvantages
- **Complex:** several parameters (initial expected length, EWMA spans, bounds)
- The **tick rule is only an approximation** of who initiated the trade; errors propagate
- **Unstable bar counts:** bar frequency can explode or collapse if `E[b]` is near zero (threshold becomes tiny) or large
- Hard to reproduce between implementations
- **Not a normalizing tool:** these bars are variable-size by design and do not aim to fix variance or normality
- Require high-quality tick data with accurate timestamps and ordering
- Easy to overfit the many parameters

### Effect on statistical properties

| Property | Effect | Why |
|---|---|---|
| Serial correlation | **Can be higher**, sometimes intentionally (momentum after one-sided flow) | Sampling is conditioned on order-flow direction |
| Heteroskedasticity | **Not controlled**; variance per bar may vary a lot, and bar durations vary widely | Sampling rule is not activity-equalizing |
| Non-normality | **Not guaranteed better** than dollar bars | Sampled at "event" moments, which tend to be larger moves |
| Fat tails | **May remain or increase** | Sampling at informed-flow events picks extreme moments |

Use them as a **signal-timing** tool, not as a way to get clean IID returns.

### When to use
- Microstructure and order-flow research
- Short-horizon strategies that try to detect informed trading, toxicity (VPIN-like ideas) or iceberg orders
- As an extra feature set alongside dollar bars, not as the only bar type for general ML

### How to manage
- Put **minimum and maximum bar lengths** (clamp `E[T]`) to avoid explosions
- Clamp `|E[b]|` away from zero
- Use true aggressor flags where available instead of the tick rule
- Validate stability: plot bars per day over time; check for clusters of micro-bars
- Standardize returns by rolling volatility; **model bar duration** as a feature (short duration = intense flow)
- Compare against a dollar-bar baseline to prove the added complexity helps

---

## 3.6 Price-Based Bars: Range Bars and Renko

### How formed
- **Range bars:** close when the `High - Low` of the bar reaches a fixed size (for example 10 ticks)
- **Renko:** draw a new "brick" only when price moves a fixed amount away from the previous brick's close; otherwise nothing is drawn
- (Point-and-figure charts follow a similar idea.)

### Advantages
- Filter out small noise: price must move a meaningful amount before a new bar appears
- Clean visual trends and support/resistance levels
- Popular in technical analysis and discretionary trading
- No bars during sideways drift

### Disadvantages
- **Ignore volume and time:** a brick may form in 3 seconds or 3 days
- **Return per bar is almost constant by construction** (about ± the brick size), so the return distribution is degenerate or two-valued; classical normality, variance and autocorrelation tests on returns are not meaningful
- The information moves into **bar duration** (which you must model separately)
- Different vendors and parameter choices give different series (and some variants repaint)
- Brick size must be chosen (fixed in price or in ATR terms) and gets stale as volatility changes
- Rarely used in quantitative ML research

### Effect on statistical properties

| Property | Effect | Why |
|---|---|---|
| Serial correlation | **Sign of consecutive bars** is the signal (continuation vs reversal); size autocorrelation is meaningless | Size is fixed |
| Heteroskedasticity | **Hidden:** variance per bar is constant by design, but volatility shows up as bar frequency | Moved into duration |
| Non-normality | **Return distribution is non-normal by construction** (two-point or narrow distribution) | Fixed-size moves |
| Fat tails | **Not visible** in per-bar returns; fat tails move into the duration distribution | Same |

### When to use
- Visual/discretionary trend analysis
- Simple trend-following rules with clearly defined price levels
- Not recommended as the base for ML or statistical inference

### How to manage
- Set the brick/range size as a multiple of ATR or rolling volatility, and recalibrate
- Model **duration** (time per bar) as a feature
- Keep a time- or dollar-bar series alongside for statistical work
- Be explicit about the variant (traditional vs ATR Renko, repainting or not)

---

## 3.7 Related Tool: CUSUM Event Sampling (not a bar type)

Sometimes you do not want to turn *all* data into bars; you want to **sample only when something happened**. A **CUSUM filter** accumulates return and triggers an event when the accumulated move exceeds a threshold h; it then resets. You use events (instead of every bar) as entry points for labeling or training.

- **Good:** fewer, more relevant samples; reduces redundancy
- **Bad:** threshold h must be chosen and adapted to volatility; events themselves are not IID bars
- **Use with:** dollar bars (build bars, then sample events from them)

---

# Part 4: Comparison Tables

## 4.1 Summary of the bar types

| Bar | Closes when | Main strength | Main weakness | Typical use |
|---|---|---|---|---|
| **Time** | Clock interval ends | Simple, universal, calendar-aligned | Ignores activity; worst stats | Reporting, daily strategies, baselines |
| **Tick** | N trades occurred | Follows activity, simple | Order fragmentation; drifts | Uniform-size instruments, microstructure |
| **Volume** | N units traded | Fixes fragmentation | Ignores price level; split/liquidity drift | Futures, short histories |
| **Dollar** | $D traded | Robust to price/splits; best stats | Threshold needs adapting; needs tick data | **Default for quant/ML research** |
| **Imbalance / Run** | Order-flow imbalance exceeds expectation | Captures informed trading | Complex, unstable, not normalizing | Microstructure and signal timing |
| **Range / Renko** | Price moves a fixed amount | Noise filtering, clean visuals | Ignores time/volume; degenerate returns | Visual trend analysis |

## 4.2 Typical statistical properties (general tendencies, **verify on your data**)

Scale: **High** = worst for IID assumptions, **Low** = best.

| Bar | Serial corr. (returns) | Serial corr. (squared returns, vol clustering) | Heteroskedasticity | Non-normality | Fat tails (excess kurtosis) |
|---|---|---|---|---|---|
| **Time** | Low to moderate (bid-ask bounce, splitting) | **High** | **High** | **High** | **High** |
| **Tick** | Low to moderate | Moderate | Moderate | Moderate | Moderate |
| **Volume** | Low | Lower-moderate | Lower-moderate (rises over long spans) | Lower-moderate | Lower-moderate |
| **Dollar** | **Lowest** | **Lowest of the standard bars** (still > 0) | **Lowest of the standard bars** | **Lowest of the standard bars** | **Lowest** (still > 0) |
| **Imbalance / Run** | Can be high by design | Variable | Variable / not controlled | Variable | Variable, can be high |
| **Range / Renko** | Sign-based only | Hidden in duration | Constant by design (moved to duration) | Degenerate | Not applicable to per-bar return |

## 4.3 Practical properties

| Bar | Needs tick data | Fixed calendar grid | Handles splits / price changes | Threshold tuning | Complexity |
|---|---|---|---|---|---|
| Time | No | Yes | Yes (with adjusted prices) | None | Very low |
| Tick | Yes | No | Yes | N must adapt | Low |
| Volume | Yes | No | **No** (needs adjustment) | V must adapt | Low |
| Dollar | Yes | No | **Yes** | D must adapt | Low to medium |
| Imbalance / Run | Yes (ideally with aggressor side) | No | Depends on version | Many parameters | High |
| Range / Renko | Yes or OHLC | No | Brick size must adapt | Brick size | Low |

---

# Part 5: Code: Build Bars and Measure Their Properties

Assumes a pandas DataFrame `ticks` with a **datetime index** and columns `price` and `qty`, sorted by time. Install: `pip install pandas numpy scipy statsmodels matplotlib`.

## 5.1 Building bars

```python
import numpy as np
import pandas as pd

# ---------- Time bars ----------
def time_bars(ticks: pd.DataFrame, rule: str = "5min") -> pd.DataFrame:
    bars = ticks["price"].resample(rule).ohlc()
    bars["volume"] = ticks["qty"].resample(rule).sum()
    bars["dollar"] = (ticks["price"] * ticks["qty"]).resample(rule).sum()
    return bars.dropna()              # drop empty bars


# ---------- Tick / Volume / Dollar bars (fixed threshold) ----------
def activity_bars(ticks: pd.DataFrame, kind: str, threshold: float) -> pd.DataFrame:
    """kind in {'tick', 'volume', 'dollar'}.
    Simplified: bar k holds ticks whose cumulative metric lies in [k*T, (k+1)*T).
    (The exact version resets the counter after each bar; the difference is small.)"""
    p = ticks["price"].to_numpy()
    q = ticks["qty"].to_numpy()
    if kind == "tick":
        metric = np.ones(len(ticks))
    elif kind == "volume":
        metric = q
    elif kind == "dollar":
        metric = p * q
    else:
        raise ValueError("kind must be tick, volume or dollar")

    bar_id = (np.cumsum(metric) // threshold).astype(int)
    df = ticks.assign(bar_id=bar_id, dollar=p * q)
    g = df.groupby("bar_id")

    bars = g["price"].agg(open="first", high="max", low="min", close="last")
    bars["volume"] = g["qty"].sum()
    bars["dollar"] = g["dollar"].sum()
    bars["end_time"] = df.index.to_series().groupby(df["bar_id"]).last()
    return bars.set_index("end_time")


# ---------- Dollar bars with an adaptive threshold (no look-ahead) ----------
def adaptive_dollar_bars(ticks: pd.DataFrame, bars_per_day: int = 50, lookback_days: int = 20) -> pd.DataFrame:
    """Threshold for each day = average dollar volume of the PREVIOUS lookback_days / bars_per_day.
    Bars are built inside each day (counter resets at the start of each day)."""
    dollar = ticks["price"] * ticks["qty"]
    daily = dollar.resample("D").sum()
    daily = daily[daily > 0]
    thr = daily.rolling(lookback_days).mean().shift(1) / bars_per_day   # shift(1) = only past data

    out = []
    for day, day_ticks in ticks.groupby(ticks.index.normalize()):
        t = thr.get(day, np.nan)
        if np.isnan(t) or t <= 0:
            continue                                  # not enough history yet
        out.append(activity_bars(day_ticks, "dollar", t))
    return pd.concat(out)
```

## 5.2 Simplified imbalance bars (educational, not production)

```python
def tick_rule(price: np.ndarray) -> np.ndarray:
    d = np.sign(np.diff(price, prepend=price[0]))
    s = pd.Series(d).replace(0, np.nan).ffill().fillna(1.0)   # repeat previous sign when price unchanged
    return s.to_numpy()

def imbalance_bars(ticks: pd.DataFrame, kind="tick", init_T=100, T_min=10, T_max=5000, span=100):
    """Very simplified version of López de Prado's imbalance bars.
    Threshold = E[T] * |E[b]|, with E[T] and E[b] estimated by EWMA."""
    p = ticks["price"].to_numpy(); q = ticks["qty"].to_numpy()
    b = tick_rule(p)
    signed = {"tick": b, "volume": b * q, "dollar": b * p * q}[kind]
    ewm_b = pd.Series(signed).ewm(span=span).mean().abs().to_numpy()   # |E[b]| per tick (uses past+current only)

    exp_T = float(init_T)
    theta, start = 0.0, 0
    ends, lengths = [], []
    for i, s in enumerate(signed):
        theta += s
        thr = exp_T * max(ewm_b[i], 1e-12)
        if abs(theta) >= thr or (i - start + 1) >= T_max:
            L = i - start + 1
            ends.append(i); lengths.append(L)
            exp_T = float(np.clip(pd.Series(lengths).ewm(span=10).mean().iloc[-1], T_min, T_max))
            theta, start = 0.0, i + 1

    bar_id = np.zeros(len(ticks), dtype=int)
    for k, e in enumerate(ends):
        bar_id[(e + 1):] = k + 1                      # ticks after bar k's end belong to the next bar
    df = ticks.assign(bar_id=bar_id)
    g = df.groupby("bar_id")
    bars = g["price"].agg(open="first", high="max", low="min", close="last")
    bars["volume"] = g["qty"].sum()
    bars["n_ticks"] = g.size()
    bars["end_time"] = df.index.to_series().groupby(df["bar_id"]).last()
    return bars.set_index("end_time")
```

## 5.3 Diagnostics: measure serial correlation, heteroskedasticity, normality, fat tails

```python
from scipy import stats
from statsmodels.stats.diagnostic import acorr_ljungbox, het_arch

def diagnose(bars: pd.DataFrame, name: str, W: int = 500) -> dict:
    r = np.log(bars["close"]).diff().replace([np.inf, -np.inf], np.nan).dropna()
    n = len(r)

    # volatility of volatility: how much does rolling std change over time?
    roll_sd = r.rolling(W).std().dropna()
    vol_cv = roll_sd.std() / roll_sd.mean()

    # standardized returns (divide by past rolling vol) -> does kurtosis fall?
    z = (r / r.rolling(W).std().shift(1)).replace([np.inf, -np.inf], np.nan).dropna()

    # bars per day stability
    per_day = bars.groupby(bars.index.date).size()

    return {
        "bar": name,
        "n_bars": n,
        "noise_band(+-)": 1.96 / np.sqrt(n),
        "ac1_returns": r.autocorr(1),                                   # serial correlation
        "ac1_squared": (r ** 2).autocorr(1),                            # vol clustering
        "ljungbox_p_sq(10)": acorr_ljungbox(r ** 2, lags=[10])["lb_pvalue"].iloc[0],  # small p = clustering
        "arch_lm_p(10)": het_arch(r, nlags=10)[1],                      # small p = heteroskedasticity
        "rolling_vol_CV": vol_cv,                                       # higher = less stable variance
        "skew": stats.skew(r),
        "excess_kurtosis": stats.kurtosis(r),                           # normal = 0
        "excess_kurtosis_vol_scaled": stats.kurtosis(z),                # after removing slow vol changes
        "jarque_bera": stats.jarque_bera(r).statistic,                  # lower = closer to normal
        "bars_per_day_CV": per_day.std() / per_day.mean(),
    }
```

## 5.4 Compare bar types fairly

**Important:** choose thresholds so that all bar types give a **similar number of bars**. Otherwise you are comparing different horizons, and longer horizons look "more normal" by themselves.

```python
total_dollar = (ticks["price"] * ticks["qty"]).sum()
n_target = 20_000                      # same bar count for everyone (approx.)

bars_dict = {
    "time":   time_bars(ticks, "1min"),                                   # tune rule so count ~ n_target
    "tick":   activity_bars(ticks, "tick",   len(ticks) / n_target),
    "volume": activity_bars(ticks, "volume", ticks["qty"].sum() / n_target),
    "dollar": activity_bars(ticks, "dollar", total_dollar / n_target),
}

results = pd.DataFrame([diagnose(b, k) for k, b in bars_dict.items()]).set_index("bar")
print(results.round(4).T)
```

## 5.5 Rolling serial correlation plot with noise band

```python
import matplotlib.pyplot as plt

def plot_rolling_ac(bars, name, W=500, lag=1):
    r = np.log(bars["close"]).diff().dropna()
    ac_r  = r.rolling(W).corr(r.shift(lag))
    ac_r2 = (r ** 2).rolling(W).corr((r ** 2).shift(lag))
    band = 1.96 / np.sqrt(W)

    fig, ax = plt.subplots(2, 1, figsize=(10, 6), sharex=True)
    ac_r.plot(ax=ax[0], title=f"{name}: rolling lag-{lag} autocorrelation of returns")
    ac_r2.plot(ax=ax[1], title=f"{name}: rolling lag-{lag} autocorrelation of squared returns")
    for a in ax:
        a.axhline(band, color="r", ls="--"); a.axhline(-band, color="r", ls="--"); a.axhline(0, color="k", lw=0.5)
    plt.tight_layout(); plt.show()
```

## 5.6 ACF plots and QQ plot

```python
from statsmodels.graphics.tsaplots import plot_acf
import statsmodels.api as sm

r = np.log(bars_dict["dollar"]["close"]).diff().dropna()
plot_acf(r, lags=30);       plt.title("ACF of returns")
plot_acf(r ** 2, lags=30);  plt.title("ACF of squared returns (volatility clustering)")
sm.qqplot(r, line="s");     plt.title("QQ plot: points curling away at the ends = fat tails")
plt.show()
```

## 5.7 How to read the results

| Metric | Closer to ideal means |
|---|---|
| `ac1_returns` | Inside the noise band (near 0) |
| `ac1_squared`, `ljungbox_p_sq` | Small autocorrelation, large p-value (less volatility clustering) |
| `arch_lm_p` | Large p-value (no ARCH effects). Note: with big samples it is almost always tiny; compare the *relative* values across bars |
| `rolling_vol_CV` | Smaller (variance more stable) |
| `excess_kurtosis` | Closer to 0 |
| `excess_kurtosis_vol_scaled` | Closer to 0 (shows how much of the fat tail is just changing volatility) |
| `jarque_bera` | Smaller |
| `bars_per_day_CV` | Smaller (stable sampling over time) |

---

# Part 6: Toolbox: How to Manage Each Problem

## 6.1 Serial correlation

| Technique | What it does |
|---|---|
| Use **larger bars** / lower frequency | Reduces bid-ask bounce and microstructure noise |
| Use **mid-price** (average of bid and ask) instead of last trade | Removes most of the bid-ask bounce |
| Use **activity bars** (dollar) | Reduces clustering of activity across bars |
| **HAC standard errors** (Newey-West) | Corrects standard errors when residuals are autocorrelated |
| **Block bootstrap** | Resampling that preserves dependence |
| **Purged and embargoed cross-validation** (ML) | Prevents leakage between overlapping training and test samples |
| **Sample weights by uniqueness** (ML) | Down-weights overlapping labels so redundant rows do not dominate |
| **Non-overlapping labels or event sampling (CUSUM)** | Fewer redundant observations |
| Do not treat 10,000 bars as 10,000 independent samples | Avoid overstating significance |

## 6.2 Heteroskedasticity and volatility clustering

| Technique | What it does |
|---|---|
| **Activity-based bars** (especially dollar) | Equalizes information per bar |
| **Volatility-standardized returns** `r_t / sigma_t` (using EWMA or rolling std of *past* data) | Makes returns closer to identically distributed |
| **Remove time-of-day seasonality** (for time bars) | Divides out the predictable U-shape |
| **GARCH / EGARCH / realized-volatility models** | Model `sigma_t` explicitly |
| **Weighted least squares** | Gives noisy observations less weight |
| **Volatility targeting** (position sizing) | Keeps risk constant even if volatility changes |
| **Robust standard errors** | Valid inference with changing variance |
| **Rolling retraining** (ML) | Adapts to regime changes |

## 6.3 Non-normality and fat tails

| Technique | What it does |
|---|---|
| **Use log returns** | Better symmetry and additivity than simple returns |
| **Volatility-scale first** | Removes the part of the fat tail caused by changing variance (compare `excess_kurtosis` vs `excess_kurtosis_vol_scaled`) |
| **Winsorize or clip** extreme returns (carefully) | Limits the influence of outliers; do not remove real crashes blindly |
| **Robust losses** (Huber, quantile) and robust statistics (median, MAD) | Less sensitive to outliers |
| **Rank / quantile transform features** | Handles skew and heavy tails in inputs |
| **Tree-based models** | Insensitive to feature scale and monotone transforms |
| **Heavy-tailed distributions** (Student-t, skewed-t), **EVT** for tail risk | Better tail probability estimates than Gaussian |
| **Historical or filtered-historical simulation, Expected Shortfall** | Risk measures that do not rely on normality |
| Never use Gaussian VaR for tail risk without a check | Underestimates extreme losses |

## 6.4 Threshold drift (activity bars)

- Recompute the threshold on a **rolling basis** from **past data only** (`shift(1)`) to avoid look-ahead bias
- Pick a **target bars per day** (e.g., 20 to 100), not a fixed absolute number
- Monitor **bars per day over time**; a drifting count means the threshold is stale
- Consider resetting at each session start, and decide explicitly how to handle overnight bars

## 6.5 Data hygiene (affects every bar type)

| Issue | What to do |
|---|---|
| Bad prints, outliers, zero or negative prices | Filter with explicit rules; keep a log |
| Out-of-order or duplicate ticks | Sort by timestamp then sequence number; deduplicate |
| Auction/opening/closing trades, odd lots | Decide consistently whether to include them |
| Overnight gaps and weekend gaps | Treat as separate returns or flag them; do not mix with intraday returns |
| Stock splits and dividends | Adjust prices and volumes (dollar bars are naturally robust) |
| Futures roll | Use back-adjusted or roll-aware series; do not compute returns across the roll |
| Time zones and daylight saving | Convert to one standard (UTC) before bar building |
| Aggressor side available? | Use it instead of the tick rule |

## 6.6 Cross-asset and calendar alignment (when using activity bars)

Activity bars from different assets do not line up in time. If you need a common grid (portfolio models), build bars per asset, then **resample to a common clock** (e.g., take the last close at each minute or day), or use time bars for the portfolio layer and dollar bars for single-asset signal research.

---

# Part 7: Decision Guide and Checklist

## 7.1 Which bar should I use?

| Your goal | Recommended bar |
|---|---|
| Charts, reporting, daily or weekly strategies, P&L and risk | **Time bars** (and apply volatility scaling) |
| ML / quant research on one or many assets, long history | **Dollar bars** (adaptive threshold) |
| Futures research with stable contract size | **Volume bars** or **dollar bars** |
| No reliable volume data, but trades are uniform | **Tick bars** |
| Detecting informed trading, order-flow signals, microstructure | **Imbalance / run bars** (alongside dollar bars) |
| Visual trend analysis | **Range / Renko** |
| Portfolio models needing a common calendar | **Time bars** (resample activity bars if needed) |
| Few relevant samples for labeling/training | **Dollar bars + CUSUM event filter** |

## 7.2 Workflow checklist

1. Clean the tick data (Part 6.5).
2. Build candidate bars with **similar bar counts** (Part 5.4).
3. Compute the diagnostics (Part 5.3) and plots (5.5, 5.6).
4. Check **bars per day** stability and the distribution of bar durations.
5. Apply **volatility scaling** to returns and recheck kurtosis and autocorrelation of squared returns.
6. Check whether results are **stable across time** (rolling ACF, split by year or regime).
7. Pick the bar that gives the best properties **for your use case**, and prefer the simplest one that works.
8. In ML: use purged cross-validation with embargo and uniqueness weights, regardless of bar type.
9. Re-evaluate periodically; market structure changes.

## 7.3 Limits of this analysis

- "Dollar bars are best" is a tendency from specific studies (largely S&P 500 futures). It can differ for illiquid assets, different periods, or different threshold choices.
- No bar type removes all problems. Even the best bars leave **excess kurtosis > 0** and some **volatility clustering**, because markets have jumps, news and regime changes.
- Comparisons must hold the **number of bars** (horizon) fixed, or you are measuring the effect of horizon, not bar type.
- Statistical tests on very large samples reject almost everything, so compare **magnitudes across bar types**, not just p-values.

---

# Glossary

| Term | Short meaning |
|---|---|
| **Tick** | One raw trade |
| **Bar** | Summary (OHLCV) of many ticks |
| **IID** | Independent and identically distributed |
| **Stationarity** | Statistical properties (mean, variance, autocorrelation) do not change over time |
| **Serial correlation / autocorrelation** | Correlation of a series with its own past values |
| **ACF** | Autocorrelation function: autocorrelation at each lag |
| **Heteroskedasticity** | Variance changes over time |
| **Homoskedasticity** | Constant variance |
| **Volatility clustering** | Large moves follow large moves, small follow small |
| **Skewness** | Asymmetry of a distribution |
| **Kurtosis** | Heaviness of tails (normal = 3; excess kurtosis = kurtosis - 3) |
| **Fat tails** | Extreme events more likely than a normal distribution implies |
| **Bid-ask bounce** | Prices alternating between bid and ask, creating negative autocorrelation |
| **Tick rule** | Classifying trades as buy or sell from price changes |
| **Aggressor side** | The side (buyer or seller) that initiated the trade |
| **Imbalance** | Net signed order flow (buys minus sells) |
| **EWMA** | Exponentially weighted moving average |
| **GARCH** | Model for time-varying volatility |
| **CUSUM filter** | Event sampler that triggers when accumulated moves exceed a threshold |
| **HAC / Newey-West** | Standard errors robust to autocorrelation and heteroskedasticity |
| **Purging / embargo** | Removing training samples that overlap or sit close to test samples in time |
| **Look-ahead bias** | Using information in a calculation that would not have been known at that time |