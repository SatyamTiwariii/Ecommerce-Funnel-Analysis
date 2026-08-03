# E-Commerce Analytical Insights

This is the SQL analysis on top of the Olist e-commerce data — the part where the schema and validation work from the earlier folders actually gets used to answer business questions. Every query here runs against the validated tables modeled in `06_star_schema/ecommerce/`.

Each section below is one business question, a sentence on why it matters, and the query I used to answer it.

## A. Customer & Geography

### 1. Which states have the most customers?

Useful for spotting where demand is concentrated, and for deciding where to focus regional efforts.

```sql
select customer_state, count(distinct customer_unique_id) as occurrences
from olist_customers
group by customer_state
order by occurrences desc;
```

### 2. Which cities generate the most orders?

Order volume by city matters for logistics planning and localized marketing — this joins customers to orders and counts by city.

```sql
select c.customer_city, count(o.order_id) as order_count
from olist_customers  c
join olist_orders o
    on c.customer_id = o.customer_id
group by c.customer_city 
order by order_count desc;
```

### 3. What share of customers come back for a second order?

Repeat customers are worth more over time than one-off buyers, so I wanted a straight percentage of customers who ordered more than once.

```sql
with order_counts as (
    select customer_unique_id, count(order_id) as order_count
    from olist_orders o
    join olist_customers c
        on c.customer_id = o.customer_id
    group by customer_unique_id	
)
select 
    round(count(case when order_count > 1 then 1 end) * 100.00 / count(*), 2) 
        as repeat_customer_percentage
from order_counts;
```

### 4. On average, how many orders does a customer place?

A quick read on overall engagement/purchase frequency.

```sql
select round(avg(order_count), 2) as avg_no_of_orders
from (
    select customer_id, count(order_id) as order_count 
    from olist_orders 
    group by customer_id
) s;
```

### 5. Which states bring in the most revenue?

Revenue concentration by geography is a useful input for where to prioritize delivery infrastructure.

```sql
select c.customer_state, sum(oi.price) as total_revenue
from olist_customers  c
join olist_orders o
    on c.customer_id = o.customer_id
join olist_order_items oi
    on o.order_id = oi.order_id
group by c.customer_state
order by total_revenue desc;
```

### 6. Top 10 customers by total spend (a rough CLV view)

Not a full lifetime-value model, but a simple sum of spend per customer to see who the highest-value buyers actually are.

```sql
select c.customer_unique_id, sum(oi.price) as customer_spending
from olist_customers  c
join olist_orders o
    on c.customer_id = o.customer_id
join olist_order_items oi
    on o.order_id = oi.order_id
group by c.customer_unique_id
order by customer_spending desc
limit 10;
```

## B. Order & Delivery Performance

### 1. What's the average time from purchase to delivery?

The headline logistics metric — how long customers actually wait.

```sql
select 
    date_trunc('day', avg(order_delivered_customer_date - order_purchase_timestamp)) as avg_days
from olist_orders;
```

### 2. How far off are deliveries from the estimated date?

This gives a per-order gap between what was promised and what actually happened — positive values mean late, negative means early.

```sql
select 
    order_id, 
    order_delivered_customer_date, 
    order_estimated_delivery_date,
    (order_delivered_customer_date - order_estimated_delivery_date) as estimate_and_actual_diff
from olist_orders;
```

### 3. How many orders were delivered late overall?

A simple count of everything that missed its estimated date.

```sql
select count(order_id)
from olist_orders
where order_delivered_customer_date > order_estimated_delivery_date;
```

### 4. Actual vs. estimated delivery duration, side by side

This builds out both timelines per order so they can be compared directly — feeds into SLA-style analysis later.

```sql
select
    order_id,
    order_purchase_timestamp,
    order_delivered_customer_date,
    order_estimated_delivery_date,
    (order_delivered_customer_date - order_purchase_timestamp) as actual_delivery_days,
    (order_estimated_delivery_date - order_purchase_timestamp) as estimated_delivery_days
from olist_orders
where order_delivered_customer_date is not null;
```

### 5. Which sellers ship fastest?

Average time between payment approval and handing the order to the carrier, ranked fastest-first.

```sql
select 
    oi.seller_id,
    date_trunc('second', avg(o.order_delivered_carrier_date - o.order_approved_at)) as avg_shipping_days
from olist_orders o
join olist_order_items oi
    on o.order_id = oi.order_id
where o.order_delivered_carrier_date is not null
  and o.order_approved_at is not null
  and o.order_delivered_carrier_date >= o.order_approved_at
group by oi.seller_id
order by avg_shipping_days;
```

### 6. Which sellers are slowest to ship?

Same query, sorted the other way — worth flagging for marketplace quality/operational review.

```sql
select 
    oi.seller_id,
    date_trunc('second', avg(o.order_delivered_carrier_date - o.order_approved_at)) as avg_shipping_days
from olist_orders o
join olist_order_items oi
    on o.order_id = oi.order_id
where o.order_delivered_carrier_date is not null
  and o.order_approved_at is not null
  and o.order_delivered_carrier_date >= o.order_approved_at
group by oi.seller_id
order by avg_shipping_days desc;
```

### 7. Which states have the slowest overall delivery times?

Distance and regional logistics capacity both show up here.

```sql
select 
    c.customer_state, 
    date_trunc('second', avg(o.order_delivered_customer_date - o.order_purchase_timestamp)) 
        as avg_delivery_days
from olist_customers c
left join olist_orders o
    on o.customer_id = c.customer_id
where o.order_delivered_customer_date is not null
group by c.customer_state
order by avg_delivery_days desc;
```

### 8. Does delivery time vary by product category?

Some product types are naturally slower to prepare or ship than others — this checks whether that shows up in the data.

```sql
select 
    p.product_category_name,
    date_trunc('second', avg(o.order_delivered_customer_date - o.order_purchase_timestamp))
        as avg_delivery_duration
from olist_orders o
join olist_order_items oi 
    on o.order_id = oi.order_id
join olist_products p
    on p.product_id = oi.product_id
where o.order_delivered_customer_date is not null
group by p.product_category_name
order by avg_delivery_duration;
```

## C. Product & Category Performance

### 1. Which categories sell the most units?

Straightforward volume ranking — helps spot the categories carrying the marketplace.

```sql
select 
    p.product_category_name,
    pct.product_category_name_english,
    count(*) as units_sold
from olist_order_items oi
join olist_products p
    on oi.product_id = p.product_id
left join product_category_name_translation pct
    on p.product_category_name = pct.product_category_name
group by p.product_category_name, pct.product_category_name_english
order by units_sold desc;
```

### 2. Which categories generate the most revenue?

Volume and revenue don't always agree — this checks revenue specifically.

```sql
select 
    p.product_category_name,
    pct.product_category_name_english, 
    sum(oi.price) as total_revenue
from olist_order_items oi
join olist_products p
    on oi.product_id = p.product_id
left join product_category_name_translation pct
    on p.product_category_name = pct.product_category_name
group by p.product_category_name, pct.product_category_name_english
order by total_revenue desc;
```

### 3. What's the average price per category?

Useful for telling premium categories apart from budget ones.

```sql
select 
    p.product_category_name, 
    pct.product_category_name_english, 
    round(avg(oi.price),2) as avg_price
from olist_products p
join olist_order_items oi
    on oi.product_id = p.product_id
left join product_category_name_translation pct
    on p.product_category_name = pct.product_category_name
group by p.product_category_name, pct.product_category_name_english
order by avg_price;
```

### 4. Which category costs the most to ship?

Average freight value per category — a proxy for logistics cost by product type.

```sql
select 
    p.product_category_name, 
    pct.product_category_name_english, 
    round(avg(freight_value),2) as avg_freight_charges
from olist_order_items oi
join olist_products p
    on oi.product_id = p.product_id
left join product_category_name_translation pct
    on p.product_category_name = pct.product_category_name
group by p.product_category_name, pct.product_category_name_english
order by avg_freight_charges desc;
```

### 5. Does a heavier product cost more?

I was curious whether weight is actually a decent proxy for price. Started with a straight correlation, then bucketed weight into ranges for a clearer read.

```sql
select 
    corr(p.product_weight_g, oi.price) as weight_price_correlation 
from olist_products p 
join olist_order_items oi 
    on p.product_id = oi.product_id 
where p.product_weight_g is not null
  and oi.price is not null;
```

```sql
select
    case 
        when p.product_weight_g < 500  then '0–500g'
        when p.product_weight_g < 1000 then '500–1000g'
        when p.product_weight_g < 2000 then '1000–2000g'
        when p.product_weight_g < 5000 then '2000–5000g'
        else '5000g+'
    end as weight_bucket,
    round(avg(oi.price), 2) as avg_price
from olist_products p
join olist_order_items oi
    on p.product_id = oi.product_id
where p.product_weight_g is not null
group by weight_bucket
order by avg_price;
```

### 6. Which categories have the worst late-delivery rate?

A category-level lens on delivery reliability — good for spotting supply-chain bottlenecks tied to specific product types.

```sql
select 
    p.product_category_name, 
    pct.product_category_name_english,
    round(count(case when o.order_delivered_customer_date > o.order_estimated_delivery_date 
        then 1 end) * 100.0 / count(*), 2) as late_delivery_percentage
from olist_orders o
join olist_order_items oi
    on o.order_id = oi.order_id
join olist_products p
    on p.product_id = oi.product_id
left join product_category_name_translation pct
    on p.product_category_name = pct.product_category_name
where o.order_delivered_customer_date is not null
group by p.product_category_name, pct.product_category_name_english
order by late_delivery_percentage;
```

### 7. Top 10 most profitable categories

Profitability here is just price minus freight — a rough but honest measure of margin after shipping cost.

```sql
select 
    p.product_category_name, 
    pct.product_category_name_english, 
    sum(oi.price - oi.freight_value) as profit
from olist_products p
join olist_order_items oi
    on p.product_id = oi.product_id
left join product_category_name_translation pct
    on p.product_category_name = pct.product_category_name
group by p.product_category_name, pct.product_category_name_english
order by profit desc
limit 10;
```

## D. Payments

### 1. What payment types are most common?

A basic count of how customers actually pay.

```sql
select payment_type, count(*) as count_payment_type
from olist_order_payments
group by payment_type
order by count_payment_type desc;
```

### 2. What installment ranges show up most?

Grouped installment counts into ranges rather than looking at every raw number — easier to read and more useful for planning.

```sql
select
    case
        when payment_installments = 1 then '1 installment'
        when payment_installments between 2 and 3 then '2–3 installments'
        when payment_installments between 4 and 6 then '4–6 installments'
        when payment_installments between 7 and 12 then '7–12 installments'
        else '13+ installments'
    end as installment_range,
    count(*) as installment_count
from olist_order_payments
group by installment_range
order by installment_count desc;
```

### 3. What percentage of orders use more than one installment?

A read on how reliant customers are on split/credit-based payments.

```sql
select 
    round(count(case when payment_installments > 1 then 1 end) * 100.0 / count(*), 2) 
        as multiple_installment_percent
from olist_order_payments;
```

### 4. How much revenue comes through each payment type?

Ties payment method back to actual dollars, not just order counts.

```sql
select 
    op.payment_type, 
    sum(oi.price) as revenue
from olist_order_items oi
join olist_order_payments op
    on oi.order_id = op.order_id
group by op.payment_type
order by revenue desc;
```

### 5. Average order value by payment type

Checks whether certain payment methods (credit card especially) tend to come with bigger baskets.

```sql
with order_total as (
    select order_id, sum(price) as order_value
    from olist_order_items
    group by order_id
)
select 
    payment_type, 
    round(avg(order_value), 2) as avg_order_value
from order_total o
join olist_order_payments p 
    on o.order_id = p.order_id
group by payment_type
order by avg_order_value desc;
```

## E. Reviews & Satisfaction

### 1. What's the average review score across the board?

A single baseline number for overall satisfaction.

```sql
select avg(review_score)
from olist_order_reviews;
```

### 2. Do late deliveries actually drag down review scores?

This was the question I most wanted an answer to in this section — comparing average scores between late and on-time orders directly.

```sql
select
    case 
        when o.order_delivered_customer_date > o.order_estimated_delivery_date then 'Late Delivery'
        else 'On-Time Delivery'
    end as delivery_status,
    round(avg(r.review_score), 2) as avg_review_score
from olist_orders o
join olist_order_reviews r
    on o.order_id = r.order_id
where o.order_delivered_customer_date is not null
group by delivery_status;
```

### 3. How are review scores distributed?

Counts per rating (1–5) — shows whether feedback is polarized or clustered.

```sql
select 
    review_score, 
    count(*) as count_review
from olist_order_reviews
group by review_score
order by review_score;
```

### 4. Which categories get the worst ratings?

Could point to quality issues or listings that oversell the product.

```sql
select 
    p.product_category_name, 
    pct.product_category_name_english, 
    round(avg(review_score),2) as avg_review_score 
from olist_order_items oi
join olist_products p 
    on oi.product_id = p.product_id
join olist_order_reviews r
    on r.order_id = oi.order_id
left join product_category_name_translation pct
    on p.product_category_name = pct.product_category_name
group by p.product_category_name, pct.product_category_name_english
order by avg_review_score;
```

### 5. Best- and worst-rated sellers

Two versions of the same query, sorted opposite ways — useful for marketplace quality monitoring.

**Best-rated**
```sql
select 
    seller_id, 
    round(avg(review_score),2) as average_rating
from olist_order_items oi
join olist_order_reviews r
    on oi.order_id = r.order_id
group by seller_id
order by average_rating desc;
```

**Worst-rated**
```sql
select 
    seller_id, 
    round(avg(review_score),2) as average_rating
from olist_order_items oi
join olist_order_reviews r
    on oi.order_id = r.order_id
group by seller_id
order by average_rating asc;
```

### 6. How quickly do customers leave reviews after delivery?

Average time between the review being created and the seller's response — a proxy for post-purchase engagement speed.

```sql
select 
    date_trunc('second', avg(review_answer_timestamp - review_creation_date))
        as avg_time_to_respond
from olist_order_reviews
where review_answer_timestamp is not null 
  and review_creation_date is not null;
```
