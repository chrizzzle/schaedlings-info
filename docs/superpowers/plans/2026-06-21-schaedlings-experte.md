# schädlings-experte.de Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a fully automated German pest-control affiliate blog that publishes AI-generated articles every 2–3 days, with Amazon affiliate product links, using WordPress + make.com + GPT-4o + Amazon PA API + Google Sheets.

**Architecture:** make.com orchestrates a single scenario that reads a Google Sheet for topic history and manual override, calls GPT-4o twice (topic selection + article generation), calls the Amazon PA API for 3 product links, publishes to WordPress via REST API, and logs the result back to the sheet. Prompts and HTML templates are version-controlled in this repo.

**Tech Stack:** WordPress (REST API + Yoast SEO), make.com, OpenAI GPT-4o, Amazon Product Advertising API 5.0, Google Sheets

## Global Constraints

- Language: all blog content in German
- Publishing frequency: Monday / Wednesday / Friday (08:00 CET)
- Article length: 800–1200 words per article
- Meta title: ≤60 characters
- Meta description: ≤155 characters
- Affiliate disclosure: required in every article (German law + Amazon TOS)
- PA API region: `eu-west-1`, endpoint: `https://webservices.amazon.de/paapi5/searchitems`
- Topic approach: long-tail specificity (e.g. "Ameisen in der Küche" not "Ameisen")
- Bootstrapping fallback: if PA API unavailable, publish without affiliate block and add manually via SiteStripe

---

## File Map

```
prompts/
  topic-generation.txt       — GPT-4o prompt for seasonal topic selection
  article-generation.txt     — GPT-4o prompt for full article (JSON output)
templates/
  affiliate-block.html       — HTML template for affiliate product section
docs/
  superpowers/
    specs/2026-06-21-schaedlings-experte-design.md   (existing)
    plans/2026-06-21-schaedlings-experte.md          (this file)
```

---

### Task 1: WordPress Installation & Core Configuration

**What this produces:** A working WordPress site with REST API enabled, Yoast SEO installed, and make.com able to authenticate.

**Files:** None (configuration in hosting control panel + WordPress admin)

- [ ] **Step 1: Install WordPress**

  Log into your hoster's control panel. Find the one-click WordPress installer (usually under "CMS" or "WordPress"). Install to the root of `schaedlings-experte.de`. Note the admin username and password.

- [ ] **Step 2: Install theme**

  In WordPress Admin → Appearance → Themes → Add New, search for **GeneratePress**. Install and Activate.

- [ ] **Step 3: Set permalink structure**

  WordPress Admin → Settings → Permalinks → select **Post name** (`/%postname%/`) → Save Changes.

  Verify: the setting shows `https://schaedlings-experte.de/sample-post/` in the preview.

- [ ] **Step 4: Install Yoast SEO**

  WordPress Admin → Plugins → Add New → search **Yoast SEO** → Install → Activate.

  After activation: complete the Yoast setup wizard (set site type to "Blog", skip the social steps).

- [ ] **Step 5: Enable Yoast REST API fields**

  Yoast SEO exposes meta fields to the REST API by default from version 14+. Verify this works later in Task 9. No extra plugin needed for now.

- [ ] **Step 6: Create Application Password for make.com**

  WordPress Admin → Users → Profile → scroll to **Application Passwords** section → enter name `make.com` → click **Add New Application Password** → copy the generated password immediately (shown only once).

  Store it somewhere safe. Format for make.com: `username:application-password` (spaces in the password are fine — WordPress displays them for readability but they're valid in Basic Auth).

- [ ] **Step 7: Verify REST API is accessible**

  Open a browser and navigate to:
  ```
  https://schaedlings-experte.de/wp-json/wp/v2/posts
  ```
  Expected: a JSON response (empty array `[]` or list of posts). If you see a 404, the permalink structure wasn't saved — redo Step 3.

- [ ] **Step 8: Commit checkpoint note**

  ```bash
  echo "WordPress installed and configured. REST API verified. Credentials stored securely." > docs/setup-log.md
  git add docs/setup-log.md
  git commit -m "chore: WordPress setup complete, REST API verified"
  ```

---

### Task 2: Static Pages & Legal Content

**What this produces:** Impressum, Datenschutz, and Über uns pages live on the site with a working navigation menu.

- [ ] **Step 1: Generate Impressum**

  Go to [e-recht24.de/muster/impressum](https://www.e-recht24.de/muster/impressum.html). Fill in your details (name, address, email — required by Telemediengesetz §5). Copy the generated HTML.

  WordPress Admin → Pages → Add New:
  - Title: `Impressum`
  - Paste the content (switch to HTML/Code editor mode to paste raw HTML)
  - Publish

- [ ] **Step 2: Generate Datenschutzerklärung**

  Go to [e-recht24.de/muster/datenschutzerklaerung](https://www.e-recht24.de/muster/datenschutzerklaerung.html). Select: no cookies (for now), no contact form, no newsletter. Copy the generated HTML.

  WordPress Admin → Pages → Add New:
  - Title: `Datenschutz`
  - Paste the content in HTML editor mode
  - Publish

- [ ] **Step 3: Create Über uns page**

  WordPress Admin → Pages → Add New:
  - Title: `Über uns`
  - Content (paste this text, adjust name if desired):

  ```
  Willkommen bei Schädlings-Experte.de — Ihrem Ratgeber rund um Schädlinge im Haus und Garten.

  Wir helfen Ihnen dabei, ungebetene Gäste schnell und effektiv loszuwerden. 
  Unsere Artikel erklären das Verhalten der häufigsten Schädlinge und zeigen Ihnen 
  bewährte Methoden zur Bekämpfung und Vorbeugung — von Ameisen in der Küche bis 
  zu Mäusen im Keller.

  Alle Empfehlungen sind unabhängig recherchiert. Einige Artikel enthalten 
  Affiliate-Links zu Amazon, über die wir eine kleine Provision erhalten — 
  für Sie entstehen dabei keine Mehrkosten.
  ```

  - Publish

- [ ] **Step 4: Create navigation menu**

  WordPress Admin → Appearance → Menus → Create New Menu named `Hauptmenü`.

  Add pages: Über uns, Impressum, Datenschutz.

  Under "Menu Settings" → check **Primary Menu** → Save Menu.

- [ ] **Step 5: Verify pages**

  Visit each URL in browser:
  - `https://schaedlings-experte.de/ueber-uns/`
  - `https://schaedlings-experte.de/impressum/`
  - `https://schaedlings-experte.de/datenschutz/`

  All three should load with the correct content and show the navigation menu.

- [ ] **Step 6: Commit**

  ```bash
  git add docs/setup-log.md
  git commit -m "chore: static pages and legal content live"
  ```

---

### Task 3: Google Sheet Control Layer

**What this produces:** A Google Sheet with `Topics` and `Control` tabs, connected to make.com.

- [ ] **Step 1: Create Google Sheet**

  Go to [sheets.google.com](https://sheets.google.com) → Blank spreadsheet.

  Name it: `schädlings-experte`

- [ ] **Step 2: Set up Topics tab**

  Rename `Sheet1` to `Topics`.

  Set row 1 headers exactly as:
  - A1: `Datum`
  - B1: `Thema`
  - C1: `WordPress Post ID`

  Format column A as Date (Format → Number → Date).

- [ ] **Step 3: Set up Control tab**

  Click `+` to add a new sheet. Name it `Control`.

  Set:
  - A1: `Nächstes Thema (leer = automatisch)`
  - A2: *(leave empty)*

  A2 is the cell make.com reads and writes. When you want to manually set next week's topic, type it in A2.

- [ ] **Step 4: Connect Google Sheet to make.com**

  In make.com → Connections → Add a connection → Google Sheets → Sign in with Google → authorize.

  Verify: create a test scenario with just a Google Sheets "Get a Cell" module, pointing to the `Control` sheet, cell A2. Run it. Expected output: empty string (since A2 is empty). This confirms OAuth and sheet access work.

  Delete the test scenario after verification.

- [ ] **Step 5: Commit**

  ```bash
  git commit -m "chore: Google Sheet created and connected to make.com" --allow-empty
  ```

---

### Task 4: Craft & Version-Control GPT-4o Prompts

**What this produces:** Two prompts saved to the repo (`prompts/`) and manually validated in the OpenAI Playground. These are the core content engine.

- [ ] **Step 1: Write topic-generation prompt**

  Create `prompts/topic-generation.txt`:

  ```
  Du bist Redakteur eines deutschen Schädlingsbekämpfungs-Blogs namens "Schädlings-Experte.de".

  Heute ist {{DATUM}}. Es ist {{MONAT}}.

  Deine Aufgabe: Wähle ein sehr spezifisches Blog-Thema, das:
  1. Saisonal relevant für {{MONAT}} in Deutschland ist
  2. Noch NICHT in dieser Liste behandelt wurde: {{BEREITS_BEHANDELT}}
  3. Einen konkreten Kontext hat (z.B. "Ameisen in der Küche loswerden" statt nur "Ameisen")

  Beispiele für gute spezifische Themen:
  - "Silberfischchen im Badezimmer loswerden"
  - "Mücken im Schlafzimmer bekämpfen"
  - "Ameisen auf dem Balkon vertreiben"
  - "Mäuse im Keller loswerden"
  - "Wespen unter dem Dach entfernen"

  Antworte NUR mit dem Thema. Keine Erklärung, keine Aufzählung, keine Anführungszeichen.
  ```

- [ ] **Step 2: Write article-generation prompt**

  Create `prompts/article-generation.txt`:

  ```
  Du bist ein erfahrener Autor für den deutschen Blog "Schädlings-Experte.de".

  Schreibe einen vollständigen Blogartikel zum Thema: "{{THEMA}}"

  Antworte ausschließlich mit validem JSON in diesem Format (kein Text davor oder danach):

  {
    "titel": "...",
    "meta_titel": "...",
    "meta_beschreibung": "...",
    "slug": "...",
    "artikel_body": "...",
    "amazon_suchbegriff": "..."
  }

  Regeln für jedes Feld:

  "titel": Aussagekräftiger H1-Titel. Enthält das Haupt-Keyword. Natürliches Deutsch. Beispiel: "Silberfischchen im Badezimmer loswerden: So bekämpfen Sie den Schädling effektiv"

  "meta_titel": SEO-Meta-Titel, maximal 60 Zeichen. Keyword am Anfang. Beispiel: "Silberfischchen im Bad loswerden | Schädlings-Experte"

  "meta_beschreibung": Meta-Beschreibung, maximal 155 Zeichen. Handlungsaufforderung enthalten. Beispiel: "Silberfischchen im Badezimmer? Erfahren Sie, warum sie kommen und wie Sie sie dauerhaft loswerden. Tipps & Mittel im Überblick."

  "slug": URL-freundlich, nur Kleinbuchstaben, Bindestriche statt Leerzeichen, keine Umlaute. Beispiel: "silberfischchen-badezimmer-loswerden"

  "artikel_body": Vollständiger Artikel als HTML. Anforderungen:
  - 800 bis 1200 Wörter
  - Beginne direkt mit einem einleitenden Absatz (kein H1, der kommt aus dem "titel"-Feld)
  - Verwende H2 und H3 für Abschnitte
  - Verwende <ul> oder <ol> für Listen
  - Verwende <strong> für Hervorhebungen
  - Pflichtabschnitte: (1) Verhalten und Lebensweise des Schädlings, (2) Warum er auftaucht, (3) Risiken und Schäden, (4) Bekämpfung Schritt für Schritt, (5) Vorbeugung
  - Ton: sachlich, hilfreich, nicht reißerisch

  "amazon_suchbegriff": 1–3 deutsche Suchwörter für Amazon-Produktsuche, passend zum Artikel. Beispiel: "Silberfischchen Falle" oder "Silberfischchen Mittel Spray"
  ```

- [ ] **Step 3: Write affiliate HTML template**

  Create `templates/affiliate-block.html`:

  ```html
  <div class="affiliate-hinweis" style="background:#f8f8f8;border-left:4px solid #e8a000;padding:12px 16px;margin:24px 0;font-size:0.9em;">
    <em>Dieser Artikel enthält Affiliate-Links. Als Amazon-Partner verdiene ich an qualifizierten Käufen – für Sie entstehen keine Mehrkosten.</em>
  </div>

  <div class="affiliate-produkte">
    <h3>Empfohlene Produkte</h3>

    <div class="produkt-karte" style="display:flex;gap:16px;margin-bottom:20px;padding:16px;border:1px solid #eee;border-radius:6px;">
      <img src="{{PRODUKT_1_BILD}}" alt="{{PRODUKT_1_NAME}}" style="width:120px;height:120px;object-fit:contain;flex-shrink:0;">
      <div>
        <strong>{{PRODUKT_1_NAME}}</strong><br>
        <span style="color:#B12704;font-weight:bold;">{{PRODUKT_1_PREIS}}</span><br>
        <a href="{{PRODUKT_1_URL}}" target="_blank" rel="nofollow sponsored" style="display:inline-block;margin-top:8px;background:#FF9900;color:#000;padding:8px 16px;text-decoration:none;border-radius:4px;font-weight:bold;">Jetzt bei Amazon kaufen</a>
      </div>
    </div>

    <div class="produkt-karte" style="display:flex;gap:16px;margin-bottom:20px;padding:16px;border:1px solid #eee;border-radius:6px;">
      <img src="{{PRODUKT_2_BILD}}" alt="{{PRODUKT_2_NAME}}" style="width:120px;height:120px;object-fit:contain;flex-shrink:0;">
      <div>
        <strong>{{PRODUKT_2_NAME}}</strong><br>
        <span style="color:#B12704;font-weight:bold;">{{PRODUKT_2_PREIS}}</span><br>
        <a href="{{PRODUKT_2_URL}}" target="_blank" rel="nofollow sponsored" style="display:inline-block;margin-top:8px;background:#FF9900;color:#000;padding:8px 16px;text-decoration:none;border-radius:4px;font-weight:bold;">Jetzt bei Amazon kaufen</a>
      </div>
    </div>

    <div class="produkt-karte" style="display:flex;gap:16px;margin-bottom:20px;padding:16px;border:1px solid #eee;border-radius:6px;">
      <img src="{{PRODUKT_3_BILD}}" alt="{{PRODUKT_3_NAME}}" style="width:120px;height:120px;object-fit:contain;flex-shrink:0;">
      <div>
        <strong>{{PRODUKT_3_NAME}}</strong><br>
        <span style="color:#B12704;font-weight:bold;">{{PRODUKT_3_PREIS}}</span><br>
        <a href="{{PRODUKT_3_URL}}" target="_blank" rel="nofollow sponsored" style="display:inline-block;margin-top:8px;background:#FF9900;color:#000;padding:8px 16px;text-decoration:none;border-radius:4px;font-weight:bold;">Jetzt bei Amazon kaufen</a>
      </div>
    </div>
  </div>
  ```

- [ ] **Step 4: Validate prompts manually in OpenAI Playground**

  Go to [platform.openai.com/playground](https://platform.openai.com/playground). Select model: `gpt-4o`.

  **Test topic prompt:** Paste `prompts/topic-generation.txt`, replace:
  - `{{DATUM}}` → `21. Juni 2026`
  - `{{MONAT}}` → `Juni`
  - `{{BEREITS_BEHANDELT}}` → *(leave empty or "keine")*

  Expected: a single specific German pest topic like "Wespen auf dem Balkon vertreiben". If the response includes extra text or quotes, tighten the prompt.

  **Test article prompt:** Paste `prompts/article-generation.txt`, replace `{{THEMA}}` with the topic from above.

  Expected: valid JSON with all 6 fields. Paste the JSON into [jsonlint.com](https://jsonlint.com) to verify it's valid. Check that `artikel_body` contains H2 headings and is 800–1200 words. Check `meta_titel` is ≤60 chars and `meta_beschreibung` is ≤155 chars.

  If the JSON is invalid or fields are missing, add to the prompt: `"Wichtig: Antworte nur mit dem JSON-Objekt. Kein Markdown-Codeblock, keine Erklärung, kein Text außerhalb des JSON."`

- [ ] **Step 5: Commit prompts and template**

  ```bash
  git add prompts/ templates/
  git commit -m "feat: add GPT-4o prompts and affiliate HTML template"
  ```

---

### Task 5: make.com Scenario — Skeleton, Trigger & Sheet Reads

**What this produces:** A make.com scenario that triggers on schedule and reads both sheet tabs. No article generation yet.

- [ ] **Step 1: Create scenario**

  make.com → Scenarios → Create a new scenario.

  Name it: `schädlings-experte — Wöchentlicher Artikel`

- [ ] **Step 2: Add Schedule trigger**

  Add module → **Schedule** (the built-in one, not a specific app).

  Settings:
  - Run scenario: **At regular intervals**
  - Interval: **Days**
  - Every: **1 day** (you'll restrict to Mon/Wed/Fri using a filter in Step 5)
  - Start time: **08:00**
  - Timezone: **Europe/Berlin**

- [ ] **Step 3: Add Google Sheets — Read Topics**

  Add module → Google Sheets → **Search Rows**.

  Settings:
  - Connection: your Google account
  - Spreadsheet: `schädlings-experte`
  - Sheet: `Topics`
  - Filter: *(leave empty — fetch all rows)*
  - Column range: A through C

  This returns all rows. make.com will aggregate them to build the "already covered" list.

- [ ] **Step 4: Add Text Aggregator — build topics list**

  Add module → **Text Aggregator**.

  Settings:
  - Source module: the Google Sheets "Search Rows" module above
  - Text: `{{B}}` (maps to column B — the Thema column)
  - Row separator: `, `

  This produces a single comma-separated string of all covered topics.

- [ ] **Step 5: Add Google Sheets — Read Control cell**

  Add module → Google Sheets → **Get a Cell**.

  Settings:
  - Spreadsheet: `schädlings-experte`
  - Sheet: `Control`
  - Cell: `A2`

  This reads the manual override value.

- [ ] **Step 6: Add day-of-week filter**

  Add a filter between the Schedule trigger and the first Sheets module:

  - Label: `Nur Mo/Mi/Fr`
  - Condition: `{{formatDate(now; "E")}}` **Equal to (text, case insensitive)** `Mon`
  - Add OR condition: `{{formatDate(now; "E")}}` **Equal to** `Wed`
  - Add OR condition: `{{formatDate(now; "E")}}` **Equal to** `Fri`

  This prevents the scenario from generating articles on Tuesday/Thursday/weekend even though it's triggered daily.

- [ ] **Step 7: Verify skeleton runs**

  Click **Run once**. The scenario should execute, fetch the (empty) Topics sheet and the (empty) Control cell, and stop without errors. Check the execution log — all modules should show green.

---

### Task 6: make.com Scenario — Topic Generation

**What this produces:** The scenario branches on manual override and produces a confirmed topic string by end of this task.

- [ ] **Step 1: Add Router**

  After the "Get a Cell" module, add a **Router** module. This creates two paths:

  - **Path 1 (Manual):** Control cell A2 is not empty
  - **Path 2 (Automatic):** Control cell A2 is empty → GPT-4o picks the topic

- [ ] **Step 2: Configure Path 1 filter (Manual override)**

  On Path 1's connection, add a filter:
  - Label: `Manuelles Thema vorhanden`
  - Condition: `{{value}}` (the cell value from Get a Cell) **Does not equal** *(empty string)*

- [ ] **Step 3: Configure Path 2 filter (Automatic)**

  On Path 2's connection, add a filter:
  - Label: `Automatische Themenwahl`
  - Condition: `{{value}}` **Equal to** *(empty string)*

- [ ] **Step 4: Add OpenAI module on Path 2**

  Path 2 → Add module → OpenAI (GPT) → **Create a Completion**.

  Settings:
  - Connection: your OpenAI API key
  - Model: `gpt-4o`
  - Messages: System role
  - System message content: paste the full contents of `prompts/topic-generation.txt` with variables replaced by make.com dynamic values:
    - `{{DATUM}}` → `{{formatDate(now; "D. MMMM YYYY"; "de")}}`
    - `{{MONAT}}` → `{{formatDate(now; "MMMM"; "de")}}`
    - `{{BEREITS_BEHANDELT}}` → the output of the Text Aggregator from Task 5 Step 4 (the comma-separated topics string). If empty, use `"keine"`.
  - Max tokens: 50
  - Temperature: 0.8

- [ ] **Step 5: Add variable consolidation after Router**

  After both paths merge (add a module after the Router that both paths connect to), add a **Set Variable** module:

  - Variable name: `thema`
  - Variable value: Use an `if` expression:
    ```
    {{if(3.value; 3.value; 6.choices[].message.content)}}
    ```
    Where `3.value` is the Control cell value (Path 1) and `6.choices[].message.content` is the GPT-4o output (Path 2). Adjust module numbers to match your scenario.

  This gives a single `{{thema}}` variable used by all subsequent modules.

- [ ] **Step 6: Test topic generation**

  Click **Run once**. Leave Control A2 empty (automatic path).

  Expected in execution log:
  - OpenAI module shows a response like `"Mücken im Schlafzimmer bekämpfen"`
  - Set Variable module shows `thema = "Mücken im Schlafzimmer bekämpfen"`

  Now test manual override: type `"Wespen auf dem Balkon"` in Control sheet A2. Run once again. Expected: OpenAI module is skipped (Path 1 taken), `thema = "Wespen auf dem Balkon"`.

---

### Task 7: make.com Scenario — Article Generation

**What this produces:** GPT-4o generates a full article as parsed JSON with all 6 fields available as make.com variables.

- [ ] **Step 1: Add OpenAI module for article**

  After the Set Variable (thema) module → Add module → OpenAI (GPT) → **Create a Completion**.

  Settings:
  - Model: `gpt-4o`
  - Messages: User role (not System — using user role produces more reliable JSON output)
  - Message content: paste full contents of `prompts/article-generation.txt`, replacing `{{THEMA}}` with `{{thema}}` (the make.com variable from Task 6)
  - Max tokens: 3000
  - Temperature: 0.7
  - Response format: if the OpenAI module offers a "JSON mode" toggle, enable it

- [ ] **Step 2: Parse JSON response**

  The OpenAI module outputs the response in `choices[].message.content`. Add a **JSON → Parse JSON** module:

  - JSON string: `{{choices[].message.content}}` from the OpenAI module

  This exposes `titel`, `meta_titel`, `meta_beschreibung`, `slug`, `artikel_body`, `amazon_suchbegriff` as individual variables.

- [ ] **Step 3: Test article generation**

  Run once. Check execution log:

  - OpenAI module: response content should start with `{` (valid JSON)
  - Parse JSON module: all 6 fields should appear as separate mapped values
  - Spot-check: `meta_titel` length ≤60 chars, `meta_beschreibung` length ≤155 chars
  - Spot-check: `artikel_body` contains `<h2>` tags

  If JSON parsing fails (GPT-4o returned Markdown code fences), add a **String → Replace** module before Parse JSON:
  - Pattern: `` ```json `` → replace with `` `` (empty)
  - Pattern: `` ``` `` → replace with `` `` (empty)

---

### Task 8: make.com Scenario — Amazon PA API Integration

**What this produces:** 3 Amazon product results fetched and formatted into the affiliate HTML block. Includes bootstrapping fallback.

- [ ] **Step 1: Set up AWS connection in make.com**

  make.com → Connections → Add a connection → **AWS**.

  Settings:
  - Access Key ID: your Amazon PA API Access Key
  - Secret Access Key: your Amazon PA API Secret Key
  - Region: `eu-west-1`

  Name the connection: `Amazon PA API`

- [ ] **Step 2: Add HTTP module for PA API call**

  Add module → HTTP → **Make a request**.

  Settings:
  - URL: `https://webservices.amazon.de/paapi5/searchitems`
  - Method: POST
  - Headers:
    - `content-type`: `application/json; charset=utf-8`
    - `x-amz-target`: `com.amazon.paapi5.v1.ProductAdvertisingAPIv1.SearchItems`
  - Body type: Raw
  - Content type: JSON (application/json)
  - Request content:
    ```json
    {
      "Keywords": "{{amazon_suchbegriff}}",
      "PartnerTag": "YOUR-PARTNER-TAG-20",
      "PartnerType": "Associates",
      "Marketplace": "www.amazon.de",
      "Resources": [
        "ItemInfo.Title",
        "Offers.Listings.Price",
        "Images.Primary.Medium",
        "DetailPageURL"
      ],
      "SearchIndex": "All",
      "ItemCount": 3
    }
    ```
    Replace `YOUR-PARTNER-TAG-20` with your actual Associates tracking ID.
  - Authentication type: **AWS**
  - AWS Connection: `Amazon PA API`
  - AWS Service: `ProductAdvertisingAPI`

- [ ] **Step 3: Add error handler for PA API fallback**

  Right-click the HTTP module → Add error handler → **Resume**.

  In the error handler path, add a **Set Variable** module:
  - Variable: `affiliate_block`
  - Value: *(empty string)*

  This handles the bootstrapping phase when PA API is unavailable — the scenario continues without the affiliate block.

- [ ] **Step 4: Map product data and build HTML**

  After the HTTP module (success path), add a **Set Variable** module to build the affiliate HTML block.

  - Variable: `affiliate_block`
  - Value (paste this make.com expression — adjust array paths if the PA API response structure differs):

  ```
  <div class="affiliate-hinweis" style="background:#f8f8f8;border-left:4px solid #e8a000;padding:12px 16px;margin:24px 0;font-size:0.9em;"><em>Dieser Artikel enthält Affiliate-Links. Als Amazon-Partner verdiene ich an qualifizierten Käufen – für Sie entstehen keine Mehrkosten.</em></div><div class="affiliate-produkte"><h3>Empfohlene Produkte</h3><div class="produkt-karte" style="display:flex;gap:16px;margin-bottom:20px;padding:16px;border:1px solid #eee;border-radius:6px;"><img src="{{data.SearchResult.Items[].Images.Primary.Medium.URL[1]}}" style="width:120px;height:120px;object-fit:contain;flex-shrink:0;"><div><strong>{{data.SearchResult.Items[].ItemInfo.Title.DisplayValue[1]}}</strong><br><span style="color:#B12704;font-weight:bold;">{{data.SearchResult.Items[].Offers.Listings[].Price.DisplayAmount[1]}}</span><br><a href="{{data.SearchResult.Items[].DetailPageURL[1]}}" target="_blank" rel="nofollow sponsored" style="display:inline-block;margin-top:8px;background:#FF9900;color:#000;padding:8px 16px;text-decoration:none;border-radius:4px;font-weight:bold;">Jetzt bei Amazon kaufen</a></div></div><div class="produkt-karte" style="display:flex;gap:16px;margin-bottom:20px;padding:16px;border:1px solid #eee;border-radius:6px;"><img src="{{data.SearchResult.Items[].Images.Primary.Medium.URL[2]}}" style="width:120px;height:120px;object-fit:contain;flex-shrink:0;"><div><strong>{{data.SearchResult.Items[].ItemInfo.Title.DisplayValue[2]}}</strong><br><span style="color:#B12704;font-weight:bold;">{{data.SearchResult.Items[].Offers.Listings[].Price.DisplayAmount[2]}}</span><br><a href="{{data.SearchResult.Items[].DetailPageURL[2]}}" target="_blank" rel="nofollow sponsored" style="display:inline-block;margin-top:8px;background:#FF9900;color:#000;padding:8px 16px;text-decoration:none;border-radius:4px;font-weight:bold;">Jetzt bei Amazon kaufen</a></div></div><div class="produkt-karte" style="display:flex;gap:16px;margin-bottom:20px;padding:16px;border:1px solid #eee;border-radius:6px;"><img src="{{data.SearchResult.Items[].Images.Primary.Medium.URL[3]}}" style="width:120px;height:120px;object-fit:contain;flex-shrink:0;"><div><strong>{{data.SearchResult.Items[].ItemInfo.Title.DisplayValue[3]}}</strong><br><span style="color:#B12704;font-weight:bold;">{{data.SearchResult.Items[].Offers.Listings[].Price.DisplayAmount[3]}}</span><br><a href="{{data.SearchResult.Items[].DetailPageURL[3]}}" target="_blank" rel="nofollow sponsored" style="display:inline-block;margin-top:8px;background:#FF9900;color:#000;padding:8px 16px;text-decoration:none;border-radius:4px;font-weight:bold;">Jetzt bei Amazon kaufen</a></div></div></div>
  ```

  Note: PA API response paths (`data.SearchResult.Items[]...`) — verify exact paths from the actual API response in the execution log and adjust if needed.

- [ ] **Step 5: Test PA API call**

  Run once (with article generation from Task 7 completing first).

  Expected in execution log:
  - HTTP module status: 200
  - Response body contains `SearchResult.Items` array with 3 items
  - Set Variable module: `affiliate_block` contains the HTML string with 3 product cards

  If status 400 or 403: check Partner Tag, Access Key/Secret Key, and that your Associates account has PA API access enabled.

---

### Task 9: make.com Scenario — WordPress Publishing

**What this produces:** The article is published to WordPress with correct meta fields set.

- [ ] **Step 1: Verify Yoast meta field names**

  Before building the module, confirm the exact field names Yoast exposes. In your browser, navigate to:
  ```
  https://schaedlings-experte.de/wp-json/wp/v2/posts?_fields=id,title,yoast_head_json
  ```

  If this works, Yoast SEO exposes data via `yoast_head_json`. However, to *write* meta fields you need the `yoast_wpseo_title` and `yoast_wpseo_metadesc` fields. Verify they're writable by checking:
  ```
  https://schaedlings-experte.de/wp-json/wp/v2/posts/schema
  ```
  Look for `yoast` in the schema response. If these fields aren't present, install the **Yoast SEO - REST API** plugin (free, by Trifenol) which exposes them explicitly.

- [ ] **Step 2: Add WordPress HTTP module**

  Add module → HTTP → **Make a request**.

  Settings:
  - URL: `https://schaedlings-experte.de/wp-json/wp/v2/posts`
  - Method: POST
  - Headers:
    - `Content-Type`: `application/json`
    - `Authorization`: `Basic {{base64("your-wp-username:your-application-password")}}`

    Replace with your actual credentials. In make.com you can store these as a **Connection** or as **Environment Variables** (Scenario → Settings → Variables). Use environment variables so credentials aren't hardcoded.

  - Body type: Raw
  - Content type: JSON
  - Request content:
    ```json
    {
      "title": "{{titel}}",
      "content": "{{artikel_body}}{{affiliate_block}}",
      "slug": "{{slug}}",
      "status": "publish",
      "meta": {
        "yoast_wpseo_title": "{{meta_titel}}",
        "yoast_wpseo_metadesc": "{{meta_beschreibung}}"
      }
    }
    ```

- [ ] **Step 3: Capture Post ID from response**

  Add a **JSON → Parse JSON** module after the HTTP module.
  - JSON string: the HTTP module's response body (`data`)

  Add a **Set Variable** module:
  - Variable: `wordpress_post_id`
  - Value: `{{id}}` (from the parsed response)

- [ ] **Step 4: Test WordPress publishing**

  Run once. Check execution log:
  - HTTP module status: 201 (Created)
  - Response body contains `"id": 123` and `"status": "publish"`

  Open WordPress Admin → Posts. The article should appear as published with the correct title.

  Open the live post URL. Check:
  - Title matches `titel` field
  - H2 subheadings are present
  - Affiliate block appears (if PA API worked) or is absent (bootstrapping fallback)

  Check in browser → View Source or Yoast SEO panel in WP Admin:
  - `<title>` tag matches `meta_titel`
  - `<meta name="description">` matches `meta_beschreibung`

  If Yoast meta fields didn't write: install **Yoast SEO - REST API** plugin and re-test.

---

### Task 10: Logging, End-to-End Test & Go Live

**What this produces:** Complete working scenario that logs results and runs automatically on schedule.

- [ ] **Step 1: Add Google Sheets — Clear Control cell**

  After the WordPress publish module → Add module → Google Sheets → **Update a Cell**.

  Settings:
  - Spreadsheet: `schädlings-experte`
  - Sheet: `Control`
  - Cell: `A2`
  - Value: *(empty string)*

  This clears the manual override after it's been used.

- [ ] **Step 2: Add Google Sheets — Log published article**

  Add module → Google Sheets → **Add a Row**.

  Settings:
  - Spreadsheet: `schädlings-experte`
  - Sheet: `Topics`
  - Values (map columns A, B, C):
    - A: `{{formatDate(now; "YYYY-MM-DD")}}`
    - B: `{{thema}}`
    - C: `{{wordpress_post_id}}`

- [ ] **Step 3: Full end-to-end dry run**

  Ensure Control A2 is empty (automatic path). Click **Run once**.

  Walk through the execution log and verify every module:
  1. Schedule — triggered ✓
  2. Google Sheets Search Rows — returns topics list (empty on first run) ✓
  3. Text Aggregator — produces empty string or comma-separated list ✓
  4. Google Sheets Get Cell — Control A2 empty ✓
  5. Router — takes automatic path ✓
  6. OpenAI (topic) — returns a specific German pest topic ✓
  7. Set Variable (thema) — thema is set ✓
  8. OpenAI (article) — returns valid JSON ✓
  9. Parse JSON — 6 fields parsed ✓
  10. HTTP PA API — 200 response with 3 products ✓ (or error handler triggers and affiliate_block is empty)
  11. Set Variable (affiliate_block) — HTML or empty ✓
  12. HTTP WordPress — 201 response, post ID returned ✓
  13. Google Sheets Update Cell — Control A2 cleared ✓
  14. Google Sheets Add Row — new row in Topics tab with date, topic, post ID ✓

  Check the published post lives at: `https://schaedlings-experte.de/{{slug}}/`

- [ ] **Step 4: Verify Topics log**

  Open the Google Sheet → Topics tab. A new row should exist with today's date, the topic, and the WordPress post ID.

- [ ] **Step 5: Activate scenario**

  In make.com, toggle the scenario from **OFF** to **ON**.

  The scenario will now run daily and publish on Mon/Wed/Fri at 08:00 CET automatically.

- [ ] **Step 6: Commit final state**

  Update prompts if you made any adjustments during testing:

  ```bash
  git add prompts/ templates/
  git commit -m "feat: prompts finalized after end-to-end test"
  ```

  Add a note to setup-log:

  ```bash
  echo "make.com scenario live as of $(date +%Y-%m-%d). First article published successfully." >> docs/setup-log.md
  git add docs/setup-log.md
  git commit -m "chore: scenario live, end-to-end verified"
  ```

---

## Post-Launch Checklist

These are not implementation tasks but things to check in the first two weeks:

- [ ] Verify second and third scheduled runs complete without errors (check make.com execution history)
- [ ] Confirm Topics log grows correctly (no duplicate topics)
- [ ] Review 3–4 published articles for quality — adjust prompts if needed and commit changes
- [ ] Check Google Search Console (add property for schaedlings-experte.de, submit sitemap: `https://schaedlings-experte.de/sitemap_index.xml`)
- [ ] Verify Amazon Associates account is still active (log in to partnernet.amazon.de)
- [ ] If PA API not yet active: manually add SiteStripe affiliate links to the first batch of articles
