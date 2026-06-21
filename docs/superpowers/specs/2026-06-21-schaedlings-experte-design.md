# schädlings-info.de — Design Spec

**Date:** 2026-06-21
**Status:** Approved

## Overview

A German affiliate blog about pest control. Every few days, an AI-generated article is automatically published covering a specific pest scenario (behavior, risks, removal tips, prevention). Each article includes Amazon affiliate product links. The entire pipeline runs without manual intervention via make.com, with a Google Sheet as a lightweight control and logging layer.

**Goal:** Side income project. Initial success = covering hosting costs (~€5–10/month). Target timeline: first meaningful organic traffic within 6–12 months.

---

## Architecture

Four components, orchestrated by make.com:

```
make.com (scheduled trigger, every few days)
    │
    ├─→ Google Sheet (read: published topics log + manual override)
    │
    ├─→ OpenAI GPT-4o (generate topic + full article + SEO metadata)
    │
    ├─→ Amazon PA API (search products by keyword → affiliate links)
    │
    └─→ WordPress REST API (publish post)
         │
         └─→ Google Sheet (write: log topic + date + post ID)
```

---

## Components

### 1. WordPress

**Hosting:** User's existing hoster (WordPress not yet installed — prerequisite step).

**Theme:** GeneratePress (free tier) or Astra — lightweight, SEO-friendly, no page builder needed.

**Plugins:**
- Yoast SEO — for meta title + meta description fields (writable via REST API)
- No additional plugins required; WP Application Passwords (built-in since WP 5.6) handles authentication

**Permalink structure:** `/%postname%/` (e.g. `/silberfischchen-im-bad-loswerden/`)

**Static pages (created once manually):**
- Über uns — brief explanation of the blog's purpose
- Impressum — legally required (generate via e-recht24.de)
- Datenschutz — legally required (generate via e-recht24.de)

---

### 2. Google Sheet

Two tabs:

**Tab: `Topics`**

| Datum | Schädling/Thema | WordPress Post ID |
|-------|-----------------|-------------------|
| 2026-06-23 | Silberfischchen im Badezimmer | 42 |

- Read at the start of each make.com run to build the "already covered" list
- Appended to after each successful publish

**Tab: `Control`**

| Nächstes Thema (leer = automatisch) |
|--------------------------------------|
| |

- If a value is present, make.com uses it as this run's topic and clears the cell afterward
- Empty = GPT-4o chooses automatically

---

### 3. make.com Scenario

One scenario, triggered on a schedule (every 2–3 days, Monday/Wednesday/Friday recommended).

**Step 1 — Read Google Sheet**
- Fetch all rows from `Topics` tab → build list of covered topics
- Check `Control` tab for manual override

**Step 2 — Generate Topic (GPT-4o)**
Skip if manual override present. Otherwise, prompt:

> *"Du bist Redakteur eines deutschen Schädlingsbekämpfungs-Blogs. Heute ist [Datum], es ist [Monat]. Wähle ein sehr spezifisches Thema (z.B. 'Silberfischchen im Badezimmer', nicht nur 'Silberfischchen'), das saisonal relevant ist und noch nicht behandelt wurde. Bereits behandelte Themen: [liste]. Antworte nur mit dem Thema, ohne Erklärung."*

**Step 3 — Generate Article (GPT-4o)**
Prompt produces a structured JSON response with these fields:
- `titel` — H1, SEO keyword-focused, natural German
- `meta_titel` — max 60 characters
- `meta_beschreibung` — max 155 characters
- `slug` — URL-friendly (e.g. `silberfischchen-im-badezimmer-loswerden`)
- `artikel_body` — 800–1200 words, HTML formatted (H2/H3 subheadings, bullet lists where appropriate). Covers: pest behavior, why it appears, risks, step-by-step removal, prevention tips.
- `amazon_suchbegriff` — 1–2 keywords for PA API search (e.g. "Silberfischchen Falle")

**Step 4 — Amazon PA API**
Call `SearchItems` with:
- `Keywords`: value of `amazon_suchbegriff`
- `SearchIndex`: "All"
- `Resources`: ItemInfo.Title, Offers.Listings.Price, Images.Primary.Medium, DetailPageURL

Fetch top 3 results. Format as HTML:

```html
<div class="affiliate-hinweis">
  <em>Dieser Artikel enthält Affiliate-Links. Als Amazon-Partner verdiene ich an qualifizierten Käufen.</em>
</div>
<div class="affiliate-produkte">
  <h3>Empfohlene Produkte</h3>
  <!-- 3× product card: image, name, price, "Jetzt bei Amazon kaufen" link -->
</div>
```

**Early bootstrapping fallback:** If PA API access is not yet active, skip Step 4. The affiliate block is added manually via Amazon Associates SiteStripe after publishing. make.com scenario continues to publish the article without the affiliate block.

**Step 5 — Publish to WordPress**
POST to `/wp/v2/posts` with:
- `title`: `titel`
- `content`: `artikel_body` + affiliate HTML block
- `slug`: `slug`
- `status`: `publish`
- `meta._yoast_wpseo_title`: `meta_titel`
- `meta._yoast_wpseo_metadesc`: `meta_beschreibung`

Note: Yoast SEO meta fields require the Yoast REST API to expose them as writable. The implementation plan should verify the exact field names and whether a Yoast add-on is needed.

Authentication via WordPress Application Password (stored as make.com environment variable).

**Step 6 — Log to Google Sheet**
Append row to `Topics` tab: topic name + publish date + WordPress post ID from Step 5 response.

---

### 4. Content Strategy

**Topic approach:** Long-tail specificity is both the SEO strategy and the content strategy.

- Not "Ameisen bekämpfen" — but "Ameisen in der Küche loswerden", "Ameisen auf dem Balkon", "Ameisen im Schlafzimmer"
- Seasonal awareness baked into GPT-4o prompt (Mücken/Wespen in summer, Mäuse/Ratten in winter, Silberfischchen/Schaben year-round)
- Topic space is large: 20+ common pests × multiple rooms/contexts/severities = 200+ unique articles before repetition

**Publishing frequency:** Every 2–3 days (approx. 2–3 articles/week)

**SEO basics per article:**
- Keyword-focused H1 title
- Meta title (≤60 chars) and meta description (≤155 chars) generated by GPT-4o
- Clean permalink slug
- Structured HTML with H2/H3 subheadings for featured snippet eligibility

---

## Legal Requirements (Germany)

- **Impressum:** Required by Telemediengesetz — generate via e-recht24.de
- **Datenschutz:** Required by DSGVO — generate via e-recht24.de
- **Affiliate disclosure:** Auto-inserted into every article above the affiliate block (see Step 4 template above)
- **Amazon TOS:** Affiliate disclosure also satisfies Amazon's requirement for link labeling

---

## Prerequisites (in order)

1. Install WordPress on existing hoster
2. Configure permalink structure (`/%postname%/`)
3. Install Yoast SEO plugin
4. Create Application Password in WordPress for make.com
5. Create static pages: Über uns, Impressum, Datenschutz
6. Create Google Sheet with `Topics` and `Control` tabs
7. Connect Google Sheet to make.com (OAuth)
8. Verify Amazon Associates account is active at partnernet.amazon.de
9. Apply for / verify PA API access (Access Key, Secret Key, Partner Tag)
10. Connect OpenAI GPT-4o to make.com (API key)
11. Build and test make.com scenario with a single dry-run article

---

## Revenue Model & Expectations

- **Monetization:** Amazon Associates (partnernet.amazon.de), ~3–5% commission per sale
- **Initial goal:** Cover hosting costs (~€5–10/month)
- **Realistic timeline:** 6–12 months to first consistent organic traffic
- **Risk:** Amazon PA API requires 3 qualifying sales within 180 days to maintain account — manual affiliate links via SiteStripe used during bootstrapping phase
- **Long-term upside:** Hundreds of long-tail articles compounding over time

---

## Out of Scope

- Newsletter / email capture
- Comment section
- Social media automation
- Paid traffic / ads
- Multiple languages
- Custom affiliate product curation (beyond PA API search)
