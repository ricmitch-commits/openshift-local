# MLflow ConfigChange Trigger Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make newly created MLflow BuildConfigs start their initial image build automatically.

**Architecture:** Add the native OpenShift `ConfigChange` trigger to the existing BuildConfig. Keep image construction and deployment references unchanged, then let Argo CD apply the published manifest and verify the live trigger.

**Tech Stack:** OpenShift BuildConfig, Argo CD, YAML, Git

---

### Task 1: Add and publish the build trigger

**Files:**
- Modify: `manifests/mlflow/buildconfig.yaml:6-10`
- Reference: `docs/superpowers/specs/2026-09-01-mlflow-configchange-design.md`

- [ ] **Step 1: Verify the trigger assertion fails before the edit**

Run:

```bash
grep -A1 '^  triggers:' manifests/mlflow/buildconfig.yaml | grep -q 'type: ConfigChange'
```

Expected: exit status 1 because `spec.triggers` is absent.

- [ ] **Step 2: Add the minimal ConfigChange trigger**

Insert this block immediately below `spec:`:

```yaml
  triggers:
    - type: ConfigChange
```

- [ ] **Step 3: Verify the manifest and assertion**

Run:

```bash
grep -A1 '^  triggers:' manifests/mlflow/buildconfig.yaml | grep -q 'type: ConfigChange'
oc apply --dry-run=server -f manifests/mlflow/buildconfig.yaml
git diff --check -- manifests/mlflow/buildconfig.yaml
```

Expected: all commands exit 0 and the server dry run reports `buildconfig.build.openshift.io/mlflow configured`.

- [ ] **Step 4: Commit the implementation and plan**

Run:

```bash
git add manifests/mlflow/buildconfig.yaml docs/superpowers/plans/2026-09-01-mlflow-configchange.md
git diff --cached --check
git commit -m "Trigger initial mlflow image build"
```

Expected: the commit contains only the BuildConfig trigger and this implementation plan.

- [ ] **Step 5: Publish to the configured upstream**

Run:

```bash
git push origin main
```

Expected: `origin/main` advances to the new commit.

### Task 2: Verify Argo CD applies the trigger

**Files:**
- Verify: `manifests/mlflow/buildconfig.yaml`

- [ ] **Step 1: Request an Argo CD refresh**

Run:

```bash
oc annotate application mlflow -n openshift-gitops argocd.argoproj.io/refresh=hard --overwrite
```

Expected: the Application is annotated successfully.

- [ ] **Step 2: Wait for the published revision and successful sync**

Run:

```bash
oc wait application/mlflow -n openshift-gitops --for="jsonpath={.status.sync.revision}=$(git rev-parse HEAD)" --timeout=180s
oc wait application/mlflow -n openshift-gitops --for=jsonpath='{.status.operationState.phase}'=Succeeded --timeout=180s
```

Expected: both wait commands report that their condition was met.

- [ ] **Step 3: Verify the live trigger and sync state**

Run:

```bash
oc get buildconfig mlflow -n mlflow -o jsonpath='triggers={.spec.triggers}{"\n"}'
oc get application mlflow -n openshift-gitops -o jsonpath='sync={.status.sync.status}{"\n"}revision={.status.sync.revision}{"\n"}'
```

Expected: the BuildConfig reports `ConfigChange`, the Application reports `sync=Synced`, and its revision matches `git rev-parse HEAD`.
