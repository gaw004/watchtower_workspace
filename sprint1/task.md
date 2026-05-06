# Sprint 1 Deliverables: Storage ADR & Cloudflare Spike

Concrete deliverables for both tasks, with what each file should contain, what "done" looks like, and what to skip.

---

## Task A — Storage Selection ADR (Gabrielle)

### File to produce

**One file:** `docs/backend/adr/0001-storage-selection.md`

That's it. No code, no diagrams beyond what fits in markdown, no separate research doc. The ADR *is* the deliverable.

### Required structure (MADR format)

MADR is a specific markdown template for ADRs. Use these section headings exactly — graders and future maintainers expect them.

```markdown
# 0001 — Storage Selection for WatchTower Backend

- Status: Proposed
- Date: 2026-05-XX
- Deciders: Backend team (Theo, Michael, Bishal, Gabrielle)
- Consulted: TA Audria (pending)
- Informed: Full WatchTower team

## Context and Problem Statement

## Decision Drivers

## Considered Options

## Decision Outcome

### Consequences

### Confirmation

## Pros and Cons of the Options

### Option 1: Cloudflare D1
### Option 2: Cloudflare KV
### Option 3: Cloudflare R2
### Option 4: Cloudflare Durable Objects

## Migration Cost Analysis

## More Information
```

### What goes in each section

**Context and Problem Statement** (1 short paragraph)
Explain that WatchTower needs to store three kinds of data — events (errors, performance samples, feedback), releases, and projects — and that the backend will run on Cloudflare Workers per project constraints. State the question being decided: *which Cloudflare storage product should hold this data?*

**Decision Drivers** (a bulleted list of 4–7 items)
The criteria you're judging the options against. Things like:
- Support for filtered/sorted queries (the dashboard needs "errors in the last 7 days for project X, grouped by message")
- Free-tier capacity (the team has no budget)
- Write throughput (one busy customer site could send thousands of events per minute)
- Migration cost out of the chosen product
- Schema flexibility (will the event shape evolve?)
- Operational simplicity for a 4-person team that has never run a database before

These drivers are the *yardstick*. Every option below gets measured against them.

**Considered Options** (a short list)
Just the names: D1, KV, R2, Durable Objects. One line each.

**Decision Outcome** (2–3 sentences)
"Chosen option: **Cloudflare D1**, because it is the only option that natively supports the filtered, sorted, time-range queries the dashboard requires, and its free tier comfortably covers WatchTower's expected volume during the academic project."

You don't need this exact wording — but it should be that decisive. ADRs are not "we'll see." They commit.

**Consequences** (split into Positive / Negative / Neutral)
- Positive: SQL queries are familiar to anyone who's taken a database course; D1 has built-in time-series indexing; integrates directly with Workers without extra config.
- Negative: D1 has row-count and storage limits on the free tier; harder to migrate out of than a portable SQL dump (D1 has its own backup format); D1 is still relatively new (released 2023), so community examples are thinner than for, say, Postgres.
- Neutral: locks the team into Cloudflare for the duration of the project (this is already required by project constraints, so not a real cost here).

**Confirmation** (1–2 sentences)
How will you know the decision was correct? Examples: "If the spike (`spike/cloudflare-bootstrap`) successfully writes and reads from D1 within free-tier limits, and if the dashboard's required queries (see ADR-0002 event data model) execute in under 200ms with seeded test data, the decision is confirmed."

**Pros and Cons of the Options** (one subsection per option)
For each of the four candidates, write **3–5 pros and 3–5 cons**, focused on the decision drivers above. Don't dump generic Cloudflare marketing copy — tie every pro/con back to a driver.

Bad: "D1 is fast."
Good: "D1 supports SQL `WHERE` and `ORDER BY` natively, which directly serves the dashboard's date-range and grouping queries (driver: filtered queries)."

For each option, also note the **free-tier limits** in concrete numbers (look these up — they change). Things like "D1 free tier: 5GB storage, 5M reads/day, 100K writes/day as of May 2026."

**Migration Cost Analysis** (a short paragraph or small table)
This is the section I specifically called out earlier. For each option, briefly say what it would take to switch *out* of it later:
- D1 → Postgres: SQL dump, schema translation (mostly compatible since D1 is SQLite-flavored), code changes to swap the D1 binding for a SQL driver. Estimated: 1–2 sprints.
- KV → anything: data is unstructured; would require a migration script that interprets keys and rebuilds rows. Estimated: 2–3 sprints.
- R2 → anything: object storage isn't really comparable to a database; if we picked R2 we'd already have built our own query layer, which would have to be replaced. Estimated: 3+ sprints.
- Durable Objects → anything: data is sharded across object instances; consolidation requires custom export tooling. Estimated: 2+ sprints.

The point isn't precision — it's showing you've thought about the lock-in. This protects the team in retrospectives ("why did we pick D1?" → "see ADR-0001, migration analysis").

**More Information** (links)
A handful of links to Cloudflare's docs for each option, plus any blog posts or comparisons Gabrielle leaned on.

### What "done" looks like for Task A

- The file exists at `docs/backend/adr/0001-storage-selection.md`
- Status is "Proposed" (not "Accepted" yet — wait for backend team review)
- All sections above are filled in, with no placeholder text
- Free-tier numbers are real and dated (look them up; don't guess)
- A pull request is opened and at least one other backend team member has commented or approved
- The ADR is referenced from the backend README (one line: "Major decisions are tracked in `docs/backend/adr/`")

### What to skip

- Code samples — this is a decision document, not a tutorial
- Comparisons to non-Cloudflare products (Postgres, MongoDB, Firebase). Project constraints rule them out, so listing them just clutters the doc.
- Diagrams — not needed; the data flow is simple enough that prose suffices.

### Estimated effort

Half a day to a full day, mostly research and writing. The decision is probably going to be D1; the value is in the documented reasoning, not the surprise.

---

## Task B — Cloudflare Spike (Theo or Gabrielle)

### Files to produce

**Two things, in two locations.**

**Location 1 — a scratch repo or personal branch (NOT the main repo's `backend/`):**

A folder containing:

```
spike-cloudflare-bootstrap/
├── wrangler.toml
├── package.json
├── src/
│   └── index.js          ← the Worker code
├── schema.sql            ← the D1 table definition
└── README.md             ← how to run the spike
```

This is the **throwaway code**. It exists only to prove the platform works. Nobody is going to merge it. It just needs to run.

**Location 2 — the main repo's docs folder:**

`docs/backend/research/cloudflare-spike.md`

This is the **knowledge artifact**. It survives the spike and gets read by whoever sets up the real `backend/` in Sprint 2.

### What the spike code should do

The Worker should expose two endpoints, just enough to prove end-to-end connectivity:

**`GET /health`** — returns `{"status": "ok", "time": "<ISO timestamp>"}`. This proves the Worker deploys and responds.

**`POST /test-event`** — accepts a JSON body, writes one row to a D1 table called `test_events`, and returns the inserted row's ID. This proves the Worker can write to D1.

**`GET /test-event`** — reads the most recent 10 rows from `test_events` and returns them as JSON. This proves the Worker can read from D1.

That's all. Three endpoints, maybe 30–50 lines of JavaScript total. **Resist the urge to add validation, error tracking, rate limiting, the real schema, or anything else.** Those are Sprint 2+ work and should not pollute the spike.

The `schema.sql` file contains exactly:

```sql
CREATE TABLE test_events (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  payload TEXT NOT NULL,
  created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

No real columns from the data model. This is a `test_events` table, deliberately named that way so nobody confuses it with the real schema later.

### What the spike's local README should cover

A short `README.md` in the spike folder, mostly so the spike author can re-run it themselves a week later:

- One-line description: "Spike to verify Cloudflare Workers + D1 deployment for WatchTower."
- How to install dependencies (`npm install`)
- How to authenticate wrangler (`npx wrangler login`)
- How to create the D1 database (`npx wrangler d1 create watchtower-spike`)
- How to apply the schema (`npx wrangler d1 execute watchtower-spike --file=schema.sql`)
- How to run locally (`npx wrangler dev`)
- How to deploy (`npx wrangler deploy`)
- How to test the three endpoints (curl examples)

Treat this README as a verification log — it should match what was actually done, not what the docs say should work.

### What the knowledge artifact should contain

This is the more important deliverable. The file `docs/backend/research/cloudflare-spike.md` is what the team reads in Sprint 2. Suggested structure:

```markdown
# Cloudflare Spike: Workers + D1 Bootstrap

**Date:** 2026-05-XX
**Author:** <name>
**Spike repo / branch:** <link>
**Goal:** Verify that we can deploy a Cloudflare Worker that reads and writes
to D1 on a free-tier account, end-to-end, before committing to D1 in ADR-0001.

## Outcome

✅ / ❌  Worker deploys
✅ / ❌  D1 database creation succeeds
✅ / ❌  Worker can write to D1
✅ / ❌  Worker can read from D1
✅ / ❌  Stayed within free-tier limits during testing

## Time taken

Total wall-clock time from "empty Cloudflare account" to "all three endpoints
working in production": <X hours>.

Breakdown:
- Account creation and team invite: <X min>
- wrangler install and login: <X min>
- First Worker deploy: <X min>
- D1 database creation: <X min>
- Wiring the Worker to D1 (binding config): <X min>
- First successful round-trip POST → SELECT: <X min>

## What worked smoothly

- ...
- ...

## What broke and how it got fixed

For each issue:
- **Symptom:** what error message or behavior I saw
- **Cause:** what was actually wrong
- **Fix:** what I did to resolve it
- **Time lost:** rough estimate

## Free-tier limits observed

- Workers: <requests/day limit>
- D1: <rows / reads / writes / storage limits>
- Any limits I came close to hitting during the spike

## Recommendations for the real backend setup

Things Bishal (or whoever does Sprint 2's environment setup) should know:

1. ...
2. ...
3. ...

## Open questions

Things the spike couldn't answer and that the real implementation will need to
resolve:

- ...

## Verdict

Does the spike confirm D1 is viable for ADR-0001? <Yes / No / With caveats>
```

The "What broke and how it got fixed" section is the highest-value part. Every Cloudflare gotcha you save Bishal from is hours back to the team in Sprint 2.

### What "done" looks like for Task B

- The spike Worker is deployed and the three endpoints respond correctly when called via curl from outside Cloudflare
- The D1 database has at least 5 test rows that were written through the Worker
- `docs/backend/research/cloudflare-spike.md` is committed to the main repo with all sections filled in
- The spike repo or branch is linked from the knowledge artifact so future readers can find the actual code
- The verdict at the bottom is decisive (yes/no/caveats), and feeds into Gabrielle's ADR-0001 confirmation section

### What to skip

- Tests. This is throwaway code. Tests on the spike would just delay the spike.
- Linting, formatting tools, CI. None of that. Spike code doesn't go through CI.
- Real validation, real error handling, real auth. The spike isn't trying to be safe; it's trying to be honest about whether the platform works.
- Pretty UI, fancy responses, multiple environments. One Worker, one database, one environment.
- Building the real schema (`events`, `releases`, `projects`). Use `test_events` only — keeps the spike obviously disposable.

### Estimated effort

Half a day to a full day for someone who has never used Cloudflare before. Faster if they've used it. The hard part is usually account setup, team invites, and the wrangler auth flow — not the code.

---

## How A and B feed each other

Gabrielle's ADR (Task A) recommends D1. The spike (Task B) verifies that D1 actually works in your team's hands. **The spike's verdict goes into the ADR's "Confirmation" section.** This is why I suggested Gabrielle and Theo sync mid-week — once the spike returns "yes, D1 works," the ADR can move from "Proposed" to "Accepted" and the team can commit.

If the spike returns "no, D1 doesn't work for us" or "yes, but with caveats X and Y," the ADR adapts. That's the whole point of doing the spike before locking the decision.

Both deliverables are due by end of Thursday so the team has Friday for review and the standup on Sunday can close out the sprint cleanly.