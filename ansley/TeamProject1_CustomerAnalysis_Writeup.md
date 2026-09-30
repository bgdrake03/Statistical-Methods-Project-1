---
output:
  word_document: default
  html_document: default
---
# Which Customers Buy Private Label? Full Analysis Report

**BZAN 535, Team Project 1: Private Label at Kroger (customer analysis section)**

This report documents the "which customers" part of the team project: the analysis in `TeamProject1_CustomerAnalysis.Rmd` and the two presentation slides built from it. Every number here is produced by that notebook.

---

## 1. Executive summary

**Question.** Which household demographics are associated with buying more store-brand (private-label) products, and which carry no signal?

**Answer.**

- **Income is the demographic that matters most.** Households earning **$150K+ choose the store brand for 20.4%** of their items, versus **about 30% for households under $50K**. That's a gap of roughly 10 percentage points, and the $150K+ group is significantly below every other income bracket.
- **Homeownership carries a smaller, real signal.** Renters and households with unknown housing status buy more store brand than homeowners.
- **Age, household size, household composition, and number of kids show no detectable difference** once multiple testing is accounted for.
- **The income gap is not just about where high earners shop.** Controlling for which departments and which stores households use shrinks the gap by about 40% (from 8.9 to 5.3 points), but it doesn't disappear. **Inside the grocery aisle alone, the gap is still 9.7 points.**
- **Ideal store-brand household** (built from the significant attributes only): an **under-$25K renter, predicted at 32.6%**. The lowest is a **$150K+ homeowner at 19.9%**. The overall average is 27.9%.
- **Demographics explain only 8-11% of household-to-household differences.** They're useful for picking a segment to target, not for predicting any one household.

---

## 2. Key analysis decisions

### 2.1 Unit of analysis: the household

**One observation = one household** (n = 801 households with demographic data).

Why households and not baskets:

1. **Demographics are household traits.** Income and age don't change from trip to trip.
2. **Baskets aren't independent.** The 801 households made about 140,000 baskets (median 125 per household). A basket-level test would treat one family's 300 trips as 300 separate people, making every p-value look far stronger than the evidence supports.
3. **Basket averages over-weight heavy shoppers.** A household with 1,000 trips would count 40 times more than one with 25. At the household level, each customer counts once, which matches a question about *customers*.
4. **The choice changes the answer.** An earlier basket-level version suggested slight age differences. At the household level, age shows no detectable effect (Holm-adjusted p = 1.0).

### 2.2 Definition of "buys private label"

**Share of a household's item lines that are store brand**, across all their trips, counted only in departments where shoppers choose between brands.

- **Item lines, not dollars:** one row in the transaction data is one product chosen on one trip, so item share measures *how often* the shopper picks the store brand. Store brands are cheaper, so dollar share understates how often they're chosen. Dollar share was run as a robustness check and gives the same conclusions (section 3.9).
- **Item lines, not `QUANTITY`:** quantity mixes units (gallons, pounds, counts). A single gas fill-up would outweigh dozens of product choices.

### 2.3 Data cleaning: removing gas and store codes

In `product.csv`, **KIOSK-GAS is coded 100% "Private"** and **MISC SALES TRAN is 97% "Private."** These are fuel and store transaction codes, not store-brand choices. Left in, a gas-only basket counts as a 100% private-label trip.

**Rule applied (in code, not hand-picked):** keep departments with **at least 1,000 item lines** where private label is **between 1% and 90%** of lines, meaning a store-brand option actually exists and the category isn't a non-product code.

| Department | Item lines (all households) | Private label | Kept? |
|---|---|---|---|
| Grocery | 1,646,076 | 35.9% | Yes |
| Drug GM | 277,232 | 9.3% | Yes |
| Produce | 257,290 | 11.6% | Yes |
| Meat-Pckgd | 111,957 | 22.5% | Yes |
| Meat | 88,416 | 6.0% | Yes |
| Deli | 62,787 | 20.1% | Yes |
| Pastry | 38,179 | 46.3% | Yes |
| Nutrition | 32,164 | 9.2% | Yes |
| **Kiosk-Gas** | 22,059 | **100.0%** | **No** |
| Seafood-Pckgd | 11,216 | 55.1% | Yes |
| Salad Bar | 9,516 | 0.2% | No |
| (blank) | 7,839 | 0.0% | No |
| Cosmetics | 7,692 | 11.7% | Yes |
| **Misc Sales Tran** | 6,050 | **97.1%** | **No** |
| Floral | 4,524 | 10.9% | Yes |
| Seafood | 4,093 | 3.1% | Yes |
| Misc. Trans. | 2,351 | 0.5% | No |
| Spirits | 2,119 | 0.0% | No |

**Result:** 12 departments kept, holding **1,395,718 item lines for the 801 demographic households (98% of their lines).**

### 2.4 Small groups

Any group with **fewer than 30 households** is flagged "SMALL GROUP" in tables and drawn in grey on charts.

- **Income brackets were combined** because several original brackets were too thin:

  | Original bracket | Households | Combined into |
  |---|---|---|
  | Under 15K / 15-24K | 61 / 74 | Under 25K (135) |
  | 25-34K / 35-49K | 77 / 172 | 25-49K (249) |
  | 50-74K | 192 | 50-74K (192) |
  | 75-99K | 96 | 75-99K (96) |
  | 100-124K / 125-149K | 34 / 38 | 100-149K (72) |
  | 150-174K / 175-199K / 200-249K / 250K+ | 30 / 11 / 5 / 11 | 150K+ (57) |

- **Still flagged:** Homeowner "Probable Owner" (11) and "Probable Renter" (11). They stay in the data but aren't interpreted on their own, and the "gap" measure excludes them.
- **Ideal-household profiles:** most renter combinations have fewer than 30 households and are flagged (section 3.11).

---

## 3. Analysis, section by section

Section names below match the chunk headers in the notebook.

### 3.1 LOAD DATA and CLEAN DATASETS

- Loads `hh_demographic.csv`, `product.csv`, and `transaction_data.csv` and joins transactions to product department and brand.
- Builds the department table (`DEPT_TABLE`) and the kept-department list (`DEPT_KEEP`) using the rule in section 2.3.
- Builds each household's **store environment** (`STORE_ENV`): the private-label rate at the stores it shops, weighted by how much it shops at each. This is computed from **all other households' purchases**, so a household's own buying doesn't feed into its own control.
- Joins the kept transactions to the 801 demographic households (`DATA_JOINED`) and frees memory by removing the large raw tables.

### 3.2 BUILD HOUSEHOLD-LEVEL DATA

Creates `HH`, one row per household, with:

| Variable | Meaning | Used for |
|---|---|---|
| `PL_SHARE` | Store-brand item lines ÷ all item lines | **Main measure** |
| `PL_DOLLAR_SHARE` | Store-brand dollars ÷ all dollars | Robustness check |
| `GROCERY_PL_SHARE` | Store-brand share inside the Grocery department only | Alternative-explanation check |
| `EXPECTED_MIX` | Share the household *would* have if it bought each department at the chain-wide rate | Department-mix control |
| `STORE_ENV` | Store-brand rate of the household's stores (from other shoppers) | Store control |
| `ITEMS`, `BASKETS` | Shopping volume | Description, controls |
| `INCOME` | Combined 6-bracket income, ordered low to high | Analysis |

It also defines the seven demographics compared (`DEMO_VARS`), the small-group threshold (`SMALL_N = 30`), and the overall mean.

### 3.3 DESCRIPTION: distribution of household private-label share

| Households | Median baskets | Mean | Median | SD | Min | Max |
|---|---|---|---|---|---|---|
| 801 | 125 | 27.9% | 26.9% | 10.5 pts | 5.2% | 65.9% |

The histogram is single-peaked and slightly right-skewed. **Half of households fall between 20% and 34%.** Households differ from each other by about 10 points (1 SD), which is the variation the demographics are trying to explain.

### 3.4 GROUP AVERAGES WITH 95% CONFIDENCE INTERVALS

Each group's CI is a t-interval on the household mean: mean ± t × SD/√n.

**Income** (the key table):

| Income | Households | Store-brand share | 95% CI |
|---|---|---|---|
| Under 25K | 135 | 30.4% | 28.8-32.0 |
| 25-49K | 249 | 29.8% | 28.4-31.2 |
| 50-74K | 192 | 27.6% | 26.1-29.1 |
| 75-99K | 96 | 25.3% | 23.4-27.3 |
| 100-149K | 72 | 26.9% | 24.5-29.3 |
| **150K+** | **57** | **20.4%** | **18.5-22.3** |

The 150K+ interval doesn't overlap any other bracket. The decline isn't perfectly smooth (100-149K sits slightly above 75-99K, within noise), but the direction is clearly downward.

**Homeowner status:**

| Homeowner | Households | Share | 95% CI |
|---|---|---|---|
| Homeowner | 504 | 26.4% | 25.5-27.3 |
| Renter | 42 | 31.7% | 27.8-35.6 |
| Unknown | 233 | 30.6% | 29.2-31.9 |
| Probable Owner *(small)* | 11 | 25.3% | 16.4-34.2 |
| Probable Renter *(small)* | 11 | 28.7% | 22.5-34.8 |

**The other five demographics** (none significant):

| Demographic | Group (households): share |
|---|---|
| Age | 19-24 (46): 30.0% · 25-34 (142): 26.7% · 35-44 (194): 27.7% · 45-54 (288): 28.5% · 55-64 (59): 27.7% · 65+ (72): 27.8% |
| Marital status | A (340): 26.8% · B (117): 26.5% · U/unknown (344): 29.5% |
| Household composition | 1 Adult Kids (47): 28.1% · 2 Adults Kids (187): 27.9% · 2 Adults No Kids (255): 27.1% · Single Female (144): 27.9% · Single Male (95): 28.8% · Unknown (73): 29.4% |
| Number of kids | 1 (114): 27.4% · 2 (60): 27.8% · 3+ (69): 28.9% · None/Unknown (558): 27.9% |
| Household size | 1 (255): 29.1% · 2 (318): 27.0% · 3 (109): 27.4% · 4 (53): 28.3% · 5+ (66): 28.4% |

Marital status codes A/B/U aren't labeled in the data. A common reading is A = married, B = single, U = unknown, but that should be confirmed before relabeling.

### 3.5 SIGNIFICANCE TESTS: which demographics carry signal?

**Seven tests were run (one per demographic)**, so the p-values are adjusted with the **Holm correction**, which controls the chance of any false positive across all seven.

Each demographic gets:

- **Welch ANOVA** (main test): compares group means without assuming equal spread across groups.
- **Kruskal-Wallis** (backup): a rank-based test that doesn't assume a normal distribution. "Significant" requires **both** tests to pass after Holm.
- **Variation explained (eta squared):** the share of household-to-household variation the demographic accounts for.
- **Gap:** highest minus lowest group mean among groups with 30+ households. This is the business-language size of the difference.

| Demographic | Groups | Gap (pts) | Variation explained | Welch p (Holm) | K-W p (Holm) | Significant |
|---|---|---|---|---|---|---|
| **Income** | 6 | **10.0** | **6.4%** | <0.0001 | <0.0001 | **Yes** |
| **Homeowner status** | 5 | 5.3 | 3.9% | 0.0012 | <0.0001 | **Yes** |
| Marital status | 3 | 3.0 | 1.7% | 0.0059 | 0.0042 | Yes (drops out in 3.7) |
| Household size | 5 | 2.1 | 0.8% | 0.78 | 0.74 | No |
| Age | 6 | 3.3 | 0.6% | 1.00 | 1.00 | No |
| Household composition | 6 | 2.2 | 0.4% | 1.00 | 1.00 | No |
| Number of kids | 4 | 1.5 | 0.1% | 1.00 | 1.00 | No |

**Pairwise follow-ups** (Holm-adjusted, unequal variances) show where the differences are:

- **Income:** 150K+ is significantly below **every** other bracket (p from <0.0001 to 0.004). 75-99K is also below Under 25K (p = 0.001) and 25-49K (p = 0.003). The brackets under $75K don't differ from each other.
- **Homeowner:** the only significant pair is **Unknown vs. Homeowner** (p < 0.0001). **Renter vs. Homeowner is only p = 0.09.** With 42 renters, that 5-point gap is suggestive, not proven on its own.
- **Marital:** only **U (unknown)** differs from A and B. A vs. B shows no difference (p = 0.82).

### 3.6 CONFIDENCE INTERVAL CHARTS

One dot-and-interval chart per demographic, ordered from most to least significant. Each shows the household mean with a 95% CI, a dashed line at the overall average (27.9%), and the Holm-adjusted p-value in the subtitle. Small groups are drawn in grey and labeled "small."

### 3.7 DO THE SIGNALS HOLD UP TOGETHER? (multiple regression)

All seven demographics go into one linear model. Each F-test asks: *does this demographic still matter once the other six are accounted for?* This catches demographics that only look important because they overlap with another one.

| Demographic | p (controlling for the other six) |
|---|---|
| **Income** | <0.0001 |
| **Homeowner status** | 0.007 |
| Number of kids | 0.08 |
| Marital status | 0.11 |
| Household size | 0.12 |
| Household composition | 0.30 |
| Age | 0.39 |

The full model's R² is 0.111 (adjusted 0.079): **all seven demographics together explain about 8-11% of the variation between households.**

**Marital status drops out** once the others are included; its apparent effect overlapped with income and homeownership.

**Rule for a "statistically significant attribute":** significant on its own after Holm **and** still significant after controlling for the other demographics. **Result: Income and Homeowner Status.** The notebook applies this rule in code (`SIG_VARS`), so it would update automatically if the data changed.

### 3.8 ALTERNATIVE EXPLANATIONS

A skeptic could argue that high-income households don't *prefer* national brands. They might just:

- **(a) Shop different departments:** more produce and meat (little store brand), less center-store grocery (lots of store brand).
- **(b) Shop at different stores** that carry or sell less store brand, such as stores in wealthier neighborhoods.

Each explanation was tested by adding a control to a regression of store-brand share on income and homeownership. **The number reported is the regression coefficient for $150K+ (vs. under-$25K) × 100**: the gap in percentage points, holding the other variables in that model constant. The interval is its 95% confidence interval.

| Model | 150K+ vs. under-25K gap (pts) | 95% CI | Income p |
|---|---|---|---|
| Income + homeowner only | **-8.9** | -12.1 to -5.7 | <0.0001 |
| + department mix | -6.2 | -9.3 to -3.2 | 0.0006 |
| + store environment | -7.4 | -10.7 to -4.0 | 0.0001 |
| + both | **-5.3** | -8.5 to -2.1 | 0.012 |
| Grocery department only | **-9.7** | -14.0 to -5.5 | <0.0001 |
| Grocery only + store | -8.1 | -12.5 to -3.7 | 0.004 |

**How the controls are built:**

- **Department mix (`EXPECTED_MIX`):** the share each household would have if it bought each department at the chain-wide store-brand rate. A household that buys mostly produce (about 12% store brand) gets a low expected value; one that buys mostly grocery (about 36%) gets a high one.
- **Store environment (`STORE_ENV`):** the store-brand rate at the stores each household uses, weighted by how much it shops at each, **calculated from other households only.** An earlier version that included the household's own purchases wrongly suggested income was fully explained by store; that was circular and has been fixed.
- **Grocery only:** repeats the comparison inside one department, which removes department mix entirely.

**What each income group looks like on these controls:**

| Income | Actual share | Expected from dept mix | Store PL rate | Grocery-only share |
|---|---|---|---|---|
| Under 25K | 30.4% | 28.4% | 29.9% | 38.5% |
| 25-49K | 29.8% | 28.5% | 28.7% | 37.4% |
| 50-74K | 27.6% | 28.3% | 27.5% | 35.0% |
| 75-99K | 25.3% | 27.9% | 27.1% | 32.5% |
| 100-149K | 26.9% | 27.7% | 26.7% | 34.9% |
| 150K+ | 20.4% | 26.6% | 25.2% | 27.8% |

**Interpretation:**

1. **Both explanations are partly true.** High earners do shop departments and stores with less store brand. That accounts for about **40% of the gap**: (8.9 − 5.3) ÷ 8.9 = 0.40.
2. **Neither explains it away.** After both controls, a **5.3-point gap remains**, and its interval (-8.5 to -2.1) is well below zero.
3. **The grocery-only result is the strongest evidence.** In the same aisle, with the same kinds of products available, $150K+ households still pick the store brand **9.7 points** less often. They are choosing national brands, not just shopping in different places.
4. **Why -8.9 and not the 10-point gap in the averages?** The raw averages differ by 10.0 points (30.4% vs. 20.4%). The regression also accounts for homeownership, and nearly all $150K+ households are homeowners, who buy slightly less store brand anyway. That accounts for about 1 point.
5. **Why the grocery interval is wider:** each household buys fewer items in one department than across the whole store, so there's less data per household.

### 3.9 ASSUMPTION CHECKS AND ROBUSTNESS

| Check | Result | Why it's fine |
|---|---|---|
| Normality of residuals (Q-Q plot) | Mild right skew (a few very high store-brand households) | With 801 households, means and CIs are reliable (Central Limit Theorem), and Kruskal-Wallis, which doesn't assume normality, agrees on every variable |
| Equal spread across groups | Largest SD ÷ smallest SD ranges 1.07-1.54 across demographics (the 150K+ group is tighter) | Welch ANOVA and unequal-variance pairwise tests don't require equal spread |
| Independence | Each household appears once | This is why households are the unit |
| Different definition (dollar share) | **The same 3 significant / 4 not-significant split** | The conclusion doesn't depend on counting items vs. dollars |

### 3.10 WHO ARE THE "UNKNOWN" HOUSEHOLDS?

233 of the 801 households (29%) didn't report homeowner status, and they buy more store brand than homeowners (30.6% vs. 26.4%).

| | Unknown homeowner status (233) | Known homeowner status (568) |
|---|---|---|
| Marital status also unknown | **82%** | 27% |
| Single-person household | **58%** | 21% |
| Under age 35 | **35%** | 19% |
| Income under $50K | **64%** | 42% |
| Median baskets | 135 | 122 |
| Store-brand share | **30.6%** | 26.8% |

**Controlled comparison:** after controlling for income, age, household size, and number of trips, unknown-status households buy **+2.8 points more store brand (95% CI +1.1 to +4.6, p = 0.001).**

**Interpretation, separating evidence from inference:**

- **Supported by the data:** these are mostly customers who **chose not to share** housing information. 82% also left marital status blank, so this is a pattern of sharing less information overall.
- **Plausible but not verifiable:** some may be **not yet settled in a living situation**. Their profile (younger, living alone, lower income) resembles the renters, and their share (30.6%) is close to renters' (31.7%). But nothing in the data records moves or temporary housing.
- They shop regularly (median 135 trips), so they are active loyalty-card customers.

**Suggested wording:**

> "About 30% of households didn't report their housing status, and most of them also left marital status blank. These households buy more store brand than homeowners, even after accounting for income and age. They skew younger, single-person, and lower-income, a profile similar to renters. That suggests people who are early in their household life or in a transitional living situation, though the data can't confirm it."

"Unknown" is left out of the ideal-household ranking (next section) because it describes missing information, not a trait Kroger can target.

### 3.11 THE "IDEAL" PRIVATE-LABEL HOUSEHOLD

**Method:**

1. Fit a regression using **only the significant attributes** (Income + Homeowner Status).
2. Predict store-brand share, with a 95% CI, for every income × homeowner combination that **actually exists in the data**.
3. Leave "Unknown" out of the ranking.
4. Flag profiles with fewer than 30 real households.

**Why not use all seven demographics?**

- Five of the seven show no reliable effect, so including them would build the "peak" out of noise.
- Composition, kids, and household size overlap heavily ("2 Adults Kids" implies kids; size follows from composition). A seven-way combination could describe a profile that doesn't exist.
- Picking the maximum out of dozens of noisy predictions systematically overstates the peak (the "winner's curse").

**Results:**

| Profile | Real households | Predicted share | 95% CI |
|---|---|---|---|
| **Under 25K / Renter** (highest) | 11 *(small)* | **32.6%** | 29.2-36.0 |
| 25-49K / Renter | 21 *(small)* | 32.2% | 29.0-35.4 |
| **Under 25K / Homeowner** (highest with 30+ households) | 60 | **28.8%** | 26.9-30.7 |
| 25-49K / Homeowner | 131 | 28.4% | 26.9-29.8 |
| 50-74K / Homeowner | 130 | 26.7% | 25.2-28.3 |
| 100-149K / Homeowner | 56 | 26.2% | 23.9-28.6 |
| 75-99K / Homeowner | 80 | 24.8% | 22.8-26.9 |
| **150K+ / Homeowner** (lowest) | 47 | **19.9%** | 17.3-22.6 |
| *All households* | 801 | 27.9% | |

**How to present it honestly:**

- The model is **additive**: it estimates the income effect from all 801 households and the renter effect from all 42 renters, then combines them. So the under-$25K renter prediction doesn't rest on only 11 households. But only 11 such households exist, so it's a small segment.
- The renter effect itself is borderline (p = 0.02 in the model, p = 0.09 in the direct pairwise test). **Income drives the ranking; homeownership is secondary.**

### 3.12 SLIDE FIGURES

Saves the four charts used in the presentation to `slide_figures/`, sized to their exact on-slide dimensions (inches = slide pixels ÷ 100, at 300 dpi) so text stays readable when projected:

| File | Chart | Slide |
|---|---|---|
| `fig1_distribution.png` | Histogram of household store-brand share with the average marked | 2 |
| `fig2_signal.png` | Variation explained by each demographic, colored by significance, labeled with gap and Holm p | 2 |
| `fig3_income.png` | Store-brand share by income with 95% CIs, $150K+ highlighted | 3 |
| `fig4_gap_checks.png` | $150K+ vs. under-$25K gap under each control, with 95% CIs | 3 |

Colors: blue `#2a78d6` for data, orange `#e8833a` to highlight the $150K+ group, grey for non-significant results.

---

## 4. The presentation

Three slides in the team's Canva deck: a title slide for the whole team, then two slides for the customer analysis.

### Slide 1: Title

- **"Private Label at Kroger"**
- Subtitle: *Who buys the store brand, where they buy it, and what to do about it*
- BZAN 535 · Team Project 1 · October 1, 2026
- Dark navy background with a blue accent circle and an orange accent bar.

### Slide 2: "Only income and homeownership predict store-brand buying"

*Label: WHICH CUSTOMERS · 1 OF 2*

| Area | Content |
|---|---|
| Left card: **How we measured it** | **Unit of analysis:** The household (n = 801), not the basket · **"Buys private label":** Share of a household's items that are store brand · **Data cleaning:** Removed gas and store codes; kept 12 departments with a real brand choice · **Methods:** 7 tests, Holm-adjusted, plus one regression with all 7 |
| Top right: **Most households buy store brand for 20-34% of items** | Histogram (`fig1`) with a large "Average 27.9%" callout |
| Bottom right: **Which demographics carry signal?** | Signal chart (`fig2`): income and homeowner status in blue, marital status in light blue (significant alone only), the other four in grey |

**Speaker notes (about 1.5 minutes):**

- Question: which customers buy Kroger's store brand, and which demographics actually matter?
- The unit of analysis is the household (801 with demographics). Income and age are household traits. The ~140K baskets come from the same families, so testing baskets would treat one family's 300 trips as 300 people and overstate certainty. It also changes the answer: at the basket level age looked like it mattered; at the household level it doesn't.
- "Buys private label" = share of a household's items that are store brand. Items measure how often they choose it; dollar share understates it because store brands are cheaper (dollars were checked too, with the same conclusions).
- Data cleaning: gas is coded 100% "Private" and a store transaction code 97%, so a gas-only basket looked like a 100% store-brand trip. Only the 12 departments where shoppers actually choose between brands were kept (98% of items).
- Description: the typical household buys store brand for about 28% of items, with a wide spread (middle half 20-34%).
- Signal chart: all 7 demographics were tested (Welch ANOVA + Kruskal-Wallis, Holm-adjusted for 7 tests). Only income and homeownership hold up, including in a regression with all 7 together. Marital status looks significant alone but drops out once income is controlled. Age, kids, household size, and composition show no detectable difference.

### Slide 3: "$150K+ households buy ~10 pts less store brand, even in the same aisle"

*Label: WHICH CUSTOMERS · 2 OF 2*

| Area | Content |
|---|---|
| Left card: **Store-brand share by income** | Income CI chart (`fig3`), $150K+ highlighted in orange at 20.4% |
| Right card: **Does the gap survive controls?** | Gap chart (`fig4`) with rows *Income + homeowner only* (-8.9), *+ department mix* (-6.2), *+ store environment* (-7.4), *+ both* (-5.3), *Grocery aisle only* (-9.7). Takeaway: **"Yes: a 5-pt gap survives every control."** |
| Bottom-left card (navy): **Ideal store-brand household** | Highest: under-$25K renter, 32.6% · Lowest: $150K+ homeowner, 19.9% |
| Bottom-middle card: **Checks** | Two tests agree · Same result in dollars · Holm-adjusted for 7 tests |
| Bottom-right card: **Limits** | 801 of ~2,500 households · Association, not causation · No price or promo data |

**Speaker notes (about 1.5 minutes):**

- Left: income is the clearest pattern. Under $50K, about 30% of items are store brand; $150K+ households drop to 20.4%. The 150K+ interval doesn't overlap any other bracket, and it's significantly below every other bracket after Holm correction (p ≤ 0.004).
- Right: is it just *where* they shop? Controlling for department mix (produce and meat have little store brand) and store environment (the store-brand rate at each household's stores, computed from other households so it isn't circular), the gap shrinks from 8.9 to 5.3 points but doesn't go away. Inside the grocery aisle alone it's still 9.7 points.
- Ideal household: using only the significant attributes (income + homeownership), the highest predicted share is an under-$25K renter at 32.6%; the lowest is a $150K+ homeowner at 19.9%. Only 11 households match the top profile exactly, so income is the solid driver and homeownership is secondary.
- Checks and limits: Welch and Kruskal-Wallis agree, and dollar share gives the same answer. Demographics cover 801 of about 2,500 households, this is association not causation, and there's no price, promotion, or shelf data.

**Suggested line for the alternative-explanations chart:**

> "The obvious pushback is that wealthier shoppers just go to different stores or different aisles. When we control for both, the gap shrinks by about 40%, but it doesn't disappear. And inside the grocery aisle alone, it's still nearly 10 points. They're choosing national brands, not just shopping in different places."

### Design choices

- **Titles state the finding**, not the topic, so the audience gets the point even if they only read the headline.
- **Every chart shows uncertainty** (95% CIs or significance coloring), which the rubric requires for any claimed difference.
- **One highlight color:** orange marks the $150K+ group on both slide 3 charts so the eye connects them.
- **Text kept short on the slides.** Justifications, caveats (the 11-renter note, the renter p = 0.09, the marital-status result), and exact p-values live in the speaker notes for delivery and Q&A.

---

## 5. How this covers the project requirements

| Requirement | Where it's addressed |
|---|---|
| 1. Defined and defended unit of analysis | Section 2.1; slide 2 "How we measured it"; speaker notes |
| 2. Description first | Section 3.3; slide 2 histogram |
| 3. Methods that fit the question | Sections 3.4-3.7 (CIs, Welch ANOVA, Kruskal-Wallis, pairwise tests, regression) |
| 4. Uncertainty and magnitude, multiplicity handled | 95% CIs on every group; gaps in percentage points; Holm correction for 7 tests and for pairwise tests; slides 2-3 |
| 5. Interrogation of the finding | Assumption checks (3.9); alternative explanations ruled on (3.8); dollar-share robustness; slide 3 "Does the gap survive controls?" |
| 6. Clear statement of limits | Section 6; slide 3 "Limits" card |
| Code reproduces slide numbers | Section 7; `SLIDE FIGURES` chunk regenerates every chart on the slides |

---

## 6. Limitations

1. **Demographics cover only 801 of about 2,500 households.** If households that share their demographics differ from those that don't, the results may not generalize. Customers who share less information (the "Unknown" group) buy more store brand, so the ~1,700 households with no profile at all may lean the same way.
2. **Association, not causation.** Income is linked to store-brand share, but education, price sensitivity, and brand attitudes aren't measured.
3. **Price and promotions aren't controlled.** If national brands are discounted more at certain stores or times, that affects choice.
4. **Assortment isn't observed.** We can't tell whether a store-brand option was on the shelf at the moment of each choice. The store-environment control only approximates this.
5. **Demographics explain little overall** (8-11% of variation). Behavior (trip frequency, categories bought, price sensitivity) likely predicts store-brand buying better.
6. **Coarse, self-reported categories.** Income is in brackets, and the 150K+ group mixes $150K and $250K+ households. Marital status codes aren't labeled.

**What would make the claim stronger:** demographics for all households; shelf price and promotion data; product-level substitution (did the household face both a store-brand and a national version of the same item?); or an experiment, such as targeted store-brand coupons by income segment.

---

## 7. Likely questions and answers

- **"Why households and not baskets?"** Demographics are household traits, baskets from the same household aren't independent, and household-level counts each customer once. Basket-level analysis would overstate certainty and change which demographics appear to matter.
- **"Why share of items instead of dollars?"** Items measure *how often* the shopper picks the store brand. Store brands are cheaper, so dollar share understates adoption. Dollars were checked too, with the same conclusions.
- **"Isn't this just rich people shopping at different stores?"** Partly. Store and department mix explain about 40% of the gap, but a 5-point gap remains, and within the grocery aisle alone it's about 10 points.
- **"How many tests did you run?"** Seven, Holm-corrected. Pairwise follow-ups are also Holm-corrected within each demographic.
- **"How did you get the numbers on the gap chart?"** Each row is a regression of store-brand share on income and homeownership plus that row's control. The number is the $150K+ coefficient × 100, the bar is its 95% CI, and 40% = (8.9 − 5.3) ÷ 8.9.
- **"How confident are you in the renter result?"** Moderately. It's significant in the model (p = 0.02) but based on 42 renters, and the direct renter vs. homeowner comparison is p = 0.09. Income is the solid finding.
- **"Who are the 'Unknown' households?"** Mostly customers who didn't share their information: 82% also left marital status blank. They skew younger, single-person, and lower-income, and buy about 2.8 points more store brand even after controls. Possibly customers in a transitional living situation, but the data can't confirm it.
- **"How much do demographics explain?"** About 8-11% of household differences. Useful for targeting a segment, not for predicting an individual household.

---

## 8. Reproducing the analysis

**Files:**

| File | Purpose |
|---|---|
| `TeamProject1_CustomerAnalysis.Rmd` | The full analysis; produces every number and chart in this report and on the slides |
| `TeamProject1_CustomerAnalysis_Writeup.md` | This report |
| `slide_figures/` | The four chart images used on the slides (created by the notebook) |

**Where the code lives:** the notebook is on the team GitHub repo (`bgdrake03/Statistical-Methods-Project-1`) at `ansley/TeamProject1_CustomerAnalysis.Rmd`.

**To run it:**

1. Put `transaction_data.csv`, `product.csv`, and `hh_demographic.csv` in the same folder as the `.Rmd`. The CSVs aren't in the repo; the transaction file is about 140 MB.
2. Install `tidyverse` and `knitr` if needed.
3. Run all chunks (or knit). Loading the transaction file takes a minute or two and uses about 1 GB of memory; the notebook removes the raw tables once they're no longer needed.

**Coordinating with teammates:** if the "where" analyses (departments, stores) use the same department filter (`DEPT_KEEP`: no gas, no misc transactions) and the same definition of "buys private label" (item share), the team's numbers will be consistent. State that definition once, up front, in the presentation.
