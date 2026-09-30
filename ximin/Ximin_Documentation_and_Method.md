# Where Do They Buy It? — Documentation for Ximin's Section
**BZAN 535 · Team Project 1 · Private Label at Kroger**
Owner: Ximin Zeng (xzeng3@utk.edu) · Section: *Department / Category analysis*

---

## 1. How to run this

```
project/
├─ Ximin_PrivateLabel_Department_Analysis.Rmd
└─ data/
   ├─ transaction_data.csv
   ├─ product.csv
   └─ hh_demographic.csv
```

```r
install.packages(c("data.table","dplyr","tidyr","stringr","ggplot2",
                   "scales","knitr","kableExtra","tibble","ggrepel"))

rmarkdown::render("Ximin_PrivateLabel_Department_Analysis.Rmd")
```

If your CSVs sit elsewhere, override the parameter instead of editing code:

```r
rmarkdown::render("Ximin_PrivateLabel_Department_Analysis.Rmd",
                  params = list(data_dir = "~/Downloads/bzan535"))
```

**Runtime:** ~1–3 minutes. The cluster bootstrap (`boot_reps = 1000`) is the slow
part — set `boot_reps = 200` while drafting, put it back to 1000 for the final run.

**Outputs created automatically**

| Path | Contents |
|---|---|
| `out/slide_numbers.txt` | Every number quoted on the slides |
| `out/table_department_summary.csv` | All departments: revenue, PL revenue, shares, sample sizes |
| `out/table_category_summary.csv` | All commodities ranked by PL revenue |
| `out/table_department_CIs.csv` | Means + t and bootstrap 95% CIs |
| `out/table_pairwise_tukey.csv` | All pairwise comparisons, adjusted |
| `figs/fig1_revenue_split.png` | Slide A chart |
| `figs/fig3_ci.png` | Slide B chart |
| `figs/fig2_distribution.png`, `figs/fig4_assortment.png` | Backup for Q&A |

Rule for the deck: **never retype a number.** Copy from `out/slide_numbers.txt`.
That is how the code provably reproduces the slides, which is an explicit
requirement in the brief.

---

## 2. How to find the Department / Category with the most private-brand sales by revenue

This is the question you were assigned, so here is the exact logic, in words.

### Step 1 — Decide what "revenue" means
`SALES_VALUE` is the dollars actually paid on that line, **after** loyalty and
retail discounts. That is the right basis: it is the money Kroger received. Do
**not** reconstruct a price from `QUANTITY`, and do **not** add the discount
columns back in — you would be reporting a list price nobody paid.

### Step 2 — Decide what "private brand" means
Private label lives in `product.csv`, not in the transaction file. So brand is a
**product attribute** that must be joined on `PRODUCT_ID`:

```
transaction_data  --PRODUCT_ID-->  product (BRAND, DEPARTMENT, COMMODITY_DESC)
```

Two traps that silently corrupt this:

1. **Whitespace.** Character fields in this dataset are space-padded
   (`"Private   "`). `BRAND == "Private"` then matches *nothing* and you get a
   0% private-label share everywhere. The Rmd runs `str_squish()` on every
   character column before any filter.
2. **Unlabelled brand.** Some products have a blank or third-value `BRAND`.
   They cannot be scored either way. The Rmd drops them **and reports how much
   revenue that cost** in the audit table — so the exclusion is a disclosed
   decision, not a hidden one.

### Step 3 — Aggregate at the right unit
For **revenue**, one observation is **one line item** (one product in one
basket), because revenue is a sum of dollars and the line item is the accounting
atom. In words:

> For each department, add up `SALES_VALUE` over all line items whose product is
> private label. Rank departments by that total, descending. The top row is the
> answer.

```r
dept_tbl <- dt[, .(revenue_private = sum(SALES_VALUE[BRAND == "Private"]),
                   revenue_total   = sum(SALES_VALUE)), by = DEPARTMENT]
dept_tbl[order(-revenue_private)][1]          # <- the department
```

### Step 4 — Repeat one level down for category
`DEPARTMENT` is too coarse for a shelf decision. `COMMODITY_DESC` is the level a
category manager owns, so group by `DEPARTMENT + COMMODITY_DESC` and rank the
same way. Grouping by commodity alone can collide identical names across
departments, which is why both keys are used.

### Step 5 — Report revenue *and* share together, and expect them to disagree
This is the part that earns marks.

- **PL revenue (absolute $)** answers *"where is the private-label business?"*
  It is dominated by whichever department is simply biggest. Almost always
  GROCERY — that is a size effect, not an insight.
- **PL dollar share (PL $ ÷ department $)** answers *"where does private label
  actually win?"* It removes department size and can crown a much smaller aisle.

The Rmd prints both rankings side by side with their Spearman correlation. The
interesting sentence for the slide is the mismatch, e.g.:

> "The biggest private-label *business* and the strongest private-label
> *performance* are not in the same aisle — which means the growth
> opportunity and the volume are in different places."

### Step 6 — Switch units before claiming a difference is real
Ranking totals needs no test. But the moment you say *"private label performs
better in A than in B,"* the line item is the wrong unit: one 400-trip household
would carry the test. So for all inference the Rmd switches to
**household × department** — one household's PL share of spend inside one
department, one vote each. Say this out loud in the presentation; the brief
promises they will ask about it.

---

## 3. Unit of analysis — the answer to give when asked

> "One observation is **one household's private-label share of spend within one
> department**, over the two years. I chose the household because the business
> question is about shopper behaviour, not about transactions — at the item or
> basket level a handful of heavy shoppers would drive the whole result and the
> independence assumption would be indefensible. Dollar totals are the one thing
> I compute at the line-item level, because revenue is an accounting sum. I
> exclude a household from a department if it spent under $30 there in two
> years, because a $4 spend produces a 0% or 100% share by accident."

Three units, each with a stated job — that is the defensible answer, not a
single-unit claim you then quietly violate.

---

## 4. Statistical choices and why each one

| Choice | Reason |
|---|---|
| **Welch one-way ANOVA** | Department SDs differ materially; classical F assumes they don't |
| **Kruskal–Wallis** | The outcome is bounded in [0,1], skewed, and piled at 0 and 1 — confirms the result without a normality assumption |
| **Eta-squared** | With ~50k observations every p-value is < 0.001. Effect size is the real evidence |
| **Cluster bootstrap over households** | A household appears in several departments; a textbook t-interval would be too narrow |
| **Tukey HSD + Bonferroni Wilcoxon** | k(k−1)/2 comparisons were run; the brief explicitly requires multiplicity to be accounted for |
| **Chi-square + Cramér's V on items** | Independent cross-check on a different unit and a different metric. If it agrees, the conclusion isn't an artefact of either choice |
| **Friedman test on complete households** | Same households compared across departments, so shopper mix is held constant by construction — kills the composition objection |

**Say this, not that:**

| Don't say | Say |
|---|---|
| "p < 0.001, so it's significant" | "Department explains about X% of the variation in how much private label a household buys" |
| "There's a difference" | "A gap of X percentage points, and I'm 95% confident the true gap is between Y and Z" |
| "We tested the departments" | "I ran N pairwise comparisons and adjusted for all of them; these M survive" |

---

## 5. Filling in the slide blanks

Open `out/slide_numbers.txt` and drop the values in:

| Slide placeholder | Field in `slide_numbers.txt` |
|---|---|
| number of departments | `departments_analysed` / `departments_passing_screen` |
| top department by PL revenue | `top_dept_by_PL_revenue`, `top_dept_PL_revenue` |
| top category by PL revenue | `top_category_by_PL_revenue`, `top_category_PL_revenue` |
| highest / lowest share | `highest_share_dept`, `lowest_share_dept` |
| the gap | `headline_gap_pp`, `headline_gap_CI` |
| effect size | `eta_squared`, `cramers_v` |
| comparisons | `comparisons_run`, `multiplicity_method` |
| sample size | `analysis_observations`, `households` |
| coverage | `dollars_covered` |

Replace the placeholder chart boxes on the two slides with `figs/fig1_revenue_split.png`
(Slide A) and `figs/fig3_ci.png` (Slide B).

---

## 6. Presenter notes — your 2–3 minutes

**Slide A — "Where the private-label dollars are" (~75 seconds)**

> "So we know who buys private label. Now: *where* in the store.
>
> I looked at all **[departments_analysed]** departments, and kept the
> **[departments_passing_screen]** that are at least 1% of chain dollars — small
> aisles give you spectacular-looking percentages built on nothing.
>
> Two different questions here, and they have two different answers. Where is
> the private-label *business*? **[top_dept_by_PL_revenue]** —
> **[top_dept_PL_revenue]** in private-label sales. But that's mostly because
> it's the biggest department. Where does private label actually *win*?
> **[highest_share_dept]** — that's the share number in white on the bar.
>
> At the category level the biggest single private-label earner is
> **[top_category_by_PL_revenue]** at **[top_category_PL_revenue]**. That's the
> level a category manager can actually act on.
>
> The takeaway from this slide: the volume and the opportunity are in different
> aisles."

**Slide B — "Are these differences real?" (~75 seconds)**

> "A gap between two bars isn't a finding, so here's the evidence.
>
> My unit is one household's private-label share within one department —
> **[analysis_observations]** observations across **[households]** households.
> Household, not basket, because otherwise a few heavy shoppers drive everything.
>
> Top to bottom the gap is **[headline_gap_pp]**, and I'm 95% confident the true
> gap is **[headline_gap_CI]**. Those intervals are bootstrapped over households,
> because a household shows up in several departments.
>
> I ran **[comparisons_run]** pairwise comparisons and adjusted all of them with
> **[multiplicity_method]**. I'm deliberately not leading with p-values — at this
> sample size everything is significant. The number that matters is effect size:
> department explains **[eta_squared]** of the variation in household
> private-label share.
>
> One honest caveat. This could be shelf space rather than preference. So I
> compared private label's share of *dollars* against its share of *products on
> the shelf*. Where the ratio is above 1, shoppers are choosing private label
> more than availability alone would predict — that's preference. Where it's
> near 1, the department's number is really an assortment story, and the
> recommendation there is about what we stock, not who we target."

Handoff line: *"Which is why our recommendation targets [department] specifically —"*

---

## 7. Q&A — prepared answers

**"Why these departments?"**
> Two screens, both set before I looked at any result: at least 1% of chain
> dollars and at least 500 households. Then I took the largest by private-label
> revenue. I didn't pick the ones with the most flattering gap — the screen is
> in the code and I can show it.

**"Are the differences real, or just random?"**
> Real. The smallest gap I'm calling a difference is [X] percentage points and
> its confidence interval doesn't cross zero after adjusting for all
> [comparisons_run] comparisons. Two tests with different assumptions — Welch
> ANOVA and Kruskal–Wallis — agree, and an item-level chi-square on a completely
> different unit agrees too. What I *won't* claim is that every pair differs;
> [the non-significant pairs] are not distinguishable in this data.

**"Isn't that just because those items are cheaper?"**
> Partly, and that's measurable. I report the private-label discount per
> department and both metrics side by side. Item share runs above dollar share
> everywhere private label is cheaper. The ranking [does / doesn't] survive the
> switch — that's in my metric-sensitivity table.

**"Couldn't it just be a different type of shopper in each aisle?"**
> I checked. I restricted to the households that shop *all* the focus
> departments and compared them to themselves across aisles, which holds
> shopper mix constant by construction. The ordering [held / changed] — so the
> pattern is about the aisle, not about who walks into it.

**"What would make this a stronger claim?"**
> Three things: margin data, because private label's real advantage is
> percentage margin and I only have revenue; planogram data, so shelf space is
> measured rather than proxied by assortment breadth; and a shelf-space or
> promotion experiment in a few stores, because everything here is
> observational — I can say private label performs better in these aisles, not
> that moving shelf space would cause it to.

**"Why share of dollars rather than items or trips?"**
> Merchandising decisions are made in dollars and margin accrues on dollars.
> Item share over-weights cheap high-count products — a 39-cent soda counts the
> same as a $14 roast. Trip incidence saturates near 100%, so it can't
> discriminate between departments at all.

---

## 8. Pre-flight checklist

- [ ] Rmd knits clean from a fresh session
- [ ] Audit table shows a high % of dollars retained (if not, explain the loss)
- [ ] `BRAND` frequency table confirms "Private"/"National" matched correctly — **not all zeros**
- [ ] Every slide number traced to `out/slide_numbers.txt`
- [ ] Charts exported at 300 dpi, axes labelled, no console output on slides
- [ ] Both blanks in "[the non-significant pairs]" and "[X] percentage points" filled from the Tukey table
- [ ] Section timed at or under 2:30
- [ ] `.Rmd` submitted to Canvas with the team deck
