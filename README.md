# URL-to-Document RAG Pipeline Feeder

An n8n workflow that automatically converts URLs into RAG-ready documents with AI-generated metadata. Built for environmental compliance document management.

## What It Does

This workflow processes a queue of URLs and produces structured, searchable documents:

1. **Fetch** - Pulls unprocessed URLs from a Supabase `rag_update_queue` table
2. **Crawl** - Sends each URL to a [Crawl4AI](https://github.com/unclecode/crawl4ai) instance for web scraping and markdown conversion
3. **Clean** - Strips excessive whitespace and calculates word count / file size metrics
4. **Store** - Saves the cleaned document to a `documents_and_metadata` table in Supabase
5. **Summarize** - An LLM agent extracts title, summary, effective date, and key terms
6. **Classify** - A second LLM agent assigns topic categories and geographic scope using a compliance-focused taxonomy
7. **Mark Complete** - Updates the queue record as processed

A separate **duplicate cleanup** sub-workflow runs on a 3-day schedule to deduplicate the queue and log cleanup actions.

## Architecture

```
┌──────────────┐     ┌───────────┐     ┌──────────┐     ┌──────────────┐
│  Supabase    │────>│ Crawl4AI  │────>│  Clean & │────>│   Supabase   │
│  Queue       │     │  (HTTP)   │     │  Parse   │     │   Store Doc  │
└──────────────┘     └───────────┘     └──────────┘     └──────┬───────┘
                                                               │
                                                    ┌──────────▼───────────┐
                                                    │  AI Agent: Summary   │
                                                    │  (title, key terms,  │
                                                    │   effective date)    │
                                                    └──────────┬───────────┘
                                                               │
                                                    ┌──────────▼───────────┐
                                                    │  AI Agent: Topics    │
                                                    │  (main_topics,       │
                                                    │   geographic_scope)  │
                                                    └──────────┬───────────┘
                                                               │
                                                    ┌──────────▼───────────┐
                                                    │  Mark Queue Item     │
                                                    │  as Processed        │
                                                    └──────────────────────┘
```

## Tech Stack

- **[n8n](https://n8n.io/)** - Workflow automation
- **[Crawl4AI](https://github.com/unclecode/crawl4ai)** - Web crawling and markdown extraction
- **[Supabase](https://supabase.com/)** - Database (PostgreSQL) for queue and document storage
- **[OpenRouter](https://openrouter.ai/)** - LLM API gateway for metadata extraction

## Database Tables

| Table | Purpose |
|-------|---------|
| `rag_update_queue` | Incoming URLs to process (`source_url`, `processed`, `flagged_at`) |
| `documents_and_metadata` | Processed documents with full metadata |
| `duplicate_cleanup_log` | Audit log for the deduplication sub-workflow |

## Setup

### Prerequisites

- n8n instance (self-hosted or cloud)
- Crawl4AI server running and accessible
- Supabase project with the tables above
- OpenRouter API key

### Installation

1. Import `URL_to_Document.json` into your n8n instance
2. Configure credentials in n8n:
   - **Supabase** - your project URL and service key
   - **OpenRouter** - your API key
   - **Postgres** - direct connection for the dedup query (uses your Supabase Postgres connection string)
3. Update the Crawl4AI URL in the HTTP Request node to point to your instance
4. Create the required database tables in Supabase
5. Activate the workflow

## Topic Taxonomy

The classification agent categorizes documents across these dimensions:

- **Jurisdictions** - Federal, EPA, state-level (AL, FL, GA, MS, NC, SC, TN)
- **Resources** - Water Quality, Stormwater, Wetlands, Air Quality, Wildlife, etc.
- **Programs** - NPDES, Section 404, CWA, NEPA, ESA, etc.
- **Activities** - Construction, Permits, Development, Mining, etc.
- **Doc Types** - Regulation, Guidance, Court Decision, Proposed Rule, etc.
- **Geographic Scope** - National, Regional, State-specific, Multi-State, Local

## License

MIT
