# FB Marketplace Lead Qualification

AI-powered lead qualification for heavy equipment listings on Facebook Marketplace. Built with n8n.

## What It Does

Reads equipment listings from a Google Sheet, enriches them with Apify scraping, and classifies each seller as private or dealer using GPT-4o-mini.

```
[Google Sheet: Qualified Leads]
        ↓
[Filter: Not Yet Classified]
        ↓
[Batch Process (10 at a time)]
        ↓
[Already has description?]
    ├── YES → Use cached data
    └── NO → Apify FB Marketplace Scraper
                    ↓
              [Has data?]
                ├── YES → Extract description, images, location
                └── NO → Mark as unavailable
                    ↓
[AI Agent: Classify seller type]
        ↓
[Write results back to Google Sheet]
```

**Total Nodes:** 19 | **AI Model:** GPT-4o-mini

## AI Classification

The AI agent classifies each listing into one of these categories:

| Type | Description | Action |
|------|-------------|--------|
| `PRIVATE_SELLER` | Individual selling their equipment | Reach out |
| `DEALER` | Equipment dealership (competitor) | Skip |
| `RENTAL` | Rental company | Skip |
| `UNCERTAIN` | Not enough signals | Manual review |
| `DISQUALIFY` | Wrong category or parts listing | Skip |

### Classification Signals

**Private Seller** (target): personal language ("my", "selling because"), specific usage history, single item, honest condition notes

**Dealer** (skip): financing language, marketing formatting, multiple inventory hints, warranty packages

**Rental** (skip): rate language ("daily rates", "weekly rates"), fleet indicators, 1-800 numbers

## Google Sheet Structure

The workflow reads from and writes to a "Qualified Leads" sheet with these columns:

| Column | Source |
|--------|--------|
| Listing URL | Input |
| Listing Title | Input |
| Price | Input |
| Location | Input |
| Equipment Type | Input |
| Brand | Input |
| Description | Apify enrichment |
| Seller Name | Apify enrichment |
| FB Seller Type | Apify enrichment |
| AI Seller Type | AI classification |
| AI Reason | AI classification |

## Setup

### Prerequisites

- n8n instance (self-hosted or cloud)
- API keys for:
  - [OpenAI](https://platform.openai.com/) - GPT-4o-mini for classification
  - [Apify](https://apify.com/) - Facebook Marketplace scraping
  - [Google Cloud](https://console.cloud.google.com/) - Google Sheets OAuth2

### Installation

1. **Import the workflow** into n8n:
   - Download `workflow.json`
   - In n8n: Workflows > Import from File

2. **Configure credentials:**
   - **OpenAI API** - Apply to the "OpenAI Chat Model" node
   - **Apify API** - Apply to the "Run FB Details Scraper" node
   - **Google Sheets OAuth2** - Apply to both Google Sheets nodes

3. **Update placeholders** in the workflow:
   - Replace `YOUR_SPREADSHEET_ID` in both Google Sheets nodes
   - Replace `YOUR_APIFY_TOKEN` in the "Get Enrichment Data" URL

4. **Create the Google Sheet** with a "Qualified Leads" tab matching the columns above

5. **Activate the workflow** - trigger manually or via webhook at `/webhook/manulift-demo`

## Cost Estimates

Per listing classification:
- Apify scraping: ~$0.005
- OpenAI GPT-4o-mini: ~$0.001
- **Total: ~$0.006 per listing**

## Technologies Used

- **[n8n](https://n8n.io/)** - Workflow automation
- **[OpenAI GPT-4o-mini](https://platform.openai.com/)** - AI classification
- **[Apify](https://apify.com/)** - Facebook Marketplace scraping
- **[Google Sheets API](https://developers.google.com/sheets/api)** - Data storage

## License

MIT
