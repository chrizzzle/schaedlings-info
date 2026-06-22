# schädlings-info.de

Automated German pest-control affiliate blog. A make.com scenario runs every Monday, Wednesday, and Friday at 08:00 CET to generate and publish an AI-written article with Amazon affiliate product links — no manual intervention required.

## How it works

```
make.com (Mon/Wed/Fri 08:00 CET)
    │
    ├─→ Google Sheet (read: published topics + manual override)
    │
    ├─→ GPT-4o (topic selection → full article as JSON)
    │
    ├─→ Amazon PA API (search products → 3 affiliate links)
    │
    └─→ WordPress REST API (publish post)
         │
         └─→ Google Sheet (log: date + topic + post ID)
```

Each article is 800–1200 words of HTML, structured with H2/H3 subheadings, covering pest behavior, causes, risks, removal steps, and prevention. GPT-4o picks a seasonally relevant long-tail topic (e.g. "Ameisen in der Küche loswerden") that hasn't been covered yet.

## Repository contents

```
prompts/
  topic-generation.txt      GPT-4o prompt for seasonal topic selection
  article-generation.txt    GPT-4o prompt for full article (JSON output)
templates/
  affiliate-block.html      HTML template for 3-product Amazon affiliate section
docs/
  superpowers/
    specs/                  Design spec
    plans/                  Implementation plan with step-by-step setup tasks
```

## Tech stack

| Component | Tool |
|-----------|------|
| CMS | WordPress (REST API + Yoast SEO) |
| Automation | make.com |
| Content generation | OpenAI GPT-4o |
| Affiliate products | Amazon Product Advertising API 5.0 (eu-west-1) |
| Topic control & logging | Google Sheets |

## Google Sheet structure

**Topics tab** — append-only log of published articles:

| Datum | Thema | WordPress Post ID |
|-------|-------|-------------------|

**Control tab** — manual override:

| Nächstes Thema (leer = automatisch) |
|--------------------------------------|
| *(type a topic here to force next run, leave empty for automatic)* |

## GPT-4o prompts

Both prompts are in `prompts/`. They use `{{PLACEHOLDER}}` syntax for make.com variable substitution.

- **topic-generation.txt** — takes `{{DATUM}}`, `{{MONAT}}`, `{{BEREITS_BEHANDELT}}`; returns a single topic string
- **article-generation.txt** — takes `{{THEMA}}`; returns a JSON object with `titel`, `meta_titel`, `meta_beschreibung`, `slug`, `artikel_body`, `amazon_suchbegriff`

## Affiliate block

`templates/affiliate-block.html` is the HTML structure for 3 Amazon product cards. In the make.com scenario, product data from the PA API is mapped into this structure dynamically. The affiliate disclosure text is included and satisfies both German law (§ 5a UWG) and Amazon Associates TOS.

**Bootstrapping fallback:** if PA API access is not yet active (requires 3 qualifying sales within 180 days to maintain), the scenario publishes without the affiliate block. Links can be added manually via Amazon SiteStripe.

## Setup

Full step-by-step setup is in `docs/superpowers/plans/2026-06-21-schaedlings-experte.md`. Summary:

1. Install WordPress on hoster, set permalink structure to `/%postname%/`
2. Install GeneratePress theme and Yoast SEO plugin
3. Create a WordPress Application Password for make.com
4. Create the Google Sheet with `Topics` and `Control` tabs
5. Apply for Amazon Associates + PA API access at partnernet.amazon.de
6. Add OpenAI and Google Sheets connections in make.com
7. Build the make.com scenario following the implementation plan
8. Run the scenario once manually to verify end-to-end
9. Toggle the scenario ON — it runs automatically from there

## Legal

- **Impressum & Datenschutz:** required pages generated via e-recht24.de
- **Affiliate disclosure:** auto-inserted into every article above the product block
- **Domain:** schaedlings-info.de
