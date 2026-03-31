# Alumni Engagement Pipeline

Segment alumni, score engagement, identify high-potential donors, and generate personalized outreach at scale — fully automated with scheduled quarterly, monthly, and campaign runs.

## Workflow Diagram

```
                    ┌─────────────────────────────────────────────┐
                    │         quarterly-engagement-pipeline        │
                    │   (also: monthly-score-refresh, outreach-   │
                    │          campaign — see schedules)          │
                    └─────────────────────────────────────────────┘

  CSV exports / alumni-master.json
          │
          ▼
  ┌───────────────┐     ┌───────────────┐     ┌───────────────┐
  │ data-ingester │────▶│ segmentation  │────▶│  engagement   │
  │               │     │    engine     │     │    scorer     │
  │ normalize,    │     │               │     │               │
  │ deduplicate,  │     │ temporal,     │     │ weighted 5-   │
  │ profile build │     │ degree, geo,  │     │ factor model  │
  │               │     │ career, engmt │     │ (0-100 score) │
  └───────────────┘     └───────────────┘     └───────────────┘
                                                      │
                         ┌────────────────────────────┘
                         │
                         ▼
  ┌───────────────┐     ┌───────────────┐
  │donor-          │    │  outreach-    │
  │identifier     │────▶│   writer      │
  │               │     │               │
  │ MG/ML/AF/SO   │     │ reunion,      │
  │ tiering,      │     │ re-engage,    │
  │ prospect list,│     │ annual fund,  │
  │ top-25 brief  │     │ volunteer ask │
  └───────────────┘     └───────────────┘
          │                     │
          └──────────┬──────────┘
                     │
                     ▼
            ┌────────────────┐
            │ review-outputs │◀─── rework loop (up to 2x)
            │  (quality gate)│
            └───────┬────────┘
                    │ approved
                    ▼
  ┌───────────────┐     ┌───────────────┐     ┌──────────────┐
  │ build-event-  │────▶│   generate-   │────▶│     VP       │
  │ invitation-   │     │    reports    │     │ Advancement  │
  │    list       │     │               │     │  review      │
  │  (Python cmd) │     │ dashboards,   │     │  (manual     │
  │               │     │ pipeline rpt, │     │   gate)      │
  │               │     │ outreach rdy  │     │              │
  └───────────────┘     └───────────────┘     └──────────────┘
```

## Quick Start

```bash
cd examples/alumni-engagement
ao daemon start

# Run the full quarterly pipeline
ao queue enqueue --title "Q1 Alumni Engagement" --workflow-ref quarterly-engagement-pipeline

# Or run a quick monthly score refresh
ao workflow run monthly-score-refresh

# Watch it run
ao daemon stream --pretty
```

## Agents

| Agent | Model | Role |
|---|---|---|
| **data-ingester** | claude-haiku-4-5 | Ingests CSV exports, normalizes records, deduplicates, builds alumni profiles |
| **segmentation-engine** | claude-haiku-4-5 | Segments alumni by temporal, degree, geographic, career, and engagement dimensions |
| **engagement-scorer** | gemini/gemini-2.5-flash | Calculates weighted 5-factor engagement scores (0-100) with trend tracking |
| **donor-identifier** | claude-opus-4-6 | Tiers prospects (MG/ML/AF/SO), builds ranked prospect lists, top-25 brief |
| **outreach-writer** | claude-sonnet-4-6 | Writes personalized HTML email templates for each segment and campaign type |
| **report-generator** | claude-sonnet-4-6 | Produces quarterly dashboards, fundraising pipeline, and outreach readiness reports |

## AO Features Demonstrated

- **Scheduled workflows** — quarterly pipeline, monthly refresh, weekly campaign toggle
- **Multi-agent pipeline** — 6 specialized agents with distinct models and responsibilities
- **Command phases** — Python3 script for invitation list assembly from pipeline outputs
- **Decision contracts** — quality gate with rework routing (up to 2 retries) before VP review
- **Output contracts** — structured output validation at each pipeline stage
- **Manual gate** — VP Advancement review before outreach campaigns activate
- **Memory MCP** — persistent alumni segment assignments across pipeline runs
- **Rework loops** — `review-outputs` phase routes back to `ingest-alumni-data` on failure
- **Multiple models** — haiku for data work, gemini for scoring, opus for donor strategy, sonnet for writing

## Requirements

### API Keys
None required for the core pipeline — all processing is local.

### MCP Servers (auto-installed via npx)
| Server | Package |
|---|---|
| filesystem | `@modelcontextprotocol/server-filesystem` |
| memory | `@modelcontextprotocol/server-memory` |
| sqlite | `@berthojoris/mcp-sqlite-server` |
| sequential-thinking | `@modelcontextprotocol/server-sequential-thinking` |

### Tools
- `python3` — for invitation list assembly command phase
- `ao` CLI with daemon support

## Input Data

Place alumni CSV exports in `data/alumni-exports/`. See `data/sample/alumni-sample.csv` for the expected schema:

```
id, first_name, last_name, email, graduation_year, degree_type, major,
employer, job_title, city, state, country, phone,
last_donation_date, total_giving, event_count, volunteer_hours, email_opens_last_year
```

If no exports are present, the pipeline uses the included sample data.

## Output Structure

```
output/
├── profiles/           # Normalized alumni profiles ({alumni_id}.json)
├── segments/           # Segment definitions and member lists
├── scores/             # Engagement scores and history
├── prospects/          # Donor tiers, prospect list, top-25 brief
├── outreach/           # HTML email templates by segment and campaign
├── events/             # Event invitation lists
└── reports/            # Quarterly reports and dashboards
```

## Schedules

| Schedule | Cron | Workflow |
|---|---|---|
| Quarterly pipeline | `0 6 1 1,4,7,10 *` | `quarterly-engagement-pipeline` |
| Monthly score refresh | `0 7 1 * *` | `monthly-score-refresh` |
| Weekly outreach | `0 8 * * 1` | `outreach-campaign` (disabled by default) |
