# CI/CD Assessment - Changes Documentation

This document outlines all issues found and fixes applied to the GitHub Actions pipeline for building, releasing, and promoting packages to Cloudsmith.
---

## 6 Issues Found and Fixed

### Issue 1: Missing Build Command in `build_package.yml`

**File:** `.github/workflows/build_package.yml`
**Line:** 36-37

**Problem:** The "Build package" step was empty with no `run:` command, meaning no package would actually be built.

**Fix:** Added the build command:
```yaml
- name: Build package
  run: python -m build
```

---

### Issue 2: Missing Required Fields in `pyproject.toml`

**File:** `pyproject.toml`
**Line:** 5-6

**Problem:** The `pyproject.toml` file was missing the required `name` and `version` fields under `[project]`, which are mandatory for building a Python package.

**Fix:** Added the missing fields:
```toml
[project]
name = "example_package"
version = "0.0.1"
```

---

### Issue 3: Swapped Repository Names in `promote_package.yml`

**File:** `.github/workflows/promote_package.yml`
**Lines:** 13-14

**Problem:** The staging and production repository environment variables were swapped:
```yaml
CLOUDSMITH_STAGING_REPO: 'production'    # Wrong!
CLOUDSMITH_PRODUCTION_REPO: 'staging'    # Wrong!
```

**Fix:** Corrected the repository assignments:
```yaml
CLOUDSMITH_STAGING_REPO: 'staging'
CLOUDSMITH_PRODUCTION_REPO: 'production'
```

---

### Issue 4: Missing `id-token: write` Permission in `release_package.yml`

**File:** `.github/workflows/release_package.yml`
**Lines:** 16-18

**Problem:** The workflow was missing the `id-token: write` permission required for OIDC authentication with Cloudsmith.

**Fix:** Added the missing permission:
```yaml
permissions:
  contents: read
  actions: read
  id-token: write  # Added for OIDC authentication
```

---

### Issue 5: Missing Cloudsmith CLI Installation in `release_package.yml`

**File:** `.github/workflows/release_package.yml`
**Lines:** 36-40

**Problem:** The workflow attempted to use `cloudsmith push` command without first installing the Cloudsmith CLI.

**Fix:** Added the Cloudsmith CLI installation step:
```yaml
- name: Install Cloudsmith CLI
  uses: cloudsmith-io/cloudsmith-cli-action@v1.0.1
  with:
    oidc-namespace: ${{ env.CLOUDSMITH_NAMESPACE }}
    oidc-service-slug: ${{ env.CLOUDSMITH_SERVICE_SLUG }}
```

---

### Issue 6: Typo in `README.md`

**File:** `README.md`
**Line:** 27

**Problem:** The README referenced `realease_package.yml` instead of `release_package.yml` (typo: "realease" vs "release").

**Fix:** Corrected the filename reference:
```markdown
### 2. `release_package.yml`
```

---

## New Features Added to `promote_package.yml`

### Webhook-Triggered Promotion Pipeline

The `promote_package.yml` workflow has been completely redesigned to meet the following requirements:

#### 1. Removed Manual Trigger
- Replaced `workflow_dispatch` with `repository_dispatch` to enable webhook triggering
- The workflow is now triggered by the event type `cloudsmith-webhook`

#### 2. Webhook-Based Package Tagging
- Extracts package information from the webhook payload (`client_payload`)
- Tags the synced package with `ready-for-production` using `cloudsmith tags add`

#### 3. Tag-Based Package Query and Promotion
- Changed `PACKAGE_QUERY` to use `tag:ready-for-production`
- Queries all packages in staging with the `ready-for-production` tag
- Promotes all tagged packages to production repository

---

## Cloudsmith Webhook Configuration

To trigger the promote pipeline when a package syncs successfully, create a webhook in your Cloudsmith staging repository:

### Steps to Create the Webhook:

1. **Navigate to your Cloudsmith Organization**
   - Go to https://cloudsmith.io
   - Select your organization/namespace

2. **Access Repository Settings**
   - Go to your `staging` repository
   - Navigate to **Settings** → **Webhooks**

3. **Create New Webhook**
   - Click **Create Webhook**
   - Configure the following:

   | Field | Value |
   |-------|-------|
   | **Target URL** | `https://api.github.com/repos/{OWNER}/{REPO}/dispatches` |
   | **Payload Format** | `Handlebars template` |
   | **Template Format** | `JSON (application/json)` |
   | **Event Subscriptions** | Select `Package Synchronised` (Sync completed) |

4. **Security Settings:**

   | Field | Value |
   |-------|-------|
   | **Secret header** | `Authorization` |
   | **Secret value** | `token {GITHUB_PAT}` |
   | **HMAC signature key** | *(leave empty)* |
   | **Verify SSL certificate** | ✅ Enabled |

5. **Payload Template (Package synchronized):**
   ```json
   {
     "event_type": "cloudsmith-webhook",
     "client_payload": {
       "data": {
         "slug_perm": "{{data.slug_perm}}",
         "name": "{{data.name}}",
         "version": "{{data.version}}"
       }
     }
   }
   ```

6. **Required GitHub PAT Permissions:**
   - Create a Fine-grained PAT or Classic PAT with `repo` scope
   - The token must have access to trigger repository dispatch events

---

## OIDC Authentication Configuration

OIDC (OpenID Connect) authentication is required for this project. This eliminates the need for long-lived API keys.

### How OIDC Works with Cloudsmith

1. GitHub Actions requests an OIDC token from GitHub's OIDC provider
2. The token is sent to Cloudsmith for verification
3. Cloudsmith validates the token and grants temporary credentials
4. The workflow can then interact with Cloudsmith repositories

### Required GitHub Repository Variables

Create the following repository variables in **Settings** → **Secrets and variables** → **Actions** → **Variables**:

| Variable Name | Description | Example |
|--------------|-------------|---------|
| `CLOUDSMITH_NAMESPACE` | Your Cloudsmith organization/namespace | `my-org` |
| `CLOUDSMITH_SERVICE_SLUG` | The OIDC service account slug | `github-actions-oidc` |

### Cloudsmith OIDC Service Account Setup

1. **Navigate to Cloudsmith Settings**
   - Go to your organization → **Settings** → **Service Accounts**

2. **Create OIDC Service Account**
   - Click **Create Service Account**
   - Select **OIDC** as the authentication type
   - Configure the following:

   | Field | Value |
   |-------|-------|
   | **Name** | `github-actions-oidc` (or your preferred name) |
   | **Provider** | GitHub Actions |
   | **Repository** | `{OWNER}/{REPO}` (your GitHub repository) |

3. **Assign Repository Permissions**
   - Grant the service account appropriate permissions:
     - **staging repository:** Read, Write (for publishing and tagging)
     - **production repository:** Write (for promoting packages)

4. **Note the Service Slug**
   - After creation, note the service account slug
   - Use this as the `CLOUDSMITH_SERVICE_SLUG` variable

### Required Workflow Permissions

Each workflow that uses OIDC authentication needs:

```yaml
permissions:
  id-token: write    # Required for OIDC token request
  contents: read     # Required for repository access
  actions: read      # Required for artifact access (if applicable)
```

### Cloudsmith CLI Action Configuration

The workflows use the official Cloudsmith CLI action with OIDC:

```yaml
- name: Install Cloudsmith CLI
  uses: cloudsmith-io/cloudsmith-cli-action@v1.0.1
  with:
    oidc-namespace: ${{ env.CLOUDSMITH_NAMESPACE }}
    oidc-service-slug: ${{ env.CLOUDSMITH_SERVICE_SLUG }}
```

---

## Summary of Repository Variables Needed

| Variable | Purpose | Where to Set |
|----------|---------|--------------|
| `CLOUDSMITH_NAMESPACE` | Cloudsmith org name | GitHub Repository Variables |
| `CLOUDSMITH_SERVICE_SLUG` | OIDC service account slug | GitHub Repository Variables |

---

## Workflow Pipeline Flow

```
┌──────────────────┐
│   Push to main   │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ build_package.yml│
│  - Build Python  │
│  - Upload artifact│
└────────┬─────────┘
         │ (on success)
         ▼
┌──────────────────┐
│release_package.yml│
│  - Download artifact│
│  - Push to staging │
└────────┬─────────┘
         │ (webhook on sync)
         ▼
┌──────────────────┐
│promote_package.yml│
│  - Tag with ready │
│  - Query tagged   │
│  - Move to prod   │
└──────────────────┘
```

---

## Testing the Pipeline

1. **Create required Cloudsmith repositories:**
   - `staging` - for initial package uploads
   - `production` - for promoted packages

2. **Configure OIDC service account in Cloudsmith**

3. **Set up GitHub repository variables**

4. **Create webhook in Cloudsmith staging repository**

5. **Push a change to the main branch to trigger the pipeline**

6. **Monitor the workflow runs in GitHub Actions**
