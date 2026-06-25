# Phase 2: Cookie-Based Seller Profile Scraping

**Status:** Not yet implemented — come back to this after Phase 1 is verified in production.
**Workflow:** `g7oQLO7sJpaDohH9` (Manulift AI Lead Qualification Demo v2)

## Why This Matters

Phase 1 (enhanced text + image analysis) improved dealer detection significantly, but the **most powerful signal** — analyzing the seller's other active listings — requires authenticated access to Facebook. A seller with 8+ Merlo machines is obviously a dealer; a seller with 1 machine + an Apple Watch + theatre seats is obviously private.

Facebook locked down all seller identity data from anonymous/cookieless scrapers. To access seller profiles, we need authenticated browser sessions.

## Architecture: n8n-nodes-puppeteer with Cookie Injection

### How It Works

1. User exports Facebook cookies using **Cookie-Editor** Chrome extension (JSON export)
2. Cookies stored in an n8n Set node or credential
3. For each listing, a Puppeteer node:
   - Injects Facebook cookies into the browser session
   - Navigates to the listing URL
   - Extracts **seller name** + **profile link** from the listing page
   - Navigates to the seller's marketplace profile (`/marketplace/profile/{seller_id}/`)
   - Counts their **active listings** and extracts **listing titles**
   - Returns: seller name, profile URL, total active listings, listing titles array
4. Data fed to the AI Agent alongside existing text + image analysis

### Toggle/Skip Mechanism

A config Set node at the top of the workflow with `enableSellerScraping: true/false`:
- **true** → Route through Puppeteer seller profile scraping before AI Agent
- **false** → Skip directly to existing listing analysis flow (current Phase 1 behavior)

An IF node after "Split In Batches" checks this toggle and routes accordingly.

## Nodes to Add

| Node | Type | Purpose |
|------|------|---------|
| Config: Enable Seller Scraping | Set | Toggle boolean at workflow start |
| Check Seller Scraping Enabled | IF | Routes based on toggle |
| Set Facebook Cookies | Set | Stores cookie JSON for injection |
| Scrape Seller Profile | Puppeteer (Custom Script) | Visits listing → extracts seller → visits profile → counts listings |
| Parse Seller Data | Set/Code | Formats Puppeteer output for AI Agent |

## AI Agent Prompt Addition (when seller data available)

```
SELLER PROFILE DATA (when available):
- seller_name: The seller's Facebook name
- total_active_listings: Number of currently active marketplace listings
  - 1-3 listings: STRONG private seller signal
  - 4-7 listings: Could be active private seller or small dealer — check listing variety
  - 8+ listings: STRONG dealer signal, especially if mostly equipment
- listing_titles: Array of the seller's other listing titles
  - Mixed items (watches, furniture, car parts + 1-2 machines): PRIVATE SELLER
  - All/mostly equipment (telehandlers, scissor lifts, boom lifts): DEALER
  - Multiples of same brand: VERY STRONG dealer signal
```

## Prerequisites

1. **Install n8n-nodes-puppeteer** community node on the n8n instance (Railway)
2. **Export Facebook cookies** — user logs into Facebook, exports cookies via Cookie-Editor extension as JSON
3. **Stealth mode** — enable in Puppeteer to reduce detection risk
4. **Consider browserless.io** — remote browser service to avoid running Chrome on the n8n server (reduces memory/CPU load)

## Cookie-Based Apify Alternative

`alien_force/facebook-scraper-pro` (FREE) has a `facebook_profiles_scraper` function that accepts cookies and returns:
- Profile category (Person/Business)
- Avatar URL
- Follower count
- Account created_at

Requires profile URLs as input (which we'd get from the Puppeteer step). Could be a supplementary enrichment for flagged sellers.

## Risk Considerations

- **Cookie expiry:** Facebook cookies typically expire in 30-90 days — need periodic refresh
- **Rate limiting:** Heavy scraping could flag/ban the Facebook account
- **Mitigation:** Add 3-5 second delays between requests, limit to 10 listings per run
- **Stealth mode:** Use Puppeteer stealth plugin to avoid bot detection
- **Separate account:** Consider using a dedicated Facebook account for scraping (not a personal one)

## Expected Impact

| Signal | Accuracy | Impact |
|--------|----------|--------|
| Seller listing count (1-3 vs 8+) | Very High | Directly answers dealer vs private |
| Listing variety (mixed items vs all equipment) | Very High | Most powerful single signal per Aaron |
| Seller profile type (Person vs Business page) | High | Direct business indicator |

Combined with Phase 1, expected valid lead rate: **70-80%+** (up from ~25% baseline).

## Implementation Estimate

- Install Puppeteer node: ~15 min
- Build Puppeteer script: ~2 hours (custom script for FB navigation, seller extraction, profile scraping)
- Add toggle + routing nodes: ~30 min
- Update AI Agent prompt: ~15 min
- Testing: ~1 hour
- **Total: ~4 hours**
