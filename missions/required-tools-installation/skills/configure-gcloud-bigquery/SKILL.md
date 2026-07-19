---
name: Configure gcloud BigQuery
description: >-
  Configure a GCP project, service account, local key, isolated gcloud profile,
  and BigQuery IAM for durable non-interactive bq use.
inputs:
  projectPreference:
    type: string
    description: select-existing | create-default
    required: true
  existingProjectId:
    type: string
    required: false
  defaultNewProjectId:
    type: string
    required: true
  serviceAccountPreference:
    type: string
    description: select-existing | create-default
    required: true
  existingServiceAccountEmail:
    type: string
    required: false
  defaultNewServiceAccountId:
    type: string
    required: true
  credentialsTargetPath:
    type: string
    required: true
  gcloudConfigurationName:
    type: string
    required: true
timeoutMs: 1800000
warmUpRules:
  - .sedea/centers/bigquery-analytics/rules/00_bigquery-analytics.mdc
  - .sedea/centers/bigquery-analytics/rules/10_required-tools.mdc
  - .sedea/centers/bigquery-analytics/missions/required-tools-installation/plan.mdc
---

# Configure gcloud BigQuery

## Preconditions

- Bare `gcloud` and `bq` resolve on PATH.
- The lane was spawned by Required Tools Installation.
- Never print credential JSON, private keys, or access/refresh tokens.

## Steps

### 1. Bootstrap non-interactive shell

Set `CLOUDSDK_CORE_DISABLE_PROMPTS=1`. Probe `gcloud version` and `bq version`.
Do not PATH-prepend the SDK as a workaround.

### 2. Establish setup authority

Creating projects, service accounts, keys, and IAM bindings requires an existing
authorized principal. Check `gcloud auth list`.

If no active user exists, explain that a **one-time** `gcloud auth login` is
needed for setup authority. This does not become the routine query credential.
Open an external-wait modal, ask the user to authenticate in their terminal,
then re-probe. Routine use later activates the service-account key.

### 3. Select or create project

List accessible projects. Apply `inputs.projectPreference`.

- Existing: validate `inputs.existingProjectId`.
- Create: confirm the legal globally unique project id before
  `gcloud projects create`.

USER_CHECKPOINT — confirm project create when a new project is requested.

Set the selected project for setup commands and enable
`bigquery.googleapis.com`.

### 4. Select or create service account

List service accounts. Derive a legal 6–30-character account id from
`inputs.defaultNewServiceAccountId`; auto-shorten an overlong seed.

USER_CHECKPOINT — select existing service account or confirm creation.

Create when approved. Record only the service-account email.

### 5. Grant project IAM

Grant the service account:

- `roles/bigquery.jobUser`
- `roles/bigquery.user`

These support query jobs and optional dataset creation. Dataset-specific data
roles are applied by New Analytics Dataset Configuration. Do not grant
project-wide `roles/bigquery.admin`.

### 6. Create local key

Expand `inputs.credentialsTargetPath`, create its parent, and fail if the target
already contains an unrelated key unless the user approves replacement.

USER_CHECKPOINT — replace existing credential file, select existing service
account/key, or abort when the target conflicts.

Create a JSON key with `gcloud iam service-accounts keys create`. On Unix set
mode `0600`. Validate in-process that the JSON contains the expected
`client_email`; never display `private_key`.

If organization policy forbids key creation, do not weaken org policy
automatically. Offer:

- use an existing approved service-account key;
- request an administrator policy exception;
- switch to service-account impersonation (requires user credentials and
  `roles/iam.serviceAccountTokenCreator`, so it does not satisfy a fully
  user-login-free routine);
- abort.

USER_CHECKPOINT — resolve service-account key policy blocker.

### 7. Create isolated gcloud configuration

Create or activate `inputs.gcloudConfigurationName`, then:

```text
gcloud config configurations activate <name>
gcloud auth activate-service-account <email> --key-file <path> --project <project>
gcloud config set project <project>
```

This stores durable service-account configuration locally. Google still mints
short-lived access tokens internally; no human copies or renews them.

### 8. Verify

Run:

1. `gcloud auth list --filter=status:ACTIVE`
2. `bq ls --project_id=<project>`
3. `bq query --use_legacy_sql=false --dry_run "SELECT 1"`

Parse output without exposing credentials. Record command names, exit codes,
and the active service-account email.

### 9. Complete

Call `mission_control_send_agent_result`; omit all host identity fields.

## Completion (spawned)

Outputs:

| Field | Type |
|-------|------|
| `projectId` | string |
| `serviceAccountEmail` | string |
| `credentialsPath` | string |
| `gcloudConfigurationName` | string |
| `bigQueryApiEnabled` | boolean |
| `projectRoles` | array |
| `verifyPassed` | boolean |
| `continuationStatus` | string (`terminal`) |

Use `errors: []` when none. Never include credential contents or tokens.

