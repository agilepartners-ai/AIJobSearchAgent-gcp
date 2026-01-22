# GitHub CI/CD Setup for Cloud Run Deployment

This guide explains how to set up automated deployments to Google Cloud Run using GitHub Actions.

## 🎯 Overview

When code is pushed to the `main` branch, GitHub Actions will automatically:
1. Authenticate to Google Cloud
2. Submit a build to Cloud Build
3. Deploy to Cloud Run

## 📊 Architecture Overview

```
GitHub Push (main branch)
    ↓
GitHub Actions Workflow
    ↓ (uses github-actions-sa service account)
gcloud builds submit
    ↓
Cloud Build (uses Cloud Build service account)
    ↓
Build Docker Image → Push to Artifact Registry
    ↓
Deploy to Cloud Run (uses myjobsearchagent-sa service account)
```

### Service Accounts Summary

| Service Account | Purpose | Created By | Used For |
|----------------|---------|------------|----------|
| `github-actions-sa@PROJECT_ID.iam.gserviceaccount.com` | GitHub Actions authentication | **You create this** | GitHub Actions → Cloud Build |
| `PROJECT_NUMBER@cloudbuild.gserviceaccount.com` | Cloud Build operations | Auto-created by GCP | Cloud Build → Build & Deploy |
| `myjobsearchagent-sa@PROJECT_ID.iam.gserviceaccount.com` | Cloud Run runtime | You created earlier | Cloud Run service execution |

## 🔐 Authentication Methods

We provide two authentication methods:

### Method 1: Workload Identity Federation (Recommended - More Secure)
- ✅ No long-lived credentials
- ✅ Automatic credential rotation
- ✅ More secure
- ⚠️ Requires initial setup

### Method 2: Service Account Key (Simpler - Less Secure)
- ✅ Easier to set up
- ✅ Works immediately
- ⚠️ Requires managing service account keys

## 📋 Prerequisites

1. **Google Cloud Project** with billing enabled
2. **GitHub Repository** with Actions enabled
3. **Required APIs enabled**:
   - Cloud Build API
   - Cloud Run API
   - Artifact Registry API

## 🚀 Setup Instructions

## ⚠️ Important Notes

### Do I Need Cloud Build Trigger?
**No!** You don't need to set up a Cloud Build trigger. The GitHub Actions workflow will directly call `gcloud builds submit`, which triggers Cloud Build automatically. 

- **GitHub Actions approach**: GitHub Actions → `gcloud builds submit` → Cloud Build → Cloud Run
- **Cloud Build Trigger approach**: GitHub Push → Cloud Build Trigger → Cloud Build → Cloud Run (alternative method, not needed here)

### Do I Need a New Service Account for GitHub Actions?
**Yes!** You need to create a **separate service account** specifically for GitHub Actions. This is different from:

- **Cloud Build Service Account** (`PROJECT_NUMBER@cloudbuild.gserviceaccount.com`) - Automatically created, used by Cloud Build
- **Cloud Run Service Account** (`myjobsearchagent-sa@PROJECT_ID.iam.gserviceaccount.com`) - For running your app
- **GitHub Actions Service Account** (`github-actions-sa@PROJECT_ID.iam.gserviceaccount.com`) - **NEW** - For GitHub Actions to authenticate

### Step 1: Enable Required APIs

```bash
gcloud services enable cloudbuild.googleapis.com
gcloud services enable run.googleapis.com
gcloud services enable artifactregistry.googleapis.com
gcloud services enable iamcredentials.googleapis.com  # For Workload Identity
```

## ✅ If Using Cloud Build Triggers: Run Builds as a Dedicated Deploy Service Account (Recommended)

When you use **Cloud Build triggers**, Cloud Build executes your `cloudbuild.yml` as the **trigger’s service account**.
Using a dedicated deploy SA keeps permissions tight and avoids reusing:
- `myjobsearchagent-sa` (Cloud Run runtime identity) — **don’t reuse**
- `github-actions-sa` (GitHub OIDC identity) — works, but usually **too broad**

### Step A: Create a deploy service account

```bash
export PROJECT_ID="YOUR_PROJECT_ID"
export DEPLOY_SA_NAME="cloudbuild-deploy-sa"
export DEPLOY_SA_EMAIL="${DEPLOY_SA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com"

gcloud iam service-accounts create ${DEPLOY_SA_NAME} \
  --display-name="Cloud Build Deploy SA" \
  --description="Used by Cloud Build trigger to deploy to Cloud Run"
```

### Step B: Grant minimal roles for Cloud Run deploy

```bash
# Deploy/update Cloud Run services (includes ability to set IAM on the service)
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member="serviceAccount:${DEPLOY_SA_EMAIL}" \
  --role="roles/run.admin"

# Allow Cloud Build to write build logs to Cloud Logging when running as this service account
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member="serviceAccount:${DEPLOY_SA_EMAIL}" \
  --role="roles/logging.logWriter"

# Allow deploying a service that runs as your runtime service account
# This must match the runtime service account you pass to `gcloud run deploy --service-account=...`
export RUNTIME_SA_EMAIL="myjobsearchagent-sa@${PROJECT_ID}.iam.gserviceaccount.com"
export RUNTIME_SA_EMAIL="${SERVICE_ACCOUNT_EMAIL}"   # both above are same

gcloud iam service-accounts add-iam-policy-binding ${RUNTIME_SA_EMAIL} \
  --member="serviceAccount:${DEPLOY_SA_EMAIL}" \
  --role="roles/iam.serviceAccountUser"

# If the deploy step reads images from Artifact Registry
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member="serviceAccount:${DEPLOY_SA_EMAIL}" \
  --role="roles/artifactregistry.reader"
```

### Step C: Configure the trigger to use this service account

In **Google Cloud Console → Cloud Build → Triggers → (your trigger) → Edit**:
- Set **Service account** to `cloudbuild-deploy-sa@...`
- Ensure your trigger points at your `cloudbuild.yml`

### Troubleshooting: `cloudbuild-deploy-sa` not shown / cannot be used by Cloud Build

If you don’t see `cloudbuild-deploy-sa@...` in the trigger **Service account** dropdown, or trigger runs fail to
impersonate it, you usually need to grant **Service Account User** (`iam.serviceAccounts.actAs`) to:

1) **Your own Google user** (so it appears in the Console dropdown)

```bash
gcloud iam service-accounts add-iam-policy-binding cloudbuild-deploy-sa@${PROJECT_ID}.iam.gserviceaccount.com \
  --member="user:YOUR_EMAIL@example.com" \
  --role="roles/iam.serviceAccountUser"
```

2) **Cloud Build’s service identity** (so Cloud Build can run as this SA)

Common member formats (use your project number):

```bash
# Cloud Build service agent (recommended / most common)
gcloud iam service-accounts add-iam-policy-binding cloudbuild-deploy-sa@${PROJECT_ID}.iam.gserviceaccount.com \
  --member="serviceAccount:service-PROJECT_NUMBER@gcp-sa-cloudbuild.iam.gserviceaccount.com" \
  --role="roles/iam.serviceAccountUser"

# Classic Cloud Build SA (also common)
gcloud iam service-accounts add-iam-policy-binding cloudbuild-deploy-sa@${PROJECT_ID}.iam.gserviceaccount.com \
  --member="serviceAccount:PROJECT_NUMBER@cloudbuild.gserviceaccount.com" \
  --role="roles/iam.serviceAccountUser"
```
-----------------------------------------------

```bash
# add artifact registry writer role to the deploy service account

gcloud artifacts repositories add-iam-policy-binding "${REPOSITORY_NAME}" --location="${REGION}" \  --member="serviceAccount:cloudbuild-deploy-sa@aijs-app-dev.iam.gserviceaccount.com" \  --role="roles/artifactregistry.writer"

gcloud artifacts repositories add-iam-policy-binding ${REPOSITORY_NAME} --location="${REGION}" --member="serviceAccount:${DEPLOY_SA_EMAIL}" --role="roles/artifactregistry.writer"
```

**Important:** run these as a **single line command** (or ensure the member email isn’t split across lines). If you see
`INVALID_ARGUMENT: Invalid service account (service-...@gcp-sa-cloudbuild...)`, it’s often because the email was broken
by a newline during copy/paste.

## ✅ Option B (Hybrid): GitHub Actions starts a Cloud Build Trigger (Webhook-style)

Use this when you want to keep GitHub Actions as the “front door”, but run the actual build/deploy as the
**Cloud Build trigger’s service account** (instead of `gcloud builds submit` running as the default Cloud Build identity).

### Step 1: Create a Cloud Build trigger (runs on `main`)

In **Google Cloud Console → Cloud Build → Triggers → Create trigger**:
- **Event**: Push to a branch
- **Branch**: `^main$`
- **Build config**: `cloudbuild.yml`
- **Service account**: `cloudbuild-deploy-sa@...` (the deploy SA you created above)

**Required IAM for the trigger service account (`cloudbuild-deploy-sa@...`):**
- **Artifact Registry push** (fixes `artifactregistry.repositories.uploadArtifacts` errors):
  - Grant `roles/artifactregistry.writer` on your Artifact Registry repo (preferred) or project.
- **Cloud Run deploy**:
  - Grant `roles/run.admin` (or equivalent) on the project.
- **If you set `--service-account=...` on `gcloud run deploy`**:
  - Grant `roles/iam.serviceAccountUser` on the *Cloud Run runtime service account* you deploy with.

Record these trigger settings (you’ll use them in GitHub):
- **Trigger ID or name**
- **Trigger region** (often `global`, but depends on how you created it)

### Step 2: Add GitHub Secrets used to run the trigger

In your GitHub repo: **Settings → Secrets and variables → Actions → New repository secret**

| Secret Name | What it is |
|------------|------------|
| `CLOUD_BUILD_TRIGGER_NAME` | The trigger name (or ID) from Cloud Build |
| `CLOUD_BUILD_TRIGGER_REGION` | The trigger region (example: `global`) |

### Step 3: Update the GitHub Actions workflow to run the trigger (instead of `gcloud builds submit`)

In `.github/workflows/deploy_to_cloud_run.yml`, replace the “Submit build to Cloud Build” step with:

```bash
gcloud builds triggers run "${CLOUD_BUILD_TRIGGER_NAME}" \
  --region="${CLOUD_BUILD_TRIGGER_REGION}" \
  --branch="main" \
  --substitutions="_GAR_LOCATION=us-central1,_REPOSITORY=myjobsearchagent-repo,_SERVICE_NAME=myjobsearchagent,_SERVICE_ACCOUNT_EMAIL=myjobsearchagent-sa@PROJECT_ID.iam.gserviceaccount.com,_NEXT_PUBLIC_FIREBASE_API_KEY=***,_NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN=***,_NEXT_PUBLIC_FIREBASE_PROJECT_ID=***,_NEXT_PUBLIC_FIREBASE_STORAGE_BUCKET=***,_NEXT_PUBLIC_FIREBASE_MESSAGING_SENDER_ID=***,_NEXT_PUBLIC_FIREBASE_APP_ID=***,_NEXT_PUBLIC_FIREBASE_MEASUREMENT_ID=***,_NEXT_PUBLIC_JSEARCH_API_KEY=***,_NEXT_PUBLIC_JSEARCH_API_HOST=***,_NEXT_PUBLIC_TAVUS_API_KEY=***,_NEXT_PUBLIC_OPENAI_API_KEY=***,_NEXT_PUBLIC_GEMINI_API_KEY=***,_NEXT_PUBLIC_RESUME_API_BASE_URL=***,_NEXT_PUBLIC_RESUME_API_MODEL_TYPE=***,_NEXT_PUBLIC_RESUME_API_MODEL=***"
```

**Where the values come from:**
- `CLOUD_BUILD_TRIGGER_NAME` / `CLOUD_BUILD_TRIGGER_REGION`: from **GitHub Secrets**
- The substitution values: typically from **GitHub Secrets** (recommended), so Cloud Build can access build-time env vars during `pnpm run build`.
- Auth to run this command: still uses Workload Identity via the existing GitHub Action `auth@v2`

**Why substitutions are needed here**
- Your `cloudbuild.yml` uses custom substitutions like `${_NEXT_PUBLIC_FIREBASE_API_KEY}` to feed Docker build args.
- If you don’t pass them (and you didn’t set them on the trigger), they default to blank and your Next.js build can fail (example: `Firebase: Error (auth/invalid-api-key)`).

### Step 4: Where to define variables/secrets (cheat sheet)

- **GCP Trigger configuration (Cloud Console)**:
  - Which repo/branch triggers
  - Which `cloudbuild.yml` to run
  - Which **trigger service account** to run as (`cloudbuild-deploy-sa@...`)
- **GitHub Secrets** (values you don’t commit):
  - `WIF_PROVIDER`, `GH_SA_EMAIL`
  - `GCP_PROJECT_ID`, `GCP_GAR_LOCATION`, `GCP_REPOSITORY`, `GCP_SERVICE_NAME`, `GCP_SERVICE_ACCOUNT_EMAIL`
  - App build-time secrets like `NEXT_PUBLIC_FIREBASE_API_KEY`, etc.
  - (For Option B only) `CLOUD_BUILD_TRIGGER_NAME`, `CLOUD_BUILD_TRIGGER_REGION`
- **Workflow YAML** (references only, no secret values):
  - `${{ secrets.* }}` and `${{ env.* }}` wiring

### Step 2: Create Service Account for GitHub Actions

**This is a NEW service account you need to create.** It's different from your existing service accounts.

```bash
# Create service account (this is NEW - you need to create it)
gcloud iam service-accounts create github-actions-sa \
  --display-name="GitHub Actions Service Account" \
  --description="Service account for GitHub Actions to trigger Cloud Build"

# Grant necessary roles
export PROJECT_ID="YOUR_PROJECT_ID"
export GH_SA_EMAIL="github-actions-sa@${PROJECT_ID}.iam.gserviceaccount.com"

gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member="serviceAccount:${GH_SA_EMAIL}" \
  --role="roles/cloudbuild.builds.editor"

gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member="serviceAccount:${GH_SA_EMAIL}" \
  --role="roles/run.admin"

gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member="serviceAccount:${GH_SA_EMAIL}" \
  --role="roles/iam.serviceAccountUser"

gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member="serviceAccount:${GH_SA_EMAIL}" \
  --role="roles/artifactregistry.writer"

# Required so the service account can call enabled services like Cloud Build
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member="serviceAccount:${GH_SA_EMAIL}" \
  --role="roles/serviceusage.serviceUsageConsumer"

# Required because `gcloud builds submit` uploads your source to the Cloud Build staging bucket
# (gs://<PROJECT_ID>_cloudbuild) before starting the build.
# Note: In some orgs/projects this bucket is NOT auto-created until the first successful build,
# and in some cases it may never be auto-created due to org policies. If you get 404 BucketNotFound,
# create it manually (or use --gcs-source-staging-dir to point to a bucket you control).
#
# Create the default Cloud Build staging bucket (only needed if it doesn't exist):
#   gsutil mb -p ${PROJECT_ID} -l us -b on gs://${PROJECT_ID}_cloudbuild
#
# Option A (recommended minimum): grant access ONLY to the Cloud Build bucket:
# Important: `roles/storage.objectAdmin` does NOT include bucket metadata permissions like `storage.buckets.get`,
# which `gcloud builds submit` may require. Use `roles/storage.admin` on the *bucket* (scoped) instead.
#   gsutil iam ch serviceAccount:${GH_SA_EMAIL}:admin gs://${PROJECT_ID}_cloudbuild
#
# Alternative: use a custom staging bucket (and grant access to that bucket instead):
#   export STAGING_BUCKET="${PROJECT_ID}-cloudbuild-staging"
#   gsutil mb -p ${PROJECT_ID} -l us -b on gs://${STAGING_BUCKET}
#   gsutil iam ch serviceAccount:${GH_SA_EMAIL}:admin gs://${STAGING_BUCKET}
#   # and then in GitHub Actions: gcloud builds submit --gcs-source-staging-dir=gs://${STAGING_BUCKET}/source ...
#
# Option B (broader): grant project-wide permissions:
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member="serviceAccount:${GH_SA_EMAIL}" \
  --role="roles/storage.admin"
```

## 🔑 Method 1: Workload Identity Federation (Recommended)

### Step 1: Create Workload Identity Pool

```bash
# Set variables
export PROJECT_ID="appdev2025-123456"
export WIF_POOL_ID="github-actions-pool"
export WIF_PROVIDER_ID="github-provider"
export GH_SA_EMAIL="github-actions-sa@${PROJECT_ID}.iam.gserviceaccount.com"

# Create Workload Identity Pool
gcloud iam workload-identity-pools create ${WIF_POOL_ID} \
  --project=${PROJECT_ID} \
  --location="global" \
  --display-name="GitHub Actions Pool"

# Create Workload Identity Provider
# use organization name instead of your github username
gcloud iam workload-identity-pools providers create-oidc ${WIF_PROVIDER_ID} \
  --project=${PROJECT_ID} \
  --location="global" \
  --workload-identity-pool=${WIF_POOL_ID} \
  --display-name="GitHub Provider" \
  --attribute-mapping="google.subject=assertion.sub,attribute.actor=assertion.actor,attribute.repository=assertion.repository" \
  --attribute-condition="assertion.repository_owner=='YOUR_GITHUB_USERNAME'" \
  --issuer-uri="https://token.actions.githubusercontent.com"
```

### Step 2: Grant Service Account Access

```bash
# Get the provider resource name
PROVIDER_NAME=$(gcloud iam workload-identity-pools providers describe ${WIF_PROVIDER_ID} \
  --project=${PROJECT_ID} \
  --location="global" \
  --workload-identity-pool=${WIF_POOL_ID} \
  --format="value(name)")

# Grant access
# use organization name instead of your github username
# you can chcek the member= value from UI - Workload Identity federation tab
gcloud iam service-accounts add-iam-policy-binding ${GH_SA_EMAIL} \
  --project=${PROJECT_ID} \
  --role="roles/iam.workloadIdentityUser" \
  --member="principalSet://iam.googleapis.com/projects/PROJECT_NUMBER/locations/global/workloadIdentityPools/${WIF_POOL_ID}/attribute.repository/YOUR_GITHUB_USERNAME/YOUR_REPO_NAME"
```

### Step 3: Get Provider Resource Name

```bash
gcloud iam workload-identity-pools providers describe ${WIF_PROVIDER_ID} \
  --project=${PROJECT_ID} \
  --location="global" \
  --workload-identity-pool=${WIF_POOL_ID} \
  --format="value(name)"
```

### Step 4: Configure GitHub Secrets

1. Go to your GitHub repository
2. Navigate to **Settings** → **Secrets and variables** → **Actions**
3. Click **New repository secret** for each secret below
4. Add the following secrets:

#### Required Secrets for Workload Identity:

| Secret Name | Value | Example |
|------------|-------|---------|
| `WIF_PROVIDER` | The provider resource name from Step 3 | `projects/123456789/locations/global/workloadIdentityPools/github-actions-pool/providers/github-provider` |
| `GH_SA_EMAIL` | GitHub Actions service account email | `github-actions-sa@PROJECT_ID.iam.gserviceaccount.com` |

#### Required Secrets for GCP Configuration:

| Secret Name | Value | Example |
|------------|-------|---------|
| `GCP_PROJECT_ID` | Your Google Cloud Project ID | `appdev2025-123456` |
| `GCP_GAR_LOCATION` | Artifact Registry location | `us-central1` |
| `GCP_REPOSITORY` | Artifact Registry repository name | `myjobsearchagent-repo` |
| `GCP_SERVICE_NAME` | Cloud Run service name | `myjobsearchagent` |
| `GCP_SERVICE_ACCOUNT_EMAIL` | Cloud Run service account email | `myjobsearchagent-sa@PROJECT_ID.iam.gserviceaccount.com` |

#### Required Secrets for App Build (Next.js build-time env vars)

These are required because the build runs `pnpm run build`, and Firebase config is read at build time.
If these are missing/blank, you may see `Firebase: Error (auth/invalid-api-key)` during Cloud Build.

| Secret Name | Description |
|------------|-------------|
| `NEXT_PUBLIC_FIREBASE_API_KEY` | Firebase Web API key |
| `NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN` | Firebase auth domain |
| `NEXT_PUBLIC_FIREBASE_PROJECT_ID` | Firebase project id |
| `NEXT_PUBLIC_FIREBASE_STORAGE_BUCKET` | Firebase storage bucket |
| `NEXT_PUBLIC_FIREBASE_MESSAGING_SENDER_ID` | Firebase messaging sender id |
| `NEXT_PUBLIC_FIREBASE_APP_ID` | Firebase app id |
| `NEXT_PUBLIC_FIREBASE_MEASUREMENT_ID` | Firebase measurement id (optional if you don't use Analytics) |
| `NEXT_PUBLIC_JSEARCH_API_KEY` | JSearch API key (if used) |
| `NEXT_PUBLIC_JSEARCH_API_HOST` | JSearch API host (if used) |
| `NEXT_PUBLIC_TAVUS_API_KEY` | Tavus API key (if used) |
| `NEXT_PUBLIC_OPENAI_API_KEY` | OpenAI API key (if used) |
| `NEXT_PUBLIC_GEMINI_API_KEY` | Gemini API key (if used) |
| `NEXT_PUBLIC_RESUME_API_BASE_URL` | Resume API base URL (if used) |
| `NEXT_PUBLIC_RESUME_API_MODEL_TYPE` | Resume API model type (if used) |
| `NEXT_PUBLIC_RESUME_API_MODEL` | Resume API model (if used) |

### Step 5: Verify Workflow File

The workflow file `.github/workflows/deploy-to-cloud-run.yml` is already configured to use GitHub secrets. No changes needed!

**Note:** All configuration values are stored in GitHub secrets, so the workflow file is safe to commit to GitHub.

## How to delete WIF provider

```bash
gcloud iam workload-identity-pools providers delete ${WIF_PROVIDER_ID} \
  --project=${PROJECT_ID} \
  --location="global"
```
## Method 1 is used in this project

<hr> 
<hr> 

## 🔑 Method 2: Service Account Key (Simpler)

### Step 1: Create Service Account Key

```bash
# Set variables (if not already set)
export PROJECT_ID="YOUR_PROJECT_ID"
export GH_SA_EMAIL="github-actions-sa@${PROJECT_ID}.iam.gserviceaccount.com"

# Create and download key
gcloud iam service-accounts keys create github-actions-key.json \
  --iam-account=${GH_SA_EMAIL}

# Display the key (copy this)
cat github-actions-key.json
```

### Step 2: Configure GitHub Secrets

1. Go to your GitHub repository
2. Navigate to **Settings** → **Secrets and variables** → **Actions**
3. Click **New repository secret** for each secret below

#### Authentication Secret:

| Secret Name | Value |
|------------|-------|
| `GCP_SA_KEY` | Paste the entire contents of `github-actions-key.json` |

#### GCP Configuration Secrets:

| Secret Name | Value | Example |
|------------|-------|---------|
| `GCP_PROJECT_ID` | Your Google Cloud Project ID | `appdev2025-123456` |
| `GCP_GAR_LOCATION` | Artifact Registry location | `us-central1` |
| `GCP_REPOSITORY` | Artifact Registry repository name | `myjobsearchagent-repo` |
| `GCP_SERVICE_NAME` | Cloud Run service name | `myjobsearchagent` |
| `GCP_SERVICE_ACCOUNT_EMAIL` | Cloud Run service account email | `myjobsearchagent-sa@PROJECT_ID.iam.gserviceaccount.com` |

### Step 3: Use the Service Account Key Workflow

Rename or use `.github/workflows/deploy-to-cloud-run-sa-key.yml` as your workflow file.

**Note:** All configuration values are now stored in GitHub secrets, so no need to edit the workflow file!

## ✅ Testing the Workflow

### Option 1: Push to Main Branch
```bash
git add .
git commit -m "Add GitHub Actions workflow"
git push origin main
```

### Option 2: Manual Trigger
1. Go to **Actions** tab in GitHub
2. Select **Deploy to Cloud Run** workflow
3. Click **Run workflow**
4. Select branch and click **Run workflow**

## 🔍 Monitoring

1. **GitHub Actions**: Check the **Actions** tab in your repository
2. **Cloud Build**: View builds in Google Cloud Console → Cloud Build → History
3. **Cloud Run**: Check service status in Google Cloud Console → Cloud Run

<hr> 
<hr> 

## 🐛 Troubleshooting

### Authentication Errors

**Workload Identity:**
- Verify `WIF_PROVIDER` secret is correct
- Check repository owner matches in attribute condition
- Ensure service account has `roles/iam.workloadIdentityUser`

**Service Account Key:**
- Verify `GCP_SA_KEY` secret contains valid JSON
- Check service account has necessary roles
- Ensure key hasn't been deleted

### Build Failures

- Check Cloud Build logs in Google Cloud Console
- Verify `.env` file exists (not committed, but needed for build)
- Ensure all substitution variables are set correctly

### Permission Errors

- Verify service account has `roles/cloudbuild.builds.editor`
- Check Cloud Build service account has `roles/run.admin`
- Ensure Artifact Registry repository exists

### Cloud Run shows `Error: Forbidden` when opening the service URL

If you open the deployed Cloud Run URL and see:

- `Error: Forbidden`
- `Your client does not have permission to get URL / from this server.`

It means the Cloud Run service is **private** (authentication required) and the caller doesn’t have the **Invoker** permission.

**For a public-facing product**, grant Invoker to all users:

```bash
gcloud run services add-iam-policy-binding "${SERVICE_NAME}" \
  --region="${REGION}" \
  --member="allUsers" \
  --role="roles/run.invoker"
```

**Why this is needed**
- Cloud Run requires the caller to have `roles/run.invoker`. Without it, browsers (anonymous users) will get HTTP 403.
- This only makes the **HTTP endpoint** callable; it does **not** grant access to your GCP project/resources.

**How to verify it was applied**

```bash
gcloud run services get-iam-policy "${SERVICE_NAME}" \
  --region="${REGION}" \
  --flatten="bindings[].members" \
  --filter="bindings.role:roles/run.invoker" \
  --format="table(bindings.members)"
```

You should see `allUsers` listed.

## 🔒 Security Best Practices

1. ✅ **Use Workload Identity Federation** when possible (more secure)
2. ✅ **Rotate service account keys** regularly if using Method 2
3. ✅ **Limit service account permissions** to minimum required
4. ✅ **Review workflow files** before committing
5. ✅ **Use branch protection** for main branch
6. ✅ **Monitor Actions logs** for suspicious activity

## 📚 Additional Resources

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Google Cloud Build Documentation](https://cloud.google.com/build/docs)
- [Workload Identity Federation](https://cloud.google.com/iam/docs/workload-identity-federation)
- [Cloud Run Deployment](https://cloud.google.com/run/docs/deploying)

## 🎯 Quick Start Checklist

- [ ] Enable required Google Cloud APIs
- [ ] Create service account for GitHub Actions
- [ ] Grant necessary IAM roles
- [ ] Set up authentication (Workload Identity or Service Account Key)
- [ ] Configure GitHub secrets
- [ ] Update workflow environment variables
- [ ] Test workflow with a push to main
- [ ] Verify deployment in Cloud Run