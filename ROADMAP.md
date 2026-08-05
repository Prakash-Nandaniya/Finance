# 4-Month Quant Risk Roadmap — Full Detail
### Main topic → ordered subtopics (basic → advanced) → where to study → libraries (basic + industry-specific) → what to build → how to write the notebook so it reads as resume-worthy work

---

## First: how to structure EVERY notebook so it's worth something on a resume

Do this same structure every single week — it's what separates "I coded a formula" from "I did an analysis." A recruiter or interviewer should be able to open the notebook and read it like a short report, not like scratch code.

1. **Problem statement (markdown cell, top of notebook):** 2-3 lines — what question are you answering, on what data, why it matters practically. Not "implement VaR" — instead "Does normal-distribution VaR understate real tail risk on Nifty? Testing on 5 years of daily data."
2. **Data section:** where the data came from, date range, any cleaning you did (missing days, corporate actions, adjusted vs unadjusted close — state which you used and why).
3. **Method section:** briefly state the method(s) in your own words before the code — 2-3 lines, not a textbook definition copy-paste. This is what proves you understand it rather than copied it.
4. **Code, well-commented:** function-level docstrings, not just inline comments. Structure as reusable functions/classes, not one giant script cell — this also lets you reuse code across weeks (build a small personal library as you go).
5. **Results section:** the actual numbers/plots, clearly labeled axes and titles (not default matplotlib labels).
6. **Interpretation (markdown cell, this is the part most people skip and the part that actually matters):** 3-5 lines answering "so what?" — what does this number mean for someone managing risk, and where would this method fail or mislead. This paragraph is what makes a notebook worth showing in an interview instead of just existing in a repo.
7. **README per project/repo:** one paragraph, what it does, what a risk/trading desk would actually use it for, how to run it.

A notebook without step 6 is a code exercise. A notebook with it is an analysis. Do not skip it, ever, even under time pressure — it's the highest-value 10 minutes of each week.

---

# MONTH 1 — Data, Statistics, VaR

## Week 1 — Main Topic: Financial Returns & Data Sampling

**Subtopics, basic → advanced:**
1. Simple returns vs log returns — why log returns are used (additive over time, better statistical properties)
2. Cumulative returns, annualizing returns and volatility
3. Rolling statistics — rolling mean, rolling std, rolling correlation
4. Missing data handling in financial time series (holidays, halts, delistings)
5. Corporate action adjustment — splits/dividends distorting raw price history (concept + why unadjusted data gives false signals)
6. **Advanced:** Time bars vs volume bars vs dollar bars — why sampling by calendar day produces statistically flawed (non-stationary, non-i.i.d.) data compared to sampling by activity

**Where to study:**
- Subtopics 1-4: `yfinance`/pandas official docs + any "pandas time series" free tutorial (Real Python has a solid free one)
- Subtopic 5: search "corporate action adjustment finance explained" — short, well-covered concept, no book needed
- Subtopic 6 (advanced): search "financial data structures volume bars dollar bars explained" — several free blog explainers cover this well; this is the one place I'd cross-check two sources since it's a less mainstream concept

**Libraries — basic:**
`pandas`, `numpy`, `matplotlib`/`seaborn`, `yfinance`

**Libraries — industry-specific (used, not black-box):**
None needed yet at this stage — Week 1 is genuinely pandas-level, no specialized library exists in industry for "volume bars," you construct them yourself with pandas groupby logic, which is also more valuable to show than importing a package that does it for you.

**Build:** Pull 10 NSE stocks (`.NS` tickers). Compute standard daily-bar log returns for all. Pick one liquid stock (e.g., Reliance), construct volume bars manually (resample by cumulative traded volume crossing a threshold instead of by calendar day). Compare skew, kurtosis, and lag-1 autocorrelation between time-bar returns and volume-bar returns on the same stock. Output: a comparison table + one paragraph interpretation of which sampling method gives more well-behaved data and why that matters for any model built downstream.

---

## Week 2 — Main Topic: Probability & Statistics for Finance

**Subtopics, basic → advanced:**
1. Normal distribution — PDF/CDF, Z-scores
2. Skewness and kurtosis in real return distributions ("fat tails")
3. Jarque-Bera normality test
4. Hypothesis testing — t-test, p-values, Type I/II errors
5. Confidence intervals — correct interpretation
6. Central Limit Theorem — why it matters for portfolio aggregation
7. **Advanced:** Extreme Value Theory (EVT) — fitting a Generalized Pareto Distribution to tail losses specifically, instead of assuming the whole distribution is normal

**Where to study:**
- Subtopics 1-6: `scipy.stats` documentation examples + Khan Academy for underlying stats intuition + any free "statistics for finance" YouTube playlist
- Subtopic 7 (advanced): search "Extreme Value Theory VaR tail risk explained" or "Generalized Pareto Distribution finance risk management" — a handful of solid free explainer articles/videos exist; this is a real technique used in tail-risk management, not academic-only

**Libraries — basic:**
`scipy.stats`, `numpy`

**Libraries — industry-specific:**
`scipy.stats.genpareto` for EVT fitting — this is the actual function practitioners use, not a black box, it's a standard distribution fit you're doing the math on yourself, the library just gives you the fitting routine

**Build:** On Nifty 50 daily losses (5+ years), fit a normal distribution to the whole return series. Separately, isolate the worst 5-10% of days and fit a Generalized Pareto Distribution to just that tail. Compute 99% VaR both ways. Output: the two VaR numbers side by side, quantify the percentage by which normal-distribution VaR underestimates the EVT-based tail estimate, and interpret what that gap means practically for a risk desk relying on normal VaR alone.

---

## Week 3 — Main Topic: Value at Risk (core risk metric)

**Subtopics, basic → advanced:**
1. VaR definition and its three parameters (confidence level, horizon, currency/base)
2. Historical Simulation VaR
3. Parametric (Variance-Covariance) VaR
4. Monte Carlo VaR
5. CVaR / Expected Shortfall
6. Square-root-of-time scaling rule (and its i.i.d. assumption/limitation)
7. **Advanced:** Filtered Historical Simulation (FHS) — combine a GARCH-forecasted volatility with historical standardized residuals, rather than raw historical returns; this is closer to what real trading-book VaR engines run overnight

**Where to study:**
- Subtopics 1-6: GARP's free FRM Part 1 study guide topic outline + Bionic Turtle/AnalystPrep free YouTube videos on VaR
- Subtopic 7 (advanced): search "Filtered Historical Simulation VaR explained" — this is a well-documented, genuinely industry-standard technique (originally from Barone-Adesi et al.), several free academic-style explainers cover the method step by step

**Libraries — basic:**
`scipy.stats.norm` (parametric VaR), `numpy` (Monte Carlo simulation, historical sim)

**Libraries — industry-specific:**
`arch` library for the GARCH-forecasted volatility step feeding into FHS — this is the real Python library practitioners and researchers use for GARCH family models, not a black box; you're using it for one well-defined step (volatility forecast) and applying the filtering logic yourself

**Build:** On a 5-8 stock NSE portfolio, compute all four VaR methods for the same date: Historical Sim, Parametric, Monte Carlo, and FHS. Output: a comparison table across methods, plus a short written explanation of why FHS diverges most from plain Historical Sim during a high-volatility window (pick a real turbulent period in your data range and show the divergence there specifically, not just on a calm day — that's where the method differences actually matter).

---

## Week 4 — Main Topic: VaR Backtesting

**Subtopics, basic → advanced:**
1. Exception counting — what counts as a VaR breach
2. Basel minimum observation window (250 trading days) and why
3. Kupiec test (unconditional coverage test) — formula and interpretation
4. **Advanced:** Christoffersen's independence test — checks whether exceptions cluster in time (regime-shift blindness), not just whether the count is right; a model can pass Kupiec and still be dangerously wrong if all its breaches happen in the same crisis week

**Where to study:**
- Subtopics 1-3: GARP free study guide (VaR backtesting is a named FRM topic), Jorion "Value at Risk" Ch.5 concept is widely summarized for free in FRM prep blogs — search "Kupiec test VaR backtesting explained," verify the exact critical value table against one of these free sources before quoting it
- Subtopic 4 (advanced): search "Christoffersen independence test VaR" — documented in the same FRM-adjacent free resources, less commonly known so it's worth the extra search effort

**Libraries — basic:**
`numpy`, `scipy.stats.chi2` (both Kupiec and Christoffersen are likelihood-ratio tests built on the chi-squared distribution — you're implementing the actual test statistic yourself, this isn't something you import pre-built, which is exactly why it's a strong notebook to show)

**Libraries — industry-specific:**
None beyond `scipy.stats` — backtesting tests are standard enough that implementing them yourself (rather than finding a packaged function) is both more common in practice and more impressive in an interview.

**Build:** Run your Week 3 VaR engine across your full 2+ years of data, log every daily exception (actual loss > VaR estimate). Run both the Kupiec and Christoffersen tests on the exception series. Output: does your model pass both tests, and if it fails one but not the other, explain concretely what that specific failure means (e.g., "correct number of breaches, but they clustered in March 2020 — the model didn't adapt fast enough to the regime shift").

---

# MONTH 2 — Volatility, Portfolio Construction, Factor Models

## Week 5 — Main Topic: Volatility Modeling

**Subtopics, basic → advanced:**
1. Why static (rolling-window) volatility is naive — volatility clustering
2. EWMA (Exponentially Weighted Moving Average) — decay factor λ, RiskMetrics standard (λ=0.94)
3. GARCH(1,1) — model structure, how parameters are fit (MLE), how to interpret them
4. **Advanced:** GJR-GARCH or EGARCH — asymmetric volatility response (bad news raises vol more than good news of equal size); standard GARCH misses this real market behavior
5. **Advanced:** Range-based realized volatility estimators — Parkinson or Garman-Klass, using daily high/low/open/close as a proxy for "true" volatility without needing tick data

**Where to study:**
- Subtopics 1-3: `arch` library's own documentation and examples (genuinely well-written), plus RiskMetrics' EWMA logic is summarized free in many FRM-prep sources — verify λ=0.94 against one of these
- Subtopic 4 (advanced): `arch` library documentation directly covers EGARCH/GJR-GARCH model specification — read the docs, they explain the asymmetry concept alongside the code
- Subtopic 5 (advanced): search "Parkinson volatility estimator" and "Garman-Klass volatility estimator explained" — short, well-documented, genuinely used as a realized-vol proxy in practice when tick data isn't available

**Libraries — basic:**
`numpy`, `pandas`

**Libraries — industry-specific:**
`arch` — this is the real library researchers and practitioners use for the entire GARCH family (GARCH, EGARCH, GJR-GARCH); it's not a black box because you still choose the model spec and interpret the fitted parameters yourself, the library just handles the MLE optimization, which nobody implements from scratch in practice either

**Build:** On Bank Nifty, fit EWMA, GARCH(1,1), and EGARCH volatility forecasts. Separately compute Parkinson realized volatility directly from OHLC data as your "ground truth" proxy. Backtest which forecasting method's predictions track realized volatility more closely — use squared forecast error against the realized-vol proxy as your metric. Output: a ranked comparison of the three forecasting methods with the actual error numbers, and interpret whether the added complexity of EGARCH over GARCH actually paid off on this data (sometimes it won't — say so if that's what the numbers show, that's more credible than always claiming the fancier model won).

---

## Week 6 — Main Topic: Portfolio Construction

**Subtopics, basic → advanced:**
1. Portfolio return and variance — weighted average, covariance matrix in matrix form
2. Diversification benefit — why portfolio variance is less than the weighted average of individual variances
3. Efficient Frontier — construction, minimum variance portfolio, maximum Sharpe portfolio
4. CAPM — beta, alpha, Security Market Line
5. **Advanced:** Black-Litterman model — blends market-implied equilibrium returns with your own views, addressing Markowitz's sensitivity to noisy expected-return inputs
6. **Advanced:** Risk Parity / Equal Risk Contribution — allocates by risk contribution rather than capital weight (the logic behind Bridgewater-style "All Weather" portfolios)

**Where to study:**
- Subtopics 1-4: `PyPortfolioOpt` documentation (has a genuinely good written walkthrough of Markowitz theory alongside the code) + Investopedia for CAPM/beta refreshers
- Subtopics 5-6 (advanced): `PyPortfolioOpt` documentation directly covers both Black-Litterman and Risk Parity/HRP with explanation of the underlying logic, not just function signatures — read the "User Guide" section of their docs, it's unusually thorough for a library doc

**Libraries — basic:**
`numpy`, `pandas`

**Libraries — industry-specific:**
`PyPortfolioOpt` — covers efficient frontier, Black-Litterman, and risk parity/HRP in one library; it's the standard open-source tool for portfolio construction experimentation and is not a black box as long as you understand and can explain the inputs (views, covariance estimation method, risk-aversion parameter) you're feeding it, which is what your interpretation paragraph should demonstrate

**Build:** On a real 15-20 stock NSE portfolio, build three versions: naive Markowitz max-Sharpe, Black-Litterman (form 2-3 of your own simple views — e.g., "IT sector outperforms broad market by X% over the next period" — and show how the resulting weights shift from the market-equilibrium baseline), and Risk Parity. Output: resulting weights and each portfolio's realized per-stock risk contribution, side by side. Interpretation: show concretely how naive Markowitz concentrates into 2-3 names while risk parity spreads exposure, and state which approach you'd trust more and why — that's the actual argument practitioners make against naive mean-variance optimization.

---

## Week 7 — Main Topic: Factor Models (built for Indian equities — genuinely advanced because you're constructing the data, not downloading it)

**Subtopics, basic → advanced:**
1. OLS regression — assumptions (Gauss-Markov), R², residual diagnostics
2. Multiple regression
3. Fama-French factor logic (Market, Size, Value) — concept, since ready-made India data doesn't exist for this
4. **Advanced:** Constructing your own India-specific factors — size (market cap), value (P/B ratio), momentum (12-month return), sorting stocks into deciles/quintiles, forming long-short factor portfolios
5. **Advanced:** PCA-based statistical factor extraction (Barra-style) — finding hidden risk factors directly from the data instead of naming them upfront

**Where to study:**
- Subtopic 1-3: `statsmodels` OLS documentation/examples + Ken French Data Library's own methodology notes (explains exactly how Fama-French factors are constructed — you'll replicate this logic on Indian data)
- Subtopic 4 (advanced): re-read Ken French's methodology page carefully — it's a genuinely well-written free explanation of factor construction mechanics you'll mirror
- Subtopic 5 (advanced): `sklearn` PCA documentation + search "PCA statistical factor model finance explained" for the interpretation side (how to map principal components back to economic meaning)

**Libraries — basic:**
`pandas`, `numpy`

**Libraries — industry-specific:**
`statsmodels` (OLS regression — the standard for statistical regression with proper diagnostics, more appropriate than `sklearn`'s regression here because you need the statistical output — t-stats, p-values — not just predictions), `sklearn.decomposition.PCA` (the standard PCA implementation, used the same way in practice)

**Build:** Using ~30-50 NSE stocks, construct your own size, value, and momentum factors by sorting stocks into deciles on each characteristic and forming long-short portfolios (top decile minus bottom decile). Regress a chosen stock or mutual fund NAV against your homemade factors — report factor exposures (betas) and whether alpha is statistically significant (t-test). Then run PCA on the same 30-50 stock return matrix, extract the top 3 principal components, and check how much of their variance correlates with your named factors. Output: factor regression table + PCA variance-explained table + interpretation of whether your named factors and the data-driven PCA factors actually agree.

---

## Week 8 — Main Topic: Time Series & Pairs Trading

**Subtopics, basic → advanced:**
1. Stationarity — concept, Augmented Dickey-Fuller (ADF) test
2. Autocorrelation (ACF) and Partial Autocorrelation (PACF)
3. Cointegration — concept, Engle-Granger test
4. Static OLS hedge ratio for a pairs-trading spread
5. Basic mean-reversion signal generation and backtest (entry/exit thresholds, Sharpe ratio, drawdown)
6. **Advanced:** Kalman filter for a dynamically-updating hedge ratio — updates the hedge ratio continuously as new data arrives instead of using one fixed value estimated once, which goes stale as the relationship between the two stocks drifts

**Where to study:**
- Subtopics 1-5: `statsmodels` time series documentation (has both ADF and cointegration test functions with worked examples) + any free "pairs trading strategy Python" tutorial
- Subtopic 6 (advanced): `pykalman` documentation's own examples (several directly demonstrate the pairs-trading dynamic-hedge-ratio use case) + search "Kalman filter pairs trading hedge ratio explained" for the intuition behind why this beats a static ratio

**Libraries — basic:**
`pandas`, `numpy`

**Libraries — industry-specific:**
`statsmodels` (ADF test, Engle-Granger cointegration test — the standard implementations used in both research and practice), `pykalman` (this is a genuinely used tool for the dynamic hedge ratio approach systematic funds run; you're not black-boxing it as long as you understand and can explain the state-space setup — state = hedge ratio, observation = price relationship — which your interpretation paragraph should cover)

**Build:** Test cointegration between two related NSE stocks (e.g., HDFC Bank/ICICI Bank, or a same-sector pair). Build a pairs-trading backtest using a static OLS hedge ratio (returns, Sharpe, drawdown). Rebuild the same backtest using a Kalman-filter-updated hedge ratio. Output: Sharpe ratio and max drawdown for both versions side by side, plus a plot of how the hedge ratio evolves over time under the Kalman approach vs staying flat under the static one. Interpretation: does the dynamic version actually earn its added complexity on this specific pair, or not — report the honest answer either way.

---

# MONTH 3 — Credit Risk, Derivatives (Real Data), Ship the App

## Week 9-10 — Main Topic: Credit Risk Modeling

**Subtopics, basic → advanced:**
1. PD (Probability of Default), LGD (Loss Given Default), EAD (Exposure at Default)
2. Expected Loss = PD × LGD × EAD
3. Logistic regression as a baseline PD model
4. Gradient boosting (XGBoost) for PD modeling — why it typically outperforms logistic regression on this kind of tabular data
5. **Advanced:** Walk-forward validation — why a random train/test split leaks time-based information in credit/financial data and produces an overstated accuracy
6. **Advanced:** SHAP values for per-borrower explainability — required in practice, not optional, under current model-explainability expectations (SR 11-7 guidance, IFRS 9 model governance)
7. **Advanced:** IFRS 9 Expected Credit Loss staging — Stage 1 (12-month ECL) vs Stage 2/3 (lifetime ECL), the actual accounting/provisioning logic banks apply on top of a raw PD model

**Where to study:**
- Subtopics 1-4: `sklearn` and `xgboost` official documentation/examples, both are unusually well-written for self-teaching
- Subtopic 5 (advanced): search "walk-forward validation time series machine learning explained" — a well-established concept with several good free explainers, `sklearn.model_selection.TimeSeriesSplit` documentation also covers the mechanics directly
- Subtopic 6 (advanced): `shap` library's own documentation — it has extensive free tutorials specifically on tabular/credit-style models
- Subtopic 7 (advanced): search "IFRS 9 ECL staging explained" or "IFRS 9 Stage 1 Stage 2 Stage 3" — this is standard, well-documented banking-accounting material, several free explainers from Big4 firms themselves (they publish client-education content on this) exist

**Libraries — basic:**
`pandas`, `numpy`, `sklearn` (logistic regression baseline, `TimeSeriesSplit`)

**Libraries — industry-specific:**
`xgboost` — the actual most-used model for credit scoring/PD modeling in practice per your own reference material; `shap` — genuinely required-in-practice for model explainability, not a decorative add-on, so learning to interpret (not just generate) SHAP plots is the real skill here

**Build:** Build an XGBoost PD model on a real credit dataset (Kaggle Lending Club or UCI German Credit) using proper walk-forward validation (not random split — explicitly show why the random-split accuracy would have been misleadingly high if you check it as a comparison). Compute Expected Loss (PD×LGD×EAD, with reasonable assumed LGD/EAD if the dataset doesn't provide them). Add SHAP explanations for a handful of individual borrowers (not just aggregate feature importance — per-borrower is the harder, more valuable skill). Layer on IFRS 9 staging logic: classify borrowers into Stage 1/2/3 using a simple deterioration rule (e.g., PD increase beyond a threshold since origination), and show how total portfolio provisioning changes between "everyone treated as Stage 1" and proper staging. Output: PD model performance, Expected Loss table, SHAP plots for a few borrowers, and the staging/provisioning comparison — this maps almost exactly onto what a bank's credit risk or model validation team produces.

---

## Week 11 — Main Topic: Options & Derivatives (priced against real market data)

**Subtopics, basic → advanced:**
1. Options basics — calls, puts, payoffs, moneyness, put-call parity
2. Black-Scholes formula — inputs, assumptions, closed-form pricing
3. The Greeks — Delta, Gamma, Theta, Vega, Rho — formulas and practical meaning
4. Monte Carlo pricing via Geometric Brownian Motion simulation — confirming it converges to the Black-Scholes closed-form price
5. Implied volatility — solving the Black-Scholes formula backward from a real market price
6. **Advanced:** Implied volatility surface — across both strike AND expiry (not just a single-expiry smile), built from real NSE option chain data
7. **Advanced:** Interpreting the smile/surface as direct evidence that Black-Scholes' constant-volatility assumption is wrong — this is the actual insight a derivatives desk lives by

**Where to study:**
- Subtopics 1-5: any solid free "Black-Scholes Python implementation" tutorial + Investopedia for Greeks intuition
- Subtopics 6-7 (advanced): `nsepython`/`nselib` library documentation for how to actually pull real NSE option chain data (this is the genuinely hard/valuable part — most tutorials use toy inputs, you're using live market prices) + search "implied volatility surface construction explained" for the fitting/interpolation approach across strikes and expiries

**Libraries — basic:**
`numpy`, `scipy.stats.norm` (for the Black-Scholes formula components), `scipy.optimize.brentq` (for solving implied volatility numerically — this is how it's actually done in practice, there's no closed-form inverse)

**Libraries — industry-specific:**
`nsepython` or `nselib` — real, free libraries that pull live/historical NSE options data, genuinely used for retail-quant projects and demonstrably not black-box since you still do the BS pricing and IV-solving yourself, the library only handles data retrieval

**Build:** Implement Black-Scholes pricing + all five Greeks from scratch (formula-level, not imported). Pull a real Nifty option chain across multiple strikes and at least 2-3 expiries via `nsepython`/`nselib`. For each contract, solve for implied volatility from the real market price (using `scipy.optimize.brentq` against the BS formula). Plot the full IV surface (strike × expiry × IV). Separately, price a few of the same options via Monte Carlo GBM simulation and confirm convergence to your BS closed-form price. Output: the IV surface plot + a written interpretation of what the smile/skew shape tells you about market-implied tail risk (e.g., steeper OTM put IV = market pricing in more downside risk than a flat-vol BS model would predict) — this is your evidence that BS's core assumption doesn't hold in real markets, which is exactly the kind of insight a derivatives desk actually cares about.

---

## Week 12 — Ship the app

Pick your single strongest notebook — most likely the VaR+FHS+backtesting stack (Weeks 3-4 combined) or the volatility forecasting notebook (Week 5). Turn it into a deployed, interactive app.
- **Streamlit** is the fastest path for a solo build in a week — recommended default
- **FastAPI + a minimal frontend** is fine instead if you specifically want to demonstrate backend/API skills for FinTech-analyst-style roles, since that's your existing comfort zone
- Deploy it (Streamlit Community Cloud is free) so you have a live link, not just a GitHub repo
- Write the README the way described at the top of this document — problem, method, how to run it, what a risk desk would actually use it for

---

# MONTH 4 — Fixed Income, Stress Testing, Interview Sprint

## Week 13 — Main Topic: Fixed Income

**Subtopics, basic → advanced:**
1. Time value of money — PV, FV, discounting
2. Bond pricing — coupon bonds, zero-coupon bonds
3. Yield to Maturity (YTM)
4. Duration — Macaulay Duration, Modified Duration
5. Convexity — why duration alone underestimates price change for large yield moves
6. DV01 (Dollar Value of 1 basis point)
7. **Advanced:** Nelson-Siegel yield curve model — fitting a smooth parametric curve to observed yields across maturities, the same general approach central banks use for curve construction

**Where to study:**
- Subtopics 1-6: any solid free "bond math Python" tutorial, Investopedia for concept refreshers
- Subtopic 7 (advanced): search "Nelson-Siegel model yield curve fitting Python" — well-documented technique with several free walkthroughs including the parameter interpretation (level, slope, curvature)

**Libraries — basic:**
`numpy`, `scipy.optimize` (for YTM solving, which requires numerical root-finding, not a closed form)

**Libraries — industry-specific:**
None strictly required — Nelson-Siegel is commonly implemented directly with `scipy.optimize.curve_fit` rather than a specialized package, which is also the more instructive path here; if you want a packaged version, the `nelson_siegel_svensson` PyPI package exists and is lightweight enough not to be a black box

**Build:** Pull real Indian G-Sec yield data across maturities (RBI/CCIL public data). Fit a Nelson-Siegel curve to it. Pick one bond, compute duration/convexity/DV01. For a large hypothetical yield shock (e.g., +200bps), compare the actual re-priced bond value against the duration-only approximation and the duration+convexity approximation. Output: the yield curve fit plot + a table showing the real price-change error from duration-only vs duration+convexity — this is where convexity's practical value becomes a concrete, quotable number instead of an abstract concept.

---

## Week 14 — Main Topic: Stress Testing & Scenario Analysis

**Subtopics, basic → advanced:**
1. Stress testing vs VaR — conceptual difference (hypothetical/historical shock replay vs statistical confidence-interval estimate)
2. Historical scenario replay — applying an actual past crisis's realized shocks to a current portfolio
3. Sensitivity analysis — bump-and-reprice for a single risk factor
4. **Advanced:** Reverse stress testing — instead of "what happens if X shock occurs," ask "what shock magnitude would break this portfolio," and solve for it

**Where to study:**
- Subtopics 1-3: GARP free FRM study guide (stress testing is a named topic), several free FRM-prep blog explainers cover the DFAST/RBI-style approach at a conceptual level
- Subtopic 4 (advanced): search "reverse stress testing methodology explained" — a real, named regulatory technique (post-2008), documented in free risk-management explainers

**Libraries — basic:**
`pandas`, `numpy`

**Libraries — industry-specific:**
None specialized — stress testing at this scale is fundamentally simple arithmetic (apply a % shock vector to your portfolio weights and revalue), which is itself worth noting in your interpretation: the complexity in real stress-testing frameworks is in scenario design and governance, not in the computation.

**Build:** Take your Week 6 portfolio. Apply real historical shock magnitudes from three periods — 2008 GFC, COVID crash (2020), and the 2016 India demonetization shock — to your current holdings. Produce a tornado chart showing which holdings drive the largest stress loss in each scenario. Compare the worst stress-test loss to your Month 1 VaR number on the same portfolio. Output: the tornado chart + a table comparing stress loss vs VaR across scenarios, with an interpretation of why stress testing catches risks (correlated, sudden, prior-precedent-defying) that a normal-times VaR model structurally can't.

---

## Week 15 — Second project polish + resume

If time allows, deepen either the options/IV-surface notebook or the homemade-factor-model notebook into a second polished portfolio piece (these two are your most differentiated, resume-worthy notebooks). Otherwise, spend the week tightening READMEs across all repos, polishing the deployed app from Week 12, and rewriting your resume to lead with the deployed app and your two strongest analyses (not a list of every topic covered).

## Week 16 — Interview sprint

- Be able to explain out loud, not just in code: FHS vs plain Historical VaR, EWMA vs GARCH vs EGARCH, Markowitz vs Black-Litterman vs Risk Parity, static vs Kalman-filter hedge ratios, VaR vs stress testing, why walk-forward validation matters for credit models
- Survivorship bias and look-ahead bias — know these cold, they come up in almost every "how do you know your backtest is real" conversation
- Mock interview questions sourced from r/FRM and r/quantfinance threads
- Keep applying throughout Month 4, not just this week — you should have started applications in Month 3

---

## Why this level of detail matters
Notice the pattern in every "Build" section: you're never just implementing a formula in isolation. You're comparing two methods on the same real data and reporting which one actually performed better, with a number attached. That comparison — not the formula itself — is the thing that makes a notebook interview-worthy. Anyone can code Black-Scholes from a textbook. Very few final-year students show up with "here's the real Nifty IV surface, and here's the specific evidence it violates Black-Scholes' core assumption." Lead with that difference in every conversation.