# E-Commerce Data Model — Star Schema

## Why a star schema at all

The raw Olist dataset ships as 9 normalized tables. That's fine for an OLTP system, but it's painful for analysis — every question ends up needing four or five joins, and it's easy to introduce fan-out bugs (double-counting rows) if the joins aren't done carefully. Reshaping it into a star schema fixes that: queries get simpler, BI tools like Power BI handle it natively, and the business entities (customer, product, seller, order) are grouped in a way that actually matches how people ask questions about the business.

## The shape of it

**1 fact table**
- `Fact_Order_Items`

**8 dimension tables**
- `Dim_Customers`, `Dim_Products`, `Dim_Sellers`, `Dim_Orders`, `Dim_Order_Payments`, `Dim_Order_Reviews`, `Dim_Geolocation`, `Dim_Product_Category_Translation`

Everything hangs off `Fact_Order_Items` because every real transaction touches a customer, a product, a seller, a payment, a delivery, and (usually) a review.

### Why the fact table is order *items*, not orders

I went with item-level grain rather than order-level, because a single order can contain multiple products, each with its own price, freight cost, and seller. Almost every metric worth computing here — GMV, revenue, freight cost, top-selling products — genuinely lives at the item level, not the order level. Rolling up to orders too early would have thrown away information I needed later.

## Fact table: `fact_order_items`

**Grain:** one row = one product item within one order. A 3-item order produces 3 rows.

| Field | Description |
|-------|-------------|
| order_id | Links to order-level attributes |
| order_item_id | Item number within the order |
| customer_id | Buyer of the item |
| product_id | Product purchased |
| seller_id | Seller who shipped the item |
| price | Item price |
| freight_value | Shipping fee |
| payment_value | Payment amount associated with the order |
| review_score | Customer review rating |
| purchase_date | Order purchase date |
| delivered_date | Final delivery date |
| estimated_delivery_date | Expected delivery date |

This is the logical query that defines the fact table (not something I physically materialized — see the note at the bottom):

```sql
SELECT 
      oi.order_id,
      oi.order_item_id,
      o.customer_id,
      oi.product_id,
      oi.seller_id,
      oi.price,
      oi.freight_value,
      op.payment_value,
      r.review_score,
      o.order_purchase_timestamp::date AS purchase_date,
      o.order_delivered_customer_date::date AS delivered_date,
      o.order_estimated_delivery_date::date AS estimated_delivery_date
FROM olist_order_items oi
LEFT JOIN olist_orders o ON oi.order_id = o.order_id
LEFT JOIN olist_order_payments op ON oi.order_id = op.order_id
LEFT JOIN olist_order_reviews r ON oi.order_id = r.order_id;
```

## Dimension tables

**`Dim_Customers`**

| Field | Description |
|---|---|
| customer_id | Unique per order |
| customer_unique_id | Unique per actual person |
| customer_zip_code_prefix | ZIP prefix |
| customer_city | Customer city |
| customer_state | Customer state |

**`Dim_Products`**

| Field | Description |
|---|---|
| product_id | Unique product identifier |
| product_category_name | Category (Portuguese) |
| product_category_name_english | Translated category |
| product_weight_g | Product weight |
| product_length_cm / product_height_cm / product_width_cm | Product dimensions |

**`Dim_Sellers`**

| Field | Description |
|---|---|
| seller_id | Unique seller ID |
| seller_zip_code_prefix | ZIP prefix |
| seller_city | Seller city |
| seller_state | Seller state |

**`Dim_Orders`**

| Field | Description |
|---|---|
| order_id | Unique order identifier |
| customer_id | Customer who placed the order |
| order_status | delivered / shipped / cancelled / etc. |
| order_purchase_timestamp | When the order was placed |
| order_approved_at | When payment was approved |
| order_delivered_carrier_date | When it was handed to the carrier |
| order_delivered_customer_date | When the customer actually received it |
| order_estimated_delivery_date | What was promised at checkout |

**`Dim_Order_Payments`**

| Field | Description |
|---|---|
| order_id | Links to orders + fact table |
| payment_sequential | Payment number (1st, 2nd…) |
| payment_type | credit_card, boleto, etc. |
| payment_installments | Number of installments |
| payment_value | Payment amount |

**`Dim_Order_Reviews`**

| Field | Description |
|---|---|
| review_id | Unique review identifier |
| order_id | Order being reviewed |
| review_score | 1–5 rating |
| review_comment_title / review_comment_message | Written feedback |
| review_creation_date | When posted |
| review_answer_timestamp | When the seller responded |

**`Dim_Geolocation`**

| Field | Description |
|---|---|
| geolocation_zip_code_prefix | ZIP prefix |
| geolocation_lat / geolocation_lng | Coordinates |
| geolocation_city / geolocation_state | Location |

**`Dim_Category_Translation`**

| Field | Description |
|---|---|
| product_category_name | Original category (Portuguese) |
| product_category_name_english | Translated category (English) |

## How it all connects

```
                 Dim_Customers
                      |
                 Dim_Orders
                      |
Dim_Products — Fact_Order_Items — Dim_Sellers
                      |
              Dim_Order_Payments
                      |
             Dim_Order_Reviews
                      |
              Dim_Geolocation
                      |
          Dim_Product_Category_Translation
```

`Fact_Order_Items` sits in the center; everything else radiates outward as descriptive context. Geolocation and category translation aren't joined in every query — I treat them as lookup dimensions used only when a specific question calls for them (e.g. mapping city → state → coordinates, or translating a category name).

## What this makes easy

With this shape, questions like "most profitable category," "average delivery time by state," "freight cost vs. price," or "does review score predict repeat purchases" all become a handful of joins instead of rebuilding logic from scratch each time.

## A note on scope

I didn't physically build these fact/dimension tables as separate materialized tables in the database — the analytical SQL in `07_Analytical_insights/` queries the source tables directly using this same logical structure. I still think it was worth designing and documenting properly: it's the mental model I used while writing every query afterward, and it's closer to how a real analytics team would plan the transformation layer before building it in dbt or a warehouse.
