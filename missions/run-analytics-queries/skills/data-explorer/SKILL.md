---
name: End-to-end BigQuery query
description: >-
  Own schema drift, exploration, SQL approve, dry-run, execute, CSV export, and
  hosting ship on one child lane; return a compact terminal summary to the Squad
  Leader.
designation:
  allowed: >-
    Detect/refresh schema cache; explore tables and saved SQL; author and run
    approved GoogleSQL; export CSV; ship tracked hosting analytics files via
    worktree/PR when required
  forbidden: >-
    Configure GCP IAM or service accounts; execute unapproved production
    mutations; rewrite center rules; call mission_control_propose_dispatch_resolution
inputs:
  projectId:
    type: string
    required: true
  datasetId:
    type: string
    required: true
  location:
    type: string
    required: true
  localPath:
    type: string
    required: true
  dataSchemaPath:
    type: string
    required: true
  queriesPath:
    type: string
    required: true
  resultsPath:
    type: string
    required: true
  serviceAccountEmail:
    type: string
    required: true
  gcloudConfigurationName:
    type: string
    required: true
  inquiry:
    type: string
    required: true
  hostingWorktreeHint:
    type: string
    required: false
    description: >-
      Optional absolute hosting WORKTREE_ROOT or open PR URL when this dispatch
      already has an in-flight hosting worktree for tracked analytics files.
timeoutMs: 3600000
warmUpRules:
  - .sedea/centers/bigquery-analytics/rules/00_bigquery-analytics.mdc
  - .sedea/centers/bigquery-analytics/missions/run-analytics-queries/plan.mdc
  - .sedea/centers/sedea/rules/6_git-commit-push-gate.mdc
  - .sedea/centers/sedea/rules/0_hosting-repo.mdc
  - .sedea/centers/sedea/rules/7_stacked-pr-worktree-naming.mdc
---

# End-to-end BigQuery query

## Orientation

Start by telling the user that they do not need to know the schema or formulate
perfect SQL. **This child lane owns the full cycle** through CSV export and any
required hosting-repo ship. The Squad Leader receives only a compact terminal
summary — keep execution detail on this lane.

**Forbidden:** resolving the dispatch; handing unfinished “query intent only”
back to the Squad Leader for dry-run/execute/ship.

## Steps

### 1. Verify scope and access

Activate `inputs.gcloudConfigurationName`. Confirm the active account is
`inputs.serviceAccountEmail`. Run metadata-only access probes against
`inputs.projectId:inputs.datasetId`. Do not switch projects/datasets silently.

### 2. Detect schema drift and refresh cache

Follow center rule `00_bigquery-analytics.mdc` § *Dataset schema cache* and the
fetch layout from **new-analytics-dataset-configuration** §6.

1. Resolve `inputs.dataSchemaPath` (already required; default shape
   `<localPath>/data-schema`).
2. Read local `_dataset.json` and inventory `tables/` / `views/` when present.
3. Fetch remote inventory/schema via metadata-only `bq` (`bq ls`,
   `bq show --schema`). Never print credentials.
4. Diff remote vs local (table/view set, column name/type/mode, view
   definitions when available). Treat missing/empty local cache for a
   non-empty remote dataset as drift.
5. If **no drift:** set `schemaDriftStatus: in-sync`; continue §3. Prefer the
   local cache in exploration.
6. If **drift** (or missing cache):

USER_CHECKPOINT — refresh schema now · skip refresh this run · more details.

   - On **refresh schema now:** rewrite schema artifacts (overwrite same
     binding). Set `schemaDriftStatus: refreshed`. Stage ship for §8 (bundle
     with query edits when tracked).
   - On **skip:** set `schemaDriftStatus: skipped`; continue §3 with
     best-available cache/live fallback per center rule; note staleness.
7. Do **not** open a second hosting worktree solely for schema when
   `inputs.hostingWorktreeHint` already names an active worktree for this pass.

### 3. Inventory saved queries

List readable `*.sql` files under `inputs.queriesPath`. For each, show filename
and a short purpose inferred from leading comments/query shape. Do not execute a
saved query merely to classify it.

USER_CHECKPOINT — inspect a saved query, continue dataset discovery, or describe
a new question.

### 4. List tables and views

Prefer the local schema cache from §2. Use `bq ls` only when the cache is
insufficient or the user asks for live inventory. Present concise table/view
names and types. When the list is long, page or group it.

USER_CHECKPOINT — inspect schemas for selected tables, ask what data exists,
preview a table, search by topic/name, or inspect saved queries.

### 5. Explain schemas

For selected tables/views, read cache JSON first; fall back to
`bq show --schema --format=prettyjson` only when needed. Summarize:

- field names and types;
- repeated/nested fields;
- likely keys and join candidates (clearly marked as inference);
- partitioning/clustering;
- freshness metadata when available.

USER_CHECKPOINT — compare tables, preview rows, formulate query ideas, inspect
another schema, or return to table list.

### 6. Preview data safely

Offer a bounded preview; default `LIMIT 10`. Use fully qualified identifiers and
Standard SQL. Before a preview that may scan substantial data, run a dry-run and
show estimated bytes.

Never expose obviously sensitive columns unnecessarily. If sensitive data may
be present, prefer a column-limited preview and ask before displaying values.

USER_CHECKPOINT — run bounded preview, change LIMIT/columns, skip preview, or
inspect another table.

### 7. Choose query direction

Based on metadata, previews, saved queries, and `inputs.inquiry`, offer concrete
inquiry directions (entities, time coverage, joins, quality, trends, reuse).
These are educational prompts, not claims about business meaning.

USER_CHECKPOINT — choose a query direction, keep exploring, inspect saved SQL,
or abandon.

### 8. Draft, approve, and save GoogleSQL

Draft:

- a kebab-case query slug;
- a one-sentence purpose;
- parameter/default assumptions;
- fully qualified table references;
- bounded date/filter scope where appropriate;
- deterministic ordering when meaningful;
- LIMIT policy for the production run.

USER_CHECKPOINT — approve query, revise query, reuse an existing saved query,
or abandon.

Save the approved body as `<queriesPath>/<query-slug>.sql`. Set
`developerApprovedQuery: true` only after this gate. If the SQL file is
intentionally local under a gitignored workspace, state that and plan
`shipStatus: skipped-local` for §11. If tracked, include it in §11 ship.

### 9. Cost/safety preflight (dry-run)

Run `bq query --use_legacy_sql=false --dry_run` against the **exact** saved
file. Do not run mutations unless explicitly requested and approved.

USER_CHECKPOINT — run query, revise SQL, add/adjust LIMIT or filters, or abort.

### 10. Execute and export CSV

For a safely sized result, run the exact saved SQL with CSV output. For a large
result, materialize then `bq extract`. Write only under `inputs.resultsPath`,
using a timestamped or approved filename. Never overwrite an existing CSV
silently.

Capture for the terminal summary:

- `sqlPath`, `csvPath`;
- data row count (header excluded);
- byte size;
- job id and bytes processed when available;
- whether LIMIT/truncation/materialization applied.

### 11. Hosting-repo ship (when tracked)

When §2 refreshed tracked `data-schema/` and/or §8 saved tracked SQL under the
hosting repo:

1. Reuse `inputs.hostingWorktreeHint` when it is an absolute active
   `WORKTREE_ROOT` for this pass; otherwise run Hosting-repo ship cadence
   (center `worktree-setup.sh` → MCP `sedea_add_worktree_folder` → edits under
   `WORKTREE_ROOT`) per Sedea rules **0** / **6** / **7**.
2. Stage **named paths only** — never `git add .`.
3. Pipeline depth via structured choice (commit / commit+push / commit+push+PR)
   per rule **6**.
4. After PR open, run inline `pr-review` when the hosting cadence requires it;
   prefer developer merge (`merged-pr-proceed`) unless same-turn
   `approve-merge-pr` (or equivalent) is selected.
5. Post-merge: MCP `sedea_remove_worktree_folder` then center
   `worktree-cleanup.sh` **only** for Path A–owned `WORKTREE_ROOT`.
6. Set `shipStatus` / `schemaShipStatus` / `prUrl` / `prNumber` accordingly.

When nothing is hosting-tracked, set `shipStatus: skipped-local` and
`schemaShipStatus: none` or `skipped` as appropriate — do not invent git work.

### 12. Complete (terminal)

Call `mission_control_send_agent_result` with compact `outputs` below. Do **not**
resolve the dispatch. Do **not** dump full SQL, full CSV, or step-by-step tool
logs into `summary` — keep `summary` to 1–3 sentences for the Squad Leader
ledger.

## Completion (spawned)

### Host protocol line

Call MCP `mission_control_send_agent_result` with `status`, `summary`,
`outputs`, and `errors` (`[]` when none). **Forbidden in args:**
`correlationId`, `dispatchId`, `slotId`, and other host-resolved identity keys.

### Outputs

| Field | Type |
|-------|------|
| `projectId` | string |
| `datasetId` | string |
| `querySlug` | string |
| `queryPurpose` | string |
| `sqlPath` | string |
| `csvPath` | string |
| `rowCount` | number |
| `bytesProcessed` | number or null |
| `jobId` | string or empty |
| `schemaDriftStatus` | `in-sync` \| `refreshed` \| `skipped` \| `failed` |
| `schemaShipStatus` | `none` \| `bundled-in-pr` \| `separate-pr` \| `skipped` |
| `shipStatus` | `none` \| `opened` \| `merged` \| `skipped-local` \| `failed` |
| `prUrl` | string or empty |
| `prNumber` | number or null |
| `developerApprovedQuery` | boolean |
| `continuationStatus` | `terminal` |
| `relevantTables` | array |
| `openAssumptions` | array |

Use `errors: []` when none. Omit credential and token material.

## Completion (inline)

Same semantic fields as spawned `outputs`, reported in prose to the invoker.
No MCP spawn/result tools.
