---
name: Explore BigQuery data
description: >-
  Help the user discover tables, schemas, previews, saved SQL, and useful query
  directions before handing an approved query intent to the Squad Leader.
designation:
  allowed: Explore tables, schemas, previews, and saved SQL; prepare approved query intent handoff to Squad Leader
  forbidden: Configure GCP IAM or service accounts; execute unapproved production mutations; rewrite center rules
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
  queriesPath:
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
timeoutMs: 1800000
warmUpRules:
  - .sedea/centers/bigquery-analytics/rules/00_bigquery-analytics.mdc
  - .sedea/centers/bigquery-analytics/missions/run-analytics-queries/plan.mdc
---

# Explore BigQuery data

## Orientation

Start by telling the user that they do not need to know the schema or formulate
perfect SQL. This lane exists to explore the dataset together and translate
their question into an evidence-backed query handoff.

## Steps

### 1. Verify scope and access

Activate `inputs.gcloudConfigurationName`. Confirm the active account is
`inputs.serviceAccountEmail`. Run metadata-only access probes against
`inputs.projectId:inputs.datasetId`. Do not switch projects/datasets silently.

### 2. Inventory saved queries

List readable `*.sql` files under `inputs.queriesPath`. For each, show filename
and a short purpose inferred from leading comments/query shape. Do not execute a
saved query merely to classify it.

USER_CHECKPOINT — inspect a saved query, continue dataset discovery, or describe
a new question.

### 3. List tables and views

Use `bq ls` for the configured dataset. Present concise table/view names and
types. When the list is long, page or group it rather than dumping everything.

USER_CHECKPOINT — inspect schemas for selected tables, ask what data exists,
preview a table, search by topic/name, or inspect saved queries.

### 4. Explain schemas

For selected tables/views, use `bq show --schema --format=prettyjson` (or an
equivalent metadata-only command). Summarize:

- field names and types;
- repeated/nested fields;
- likely keys and join candidates (clearly marked as inference);
- partitioning/clustering;
- freshness metadata when available.

Offer to compare related schemas and identify join paths.

USER_CHECKPOINT — compare tables, preview rows, formulate query ideas, inspect
another schema, or return to table list.

### 5. Preview data safely

Offer a bounded preview; default `LIMIT 10`. Use fully qualified identifiers and
Standard SQL. Before a preview that may scan substantial data, run a dry-run and
show estimated bytes.

Never expose obviously sensitive columns unnecessarily. If sensitive data may
be present, prefer a column-limited preview and ask before displaying values.

USER_CHECKPOINT — run bounded preview, change LIMIT/columns, skip preview, or
inspect another table.

### 6. Help the user learn the dataset

Based on metadata, previews, saved queries, and `inputs.inquiry`, offer concrete
inquiry directions such as:

- available business entities;
- time coverage and freshness;
- useful dimensions and metrics;
- common joins;
- data quality/null patterns;
- trends, cohorts, funnels, distributions, or anomalies;
- existing query reuse or extension.

These are educational prompts, not claims about business meaning. Ask the user
which direction best serves their decision.

USER_CHECKPOINT — choose a query direction, keep exploring, inspect saved SQL,
or abandon.

### 7. Build handoff

When the user has a useful direction, prepare:

- one-sentence query purpose;
- chosen tables/views and fields;
- joins and filters;
- date range;
- aggregation/granularity;
- preview-derived evidence;
- suggested output columns;
- LIMIT policy;
- existing saved query to reuse, if any;
- open assumptions.

USER_CHECKPOINT — approve exploration handoff, revise it, or continue exploring.

### 8. Complete

After approval, call `mission_control_send_agent_result`. Do not resolve the
dispatch.

## Completion (spawned)

Outputs:

| Field | Type |
|-------|------|
| `projectId` | string |
| `datasetId` | string |
| `queryPurpose` | string |
| `relevantTables` | array |
| `relevantFields` | array |
| `joinAndFilterPlan` | object |
| `previewEvidence` | array |
| `existingQueryPath` | string or empty |
| `suggestedOutputColumns` | array |
| `limitPolicy` | string |
| `openAssumptions` | array |
| `developerApprovedHandoff` | boolean |
| `continuationStatus` | `terminal` |

Use `errors: []` when none. Omit credential and token material.

