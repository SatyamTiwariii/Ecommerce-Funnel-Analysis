# Marketing Funnel — Analytical Insights

This is the SQL analysis on the seller-acquisition side: how well the funnel converts leads, what kind of sellers it brings in, and whether acquisition quality shows up later in actual marketplace performance. All of it runs against the funnel model described in `06_star_schema/marketing/`.

## A. Funnel Overview

### 1. What's the overall MQL → closed deal conversion rate?

The single most important funnel metric — how many marketing-qualified leads actually turn into onboarded sellers.

```sql
SELECT
    COUNT(DISTINCT m.mql_id) AS total_mqls,
    COUNT(DISTINCT c.mql_id) AS closed_mqls,
    ROUND(
        COUNT(DISTINCT c.mql_id)::NUMERIC / NULLIF(COUNT(DISTINCT m.mql_id), 0), 4)
   AS mql_to_close_conversion_rate
FROM marketing_qualified_leads m
LEFT JOIN marketing_closed_deals c
    ON m.mql_id = c.mql_id;
```

## B. Conversion by Lead Origin

### 2. Which lead origins convert best?

Breaking the same conversion rate out by acquisition channel — this is what actually tells you where to spend more (or less) marketing effort.

```sql
SELECT
    COALESCE(m.origin, 'unknown_origin') AS origin,
    COUNT(DISTINCT m.mql_id) AS total_mqls,
    COUNT(DISTINCT c.mql_id) AS closed_mqls,
    ROUND(
        COUNT(DISTINCT c.mql_id)::NUMERIC / NULLIF(COUNT(DISTINCT m.mql_id), 0), 4)
      AS conversion_rate
FROM marketing_qualified_leads m
LEFT JOIN marketing_closed_deals c
    ON m.mql_id = c.mql_id
GROUP BY COALESCE(m.origin, 'unknown_origin')
ORDER BY conversion_rate DESC;
```

## C. Revenue by Seller Type

### 3. Which business types bring in the most declared revenue?

Conversion volume alone doesn't tell you if the funnel is bringing in *good* sellers — this looks at revenue by business segment instead.

```sql
SELECT
    business_type,
    COUNT(*) AS sellers_closed,
    ROUND(SUM(declared_monthly_revenue), 2) AS total_declared_revenue,
    ROUND(AVG(declared_monthly_revenue), 2) AS avg_declared_revenue
FROM marketing_closed_deals
GROUP BY business_type
ORDER BY total_declared_revenue DESC;
```

## D. Seller Quality Signals

### 4. Do sellers with a registered company earn more?

Testing whether `has_company` — a simple, easy-to-collect field at signup — actually correlates with higher declared revenue. If it does, it's a cheap early quality signal.

```sql
SELECT
    COALESCE(has_company::TEXT, 'unknown') AS has_company,
    COUNT(*) AS sellers,
    ROUND(AVG(declared_monthly_revenue), 2) AS avg_revenue
FROM marketing_closed_deals
GROUP BY COALESCE(has_company::TEXT, 'unknown')
ORDER BY avg_revenue DESC;
```

## E. Geography & Revenue

### 5. Which seller regions bring in higher average revenue?

Regional differences here matter for where acquisition effort might pay off more.

```sql
SELECT
    s.seller_state,
    COUNT(*) AS sellers,
    COALESCE(
        ROUND(
            AVG(CASE WHEN c.declared_monthly_revenue > 0 THEN c.declared_monthly_revenue ELSE NULL END), 2), 0)
    AS avg_revenue
FROM marketing_closed_deals c
JOIN ecommerce.olist_sellers s
    ON c.seller_id = s.seller_id
GROUP BY s.seller_state
ORDER BY avg_revenue DESC;
```

## F. Acquisition Quality vs. Real Operational Performance

### 6. Does a seller's business type predict how fast they actually deliver?

This is the query that connects the two halves of the whole project — pulling the acquisition data (marketing) together with the actual operational data (e-commerce) to see if what we know about a seller *before* onboarding tells us anything about how they perform *after*.

```sql
SELECT
    c.business_type,
    ROUND(
        AVG(o.order_delivered_customer_date::DATE - o.order_purchase_timestamp::DATE), 2)
   AS avg_delivery_days
FROM marketing_closed_deals c
JOIN ecommerce.olist_order_items oi
    ON c.seller_id = oi.seller_id
JOIN ecommerce.olist_orders o
    ON oi.order_id = o.order_id
WHERE o.order_delivered_customer_date IS NOT NULL
GROUP BY c.business_type
ORDER BY avg_delivery_days;
```

## What this section adds up to

- The overall conversion rate sets a baseline for how efficient the funnel is.
- Lead origin has a real, measurable effect on conversion quality — not all channels are equal.
- Revenue contribution varies a lot by business type, so raw conversion counts can be misleading on their own.
- Having a registered company at signup looks like a decent proxy for seller quality.
- Seller value differs meaningfully by region.
- Acquisition attributes carry through into operational performance — the clearest evidence in this project that marketing and operations aren't really separate problems.
