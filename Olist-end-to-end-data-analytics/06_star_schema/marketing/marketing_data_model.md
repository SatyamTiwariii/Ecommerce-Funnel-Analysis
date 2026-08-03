# Marketing Funnel Data Model

## Why this one isn't a star schema

I initially assumed I'd build a star schema here too, to match the e-commerce side. Once I actually looked at the data, that stopped making sense.

The marketing dataset is just two tables, and each row in them is already a discrete business event (a lead came in, a deal closed) — there's no repeating transactional grain to build a fact table around the way `order_items` gives you one on the e-commerce side. Metrics here come from the *relationship* between two event tables, not from joining a fact table out to a bunch of dimensions.

So instead of forcing a dimensional model onto data that doesn't have that shape, I modeled it as a simple funnel: leads flow in at the top, a subset convert into closed deals at the bottom.

## The two tables

| Table | What it captures |
|---|---|
| `marketing_qualified_leads` | Every marketing-qualified lead — top of the funnel |
| `marketing_closed_deals` | The leads that actually converted into onboarded sellers — bottom of the funnel |

They're linked by `mql_id`.

```
     marketing_qualified_leads
                |
            (mql_id)
                |
      marketing_closed_deals
```

It's a one-to-many relationship in the loose sense that not every MQL closes — each MQL converts into zero or one closed deal, never more. That's really the entire model: everything else (drop-off rate, conversion rate, lead-source quality) is derived by joining these two tables and counting.

## Table definitions

**`marketing_qualified_leads`** — one row per marketing-qualified lead

| Field | Description |
|---|---|
| mql_id | Unique identifier for the lead |
| first_contact_date | When the lead first showed up |
| landing_page_id | Which landing page brought them in |
| origin | Acquisition channel (organic, paid, etc.) |

**`marketing_closed_deals`** — one row per successfully acquired seller

| Field | Description |
|---|---|
| mql_id | Links back to the originating lead |
| seller_id | The seller that resulted from this lead |
| business_type | manufacturer, reseller, etc. |
| lead_type | Lead classification |
| has_company | Whether the seller has a registered company |
| declared_monthly_revenue | Self-declared revenue at signup |
| declared_product_catalog_size | Approx. number of products they planned to list |
| won_date | When the deal closed |

## What this model is built to answer

- How many leads actually convert into sellers?
- Which lead origins convert best?
- Which sources bring in the highest-value sellers?
- Does business type correlate with seller quality?
- What does the revenue profile of acquired sellers actually look like?

## Why I'm comfortable with this decision

It would have been easy to force a star schema here just to keep the project "consistent," but that would have added dimensional overhead the data doesn't need and made the funnel logic harder to follow, not easier. Recognizing when *not* to build a star schema felt like a more useful thing to demonstrate than mechanically applying the same pattern everywhere.

This model feeds directly into `07_Analytical_insights/marketing/` for the SQL analysis and into the Power BI funnel dashboard.
