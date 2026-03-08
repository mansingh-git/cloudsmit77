# cloudsmit77 — Python Package Pipeline with OIDC Authentication

A production-style CI/CD pipeline designed and implemented to automate the full lifecycle of a Python package — from build through staged promotion — using GitHub Actions and Cloudsmith.

Authentication is handled exclusively via **OIDC (OpenID Connect)** token exchange. No static API keys or long-lived secrets are stored anywhere in the pipeline.

---

## Architecture

Three chained workflows handle the package lifecycle with clean separation of concerns:

| Workflow | Trigger | Responsibility |
|---|---|---|
| `build_package.yml` | Push or PR to `main` | Builds Python package, saves as Actions artifact |
| `release_package.yml` | `build_package` completes successfully | Downloads artifact, publishes to Cloudsmith **staging** |
| `promote_package.yml` | Manual dispatch by maintainer | Promotes package from **staging** → **production** |

---

## Key Design Decisions

**OIDC over API keys**  
GitHub Actions requests a short-lived OIDC token at runtime. Cloudsmith validates this token before granting access. Tokens are scoped to a single workflow run and expire within minutes — eliminating the risk of credential leaks from stored secrets.

**Workflow chaining**  
Each workflow has a single responsibility and hands off to the next only on success. This makes failures easy to isolate and keeps the blast radius of any broken step small.

**Manual promotion gate**  
The staging → production promotion is intentionally manual. This ensures a human sign-off before any package reaches production, which is appropriate for any release process where correctness matters.

---

## Repository Structure

```
.github/workflows/
  build_package.yml       # Build and artifact upload
  release_package.yml     # Publish to Cloudsmith staging
  promote_package.yml     # Promote staging → production
src/
  example_package/        # Python package source
pyproject.toml            # Package build configuration
CHANGES.md                # Changelog
```

---

## Note on Live Runs

Cloudsmith OIDC credentials for this repository are no longer active, so workflow runs will not complete end-to-end. The pipeline architecture, authentication design, and promotion pattern are the focus of this sample.

---

## Author

**Mansingh Chauhan** — Senior DevOps / SRE Engineer  
[linkedin.com/in/mansingh-chauhan-938b127a](https://linkedin.com/in/mansingh-chauhan-938b127a)
