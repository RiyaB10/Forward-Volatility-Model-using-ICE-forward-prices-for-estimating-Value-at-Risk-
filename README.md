# Forward-Volatility-Model-using-ICE-forward-prices-for-estimating-Value-at-Risk-

⚙️ Overview

This project develops a comprehensive forward volatility and VaR modeling framework based on Samuelson effect.
It models forward contract price dynamics from ICE data, builds empirical volatility and correlation surfaces, and computes the portfolio Value at Risk (VaR) across NP15 and SP15 hubs.
It computes:

✔ Monthly unhedged exposure (MWh)
✔ Joins exposure with market volatility & forward prices
✔ Computes 95% one-month VaR by TOU (HLH & LLH)
✔ Generates a single portfolio-level VaR number
✔ Produces diagnostic charts (Samuelson effect, VaR curves, exposure curves)

[Everything is implemented using PySpark for scalability and Matplotlib/Pandas for visualization]

🧱 Model Workflow
Step 1 – ICE Price Cleaning

- Reads raw ICE market data
- Filters last 3 years of effective data (only accounting for recent price trends)
- Derives:
  - hub (NP15/SP15) (major hub for California)
  - tou (HLH/LLH) (High load and low load hours)
  - delivery_month
  - tenor_m (time-to-delivery in months)
- Saves cleaned dataset 

Step 2 – Return Calculation

- Computes log returns across quote dates for each delivery contract. 
- Generates ice_same_delivery_returns table.

Step 3 – Empirical Volatility Estimation

- Groups return by hub, TOU, month, and tenor.
- Calculates empirical volatility (σ) for each tenor-month pair.
- Stores results in different tables for accounting intermediate results.

Step 4 – Cross-Tenor Correlation

- Calculates correlation (ρ) between returns for tenor pairs (Δτ).
- Fits exponential decay ρ(Δτ) = exp(-λΔτ) to derive λ parameters.
- Saves fitted correlations.

Step 5 – Volatility Term-Structure Fitting

- Fits both Exponential and Power-law σ(τ) models:
  - Exponential: σ(τ) = σ∞ + (σ₀ − σ∞) e^(−kτ)
  - Power-law: σ(τ) = σ∞ + aτ^(−b)
- Chooses best fit per hub/TOU and stores parameters.

Step 6 – Seasonality Adjustment - to account for correlation between - considering how months move together

- Computes median ratios of empirical-to-fitted volatilities by month.
- Produces monthly seasonality multipliers (S_m).
- Saves results.

Step 7 – Alpha Calibration (Bootstrap Validation) - Optional - to verify if the model results match the historical - sometimes the model needs a scalar to match the historical pattern

- Uses bootstrapping on rolling delivery paths to compare realized volatility vs model variance.
- Computes α correction = √(Realized / Modeled)
- Saves per-hub calibration

Step 8 – Volatility Table Generation

- Creates volatility tables across multiple time horizons:

| Table                | Description                                         |
| -------------------- | --------------------------------------------------- |
| `ice_vol_monthly`    | Monthly (1-month) volatility                        |
| `ice_vol_quarter`    | Quarterly (3-month) strip volatility                |
| `ice_vol_semiannual` | Semiannual (H1, H2) volatility                      |
| `ice_vol_annual`     | Annual (FY) and ATC volatility (HLH + LLH weighted) |

Each table includes: hub | strip_type | tou | quote_date | period | volatility_annualized | volatility_period

Step 9 – Houlry Data Integration and VaR Computation

- Pulls hourly data for each energy contract of the company
- Aggregates by month and TOU:
  Unhedged_MWh = WHLoadTotal × (1 − hedge_pct)
- Joins with:
- Volatility table (ice_vol_monthly)
- Calculates monthly VaR:
  VaR = Unhedged_MWh × Price × Volatility × 1.645
where 1.645 = 90% confidence Z-score.

Outputs
| Table                                                    | Description                        |
| -------------------------------------------------------- | ---------------------------------- |
| `ice_clean_prices`                                       | Cleaned forward prices             |
| `ice_same_delivery_returns`                              | Daily returns                      |
| `ice_emp_vol_tau_moy`                                    | Empirical volatilities             |
| `ice_param_sigma`                                        | Fitted volatility parameters       |
| `ice_param_rho`                                          | Fitted correlation parameters      |
| `ice_vol_monthly` / `ice_vol_quarter` / `ice_vol_annual` | Volatility tables                  |
| `ermp_var`                                               | Final monthly Value at Risk by TOU |

Model Formulas Summary

σ(τ) = σ∞ + aτ^(−b)     or     σ(τ) = σ∞ + (σ₀ − σ∞)e^(−kτ)
ρ(Δτ) = e^(−λΔτ)
σ(t,month) = σ(τ) × S_m

Value at Risk:
VaR = Unhedged_MWh × Price × Volatility × Z

Core Dependencies
- pyspark
- numpy
- pandas
- matplotlib
- scipy

Assumptions
- ICE volatilities de-annualized to 1-month using √t.
- No inflation or alpha-scaling used unless bootstrapped calibration is applied.
- Hubs analyzed: NP15, SP15.
- Correlation (HLH–LLH) for ATC = 0.8.
- Forward price reference: latest ICE quote.


