# Paint Pricing Optimization

I've always been curious about how retail chains set prices — and whether 
the answer is usually "someone made a spreadsheet in 2015 and nobody's 
touched it since." This project explores that question using a synthetic 
retail dataset covering 30 stores and 6 paint SKUs over 2.5 years.

The core question: what prices should SKU B (Interior Standard) and 
SKU D (Exterior Premium) be set at to maximise total gross margin 
(Total Revenue minus Total COGS) across the chain?

The short answer: the current prices aren't optimal, and there's 
meaningful money being left on the table.

---
## Data 
The dataset was synthetically generated to simulate realistic retail 
pricing patterns across 30 stores and 6 paint SKUs over 2.5 years 
(January 2021 – June 2023). It is not included in the repository.

Each row represents one store on one day for one SKU, with the 
following columns:

| Column | Description                |
|--------|----------------------------|
| `date` | Transaction date           |
| `store_id` | Unique store identifier    |
| `category` | Product category           |
| `sku` | Product identifier         |
| `price` | Retail price charged          |
| `unit_cost` | The cost of goods sold (COGS) for the item          |
| `beginning_inventory` | Unit inventory count at store opening |
| `sales_units` | Total units sold that day        |

The notebook has all cell outputs saved so the full analysis is 
readable without running the code.

---

## What I found

| | SKU B — Interior Standard | SKU D — Exterior Premium |
|--|--------------------------|--------------------------|
| Current price | $34.91 | $57.55 |
| Conservative recommendation | $38.99 | $64.99 |
| Model optimum | $44.00 | $78.00 |
| Conservative gain | +$0.57M / yr | +$1.27M / yr |
| Full gain | +$1.65M / yr | +$3.37M / yr |

Combined, that's +$1.83M/yr from the conservative prices, and a potential +$5.02M/yr at the model optimum — assuming the experiment validates the demand holds.

---

## The data was messier than it looked

Before I ran a single model, I spent time understanding what was actually in the data. A few things stood out.

**SKU B had only 4 unique prices over 2.5 years.** Essentially one price for the entire period, with a few exceptions. It had never really been tested at different price points, which makes standard elasticity estimation pretty fragile — there's just not enough variation to draw a reliable demand curve from.

**SKU D had a seasonal pricing problem.** Prices were set 12% higher in summer, which is also when exterior paint demand naturally peaks. This creates an endogeneity issue: OLS sees high prices and high demand together every summer and partially concludes the product is price-insensitive, when really both are just driven by the season. I used a quasi-experiment to get around this.

**Store identity swamped everything else.** When I ran XGBoost, store encoding came out as 73% of feature importance for SKU B. Some stores reliably sell 50 units a day, others reliably sell 10 — nothing to do with price. OLS had no way to account for this, which is why its R² was 7%. XGBoost hit 86–92%.

---

## How I approached it

I layered four methods on top of each other, each one addressing something the previous one couldn't.

**OLS first** — quick interpretable elasticity estimates. Log-log specification, so the coefficient is directly the elasticity. R² of 7% confirmed this was just a starting point. Elasticity: SKU B = -0.341, SKU D = -0.640.

**XGBoost for the actual demand model** — with store encoding, seasonal features, and inventory. R² jumped to 86–92%. I validated on 2023 data after training on everything before it, and performance held up, which means the model is learning real patterns rather than memorising the training period.

For price optimization, I swept across a defensible range — cost × 1.05 as the floor, 25% above the historical maximum as the ceiling — and found the price that maximised margin. Both SKUs kept rising to the ceiling, which is interesting in itself.

**Quasi-experiment for causal credibility** — on days when different stores happened to charge different prices for the same SKU, I used the cheaper stores as a natural control group. Comparing stores on the same day eliminates seasonal confounding automatically. For SKU D, this gave an elasticity of -0.866 versus OLS's -0.640 — a meaningful gap that confirms the seasonal bias concern. For SKU B, the two were close (-0.410 vs -0.341), suggesting OLS is reliable there.

**Segmented elasticity to look for heterogeneity** — I ran the OLS separately on each store revenue tier and found a 20x difference in price sensitivity:

| Store Tier | SKU B | SKU D |
|------------|-------|-------|
| Q1 Low | -1.052 | -1.062 |
| Q2 Mid-Low | -0.539 | -0.449 |
| Q3 Mid-High | -0.200 | -0.221 |
| Q4 High | -0.055 | -0.040 |

High-revenue stores are barely sensitive to price at all. Low-revenue stores are close to elastic. A single chain-wide price is leaving a lot on the table.

---

## Cannibalization turned out not to be a problem

One thing I wanted to check before recommending a price increase on SKU D was whether it would push customers toward cheaper alternatives. The cross-price elasticity to SKU E was +0.528 and to SKU F was +0.232 — so yes, some customers do switch.

But the switching customers don't leave the store. SKU E generates a $20.04 margin per unit, SKU F generates $12.46. The portfolio optimum, including all three exterior SKUs, agreed with the isolated optimum at $78. Cannibalization here is actually margin-positive.

---

## I didn't just take the model's word for it

200 bootstrap resamples all recommended the same ceiling price — zero variance across samples. A sensitivity table confirmed the recommendation stays profitable across elasticity scenarios from -0.2 to -2.0.

I also designed an 8-week A/B experiment rather than recommending an immediate chain-wide rollout. Power analysis showed 15 stores per group is enough to detect a 15% GM improvement with 80% power. Store assignment is stratified by revenue quartile so any observed difference can actually be attributed to the price change.

The recommendation is staged: conservative prices now to start capturing uplift while the experiment runs, then the model optimum chain-wide once the demand response is confirmed.

---

## Stack

Python — pandas, numpy, statsmodels, xgboost, scikit-learn, scipy, matplotlib

Methods — OLS (log-log and semi-log), HC3 robust standard errors, VIF, quasi-experiment, segmented elasticity, cross-price elasticity, bootstrap resampling, power analysis, row-level counterfactual projection

---

## Files

```
├── probuild_pricing_analysis.ipynb     # Full notebook with all outputs saved
└── README.md
```
