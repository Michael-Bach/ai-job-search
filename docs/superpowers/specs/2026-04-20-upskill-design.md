# `/upskill` Skill — Design Spec

**Date:** 2026-04-20
**Status:** Approved

---

## Overview

`/upskill` compares tracked job postings against the candidate's current profile, identifies skill gaps, and produces a prioritized learning plan with concrete, web-searched study resources and a suggested study order.

---

## Invocation Modes

### Aggregate mode — `/upskill`
Analyzes all jobs in `job_search_tracker.csv`. Gaps are weighted by:
- **Fit rating** — lower fit jobs contribute more weight (those are the roles where the profile fell short)
- **Frequency** — skills appearing across many postings matter more

Report saved to: `upskill/report-YYYY-MM-DD.md`

On subsequent runs, a **"Since last report"** diff section is prepended:
- Gaps closed (skills added to profile since last report)
- New gaps appeared (from newly tracked jobs)

### Targeted mode — `/upskill <URL>`
Fetches a single job posting via WebFetch and runs the full analysis against only that role. Ignores the tracker entirely. Still saves a report named:
`upskill/report-YYYY-MM-DD-<company>-<role>.md`

---

## Gap Analysis Pipeline

### Pass 1 — Hard skill diff
1. Extract required and preferred skills from each job source
2. Build a frequency-weighted skill list
3. Subtract skills already present in the candidate profile (`01-candidate-profile.md` + `CLAUDE.md`)
4. Output: ranked list of missing hard skills with a score = `frequency × fit-weight`

### Pass 2 — LLM synthesis
Feed job data + candidate profile + Pass 1 results to Claude. Ask it to surface gaps Pass 1 missed:
- Domain knowledge
- Soft skills
- Ways of working
- Industry context

Tag these as `[soft]` or `[domain]` to distinguish from hard skill gaps.

### Heatmap output
Combines both passes into a tiered table:

| Priority | Skill/Area | Type | Gap Source |
|----------|------------|------|------------|
| Critical | Kubernetes | Hard | 4/5 jobs |
| High | Security domain knowledge | Domain | LLM synthesis |
| Medium | ... | ... | ... |

Priority tiers: **Critical**, **High**, **Medium** (Low gaps are listed but excluded from the learning plan by default).

---

## Learning Plan

For each Critical and High priority gap (Medium included if few gaps):

1. **Live WebSearch** for current top-rated resources (e.g. `"best Kubernetes course 2025 site:reddit.com OR coursera.org OR fast.ai"`)
2. **2-3 concrete resources** per topic — named course, book, or docs link — with a one-line reason each
3. **Study direction** tailored to the candidate's existing background (e.g. "you know Docker, skip containers basics, go straight to orchestration")
4. **Rough time estimate** to working proficiency (e.g. "~20h")

Grouped by theme (e.g. Cloud, MLOps, Domain Knowledge, Soft Skills), not alphabetically.

### Suggested Study Order
A numbered sequence at the end of the report:
- Respects skill dependencies (learn foundational topics before advanced ones)
- Prioritizes Critical gaps first
- Includes cumulative time estimates so the full roadmap is visible at a glance

---

## Report Format

```markdown
# Upskill Report — YYYY-MM-DD
[Mode: Aggregate | Targeted: <Job Title> @ <Company>]

## Since Last Report (aggregate mode only)
- Gaps closed: ...
- New gaps: ...

## Gap Heatmap
| Priority | Skill/Area | Type | Gap Source |
...

## Learning Plan

### Cloud
**Kubernetes** (~20h)
- [Resource 1] — reason
- [Resource 2] — reason
Study direction: ...

### MLOps
...

## Suggested Study Order
1. Kubernetes (~20h) — foundational for deployment gaps
2. ...

Total estimated time: Xh
```

---

## Files

| Path | Purpose |
|------|---------|
| `job_search_tracker.csv` | Source of tracked jobs (aggregate mode) |
| `.claude/skills/job-application-assistant/01-candidate-profile.md` | Current skills to diff against |
| `upskill/report-YYYY-MM-DD.md` | Saved aggregate reports |
| `upskill/report-YYYY-MM-DD-<company>-<role>.md` | Saved targeted reports |

---

## Constraints

- Never fabricate job requirements or resources — all data from actual fetched content or WebSearch results
- WebSearch for resources must target current results (include year in query)
- Targeted mode must fetch the URL fresh, not rely on seen_jobs.json
- Diff section only appears in aggregate mode and only if a previous report exists
