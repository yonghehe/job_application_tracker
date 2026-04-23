# Job Tracker Schema — Application Second Brain

This file governs every interaction in this vault. Read it at the start of every session.

---

## Context

**Domain:** Job Applications  
**Purpose:** Track every job applied for, synthesise job listing information, and maintain an Obsidian graph of applications clustered by sector and company.  
**Human role:** Paste or drop Webclipper `.md` clips into `raw/clips/`, direct operations, update statuses.  
**Agent role:** All parsing, writing, CSV updates, graph node maintenance, and synthesis.

---

## Directory Structure

```
job-tracker/
├── CLAUDE.md                    ← This file. Schema and rules.
├── dashboard.md                 ← Landing page. Summary stats, recent activity, status breakdown.
├── log.md                       ← Append-only chronological log.
├── tracker.csv                  ← Master application tracker. One row per application.
├── raw/
│   └── clips/                   ← IMMUTABLE Webclipper .md files. Never modify.
├── applied/
│   ├── sectors/                 ← Sector node .md files. One per inferred sector.
│   ├── companies/               ← Company node .md files. One per company.
│   └── jobs/                    ← Individual job listing .md files. One per application.
└── outputs/
    └── reports/                 ← Lint and audit reports.
```

---

## Graph Architecture (Obsidian)

The graph forms a three-level tree, viewable using Obsidian's Graph View with "Group by tag" or by filtering on node type.

```
[Sector Node]
    └── [[Company Node]]
            └── [[Job Title Node]]   ← terminal node
```

- **Sector nodes** (`applied/sectors/`) are the root cluster nodes. They link to nothing.
- **Company nodes** (`applied/companies/`) link to their parent sector node.
- **Job nodes** (`applied/jobs/`) link to their parent company node. They are terminal: no outgoing links.

This means every job application is reachable from a sector node in exactly two hops, which produces clean sector-based clusters in the Obsidian graph.

### Sector Merging

Each company node stores its `sector:` in frontmatter as plain text. If two sector nodes need to be merged later (e.g., "Biotech" and "Life Sciences" → "Life Sciences"), the operation is:
1. Rename/delete the old sector `.md` file.
2. Run a find-and-replace across `applied/companies/` to update the `sector:` field and wikilink.
3. Update the `sector` column in `tracker.csv`.
4. Update `applied/sectors/` — add merged companies as outgoing links in the surviving sector node.

---

## tracker.csv Schema

One row per job application. Columns (in order):

| Column | Notes |
|---|---|
| `id` | Auto-incrementing integer. e.g. `001`, `002` |
| `company` | Company or agency name |
| `role` | Exact job title from listing |
| `sector` | Agent-inferred sector label |
| `date_applied` | YYYY-MM-DD |
| `status` | See Status Stages below |
| `location` | City / country, or "Remote" |
| `contract_type` | Permanent, Contract, Internship, Part-time |
| `salary_range` | As listed, or `Undisclosed` |
| `closing_date` | YYYY-MM-DD, or blank if not stated |
| `source_url` | Original job listing URL from clip frontmatter |
| `job_file` | Relative path to job `.md` file, e.g. `applied/jobs/hsa-laboratory-officer.md` |
| `notes` | Free text. Interview dates, recruiter name, anything relevant |

### Status Stages

Valid values for the `status` column:

```
Applied → Screen → Interview → Offer → Accepted
                                     → Rejected
              → Rejected
         → Ghosted
         → Withdrawn
```

---

## File Naming Conventions

- All files: lowercase, hyphens for spaces.
- Sector nodes: sector label as filename. e.g. `healthcare.md`, `agri-food-tech.md`
- Company nodes: company short name. e.g. `health-sciences-authority.md`, `temasek.md`
- Job nodes: `<company-short>-<role-slug>.md`. e.g. `hsa-laboratory-officer.md`, `dbs-data-analyst.md`
- Clips (raw): preserve the Webclipper filename exactly. Never rename.
- Reports: `lint-YYYY-MM-DD.md`

---

## Frontmatter Schemas

### Sector node (`applied/sectors/`)
```yaml
---
title: "Healthcare"
type: sector
node_level: root
tags: [sector]
created: YYYY-MM-DD
updated: YYYY-MM-DD
companies: ["[[Health Sciences Authority]]", "[[Raffles Medical]]"]
---
```
Body: 2–3 sentence description of the sector. List of linked company nodes using wikilinks.

---

### Company node (`applied/companies/`)
```yaml
---
title: "Health Sciences Authority"
type: company
node_level: branch
sector: "[[Healthcare]]"
tags: [company, healthcare]
created: YYYY-MM-DD
updated: YYYY-MM-DD
applications: ["[[Laboratory Officer, Pharmaceutical Laboratory]]"]
---
```
Body: 2–3 sentence overview of the company (sourced from the "About" section of the clip). List of linked job nodes.

---

### Job node (`applied/jobs/`)
```yaml
---
title: "Laboratory Officer, Pharmaceutical Laboratory"
type: job
node_level: terminal
company: "[[Health Sciences Authority]]"
sector: "[[Healthcare]]"
tags: [job, healthcare, contract]
date_applied: YYYY-MM-DD
status: Applied
location: "Singapore"
contract_type: "Contract"
salary_range: "Undisclosed"
closing_date: "2026-05-04"
source_url: "https://jobs.careers.gov.sg/..."
tracker_id: "001"
---
```
Body: structured sections (see Compilation Standards below). No outgoing wikilinks in the body — job nodes are terminal.

---

### Lint report (`outputs/reports/`)
```yaml
---
title: "Lint Report YYYY-MM-DD"
type: output
stage: lint-report
created: YYYY-MM-DD
issues_found: N
issues_fixed: N
---
```

---

## Workflows

### INGEST — Processing a new Webclipper clip

When the user says "ingest [filename]" or drops a `.md` file into `raw/clips/`:

1. **Read** the clip file. Extract from frontmatter: `title`, `source`, `published`, `created`. Extract from body: role description, responsibilities, requirements, contract details, closing date, company "About" section.

2. **Infer sector** from the job title, company name, and role description. Use a concise label (e.g., "Healthcare", "Finance", "Agri-Food Tech", "Software Engineering", "Public Sector"). When uncertain, choose the most specific accurate label rather than a generic one.

3. **Check for duplicates**: before creating any file, check `tracker.csv` and `applied/jobs/` for an existing entry with the same company + role. If a duplicate exists, report it to the user and stop.

4. **Update `tracker.csv`**: add a new row. Auto-increment the `id`. Set `status` to `Applied`. Set `date_applied` to today's date unless the user specifies otherwise. Set `salary_range` to the listed salary range, or `Undisclosed` if none is stated.

5. **Write the job node** at `applied/jobs/<company-short>-<role-slug>.md` following the job node schema and Compilation Standards below.

6. **Update or create the company node** at `applied/companies/<company-slug>.md`:
   - If the company file exists: append the new job title as a wikilink under `applications:` in frontmatter and in the body list.
   - If new: create the file using the company "About" section from the clip as the body overview.

7. **Update or create the sector node** at `applied/sectors/<sector-slug>.md`:
   - If the sector file exists: append the company (if not already listed) under `companies:` in frontmatter and body.
   - If new: create the file with a short sector description and the company as the first entry.

8. **Update `dashboard.md`**: increment total applications count, update last-activity date, update status breakdown counts.

9. **Append to `log.md`**: `## [YYYY-MM-DD] ingest | <Role Title> at <Company> (Sector: <Sector>)`

10. Report back: confirm files written, confirm CSV row added, state the inferred sector, flag any fields that were `Undisclosed` or missing.

---

### STATUS — Updating an application status

When the user says "update status [company] [role] to [new status]":

1. Find the matching row in `tracker.csv` by company + role.
2. Update the `status` column to the new value. Validate it is one of: `Applied`, `Screen`, `Interview`, `Offer`, `Accepted`, `Rejected`, `Withdrawn`, `Ghosted`.
3. Update the `status:` field in the corresponding job node frontmatter.
4. If the new status is `Interview`, prompt the user: "Add interview date to the notes field?"
5. Update `dashboard.md` status breakdown.
6. Append to `log.md`: `## [YYYY-MM-DD] status | <Role> at <Company>: <Old Status> → <New Status>`

---

### QUERY — Searching or filtering applications

When the user asks a question about their applications (e.g., "show me all roles in Healthcare", "how many applications are ghosted", "which roles close this week"):

1. Read `tracker.csv` as the primary data source.
2. Answer the question directly with a formatted table or summary.
3. Offer to save the result as a report under `outputs/reports/` if it is non-trivial.
4. Append to `log.md`: `## [YYYY-MM-DD] query | <Question summary>`

---

### LINT — Health check

When the user says "lint" or "health check":

Check for the following issues:

1. **CSV vs file mismatch**: every row in `tracker.csv` should have a corresponding job `.md` file at the path in `job_file`. Flag missing files.
2. **Orphan job nodes**: job `.md` files in `applied/jobs/` with no corresponding CSV row.
3. **Broken wikilinks**: company nodes referencing sector nodes that do not exist; job nodes referencing company nodes that do not exist.
4. **Missing company nodes**: a company referenced in a job node frontmatter has no file in `applied/companies/`.
5. **Missing sector nodes**: a sector referenced in a company node frontmatter has no file in `applied/sectors/`.
6. **Stale statuses**: applications with `status: Applied` and a `closing_date` in the past — flag as potentially ghosted.
7. **Format violations**: job nodes missing required frontmatter fields; nodes with wrong `node_level` values.

Save the full lint report to `outputs/reports/lint-YYYY-MM-DD.md` with sections:
```
## CSV vs File Mismatches
## Orphan Job Nodes
## Broken Wikilinks
## Missing Company Nodes
## Missing Sector Nodes
## Stale Statuses (possibly ghosted)
## Format Violations
```

Ask the user which issues to fix, then fix them. Append to `log.md`: `## [YYYY-MM-DD] lint | N issues found, N fixed`

---

### MERGE SECTORS — Combining two sector nodes

When the user says "merge sector [A] into [B]":

1. Read all company nodes currently pointing to sector A.
2. Update each company node: change `sector:` frontmatter field from `[[A]]` to `[[B]]`, update body wikilink.
3. Update each job node in those companies: change `sector:` frontmatter field.
4. Update `tracker.csv`: replace all values of `A` in the `sector` column with `B`.
5. Add all companies from A into sector B's `companies:` frontmatter list and body.
6. Delete the sector A `.md` file.
7. Update `dashboard.md`.
8. Append to `log.md`: `## [YYYY-MM-DD] merge-sector | [[A]] → [[B]] (N companies moved)`

---

## Compilation Standards — Job Node Body

Every job node body must contain the following sections, in order. Use information extracted from the Webclipper clip.

```markdown
## Role Summary
2–4 sentences. What the role is, the team or function it sits in, and why it was applied for.

## Responsibilities
Bullet list. Extracted directly from the listing's "What you will be working on" or equivalent section.

## Requirements
Bullet list. Extracted from "What we are looking for" or equivalent. Separate hard requirements from nice-to-haves if the listing distinguishes them.

## Contract Details
- **Contract Type:** Permanent / Contract / Internship / Part-time
- **Duration:** e.g. "1-year contract with option to renew" (or blank if not stated)
- **Salary:** As listed, or `Undisclosed`
- **Closing Date:** YYYY-MM-DD (or blank)
- **Location:** City / Remote

## Application Notes
Free text. Add any personal notes here: why you applied, how you found it, tailoring notes for cover letter or interview prep. Leave blank on ingest — this is for manual updates.
```

No outgoing wikilinks anywhere in the job node body. Job nodes are terminal graph nodes.

---

## dashboard.md Structure

Keep `dashboard.md` updated after every INGEST and STATUS operation.

```markdown
# Job Tracker Dashboard

**Last updated:** YYYY-MM-DD

## Summary
- Total applications: N
- Active (Applied / Screen / Interview): N
- Offers: N
- Closed (Rejected / Withdrawn / Ghosted / Accepted): N

## Status Breakdown
| Status | Count |
|---|---|
| Applied | N |
| Screen | N |
| Interview | N |
| Offer | N |
| Accepted | N |
| Rejected | N |
| Withdrawn | N |
| Ghosted | N |

## Sector Breakdown
| Sector | Applications |
|---|---|
| Healthcare | N |
| ... | N |

## Recent Activity
(Last 5 log entries, copied from log.md)
```

---

## log.md Op Types

Append-only. Format: `## [YYYY-MM-DD] <op> | <detail>`

Op types: `ingest`, `status`, `query`, `lint`, `merge-sector`, `update`

---

## Session Start Protocol

At the start of every new session:
1. Read `CLAUDE.md` (this file).
2. Read `dashboard.md` to load current tracker state.
3. Read the last 10 entries of `log.md`.
4. Greet the user with a one-line status: total applications, active pipeline count, any closing soon (within 7 days).
