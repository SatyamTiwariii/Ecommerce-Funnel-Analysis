## Loading the marketing funnel data

The marketing funnel CSVs came out messier than the e-commerce ones — which makes sense, since they're CRM exports rather than a clean transactional system. A few of the issues I ran into:

- Boolean-looking fields weren't consistently `true`/`false` — sometimes text, sometimes flags
- Numbers (like declared revenue) occasionally showed up as text
- Some fields were really categorical ranges rather than clean numeric values

Loading this straight into typed analytical tables risked silently dropping or mangling rows, so I staged it instead:

1. **Load raw into staging tables** with everything as text — nothing gets rejected at this stage, so I can see exactly what's actually in the source data.
2. **Check the columns line up** — compare staging table contents against what the analytical schema expects.
3. **Cast and normalize into the analytical tables**, applying type conversions and cleanup only once I know what I'm dealing with.

It's a bit more work upfront than loading directly, but it meant I never had to guess why a row silently disappeared — I could always go back to the staging table and see the original value.
