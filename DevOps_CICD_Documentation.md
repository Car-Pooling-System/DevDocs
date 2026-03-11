# DevOps Strategy, CI/CD Pipeline & Deployment Documentation

**Car-Pooling System | Team Betamax | B.Tech CSE 2025-26**

**Amrita Vishwa Vidyapeetham**

---

## Table of Contents

1. [DevOps Strategy Overview](#1-devops-strategy-overview)
2. [Multi-Repository Architecture](#2-multi-repository-architecture)
3. [The 4th Repository — Deployment Orchestrator](#3-the-4th-repository--deployment-orchestrator)
4. [Deployment Environments](#4-deployment-environments)
5. [Deployment Platforms](#5-deployment-platforms)
6. [CI/CD Pipeline — Detailed Breakdown](#6-cicd-pipeline--detailed-breakdown)
7. [Master Deployment Pipeline (deploy-all.yml)](#7-master-deployment-pipeline-deploy-allyml)
8. [Backend CI/CD Pipeline (ci-cd.yml)](#8-backend-cicd-pipeline-ci-cdyml)
9. [Web Frontend CI/CD Pipeline (ci-cd.yml)](#9-web-frontend-cicd-pipeline-ci-cdyml)
10. [Mobile Frontend CI/CD Pipeline (ci-cd.yml)](#10-mobile-frontend-cicd-pipeline-ci-cdyml)
11. [Health Monitoring & Auto-Alerting (health-check.yml)](#11-health-monitoring--auto-alerting-health-checkyml)
12. [Integration Testing Pipeline (integration-test.yml)](#12-integration-testing-pipeline-integration-testyml)
13. [Emergency Rollback Pipeline (rollback.yml)](#13-emergency-rollback-pipeline-rollbackyml)
14. [Environment Variables & Secrets Management](#14-environment-variables--secrets-management)
15. [Branching Strategy](#15-branching-strategy)
16. [Deployment Flow Diagrams](#16-deployment-flow-diagrams)
17. [Testing Strategy](#17-testing-strategy)
18. [Utility Scripts](#18-utility-scripts)
19. [Troubleshooting Common Issues](#19-troubleshooting-common-issues)

---

## 1. DevOps Strategy Overview

The Car-Pooling System follows a **distributed, Backend-First deployment strategy**. The system is composed of four independent Git repositories, each with its own CI/CD pipeline, but coordinated through a centralized **Deployment Orchestrator** repository.

### Core Principles

| Principle | Implementation |
|:---|:---|
| **Backend-First** | Backend must be healthy before any frontend deploys |
| **Sequential Deployment** | Services deploy in a fixed order: Backend → Web → Mobile |
| **Automated Quality Gates** | Every push triggers linting, testing, security audits, and build checks |
| **Health Verification** | Automated health checks run every 6 hours via cron |
| **Emergency Rollback** | One-click rollback for any or all services |
| **Environment Isolation** | Staging (`develop` branch) and Production (`main` branch) are fully separated |

### Why This Strategy?

In a carpooling platform, the **backend API is the single source of truth** for ride data, user sessions, and payments. If the frontend updates before the backend, users may encounter broken API calls, incorrect data formats, or authentication failures. The Backend-First strategy guarantees that:

1. The API is live and responding before any UI changes go live.
2. Database migrations (if any) complete before clients start sending new request formats.
3. A failed backend deploy **blocks** frontend deploys, preventing cascading failures.

---

## 2. Multi-Repository Architecture

The system is split across four repositories under the `Car-Pooling-System` GitHub organization:

```
Car-Pooling-System (GitHub Organization)
│
├── Car-Pooling-System-Backend          ← Node.js/Express REST API
│   └── .github/workflows/ci-cd.yml
│
├── Car-Pooling-System-Web-Frontend     ← React + Vite SPA
│   └── .github/workflows/ci-cd.yml
│
├── Car-Pooling-System-Mobile-Frontend  ← React Native + Expo
│   └── .github/workflows/ci-cd.yml
│
└── carpooling-deployment               ← Master Orchestrator (4th Repo)
    └── .github/workflows/
        ├── deploy-all.yml              ← Master Pipeline
        ├── health-check.yml            ← Cron Monitoring
        ├── integration-test.yml        ← Cross-Service Tests
        └── rollback.yml                ← Emergency Rollback
```

### Why Separate Repositories?

| Reason | Explanation |
|:---|:---|
| **Independent Scaling** | Each service can be scaled, versioned, and deployed independently |
| **Team Isolation** | Different team members can work on frontend vs backend without merge conflicts |
| **Platform-Specific CI** | Backend needs MongoDB containers in CI; Mobile needs Expo CLI; Web needs Vite build |
| **Security** | Each repo has its own secrets (API keys, tokens) scoped to its deployment platform |
| **Rollback Granularity** | A broken frontend can be rolled back without affecting the backend |

---

## 3. The 4th Repository — Deployment Orchestrator

### Purpose

The `carpooling-deployment` repository is **not** a code repository. It contains zero application code. Instead, it serves as the **centralized command center** for:

1. **Triggering deployments** across all three service repos in the correct order.
2. **Running system-wide integration tests** that require all services to be live.
3. **Monitoring system health** via scheduled cron jobs.
4. **Emergency rollback** scripts.
5. **Documentation** of the overall system architecture.

### Repository Structure

```
carpooling-deployment/
├── .github/workflows/
│   ├── deploy-all.yml           # Master orchestration pipeline
│   ├── health-check.yml         # Cron-based health monitoring
│   ├── integration-test.yml     # Cross-service integration tests
│   └── rollback.yml             # Emergency rollback
├── config/
│   ├── repositories.yml         # Registry of all managed repos
│   ├── deployment-config.yml    # Deploy order, timeouts, environments
│   ├── environment-vars.yml     # Centralized env var documentation
│   ├── backend.json             # Backend-specific config
│   ├── frontend-web.json        # Web frontend config
│   └── frontend-mobile.json     # Mobile frontend config
├── scripts/
│   ├── deploy-all.sh            # CLI trigger for full deployment
│   ├── deploy.sh                # Single-service deploy
│   ├── health-check.sh          # Health check script
│   ├── rollback.sh              # Rollback script
│   ├── setup-local.sh           # Local dev environment setup
│   └── test-integration.sh      # Integration test runner
├── monitoring/                  # Status dashboards & alerts
├── tests/                       # System-wide integration test suites
├── docs/                        # Architecture guides
├── SECRETS.md                   # Secret configuration guide
├── CHANGELOG.md                 # Deployment history
└── README.md                    # Overview and quick start
```

### Deployment Configuration (`config/deployment-config.yml`)

```yaml
deployment:
  strategy: sequential
  order:
    - backend
    - frontend_web
    - frontend_mobile
  
  environments:
    staging:
      auto_deploy: true
      branch: develop
      notifications: true
    
    production:
      auto_deploy: false
      branch: main
      require_approval: true
      notifications: true

  timeouts:
    backend: 300s
    frontend: 600s
    mobile: 900s
```

Key takeaways:
- **Sequential strategy**: Services deploy one after another, never in parallel.
- **Production requires approval**: Manual confirmation is needed before production deploys.
- **Timeouts**: Backend has the shortest timeout (5 min); Mobile has the longest (15 min) due to Expo EAS build times.

---

## 4. Deployment Environments

### Staging Environment

| Service | Platform | URL | Trigger |
|:---|:---|:---|:---|
| Backend | Railway | `backend-staging.railway.app` | Push to `develop` |
| Web Frontend | Vercel | `carpooling-staging.vercel.app` | Push to `develop` |
| Mobile | Expo (Staging branch) | Internal test build | Push to `develop` |

### Production Environment

| Service | Platform | URL | Trigger |
|:---|:---|:---|:---|
| Backend | Railway | `car-pooling-system-backend-production.up.railway.app` | Push to `main` |
| Web Frontend | Vercel | Vercel auto-deploy from `main` | Push to `main` |
| Mobile (Android APK) | Expo EAS Build | Downloaded from `expo.dev` | Manual / `main` push |
| Mobile (Web Preview) | Firebase Hosting | Project: `swiftly-20535` | CI/CD on `main` push |

### Environment Flow

```
Developer pushes code
        │
        ├── to `develop` branch ──→ Staging Environment (auto-deploy)
        │                              ├── Railway Staging
        │                              ├── Vercel Preview
        │                              └── Expo Staging
        │
        └── to `main` branch ──→ Production Environment
                                       ├── Railway Production
                                       ├── Vercel Production
                                       └── EAS Build + Firebase Hosting
```

---

## 5. Deployment Platforms

### Railway (Backend API)

| Feature | Value |
|:---|:---|
| **Service** | Node.js 18 + Express |
| **Database** | MongoDB Atlas (external) |
| **Auto-Deploy** | Yes, on push to linked branch |
| **Health Endpoint** | `GET /health` → `{"message": "Server is running"}` |
| **Port** | Dynamically assigned via `process.env.PORT` |

**Why Railway?**
- Native Node.js support with zero-config deployment.
- Automatic HTTPS and custom domain support.
- Built-in logging, metrics, and easy env var management.
- Cost-effective for small-to-medium scale backends.

### Vercel (Web Frontend)

| Feature | Value |
|:---|:---|
| **Framework** | Vite + React 19 |
| **Build Command** | `npm run build` |
| **Output Directory** | `dist/` |
| **Preview Deployments** | Automatic for every Pull Request |
| **SPA Routing** | Handled via `vercel.json` rewrite to `index.html` |

**Why Vercel?**
- First-class Vite/React support.
- Global CDN for fast page loads.
- Automatic Preview Deployments for every PR.
- Seamless GitHub integration.

**Important Configuration (`vercel.json`):**
```json
{
    "rewrites": [
        {
            "source": "/(.*)",
            "destination": "/index.html"
        }
    ]
}
```
This catch-all rewrite enables client-side routing in the SPA. Without it, direct navigation to routes like `/search` or `/my-rides` would return 404.

### Expo EAS (Mobile — Android APK)

| Feature | Value |
|:---|:---|
| **Framework** | React Native + Expo SDK |
| **Build Profile** | `preview` (APK) and `production` (AAB) |
| **OTA Updates** | Supported via `eas update` |
| **Package Name** | `com.nanducc.carpooling` |

**Why Expo EAS?**
- Industry standard for React Native builds.
- Cloud-based build infrastructure (no local Android SDK needed).
- Over-the-air (OTA) updates without app store re-submission.

### Firebase Hosting (Mobile — Web Preview)

| Feature | Value |
|:---|:---|
| **Project** | `swiftly-20535` |
| **Purpose** | Web-exported version of mobile app |
| **Build** | `npm run build:web` (Expo web export) |
| **Deploy Action** | `FirebaseExtended/action-hosting-deploy@v0` |

---

## 6. CI/CD Pipeline — Detailed Breakdown

Each repository contains a `ci-cd.yml` GitHub Actions workflow. All pipelines follow a consistent multi-stage structure:

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│   LINT       │───→│  UNIT TESTS  │───→│ INTEGRATION  │───→│    BUILD     │───→│   DEPLOY     │
│              │    │              │    │    TESTS     │    │  + SECURITY  │    │              │
└──────────────┘    └──────────────┘    └──────────────┘    └──────────────┘    └──────────────┘
 Code Quality        Fast Feedback       DB/API Tests       Artifact + Audit    Platform Deploy
```

### Trigger Events (Common Across All Repos)

```yaml
on:
  workflow_dispatch:        # Manual trigger with environment selection
    inputs:
      environment:
        type: choice
        options: [staging, production]
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]
```

- **`push`**: Triggers full pipeline on code push to `main` or `develop`.
- **`pull_request`**: Runs quality checks and tests on PRs (does NOT deploy).
- **`workflow_dispatch`**: Allows manual triggering from GitHub UI or API (used by the orchestrator).

---

## 7. Master Deployment Pipeline (`deploy-all.yml`)

**Location:** `carpooling-deployment/.github/workflows/deploy-all.yml`

This is the **most critical workflow** in the entire system. It orchestrates a full system deployment by triggering each service's individual pipeline in sequence.

### Workflow Inputs

| Input | Type | Default | Description |
|:---|:---|:---|:---|
| `environment` | choice | `staging` | Target environment: `staging` or `production` |
| `skip_tests` | boolean | `false` | Skip integration tests after deployment |

### Pipeline Jobs (Sequential)

```
┌──────────────────┐
│  deploy-backend  │  ← Step 1: Triggers Backend CI/CD
│                  │     Waits 60s for Railway deploy
│                  │     Pings /health endpoint
│                  │     FAILS ENTIRE PIPELINE if unhealthy
└────────┬─────────┘
         │ (depends on backend success)
         ▼
┌──────────────────┐
│   deploy-web     │  ← Step 2: Triggers Web CI/CD
│                  │     Vercel builds and deploys
└────────┬─────────┘
         │ (depends on web success)
         ▼
┌──────────────────┐
│  deploy-mobile   │  ← Step 3: Triggers Mobile CI/CD
│                  │     Expo/Firebase deploy
└────────┬─────────┘
         │ (depends on ALL three)
         ▼
┌──────────────────┐
│ integration-test │  ← Step 4: Cross-service integration tests
│                  │     Tests live API + Web + Mobile URLs
│                  │     Skippable via input flag
└──────────────────┘
```

### How Cross-Repo Triggering Works

The orchestrator uses the **GitHub Actions API** via `actions/github-script@v6` to trigger workflows in other repositories:

```yaml
- name: Trigger Backend Workflow
  uses: actions/github-script@v6
  with:
    github-token: ${{ secrets.GH_PAT }}
    script: |
      await github.rest.actions.createWorkflowDispatch({
        owner: 'Car-Pooling-System',
        repo: 'Car-Pooling-System-Backend',
        workflow_id: 'ci-cd.yml',
        ref: 'main',
        inputs: { environment: '${{ inputs.environment }}' }
      })
```

**Key Points:**
- Requires a `GH_PAT` (GitHub Personal Access Token) with `repo` and `workflow` scopes.
- The token must belong to a user with write access to all target repositories.
- Each triggered workflow runs its full lint → test → build → deploy pipeline.

### Health Verification Gate

After triggering the backend deploy, the pipeline **waits 60 seconds** and then verifies the backend is healthy:

```yaml
- name: Verify Backend Health
  run: |
    URL="${{ inputs.environment == 'production' && 'https://backend.railway.app' || 'https://backend-staging.railway.app' }}"
    curl -f "$URL/health" || exit 1
```

If the health check fails (`exit 1`), the entire pipeline stops. Web and Mobile deployments are **never triggered** if the backend is unhealthy.

---

## 8. Backend CI/CD Pipeline (`ci-cd.yml`)

**Location:** `Car-Pooling-System-Backend/.github/workflows/ci-cd.yml`  
**Node Version:** 18

### Job Stages

| # | Job | Depends On | Purpose |
|:--|:---|:---|:---|
| 1 | `lint` | — | ESLint + Prettier formatting check |
| 2 | `unit-tests` | `lint` | Jest unit tests with `NODE_ENV=test` |
| 3 | `integration-tests` | `unit-tests` | Tests with real MongoDB instance |
| 4 | `build` | `integration-tests` | Security audit (`npm audit`) + build check |
| 5 | `deploy-staging` | `build` | Notify Railway staging (auto-deploy on `develop`) |
| 6 | `deploy-production` | `build` | Notify Railway production (auto-deploy on `main`) |

### MongoDB Service Container

The integration tests spin up a **real MongoDB 7 instance** inside the GitHub Actions runner:

```yaml
services:
  mongodb:
    image: mongo:7
    ports:
      - 27017:27017
```

This allows integration tests to run against a real database with the connection string:
```
MONGODB_URI: mongodb://localhost:27017/carpooling_test
```

### Deployment Strategy

Railway is configured for **auto-deploy on push**. The CI/CD pipeline's deploy steps simply log a notification because Railway watches the branch directly:

```yaml
deploy-production:
  if: github.ref == 'refs/heads/main' && github.event_name == 'push'
  steps:
    - name: Notify deployment
      run: echo "Railway auto-deploys from main branch."
```

---

## 9. Web Frontend CI/CD Pipeline (`ci-cd.yml`)

**Location:** `Car-Pooling-System-Web-Frontend/.github/workflows/ci-cd.yml`
**Node Version:** 20

### Job Stages

| # | Job | Depends On | Purpose |
|:--|:---|:---|:---|
| 1 | `lint` | — | ESLint + Prettier |
| 2 | `test` | `lint` | Unit tests |
| 3 | `integration-tests` | `test` | Integration tests |
| 4 | `e2e-tests` | `integration-tests` | End-to-end / Cypress tests |
| 5 | `build` | `e2e-tests` | Vite production build + security audit + artifact upload |
| 6 | `deploy-staging` | `build` | Vercel auto-deploy from `develop` |
| 7 | `deploy-production` | `build` | Vercel auto-deploy from `main` |

### Build with Environment-Aware API URL

The build step dynamically sets the backend URL based on the branch:

```yaml
- name: Build application
  run: npm run build
  env:
    REACT_APP_API_URL: ${{ github.ref == 'refs/heads/main' && 'https://backend.railway.app' || 'https://backend-staging.railway.app' }}
    REACT_APP_ENV: ${{ github.ref == 'refs/heads/main' && 'production' || 'staging' }}
```

### Build Artifact Upload

The built `dist/` folder is uploaded as a GitHub Actions artifact for debugging or manual deployment:

```yaml
- name: Upload build artifacts
  uses: actions/upload-artifact@v4
  with:
    name: build
    path: dist/
    retention-days: 7
```

### Build Size Tracking

Each build logs the total bundle size to the GitHub Actions step summary:

```yaml
- name: Check build size
  run: |
    if [ -d "dist" ]; then
      SIZE=$(du -sh dist | cut -f1)
      echo "Build size: $SIZE"
      echo "## Build Info" >> $GITHUB_STEP_SUMMARY
      echo "Build size: $SIZE" >> $GITHUB_STEP_SUMMARY
    fi
```

---

## 10. Mobile Frontend CI/CD Pipeline (`ci-cd.yml`)

**Location:** `Car-Pooling-System-Mobile-Frontend/.github/workflows/ci-cd.yml`  
**Node Version:** 18

### Job Stages

| # | Job | Depends On | Purpose |
|:--|:---|:---|:---|
| 1 | `lint` | — | ESLint (with config file detection) |
| 2 | `unit-tests` | `lint` | Jest with `--passWithNoTests` |
| 3 | `integration-tests` | `unit-tests` | Integration tests |
| 4 | `api-tests` | `integration-tests` | API / E2E tests |
| 5 | `build-security` | `unit-tests` + `integration-tests` + `api-tests` | Security audit + build validation |
| 6 | `deploy-staging` | `build-security` | Expo publish to staging branch |
| 7 | `deploy-production` | `build-security` | Firebase Hosting deploy (web) |

### Expo-Specific Configuration

The mobile pipeline uses the official Expo GitHub Action:

```yaml
- name: Setup Expo
  uses: expo/expo-github-action@v8
  with:
    expo-version: latest
    token: ${{ secrets.EXPO_TOKEN }}
```

### Legacy Peer Dependency Handling

React Native has complex dependency trees. The install step falls back gracefully:

```yaml
- name: Install dependencies
  run: npm install --legacy-peer-deps || npm install
  continue-on-error: false
```

### Production Deploy to Firebase

The production deploy builds the web export and pushes to Firebase Hosting:

```yaml
- name: Build Web App
  run: npm run build:web

- name: Deploy to Firebase Hosting
  uses: FirebaseExtended/action-hosting-deploy@v0
  with:
    repoToken: "${{ secrets.GITHUB_TOKEN }}"
    firebaseServiceAccount: "${{ secrets.FIREBASE_SERVICE_ACCOUNT_SWIFTLY_20535 }}"
    projectId: swiftly-20535
    channelId: live
```

---

## 11. Health Monitoring & Auto-Alerting (`health-check.yml`)

**Location:** `carpooling-deployment/.github/workflows/health-check.yml`

### Schedule

```yaml
on:
  schedule:
    - cron: '0 */6 * * *'  # Every 6 hours
  workflow_dispatch:         # Manual trigger
```

### What It Checks

The workflow runs `scripts/health-check.sh`, which:

1. **Pings the Backend** at `/health` and expects HTTP 200.
2. **Pings the Web Frontend** at `/` and expects HTTP 200.
3. If either check fails, exits with code 1.

```bash
# Check Backend
CODE=$(curl -s -o /dev/null -w "%{http_code}" "$BACKEND_URL/health")
if [ "$CODE" -eq 200 ]; then
  echo "✅ UP"
else
  echo "❌ DOWN ($CODE)"
  exit 1
fi
```

### Auto-Alerting on Failure

If the health check fails, a **GitHub Issue is automatically created**:

```yaml
- name: Send Alert on Failure
  if: failure()
  uses: actions/github-script@v6
  with:
    script: |
      github.rest.issues.create({
        owner: context.repo.owner,
        repo: context.repo.repo,
        title: '🚨 Health Check Failed',
        body: 'The system health check failed. verify the logs.'
      })
```

---

## 12. Integration Testing Pipeline (`integration-test.yml`)

**Location:** `carpooling-deployment/.github/workflows/integration-test.yml`

Runs cross-service integration tests against live deployments. Can be triggered:
- **Manually** via `workflow_dispatch`
- **Programmatically** via `repository_dispatch` (e.g., after a successful deployment)

```yaml
on:
  workflow_dispatch:
  repository_dispatch:
    types: [trigger-tests]
```

---

## 13. Emergency Rollback Pipeline (`rollback.yml`)

**Location:** `carpooling-deployment/.github/workflows/rollback.yml`

### Inputs

| Input | Type | Options | Required |
|:---|:---|:---|:---|
| `service` | choice | `all`, `backend`, `web`, `mobile` | Yes |
| `reason` | text | Free-form | Yes |

### Execution

Runs `scripts/rollback.sh` with the selected service:

```bash
#!/bin/bash
SERVICE=${1:-all}
ORG="Car-Pooling-System"

echo "⚠️  Initiating Rollback for: $SERVICE"

gh workflow run rollback.yml \
  --repo $ORG/deployment \
  --ref main \
  -f service=$SERVICE \
  -f reason="Manual Trigger via CLI"
```

The rollback script uses the GitHub CLI (`gh`) to trigger rollback actions, which can revert Railway deployments, Vercel builds, and Expo updates to their previous stable versions.

---

## 14. Environment Variables & Secrets Management

### GitHub Secrets (Per Repository)

| Repository | Secret Name | Source | Purpose |
|:---|:---|:---|:---|
| `carpooling-deployment` | `GH_PAT` | GitHub Developer Settings | Cross-repo workflow triggers |
| `Backend` | `JWT_SECRET` | Generated (random 32-byte hex) | Token signing |
| `Web Frontend` | `VERCEL_TOKEN` | Vercel Dashboard | Deployment auth |
| `Mobile Frontend` | `EXPO_TOKEN` | Expo Dashboard | EAS build + publish |
| `Mobile Frontend` | `FIREBASE_SERVICE_ACCOUNT_SWIFTLY_20535` | Firebase Console | Firebase Hosting deploy |

### Backend Environment Variables

| Variable | Purpose |
|:---|:---|
| `MONGO_URI` | MongoDB Atlas connection string |
| `PORT` | Server port (auto-assigned by Railway) |
| `TWILIO_ACCOUNT_SID` | SMS verification service |
| `TWILIO_AUTH_TOKEN` | SMS auth token |
| `CLERK_SECRET_KEY` | Clerk authentication |
| `NODE_ENV` | `production` / `staging` / `test` |

### Web Frontend Environment Variables

| Variable | Purpose |
|:---|:---|
| `VITE_API_URL` | Backend Railway URL |
| `VITE_GOOGLE_MAPS_API_KEY` | Google Maps widget |
| `VITE_CLERK_PUBLISHABLE_KEY` | Clerk client auth |

### Mobile Frontend Environment Variables

| Variable | Purpose |
|:---|:---|
| `EXPO_PUBLIC_API_URL` | Backend Railway URL |
| `EXPO_PUBLIC_GOOGLE_MAPS_API_KEY` | Google Maps |
| `EXPO_PUBLIC_CLERK_PUBLISHABLE_KEY` | Clerk auth |
| `EXPO_PUBLIC_FIREBASE_API_KEY` | Firebase config |
| `EXPO_PUBLIC_FIREBASE_PROJECT_ID` | Firebase project |

### Security Notes

- All `.env` and `.env.production` files are listed in `.gitignore` and **never committed** to Git.
- Secrets are managed via GitHub Secrets (for CI/CD) and platform-specific env var dashboards (Railway, Vercel, Expo).
- The `SECRETS.md` file in the deployment repo provides step-by-step instructions for generating and configuring each secret.

---

## 15. Branching Strategy

```
main ─────────────────────────────────────────── Production
  │
  └── develop ────────────────────────────────── Staging
       │
       ├── feature/ride-search ────── Feature Branch
       ├── feature/payment-gateway ── Feature Branch
       └── bugfix/map-crash ────────── Bugfix Branch
```

| Branch | Environment | Auto-Deploy? | Protected? |
|:---|:---|:---|:---|
| `main` | Production | Yes (Railway, Vercel) | Yes (requires PR review) |
| `develop` | Staging | Yes | Recommended |
| `feature/*` | Local / PR Preview | Vercel Preview only | No |
| `bugfix/*` | Local / PR Preview | Vercel Preview only | No |

### Workflow

1. Developer creates a `feature/*` branch from `develop`.
2. Development and testing happen locally.
3. A Pull Request is opened to `develop` → CI/CD runs lint + tests.
4. After PR merge → Staging auto-deploys.
5. After staging validation → PR from `develop` to `main`.
6. After merge to `main` → Production auto-deploys.
7. Optionally, the Master Pipeline (`deploy-all.yml`) is triggered manually for a coordinated release.

---

## 16. Deployment Flow Diagrams

### Full System Deployment (Triggered via Master Pipeline)

```
┌─────────────────────────────────────────────────────────────────┐
│                     MASTER PIPELINE                             │
│                   (deploy-all.yml)                              │
│                                                                 │
│  ┌─────────────────────┐                                        │
│  │   Manual Trigger    │  environment: staging/production       │
│  │  (workflow_dispatch)│  skip_tests: true/false                │
│  └──────────┬──────────┘                                        │
│             │                                                   │
│             ▼                                                   │
│  ┌─────────────────────┐                                        │
│  │  DEPLOY BACKEND     │  → GitHub API triggers Backend ci-cd   │
│  │  Wait 60s           │  → curl /health                       │
│  │  Verify Health ✅   │  → If FAIL: pipeline STOPS ❌         │
│  └──────────┬──────────┘                                        │
│             │                                                   │
│             ▼                                                   │
│  ┌─────────────────────┐                                        │
│  │  DEPLOY WEB         │  → GitHub API triggers Web ci-cd      │
│  │  Vercel builds      │  → Preview URL generated              │
│  └──────────┬──────────┘                                        │
│             │                                                   │
│             ▼                                                   │
│  ┌─────────────────────┐                                        │
│  │  DEPLOY MOBILE      │  → GitHub API triggers Mobile ci-cd   │
│  │  EAS Build / Firebase│  → APK generated or web deployed     │
│  └──────────┬──────────┘                                        │
│             │                                                   │
│             ▼                                                   │
│  ┌─────────────────────┐                                        │
│  │  INTEGRATION TESTS  │  → Tests all live URLs together       │
│  │  (if not skipped)   │  → API calls, health checks           │
│  └─────────────────────┘                                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Individual Service CI/CD Flow

```
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│   LINT   │───→│  UNIT    │───→│ INTEGRA- │───→│  BUILD   │───→│  DEPLOY  │
│          │    │  TESTS   │    │  TION    │    │  + AUDIT │    │          │
│ ESLint   │    │  Jest    │    │  Tests   │    │ npm audit│    │ Railway  │
│ Prettier │    │          │    │ MongoDB  │    │ Vite     │    │ Vercel   │
│          │    │          │    │ (Backend)│    │          │    │ Expo/FB  │
└──────────┘    └──────────┘    └──────────┘    └──────────┘    └──────────┘
```

---

## 17. Testing Strategy

### Testing Pyramid

```
        ┌───────────────┐
        │   E2E Tests   │  ← Cypress / manual (Web Frontend)
        │   (Slowest)   │
        ├───────────────┤
        │  Integration  │  ← Real MongoDB, API calls
        │    Tests      │
        ├───────────────┤
        │  Unit Tests   │  ← Jest, fast isolated tests
        │  (Fastest)    │
        └───────────────┘
```

### Testing Tools

| Tool | Purpose | Used In |
|:---|:---|:---|
| **ESLint** | Static code analysis, lint rules | All repos |
| **Prettier** | Code formatting enforcement | All repos |
| **Jest** | Unit + integration testing | All repos |
| **MongoDB Service Container** | Real DB for integration tests | Backend CI |
| **npm audit** | Security vulnerability scanning | All repos |
| **curl** | Health endpoint verification | Deployment repo |
| **Cypress** | E2E browser testing (if configured) | Web Frontend |
| **GitHub Actions** | CI runner (`ubuntu-latest`) | All repos |

### Test Commands

| Repo | Command | What It Tests |
|:---|:---|:---|
| Backend | `npm run test:unit` | Business logic, utils |
| Backend | `npm run test:integration` | Database operations with real MongoDB |
| Web | `npm run test` | Component rendering, hooks |
| Web | `npm run test:e2e` | Full user flows in browser |
| Mobile | `npm run test:unit -- --passWithNoTests` | Component logic |
| Mobile | `npm run test:api -- --passWithNoTests` | API interaction tests |

---

## 18. Utility Scripts

Located in `carpooling-deployment/scripts/`:

| Script | Purpose | Usage |
|:---|:---|:---|
| `deploy-all.sh` | Trigger full deployment from CLI | `./scripts/deploy-all.sh staging` |
| `deploy.sh` | Deploy a single service | `./scripts/deploy.sh backend production` |
| `health-check.sh` | Check all services are up | `./scripts/health-check.sh` |
| `rollback.sh` | Emergency rollback | `./scripts/rollback.sh backend` |
| `setup-local.sh` | Set up local dev environment | `./scripts/setup-local.sh` |
| `test-integration.sh` | Run integration test suite | `./scripts/test-integration.sh` |

---

## 19. Troubleshooting Common Issues

### "Unexpected token '<'" Error on Vercel

**Cause:** `VITE_API_URL` is not set in Vercel's environment variables. The frontend calls itself instead of the backend, receiving HTML (`index.html`) instead of JSON.

**Fix:** Add `VITE_API_URL=https://car-pooling-system-backend-production.up.railway.app` in Vercel Dashboard → Settings → Environment Variables. Redeploy.

### Railway 502 Error

**Cause:** Server crashes during startup due to missing environment variables (`MONGO_URI`, `TWILIO_ACCOUNT_SID`, etc.) or failed database connection.

**Fix:** Verify all required env vars are set in Railway Dashboard. Check Railway deployment logs for the specific error.

### EAS Build Validation Error

**Cause:** Invalid fields in `eas.json` (e.g., `npmInstallFlags` at the wrong level).

**Fix:** Use `env` block for npm flags:
```json
{
  "build": {
    "preview": {
      "env": {
        "NPM_CONFIG_LEGACY_PEER_DEPS": "true"
      }
    }
  }
}
```

### Cross-Repo Workflow Trigger Fails

**Cause:** `GH_PAT` token expired or doesn't have `workflow` scope.

**Fix:** Generate a new token at GitHub Settings → Developer Settings → Personal Access Tokens with `repo` + `workflow` scopes. Update the secret in the deployment repo.

---

**Ride-Sharing and Sustainable Mobility Platform | Team Betamax | B.Tech CSE 2025-26**

**Amrita Vishwa Vidyapeetham**
