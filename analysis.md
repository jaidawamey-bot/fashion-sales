# Fashion Sales — Dataset Overview and Full Analysis

This document has two parts:

1. **Part A — What this dataset is** (for someone seeing it for the first time): coverage, structure, column meanings, how the columns depend on each other, and what's surprising.
2. **Part B — Full analysis**: seasonality, 2024 vs 2025, temperature effects, discounts, advertising, returns, and ratings — with the numbers behind every claim.

All figures come from `fashion_sales.csv` (240 rows). Charts referenced below live in `charts/`.

---

# Part A — What this dataset is

## A.1 What it covers

`fashion_sales.csv` is a monthly sales summary for a fashion retailer, covering **January 2024 through December 2025** (24 months). Instead of individual transactions, it is **pre-aggregated**: every row summarizes sales of one **product category** through one **sales channel** in one **month**.

- **5 categories**: Outerwear, Dresses, Footwear, Denim, Accessories
- **2 channels**: Online, Store
- **24 months** × 5 categories × 2 channels = **240 rows** — and every one of those 240 combinations appears **exactly once**.

So the file is a complete, balanced grid with no gaps: no missing month, no missing category/channel combination, no duplicate rows.

## A.2 How it is organized

Each row is one cell of that grid. The first three columns identify *when*, the next two identify *what* and *where*, and the remaining nine describe *how much* and *how well*.

| Column | Meaning | Notes |
|---|---|---|
| `month` | Period, `YYYY-MM` | 2024-01 … 2025-12 (24 values) |
| `year` | Year | 2024 or 2025 |
| `month_num` | Month number | 1–12 |
| `category` | Product category | One of 5 |
| `channel` | Sales channel | Online or Store |
| `avg_temp_F` | Average temperature that month, °F | 31.3–80.4 |
| `list_price` | Full retail price of the category | 35–180 |
| `discount_pct` | Discount applied, in percent | 0–40 |
| `sale_price` | Actual selling price per unit | 19.57–197.18 |
| `ad_spend` | Advertising spend for that cell | 2,050–19,270 |
| `avg_rating` | Average customer rating | 3.4–5.0 |
| `units_sold` | Units sold | 220–2,488 |
| `revenue` | Gross revenue | 17,257–154,512 |
| `return_rate` | Share of units returned | 0.01–0.356 (**a fraction, not a percent**) |

**Totals across the file:** ~13.2M revenue, 202,500 units, ~2.34M ad spend, blended return rate 17.3%.

## A.3 Which columns depend on each other

Several columns are **not independent** — they are derived from others. Understanding this is essential to interpreting the data correctly.

**1. `revenue` is exactly `units_sold × sale_price`.** This is a strict identity: it holds in all 240 rows with zero error. Revenue is not an independent measurement; it is computed from the other two.

**2. `sale_price` is basically `list_price` after the stated discount — plus noise.**
The formula `sale_price ≈ list_price × (1 − discount_pct / 100)` holds to within about ±10%: the relative error across all rows runs from −9.8% to +9.9% with a mean of ≈ 0 (−0.2%). So the discount is real, but the actual selling price jitters around the "theoretical" discounted price by up to a tenth either way.

**3. `list_price` depends only on `category`.** Each category has exactly one list price:

| Category | List price |
|---|---:|
| Accessories | 35 |
| Denim | 70 |
| Dresses | 85 |
| Footwear | 110 |
| Outerwear | 180 |

So `list_price` carries no row-level information beyond the category name.

**4. `avg_temp_F` depends only on `month`.** Every row for a given month has the same temperature (each month has exactly one distinct value). Temperature is therefore a **perfect proxy for the month** — it is not independent weather data. "Temperature effect" and "seasonality" are the same thing in this dataset.

**5. `year` and `month_num` are redundant with `month`.** They are just the two halves of the `month` string.

**6. The genuinely independent columns** are: `units_sold`, `ad_spend`, `avg_rating`, `return_rate`, and the noise component of `sale_price`. These are the fields that carry real, non-derived information.

**Practical consequence:** because `revenue = units × sale_price` and `sale_price ≈ list × (1 − discount)`, the columns `list_price`, `discount_pct`, `sale_price`, and `revenue` overlap heavily. Analysing "revenue vs discount" is partly analysing a formula against itself. The clean independent levers to study are `units_sold`, `ad_spend`, `avg_rating`, and `return_rate`.

## A.4 Surprising things worth flagging

- **`sale_price` can exceed `list_price`.** In **21 rows** the actual selling price is above the full retail price — all of them with `discount_pct = 0`. This is a side effect of the ±10% price noise: at 0% discount, positive noise pushes the price above list. (E.g. 2024-09 Outerwear Online: list 180, discount 0, sale 197.18.)
- **43 rows have `discount_pct = 0` but `sale_price ≠ list_price`** — the same noise, in both directions.
- **`discount_pct` is not continuous** — it only ever takes 7 values: 0, 10, 15, 20, 25, 30, 40.
- **`return_rate` is a fraction (0.01–0.356), not a percentage.** This is easy to misread, especially sitting next to `discount_pct`, which *is* a percentage (0–40).
- **Ratings barely move.** The whole `avg_rating` column spans 3.4–5.0 with a standard deviation of only **0.282**, and every category sits between 4.18 and 4.25. There is almost no rating signal to explain.
- **One row looks hand-edited.** `2025-04, Accessories, Store` is the only row where `avg_rating` is written as the integer `5` (not `5.0`) and the only row where `ad_spend` has a decimal (`8770.0`). Cosmetic, but it stands out.
- **The data appears synthetic.** A perfect 24×5×2 grid, an exact `revenue = units × price` identity, prices that are a fixed list price with bounded ±10% noise, and temperature that is a pure function of month all point to generated data. That's fine for demonstrating an analysis workflow, but it means the patterns are the ones that were designed in, and no real-world causal claims should be drawn from them.
- **The Online/Store return gap is enormous** — 24.6% vs 9.9% — which is the single most striking real pattern in the file (details in Part B).

## A.5 How to read the rest of this document

Every correlation below is Pearson's *r* unless stated. Because seasonality is strong and confounds many relationships, key findings are re-tested with **within-group correlations** (centering the data inside each month, or inside each category × channel, to remove those effects). A finding is only trusted if it survives those tests.

---

# Part B — Full analysis

## B.1 Seasonal patterns

Revenue is strongly seasonal, and the seasonality is driven by two categories pulling in opposite directions, plus a holiday lift for everything.

Seasonal index = that month's average revenue ÷ the category's overall monthly average (1.00 = average month).

| Category | Peak month (index) | Trough month (index) | Peak/trough swing |
|---|---|---|---|
| Outerwear | Dec (1.69) | Jul (0.52) | 3.3× |
| Dresses | Jul (1.69) | Feb (0.51) | 3.3× |
| Footwear | Dec (1.39) | Feb (0.79) | 1.8× |
| Denim | Dec (1.25) | Apr (0.75) | 1.7× |
| Accessories | Nov (1.30) | May (0.84) | 1.6× |

Full monthly indices:

- **Outerwear:** Jan 1.36, Feb 1.16, Mar 1.24, Apr 0.92, May 0.83, Jun 0.55, Jul 0.52, Aug 0.60, Sep 0.69, Oct 0.89, Nov 1.55, Dec 1.69
- **Dresses:** Jan 0.55, Feb 0.51, Mar 0.71, Apr 1.00, May 1.08, Jun 1.12, Jul 1.69, Aug 1.27, Sep 1.13, Oct 1.04, Nov 1.05, Dec 0.84
- **Footwear:** Jan 0.86, Feb 0.79, Mar 0.91, Apr 0.87, May 0.95, Jun 1.00, Jul 0.98, Aug 1.05, Sep 0.95, Oct 0.95, Nov 1.28, Dec 1.39
- **Denim:** Jan 0.94, Feb 1.03, Mar 0.91, Apr 0.75, May 1.13, Jun 1.03, Jul 0.92, Aug 0.95, Sep 0.86, Oct 1.00, Nov 1.24, Dec 1.25
- **Accessories:** Jan 0.98, Feb 0.96, Mar 0.94, Apr 1.02, May 0.84, Jun 0.95, Jul 0.96, Aug 0.97, Sep 0.96, Oct 0.88, Nov 1.30, Dec 1.24

**Findings**

- **Outerwear is a cold-weather business** (Dec index 1.69 vs Jul 0.52) and **Dresses is a warm-weather business** (Jul 1.69 vs Feb 0.51). Their swings are nearly mirror images.
- **Footwear, Denim, and Accessories are comparatively stable** (all indices within 0.75–1.4) but share a **Nov–Dec holiday bump** (Footwear 1.28/1.39, Denim 1.24/1.25, Accessories 1.30/1.24).
- **Discounting is pro-cyclical, not clearance-style.** Average discount by month is *lowest* in Feb (13.5%) and Sep (14.0%) and *highest* in Jan (32.5%), Jul (33.5%), Nov (28.3%), and Dec (32.5%). The correlation between a month's average discount and its average units is **r = 0.862** — discounts cluster in the months that already sell the most, which is the opposite of using discounts to rescue weak months.

*Chart: `charts/01_revenue_by_month_category.png`*

## B.2 2024 vs 2025

| Metric | 2024 | 2025 | Change |
|---|---:|---:|---:|
| Revenue | 6,406,734 | 6,813,748 | **+6.4%** |
| Units | 97,712 | 104,788 | **+7.2%** |
| Ad spend | 1,098,180 | 1,239,350 | **+12.9%** |
| Avg discount | 22.88% | 22.25% | −0.63 pts |
| Avg rating | 4.238 | 4.200 | −0.037 |
| Return rate (unit-weighted) | 17.2% | 17.9% | +0.72 pts |
| Revenue per unit | 65.57 | 65.02 | −0.8% |

**Revenue by category, year over year**

| Category | 2024 | 2025 | Change |
|---|---:|---:|---:|
| Outerwear | 1,968,818 | 2,077,174 | +5.5% |
| Dresses | 1,435,049 | 1,560,027 | +8.7% |
| Footwear | 1,362,279 | 1,432,204 | +5.1% |
| Denim | 1,008,615 | 1,070,034 | +6.1% |
| Accessories | 631,974 | 674,307 | +6.7% |

**Findings**

- **Growth was volume-driven, not price-driven.** Units grew faster than revenue (+7.2% vs +6.4%), and revenue per unit actually *fell* 0.8% (65.57 → 65.02). 2025 sold more for slightly less each.
- **Ad efficiency declined.** Ad spend grew almost twice as fast as revenue (12.9% vs 6.4%). Blended ROAS fell from **5.83 (2024) to 5.50 (2025)**, a −5.8% decline.
- **Every category grew** (range +5.1% to +8.7%). No category shrank.
- **Returns edged up, entirely in Store** (9.7% → 10.2%); Online was flat at 24.6% in both years.
- **Weather confound:** 2025 was colder in 9 of 12 months (Jan −4.9°F, Aug −4.7°F, Feb −3.6°F), which would modestly favour cold-weather categories in 2025. So the year-over-year comparison is not fully like-for-like.

## B.3 How temperature affects each category

Using 24 monthly points per category (units and revenue summed across channels):

| Category | r(temp, units) | r(temp, revenue) | Slope (units/°F) | R² |
|---|---:|---:|---:|---:|
| Outerwear | **−0.823** | **−0.894** | −32.6 | 0.68 |
| Dresses | **+0.677** | **+0.847** | +37.2 | 0.46 |
| Denim | −0.347 | −0.300 | −8.7 | 0.12 |
| Accessories | −0.296 | −0.344 | −7.8 | 0.09 |
| Footwear | −0.263 | −0.074 | −6.4 | 0.07 |

**Findings**

- **Temperature strongly drives only two categories.** Outerwear (r = −0.82 on units, −0.89 on revenue) and Dresses (r = +0.68 / +0.85) are the real weather plays. A month 10°F warmer is associated with roughly **326 fewer Outerwear units** and **372 more Dresses units**.
- **Footwear is essentially weather-insensitive** (r = −0.26 on units, −0.07 on revenue).
- **Denim and Accessories show weak, mildly negative relationships** (r ≈ −0.3), i.e. they sell slightly better in cooler months, but with R² under 0.12 this is not a strong pattern.
- **Critical caveat:** temperature is a deterministic function of month, so this *is* the seasonality from B.1. The two cannot be separated in this dataset.

*Chart: `charts/04_units_vs_temp_by_category.png`*

## B.4 Do bigger discounts sell more units?

**Yes — and the relationship survives removing seasonality.**

| Test | r(discount, units) |
|---|---:|
| Raw, all 240 rows | **+0.513** |
| Within-month (season removed) | **+0.391** |
| Within category × channel | **+0.606** |

Mean units by discount level:

| Discount | n | Mean units |
|---|---:|---:|
| 0% | 43 | 586 |
| 10% | 12 | 607 |
| 15% | 13 | 812 |
| 20% | 45 | 784 |
| 25% | 28 | 802 |
| 30% | 46 | 911 |
| 40% | 53 | 1,129 |

**Findings**

- **Units rise monotonically with discount**, from 586 at 0% off to 1,129 at 40% off — a **1.93×** increase, with a steady climb at every level in between.
- **It holds within every category**: Denim r = 0.748, Footwear 0.694, Outerwear 0.569, Accessories 0.552, Dresses 0.550 — and within both channels (Online 0.587, Store 0.454).
- **But more units ≠ more revenue.** corr(discount, revenue) is only **+0.202** overall, and **negative for Accessories (−0.132)**. Discounts reliably lift volume, but the price cut eats much of the gain, and for cheap high-volume Accessories it destroys revenue outright.
- **Caveat on direction:** because discounting is pro-cyclical (r = 0.862 between monthly discount and monthly units), some of the discount↔units link reflects "both are high in peak months." The within-month test (r = 0.391) still supports a genuine within-month association, but the data cannot prove that discounting *causes* the extra volume.

*Chart: `charts/03_units_vs_discount.png`*

## B.5 Does ad spending pay off?

**Weakly and inconsistently — this is the least convincing lever in the data.**

- corr(ad_spend, revenue) = **0.208**; OLS gives `revenue = 43,063 + 1.23 × ad_spend`, with **R² = 0.043** — spend explains only ~4% of revenue variation.
  *(The 1.23 slope is not a ROAS; it is the marginal relationship given how little spend varies. Average ROAS is 5.65.)*
- Within-month r = 0.226; within category × channel r = 0.261. The relationship stays weak however it is sliced.
- corr(ad_spend, units_sold) = 0.252.

ROAS (gross revenue per dollar of ad spend):

| Segment | ROAS | Net-of-returns ROAS |
|---|---:|---:|
| Outerwear | 8.52 | — |
| Dresses | 6.57 | — |
| Footwear | 6.18 | — |
| Denim | 4.25 | — |
| Accessories | 2.81 | — |
| **Online** | 5.20 | **3.92** |
| **Store** | 6.27 | **5.65** |

**Findings**

- **Average ROAS looks healthy (5.65×) but varies widely** — from 8.52 (Outerwear) down to 2.81 (Accessories).
- **Spend is nearly flat (2,050–19,270) while revenue swings ~9×.** Revenue is driven by season, category, and price — not by ad dollars. There is no evidence that marginal ad spend drives sales.
- **Accessories is the weakest advertiser** at 2.81× gross — it would need a near-2× efficiency gain just to reach parity.
- **No cost/margin data exists**, so ROAS here is gross revenue per ad dollar, not profit.

*Chart: `charts/05_revenue_vs_ad_spend.png`*

## B.6 What is related to returns?

**Channel is overwhelmingly the dominant factor.**

Return rate (unit-weighted) by category and channel:

| Category | Online | Store |
|---|---:|---:|
| Footwear | **29.7%** | 14.7% |
| Dresses | **28.9%** | 14.9% |
| Denim | 22.3% | 6.6% |
| Accessories | 21.2% | 6.5% |
| Outerwear | 21.1% | 7.0% |
| **Overall** | **24.6%** | **9.9%** |

**Findings**

- **Online returns are 2.5× Store returns** (24.6% vs 9.9%). Every category returns far more online than in store.
- **Category matters mainly within a channel**: Online spans 21.1–29.7% while Store spans 6.5–14.9%. Dresses and Footwear are worst in both channels.
- **Discounts do not drive returns.** Return rate by discount band is essentially flat: 0% off → 19.2%, 1–19% → 17.8%, 20–29% → 17.2%, 30–40% → 17.4%.
- **Ad spend is a spurious correlate.** Raw r = 0.228, but within-channel it collapses (Online −0.127, Store −0.006) — the raw link only reflects that Online has both more ads and more returns.
- **Price tier** shows the highest returns in the middle (list $85 and $110 → 22.4%) and lower at the extremes ($35 → 14.1%, $180 → 14.4%), but that pattern is really just "Dresses and Footwear."
- **Temperature is unrelated** (r = −0.004).
- **The one real internal driver is rating**: corr(return_rate, avg_rating) = **−0.353** raw, strengthening to **−0.540** within category × channel. Lower-rated combinations are returned more.

*Chart: `charts/06_return_rate_category_channel.png`*

## B.7 What is related to ratings?

**Almost nothing — ratings are near-constant, and the only meaningful relationship is with returns.**

- Spread is tiny: range 3.4–5.0, **standard deviation 0.282**; every category is between 4.18 and 4.25.
- **Channel matters most**: Store 4.288 vs Online 4.149 (+0.14).
- **Discounts don't matter**: corr(rating, discount) = −0.013; mean rating by discount band is flat (4.18–4.26).
- **Price doesn't matter**: corr(rating, sale_price) = −0.014.
- **Ad spend doesn't matter**: corr = −0.085.
- **The one real link is returns**: corr(rating, return_rate) = **−0.353** raw, **−0.540** within category × channel. Ratings and returns move together; the direction of causation is unclear.

Rating by channel and category (means):

| | Rating |
|---|---:|
| Store (all) | 4.288 |
| Online (all) | 4.149 |
| Accessories | 4.252 |
| Outerwear | 4.229 |
| Dresses | 4.227 |
| Footwear | 4.202 |
| Denim | 4.183 |

*Chart: `charts/07_avg_rating_category_channel.png`*

## B.8 Cross-channel comparison, net of returns

Because `revenue` is gross (it does not subtract returns), the headline channel numbers are misleading. Applying each channel's return rate:

| | Online | Store |
|---|---:|---:|
| Gross revenue | 6,966,152 | 6,254,329 |
| Units | 105,687 | 96,813 |
| Return rate | 24.6% | 9.9% |
| **Net-of-returns revenue** | **5,251,290** | **5,632,511** |
| Ad spend | 1,340,200 | 997,330 |
| Gross ROAS | 5.20 | 6.27 |
| **Net ROAS** | **3.92** | **5.65** |
| Avg rating | 4.149 | 4.288 |

**Finding:** although Online shows higher *gross* revenue, once returns are removed **Store out-earns Online (5.63M vs 5.25M)** and its net ROAS is 5.65 vs 3.92. The Online revenue lead is largely illusory.

## B.9 Headline takeaways

1. **The business is seasonal and weather-bound.** Outerwear and Dresses swing ~3.3× between peak and trough; Footwear, Denim, and Accessories are comparatively stable with a shared Nov–Dec holiday lift.
2. **Growth is real but getting less efficient.** 2025 revenue +6.4% and units +7.2%, but ad spend +12.9% and ROAS down from 5.83 to 5.50.
3. **Discounts move units, not revenue.** Strong volume link (within-month r = 0.39), weak revenue link (r = 0.20), negative for Accessories.
4. **Returns are the biggest value leak.** 24.6% Online vs 9.9% Store. Net of returns, Store out-earns Online and is far more ad-efficient.
5. **Ad spend barely predicts revenue** (R² = 0.04). It is not the growth driver.
6. **Ratings tell you almost nothing** except as a weak mirror of returns (r = −0.35, up to −0.54 within group).

## B.10 Caveats and method notes

- **`revenue` is gross**, not net of returns, and is exactly `units_sold × sale_price`. Every revenue figure in this document is therefore overstated by roughly the return rate (~17% blended).
- **Temperature is perfectly collinear with month**, so seasonal and temperature effects cannot be separated.
- **Several columns are redundant** (`year`/`month_num` vs `month`; `list_price` vs `category`; `revenue` vs `units × sale_price`), so some "relationships" are partly definitional.
- **The data appears synthetic** (perfect balanced grid, exact revenue identity, bounded price noise, temperature as a pure function of month). Correlations are descriptive, not causal, and there is no cost/margin data to assess profitability.
- **Method:** Pearson correlations on the 240 rows unless stated; key findings re-tested with within-group (fixed-effects) correlations to remove month and category × channel effects; a simple OLS regression for ad spend vs revenue.
