## Data Sources

Both datasets used here are public, hosted on Kaggle, and released by Olist:

**1. Olist Brazilian E-Commerce dataset**
https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce

This is the transactional side — customers, orders, order items, sellers, products, payments, and reviews. It's split across multiple CSVs that need to be joined together (order → order items → products/sellers, orders → payments, orders → reviews), which is part of why a proper schema was worth building rather than working off raw joins every time.

**2. Olist Marketing Funnel dataset**
https://www.kaggle.com/datasets/olistbr/marketing-funnel-olist

This is the CRM/acquisition side — marketing-qualified leads (MQLs) and the subset of those leads that turned into closed deals (i.e., sellers who actually joined the platform). It's a much smaller, event-style dataset compared to the e-commerce one.

**Why I treated them as separate sources**

These two datasets come from different systems in real life — one from an e-commerce backend, the other from a CRM/sales pipeline. I kept that separation intentional throughout ingestion and modeling instead of merging everything into a single schema. The two domains only connect where it makes business sense: through `seller_id`, once a lead has actually converted into a seller with real transactions.
