# Phase 5 — CI/CD

## Objective

Verify the Daig CI/CD pipeline so that failed tests block image builds and integration, workflow actions are pinned to immutable commit SHAs, `latest` publishing is restricted to the default branch, and deployment follows a guarded canary → rollout → rollback flow.

---

## 1. CI Job Dependency Graph

The CI workflow is defined in `.github/workflows/ci.yml`.

The dependency flow is:

```text
static checks ─────┐
                   ├──> build images ───> integration / full-stack verification
unit tests ────────┘
```

The `build` job requires both `static` and `test` to succeed:

```yaml
needs: [static, test]
```

Therefore:

- Static-check failure blocks the build.
- Unit-test failure blocks the build.
- The build must succeed before downstream integration verification.
- The dependency graph acts as a quality gate before the later CI stages.

### Evidence

- `34-ci-dependency-graph.png` — CI job dependency graph.
- `35-ci-build-push-pinned.png` — build job, pinned actions, and conditional image push.

---

## 2. Build vs Push

The CI workflow uses Docker Buildx and the Docker Build/Push action.

The build stage uses:

```yaml
uses: docker/build-push-action@<commit-sha>
```

Image publishing is conditional:

```yaml
push: ${{ github.event_name == 'push' }}
```

The image metadata includes an immutable commit-SHA tag and a controlled `latest` tag:

```yaml
type=raw,value=${{ github.sha }}
type=raw,value=latest,enable={{is_default_branch}}
```

### Evidence

- `35-ci-build-push-pinned.png`
- `36-ci-latest-gate.png`

---

## 3. Broken Unit Test — Red Pipeline

The orders service health test was intentionally changed from:

```javascript
assert.strictEqual(total, 65500);
```

to:

```javascript
assert.strictEqual(total, 65501);
```

This created an intentional unit-test failure.

The local test run showed the expected value did not match the actual value:

```text
Expected: 65501
Actual:   65500
```

The GitHub Actions run then demonstrated the CI gate:

```text
Static checks        ✓
Unit tests - orders  ✗
Build image          skipped
Integration          skipped
```

This proves that a failing unit test prevents the later pipeline stages from running.

### Evidence

- `37-broken-test.png`
- `38-local-broken-test.png`
- `39-ci-broken-test-gate.png`

> Note: The test exercise was performed on `main` rather than an isolated feature branch. The required CI gate behavior was still demonstrated by the red pipeline.

---

## 4. Test Fix — Green Pipeline

The incorrect assertion was restored:

```javascript
assert.strictEqual(total, 65500);
```

The local orders test suite then passed.

### Evidence

- `40-test-fixed-local.png`
- `41-ci-green-after-fix.png`

The green CI run verified:

- Static checks passed.
- Orders unit tests passed.
- Docker image builds passed.
- Full-stack integration passed.

The demonstrated gate is:

```text
bad test → CI red → build/integration blocked

fixed test → CI green → build/integration allowed
```

---

## 5. Supply-Chain Hygiene — SHA-Pinned Actions

All workflow `uses:` entries were checked with:

```bash
grep -RniE '^[[:space:]]*uses:' .github/workflows/
```

The workflow actions are pinned to commit SHAs rather than floating branch/tag references.

The inspected workflows include pinned actions for Docker, SonarQube, Gitleaks, Semgrep, Trivy, Checkov, SBOM generation, Cosign, ZAP, CodeQL, and related CI/security tooling.

Using commit SHAs prevents an action from silently changing underneath the workflow because a mutable tag or branch was moved.

### `latest` Publishing Gate

The CI metadata configuration contains:

```yaml
type=raw,value=${{ github.sha }}
type=raw,value=latest,enable={{is_default_branch}}
```

Therefore:

- Images have an immutable commit-SHA tag.
- `latest` is enabled only for the repository's default branch.
- Other branches do not publish the mutable `latest` application tag.

### Evidence

- `35-ci-build-push-pinned.png`
- `36-ci-latest-gate.png`

---

## 6. CD — Progressive Delivery Flow

The deployment workflow is defined in `.github/workflows/cd.yml`.

The declared flow is:

```text
Deployment window guard
          ↓
      Canary 10%
          ↓
      SLO watch 5 minutes
          ↓
    Full rollout 100%
          ↓
   Record deployment

If unhealthy:
          ↓
       Rollback
          ↓
  Previous revision
```

### Canary

The `canary` job:

- Requires the deployment-window guard.
- Deploys to 10% of traffic.
- Watches the SLO for 5 minutes.

### Full Rollout

The `rollout` job:

- Depends on the canary job.
- Promotes the deployment to 100%.
- Records the deployment.

### Rollback

The `rollback` job is defined to run when a failure occurs and revert to the previous revision.

The workflow explicitly requires:

```text
Rollback must be one step and must not require rebuild.
```

### Important Implementation Note

The current `cd.yml` is a deployment workflow skeleton containing TODO deployment/watch/record operations. Phase 5 evidence is based on inspection of the declared workflow. It does **not** claim that a real production deployment was executed.

### Evidence

- `42-cd-no-cancel.png`
- `43-cd-iftar-guard.png`
- `44-cd-rollback.png`

---

## 7. Deployment Window / Iftar Guard

The CD workflow contains a deployment-window guard for the iftar peak period.

The guard blocks deployment during:

```text
16:00–21:00 PKT
```

This is implemented directly in the workflow by checking the hour and exiting with an error during the blocked period.

This is a safety control that makes an unsafe deployment window impossible rather than relying on an engineer to remember not to deploy.

### Evidence

- `43-cd-iftar-guard.png`

---

## 8. Concurrency Protection

The CD workflow contains:

```yaml
concurrency:
  group: cd-${{ github.event.inputs.environment || 'production' }}
  cancel-in-progress: false
```

The important setting is:

```yaml
cancel-in-progress: false
```

This prevents a newer workflow run from automatically canceling an already-running deployment.

A deployment can be in the middle of:

```text
Canary → SLO watch → Full rollout
```

If that run were canceled halfway through, the environment could be left in an uncertain or partially rolled-out state.

Keeping `cancel-in-progress: false` protects rollout consistency and allows the active deployment to finish or fail cleanly.

### Evidence

- `42-cd-no-cancel.png`

---

## 9. Why Rollback One Step and No Rebuild?

Rollback should use the previously deployed revision rather than rebuilding the application.

Reasons:

1. **Speed** — recovery is faster.
2. **Reproducibility** — the previous artifact was already built and validated.
3. **Lower incident risk** — no new build is introduced while production is unhealthy.
4. **Predictability** — rollback returns to a known revision.
5. **Fast recovery** — recovery during an incident should minimize service disruption.

In short:

```text
Incident
   ↓
Rollback to known revision
   ↓
Service recovery
```

instead of:

```text
Incident
   ↓
Rebuild
   ↓
Test
   ↓
Publish
   ↓
Deploy
```

---

## 10. Phase 5 Verification Summary

| Requirement | Result | Evidence |
|---|---|---|
| CI dependency graph identified | PASS | `34-ci-dependency-graph.png` |
| Build vs push identified | PASS | `35-ci-build-push-pinned.png` |
| Intentional unit-test failure | PASS | `37-broken-test.png`, `38-local-broken-test.png` |
| Failed test blocks later stages | PASS | `39-ci-broken-test-gate.png` |
| Test fixed | PASS | `40-test-fixed-local.png` |
| CI green after fix | PASS | `41-ci-green-after-fix.png` |
| Actions SHA pinned | PASS | `35-ci-build-push-pinned.png` |
| `latest` default-branch gate | PASS | `36-ci-latest-gate.png` |
| Canary → rollout → rollback flow reviewed | PASS | `42-cd-no-cancel.png`, `44-cd-rollback.png` |
| Iftar deployment guard reviewed | PASS | `43-cd-iftar-guard.png` |
| `cancel-in-progress: false` reviewed | PASS | `42-cd-no-cancel.png` |

---

## 11. Final Checkpoint Answers

### Q1. Why rollback one step and no rebuild?

Because rollback should return to an already-built, previously validated revision. Rebuilding during an incident adds delay and uncertainty. A one-step rollback is faster, safer, and more predictable.

### Q2. What does `concurrency: cancel-in-progress: false` protect?

It prevents a newer CD run from canceling an active rollout. This protects the deployment from being interrupted halfway through canary or full rollout and helps maintain a consistent production state.

---

## Phase 5 Status

**PASS — CI/CD controls and evidence completed.**
