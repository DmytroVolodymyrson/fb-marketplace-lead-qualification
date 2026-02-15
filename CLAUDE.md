# Manulift Facebook Marketplace Lead Generation

## Project Overview

**Client:** Aaron Gaudun-Ungar
**Company:** Manulift (Equipment Dealer)
**Contact:** aarongu31@gmail.com
**Goal:** Automate lead generation by scraping Facebook Marketplace for equipment sellers in Canada

## Business Context

### Company
- **Manulift** - Equipment dealer in Canada
- **Brands Sold:** Merlo (telehandlers), Snorkel (manlifts & scissor lifts)
- **Target Market:** Canada only, private sellers (not dealerships)

### Sales Process
- In-person visits to construction sites
- Online outreach via Facebook Marketplace
- Posts non-paid ads for used machinery
- Sends mass messages to sellers asking why they're selling

### Pain Points (from Discovery Call)
- Lack of structured sales process
- Manual data entry consuming significant time
- Volume of outreach limited due to manual processes
- Unstructured lead tracking

## Equipment Keywords

### Telehandlers
- Generic: `telehandler`, `tele handler`, `telescopic forklift`, `zoom boom`, `carelift`
- Brands: `Merlo`, `JCB`, `Manitou`, `Genie`, `SkyTrak`, `CAT`, `Bobcat`, `JLG`, `Liftking`

### Manlifts / Boom Lifts
- Generic: `manlift`, `man lift`, `boom lift`, `aerial lift`, `basket lift`, `personnel lift`, `cherry picker`, `bucket lift`, `lift basket`
- Brands: `Genie boom`, `JLG boom`, `Skyjack boom`, `Snorkel boom`, `Haulotte boom`

### Scissor Lifts
- Generic: `scissor lift`, `scissor platform`, `electric scissor`, `rough terrain scissor`, `RT scissor lift`, `platform lift`, `lift table`, `scisor lift`
- Brands: `Skyjack scissor`, `Genie scissor`, `Snorkel scissor`, `JLG scissor`, `Haulotte scissor`

## Lead Qualification Rules

### QUALIFY (reach out) when:
- Equipment type: telehandler, manlift, or scissor lift
- Location: Canada
- Seller type: Private seller
- Listing type: For sale (not rental)

### DISQUALIFY (skip) when:
- Rental listing (description contains "rent", "rental", "for rent")
- Outside Canada
- Brand new machine (description contains "new", "0 hours", "brand new")
- Dealership seller (business profile, multiple similar listings)
- Wrong equipment type (loaders, excavators, straight mast forklifts, skid steers)

## Message Templates

### Initial Message Options
1. "Out of curiosity, what has led you to sell this machine?"
2. "How does the machine run?"
3. "Are you looking for a replacement?"

### Follow-up (after seller responds)
"Got it. Have you looked at a [BRAND] before? We can give you a buyback offer on your machine as part of a deal for a new or used [EQUIPMENT_TYPE]. Is that something you'd be interested in taking a quick look at?"

| If selling... | Offer brand |
|---------------|-------------|
| Telehandler | Merlo |
| Manlift | Snorkel |
| Scissor lift | Snorkel |

### Response Handlers
| Customer Response | Next Message |
|------------------|--------------|
| Positive/interested | "Ok great. What's your number?" |
| Negative/hesitant | "What makes you say that?" |
| Asks what's available | "I can check our inventory and give you a call. What works for you?" |

## Technical Resources

### GoHighLevel
- **Location ID:** `nPLb7W745vPcZslsnXTN`
- **Contact ID:** `9B3GRBMLjB2KUspl5kya`
- **Meeting Recording:** https://www.loom.com/share/822060fea48c4c3d86f9e515d39a4715

### Google Sheet
- **Spreadsheet ID:** `1CXHKn6J03zSoh4jOiw43mSWcHYTa0uzLr8yUJstszmk`
- **Title:** Manulift FB Marketplace Leads
- **Sheets:** Keyword Pool, Qualified Leads

### Apify
- **Actor:** `apify/facebook-marketplace-scraper`
- **Cost:** $5 per 1,000 listings
- **Token:** Use existing from n8n credentials

### n8n
- **Workflow Template:** Toufic Instagram Reels Scraper (`7c0YvfukeTC7uxgH`)
- **Original Scraper Workflow:** Manulift FB Marketplace Lead Generation (`vWaS1NbDD9ysFH0d`)
- **Active Qualification Workflow:** Manulift AI Lead Qualification Demo v2 (`g7oQLO7sJpaDohH9`)
- **Instance:** MyNATN

## AI Lead Qualification Workflow (Active)

**Workflow ID:** `g7oQLO7sJpaDohH9` — 22 nodes

### Data Flow
```
Webhook Trigger (GET /webhook-test/...)
    ↓
Read Sheet Rows (Google Sheets - Qualified Leads)
    ↓
Split In Batches (10 at a time)
    ↓
Run FB Details Scraper (Apify actor: apify/facebook-marketplace-scraper)
    ↓
Get Enrichment Data (Apify dataset results)
    ↓
Has Apify Data? (IF node)
    ├── YES → Extract Description (title, price, location, brand, equipment type, images)
    │         ↓
    │         Has Images? (IF node - checks Images array length > 0)
    │         ├── YES → Analyze Listing Images (Code node - downloads images, base64 encodes)
    │         │         → Call OpenAI Vision (HTTP Request - gpt-4o-mini vision)
    │         │         → Format Image Analysis (Set node - extracts response, re-attaches listing data)
    │         │         → AI Agent
    │         └── NO  → Default Image Analysis (Set node - default "no images" JSON)
    │                   → AI Agent
    └── NO  → No Apify Data (Set node - fallback with defaults)
              → AI Agent
                ↓
AI Agent (gpt-4o-mini + Structured Output Parser)
    - Classifies: PRIVATE_SELLER, DEALER, RENTAL, DISQUALIFY, UNCERTAIN
    - Uses text (title, description, price, location) + image analysis context
    ↓
Merge AI Result (Set node - combines classification fields)
    ↓
Update Qualified Leads Row (Google Sheets - writes classification + image summary)
    ↓
Loop Back → Split In Batches (next item)
```

### Image Analysis Architecture

The image analysis pipeline downloads FB Marketplace listing photos server-side and sends them to OpenAI Vision as base64-encoded images.

**Why base64 instead of URLs:** Facebook CDN URLs are signed/authenticated and inaccessible from OpenAI's servers. Images must be downloaded within n8n and converted to base64.

**Key node: "Analyze Listing Images (Code)"**
- Uses `this.helpers.httpRequest` (n8n's built-in HTTP helper) to download images
- Downloads first 3 images per listing as arraybuffer
- Converts to base64 `data:image/jpeg;base64,...` format
- Builds OpenAI Chat Completions request body with vision content
- Uses `detail: "low"` for cost savings (~$0.004-0.007 per listing)
- Falls back gracefully if images can't be downloaded

**Image analysis evaluates:**
1. Equipment count (single, few, many)
2. Setting (residential, farm, jobsite, dealer lot, warehouse, showroom)
3. Photo style (casual, semi-professional, professional)
4. Dealer signals (signage, logos, watermarks, inventory rows, etc.)
5. Private seller signals (home driveway, single machine, casual background, etc.)

**Important:** Image analysis is descriptive only — no dealer_probability scoring. The AI Agent makes the final classification decision using both text and image context.

### AI Agent Classification

The AI Agent receives text fields + image analysis JSON and outputs structured classification:
- `seller_type`: PRIVATE_SELLER | DEALER | RENTAL | DISQUALIFY | UNCERTAIN
- `confidence`: 1-100
- `equipment_type`: telehandler | manlift | scissor_lift | other | unknown
- `brand_detected`: string or null
- `disqualify_reason`: string (if applicable)
- `notes`: brief explanation
- `image_summary`: what the listing photos show

## Original Scraper Workflow (Reference)

```
[Schedule Trigger - Daily 7 AM]
    ↓
[Read Keyword Pool from Google Sheets]
    ↓
[Select 10-15 Keywords (rotation + cooldown)]
    ↓
[For Each Keyword]
    ↓
[Apify FB Marketplace Scraper]
    ↓
[Get Dataset Results]
    ↓
[Qualification Filter]
    ├── PASS → Transform → Deduplicate → Append to Sheet
    └── FAIL → Skip
    ↓
[Update Keyword Stats]
    ↓
[Email Summary]
```

## Data Fields

### Keyword Pool Sheet
| Column | Description |
|--------|-------------|
| Keyword | Search term |
| Category | telehandler/manlift/scissor |
| Last Used | Date last searched |
| Total Leads | Cumulative leads found |
| Zero Runs | Consecutive runs with 0 results |
| Status | active/paused |

### Qualified Leads Sheet
| Column | Description |
|--------|-------------|
| Listing URL | FB Marketplace link |
| Listing Title | Equipment title |
| Price | Listed price |
| Description | Full listing description |
| Location | City/Province |
| Seller Name | From listing |
| Seller Profile URL | FB profile link |
| Equipment Type | telehandler/manlift/scissor |
| Brand (Detected) | Extracted from title/description |
| Post Date | When listed |
| Date Scraped | When we found it |
| Status | new/contacted/responded/converted |
| Notes | For sales team |
| AI Seller Type | PRIVATE_SELLER/DEALER/RENTAL/DISQUALIFY/UNCERTAIN |
| AI Confidence | 1-100 confidence score |
| AI Equipment Type | AI-detected equipment type |
| AI Brand | AI-detected brand |
| AI Notes | AI classification reasoning |
| Image Summary | Description of what listing photos show |

## Canadian Location Validation

### Valid Provinces
- Ontario (ON), Quebec (QC), British Columbia (BC), Alberta (AB)
- Manitoba (MB), Saskatchewan (SK), Nova Scotia (NS), New Brunswick (NB)
- Newfoundland (NL), Prince Edward Island (PE)

### Postal Code Pattern
`/^[A-Za-z]\d[A-Za-z][ -]?\d[A-Za-z]\d$/`

## n8n Technical Notes

### Code Node Limitations (n8n v2 sandbox)
- `this.helpers.httpRequest` — available, works for unauthenticated HTTP requests (used for FB CDN image downloads)
- `this.helpers.httpRequestWithAuthentication` — NOT available in Code node
- `fetch()` — exists but fails silently on FB CDN URLs; prefer `this.helpers.httpRequest`
- For authenticated API calls (e.g., OpenAI), use a separate HTTP Request node with n8n credentials

### n8n MCP Partial Update API Gotchas
- `addConnection` can create malformed connections with `"0"` type instead of `"main"`
- `removeConnection` then can't find these malformed connections to fix them
- When connections get corrupted, use full workflow update (`n8n_update_full_workflow`) to reset all connections

### OpenAI Credentials
- **Credential ID:** `SkjoKB33iQQV1xYW` (openAiApi)
- Used by: AI Agent node (via langchain), Call OpenAI Vision HTTP Request node (via header auth)

## Files in Project

- `Aaron - Manulift.pdf` - Client specification document with keywords and message templates
- `AI-Lead-Qualification-Plan.md` - Initial plan for AI-powered lead qualification (Phase 1: text, Phase 2: images)
- `conversation-history.md` - DM history with Aaron and project context
- `CLAUDE.md` - This documentation file

## Success Metrics

From client examples:
- CAT TH460B Telehandler - $35,000 (Ontario) → Got phone number
- 2005 Liftking lk60 - $33,000 (Highlands East, ON) → Got phone number

Goal: Find sellers → Offer trade-in/buyback → Convert to buyers
