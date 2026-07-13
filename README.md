# Local Business Lead Pipeline

An [n8n](https://n8n.io) workflow that pulls local business leads scraped via [Apify](https://apify.com), validates and deduplicates them, saves new leads to [Supabase](https://supabase.com), and emails a run summary.

## 📋 Table of Contents

- [What It Does](#what-it-does)
- [Data Flow](#data-flow)
- [Lead Record Fields](#lead-record-fields)
- [Requirements](#requirements)
- [Setup & Import Instructions](#setup--import-instructions)
- [Environment Variables](#environment-variables)
- [Security Notes](#security-notes)
- [License](#license)

---

## What It Does

**File:** `local-business-lead-pipeline— Apify Import.json`

1. **Manual trigger** – Run manually after configuring the Apify dataset ID for a completed scrape.
2. **Set Dataset ID** – Set the Apify dataset ID and a human-readable description of the search (both edited directly in the node before each run).
3. **Fetch Dataset Items** – Pulls up to 500 items from the specified Apify dataset.
4. **Parse & Map Results** – Normalizes raw scrape data into a consistent lead shape: business name, phone, email, website/domain, city, state.
5. **Validate & filter**:
   - Must have a business name
   - Must be located in Jacksonville
   - Must have a phone number or email (website-only leads are skipped)
   - Duplicate phone numbers, emails, or domains within the same run are skipped
6. **Save Lead to Supabase** – Inserts each valid lead into the `glow_guys_leads` table, ignoring rows that already exist (matched via a unique constraint).
7. **Build Summary** – Tallies totals: raw fetched, valid, skipped (with reasons), newly saved, duplicates, and failures. Also splits leads into "phone" (actionable now) vs "email-only" (needs enrichment).
8. **Save Import Log** – Records the run's stats into a `glow_guys_import_logs` table for historical tracking.
9. **Send Summary Email** – Emails an HTML report of the run, including a breakdown of why any leads were filtered out.

## Data Flow

```
Apify Dataset → Parse/Normalize → Validate & Dedupe → Supabase (leads) → Build Summary → Supabase (logs) → Email
```

## Lead Record Fields

| Field           | Description |
|-----------------|-------------|
| `business_name` | From the scrape's title/name field |
| `category`      | Business category/type from Apify |
| `phone`         | Raw phone number, if present |
| `email`         | Business email, if present |
| `website`       | Website URL, if present |
| `domain`        | Derived from website (not from Google Maps URLs) or email domain |
| `city`, `state` | Location; state is normalized to a 2-letter code |
| `source`        | Always `apify` |
| `status`        | `new` (has phone) or `email_only` (email but no phone) |

---

## Requirements

- An [n8n](https://n8n.io) instance — n8n Cloud, self-hosted, or [desktop](https://n8n.io/download)
- An [Apify](https://apify.com) account with a completed scraping run/dataset
- A [Supabase](https://supabase.com) project with two tables: `glow_guys_leads` and `glow_guys_import_logs` (schema should match the fields written by the workflow — see [Lead Record Fields](#lead-record-fields) and the "Save Import Log" node for log fields)
- A Gmail account for sending run summaries (or swap the Gmail node for any other email node)

## Setup & Import Instructions

1. **Import the workflow**: in n8n, go to **Workflows → Import from File**, and select `local-business-lead-pipeline— Apify Import.json`.
2. **Set environment variables** — see [Environment Variables](#environment-variables) below.
3. **Set up Supabase tables**: create `glow_guys_leads` (with a unique constraint on phone/email/domain to make the "ignore duplicates" behavior work) and `glow_guys_import_logs`.
4. **Before each run**: open the **Set Dataset ID** node and update `datasetId` with the Apify dataset ID from your latest scrape, and update `searchDescription` with a short label for that search (used in the summary email).
5. **Run manually** via the Manual Trigger — this workflow is designed for on-demand runs, not scheduled automation, since a new Apify dataset ID is needed each time.

## Environment Variables

This workflow reads all secrets and endpoints from environment variables — nothing sensitive is stored in the workflow file itself. Set these in your n8n instance (Settings → Variables on n8n Cloud, or your `.env`/Docker config if self-hosted):

| Variable | Description |
|---|---|
| `APIFY_TOKEN` | Your Apify API token |
| `SUPABASE_URL` | Your Supabase project URL, e.g. `https://yourproject.supabase.co` |
| `SUPABASE_ANON_KEY` | Supabase anon/public API key |
| `SUPABASE_SERVICE_ROLE_KEY` | Supabase service_role key — grants full database access, treat as highly sensitive (see [Security Notes](#security-notes)) |
| `NOTIFY_EMAIL` | Address that should receive the run summary email |

## Security Notes

- **The Supabase `service_role` key bypasses Row Level Security** and grants full read/write/delete access to your entire Supabase project, not just the leads table. Never hardcode it in the workflow file, never commit it to version control, and store it only as an environment variable on your n8n instance.
- **Never commit real credentials, API keys, or tokens.** If you ever export this workflow again from n8n after editing it, double-check that no plaintext secrets were reintroduced into `Set`/`Config` nodes or HTTP node parameters before committing.
- **Sanitize `pinData`.** If you pin test data while editing this workflow, it gets saved in the export and may contain real lead data. Clear pinned data before exporting/committing.
- **Rotate any credential immediately** if you suspect it was ever exposed in a public location, committed to git history, or shared outside your team.

## License

This project is licensed under the MIT License — see [LICENSE](LICENSE) for details.
