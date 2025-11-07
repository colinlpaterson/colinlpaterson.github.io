---
layout: post
title: "Calculating Loan Portfolio Yields in R: My Contribution to the FinCal Package"
date: 2024-11-06
categories: [r-programming, finance]
excerpt: "How I built and contributed a function to calculate effective yields for analyzing loan pools."
---

When analyzing loan participation pools, I needed to calculate effective yields using the actual/actual day count convention in order to properly price the pool. The internal rate of return could be done in Excel's built-in functions but I wanted to do the calculation programatically and not be limited by Excel. So I built `yield.actual()` in R - and contributed it to the open-source FinCal package.

## The Problem

A critical metric used to evaluate loan participations is the effective yield, which is the internal rate of return of a series of cash flows (including the purchase price of the pool). I needed a function that could:
- Accept actual calendar dates for when cash flows are expected to be received
- Be flexible to allow irregular cash flows
- Calculate effective yield using actual/actual day count
- Support custom compounding frequencies (semi-annual, annual, etc.). This allows for easy comparison to investments' yields that are quoted on a semi-annually compounded basis

## The Solution: yield.actual()

Working with Claude Sonnet 4.5, I developed a function that calculates the internal rate of return using actual calendar days between cash flows. Claude helped me refine the implementation, ensure proper error handling, and structure the function to follow R package conventions.

The function signature:
```r
yield.actual(cf, pv, start_date = NULL, compounding = "semiannual", 
             max_iter = 100, tol = 1e-8)
```

**Key parameters:**
- `cf`: Data frame with 'date' and 'amount' columns for expected cash flows
- `pv`: Present value - the purchase price or initial investment (positive number)
- `start_date`: Investment date (if NULL, uses first cash flow date)
- `compounding`: "monthly", "semiannual", "quarterly", "annual", or "continuous"

## Real-World Example

Here's how I use it for evaluating a loan pool purchase:
```r
library(FinCal)

# Example: Investor pays $100,000 and receives $8,500/month for 12 months
cash_flows <- data.frame(
  date = seq(as.Date("2025-02-01"), by = "month", length.out = 12),
  amount = rep(8500, 12)
)

# Calculate with monthly compounding
pool_yield_monthly <- yield.actual(
  cf = cash_flows, 
  pv = 100000, 
  start_date = NULL, 
  compounding = "monthly"
)

cat(sprintf("Monthly compounding: %.2f%%\n", pool_yield_monthly * 100))

# Calculate with semiannual compounding (for comparison to Treasuries)
pool_yield_semiannual <- yield.actual(
  cf = cash_flows, 
  pv = 100000, 
  start_date = NULL, 
  compounding = "semiannual"
)

cat(sprintf("Semiannual compounding: %.2f%%\n", pool_yield_semiannual * 100))

```

This allowed me to quickly compare the pool's yield against comparable-duration risk-free investment plus our required credit spread to determine if the investment met our hurdle rate.

(Of course a key determinant of the effective yield is the projected cash flows of the pool; more to come on that.)

## Contributing to Open Source

After using this function successfully in my work, I decided to contribute it to the FinCal package - a package I've used for years for financial calculations. Claude Sonnet 4.5 was instrumental in helping me navigate the contribution process.

**The contribution workflow:**

1. **Forked the FinCal repository** on GitHub
2. **Added the function** with comprehensive documentation following R package standards
3. **Wrote unit tests** to ensure accuracy across various scenarios
4. **Created a pull request** with clear explanation of the functionality
5. **Collaborated with the maintainer** on any requested changes

The maintainer accepted the contribution, and it was merged into FinCal version 0.6.5.

## Why This Matters

**For my workflow:**
- Programmatic effective yield calculations integrated into my R analysis pipeline
- Eliminated manual Excel calculations prone to formula errors
- Reproducible analysis with version-controlled code

**For other analysts:**
- Handles real-world complexities of irregular cash flows
- Flexible compounding conventions for different comparison needs (e.g. fixed income analysis)
- Open source and free to use
- **Can be integrated into automated reporting pipelines**

**For my development:**
- First open-source contribution to a CRAN package
- Learned R package development and testing workflows
- Experience collaborating on public repositories

## Technical Notes

**Day Count Convention:**
Uses actual/actual, meaning actual days between cash flows divided by actual days in the year (365 or 366).

**Compounding Options:**
The function supports multiple compounding conventions to match how different investments quote yields:
- Semi-annual: Standard for Treasuries and most bonds
- Quarterly: Common for some loan products
- Annual: Simple annual effective rate
- Monthly: For consumer loans
- Continuous: For theoretical calculations

**Implementation:**
Uses numerical root-finding to solve for the yield where net present value equals zero, with configurable tolerance and iteration limits.

## Key Takeaways

1. **Identify real gaps** - Build tools that solve actual problems in your workflow
2. **Leverage AI assistance** - Claude helped me write cleaner code and navigate the contribution process
3. **Test thoroughly** - Financial calculations demand accuracy; comprehensive testing is essential
4. **Give back** - Contributing to open source helps others, is in the spirit of credit unions helping each other, builds your professional presence as an finance focused open source developer
5. **Document well** - Clear documentation makes functions actually usable

## Resources

- [FinCal GitHub repository](https://github.com/felixfan/FinCal)


---

