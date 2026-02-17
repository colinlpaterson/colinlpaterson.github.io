---
layout: post
title: "Forecasting Loan Portfolio Cash Flows in R Using the cfit Package"
date: 2026-02-17
categories: [r-programming, finance]
excerpt: "How to accurately forecast loan portfolio cash flows with transparent, reproducible calculations using the cfit package."
---

## The Problem

Community Financial Institutions (FI) need to forecast loan portfolio cash flows for participation loan pool analysis, interest rate risk modeling, and strategic planning. Many FIs rely on brittle Excel models that are hard to audit, prone to errors, and lack transparency in their calculations.

## The Solution: cfit::calculate_cash_flows()

`calculate_cash_flows()` generates loan-level cash flows with user-defined prepayment and credit cost assumptions, then optionally aggregates monthly totals for portfolio yield analysis.

**Benefits:**
- **Transparent** - Open-source methodology documented in code and help files, no black boxes
- **Reproducible** - Same configuration produces identical results, making outputs audit-ready
- **Flexible** - Handles tier-based assumptions, multiple fee structures, and different accounting treatments

## Basic Usage
```r
# Install dependencies (FinCal is not a cfit dependency)
devtools::install_github("felixfan/FinCal")
library(FinCal)
library(cfit)

# Sample loan portfolio snapshot
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
  credit_cost_vec = c("default" = 0.01),   # 1% annual net charge-off rate
  servicing_fee = 0.0025,                  # 25 bps annual servicing fee
  return_monthly_totals = TRUE             # Return aggregated monthly totals
)

# Generate cash flows
results <- calculate_cash_flows(loan_portfolio, config)

# View aggregated monthly totals
head(results$monthly_totals)

# Calculate net portfolio yield using investor_total (after fees and losses)
pool_cfs <- data.frame(
  date = results$monthly_totals$date,
  amount = results$monthly_totals$investor_total  # Net cash flow to owner
)

portfolio_yield <- yield.actual(
  cf = pool_cfs,
  pv = sum(loan_portfolio$balance),
  start_date = min(loan_portfolio$eff_date),
  compounding = "monthly"
)

print(paste("Net Portfolio Yield:", round(portfolio_yield * 100, 2), "%"))

# Compare with gross yield using total_payment (before fees and losses)
pool_cfs_gross <- data.frame(
  date = results$monthly_totals$date,
  amount = results$monthly_totals$total_payment  # Gross cash flow
)

gross_yield <- yield.actual(
  cf = pool_cfs_gross,
  pv = sum(loan_portfolio$balance),
  start_date = min(loan_portfolio$eff_date),
  compounding = "monthly"
)

print(paste("Gross Portfolio Yield:", round(gross_yield * 100, 2), "%"))
```

## Key Configuration Options

The function accepts a `config` list to customize calculations:

**Column Mapping** (adapt to your data structure):
- `col_loanid`, `col_balance`, `col_rate`, `col_term`, `col_start_date` - Required columns
- `col_tier` - Optional tier classification for differentiated credit loss and prepay speed assumptions
- `col_monthly_payment` - Use actual monthly payment or let function calculate
- `col_orig_balance` - Original balance for origination fee calculation

**Economic Assumptions**:
- `cpr_vec` - Annual CPR by tier (e.g., `c("A" = 0.05, "B" = 0.10)`)
- `credit_cost_vec` - Annual net charge-off rates by tier (e.g., `c("A" = 0.008, "B" = 0.015)`)
- `pd_vec` / `lgd_vec` - Alternative approach: specify PD × LGD instead of direct credit costs

**Fee Structure** (investor/servicer economics):
- `servicing_fee` - Annual servicing fee rate (default: 0.0025 = 25 bps)
- `annual_reporting_fee` - Additional reporting fee (default: 0.00)
- `origination_fee` - Upfront fee amortized monthly (default: 0.00)
- `investor_share` - Ownership percentage (default: 1.0 = 100%)

**Accounting Treatment**:
- `credit_loss_reduces_interest` - Apply losses to investor cash flows (TRUE, default - participation accounting) or only to principal (FALSE - balance sheet accounting)
- `interest_on_starting_balance` - Accrue interest on starting balance (TRUE) or adjusted balance after prepay/losses (FALSE, default and more conservative)

**Output Options**:
- `return_monthly_totals` - Return aggregated monthly totals (default: FALSE)
- `monthly_totals_group_vars` - Group totals by additional columns like tier (default: NULL)

## What Gets Calculated?

The function returns loan-level cash flows with two important resulting columns:

**`total_payment`** - Gross cash flow from the loan (gross_interest + total_principal) before any deductions. Use this for gross portfolio yield calculations.

**`investor_total`** - Net cash flow to the owner after deducting servicing fees, reporting fees, origination fees, and credit losses (if `credit_loss_reduces_interest = TRUE`), then applying the investor share. Use this for net portfolio yield calculations - the true economic return.

## Real-World Use Cases

**1. Loan Participation Evaluation**
Analyze potential loan pool purchases by modeling cash flows under various prepayment and credit cost scenarios. Critical for comparing portfolio yields against alternative investments and conducting due diligence on participation opportunities.

**2. ALCO Interest Rate Risk Analysis**
Project monthly cash flows to calculate portfolio duration, weighted average life, and convexity. Essential for measuring interest rate risk exposure and ensuring asset-liability management stays within policy limits.

**3. Strategic Planning and Budgeting**
Forecast net interest income from the loan portfolio under different economic scenarios. Supports accurate multi-year financial planning and stress testing against adverse conditions.

**4. CECL Reserve Calculation**
Generate loan-level cash flow projections incorporating probability of default and loss given default assumptions. Provides the foundation for expected credit loss calculations required under the CECL accounting standard.

## Data Requirements

**Input**: A data frame with one row per loan containing:

**Required columns** (defaults shown, customizable via config):
- `col_loanid` - Loan identifier (default: "LOAN_ID")
- `col_balance` - Current outstanding balance (default: "balance")
- `col_rate` - Interest rate as decimal, e.g., 0.0599 for 5.99% (default: "current_interest_rate")
- `col_term` - Remaining months to maturity from effective date (default: "months_to_maturity")
- `col_start_date` - Effective/snapshot date (default: "eff_date")

**Optional columns**:
- `col_tier` - Risk tier or classification for differentiated assumptions
- `col_monthly_payment` - Contractual monthly payment amount
- `col_orig_balance` - Original loan amount at origination

## Get Started

Install the latest version from GitHub:
```r
devtools::install_github("CommunityFIT/cfit")
library(cfit)

# View detailed documentation
?calculate_cash_flows
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