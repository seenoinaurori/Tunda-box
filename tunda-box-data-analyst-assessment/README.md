# Tunda Box Junior Data Analyst Assessment

**Candidate:** [Add your name]  
**Date:** [Add submission date]

This is a worked response to the fictional assessment. The source values are invented, as stated in the prompt.

## Submission links

- **GitHub:** [Tunda-box assessment folder](https://github.com/seenoinaurori/Tunda-box/tree/main/tunda-box-data-analyst-assessment)
- **Drive copy of this response:** [Add the Google Drive URL after uploading this README]
- **Solutions notebook:** [tunda_box_assessment_solutions.ipynb](./tunda_box_assessment_solutions.ipynb)

The assessment files are published in the linked GitHub repository. To complete the Drive step, upload this README to Google Drive, set the intended sharing permissions, and replace the placeholder above with its shareable link.

## Assumptions and approach

- Treat an order as uniquely identified by `order_id`. The repeated `1006` rows are identical, so remove the exact duplicate once. If duplicate IDs had conflicting values, investigate them rather than arbitrarily selecting a row.
- Revenue means the sum of non-null `amount_kes` for completed orders. Exclude canceled and refunded orders. Include completed order `1013` at KES 185,000 in the initial calculation, but flag it for validation rather than silently changing the source.
- For repeat purchase rate, count distinct completed order IDs per registered customer, including a completed order whose amount is missing; divide the number of registered customers with at least two such orders by all 10 customers.
- Normalize city labels by trimming whitespace and ignoring case for grouping. Keep unmatched orders visible as `Unmatched` rather than assigning them to a city.
- The sample is small and deliberately contains data-quality issues; figures are descriptive of this sample, not a basis for broad business claims.

## Part 1: Experience

### Your analysis

In a personal project, I explored a synthetic dataset of 1,000 Duka Electronics orders. I wanted to understand which branches and products led by revenue and whether those rankings matched order volume and units sold. The analysis showed that the revenue leaders were not always the volume leaders, so I changed the way I presented the results: I separated revenue, order count, and units instead of calling one branch or product the overall “best.” It was a simulated dataset, so it did not change a real company decision; it improved the clarity and usefulness of my recommendation.

### Conflicting numbers

I would first make sure we mean the same metric, time period, population, currency, and status rules. Then I would trace both numbers back to their source rows and compare filters, joins, duplicate handling, nulls, and calculation logic. I would reproduce the result with a small auditable query, agree on the definition with my colleague, document the resolution, and communicate any remaining uncertainty rather than choosing a number based on seniority.

## Part 2: SQL and Python

The SQL below uses PostgreSQL syntax. `SELECT DISTINCT` removes the exact duplicate row in the sample; it is not a substitute for investigating conflicting rows that share an order ID.

### S1. Orders by status

```sql
WITH orders_clean AS (
    SELECT DISTINCT order_id, customer_id, order_date, status, amount_kes, payment_method
    FROM orders
)
SELECT status, COUNT(DISTINCT order_id) AS order_count
FROM orders_clean
GROUP BY status
ORDER BY status;
```

| Status | Distinct orders |
|---|---:|
| cancelled | 2 |
| completed | 16 |
| refunded | 2 |

These are 20 unique orders. Counting raw rows instead would show 17 completed rows because order `1006` is duplicated.

### Revenue by city

#### (a) Highest completed-order revenue

```sql
WITH orders_clean AS (
    SELECT DISTINCT order_id, customer_id, order_date, status, amount_kes, payment_method
    FROM orders
)
SELECT COALESCE(NULLIF(LOWER(TRIM(c.city)), ''), 'unmatched') AS city,
       SUM(o.amount_kes) AS completed_revenue_kes
FROM orders_clean AS o
LEFT JOIN customers AS c ON c.customer_id = o.customer_id
WHERE o.status = 'completed'
GROUP BY 1
ORDER BY completed_revenue_kes DESC;
```

| City | Completed revenue (KES) |
|---|---:|
| Kisumu | 187,100 |
| Nairobi | 20,650 |
| Mombasa | 5,150 |
| Nakuru | 3,300 |
| Unmatched customer | 1,900 |

**Kisumu leads with KES 187,100** in the sample. This includes KES 185,000 from order `1013`, which is a likely outlier and needs source verification. If that amount is a data-entry error, the ranking and revenue should be recalculated after correction. The unmatched customer `999` contributes KES 1,900 to the unmatched bucket, not to a known city.

#### (b) What I would send in ten minutes

> Preliminary result: Kisumu has the highest completed-order revenue in this extract, KES 187,100, including a KES 185,000 order that looks unusual. I removed the exact duplicate order row and excluded canceled/refunded orders; I’m validating the unusually large amount and the unmatched customer before treating this as a final figure.

### S3. Customers who never ordered

```sql
SELECT c.customer_id
FROM customers AS c
WHERE NOT EXISTS (
    SELECT 1
    FROM orders AS o
    WHERE o.customer_id = c.customer_id
)
ORDER BY c.customer_id;
```

**Customers 109 and 110** have never placed an order in the extract. This checks for any order status, not just completed orders.

### S4. Repeat purchase rate

```sql
WITH orders_clean AS (
    SELECT DISTINCT order_id, customer_id, order_date, status, amount_kes, payment_method
    FROM orders
),
completed_orders_per_customer AS (
    SELECT customer_id, COUNT(DISTINCT order_id) AS completed_orders
    FROM orders_clean
    WHERE status = 'completed'
    GROUP BY customer_id
)
SELECT
    100.0 * COUNT(*) FILTER (WHERE COALESCE(x.completed_orders, 0) >= 2)
        / COUNT(*) AS repeat_buyer_pct
FROM customers AS c
LEFT JOIN completed_orders_per_customer AS x
    ON x.customer_id = c.customer_id;
```

**40% (4 of 10 registered customers)** are repeat buyers: customers 101, 102, 104, and 107 have at least two distinct completed orders. Customer 107 qualifies based on orders 1009 and 1019 even though order 1009 has a null amount; it is still marked completed. I use all registered customers as the denominator, so customers with no orders count as non-repeat buyers. This is a sample rate, not a cohort-adjusted or time-windowed retention measure.

### S5. Month-on-month completed revenue

```sql
WITH orders_clean AS (
    SELECT DISTINCT order_id, customer_id, order_date, status, amount_kes, payment_method
    FROM orders
),
monthly AS (
    SELECT DATE_TRUNC('month', order_date)::date AS month,
           SUM(amount_kes) AS revenue_kes
    FROM orders_clean
    WHERE status = 'completed'
    GROUP BY 1
),
with_previous AS (
    SELECT month,
           revenue_kes,
           LAG(revenue_kes) OVER (ORDER BY month) AS previous_revenue_kes
    FROM monthly
)
SELECT month, revenue_kes,
       100.0 * (revenue_kes - previous_revenue_kes)
           / NULLIF(previous_revenue_kes, 0) AS pct_change_from_previous_month
FROM with_previous
ORDER BY month;
```

| Month | Completed revenue (KES) | Change from previous month |
|---|---:|---:|
| 2026-06 | 9,000 | — |
| 2026-07 | 6,800 | -24.4% |
| 2026-08 | 192,000 | +2,723.5% |
| 2026-09 | 10,300 | -94.6% |

The August spike is driven by order `1013`; validate it before using the month-on-month trend for planning. If a month had no rows, this query would omit that month; a calendar table would be needed to show zero-revenue months.

## Part 3: Data quality and spreadsheets

### D1. Audit the data

- **Duplicate order ID 1006:** the complete row appears twice. Deduplicate the exact duplicate for the provisional analysis, then check the source system and enforce/monitor order-ID uniqueness. Conflicting duplicate IDs need investigation, not automatic deletion.
- **Missing completed amount:** order 1009 has a null amount. Retain the order for order counts, exclude null from revenue sums, and check the payment/order record before imputation.
- **Unusually large amount:** order 1013 is KES 185,000 versus other values near a few thousand. Validate against the source/invoice; do not winsorize or replace it without an agreed correction.
- **Orphan customer:** order 1020 references customer 999, who is absent from `customers`. Keep the amount in an unmatched bucket for company-wide totals, but do not attribute it to a city; investigate late-arriving or invalid customer data.
- **Inconsistent city formatting:** `Nairobi`, `nairobi`, and `Nairobi ` should be standardized for analysis (trim whitespace and normalize case), while preserving the original value for audit.
- **Canceled and refunded statuses:** exclude both from completed revenue, while reporting them separately and confirming whether refunded amounts or partial refunds need a separate net-revenue definition.
- **Limited sample and period:** this is a small extract with June–September orders and January–July signups. Check completeness, date boundaries, and representativeness before generalizing.

### D2. Spreadsheet formulas

Total completed M-Pesa revenue, assuming headers are in row 1:

```excel
=SUMIFS(E:E,D:D,"completed",F:F,"M-Pesa")
```

Add each order’s customer city to a new column in the orders sheet, starting on row 2:

```excel
=XLOOKUP(B2,customers!A:A,customers!B:B,"Unknown")
```

Fill the lookup formula down. If `XLOOKUP` is unavailable, use `=IFERROR(VLOOKUP(B2,customers!A:B,2,FALSE),"Unknown")`.

### D3. Mean or median

I would report the **median**, with the number of valid completed orders and an outlier note. After removing the exact duplicate and excluding the null amount, there are 15 valid completed order values; their median is **KES 2,400**. The KES 185,000 value would pull the mean to about KES 14,540, so the mean is not representative of the typical order until that value is validated.

## Part 4: Statistics

### T1. Concepts in plain language

#### (a) P-value

A p-value is the probability of getting a result at least this extreme if the “no real difference” assumption were true and the test assumptions held. A small p-value is evidence against that assumption; it is not the probability that the null hypothesis is true, nor does it tell us whether an effect matters commercially.

#### (b) SMS offers and spending

Not from this comparison alone. Customers who receive SMS offers may already differ in purchase frequency, loyalty, or expected spend, and those differences could explain the gap. To estimate causality, randomly assign eligible customers to receive or not receive the offer, measure outcomes over the same period, and check assignment and measurement quality.

### T2. Read an A/B test

Current conversion is **50/1,000 = 5%**; new conversion is **60/1,000 = 6%**. That is a **20% relative increase** but only a **1 percentage-point absolute increase**. A two-sided two-proportion test gives approximately `p = 0.33` (z ≈ 0.98), so this sample does not provide strong evidence that the underlying conversion rate increased. I would not ship solely on this result; check randomization, instrumentation, and guardrails, then continue the test or collect a suitably powered sample and report an interval as well as the point estimate.

## Part 5: Business case — falling conversion

### (a) What is happening?

| Channel | Sep conversion | Oct conversion | Session mix, Sep → Oct |
|---|---:|---:|---:|
| Organic | 2,400 / 40,000 = **6.0%** | 2,318 / 38,000 = **6.1%** | 80.0% → 55.9% |
| Paid Social | 200 / 10,000 = **2.0%** | 630 / 30,000 = **2.1%** | 20.0% → 44.1% |
| **All traffic** | 2,600 / 50,000 = **5.2%** | 2,948 / 68,000 = **4.34%** | 100% → 100% |

Both channels improved slightly, while lower-converting Paid Social supplied a much larger share of sessions. The aggregate conversion decline is therefore consistent with a **mix shift**, not evidence that Paid Social’s within-channel conversion got worse. I would also check spend, CAC/ROAS and contribution margin, campaign/creative/landing-page mix, audience and device mix, attribution windows, bot/invalid traffic, tracking or checkout changes, seasonality, and whether “orders” and “sessions” are defined consistently.

### (b) Recommendation (two sentences)

I would not pause Paid Social based on the blended conversion rate alone: its conversion rose from 2.0% to 2.1%, as did Organic’s from 6.0% to 6.1%, while Paid Social’s traffic share more than doubled. I recommend a rapid channel/campaign and unit-economics review (including spend, incremental orders, CAC/ROAS, and traffic quality) before changing budget, with a controlled holdout if we need to estimate incrementality.

## Part 6: Questions for the team

1. What would success look like for this analyst in the first 90 days?
2. How does the team define and document core metrics, and how are disagreements resolved?
3. How do analysts partner with Growth, Product, and Operations when results are uncertain or data quality is imperfect?
4. What tools and opportunities for mentorship or skills development are available?

## Reproducibility

Open `tunda_box_assessment_solutions.ipynb` and run the cells from top to bottom. It contains the sample data, pandas calculations, SQL examples, A/B-test arithmetic, and channel-mix calculation. All values are invented and used only for this assessment.
