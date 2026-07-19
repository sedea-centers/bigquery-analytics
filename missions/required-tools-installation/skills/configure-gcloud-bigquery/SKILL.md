---
name: Configure gcloud BigQuery
description: >-
  Configure a GCP project, service account, local key, isolated gcloud profile,
  and BigQuery IAM for durable non-interactive bq use.
designation:
  allowed: Configure GCP project, service account, local key, gcloud profile, and BigQuery IAM for durable non-interactive bq use
  forbidden: Explore datasets or run analytics queries; rewrite center rules or mission plans
inputs:
  projectPreference:
    type: string
    description: >-
      Optional hint only — select-existing | create-default. Child lane still
      authenticates gcloud when needed, then offers select vs create and lists
      live projects via gcloud for select.
    required: false
  existingProjectId:
    type: string
    description: >-
      Optional hint project id; validated against live gcloud projects list when
      selecting existing.
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

### 2. Confirm Google Cloud Console readiness (browser)

Before any terminal authentication or project/service-account work, confirm the
user can use Google Cloud Console in a browser.

Present short, friendly instructions (paraphrase freely; keep the checklist):

1. Open **[https://console.cloud.google.com/](https://console.cloud.google.com/)**
   in a browser (sign in with the Google account they will use for this setup).
2. Confirm they can reach the **Google Cloud console** (not an error page or a
   blocked-org screen).
3. **Readiness signal (binding):** Console is ready when they can open the
   **navigation sidebar** with Cloud dashboard items **and** use the
   **organization / project selector** to choose the intended org and project.
   A **“Start for Free”** / **“Try for free”** / **“Try for free now”** banner
   (or similar) is **not** diagnostic — it does **not** mean Cloud is missing
   or unusable. Do **not** require free-trial signup merely because that message
   appears.
4. If they cannot open the sidebar or select an organization/project, and their
   **organization** manages Google Cloud, they may need an admin to turn on
   Cloud for their user; the skill cannot do that for them.

Open an **external-wait / next-step** structured choice before ending the turn,
for example: **Sidebar and project selector work — continue to login**, **Still
blocked / need help**, **Abort**, **More details for option _**. Do **not**
proceed to `gcloud auth login` until the user selects a continue path that means
the Console sidebar and organization/project selector are usable.

### 3. Establish setup authority

Creating projects, service accounts, keys, and IAM bindings requires an existing
authorized principal. Check `gcloud auth list`.

If no active user exists, explain that a **one-time** `gcloud auth login` is
needed for setup authority. This does not become the routine query credential.
Open an external-wait modal, ask the user to authenticate in their terminal,
then re-probe until an active user is present. **Do not** list or create
projects until authentication succeeds. Routine use later activates the
service-account key.

### 4. Select or create project

After setup authority is authenticated, choose create vs select — then list or
create. Treat `inputs.projectPreference` / `inputs.existingProjectId` as
**hints only**; do not skip the live list when the user is selecting an
existing project.

USER_CHECKPOINT — **Select existing project** · **Create new project** ·
**More details for option _** (pre-fill from `projectPreference` when set).

- **Select existing project:** run `gcloud projects list` (or equivalent) and
  present accessible project ids in a USER_CHECKPOINT for the user to pick.
  Validate the chosen id; prefer `inputs.existingProjectId` only when it still
  appears in the live list and the user confirms it.
- **Create new project:** confirm the legal globally unique project id before
  `gcloud projects create`.

USER_CHECKPOINT — confirm project create when a new project is requested; or
pick from the live `gcloud` project list when selecting existing.

Set the selected project for setup commands and enable
`bigquery.googleapis.com`.

### 5. Select or create service account

List service accounts. Derive a legal 6–30-character account id from
`inputs.defaultNewServiceAccountId`; auto-shorten an overlong seed.

USER_CHECKPOINT — select existing service account or confirm creation.

Create when approved. Record only the service-account email.

### 6. Grant project IAM

Grant the service account:

- `roles/bigquery.jobUser`
- `roles/bigquery.user`

These support query jobs and optional dataset creation. Dataset-specific data
roles are applied by New Analytics Dataset Configuration. Do not grant
project-wide `roles/bigquery.admin`.

### 7. Create local key

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

### 8. Create isolated gcloud configuration

Create or activate `inputs.gcloudConfigurationName`, then:

```text
gcloud config configurations activate <name>
gcloud auth activate-service-account <email> --key-file <path> --project <project>
gcloud config set project <project>
```

This stores durable service-account configuration locally. Google still mints
short-lived access tokens internally; no human copies or renews them.

### 9. Verify

Run:

1. `gcloud auth list --filter=status:ACTIVE`
2. `bq ls --project_id=<project>`
3. `bq query --use_legacy_sql=false --dry_run "SELECT 1"`

Parse output without exposing credentials. Record command names, exit codes,
and the active service-account email.

### 10. Complete

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

