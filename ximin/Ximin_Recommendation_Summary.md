# Where Do They Buy It? — Recommendation to the Merchandising Team

**BZAN 535 · Team Project 1 · Private Label at Kroger**
Section owner: Ximin Zeng (xzeng3@utk.edu) — *which part of the store*

---

## The recommendation in one sentence

**Expand private-label assortment in MEAT-PCKGD first and DRUG GM second. Leave GROCERY alone — it is already won, and pushing it further spends shelf space where the return is lowest.**

---

## The position

| | Department | Action | Why |
|---|---|---|---|
| **Protect** | GROCERY | Hold assortment; defend | $1.11M private-label revenue, ~68% of all genuine private-label dollars. Shoppers already choose store brand 28.4% of the time. Nothing to fix. |
| **Act first** | MEAT-PCKGD | Add SKUs / facings | Shoppers over-select private label more than anywhere else (index **1.31**) while it holds only **17.3%** of the assortment. Clearest supply-side gap in the data. |
| **Act second** | DRUG GM | Test, then expand | Largest underperformance: a **$1.06M** department where private label takes only **9.4%** of dollars on **7.9%** of assortment. Upside is big but the cause is less certain. |
| **Ignore** | KIOSK-GAS, MISC SALES TRAN | No action | 100% and 95.3% private label because no national brand exists there. Mechanical, not a shopper choice. |

---

## The evidence, in business language

### 1. The same shopper behaves completely differently in different aisles

A household that buys store-brand groceries will not buy store-brand health and beauty products. Same customer, same trip, same store:

- **GROCERY: 28.4%** of a household's dollars go to private label (95% CI 27.8–29.0%)
- **DRUG GM: 10.2%** (95% CI 9.7–10.6%)

That is an **18.2 percentage-point gap**, and we are 95% confident the true gap is between **17.2 and 19.3 points**. The two intervals are nowhere near touching — this is not noise.

We ran **all 10 pairwise department comparisons** and adjusted the significance threshold for having run 10 of them. All 10 separate cleanly. Welch ANOVA handles the very unequal spread across departments; Kruskal–Wallis confirms the same ranking without assuming a bell curve. The conclusion does not rest on either assumption.

### 2. Revenue and performance point at different aisles — and that is the finding

Ranking departments by private-label **dollars** gives you GROCERY. Ranking by private-label **share** gives you PASTRY (51.6%) and SEAFOOD-PCKGD (62.6%). GROCERY's 27.0% share is mid-pack.

> **The volume and the opportunity are not in the same aisle.** Reporting only revenue would send the team to defend an aisle that needs no defending.

### 3. Shelf space suppresses the level but does not create the ranking

The obvious objection: private label wins in GROCERY simply because Kroger stocks more of it there (22.7% of products vs. 7.9% in DRUG GM). We tested it by dividing dollar share by assortment share:

| Department | PL assortment | PL dollars | Preference index |
|---|---|---|---|
| MEAT-PCKGD | 17.3% | 22.6% | **1.31** |
| DRUG GM | 7.9% | 9.4% | **1.20** |
| GROCERY | 22.7% | 27.0% | **1.19** |

**Every real department is above 1.0** — shoppers spend more on private label than the shelf allocation alone would predict. They are not simply taking what is in front of them. And MEAT-PCKGD, not GROCERY, has the strongest pull relative to what is stocked.

### 4. It is not a different type of shopper in each aisle

Second objection: maybe budget-conscious households shop GROCERY and affluent ones shop DRUG GM. We re-ran the comparison using **only households that shop all five departments**, so the shopper mix is held constant by construction. The ranking is unchanged: GROCERY 27.3%, MEAT-PCKGD 21.3%, DRUG GM 9.9%. The difference travels with the aisle, not with the customer.

---

## What we would do Monday

1. **MEAT-PCKGD** — add private-label SKUs in the sub-categories where shoppers already convert. MEAT - MISC is $65.8K in sales and already **52.7%** private label on thin assortment. Lowest-risk expansion in the dataset.
2. **DRUG GM** — do not mass-expand. Run a controlled assortment test in 20–30 stores in the highest-volume sub-categories first, because a trust barrier in medicine and cosmetics will not be solved by facings.
3. **GROCERY** — no assortment change. FLUID MILK PRODUCTS is already **88.9%** private label on $205K; there is no headroom left to buy.

---

## What we are not claiming

- **DRUG GM's gap has two possible causes and we cannot separate them.** Either it is under-shelved (fixable) or shoppers genuinely distrust store-brand medicine and cosmetics (not fixable with shelf space). The preference index of 1.20 suggests at least part is supply-side, but transaction data alone cannot settle it.
- **Assortment is measured as SKU count, not linear feet.** We have no planogram, so shelf space is a proxy.
- **This is observational.** Nothing here establishes that adding SKUs *causes* share to rise. A store-level assortment pilot with a matched control group would — and that is the design we would ask for before a chain-wide rollout.
- **Margin is absent from the data.** We rank opportunity by revenue and share; the team should re-rank by contribution margin before committing capital.

---

## Data handling

2,595,732 transaction rows over two years. After dropping non-sale lines (returns, zero-quantity) and joining to products with a known brand and department: **2,576,815 rows retained — 99.3% of rows and 100.0% of dollars.** No material exclusions.

**Unit of analysis:** one observation is **one household's private-label share of dollars within one department** (households spending under $30 in a department over two years are excluded, because a $4 spend produces a 0% or 100% share by accident). The question is about shopper behaviour, so every shopper gets one vote — otherwise a handful of 400-trip households would drive the entire result. Revenue totals are the one figure computed at the line-item level, because revenue is an accounting sum.
