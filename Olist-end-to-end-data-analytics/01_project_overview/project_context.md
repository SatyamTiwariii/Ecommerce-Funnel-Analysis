## Why this project

Olist is a Brazilian marketplace that connects independent sellers with customers — think of it as the infrastructure layer that lets small sellers list and sell products online without building their own storefront.

I picked this dataset because it comes in two pieces that are normally analyzed separately: the transactional e-commerce side (orders, payments, deliveries, reviews) and a marketing/CRM side (how sellers are recruited onto the platform in the first place). Most portfolio projects only touch one of these. I wanted to connect them — to see whether *how* a seller was acquired says anything about *how they perform* once they're on the platform.

Concretely, I set out to answer:

- Does the source/quality of a lead predict whether it converts into an active seller?
- Once a seller is onboarded, how do they actually perform — orders, revenue, delivery speed, reviews?
- Is there a measurable link between acquisition quality and downstream operational outcomes (e.g. do sellers brought in through certain channels ship faster or slower)?

Everything else in this repo — the schema design, the validation, the SQL, the dashboards — exists to answer those three questions properly rather than just to demonstrate that I can write a query.
