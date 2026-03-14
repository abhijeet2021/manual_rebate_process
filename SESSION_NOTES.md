# Session Notes — Manual Rebate Process Automation

**Date:** 2026-03-14
**Repo:** [abhijeet2021/manual_rebate_process](https://github.com/abhijeet2021/manual_rebate_process)
**Live Dashboard:** [https://abhijeet2021.github.io/manual_rebate_process/rebate_workflow_dashboard.html](https://abhijeet2021.github.io/manual_rebate_process/rebate_workflow_dashboard.html)

---

## What We Built

A **standalone interactive HTML dashboard** (`rebate_workflow_dashboard.html`) that fully documents the `manual_rebate_process` n8n workflow — designed for both CTO-level walkthroughs and technical KT/handover sessions.

---

## Source Workflow

**File:** `ref/manual_rebate_process.json`
**n8n Workflow ID:** `IYcXoxa3opDQHw1dZYZQU`
**Status:** Active

---

## Dashboard Sections

| Tab | Audience | Content |
|-----|----------|---------|
| **Overview** | CTO / Business | Pipeline stages, tech stack, domain types, key infrastructure |
| **Flow Diagram** | All | Clickable interactive node map — click any node for technical details |
| **Validation Rules** | Tech / QA | All 13 validation rules across 3 phases (JS → SQL → post-enrichment) |
| **Data Architecture** | Tech / DBA | Input schema, BigQuery temp table schema, all source tables |
| **KT / Runbook** | New engineers | How to trigger, upload files, read reports, query BQ, maintain |
| **Error Codes** | Support / Ops | Every error code with root cause + fix steps |

---

## Workflow Summary

### Trigger Options
- **Flock command:** Send `"process rebates"` to the connected Flock channel (token-authenticated via webhook)
- **Manual:** Click "Execute Workflow" in the n8n editor

### Pipeline Stages
1. **Trigger** → Webhook or Manual → Flock start notification
2. **File Ingestion** → List GCS bucket `manual_rebate_process`, filter `.xlsx`/`.csv`, download
3. **Data Cleaning** (JavaScript) → Normalize domains, parse dates, deduplicate, standardize domain types
4. **BigQuery Load** → TRUNCATE + INSERT into `rebate_process_temp.rebate_cleaned_data`
5. **Validation** → 11 SQL UPDATE rules flag errors; auto-upgrade `new` → `premium new` / `renew` → `premium renew`
6. **Domain Enrichment** → Lookup `domain_id`, `createrenewdate`, `registrar_shortname` from registry tables
7. **Post-Enrichment Validation** → 180-day limit, deleted/restored domain checks
8. **Report** → BigQuery summary query → Flock final notification

### Key Infrastructure
| Item | Value |
|------|-------|
| GCS Bucket | `manual_rebate_process` |
| BQ Project | `radixbi-249015` |
| Temp Table | `rebate_process_temp.rebate_cleaned_data` |
| Webhook Path | `POST /process-rebates` |
| BQ Source Tables | `radix_data.newreg`, `radix_data.renews`, `radix_data.premium_newreg`, `radix_data.premium_renews` |
| Deleted/Restore Check | `cnic_dump.deleted`, `cnic_dump.restore` |

---

## Validation Rules Reference

### Phase 1 — JavaScript (pre-load)
| Code | Description |
|------|-------------|
| *(silent drop)* | Missing/invalid domain — row skipped before insert |
| *(silent drop)* | Duplicate domain within file — first occurrence kept |

### Phase 2 — SQL (BigQuery temp table)
| Error Code | Trigger |
|------------|---------|
| `INCORRECT_DOMAIN_TYPE` | domain_type not in allowed list |
| `INVALID_DOMAIN_NAME` | `NET.REG_DOMAIN()` returns NULL |
| `DUPLICATE_ENTRY` | Same domain submitted more than once |
| `MISSING_DOMAIN` | domain is NULL or empty |
| `MISSING_DOMAIN_TYPE` | domain_type is NULL or empty |
| `MISSING_DATE` | date is NULL |
| `MISSING_CLIENT` | client is NULL or empty |
| `MISSING_MBG_FOR_NEW_DOMAIN` | domain in newreg with mbg=NULL |
| `MISSING_MBG_FOR_RENEW_DOMAIN` | domain in renews with mbg=NULL |
| `MISSING_MBG_FOR_PREMIUM_NEW_DOMAIN` | domain in premium_newreg with mbg=NULL |
| `MISSING_MBG_FOR_PREMIUM_RENEW_DOMAIN` | domain in premium_renews with mbg=NULL |

### Phase 3 — Post-Enrichment
| Error Code | Trigger |
|------------|---------|
| `DATE_EXCEEDS_180_DAY_LIMIT` | createrenewdate > 180 days old |
| `DOMAIN_MARKED_AS_DELETED` | domain_id NULL + found in cnic_dump.deleted |
| `DOMAIN_FOUND_IN_RESTORE` | domain_id NULL + found in cnic_dump.restore |
| `DOMAIN_NOT_FOUND_FOR_DOMAIN_TYPE` | domain_id still NULL after all lookups |

---

## Git Setup

```bash
# Clone
git clone https://github.com/abhijeet2021/manual_rebate_process.git

# Files
rebate_workflow_dashboard.html   # Standalone interactive dashboard (open in browser)
ref/manual_rebate_process.json   # Source n8n workflow JSON
SESSION_NOTES.md                 # This file
.gitignore                       # Excludes .claude/, OS, editor files
```

---

## GitHub Pages

Dashboard is published via GitHub Pages:
- **URL:** `https://abhijeet2021.github.io/manual_rebate_process/rebate_workflow_dashboard.html`
- **Branch:** `master` / root
- **Visibility:** Public — shareable with anyone, no login required

---

## Notes for Future Sessions

- The temp table **is truncated on every run** — export data before re-triggering if needed
- All `.xlsx`/`.csv` files in the GCS bucket are processed each run — remove old files to avoid reprocessing
- To add a new domain type: update (1) `cleanDomainType()` in JS, (2) INCORRECT_DOMAIN_TYPE allowlist in SQL, (3) new UPDATE block in `find domain in system` BQ node
- Webhook token and Flock URL are hardcoded in n8n nodes — rotate via n8n editor if needed
