# AI-Powered Lead Qualification Plan for Manulift FB Marketplace Scraper

## Problem Statement
Client feedback indicates only **10% of 600 scraped leads are useful**:
- 75% are dealers (not private sellers)
- 10% are low-value listings
- 5% are out-of-scope (rentals, wrong equipment)
- Only 10% are valid leads

**Goal:** Improve lead qualification rate from 10% to 50%+ using AI-powered filtering.

---

## Research Findings

### 1. n8n + OpenAI Vision Integration
- **Native support:** n8n has built-in OpenAI "Analyze Image" operation
- **Models:** gpt-4o, gpt-4-turbo, gpt-4o-mini available
- **Input types:** Base64 encoded images or image URLs
- **Existing templates:** Multiple n8n workflows exist for image analysis
- **AI Agent node:** Can combine vision + text analysis in multi-step workflows

### 2. Object Counting Capabilities
- **GPT-4 Vision limitations:** Struggles with crowded scenes (hallucination risk)
- **Simple scenes:** Can reliably count 4-10 objects
- **Complex scenes:** May count 2,500 items as 1
- **Recommendation:** Use for "multiple vs single" detection, not exact counts
- **Alternative:** Google Cloud Vision Object Localization for precise counting

### 3. Dealer vs Private Seller Detection (Text Analysis)
- **LLM classification:** 90-99% precision with proper prompts
- **No training data needed:** Just clear rules in prompts
- **Key indicators for dealers:**
  - Multiple similar listings
  - Business profile language
  - Professional descriptions (specs, warranty mentions)
  - Pricing patterns (negotiable, financing)
  - Contact info (business phone, website)
- **Private seller indicators:**
  - Personal story ("upgrading," "moving," "don't need")
  - Single item focus
  - Casual language
  - Home location photos

### 4. Image-Based Dealer Detection
- **Multiple machines in photo = dealer** (inventory lot)
- **Professional studio backgrounds = dealer**
- **Outdoor/garage setting = likely private**
- **Watermarks/logos on images = dealer**
- **Implementation:** GPT-4o Vision with specific prompts

---

## Recommended Implementation

### Phase 1: Enhanced Text Analysis (Primary Filter)
Use OpenAI to analyze listing title + description with structured output.

**Classification prompt:**
```
Analyze this Facebook Marketplace listing and classify the seller type.

LISTING TITLE: {title}
DESCRIPTION: {description}
LOCATION: {location}
PRICE: {price}

CLASSIFY AS:
1. DEALER - If any of these apply:
   - Business/company name in description
   - Multiple units available
   - Mentions financing, warranty, delivery
   - Professional/formal language
   - Website, business phone, or email
   - "Call for pricing" or similar

2. PRIVATE_SELLER - If:
   - Personal reason for selling (upgrading, moving, don't need)
   - Single item described
   - Casual/informal language
   - Residential location implied

3. RENTAL - If listing mentions rent, rental, lease, per day/week/month

4. DISQUALIFY - If:
   - Brand new / 0 hours
   - Outside Canada
   - Wrong equipment (loader, excavator, skid steer, straight mast forklift)

OUTPUT JSON:
{
  "seller_type": "DEALER|PRIVATE_SELLER|RENTAL|DISQUALIFY",
  "confidence": 0-100,
  "equipment_type": "telehandler|manlift|scissor_lift|unknown",
  "brand_detected": "string or null",
  "disqualify_reason": "string or null",
  "notes": "brief explanation"
}
```

### Phase 2: Image Analysis (Secondary Validation)
For listings that pass text analysis, analyze images.

**Image analysis prompt:**
```
Analyze this equipment listing photo.

COUNT:
- How many pieces of heavy equipment are visible? (1, 2-3, 4+)

SETTING:
- Indoor warehouse/showroom
- Outdoor dealership lot (organized, multiple machines)
- Residential/farm/construction site
- Unknown

PHOTO QUALITY:
- Professional (studio lighting, clean background)
- Semi-professional (clear but not staged)
- Casual (phone camera, cluttered background)

DEALER INDICATORS (yes/no):
- Multiple machines visible
- Business signage/logos
- Price stickers/tags
- Organized inventory rows
- Watermarks on image

OUTPUT JSON:
{
  "equipment_count": "single|few|many",
  "setting": "warehouse|dealer_lot|residential|construction|unknown",
  "photo_quality": "professional|semi|casual",
  "dealer_probability": 0-100,
  "dealer_indicators": ["list of observed indicators"]
}
```

### Phase 3: Combined Scoring
```
Final Score = (Text Analysis Confidence × 0.7) + (Image Analysis × 0.3)

PASS (add to qualified leads):
- seller_type = "PRIVATE_SELLER" AND confidence > 70%
- OR seller_type = "PRIVATE_SELLER" AND image dealer_probability < 30%

REVIEW (manual check needed):
- seller_type = "PRIVATE_SELLER" AND confidence 40-70%
- Mixed signals between text and image

REJECT:
- seller_type = "DEALER" or "RENTAL" or "DISQUALIFY"
- OR image dealer_probability > 70%
```

---

## n8n Workflow Implementation

### Node Structure:
```
[Apify FB Marketplace Scraper]
    ↓
[Loop Over Items]
    ↓
[OpenAI - Text Classification]
    ↓
[IF seller_type = PRIVATE_SELLER]
    ├── YES → [OpenAI - Image Analysis] → [Calculate Final Score]
    └── NO → [Log Rejection] → Skip
    ↓
[IF Final Score > threshold]
    ├── PASS → [Add to Qualified Leads Sheet]
    └── REVIEW → [Add to Review Queue Sheet]
```

### Key n8n Nodes:
1. **OpenAI Node** (Text Classification)
   - Resource: Chat
   - Model: gpt-4o-mini (cost-effective for text)
   - Structured output with JSON schema

2. **OpenAI Node** (Image Analysis)
   - Resource: Image
   - Operation: Analyze Image
   - Model: gpt-4o (better vision)
   - Input: Image URL from listing

3. **IF Node** - Route based on classification
4. **Set Node** - Calculate combined score
5. **Google Sheets Node** - Output to appropriate sheet

---

## Cost Estimation

| Component | Cost per 1000 listings |
|-----------|----------------------|
| Text Analysis (gpt-4o-mini) | ~$0.50 |
| Image Analysis (gpt-4o) | ~$5-10 |
| **Total** | ~$5-11 per 1000 |

**Optimization:** Only run image analysis on listings that pass text filter (~25% of total), reducing image analysis costs by 75%.

---

## Expected Results

| Metric | Before | After |
|--------|--------|-------|
| Valid lead rate | 10% | 50-60% |
| Dealer false positives | 75% | 10-15% |
| Manual review needed | 100% | 10-20% |
| Cost per qualified lead | $0.50 | $0.15-0.20 |

---

## Alternative Tools Discovered

1. **Vettx AI** - Commercial "AI Image Dealer Detection" (90%+ accuracy)
2. **Kaihive CoPilot** - Filters 95% of dealer listings
3. **CarFinder** - AI filters for dealer/flipper detection
4. **Google Cloud Vision** - Object Localization for precise counting

---

## Implementation Steps

1. **Update n8n workflow** (`vWaS1NbDD9ysFH0d`)
   - Add OpenAI text classification after scraper
   - Add conditional image analysis
   - Add scoring logic
   - Route to different output sheets

2. **Create new Google Sheets tabs**
   - "Qualified Leads" (score > 70%)
   - "Review Queue" (score 40-70%)
   - "Rejected" (with reason logged)

3. **Test with sample data**
   - Run on 100 existing scraped listings
   - Manually verify classification accuracy
   - Adjust prompts based on results

4. **Deploy and monitor**
   - Enable workflow
   - Track qualification rates daily
   - Refine prompts weekly based on feedback

---

## Files to Modify

- **n8n Workflow:** `vWaS1NbDD9ysFH0d` (Manulift FB Marketplace Lead Generation)
- **Google Sheet:** `1CXHKn6J03zSoh4jOiw43mSWcHYTa0uzLr8yUJstszmk` (add new tabs)

## Verification

1. Run workflow on 100 sample listings
2. Manually verify 20 classified as "PRIVATE_SELLER"
3. Manually verify 20 classified as "DEALER"
4. Calculate precision/recall
5. Adjust prompts if accuracy < 80%
