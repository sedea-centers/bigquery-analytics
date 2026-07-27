---
name: End-to-end BigQuery query
description: >-
  Own schema drift, exploration, SQL approve, dry-run, execute, CSV export, and
  hosting ship on one child lane; return a compact terminal summary to the Squad
  Leader.
designation:
  allowed: >-
    Detect/refresh schema cache; explore tables and saved SQL; author and run
    approved GoogleSQL; export CSV; transparently ship tracked hosting analytics
    files through merge when SQL-save approval authorizes the cadence
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
required hosting-repo ship. When the user approves saving tracked SQL, ship
through merge runs transparently (worktree/PR not user-facing). The Squad
Leader receives only a compact terminal summary — keep execution detail on
this lane.

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

When the user **inspects** a saved query, resolve its **absolute** path under
`HOSTING_ROOT` (same clickable-path rules as §10) and call MCP
`mission_control_update_relevant_documents` with:

```json
{ "paths": [{ "path": "<absolute-sqlPath>", "kind": "other", "label": "Query: <filename-or-slug>" }] }
```

Registration failure must **not** block exploration; note it in recap and
continue.

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

USER_CHECKPOINT — approve and save query (also authorizes transparent hosting
ship through merge when tracked) · revise query · reuse an existing saved
query · abandon.

**Save approval = transparent ship consent (binding):** When the user picks
**approve and save** for a **hosting-tracked** SQL path, that same pick
authorizes §11 end-to-end (worktree → commit → push → PR → review → merge →
cleanup) with **no** further worktree, pipeline-depth, PR-review, or merge
consent modals. Worktree and PR mechanics are **agent-internal** — user-facing
copy must describe the outcome as the query landing in the **primary hosting
repo**, not as a multi-step git/PR workflow.

Save the approved body as `<queriesPath>/<query-slug>.sql`. Set
`developerApprovedQuery: true` only after this gate. If the SQL file is
intentionally local under a gitignored workspace, state that and plan
`shipStatus: skipped-local` for §11 (no transparent ship). If tracked, run
§11 after dry-run/execute when those steps complete (or immediately after save
when the user only asked to store without running — still finish §11 before
terminal).

**Relevant Links (binding):** After a successful save, resolve absolute
`sqlPath` via `sedea_get_hosting_root` (same form as §10) and call MCP
`mission_control_update_relevant_documents` on **this child lane**:

```json
{ "paths": [{ "path": "<absolute-sqlPath>", "kind": "other", "label": "Query: <query-slug>" }] }
```

Do **not** block dry-run/execute when registration fails — set
`relevantDocumentsRegistered: partial` or `skipped` and continue. **Forbidden:**
hand-editing Relevant Links / bundle JSON; registering out-of-workspace paths;
Squad Leader registering child artifacts (this lane owns the call).

### 9. Cost/safety preflight (dry-run)

Run `bq query --use_legacy_sql=false --dry_run` against the **exact** saved
file. Do not run mutations unless explicitly requested and approved.

USER_CHECKPOINT — run query, revise SQL, add/adjust LIMIT or filters, or abort.

### 10. Execute and export CSV

For a safely sized result, run the exact saved SQL with CSV output. For a large
result, materialize then `bq extract`. Write only under `inputs.resultsPath` on
the **main hosting clone** (`HOSTING_ROOT` from MCP `sedea_get_hosting_root`).
Using a timestamped or approved filename. Never overwrite an existing CSV
silently. **Forbidden:** writing results into `WORKTREE_ROOT` or copying CSVs
into a hosting worktree to make relative links resolve.

**Clickable path contract (binding):** Mission Host resolves relative markdown /
file links against the **active** workspace folder. When a hosting worktree is
mounted, repo-relative paths open under the worktree — not under
`HOSTING_ROOT`, where gitignored `results/` CSVs actually live.

1. Resolve **`HOSTING_ROOT`** via `sedea_get_hosting_root` before presenting any
   openable CSV (or SQL) path.
2. Set **`csvPath`** to the **absolute** path:
   `HOSTING_ROOT` + `/` + repo-relative results file under `inputs.resultsPath`.
3. Prefer the same absolute form for **`sqlPath`** when linking for
   open-in-editor.
4. **Forbidden** in `displayMarkdown`, chat, and other clickable surfaces: bare
   repo-relative paths (for example `analytics/.../results/foo.csv`) whenever a
   worktree folder may be mounted.
5. Repo-relative paths may appear only when explicitly labeled **non-clickable**
   ledger text; prefer absolute for both `csvPath` and `sqlPath` to avoid
   ambiguity.

Capture for the terminal summary:

- `sqlPath`, `csvPath` (absolute `HOSTING_ROOT` paths for clickable use);
- data row count (header excluded);
- byte size;
- job id and bytes processed when available;
- whether LIMIT/truncation/materialization applied.

**Relevant Links (binding):** After a successful CSV write, call MCP
`mission_control_update_relevant_documents` on **this child lane** with absolute
`csvPath` (and include `sqlPath` in the same call when both exist this turn):

```json
{
  "paths": [
    { "path": "<absolute-sqlPath>", "kind": "other", "label": "Query: <query-slug>" },
    { "path": "<absolute-csvPath>", "kind": "other", "label": "Result: <query-slug>" }
  ]
}
```

Omit the SQL entry when already registered in §8 (host dedupes). Registration
failure must **not** block §11 ship or terminal — set
`relevantDocumentsRegistered` accordingly. **Forbidden:** writing CSVs into
`WORKTREE_ROOT` so links resolve; bare repo-relative paths in `paths`.

### 11. Hosting-repo ship (when tracked) — transparent cadence

When §2 refreshed tracked `data-schema/` and/or §8 saved tracked SQL under the
hosting repo **and** §8 granted save approval (or schema-only ship was
explicitly approved at the §2 refresh gate):

**Mission-owned git (binding):** This skill owns the hosting ship for this
pass. Do **not** offer rule **6** Before-the-first-write or pipeline-depth
modals. §8 **approve and save** (or an explicit schema-ship approval) is the
sole consent for worktree create/attach, `commit-push-pr`, inline `pr-review`,
and agent `approve-merge-pr` / merge for **these named paths**.

**Transparent UX (binding):** Do **not** present USER_CHECKPOINTs for
create-worktree, commit/push/PR depth, start-pr-review, or merge after that
consent. Do **not** narrate worktree names, PR numbers, or review loops as the
primary user story. Prefer one outcome line: the SQL (and bundled schema when
applicable) is now in the **primary hosting repo** at the absolute
`HOSTING_ROOT` path. Keep `prUrl` / `prNumber` in terminal `outputs` for the
Squad Leader ledger only.

**Procedure (auto-advance):**

1. Reuse `inputs.hostingWorktreeHint` when it is an absolute active
   `WORKTREE_ROOT` for this pass; otherwise run center `worktree-setup.sh` →
   MCP `sedea_add_worktree_folder` → copy/write named tracked files under
   `WORKTREE_ROOT` per Sedea rules **0** / **6** / **7**.
2. Stage **named paths only** — never `git add .`.
3. `commit-push-pr` from `WORKTREE_ROOT` without a pipeline-depth modal.
4. Run inline `pr-review` to completion when required; do **not** open
   user pick loops for clean review. On Must/Should blockers only: open
   **one** recovery USER_CHECKPOINT (fix / defer / abandon ship) — not the
   full cadence restart.
5. Merge into the default branch using agent `gh pr review --approve` +
   `gh pr merge` authorized by the §8 save (or schema-ship) consent — treat
   that prior pick as same-pass `approve-merge-pr` for **this** PR. If merge
   is blocked (CI, permissions, conflicts), open **one** recovery gate; set
   `shipStatus: opened` or `failed` and keep `prUrl` / `prNumber`.
6. Post-merge: MCP `sedea_remove_worktree_folder` then center
   `worktree-cleanup.sh` **only** for Path A–owned `WORKTREE_ROOT`.
7. Set `shipStatus: merged` on success (else `opened` / `failed`); set
   `schemaShipStatus` / `prUrl` / `prNumber` accordingly.

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
| `sqlPath` | string — absolute under `HOSTING_ROOT` when used as an openable link |
| `csvPath` | string — **absolute** under `HOSTING_ROOT` (see §10 clickable path contract) |
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
| `relevantDocumentsRegistered` | `true` \| `partial` \| `skipped` — MCP Relevant Links registration outcome for SQL/CSV |

Use `errors: []` when none. Omit credential and token material.

## Completion (inline)

Same semantic fields as spawned `outputs`, reported in prose to the invoker.
No MCP spawn/result tools.
