## Post-Deployment Workflow: Rollback & Bugfix Strategy

> **zen-pharma-backend CI/CD Architecture**  
> GitOps-based pipeline with immutable image promotion across DEV → QA → PROD  
> Built for safety, auditability, and fast incident response

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [When to Rollback vs Bugfix](#when-to-rollback-vs-bugfix)
3. [Strategy 1: Production Rollback](#strategy-1-production-rollback)
4. [Strategy 2: Production Bugfix](#strategy-2-production-bugfix)
5. [Architecture Diagrams](#architecture-diagrams)
6. [Decision Tree](#decision-tree)
7. [Timeline & SLA](#timeline--sla)

---

## Architecture Overview

### Current Deployment State

Your architecture has this structure:

```
Source Code Repository (zen-pharma-backend)
├── develop branch
│   └── Current deployed code (running in PROD)
│       └── Builds new image on push
│           └── Image promotion: DEV (auto) → QA (PR) → PROD (manual)
│
└── main branch
    └── Last stable release snapshot
        └── Updated after PROD is confirmed stable
        └── NOT used for PROD deployment


GitOps Repository (zen-gitops)
├── envs/dev/values-*.yaml
│   └── Auto-updated by CI (every develop push)
│       └── ArgoCD auto-syncs (~3 min)
│
├── envs/qa/values-*.yaml
│   └── Updated via Promotion PR (QA team merges)
│       └── ArgoCD auto-syncs on merge
│
└── envs/prod/values-*.yaml
    └── Updated via Promotion PR (Release Manager merges)
        └── ArgoCD manual sync (engineer clicks "Sync" button)
        └── Same image (immutable SHA) flows through all environments
```

### Key Principle: Image is Immutable

Once an image is built with tag `sha-abc1234`, those exact bytes flow through:
- DEV (tested)
- QA (tested)
- PROD (deployed)

No rebuild per environment = no "works in QA but fails in PROD" surprises.

---

## When to Rollback vs Bugfix

### Rollback (Go back to previous version)

**Use when:**
- PROD is broken and needs immediate recovery
- Previous version was stable and tested
- Time is critical (SLA)
- Root cause not yet known

**Example scenarios:**
- New image deployed, causing 500 errors → rollback to previous working image
- Database migration failed → rollback to pre-migration image
- Memory leak causing pod crashes → rollback to stable version

**Recovery time:** ~7 minutes

**Risk:** Minimal (reverting to known-good version)

---

### Bugfix (Deploy new code with fix)

**Use when:**
- Root cause is identified and understood
- Fix is isolated and tested locally
- Need to keep forward momentum
- Previous version unacceptable for business reasons

**Example scenarios:**
- Logic bug in token expiry (wrong TimeUnit) → fix code, rebuild, promote
- Configuration error in environment variables → fix and re-promote
- Database query bug → fix query, rebuild, promote

**Recovery time:** ~40 minutes (standard) or ~20 minutes (fast-track through QA)

**Risk:** Moderate (new code, but tested through QA)

---

## Strategy 1: Production Rollback

### High-Level Flow

```
INCIDENT: PROD Broken (sha-abc1234)
              ↓
         Identify bad PR in zen-gitops
              ↓
         GitHub: Click "Revert" button
              ↓
         Revert PR auto-created
              ↓
         Get approval & merge
              ↓
         ArgoCD detects change (~3 min)
              ↓
         Engineer syncs in ArgoCD UI
              ↓
         PROD rolls back to previous version ✓
```

### Detailed Step-by-Step

#### STEP 1: Identify the bad deployment

**In zen-gitops repo on GitHub:**

1. Go to **Pull requests → Merged**
2. Find the PROD promotion PR that broke things
   - Example: "Promote auth-service to PROD: sha-abc1234"
   - Status: Merged

3. This PR updated `envs/prod/values-auth-service.yaml`
   - Changed tag from `sha-xyz5678` → `sha-abc1234` (broken)

#### STEP 2: Click "Revert" button (GitHub does the work)

**On the merged PROD PR:**

```
[← Back]  [Revert]  [Create branch]  [Delete branch]  ...
```

Click: **Revert**

GitHub automatically:
- ✓ Creates a new branch with revert changes
- ✓ Opens a new PR with reverted values file
- ✓ Pre-fills commit message: "Revert 'Promote auth-service to PROD: sha-abc1234'"

**Revert PR opens automatically (example PR #224):**

```
Title: Revert "Promote auth-service to PROD: sha-abc1234"

Changes:
envs/prod/values-auth-service.yaml
- tag: sha-abc1234  (broken)
+ tag: sha-xyz5678  (previous stable)

Commit:
"Revert "Promote auth-service to PROD: sha-abc1234"
This reverts commit abc123def456."
```

#### STEP 3: Get approval & merge

**Release Manager (or engineer with approval rights):**

1. Review the revert PR (takes 30 seconds)
   - Verify tag reverts to previous known-good version
   - Check that it's the correct previous version

2. Approve & merge PR #224

#### STEP 4: ArgoCD detects and syncs

**Automatic (after merge):**

1. Revert commit merged to `main` in zen-gitops
2. `envs/prod/values-auth-service.yaml` now shows tag: `sha-xyz5678`
3. ArgoCD polls zen-gitops (every ~3 minutes)
4. ArgoCD detects change
5. UI shows: **OutOfSync**

**Manual (engineer action):**

1. Open ArgoCD UI
2. Find `pharma-prod` application
3. Click **"Sync"** button
4. Select the service (or Sync All)
5. ArgoCD reconciles:
   - Terminates pods running `sha-abc1234`
   - Pulls image `sha-xyz5678` from ECR
   - Starts new pods with stable image ✓

#### STEP 5: Verify rollback success

```bash
# Check ArgoCD UI
# → pharma-prod app shows "Synced" ✓
# → Service pods restarted with sha-xyz5678
# → Health checks passing

# Check application behavior
# → Logs show normal startup
# → Application endpoints responding
# → Previous issue resolved ✓
```

---

### Rollback Timeline

| Step | Duration |
|------|----------|
| Identify bad PR | ~1 min |
| Click Revert button | ~30 sec |
| Review & merge revert PR | ~2 min |
| ArgoCD polls & detects | ~3 min (worst case) |
| Engineer syncs in UI | ~1 min |
| Pods restart & stabilize | ~1 min |
| **Total** | **~7–8 minutes** |

---

### Rollback Audit Trail

Git history in zen-gitops:

```
def456abc123 [Engineer] Revert "Promote auth-service to PROD: sha-abc1234"
abc123def456 [Release Manager] Promote auth-service to PROD: sha-abc1234 (broken)
xyz789abc123 [Release Manager] Promote auth-service to PROD: sha-xyz5678 (good)
aaa111bbb222 [QA] Promote auth-service to QA: sha-xyz5678
```

Every change is:
- ✓ Attributed to an author
- ✓ Timestamped
- ✓ Reversible (git history preserved)
- ✓ Reviewable (PR review comments available)

---

## Strategy 2: Production Bugfix

### High-Level Flow

```
INCIDENT: Bug in PROD code (running sha-abc1234)
              ↓
         Create bugfix branch from develop
              ↓
         Fix the bug locally
              ↓
         Test locally (mvn verify)
              ↓
         Push to bugfix branch
              ↓
         Open PR to develop
              ↓
         Code review & merge to develop
              ↓
         CI triggers: ci-<service>.yml (full build)
              ├─ Build new image: sha-fix5678
              ├─ ECR push
              ├─ DEV auto-deploy
              └─ QA Promotion PR opens
              ↓
         QA tests new image (sha-fix5678)
              ↓
         QA approves & merges PR
              ↓
         Release Manager triggers PROD promotion
              ↓
         Promote PR created in zen-gitops
              ↓
         Get approval & merge PROD PR
              ↓
         ArgoCD syncs PROD to sha-fix5678 ✓
```

---

### Detailed Step-by-Step

#### STEP 1: Assess the bug

**When incident is declared:**

1. Understand the issue
   - What functionality is broken?
   - Is it a logic bug or configuration issue?
   - Can we rollback instead? (if previous version acceptable)

2. If bugfix is required: proceed to STEP 2

---

#### STEP 2: Create bugfix branch from develop (current PROD code)

```bash
cd zen-pharma-backend/

# Start from develop (current code running in PROD)
git checkout develop
git pull origin develop

# Create bugfix branch
# Naming convention: bugfix/<service>/<description>
git checkout -b bugfix/auth-service/token-expiry-fix
```

**Why from develop (not main)?**
- `develop` = current code deployed to PROD (contains the bug)
- `main` = previous stable release (too old, doesn't have the bug to fix)
- Bugfix must start where the bug exists = `develop`

---

#### STEP 3: Fix the bug & test locally

```bash
# Edit the source file
vim auth-service/src/main/java/com/pharma/auth/TokenProvider.java

# Before (buggy code):
# public LocalDateTime getTokenExpiry() {
#   return LocalDateTime.now().plus(1, ChronoUnit.MINUTES);  // ❌ 1 minute!
# }

# After (fixed code):
# public LocalDateTime getTokenExpiry() {
#   return LocalDateTime.now().plus(24, ChronoUnit.HOURS);   // ✓ 24 hours!
# }

# Test locally
cd auth-service
mvn verify

# Expected output:
# [INFO] BUILD SUCCESS
# [INFO] TokenProvider tests pass with 24-hour expiry ✓
```

**Testing checklist:**
- ✓ Unit tests pass
- ✓ Integration tests pass (if applicable)
- ✓ Code coverage maintained
- ✓ No new security warnings (CodeQL/Semgrep locally)

---

#### STEP 4: Commit & push bugfix branch

```bash
git add auth-service/

git commit -m "Fix: token expiry bug in TokenProvider

Changed token expiration from 1 minute to 24 hours.
Root cause: incorrect ChronoUnit in LocalDateTime calculation.

The bug was caused by using ChronoUnit.MINUTES instead of ChronoUnit.HOURS
in the getTokenExpiry() method, causing tokens to expire after 1 minute
instead of the intended 24 hours.

Testing: auth-service unit and integration tests pass with 24-hour expiry.

Fixes: #123 (production incident)
Signed-off-by: Developer <dev@pharma.io>"

git push origin bugfix/auth-service/token-expiry-fix
```

**Commit message best practices:**
- ✓ Clear subject line (what was fixed)
- ✓ Explain the root cause (why it happened)
- ✓ Show what was tested locally
- ✓ Reference the incident/issue
- ✓ Show author/sign-off

---

#### STEP 5: Open PR to develop & get code review

```bash
gh pr create \
  --base develop \
  --title "Fix: token expiry bug in auth-service" \
  --body "## Summary

Fixed critical bug in TokenProvider causing tokens to expire after 1 minute instead of 24 hours.

## Root Cause

TokenProvider.getTokenExpiry() was using ChronoUnit.MINUTES instead of ChronoUnit.HOURS.

## Changes

- TokenProvider.java: Changed ChronoUnit.MINUTES → ChronoUnit.HOURS

## Testing

- Local: mvn verify passes ✓
- Unit tests for token expiry: PASS ✓
- Integration tests: PASS ✓

## Deployment Plan

1. Merge to develop → CI builds new image
2. QA tests new image
3. Release Manager promotes to PROD

Fixes #123 (production incident)"
```

**Code review checklist:**
- ✓ Reviewer confirms bug is understood
- ✓ Fix is isolated (no unrelated changes)
- ✓ Tests pass locally
- ✓ No new security issues introduced

**Merge to develop:**

Once approved, merge PR:
```bash
gh pr merge <PR-number> --squash --delete-branch
```

---

#### STEP 6: CI/CD triggers automatically (on develop push)

**Because branch filter matches `branches: [develop, 'release/**']`:**

```yaml
on:
  push:
    branches: [develop, 'release/**']  ← Your push to develop triggers this
```

**Workflow: `ci-auth-service.yml` runs:**

```
Job 1: build
├─ Maven: compile, test, verify
├─ CodeQL SAST
├─ Semgrep SAST
├─ OWASP Dependency Check
├─ Docker build (multi-stage)
├─ Trivy image scan
└─ ECR push: sha-fix5678 (NEW image with fix)
   └─ Cosign keyless sign

Job 2: deploy-dev
└─ Commit to zen-gitops main
   envs/dev/values-auth-service.yaml
   tag: sha-fix5678
   └─ ArgoCD auto-syncs DEV (~3 min)

Job 3: open-qa-pr
└─ Create PR in zen-gitops
   envs/qa/values-auth-service.yaml
   tag: sha-abc1234 → sha-fix5678
   └─ QA Promotion PR #225 opens
```

**What you see in GitHub:**
1. Bugfix branch shows: "Checks passed" ✓
2. New commit in zen-gitops `main`: DEV values updated
3. New PR in zen-gitops: QA Promotion PR #225

**Timeline:**
- Build + tests: ~5 min
- Docker + ECR: ~5 min
- DEV deploy + QA PR: ~3 min
- Total: ~13 minutes

---

#### STEP 7: QA reviews & tests the promotion PR

**In zen-gitops repo, QA Promotion PR #225:**

```
Title: Promote auth-service to QA: sha-fix5678

Changes:
envs/qa/values-auth-service.yaml
- tag: sha-abc1234  (buggy version currently in QA)
+ tag: sha-fix5678  (fixed version)

Description:
Auth-service bugfix: token expiry issue
New image with fix ready for QA testing
```

**QA team actions:**

1. **Review the change** (30 sec)
   - Verify tag changed as expected
   - Check that this is the only change (safety)

2. **Test in QA environment** (~10 min)
   ```bash
   # QA's test plan:
   # 1. Deploy new image (PR merge will trigger ArgoCD)
   # 2. Login to test app
   # 3. Verify token expiry: should be 24 hours now
   # 4. Run regression tests: no other auth flows broken
   # 5. Check logs: no errors or warnings
   ```

3. **Approve & merge PR**
   ```bash
   # After testing succeeds:
   gh pr review <PR-225> --approve
   gh pr merge <PR-225>
   ```

**After merge:**
- ArgoCD detects `envs/qa/values-auth-service.yaml` changed
- ArgoCD auto-syncs QA environment (~3 min)
- QA pods restart with `sha-fix5678` ✓
- Token expiry bug fixed in QA ✓

**Timeline:**
- QA review: ~1 min
- QA testing: ~10 min
- Merge: ~1 min
- ArgoCD sync: ~3 min
- Total: ~15 minutes

---

#### STEP 8: Release Manager triggers PROD promotion

**After QA confirms the fix works:**

```bash
# Go to GitHub Actions → promote-prod.yml
# OR use CLI:

gh workflow run promote-prod.yml \
  --ref main \
  -F service=auth-service
```

**What happens automatically:**

```
promote-prod.yml reads QA values:
  └─ Reads envs/qa/values-auth-service.yaml
     └─ Gets: sha-fix5678

Creates PROD Promotion PR in zen-gitops:
  └─ envs/prod/values-auth-service.yaml
     - tag: sha-abc1234  (buggy, current)
     + tag: sha-fix5678  (fixed, tested in QA)

PR Details:
  Title: Promote auth-service to PROD: sha-fix5678
  Body: Shows tag change, build info, QA sign-off
```

**Timeline:**
- Trigger workflow: ~1 min
- Workflow creates PR: ~2 min
- Total: ~3 minutes

---

#### STEP 9: Get approval & merge PROD PR

**PROD Promotion PR #226 in zen-gitops:**

```
Requires approval from:
  ✓ Release Manager
  ✓ QA Lead
```

**Approval process:**

1. Release Manager reviews:
   - Confirms image tag is the QA-tested version ✓
   - Confirms business window acceptable for deployment ✓
   - Comments: "Approved for immediate deployment"

2. QA Lead reviews:
   - Confirms they tested this image ✓
   - Confirms fix works ✓
   - Comments: "QA sign-off"

3. Merge PR #226

---

#### STEP 10: ArgoCD syncs PROD

**After PR #226 merges:**

1. Git commit merged to zen-gitops `main`
2. `envs/prod/values-auth-service.yaml` now shows: `tag: sha-fix5678`
3. ArgoCD polls zen-gitops (~3 min)
4. ArgoCD detects: `envs/prod/values-auth-service.yaml` changed
5. ArgoCD UI shows: **OutOfSync**
6. Engineer clicks **"Sync"** in ArgoCD UI
7. Kubernetes:
   - Terminates pods with `sha-abc1234` (buggy)
   - Pulls image `sha-fix5678` (fixed) from ECR
   - Starts new pods ✓
   - Health checks pass ✓

**PROD is now fixed!** ✓

**Timeline:**
- Get approvals: ~5 min
- Merge PR: ~1 min
- ArgoCD polls: ~3 min (worst case)
- Engineer syncs: ~1 min
- Pods restart: ~1 min
- Total: ~11 minutes

---

#### STEP 11: (Next day) Merge bugfix back to main

After PROD is stable for ~1 hour, ensure main branch is up-to-date:

```bash
cd zen-pharma-backend/

git checkout main
git pull origin main

# Merge develop into main (now includes the bugfix)
git merge --no-ff develop \
  -m "Release: include token expiry bugfix (sha-fix5678)

Fixed critical bug in token expiration.
Tested in QA and deployed to PROD successfully.
Now syncing into release branch to keep main = current PROD code."

git push origin main
```

This ensures: `main` branch always reflects what's in PROD.

---

### Bugfix Timeline (Standard Flow)

| Phase | Duration |
|-------|----------|
| **Local fix & test** | ~15 min |
| Create branch + commit + push | ~3 min |
| Open PR + code review | ~5 min |
| CI: Build + test + ECR | ~10 min |
| QA testing | ~15 min |
| Release Manager triggers PROD | ~3 min |
| Get approvals + merge | ~5 min |
| ArgoCD sync | ~4 min |
| **Total** | **~60 minutes** |

**Can be accelerated to ~40 min if:**
- QA pre-approves while CI is building
- Release Manager waits on merge approval
- All stakeholders ready immediately

---

## Architecture Diagrams

### Diagram 1: Rollback Flow (Both Repos)

```
ROLLBACK: Using GitHub Revert Button

zen-pharma-backend                        zen-gitops
(no changes)                              ──────────────

                                          PROD PR #223 (MERGED)
                                          ├─ Status: Merged ✓
                                          ├─ Commit: abc123def456
                                          └─ Updated: envs/prod/values-auth-service.yaml
                                             tag: sha-abc1234 (BROKEN)


                                          INCIDENT DETECTED
                                          PROD returning 500 errors
                                                   │
                                                   ▼

                                          Open PR #223 on GitHub
                                          Click: "Revert" button
                                                   │
                                                   ▼

                                          GitHub auto-creates
                                          Revert PR #224
                                          
                                          ├─ Title: "Revert 'Promote auth-service to PROD...'"
                                          ├─ Changes:
                                          │  envs/prod/values-auth-service.yaml
                                          │  - tag: sha-abc1234  (broken)
                                          │  + tag: sha-xyz5678  (restore good)
                                          │
                                          └─ Commit msg auto-generated:
                                             "Revert 'Promote auth-service to PROD...'
                                              This reverts commit abc123def456."

                                                   │
                                                   ▼

                                          Release Manager
                                          ├─ Reviews (30 sec)
                                          ├─ Approves
                                          └─ Merges PR #224

                                          Git history:
                                          ├── def456abc123 Revert "Promote..."
                                          ├── abc123def456 Promote auth-service (broken)
                                          └── xyz789abc123 Promote auth-service (good)
                                                   │
                                                   ▼

                                          ArgoCD polls zen-gitops
                                          (every ~3 minutes)
                                          
                                          Detects: envs/prod/values-auth-service.yaml
                                          changed to sha-xyz5678
                                                   │
                                                   ▼

                                          ArgoCD UI shows:
                                          pharma-prod: OutOfSync
                                                   │
                                                   ▼

                                          Engineer opens ArgoCD UI
                                          Clicks: "Sync" button
                                                   │
                                                   ▼

                                          Kubernetes reconciles:
                                          ├─ Terminates pods (sha-abc1234)
                                          ├─ Pulls image (sha-xyz5678)
                                          └─ Starts new pods ✓

                                          PROD RECOVERED ✓
                                          Total time: ~7 min
```

---

### Diagram 2: Bugfix Flow (Both Repos)

```
BUGFIX: Create from develop, Deploy through standard flow

zen-pharma-backend                        zen-gitops
──────────────────                        ──────────────

develop (PROD code with bug)              
├─ sha-abc1234 (deployed to PROD)         envs/prod/values-auth-service.yaml
├─ 🐛 Bug: token expires at 1 min         tag: sha-abc1234 (PROD running)
└─ Other untested changes
                  │
                  ▼

BUG REPORTED IN PROD
                  │
                  ▼

git checkout develop
git pull
git checkout -b bugfix/auth-service/token-expiry
                  │
                  ▼

vim TokenProvider.java
# Change: ChronoUnit.MINUTES → ChronoUnit.HOURS

mvn verify  ✓
                  │
                  ▼

git commit -m "Fix: token expiry..."
git push origin bugfix/auth-service/token-expiry
                  │
                  ▼

gh pr create --base develop
(opens PR to develop for code review)
                  │
                  ▼

Code Review ✓
Merge to develop
                  │
                  ▼

GitHub Actions: ci-auth-service.yml TRIGGERS
(because push to develop matches branch filter)
                  │
    ┌─────────────┼─────────────┐
    │             │             │
    ▼             ▼             ▼

Job 1:        Job 2:          Job 3:
build         deploy-dev      open-qa-pr
├─Maven       └─Commit to     └─Create QA
├─CodeQL        zen-gitops      Promotion
├─Docker         main            PR in
├─Trivy          │               zen-gitops
└─ECR push       ▼               
  sha-fix5678  envs/dev/        Promotion
              values-*          PR #225
              tag:              (OPEN)
              sha-fix5678       │
              │                 ▼
              ▼             QA reviews +
            ArgoCD          tests
            auto-syncs      sha-fix5678
            DEV (~3 min)    ✓
                            │
                            ▼
                            Merge PR #225
                            │
                            ▼
                            ArgoCD
                            syncs QA
                            to sha-fix5678 ✓

                                          ──────────────
                                          
                                          After QA confirms:

                                          Release Manager triggers:
                                          gh workflow run promote-prod.yml \
                                            -F service=auth-service
                                                    │
                                                    ▼

                                          promote-prod.yml reads QA tag
                                          (sha-fix5678)
                                          
                                          Creates PROD PR #226
                                          (zen-gitops)
                                          │
                                          ├─ Title: "Promote auth-service..."
                                          ├─ Changes:
                                          │  envs/prod/values-auth-service.yaml
                                          │  - tag: sha-abc1234
                                          │  + tag: sha-fix5678
                                          │
                                          └─ Requires: Release Manager + QA Lead
                                          
                                          Get approvals & merge
                                                    │
                                                    ▼

                                          ArgoCD detects change
                                          Shows: OutOfSync
                                                    │
                                                    ▼

                                          Engineer: Click "Sync"
                                                    │
                                                    ▼

                                          Kubernetes reconciles:
                                          ├─ Terminates pods (sha-abc1234)
                                          ├─ Pulls image (sha-fix5678)
                                          └─ Starts new pods ✓

                                          PROD FIXED ✓
                                          Total time: ~60 min
```

---

### Diagram 3: Comparison Overview

```
┌────────────────────────────────────────────────────────────────┐
│ INCIDENT IN PRODUCTION                                         │
│ (something is broken)                                          │
└──────────────────────┬───────────────────────────────────────┘
                       │
         Is previous version acceptable?
                       │
        ┌──────────────┴──────────────┐
        │ YES                         │ NO
        ▼                             ▼
    ROLLBACK                      BUGFIX
    ────────                      ──────
    
    GitHub Revert PR              Create bugfix/ branch
    (zen-gitops)                  from develop
         │                             │
         ▼                             ▼
    Revert old tag                Fix code
    (30 sec action)               Test locally
         │                             │
         ▼                             ▼
    Merge & approve               Open PR to develop
    (2 min)                        Code review
         │                             │
         ▼                             ▼
    ArgoCD syncs                   Merge to develop
    (3 min)                        CI builds (10 min)
         │                             │
         ▼                             ▼
    PROD restored                  QA tests (15 min)
         │                             │
    ✓ 7 min total                  PROD promote (5 min)
                                       │
                                       ▼
                                   ArgoCD syncs
                                       │
                                   ✓ 60 min total
```

---

## Decision Tree

```
┌─────────────────────────────────────────────────────────┐
│ PRODUCTION INCIDENT RESPONSE DECISION TREE              │
└─────────────────────────────────────────────────────────┘

START: Something is broken in PROD
  │
  ├─ Q1: Do we know the root cause?
  │
  ├─ NO → Q2: Was previous version working?
  │  │
  │  ├─ YES → ROLLBACK ✓
  │  │   └─ Time: ~7 min
  │  │   └─ Risk: Very low
  │  │   └─ Action: GitHub Revert button
  │  │
  │  └─ NO → Investigate first, then decide
  │      └─ Check logs, metrics, recent changes
  │
  └─ YES → Q3: Is the root cause a code bug?
     │
     ├─ YES → BUGFIX ✓
     │   ├─ Can you fix quickly? (< 30 min dev time)
     │   │  ├─ YES → Bugfix
     │   │  │    └─ Time: ~60 min
     │   │  │    └─ Risk: Low (tested through QA)
     │   │  │    └─ Action: bugfix/ branch from develop
     │   │  │
     │   │  └─ NO → Consider fast-track
     │   │       └─ Skip QA (~20 min, requires pre-approval)
     │   │
     │   └─ Is it critical enough to skip QA?
     │        ├─ YES → Fast-track bugfix (20 min)
     │        └─ NO → Standard bugfix (60 min)
     │
     └─ NO → Other issues
         ├─ Configuration error? → Update env vars
         ├─ Resource exhaustion? → Scale up, then fix
         ├─ External service down? → Rollback or wait
         └─ Other? → Investigate with SRE team
```

---

## Timeline & SLA

### Rollback SLA

```
Scenario: PROD broken, previous version stable

Timeline:
├─ Detect incident: 0–2 min (monitoring/alerts)
├─ Identify root cause: 1–2 min
├─ Decide rollback: 1 min
├─ Click GitHub Revert: 30 sec
├─ Review & merge: 2 min
├─ ArgoCD polls: 3 min (worst case)
├─ Engineer syncs: 1 min
├─ Pods restart: 1 min
└─ Total: 7–13 minutes

SLA Target: Recovery under 15 minutes
Success Rate: 99.9% (only fails if git/GitHub down)
```

---

### Bugfix SLA

```
Scenario: Bug identified, fix is known and tested locally

Timeline:
├─ Local fix & test: 15 min
├─ Create PR to develop: 3 min
├─ Code review: 5 min
├─ CI build: 10 min
├─ QA test: 15 min
├─ Release Manager triggers: 3 min
├─ Get approvals: 5 min
├─ ArgoCD sync: 4 min
└─ Total: ~60 minutes

Accelerated (pre-approvals):
├─ Remove QA wait (do testing in parallel): ~20 min saved
├─ Release Manager waits on CI: no extra wait
└─ Total: ~40 minutes

SLA Target: Bugfix deployed in < 90 min
Success Rate: 98% (depends on QA availability + approvals)
```

---

### When to Use Each Strategy

| Factor | Rollback | Bugfix |
|--------|----------|--------|
| **Time to recovery** | 7–13 min | 40–60 min |
| **Risk level** | Very low | Low–moderate |
| **Need code change?** | No | Yes |
| **QA testing required?** | No | Yes |
| **Approvals needed** | 1 (Release Manager) | 3+ (Dev, QA, Release) |
| **Best for** | Unknown issues, need quick recovery | Identified bugs, can wait for QA |
| **Worst for** | Bugs in current code | Broken previous version |

---

## Operational Checklists

### Pre-Incident Preparation

- [ ] Ensure team knows how to use GitHub's Revert button
- [ ] Ensure Release Manager has approval rights in both repos
- [ ] Ensure QA Lead has approval rights in zen-gitops
- [ ] Verify ArgoCD is set to manual sync for PROD
- [ ] Ensure engineers have ArgoCD UI access
- [ ] Document on-call escalation path

### Rollback Checklist

- [ ] Identify the bad PROD PR in zen-gitops
- [ ] Verify previous version was stable
- [ ] Click GitHub "Revert" button (automatic)
- [ ] Review revert PR (30 sec)
- [ ] Merge revert PR (1 person)
- [ ] Wait for ArgoCD to detect (~3 min)
- [ ] Click "Sync" in ArgoCD UI
- [ ] Verify PROD health (metrics, logs, basic tests)
- [ ] Post-mortem: why did the previous deploy break?

### Bugfix Checklist

- [ ] Reproduce the bug locally
- [ ] Create bugfix branch from `develop` (not main)
- [ ] Write minimal fix (no refactoring)
- [ ] Test locally: `mvn verify` passes
- [ ] Commit with clear message
- [ ] Open PR to develop (triggers code review)
- [ ] Wait for CI build (~10 min)
- [ ] QA reviews + tests in QA environment (~15 min)
- [ ] QA approves & merges
- [ ] Release Manager triggers PROD promotion
- [ ] Get 2 approvals (Release Manager + QA Lead)
- [ ] Merge PROD PR
- [ ] Wait for ArgoCD (~3 min)
- [ ] Click "Sync" in ArgoCD
- [ ] Verify PROD health
- [ ] (Next day) Merge develop → main

---

## FAQ

**Q: Why create bugfix from develop, not main?**

A: Your PROD is deployed from `develop`, not `main`. The bug exists in the code running in PROD, which is `develop`. Creating from `main` means starting from old code that doesn't contain the bug or the bug-causing code.

---

**Q: What if develop has other untested changes?**

A: QA will test the new image, which includes all changes currently in develop. If those untested changes break things, QA doesn't approve the QA PR, and you must either:
- Fix those changes
- Revert them from develop
- Wait for them to stabilize

This is a trade-off: you can merge your bugfix immediately to develop, but you risk promoting other changes. If you need isolation, you can:
- Create release/hotfix branch from develop
- Cherry-pick only your bugfix commit
- Push to release/hotfix
- CI triggers (matches release/**)
- Promote only the fix, avoiding other untested changes

---

**Q: Can we skip QA for critical bugs?**

A: Technically yes, but not recommended. GitHub's environment protection rules require approval from Release Manager + QA Lead on PROD. You can:
- Fast-track if both approve immediately (phone call)
- Get verbal sign-off, then merge
- But it bypasses testing → higher risk of cascading failures

Use only for bugs where you're 100% certain of the fix.

---

**Q: What if Git/GitHub is down?**

A: You can't rollback or promote through normal flow. Fallback:
- Manual kubectl patch (requires cluster access)
- Direct edit of ConfigMap
- But you lose audit trail and traceability

This is rare. Better to have strong redundancy/HA for GitHub and zen-gitops.

---

**Q: How do we prevent bugs from reaching PROD?**

A: Multiple gates:
1. **Local testing** (developer)
2. **CI security gates** (SAST, SCA)
3. **Code review** (another developer)
4. **QA testing** (QA environment, real test suite)
5. **PROD approval** (Release Manager + QA Lead)

The more gates, the slower. This architecture balances safety with speed.

---

## Summary

| Scenario | Strategy | Time | Risk |
|----------|----------|------|------|
| PROD broken, previous version stable | Rollback (GitHub Revert) | ~7 min | ✓✓✓ Low |
| Bug identified, fix tested locally | Bugfix (develop → QA → PROD) | ~60 min | ✓✓ Low–Moderate |
| Critical bug, need fast fix | Bugfix (fast-track, skip QA) | ~20 min | ✓ Moderate–High |
| Unknown issue, need to investigate | Neither (investigate first) | Variable | Depends |

---

## Related Documentation

- `CI-ARCHITECTURE.md` — Full CI/CD pipeline details
- `promote-prod.yml` — Manual PROD promotion workflow
- `ci-<service>.yml` — Per-service build & deployment workflows
- ArgoCD docs: https://argo-cd.readthedocs.io

---

**Last updated:** 2026-05-01  
**Maintained by:** Zen Pharma Backend Team  
**Contact:** DevOps / Release Manager
