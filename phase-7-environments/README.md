# Phase 7 — Environments and Deployment Gates

> **Concepts GitLab CI introduced:** `environment`, `when: manual`, protected environments, deployment approvals, rollback | **Cost:** $0

---

## The problem

Lumio's `Jenkinsfile.deploy` gates production deployments with a single `input` step:

```groovy
stage('Deploy to production') {
    steps {
        input message: 'Approve deploy to production?', ok: 'Deploy'
        sh './scripts/deploy.sh production'
    }
}
```

This pattern has three compounding problems. First, the gate is held by whoever is watching the Jenkins UI — there is no structured approval list, no required reviewer, no role enforcement. Any engineer with pipeline run access can click "Proceed." Second, there is no audit trail. Jenkins records that the `input` step was acknowledged, but not who approved it, from which IP, or in what context. Third, the Jenkins job stays in a "pending" state while waiting for approval, holding an executor slot and contributing to queue pressure on the three-agent cluster.

When a production incident happens at 2 AM and you need to roll back, the answer today is: manually SSH into the Kubernetes cluster and run `kubectl rollout undo`. There is no GitLab Environments equivalent. There is no deployment history. There is no one-click rollback.

GitLab CI has a native environments system that solves all of this. Deployments are tracked as first-class objects — you can see the current deployed revision, the history of every deployment, who triggered it, and roll back to any previous version in one click. Protected environments enforce who can deploy. Deployment approvals enforce a structured review. Freeze windows block deployments during periods your team defines.

---

## Jenkins vs GitLab CI: the deployment gate comparison

```mermaid
flowchart TD
    subgraph jenkins["Jenkins — Jenkinsfile.deploy"]
        JC["git push"] --> JB["Jenkins build triggers"]
        JB --> JS["stage: Deploy to staging\nsh deploy.sh staging"]
        JS --> JI["input step\n'Approve deploy to production?'\nJob hangs — holds executor slot"]
        JI -->|"Anyone with run access clicks Proceed"| JP["stage: Deploy to production\nsh deploy.sh production"]
        JP --> JX["Job complete\nNo audit trail\nNo rollback UI"]
    end

    subgraph gitlab["GitLab CI — .gitlab-ci.yml"]
        GC["git push"] --> GB["Pipeline triggers"]
        GB --> GS["deploy-staging job\nenvironment: staging\nauto-runs on main"]
        GS --> GD["GitLab records deployment\nwho, when, commit SHA, status"]
        GD --> GM["deploy-prod job\nwhen: manual\nenvironment: production"]
        GM -->|"Only Maintainers\nwith required approvals"| GP["Deployment runs\nProtected environment enforced"]
        GP --> GR["Full audit trail\nOne-click rollback\nDeployment history"]
    end
```

---

## What you will build

By the end of this phase, Lumio's deployment pipeline for `lumio-api` will:

- Track every staging and production deployment in GitLab's Environments UI
- Require manual approval with a structured approver list before any production deployment runs
- Restrict production deployments to Maintainers via Protected Environments
- Support one-click rollback from the GitLab UI
- Gate database migrations before application deployment with a verify step
- Block deployments during configured freeze windows

---

## Before You Begin — Provisioning a Local Kubernetes Cluster

The `kubectl` commands in this phase require a running Kubernetes cluster reachable by GitLab CI runners. The fastest zero-cost path is **Rancher Desktop**, which ships with a built-in Kubernetes cluster (k3s) and also handles container builds — no separate Docker Desktop licence required.

### 1. Install Rancher Desktop and enable Kubernetes

1. Download and install Rancher Desktop from [rancherdesktop.io](https://rancherdesktop.io)
2. On first launch, select your container runtime:
   - Choose **dockerd (moby)** if you want `docker` CLI compatibility
   - Choose **containerd** for a lighter footprint (kubectl still works either way)
3. Kubernetes is enabled by default — wait for the status indicator in the bottom-left corner to turn green (takes 1–3 minutes on first launch while k3s images are pulled)
4. Verify the cluster is running:

```bash
kubectl cluster-info
# Expected:
# Kubernetes control plane is running at https://127.0.0.1:6443
# CoreDNS is running at https://127.0.0.1:6443/api/v1/namespaces/...
```

```bash
kubectl get nodes
# Expected:
# NAME                   STATUS   ROLES                  AGE   VERSION
# lima-rancher-desktop   Ready    control-plane,master   ...   v1.x.x+k3s1
```

```bash
# Confirm the active context
kubectl config current-context
# Expected: rancher-desktop
```

### 2. Create the namespaces

```bash
kubectl create namespace lumio-staging
kubectl create namespace lumio-production

# Verify
kubectl get namespaces | grep lumio
```

### 3. Create a GitLab registry pull secret

The cluster needs credentials to pull images from `registry.gitlab.com`. Create a Deploy Token first:

1. In **lumio-api → Settings → Repository → Deploy tokens**, create a token with `read_registry` scope
2. Note the username and token value — the token is only shown once

Then create the secret in both namespaces:

```bash
kubectl create secret docker-registry gitlab-registry --docker-server=registry.gitlab.com --docker-username=<deploy-token-username> --docker-password=<deploy-token-value> --namespace=lumio-staging && kubectl create secret docker-registry gitlab-registry --docker-server=registry.gitlab.com --docker-username=<deploy-token-username> --docker-password=<deploy-token-value> --namespace=lumio-production
```

> The manifests in `kubernetes/` already reference `imagePullSecrets: gitlab-registry` — the secret name must match exactly.

### 4. Create the initial deployment

The `kubectl set image` commands in this phase require a deployment to already exist. The `kubernetes/` directory in this phase contains ready-made manifests — apply them directly:

```bash
kubectl apply -f phase-7-environments/kubernetes/lumio-api-staging.yaml
kubectl apply -f phase-7-environments/kubernetes/lumio-api-production.yaml
```

Wait for both deployments to become ready:

```bash
kubectl rollout status deployment/lumio-api -n lumio-staging
kubectl rollout status deployment/lumio-api -n lumio-production
```

Verify the pods and services are up:

```bash
kubectl get all -n lumio-staging
kubectl get all -n lumio-production
```

Expected output (for each namespace):

```
NAME                             READY   STATUS    RESTARTS   AGE
pod/lumio-api-xxxxxxxxx-xxxxx    1/1     Running   0          30s

NAME                TYPE        CLUSTER-IP      PORT(S)    AGE
service/lumio-api   ClusterIP   10.96.x.x       3000/TCP   30s

NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/lumio-api   1/1     1            1           30s
```

> The manifests use a lightweight `node:18-alpine` placeholder that serves a simple HTTP response on port 3000. The `kubectl set image` commands in the pipeline challenges will swap this out for the real image tag once CI is wired up.

### 5. Expose the Kubernetes API to GitLab via ngrok

GitLab SaaS runners cannot reach `https://127.0.0.1:6443`. Use ngrok to create a public tunnel to the Kubernetes API server:

```bash
ngrok tcp 6443
```

ngrok will print a forwarding address like:

```
Forwarding  tcp://0.tcp.ngrok.io:12345 -> localhost:6443
```

> **Important:** TCP tunnels require a paid ngrok account. If you have a free account, use the **GitLab Agent for Kubernetes** approach in step 6 instead — it is free and does not require exposing the API publicly.

### 6. Configure kubectl to use the ngrok endpoint

Export a modified kubeconfig that points to the ngrok address instead of localhost:

```bash
# Get the rancher-desktop context kubeconfig and replace the API server address
kubectl config view --raw \
  | sed 's|https://127.0.0.1:6443|https://0.tcp.ngrok.io:12345|g' \
  > /tmp/kubeconfig-ngrok.yaml

# Base64-encode it for the GitLab CI variable
cat /tmp/kubeconfig-ngrok.yaml | base64 -w 0
```

Copy the base64 output and create a GitLab CI variable:

| Key | Value | Type | Masked |
|---|---|---|---|
| `KUBECONFIG_STAGING` | `<base64 output>` | Variable | Yes |
| `KUBECONFIG_PRODUCTION` | `<base64 output>` | Variable | Yes |

Add this `before_script` to your deploy jobs to decode the kubeconfig:

```yaml
deploy-staging:
  before_script:
    - mkdir -p ~/.kube
    - echo "$KUBECONFIG_STAGING" | base64 -d > ~/.kube/config
```

### 7. Alternative — GitLab Agent for Kubernetes (recommended, free)

The GitLab Agent for Kubernetes (`agentk`) runs inside your cluster and maintains an outbound connection to GitLab — no public exposure of the Kubernetes API is needed. This is the production-recommended approach and works with Rancher Desktop out of the box.

> **Why this instead of ngrok TCP?** ngrok TCP tunnels require a credit card on file even on the free tier (`ERR_NGROK_8013`). The GitLab agent has no such requirement and is the production-recommended pattern regardless.

**Step 1 — Create the agent config file in your `lumio-api` repo**

The agent name is derived from the directory name under `.gitlab/agents/`. Create it before registering in the UI:

```bash
mkdir -p .gitlab/agents/lumio-agent
cat > .gitlab/agents/lumio-agent/config.yaml << 'EOF'
# Empty config is valid — enables the agent to connect to GitLab
EOF

git add .gitlab/agents/lumio-agent/config.yaml
git commit -m "chore: register GitLab Kubernetes agent config"
git push origin main
```

**Step 2 — Register the agent in GitLab**

1. Navigate to **lumio-api > Operate > Kubernetes clusters**
2. Click **Connect a cluster (agent)**
3. Select `lumio-agent` from the dropdown (it reads from `.gitlab/agents/` in your repo)
4. Click **Register** and copy the token shown — it is only displayed once

**Step 3 — Install the agent into your cluster**

```bash
# Add the GitLab Helm repository
helm repo add gitlab https://charts.gitlab.io
helm repo update

# Create a namespace for the agent
kubectl create namespace gitlab-agent

# Install the agent (replace <TOKEN> with the token from step 2)
helm upgrade --install lumio-agent gitlab/gitlab-agent \
  --namespace gitlab-agent \
  --set config.token=<TOKEN> \
  --set config.kasAddress=wss://kas.gitlab.com
```

**Step 4 — Verify the agent is connected**

```bash
kubectl get pods -n gitlab-agent
# Expected:
# NAME                                        READY   STATUS    RESTARTS   AGE
# lumio-agent-gitlab-agent-xxxxxxxxx-xxxxx   1/1     Running   0          30s
```

Back in GitLab under **Operate > Kubernetes clusters**, the agent should show a green connected status within a few seconds.

**Step 5 — Use the agent context in your pipeline**

```yaml
deploy-staging:
  image: alpine/k8s:1.29.14
  script:
    - kubectl config use-context lumio4615817/lumio-api:lumio-staging-agent
    - kubectl set image deployment/lumio-api lumio-api=$IMAGE -n lumio-staging
```

> The agent context name follows the format `<gitlab-namespace>/<project>:<agent-name>`. GitLab injects it automatically into the runner environment when the agent is connected — no kubeconfig variable needed. `$IMAGE` is the variable passed from the build job via `build.env` artifacts.

### 8. Verify end-to-end connectivity

Add a debug job to confirm the runner can reach the cluster before running deployments:

```yaml
kubectl-check:
  stage: build
  image: alpine/k8s:1.29.14
  script:
    - kubectl config use-context lumio4615817/lumio-api:lumio-staging-agent
    - kubectl cluster-info
    - kubectl get nodes
    - kubectl get namespaces | grep lumio
```

If this job passes, the runner can communicate with your cluster and the deploy jobs in the challenges below will work.

---

## Challenge 1 — Create GitLab environments and track deployments

**Goal:** Replace the untracked `sh deploy.sh` calls with proper `environment:` declarations so GitLab records every deployment.

### Step 1 — Add environment declarations to your `.gitlab-ci.yml`

```yaml
stages:
  - build
  - test
  - deploy-staging
  - deploy-production

variables:
  KUBE_CONTEXT_STAGING: "lumio4615817/lumio-api:lumio-staging-agent"
  KUBE_CONTEXT_PROD: "lumio4615817/lumio-api:lumio-prod-agent"
  APP_IMAGE: "$CI_REGISTRY_IMAGE/lumio-api:$CI_COMMIT_SHORT_SHA"

# --- Build stage (from Phase 6) ---
build:
  stage: build
  image: docker:24
  services:
    - docker:24-dind
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker build -t $APP_IMAGE .
    - docker push $APP_IMAGE
  only:
    - main

# --- Deploy to staging (automatic on every main push) ---
deploy-staging:
  stage: deploy-staging
  image: alpine/k8s:1.29.14
  environment:
    name: staging
    url: https://staging.lumio.internal
  script:
    - kubectl config use-context $KUBE_CONTEXT_STAGING
    - kubectl set image deployment/lumio-api lumio-api=$APP_IMAGE -n lumio-staging
    - kubectl rollout status deployment/lumio-api -n lumio-staging --timeout=120s
  only:
    - main

# --- Deploy to production (manual gate) ---
deploy-production:
  stage: deploy-production
  image: alpine/k8s:1.29.14
  environment:
    name: production
    url: https://app.lumio.io
  when: manual
  script:
    - kubectl config use-context $KUBE_CONTEXT_PROD
    - kubectl set image deployment/lumio-api lumio-api=$APP_IMAGE -n lumio-production
    - kubectl rollout status deployment/lumio-api -n lumio-production --timeout=180s
  only:
    - main
```

### Step 2 — Push a commit and observe the Deployments UI

```bash
git add .gitlab-ci.yml
git commit -m "chore: add environment declarations for staging and production"
git push origin main
```

### Step 3 — Navigate to the Environments UI

In your GitLab project, go to **Deploy > Environments**. After the pipeline runs you will see:

```
Environments

staging          Last deployed: just now    lumio-api @ a3f9c12    ● running
production       Never deployed             —
```

Click on `staging` to see the deployment history:

```
Deployments — staging

#1   a3f9c12   Deploy lumio-api 1.4.2   Will Dubois   just now   ● success
```

### Step 4 — Verify the environment URL is live

Click the external link icon next to `staging` — GitLab opens `https://staging.lumio.internal` directly from the Environments panel.

**What just happened:** Every `kubectl set image` is now a recorded deployment event in GitLab. The commit SHA, the pipeline link, the triggering user, and the timestamp are all captured. You did not write any of that instrumentation — `environment:` gave it to you for free.

---

## Challenge 2 — Add manual approval for production

**Goal:** Replace the `input` Jenkinsfile gate with a proper `when: manual` job that any authorized user can trigger from the pipeline UI, with a full audit trail.

### Step 1 — The `.gitlab-ci.yml` already has `when: manual` on `deploy-production`

The `when: manual` keyword from Challenge 1 is the GitLab equivalent of Jenkins `input`. The difference: GitLab records who clicked Play, at what time, from which pipeline.

### Step 2 — Trigger a full deployment cycle

Push a change that bumps the API version:

```bash
# In lumio-app/api/package.json, bump the version
sed -i '' 's/"version": "1.4.2"/"version": "1.4.3"/' lumio-app/api/package.json
git add lumio-app/api/package.json
git commit -m "feat: release 1.4.3 — new bulk export endpoint"
git push origin main
```

### Step 3 — Watch staging deploy automatically

In the **CI/CD > Pipelines** view you will see the pipeline progress:

```
Pipeline #1847 — feat: release 1.4.3 — new bulk export endpoint

● build          ✓ passed    2m 14s
● test           ✓ passed    4m 38s
● deploy-staging ✓ passed    1m 02s
▶ deploy-production  [manual]  — click to run
```

The `▶` icon means the job is waiting for a human to trigger it.

### Step 4 — Approve the production deployment

Click the `▶` button next to `deploy-production`. GitLab will prompt:

```
Are you sure you want to run deploy-production?
This job will deploy to environment: production

[ Cancel ]  [ Run job ]
```

Click **Run job**.

### Step 5 — Check the audit trail

After the deployment completes, go to **Deploy > Environments > production**. You will see:

```
Deployments — production

#1   a3f9c13   feat: release 1.4.3 — new bulk export endpoint
     Triggered by: Will Dubois   Approved via: pipeline play   2026-04-22 14:32:07 UTC
     ● success
```

Compare this to what Jenkins recorded: nothing. The `input` step in Jenkins has no audit log unless you install a separate plugin and wire it up manually.

---

## Challenge 3 — Configure Protected Environments

**Goal:** Enforce that only Maintainers can deploy to production, and require two approvals before the `deploy-production` job can run.

### Step 1 — Enable Protected Environments in GitLab

Navigate to **Settings > CI/CD > Protected environments** and click **Protect an environment**.

```
Environment name:   production
Allowed to deploy:  Maintainers
Required approvals: 2
```

Click **Protect**.

### Step 2 — Observe the enforcement

Now create a test user with Developer role in your project, and try to trigger `deploy-production` as that user. GitLab will return:

```
403 Forbidden

You do not have permission to manually trigger this job.
Only users with the Maintainer role or above can deploy to the
protected environment: production.
```

### Step 3 — Configure deployment approvals (GitLab Ultimate / trial)

If you have GitLab Ultimate or an active trial, go to **Settings > CI/CD > Protected environments > production** and enable **Deployment approvals**:

```
Require approvals before deployment:   ✓ enabled
Number of required approvals:          2
Approval rules:
  ● Any eligible approver (Maintainer+)
```

### Step 4 — Trigger a deployment and observe the approval flow

Push a new commit to main. When the pipeline reaches `deploy-production`, the job will show:

```
deploy-production   ⏸ awaiting approval   (0 of 2 approvals)
```

In **Deploy > Environments > production**, an approval panel appears:

```
Deployment approval required

Pipeline #1851 wants to deploy to production.
Commit: b7d1e44 — fix: patch XSS in export filename handling

Approvals (0 / 2):
  ○ —
  ○ —

[ Approve ]  [ Reject ]
```

Two Maintainers must each click **Approve**. After the second approval:

```
Approvals (2 / 2):
  ✓ Will Dubois     2026-04-22 16:04:11 UTC
  ✓ Sara Chen       2026-04-22 16:05:43 UTC

Deployment started automatically after final approval.
```

### Step 5 — Test rejection

Have one Maintainer click **Reject** instead. The `deploy-production` job transitions to:

```
deploy-production   ✗ failed   Deployment rejected by Sara Chen: "deploy after 17:00 UTC only"
```

The pipeline marks the job as failed with the rejection reason recorded in the audit log.

---

## Challenge 4 — Implement rollback

**Goal:** Configure a `rollback-production` job that redeploys the previous image tag, and demonstrate GitLab's one-click rollback from the Environments UI.

### Step 1 — Add a rollback job to `.gitlab-ci.yml`

```yaml
rollback-production:
  stage: deploy-production
  image: alpine/k8s:1.29.14
  environment:
    name: production
    action: start
  when: manual
  allow_failure: false
  variables:
    ROLLBACK_IMAGE: ""   # set at trigger time via CI/CD variables
  script:
    - |
      if [ -z "$ROLLBACK_IMAGE" ]; then
        echo "ERROR: ROLLBACK_IMAGE must be set. Example: registry.gitlab.com/lumio/api:a3f9c12"
        exit 1
      fi
    - kubectl config use-context $KUBE_CONTEXT_PROD
    - kubectl set image deployment/lumio-api lumio-api=$ROLLBACK_IMAGE -n lumio-production
    - kubectl rollout status deployment/lumio-api -n lumio-production --timeout=180s
    - echo "Rolled back to $ROLLBACK_IMAGE"
  only:
    - main
```

### Step 2 — Use GitLab's built-in re-deploy button

GitLab tracks every successful deployment. Navigate to **Deploy > Environments > production**. Click the clock icon to expand deployment history:

```
Deployments — production

#4   b7d1e44   fix: patch XSS in export filename             ● success   16:07 UTC   [ Re-deploy ] [ Rollback ]
#3   a3f9c13   feat: release 1.4.3 — new bulk export          ● success   14:32 UTC   [ Re-deploy ] [ Rollback ]
#2   ff2a801   chore: update dependencies                     ● success   09:15 UTC   [ Re-deploy ] [ Rollback ]
#1   c1d3b99   Initial deploy                                 ● success   yesterday   [ Re-deploy ] [ Rollback ]
```

Click **Re-deploy** on deployment `#3` (release 1.4.3). GitLab re-runs the exact pipeline job that created that deployment, with the same image tag, the same environment configuration, and the same approval gates.

### Step 3 — Verify the rollback in Kubernetes

```bash
kubectl get deployment lumio-api -n lumio-production -o jsonpath='{.spec.template.spec.containers[0].image}'
```

Expected output:
```
registry.gitlab.com/lumio/lumio-api:a3f9c13
```

### Step 4 — Confirm the rollback appears in the deployment history

```
Deployments — production

#5   a3f9c13   Rollback to: feat: release 1.4.3               ● success   16:22 UTC
               Triggered by: Will Dubois (rollback)
#4   b7d1e44   fix: patch XSS in export filename              ● failed    16:18 UTC
```

GitLab marks the intent as "rollback" in the deployment log so the audit trail is unambiguous.

---

## Challenge 5 — Migrate the `Jenkinsfile.db-migrate`

**Goal:** Replace Lumio's Flyway migration Jenkins job with a GitLab CI pipeline that follows the pattern: `migrate → verify → deploy`, with the ability to roll back the migration if verify fails.

### The original `Jenkinsfile.db-migrate`

```groovy
pipeline {
    agent { label 'docker' }
    parameters {
        string(name: 'TARGET_ENV', defaultValue: 'staging')
    }
    stages {
        stage('Run Flyway migration') {
            steps {
                withCredentials([string(credentialsId: "db-url-${params.TARGET_ENV}", variable: 'DB_URL')]) {
                    sh "flyway -url=$DB_URL -user=$DB_USER -password=$DB_PASS migrate"
                }
            }
        }
        stage('Verify schema') {
            steps {
                sh 'npm run db:verify'
            }
        }
        stage('Deploy app') {
            steps {
                build job: 'lumio-api/deploy', parameters: [string(name: 'ENV', value: params.TARGET_ENV)]
            }
        }
    }
    post {
        failure {
            sh "flyway -url=$DB_URL repair"
        }
    }
}
```

### Step 1 — Add the migration stages to `.gitlab-ci.yml`

```yaml
stages:
  - build
  - test
  - migrate
  - verify-migration
  - deploy-staging
  - deploy-production

# --- Database migration ---
db-migrate-staging:
  stage: migrate
  image: flyway/flyway:10
  environment:
    name: staging
    action: prepare  # signals a pre-deployment step, not the deploy itself
  variables:
    FLYWAY_URL: "$DB_URL_STAGING"
    FLYWAY_USER: "$DB_USER_STAGING"
    FLYWAY_PASSWORD: "$DB_PASSWORD_STAGING"
  script:
    - flyway migrate
    - echo "Migration complete. Current version:"
    - flyway info | tail -5
  artifacts:
    reports:
      dotenv: migration.env   # pass migration metadata to next jobs
    paths:
      - flyway-output.txt
    expire_in: 1 day
  only:
    - main

# --- Verify schema after migration ---
verify-migration-staging:
  stage: verify-migration
  image: node:20-alpine
  needs: ["db-migrate-staging"]
  environment:
    name: staging
  variables:
    DATABASE_URL: "$DB_URL_STAGING"
  script:
    - npm ci --prefix lumio-app/api
    - npm run db:verify --prefix lumio-app/api
  after_script:
    - |
      if [ $CI_JOB_STATUS == "failed" ]; then
        echo "Schema verification failed — triggering Flyway repair"
        docker run --rm flyway/flyway:10 \
          -url=$DB_URL_STAGING \
          -user=$DB_USER_STAGING \
          -password=$DB_PASSWORD_STAGING \
          repair
      fi
  only:
    - main

# --- App deploy only runs after migration verified ---
deploy-staging:
  stage: deploy-staging
  image: alpine/k8s:1.29.14
  needs: ["verify-migration-staging", "build"]
  environment:
    name: staging
    url: https://staging.lumio.internal
  script:
    - kubectl config use-context $KUBE_CONTEXT_STAGING
    - kubectl set image deployment/lumio-api lumio-api=$APP_IMAGE -n lumio-staging
    - kubectl rollout status deployment/lumio-api -n lumio-staging --timeout=120s
  only:
    - main
```

### Step 2 — Push a migration and observe the pipeline

Create a new Flyway migration file:

```bash
mkdir -p lumio-app/api/db/migrations
cat > lumio-app/api/db/migrations/V9__add_export_audit_table.sql << 'EOF'
CREATE TABLE export_audit (
    id          BIGSERIAL PRIMARY KEY,
    user_id     BIGINT NOT NULL REFERENCES users(id),
    export_type VARCHAR(64) NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_export_audit_user_id ON export_audit(user_id);
EOF

git add .
git commit -m "feat: add export_audit table migration V9"
git push origin main
```

Expected pipeline view:

```
Pipeline #1855

● build                      ✓ passed    2m 14s
● test                       ✓ passed    4m 38s
● db-migrate-staging         ✓ passed    0m 47s
● verify-migration-staging   ✓ passed    1m 12s
● deploy-staging             ✓ passed    1m 04s
▶ deploy-production          [manual]
```

### Step 3 — Observe what happens when verification fails

```bash
# Introduce a bad migration that breaks schema verification
cat > lumio-app/api/db/migrations/V10__bad_migration.sql << 'EOF'
ALTER TABLE users DROP COLUMN email;  -- breaks the db:verify check
EOF

git add . && git commit -m "test: intentionally bad migration" && git push origin main
```

Expected pipeline result:

```
● db-migrate-staging         ✓ passed    0m 52s
✗ verify-migration-staging   ✗ failed    0m 31s
                              Schema check failed: users.email column not found
                              Running Flyway repair...

✗ deploy-staging             skipped (upstream job failed)
```

The `deploy-staging` job never runs because `needs: ["verify-migration-staging"]` creates a hard dependency. The `after_script` in `verify-migration-staging` runs `flyway repair` automatically.

---

## Challenge 6 (Bonus) — Configure deployment freeze windows

**Goal:** Block all production deployments on weekends and between 21:00–06:00 UTC, matching Lumio's on-call schedule.

### Step 1 — Create a freeze window via the GitLab API

```bash
# Set your variables
GITLAB_URL="https://gitlab.com"
PROJECT_ID="42"          # replace with your project ID
PRIVATE_TOKEN="glpat-xxxxxxxxxxxxxxxxxxxx"

# Create freeze window: Friday 21:00 UTC to Monday 06:00 UTC
curl --request POST \
  --header "PRIVATE-TOKEN: $PRIVATE_TOKEN" \
  --header "Content-Type: application/json" \
  --data '{
    "freeze_start": "0 21 * * 5",
    "freeze_end": "0 6 * * 1",
    "cron_timezone": "UTC"
  }' \
  "$GITLAB_URL/api/v4/projects/$PROJECT_ID/freeze_periods"
```

Expected response:

```json
{
  "id": 1,
  "freeze_start": "0 21 * * 5",
  "freeze_end": "0 6 * * 1",
  "cron_timezone": "UTC",
  "created_at": "2026-04-22T14:00:00.000Z",
  "updated_at": "2026-04-22T14:00:00.000Z"
}
```

### Step 2 — Add a nightly freeze window

```bash
# Nightly freeze: every day 21:00–06:00 UTC (Mon–Thu)
curl --request POST \
  --header "PRIVATE-TOKEN: $PRIVATE_TOKEN" \
  --header "Content-Type: application/json" \
  --data '{
    "freeze_start": "0 21 * * 0-4",
    "freeze_end": "0 6 * * 1-5",
    "cron_timezone": "UTC"
  }' \
  "$GITLAB_URL/api/v4/projects/$PROJECT_ID/freeze_periods"
```

### Step 3 — Attempt a deployment during a freeze

Try to trigger `deploy-production` during a configured freeze window. GitLab will block the job with:

```
This job could not be triggered because the project is in a deploy freeze period.

Freeze periods active:
  ● 0 21 * * 5 → 0 6 * * 1 (UTC) — weekend freeze
  ● 0 21 * * 0-4 → 0 6 * * 1-5 (UTC) — nightly freeze

The job will remain in a blocked state until the freeze period ends.
Next available deployment window: Monday 06:00 UTC
```

### Step 4 — Verify freeze periods in the UI

Navigate to **Settings > CI/CD > Deploy freezes**:

```
Deploy freeze periods

# Freeze period 1
  Freeze start:   0 21 * * 5   (Friday 21:00 UTC)
  Freeze end:     0 6 * * 1    (Monday 06:00 UTC)
  Timezone:       UTC

# Freeze period 2
  Freeze start:   0 21 * * 0-4 (Mon-Thu 21:00 UTC)
  Freeze end:     0 6 * * 1-5  (Tue-Fri 06:00 UTC)
  Timezone:       UTC
```

---

## Outcome

After completing this phase, Lumio's deployment pipeline for `lumio-api` replaces the fragile Jenkins `input` step with a production-grade deployment system:

| Jenkins `Jenkinsfile.deploy` | GitLab CI equivalent |
|---|---|
| `input 'Approve deploy to production?'` | `when: manual` + Protected Environment approvals |
| No role enforcement | Protected Environments (Maintainer only) |
| No audit trail | Full deployment history in Environments UI |
| Manual SSH for rollback | One-click Re-deploy from Environments UI |
| No freeze windows | Configurable freeze periods via API |
| `build job: 'deploy'` (cross-job trigger) | `needs:` with inline environment gate |

Deployments are now first-class objects in GitLab. Every deployment is traceable to a commit, a pipeline, a user, and a timestamp. Rolling back to any previous version is a single click. Approvals are enforced at the platform level, not via a Groovy script that someone can comment out.

---

[Back to main README](../README.md) | [Previous: Phase 6 — Docker and Registry](../phase-6-docker-registry/README.md) | [Next: Phase 8 — GitLab-Native Features](../phase-8-gitlab-native/README.md)
