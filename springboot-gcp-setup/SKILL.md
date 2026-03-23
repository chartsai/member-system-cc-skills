---
name: springboot-gcp-setup
description: >
  TRIGGER this skill when the user says "set up GCP", "configure Google Cloud", "set up Cloud Run",
  "enable GCP APIs", "run springboot-gcp-setup", "set up gcloud", "configure my GCP project",
  or anything about connecting the project to Google Cloud Platform for deployment. This skill
  guides through the gcloud CLI setup interactively — it is a mix of automated steps and manual ones.
  Requires only springboot-setup (does NOT require scaffold or db).
---

# springboot-gcp-setup

A guided walkthrough for setting up Google Cloud Platform for this project. Mixes automated
`gcloud` CLI commands with instructions for steps that must be done in the GCP Console browser.
Outputs a checklist of what was completed.

---

## Step 0 — Read config

Read `.spring-config.json`. Extract:

```
app_name         → used for naming GCP resources
gcp_project_id   → GCP project ID (may be "TBD" — will ask user to confirm)
gcp_region       → deployment region (may be "TBD")
installed_modules → must contain "setup"
language         → respond in this language
translate_terms  → whether to translate technical terms
```

> **Language activation**: After reading `language` above, switch ALL your responses to that
> language immediately — including every question, status message, explanation, and all
> human-readable text in any generated files (HTML, Thymeleaf templates, SQL seed labels).
> If `language` is not set, ask: "What language should I use? (English / 繁體中文 / other)"



---

## Step 1 — Prerequisites check

Verify `installed_modules` contains `"setup"`. If not, tell the user to run `springboot-setup` first.

If `gcp_project_id` is `"TBD"`, ask the user:
```
Your .spring-config.json has gcp_project_id = "TBD".
What is your GCP project ID? (You can find it in the GCP Console at console.cloud.google.com)
```
Update `.spring-config.json` with the provided value before continuing.

Same for `gcp_region` if it is `"TBD"`.

---

## Beginner-Friendly Mode

If `beginner_friendly` is `true` in `.spring-config.json`, explain key concepts as you work. Examples:
- When running `gcloud init`: "gcloud is Google's command-line tool for managing GCP resources. `gcloud init` connects your terminal to your Google account and selects which GCP project to work with."
- When enabling APIs: "GCP APIs are features you have to explicitly turn on. Cloud Run (for hosting), Secret Manager (for passwords), and Artifact Registry (for Docker images) all need to be enabled before you can use them."
- When creating a Service Account: "A Service Account is like a user account for your app — it has its own credentials and you grant it only the permissions it needs. This is safer than using your personal account."

## Step 2 — Check if `gcloud` is installed

Run: `gcloud version`

**If installed:** Show the version and continue.

**If not installed:** Tell the user:

```
gcloud CLI is not installed. Please install it before continuing:

  macOS:   brew install --cask google-cloud-sdk
  Linux:   https://cloud.google.com/sdk/docs/install
  Windows: https://cloud.google.com/sdk/docs/install-sdk#windows

After installation, run: gcloud init
Then come back and re-run this skill.
```

Stop here if gcloud is not installed.

---

## Step 3 — Authenticate and set project

Run these commands one at a time, reporting output to the user:

```bash
# Check current auth status
gcloud auth list

# If not logged in:
gcloud auth login

# Set the project
gcloud config set project {{gcp_project_id}}

# Verify
gcloud config get project
```

If the user is already logged in with the correct account, skip `gcloud auth login`.

---

## Step 4 — Enable required GCP APIs

Run:

```bash
gcloud services enable \
  run.googleapis.com \
  secretmanager.googleapis.com \
  storage.googleapis.com \
  artifactregistry.googleapis.com \
  cloudbuild.googleapis.com \
  sqladmin.googleapis.com \
  --project={{gcp_project_id}}
```

This may take 1–2 minutes. Tell the user to wait.

Report which APIs were enabled.

---

## Step 5 — Create Artifact Registry repository

```bash
gcloud artifacts repositories create {{app_name_kebab}}-repo \
  --repository-format=docker \
  --location={{gcp_region}} \
  --description="Docker images for {{app_name}}" \
  --project={{gcp_project_id}}
```

If the repository already exists, note that and continue.

Save the full registry URL to use later:
```
{{gcp_region}}-docker.pkg.dev/{{gcp_project_id}}/{{app_name_kebab}}-repo
```

---

## Step 6 — Create a Service Account for the app

```bash
# Create service account
gcloud iam service-accounts create {{app_name_kebab}}-sa \
  --display-name="{{app_name}} App Service Account" \
  --project={{gcp_project_id}}

# Grant necessary roles
gcloud projects add-iam-policy-binding {{gcp_project_id}} \
  --member="serviceAccount:{{app_name_kebab}}-sa@{{gcp_project_id}}.iam.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"

gcloud projects add-iam-policy-binding {{gcp_project_id}} \
  --member="serviceAccount:{{app_name_kebab}}-sa@{{gcp_project_id}}.iam.gserviceaccount.com" \
  --role="roles/storage.objectAdmin"

gcloud projects add-iam-policy-binding {{gcp_project_id}} \
  --member="serviceAccount:{{app_name_kebab}}-sa@{{gcp_project_id}}.iam.gserviceaccount.com" \
  --role="roles/cloudsql.client"
```

---

## Step 7 — Create a local credentials key (for development)

```bash
gcloud iam service-accounts keys create \
  ./gcp-credentials-local.json \
  --iam-account={{app_name_kebab}}-sa@{{gcp_project_id}}.iam.gserviceaccount.com \
  --project={{gcp_project_id}}
```

**IMPORTANT:** Tell the user:
```
gcp-credentials-local.json has been created. This file contains sensitive credentials.

1. Add it to .gitignore immediately (already done if you used springboot-scaffold)
2. Never commit this file to git
3. Set GOOGLE_APPLICATION_CREDENTIALS=./gcp-credentials-local.json in your local environment
   when testing GCP-dependent features locally
```

Update `.spring-config.json` to add the credentials path:
```json
"gcp_credentials_path": "./gcp-credentials-local.json"
```

Also add `gcp-credentials-local.json` to `.gitignore` if it isn't already there.

---

## Step 8 — Steps that require the GCP Console (browser)

Tell the user these steps must be done manually in the browser:

```
The following steps require the GCP Console (https://console.cloud.google.com):

1. CLOUD SQL (Database for production):
   - Go to: SQL → Create Instance → PostgreSQL
   - Instance ID: {{app_name_kebab}}-db
   - Region: {{gcp_region}}
   - Database version: PostgreSQL 16
   - Machine type: db-f1-micro (for starters)
   - Create a database named: {{db_name}}
   - Create a user named: app (save the password!)
   - Note the "Connection name" (format: project:region:instance) — you'll need it for Cloud Run

2. SECRET MANAGER:
   - Go to: Secret Manager → Create Secret
   - Create these secrets:
     - DB_PASSWORD → your Cloud SQL app user password
     - GOOGLE_CLIENT_SECRET → your Google OAuth2 client secret (if using Google auth)
     - Any other sensitive config values

3. GOOGLE OAUTH2 (if using springboot-auth-google):
   - Go to: APIs & Services → Credentials → Create Credentials → OAuth 2.0 Client IDs
   - Application type: Web application
   - Authorized redirect URIs: {{app_url}}/login/oauth2/code/google
   - Save the Client ID and Client Secret

Let me know when you've completed these steps, or tell me which ones you want to skip for now.
```

---

## Step 9 — Output checklist

Print a summary checklist:

```
GCP Setup Checklist:
✅ gcloud authenticated
✅ Project set to: {{gcp_project_id}}
✅ APIs enabled: Cloud Run, Secret Manager, Cloud Storage, Artifact Registry, Cloud Build, Cloud SQL Admin
✅ Artifact Registry repo created: {{gcp_region}}-docker.pkg.dev/{{gcp_project_id}}/{{app_name_kebab}}-repo
✅ Service Account created: {{app_name_kebab}}-sa@{{gcp_project_id}}.iam.gserviceaccount.com
✅ Local credentials key: gcp-credentials-local.json

⏳ Needs manual action in GCP Console:
  □ Cloud SQL instance setup
  □ Secret Manager secrets
  □ OAuth2 credentials (if using Google auth)
```

Adjust checkmarks based on which steps actually succeeded.

---

## Step 10 — Update `installed_modules`

Read `.spring-config.json`, add `"gcp"` to `installed_modules`, write it back.

Also persist the artifact registry URL:
```json
"artifact_registry_url": "{{gcp_region}}-docker.pkg.dev/{{gcp_project_id}}/{{app_name_kebab}}-repo"
```

Tell the user:
```
GCP setup complete. You can now run:
- springboot-deploy → to set up Cloud Run deployment (requires scaffold + db)
- springboot-file-upload → for GCS file storage (requires scaffold + db)
```
