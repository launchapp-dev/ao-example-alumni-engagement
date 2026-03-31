# Alumni Engagement Pipeline — Agent Context

## What This Project Does

This is an alumni engagement and donor identification pipeline for university advancement offices.
It automates the full lifecycle: ingesting alumni CSV exports → segmenting by cohort → scoring
engagement → identifying donor prospects → generating personalized outreach → producing reports.

## Directory Layout

```
alumni-engagement/
├── .ao/workflows/          # AO workflow configuration
│   ├── agents.yaml         # 6 agents: ingester, segmentation, scorer, donor-id, writer, reporter
│   ├── phases.yaml         # 10 phases including command phase and manual gate
│   ├── workflows.yaml      # 3 workflows: quarterly, monthly refresh, outreach campaign
│   ├── mcp-servers.yaml    # filesystem, memory, sqlite, sequential-thinking
│   └── schedules.yaml      # quarterly, monthly, weekly cron schedules
├── data/
│   ├── alumni-exports/     # Drop raw CSV files here for ingestion
│   ├── alumni-master.json  # Consolidated normalized alumni records (written by pipeline)
│   └── sample/
│       └── alumni-sample.csv   # 15-record sample for testing
└── output/
    ├── profiles/           # {alumni_id}.json — normalized profiles
    ├── segments/           # Segment definitions with member IDs
    ├── scores/             # Engagement scores (0-100) with trend data
    ├── prospects/          # Donor tiers and ranked prospect list
    ├── outreach/           # HTML email templates by segment + campaign type
    ├── events/             # Event invitation lists
    └── reports/            # Quarterly and monthly reports (markdown)
```

## Engagement Scoring Model

The engagement-scorer uses a 5-factor weighted model:
- Donation history: 35% (recency, frequency, amount tier)
- Event attendance: 25%
- Volunteer activity: 20%
- Email/digital engagement: 15%
- Network activity: 5%

Score → Status: 80-100=Highly Engaged, 60-79=Active, 40-59=Warm, 20-39=Lapsed, 0-19=Lost

## Donor Tiers

- **MG (Major Gift)**: score ≥ 70, total_giving ≥ $5K, employer known → capacity $100K+
- **ML (Mid-Level)**: score ≥ 55, total_giving ≥ $500 → capacity $10K-$100K
- **AF (Annual Fund)**: score ≥ 40, email known → capacity $1K-$10K
- **SO (Steward Only)**: donated ≥ $10K in last 18 months → cooldown, no asks

## Alumni Segments

Temporal: recent (0-5yr) / early_career (6-15yr) / mid_career (16-25yr) / senior (26+yr)
Degree: undergraduate / graduate / professional / phd
Geographic: local / regional / national / international
Career: corporate / nonprofit / entrepreneur / academia / government / unknown
Engagement: highly_engaged / active / lapsed / lost

## Outreach Templates

Templates use personalization tokens: `{{first_name}}`, `{{class_year}}`, `{{degree}}`
Each HTML file includes a subject line comment and plain-text alternative block.
Files named: `{segment}_{campaign}_{version}.html`

## Workflow Flow

1. `ingest-alumni-data` — normalize CSVs → output/profiles/ + data/alumni-master.json
2. `segment-alumni` — assign segments → output/segments/
3. `score-engagement` — calculate scores → output/scores/
4. `identify-donors` — tier prospects → output/prospects/
5. `generate-outreach` — write templates → output/outreach/
6. `review-outputs` — quality gate (reworks back to step 1 if missing outputs, max 2x)
7. `build-event-invitation-list` — Python command phase → output/events/
8. `generate-reports` — quarterly dashboards → output/reports/
9. `vp-advancement-review` — manual gate for VP Advancement sign-off

## Important Notes

- Drop CSV exports into `data/alumni-exports/` before running; sample data used if empty
- The `review-outputs` phase will rework the full pipeline (up to 2 times) if outputs are incomplete
- The `vp-advancement-review` manual gate requires human approval before outreach activates
- Memory MCP stores segment assignments under key `alumni_segments_{date}` for cross-run context
- SQLite DB path: `./data/alumni.db` (created automatically by sqlite MCP if needed)
- Weekly outreach schedule is disabled by default — enable in schedules.yaml during campaigns
