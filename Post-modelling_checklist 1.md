# Post-Modelling Test Checklist for SKU Time-Series

---

## A. Actual Demand Analysis

_Tests that characterize the historical actuals time-series alone._

### A1 — Data Quality & Demand Profiling

| # | **Test Name** | **Metric / Logic** | **Expected Output Type** | **Objective & Interpretation** |
|---|---|---|---|---|
| A1.1 | **Date Continuity** | $$\text{Missing \%} = \frac{\text{Missing Count}}{\text{Expected Count}}$$ | Numeric (percentage, 0–100%) | **Checks Data Integrity.** Ensures you aren't forecasting on a "Swiss cheese" dataset. If > 0%, impute or flag as "Data Quality Risk". |
| A1.2 | **Coefficient of Variation (CoV)** | $$CoV = \frac{\sigma}{\mu}$$ | Numeric (ratio, ≥ 0) | **Measures Volatility.** Low (< 0.5): Stable, predictable. High (> 1.0): Highly volatile/erratic. Widen confidence intervals; expect lower forecast accuracy. |
| A1.3 | **Intermittency Ratio** | $$\frac{\text{Count(Zero Demand Periods)}}{\text{Total Periods}}$$ | Numeric (ratio, 0–1) | **Measures Sporadic Demand.** High (> 0.5): Intermittent/Sparse demand.|
| A1.4 | **ADI / CV² Segmentation** | Plot Average Inter-Demand Interval (ADI) vs. CV². | Ordinal Category (Smooth / Erratic / Intermittent / Lumpy) | **Classifies Demand Type:** Smooth (regular), Erratic (high variance, low intermittency), Lumpy (high variance, high intermittency — hardest to forecast). |
| A1.5 | **Spike Factor** | $$\frac{\text{Max(Actuals)}}{\text{Median(Actuals)}}$$ | Numeric (ratio, ≥ 1) | **Detects Historical Extreme Events.** If high, check if the Delivery Month Forecast predicts a similar spike and why. |
| A1.6 | **Demand Spike** | $$\text{Backtest Actuals} > \mu + 5\sigma$$ | Binary (Yes / No) | **Extreme Outlier Detection.** Flags "Black Swan" events. These should likely be treated as outliers and removed from training data. |

### A2 — Trend Analysis

| # | **Test Name** | **Metric / Logic** | **Expected Output Type** | **Objective & Interpretation** |
|---|---|---|---|---|
| A2.1 | **Mann-Kendall Trend Test** | Non-parametric statistical test for monotonic trend. | Nominal Category (Increasing / Decreasing / No Trend) + Numeric (p-value) | **Verifies Directionality.** Returns p-value and direction (increasing/decreasing). If test says "Decreasing" but forecast says "Increasing", flag as "Trend Reversal Risk". |
| A2.2 | **Linear Trend Significance** | OLS Regression: `actual ~ time` (extract Slope & P-value) | Numeric (slope) + Binary (significant Yes / No at p < 0.05) | **Validates Growth/Decline.** Is the trend statistically real or just noise? If P-value < 0.05, the trend is significant. |
| A2.3 | **Recent Growth/Decline** | $$\frac{\text{Mean(Last 3 Months)}}{\text{Mean(Same 3 Months Last Year)}} - 1$$ | Numeric (signed percentage) | **Measures Short-term Momentum.** Detects if the key is currently heating up or cooling down compared to last year. |
| A2.4 | **Acceleration Rate** | 2nd-order coefficient from OLS on Last 3 Months: `actual ~ time + time²` (extract coefficient of time²) | Numeric (regression coefficient) | **Flags Extreme Curvature in L3M.** Large positive = demand is accelerating (growth picking up speed); large negative = decelerating/collapsing. Catches runaway momentum that a linear trend slope alone would understate. |

### A3 — Seasonality Analysis

| # | **Test Name** | **Metric / Logic** | **Expected Output Type** | **Objective & Interpretation** |
|---|---|---|---|---|
| A3.1 | **Seasonality Strength** | STL Decomposition: $$F_s = \max\left(0,\; 1 - \frac{\text{Var(Residual)}}{\text{Var(Seasonal + Residual)}}\right)$$ | Numeric (ratio, 0–1) | **Quantifies Seasonality (0 to 1).** > 0.6: Strong seasonality. Ensure selected model is seasonally aware and delivery month matches historical seasonal index. |
| A3.2 | **LYSM Deviation** | $$\frac{\text{Recent Seasonal Index} - \text{Hist Index}}{\text{Hist Index}}$$ | Numeric (signed percentage) | **Detects Seasonality Shifts.** Is the "Summer Peak" happening earlier or later than usual? (LYSM = Last Year Same Month). |
| A3.3 | **Seasonal Peaks Alignment** | Define peak months as months where the seasonal index exceeds a threshold (e.g., > 1 + 0.5 × StdDev of seasonal indices). Compare the set of peak months in the most recent year vs. the historical average. Report: overlap count, any new peaks, any missing peaks. | Numeric (overlap count) + Nominal (list of new / missing peak months) | **Validates Peak Timing Integrity.** Ensures that high-demand months are occurring in the expected calendar positions. A shift in peak months (e.g., peak moving from October to August) invalidates seasonally-aware models trained on historical patterns and must be flagged before delivery. |

---

#### **A3 — Alternative Methods**

**STL-Based Tests (A3.1 - A3.3) Data Requirements:**
- **Minimum:** 24 months (2 complete seasonal cycles)
- **Recommended:** 36+ months (3+ complete seasonal cycles)
- **Intermittency:** < 50% zero periods
- **Completeness:** < 5% missing values (impute before STL)

**For SKUs with < 24 months of data, use the following Simple Seasonal Indices alternatives:**

| # | **Alternative Test Name** | **Metric / Logic** | **Expected Output Type** | **Objective & Interpretation** |
|---|---|---|---|---|
| **A3.1-ALT** | **Seasonality Strength (Simple)** | For each calendar month _m_, calculate: $$\text{SI}_m = \frac{\text{Mean(Demand}_m\text{)}}{\text{Mean(All Demand)}}$$. Then compute: $$\text{Seasonality Strength} = \frac{\text{StdDev}(\text{SI}_1, \text{SI}_2, ..., \text{SI}_{12})}{\text{Mean}(\text{SI}_1, \text{SI}_2, ..., \text{SI}_{12})}$$ | Numeric (ratio, ≥ 0) | **Simplified Seasonality Metric.** Measures how much monthly indices vary from the mean. > 0.25: Strong seasonality. < 0.15: Weak/no seasonality. Less robust than STL but workable with 12-23 months of data. |
| **A3.2-ALT** | **LYSM Deviation (Simple)** | For delivery month _m_: $$\text{SI}_{\text{recent, }m} = \frac{\text{Demand}_{\text{last occurrence of }m}}{\text{Mean(Last 12M)}}$$,  $$\text{SI}_{\text{hist, }m} = \frac{\text{Mean(All month }m\text{ values)}}{\text{Mean(All Demand)}}$$. Then: $$\frac{\text{SI}_{\text{recent, }m} - \text{SI}_{\text{hist, }m}}{\text{SI}_{\text{hist, }m}}$$ | Numeric (signed percentage) | **Simple Index Drift.** Compares the most recent occurrence of the month to the historical average for that month. Works with as little as 13 months (requires at least 2 observations of the delivery month). |
| **A3.3-ALT** | **Seasonal Peaks Alignment (Simple)** | Calculate seasonal index for each month: $$\text{SI}_m = \frac{\text{Mean(Demand}_m\text{)}}{\text{Mean(All Demand)}}$$. Define peak threshold as $$\text{SI} > 1 + 0.5 \times \text{StdDev(all SI)}$$. Compare current year's peak months (using last 12 months) vs. all available history's peak months. | Numeric (overlap count) + Nominal (list of new / missing peak months) | **Simplified Peak Detection.** Uses simple averaging instead of STL decomposition. Requires minimum 12 months to identify a full seasonal cycle, ideally 18+ months to compare "recent" vs "historical" patterns. |

---

### A4 — Regime & Shift Detection

| # | **Test Name** | **Metric / Logic** | **Expected Output Type** | **Objective & Interpretation** |
|---|---|---|---|---|
| A4.1 | **Change Point Detection** | Algorithm (e.g., Ruptures) or $$\text{Rolling Mean Diff} > \text{Threshold}$$ | Binary (detected Yes / No) + Numeric (count of change points) + Nominal (dates) | **Detects Structural Breaks.** Finds exact dates where baseline demand shifted permanently (e.g., new regulation, competitor entry). |
| A4.2 | **Demand Mean Shift** | $$\frac{\mu(\text{Last 3M}) - \mu(\text{Prior 3M})}{\mu(\text{Prior 3M})}$$  | Numeric (signed percentage) | **Short-term Level Shift.** Did demand suddenly jump or crash in the last quarter? Critical for deciding if recent history > long history. |
| A4.3 | **Demand Variance Shift** | $$\frac{\text{Var(Last 3M)}}{\text{Var(Prior 3M)}}$$  | Numeric (ratio, > 0) | **Volatility Regime Change.** Has the key suddenly become harder to predict (more volatile) than it used to be? |
| A4.4 | **Trend Drift** | Slope(L6M ) vs. Long-term Slope | Numeric (slope difference) | **Trend Decoupling.** Is the recent trend diverging from the long-term historical trend? |

---

## B. Backtest Forecast Evaluation

_Tests that evaluate the quality of backtest forecasts against actuals._

### B1 — Accuracy Metrics

| # | **Test Name** | **Metric / Logic** | **Expected Output Type** | **Objective & Interpretation** |
|---|---|---|---|---|
| B1.1 | **WMAPE** | $$\frac{\sum |A - F|}{\sum A}$$ | Numeric (percentage, ≥ 0) | **Weighted Absolute Error.** Primary accuracy metric. Lower is better. Robust to intermittent zeroes unlike MAPE. |
| B1.2 | **Baseline Lift** | $$\frac{\text{WMAPE(Selected Model)}}{\text{WMAPE(Naive / Moving Avg)}}$$ | Numeric (ratio, > 0) | **Value-Add Check.** If ratio > 1, the complex ML model performs _worse_ than a simple baseline. Ratio < 1 = model beats naive. **Action:** Fallback to baseline if ratio > 1. |
| B1.3 | **Beat Seasonal Naive** | Compare model WMAPE vs. Seasonal Lag Forecast WMAPE | Binary (Yes / No) | **Seasonality Check.** Does the model capture seasonality better than simply copying last year's data? |
| B1.4 | **Models Beating Naive Count** | $$\frac{\text{Count(Models where WMAPE}_m < \text{WMAPE}_{\text{Naive}}\text{)}}{\text{Total Models}}$$ | Numeric (ratio, 0–1) | **Ensemble Health Check.** What fraction of the pipeline's model pool actually beats a naive baseline on the backtest period? A low ratio (e.g., < 30%) signals that the key is inherently hard to forecast and model selection risk is high. |
| B1.5 | **Models Beating Seasonal Naive Count** | $$\frac{\text{Count(Models where WMAPE}_m < \text{WMAPE}_{\text{Seasonal Naive}}\text{)}}{\text{Total Models}}$$ | Numeric (ratio, 0–1) | **Seasonality Capture Health.** What fraction of models beat last-year's-same-month baseline? Complements B1.4 — a model pool that beats naive but not seasonal naive is ignoring seasonal structure. |

### B2 — Bias & Consistency

| # | **Test Name** | **Metric / Logic** | **Expected Output Type** | **Objective & Interpretation** |
|---|---|---|---|---|
| B2.1 | **Forecast Bias (MPE)** | $$\frac{\sum(A - F)}{\sum A}$$ | Numeric (signed percentage) | **Directional Error.** Positive: Systematic under-forecasting (lost sales risk). Negative: Systematic over-forecasting (inventory risk). |
| B2.2 | **Rolling Bias Count** | Count of consecutive windows where bias sign (+/−) is consistent. | Numeric (integer count) | **Detects "Stubborn" Errors.** If a model has been over-forecasting for 6 months straight, it is structurally broken. |
| B2.3 | **Error Volatility** | $$\text{StdDev}(\text{Rolling WMAPE})$$ | Numeric (standard deviation, ≥ 0) | **Reliability Score.** Does the model perform consistently, or does its accuracy fluctuate wildly from month to month? |
| B2.4 | **Error Deterioration** | Slope of Regressed Rolling WMAPE | Numeric (regression slope) | **Model Decay.** Is the model getting worse over time? If slope > 0, the model is degrading and needs retraining. |

### B3 — Model Diagnostics

| # | **Test Name** | **Metric / Logic** | **Expected Output Type** | **Objective & Interpretation** |
|---|---|---|---|---|
| B3.1 | **Residual Autocorrelation** | Ljung-Box Test on Residuals (p-value) | Binary (significant Yes / No at p < 0.05) + Numeric (p-value) | **Checks Information Leakage.** If significant (p < 0.05), the model left patterns in the error term — it "missed" a signal that could improve accuracy. |

---

## C. Delivery Forecast Validation

_Sanity checks on the upcoming delivery forecast._

### C1 — Magnitude & Sanity Checks

| # | **Test Name** | **Metric / Logic** | **Expected Output Type** | **Objective & Interpretation** |
|---|---|---|---|---|
| C1.1 | **Forecast Drift (Value)** | $$\frac{\text{Delivery Forecast} - \text{Mean(Last 3M Actuals)}}{\text{Mean(Last 3M Actuals)}}$$ | Numeric (signed percentage) | **Sanity Check on Magnitude.** Is the forecast predicting a massive jump (+50%) or drop compared to recent history? If yes, flag for review. |
| C1.2 | **Logical Bounds** | Forecast < 0 OR Forecast > Max(Historical Actuals) × _k_ | Binary (Pass / Fail) | **Physics Check.** Negative sales? Impossible. Sales far exceeding the all-time high? Highly unlikely without justification. |
| C1.3 | **Delivery Z-Score** | $$\frac{\text{Delivery} - \mu(\text{Hist})}{\sigma(\text{Hist})}$$ | Numeric (z-score, unbounded) | **Statistical Outlier Check.** How many standard deviations away is the forecast from history? > 2 flags a prediction interval breach; > 3 is highly suspicious. |
| C1.4 | **Forecast Jump vs. Last Year** | $$\frac{F_{\text{delivery}} - \text{Actual}_{\text{LY}}}{\text{Actual}_{\text{LY}}}$$ | Numeric (signed percentage) | **Year-Over-Year Reality.** Are we predicting to double last year's sales? If so, verify with marketing/business. |

---

## D. Combined

_Tests that compare across actuals, backtest forecasts, and delivery forecasts._

### D1 — Directional & Trend Consistency

| # | **Test Name** | **Metric / Logic** | **Expected Output Type** | **Objective & Interpretation** |
|---|---|---|---|---|
| D1.1 | **Directional Mismatch** | Trend Test (History) = Down AND Forecast Trend (Future) = Up | Binary (Mismatch / Aligned) | **Conflicting Signals.** The model is betting on a turnaround. Flag to ensure it's driven by a known feature (e.g., promotion) and not noise. |
| D1.2 | **Growth Rate Shift** | Backtest Growth Rate vs. Delivery Implied Growth | Numeric (signed percentage difference) | **Momentum Consistency.** If history was growing at +5% but delivery predicts +20%, flag as "Aggressive Forecast". |
| D1.3 | **Seasonal Consistency** | Correlation(Backtest Seasonal Index vs. Delivery) | Numeric (correlation coefficient, −1 to 1) | **Pattern Validation.** Does the delivery month shape match the historical seasonal shape? |

### D2 — Distribution & Level Shifts

| # | **Test Name** | **Metric / Logic** | **Expected Output Type** | **Objective & Interpretation** |
|---|---|---|---|---|
| D2.1 | **Forecast Mean Shift** | $$\frac{\mu(\text{Delivery}) - \mu(\text{Backtest})}{\mu(\text{Backtest})}$$ | Numeric (signed percentage) | **Level Jump Alert.** Is the forecast predicting a massive change in volume compared to the backtest period? |
| D2.2 | **Forecast Variance Shift** | $$\frac{\text{Var(Delivery)}}{\text{Var(Backtest)}}$$ | Numeric (ratio, > 0) | **Confidence Shift.** Is the future forecast much flatter (or wigglier) than the model's past behavior? |

---

## D4. Delivery Model Selection Validation

_Tests that validate whether the pipeline's selected delivery model is trustworthy, using backtest evidence. All tests here use backtest-period actuals and forecasts as the evidence base, but are triggered by and scoped to the model the DAG selected for the delivery month._

| # | **Test Name** | **Metric / Logic** | **Expected Output Type** | **Objective & Interpretation** |
|---|---|---|---|---|
| D4.1 | **Delivery Model Beats Naive (Backtest)** | $$\text{WMAPE}_{\text{delivery model, backtest}} < \text{WMAPE}_{\text{Naive, backtest}}$$ | Binary (Yes / No) | **Minimum Credibility Bar.** Does the pipeline's chosen delivery model at least beat a naive forecast on the backtest period? If No, the model has no earned right to be trusted on the delivery month. Flag as "High Risk Selection". |
| D4.2 | **Delivery Model Beats Seasonal Naive (Backtest)** | $$\text{WMAPE}_{\text{delivery model, backtest}} < \text{WMAPE}_{\text{Seasonal Naive, backtest}}$$ | Binary (Yes / No) | **Seasonality Capture Bar.** Does the selected delivery model outperform last-year's-same-month on the backtest? Failure here means the model is not capturing seasonal structure — a critical gap for seasonal SKUs. |
| D4.3 | **Delivery Model vs. Backtest Pool Median** | $$\text{WMAPE}_{\text{delivery model, backtest}} < \text{Median}(\text{WMAPE}_{\text{all models, backtest}})$$ | Binary (Yes / No) | **Relative Pool Rank Check.** Is the selected model at least better than the median model in the pipeline's own ensemble on the backtest months? If it falls below median, the DAG's selection logic may be miscalibrated for this key. |
| D4.4 | **Delivery Model vs. ABC Segment Benchmark** | $$\text{WMAPE}_{\text{delivery model, backtest}} < \overline{\text{WMAPE}}_{\text{same ABC segment, backtest}}$$ | Binary (Yes / No) | **Segment-Relative Performance.** Compares the selected model's backtest error against the average backtest error of all keys in the same Pareto (A/B/C) segment. A-class keys have tighter tolerances; failing this check for an A-class key is a high-priority flag. |
| D4.5 | **Backtest Selection Agreement** | $$\text{Best Model (Backtest DAG)} == \text{Selected Model (Delivery DAG)}$$ | Binary (Yes / No) | **Pipeline Stability Check.** Did the same DAG select the same model for backtest as it did for delivery? Disagreement (e.g., ARIMA selected for backtest but XGBoost for delivery) means the selection is sensitive to the small data window difference and should be treated with lower confidence. |
| D4.6 | **Delivery Model Directional Accuracy (Backtest)** | For each of the _n_ backtest months (typically n = 3): check if sign(F − F_prev) == sign(A − A_prev). Report score as k/n (0 to n correct). | Numeric (ordinal score, k/n e.g. 0/3 to 3/3) | **Directional Skill Score.** Did the selected model correctly call whether demand went up or down in each backtest month? A model that is accurate on magnitude but wrong on direction is dangerous for procurement decisions. Score < 2/3 should be flagged. |
