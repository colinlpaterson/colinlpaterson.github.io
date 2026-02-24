---
layout: post
title: "Loan Portfolio Duration and WAL in R Using the cfit Package"
date: 2026-02-23
categories: [r-programming, finance]
excerpt: "How to measure the duration and WAL of a loan portfolio using the cfit package."
---

## The Problem

Interest Rate Risk (IRR) management is a core function of banking. Financial institutions must understand how changes in market rates affect the value and timing of their asset cash flows relative to their liabilities. Key metrics used to manage IRR include:

- **Macaulay Duration**: the weighted average time until you receive all cash flows, measured in years. Each cash flow (principal and interest payment) is weighted by its present value as a percentage of the loan's total present value.
- **Modified Duration**: measures the percentage change in price (present value) for a 1% change in interest rates. It's the practical, albeit crude tool to manage interest rate risk.
- **Analytical Convexity**: measures the curvature of the price-yield relationship. It captures how duration itself changes as interest rates change.
- **Weighted Average Life (WAL)**: measures the average time (in years) for principal to repaid (no discounting).

Understanding a loan pool's duration, convexity, and WAL enables management to evaluate balance sheet sensitivity, assess duration gaps, and make informed asset allocation decisions. In practice, these calculations are often outsourced to third-party vendors or not performed at the homogeneous pool level due to computational complexity.

## The Solution: cfit::calculate_duration() and cfit::calculate_wal()

`calculate_duration()` accepts a cash flow data frame, generated from cfit:: calculate_cash_flows() and calculates Macaulay Duration, Modified Duration, and Analytical Convexity. `calculate_wal() accepts the same cash flow data frame and returns the WAL.

**Benefits**:
- **Transparent** - Open-source methodology documented in code and help files, no black boxes
- **Reproducible** - Same configuration produces identical results, making outputs audit-ready
- **Scalable** - The function can be applied to many cash flow data frames without having to rebuild the calculations each time. This facilitates comparing different loan portfolios.

## Basic Usage
Using calculate_cash_flows(), forecast a loan portfolio's lifetime cash flows. Then use calculate_duration() to measure duration and convexity. Example below.

```r
# Duration and Convexity Analysis for Fixed-Rate Loan Portfolios
# This example demonstrates how to:
#   1. Calculate Macaulay Duration, Modified Duration, and Analytical Convexity (fixed rate portfolios)
#   2. Estimate portfolio value changes under interest rate shocks
#   3. Quantify the impact of positive convexity on valuation
# Key Concepts:
# - Modified Duration: First-order approximation of price sensitivity (% change per 1% rate move)
# - Analytical Convexity: Second-order adjustment for non-linear price-yield relationship
# - Positive Convexity: Reduces losses when rates rise, amplifies gains when rates fall

library(cfit)
# Sample loan portfolio snapshot: Three auto loans totaling $90K with rates 5.5-6.5% and terms 3-5 years
loan_portfolio <- data.frame(
  LOAN_ID = c("L001", "L002", "L003"),
  balance = c(25000, 50000, 15000),
  current_interest_rate = c(0.0599, 0.0649, 0.0549),
  months_to_maturity = c(60, 48, 36),
  eff_date = as.Date("2025-01-01")
)

# Configure cash flow parameters
config <- list(
  cpr_vec = c("default" = 0.05),           # 5% CPR assumption
  credit_cost_vec = c("default" = 0.01),   # 1% annual credit cost
  servicing_fee = 0.0025,                  # 25 bps servicing fee
  return_monthly_totals = TRUE             # Return aggregated monthly totals
)

# Generate cash flows
results <- calculate_cash_flows(loan_portfolio, config)

# Using cash flows from previous example
cash_flows <- results$loan_cash_flows

# Calculate duration metrics
duration_results <- calculate_duration(
  loan_cash_flows = cash_flows,
  include_convexity = TRUE
)

print(duration_results)
#portfolio_pv macaulay_duration modified_duration analytical_convexity
#      88867.3           1.64498          1.636546             3.951401

# Analytical Convexity impact on portfolio value examples:

#Scenario 1: Rates rise 3% - PV decreases
pct_change_up_300 <- (-duration_results[[3]] * 0.03) 
portfolio_pv_up_300 <- duration_results[[1]] + duration_results[[1]] * pct_change_up_300 
pct_change_up_300_w_convexity <- (-duration_results[[3]] * 0.03) + (0.5 * duration_results[[4]] * 0.03^2)
portfolio_pv_up_300_w_convexity <- duration_results[[1]] + duration_results[[1]] * pct_change_up_300_w_convexity

#Scenario 2: Rates fall 3% - PV increases
pct_change_down_300 <- (-duration_results[[3]] * -0.03) 
portfolio_pv_down_300 <- duration_results[[1]] + duration_results[[1]] * pct_change_down_300 
pct_change_down_300_w_convexity <- (-duration_results[[3]] * -0.03) + (0.5 * duration_results[[4]] * (-0.03)^2)
portfolio_pv_down_300_w_convexity <- duration_results[[1]] + duration_results[[1]] * pct_change_down_300_w_convexity 

# Results Table

results_summary <- data.frame(
  Scenario = c("Base Case", "Rates Up 300bp (Duration Only)", 
               "Rates Up 300bp (Duration + Convexity)", 
               "Rates Down 300bp (Duration Only)", 
               "Rates Down 300bp (Duration + Convexity)"),
  Portfolio_Value = c(
    duration_results[[1]],
    portfolio_pv_up_300,
    portfolio_pv_up_300_w_convexity,
    portfolio_pv_down_300,
    portfolio_pv_down_300_w_convexity
  ),
  Change_from_Base = c(
    0,
    portfolio_pv_up_300 - duration_results[[1]],
    portfolio_pv_up_300_w_convexity - duration_results[[1]],
    portfolio_pv_down_300 - duration_results[[1]],
    portfolio_pv_down_300_w_convexity - duration_results[[1]]
  ),
  Pct_Change = c(
    0,
    pct_change_up_300 * 100,
    pct_change_up_300_w_convexity * 100,
    pct_change_down_300 * 100,
    pct_change_down_300_w_convexity * 100
  )
)

# Calculate convexity benefit
convexity_benefit <- data.frame(
  Scenario = c("Rates Up 300bp", "Rates Down 300bp"),
  Duration_Only_Value = c(portfolio_pv_up_300, portfolio_pv_down_300),
  With_Convexity_Value = c(portfolio_pv_up_300_w_convexity, portfolio_pv_down_300_w_convexity),
  Convexity_Benefit = c(
    portfolio_pv_up_300_w_convexity - portfolio_pv_up_300,
    portfolio_pv_down_300_w_convexity - portfolio_pv_down_300
  ),
  Benefit_Pct = c(
    (portfolio_pv_up_300_w_convexity - portfolio_pv_up_300) / portfolio_pv_up_300 * 100,
    (portfolio_pv_down_300_w_convexity - portfolio_pv_down_300) / portfolio_pv_down_300 * 100
  )
)

print("Portfolio Valuation Under Rate Scenarios:")
print(results_summary)
#                                       Scenario Portfolio_Value Change_from_Base Pct_Change
# 1                               Base Case        88867.30            0.000   0.000000
# 2          Rates Up 300bp (Duration Only)        84504.24        -4363.063  -4.909639
# 3   Rates Up 300bp (Duration + Convexity)        84662.26        -4205.046  -4.731825
# 4        Rates Down 300bp (Duration Only)        93230.36         4363.063   4.909639
# 5 Rates Down 300bp (Duration + Convexity)        93388.38         4521.081   5.087452

print("Convexity Impact Summary:")
print(convexity_benefit)
#           Scenario Duration_Only_Value With_Convexity_Value Convexity_Benefit Benefit_Pct
# 1   Rates Up 300bp            84504.24             84662.26          158.0176   0.1869937
# 2 Rates Down 300bp            93230.36             93388.38          158.0176   0.1694916

#Conclusion
cat("\n=== KEY FINDINGS ===\n\n")
cat("1. POSITIVE CONVEXITY IMPACT:\n")
cat(sprintf("   • Reduces loss by $%.2f (%.3f%%) when rates rise 300bp\n", 
            convexity_benefit$Convexity_Benefit[1],
            convexity_benefit$Benefit_Pct[1]))
cat(sprintf("   • Increases gain by $%.2f (%.3f%%) when rates fall 300bp\n\n", 
            convexity_benefit$Convexity_Benefit[2],
            convexity_benefit$Benefit_Pct[2]))
cat("2. WHY AUTO LOANS HAVE POSITIVE CONVEXITY:\n")
cat("   • Limited prepayment optionality (penalties/fees common)\n")
cat("   • Borrowers less rate-sensitive than mortgage holders\n")
cat("   • Smaller loan balances reduce refinancing incentives\n\n")
cat("3. CONTRAST WITH MORTGAGES:\n")
cat("   • Mortgages typically have NEGATIVE convexity\n")
cat("   • Borrowers refinance aggressively when rates fall\n")
cat("   • This creates asymmetric risk (losses accelerate, gains dampen)\n\n")
cat("4. RISK MANAGEMENT IMPLICATIONS:\n")
cat("   • Positive convexity provides natural interest rate protection\n")
cat("   • Duration matching alone underestimates this benefit\n")
cat("   • Auto loan portfolios can offset mortgage convexity risk\n\n")

```
Finally, calculate WAL to understand principal repayment timing

```r
# Calculate weighted average life
wal_results <- calculate_wal(
  loan_cash_flows = cash_flows
)

print(wal_results)
# portfolio_wal (years)
#      1.78

cat("\nWeighted Average Life (WAL) Interpretation:\n")
cat(sprintf("• WAL: %.2f years\n", wal_results[[1]]))
cat(sprintf("• Macaulay Duration: %.2f years\n", duration_results[[2]]))
cat(sprintf("• WAL > Duration by %.2f years\n\n", 
            wal_results[[1]] - duration_results[[2]]))
cat("Why WAL > Duration:\n")
cat("• WAL: Simple average of principal payments (no discounting)\n")
cat("• Duration: Present-value weighted (distant cash flows worth less)\n")
cat("• Both measure 'average life' but Duration is more precise for IRR\n")

``` 

## Calculations

**Macaulay Duration** - For each monthly payment:
- Calculate its present value (PV)
- Multiply by the time (in years) when you receive it
- Sum all these weighted times
- Divide by the total PV of the loan

The calculation recognizes that much of the total cash flows is received in the earlier years. Higher coupon loans have shorter duration than lower coupon loans of the same maturity because you receive more cash flows sooner through interest payments.

**Modified Duration** Measures the slope of the price-yield relationship (first derivative: rate of change). It estimates the percentage in portfolio economic value given the change in interest rates - the steeper the slope, the more sensitive the portfolio is to rate changes.
- Equals Macaulay Duration / (1 + yield/12), where yield is the current loan yield and 12 is monthly payments. 

**Analytical Convexity**: Measures the curvature of the price-yield relationship.
- Why "Analytical"?: It assumes cash flows stay constant (ignores prepayment changes). 
    - This answers: "What convexity would this portfolio have if borrowers didn't change prepayment behavior?"
- The **Math**: For each monthly payment, we calculate:
  - PV_t × t × (t + 1/12), where:
    - t = time in years
    - 1/12 = monthly compounding adjustment
  - Sum these, then divide by: PV × (1 + y/12)²
- **Key Insight**: Positive convexity means your losses are smaller than duration predicts when rates rise, and gains are larger when rates fall.

**Note**: `calculate_duration()` computes *analytical* (fixed cash flow) convexity, not *effective* (option-adjusted) convexity. For portfolios with significant prepayment optionality (like mortgages), analytical convexity may overstate the true convexity benefit.
    
**Weighted Average Life**
- WAL = Σ(Principal_t × t) / Total Principal
  - Principal_t = Principal payment received at time t
  - t = Time period (in years) when the principal payment occurs
  - Total Principal = Original loan balance
  
## When to Use Each Metric

**Modified Duration**: 
- Primary tool for IRR management
- Use for: Setting hedging strategies, calculating gap limits
- Best for: Small rate changes (±1-2%)

**Analytical Convexity**: 
- Refines duration estimates for large rate moves
- Use for: Stress testing, portfolio comparison
- Best for: Understanding asymmetric risk profiles

**WAL**: 
- Pure timing metric (no rate sensitivity)
- Use for: Liquidity planning, participation pricing
- Best for: Comparing portfolios with similar rates but different amortization  

## Real-World Use Cases

**1. Loan Participation Evaluation**
Analyze potential loan pool purchases by comparing their respective duration, analytical convexity, and WALs.

**2. ALM Gap Analysis**
If your loan pool has a duration of 5 years and is funded with deposits with a duration of 2 years, there exists a 3-year duration gap. This could be a problem in an rising interest environment where deposits will be repriced higher, thereby compression net interest rate margin (NIM).

**3. Strategic Planning and Budgeting**
Macaulay Duration, Modified Duration, and WAL help inform hedging, pricing, and planning decisions, such as how much long duration assets should we acquire given our modified duration and ALM gap limits.

## Data Requirements

### calculate_duration()

**Input**: A data frame output from `calculate_cash_flows()` with monthly projections.

**Required columns:**
- `LOAN_ID` - Unique loan identifier
- `rate` - Annual interest rate (e.g., 0.06 for 6%), as standardized in the output of calculate_cash_flows()
- `eff_date` - Effective/snapshot date
- `month` - Projection index (month = 1 corresponds to time 0)
- `total_payment` OR `investor_total` - Monthly Cash flow amounts

**Optional Arguments**
- discount_rate - If NULL (default), the function uses each loan's rate from the loan_cash_flows data. If a scalar, it applies that rate uniformly across all loans. 
- include_convexity - If TRUE, calculates and includes analytical convexity in the output. Default is FALSE.

### calculate_wal()

**Input**: A data frame with one row per loan per month containing these columns:
- `LOAN_ID`
- `eff_date`
- `date`
- `month`
- `total_principal (default)` OR `investor_principal (optional)`

## Get Started

Install the latest version from GitHub:
```r
devtools::install_github("CommunityFIT/cfit")
library(cfit)

# View detailed documentation
?calculate_duration
?calculate_wal
```

**Resources:**
- [cfit GitHub repository](https://github.com/CommunityFIT/cfit)
- [Full documentation](https://github.com/CommunityFIT/cfit#readme)
- [Report issues or request features](https://github.com/CommunityFIT/cfit/issues)

**Next Steps:**
- Try it with your own loan portfolio data
- Combine with `calculate_duration()` and `calculate_wal()` for comprehensive risk analysis
- Star the repo if you find it useful
- Contribute improvements or report bugs

---

*Part of the [CommunityFIT](https://github.com/CommunityFIT) initiative - open-source computational finance tools for community financial institutions.*
