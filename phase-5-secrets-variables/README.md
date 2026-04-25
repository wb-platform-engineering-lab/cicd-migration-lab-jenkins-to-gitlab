# Phase 5 — Secrets and Variables: Securing Credentials

> **Concepts GitLab CI introduced:** CI/CD Variables (masked, protected, env-scoped), HashiCorp Vault integration, Group variables | **Cost:** $0

---

## The problem

The security audit of Lumio's Jenkins setup found **14 jobs with credentials stored in plain text** inside their job configuration XML files. Some of these are AWS access keys. Some are database URLs with passwords embedded. One is a Stripe API key in a webhook notification job.

Jenkins `withCredentials([])` is a powerful mechanism — it injects credentials as environment variables scoped to a block, then scrubs them from the log. But it relies on the Jenkins credential store, which:
- Stores credentials encrypted in `$JENKINS_HOME/credentials.xml` on the Jenkins controller
- Is accessible to anyone with administrator access to Jenkins
- Has no concept of environment scoping (staging vs. production) built into the pipeline syntax
- Has no way to enforce "this credential can only be used on protected branches"

The audit also found that three credentials were being passed as plain-text `parameters {}` in Jenkinsfiles so that they could be overridden at build time — meaning they appear in every build's parameter list visible to any Jenkins user.

The mandate from the security team: **zero credentials in code, zero plain-text values visible in logs, environment-scoped secrets enforced at the platform level**.

---

## GitLab CI variable hierarchy

GitLab CI variables form a four-level hierarchy. Variables at lower levels override variables at higher levels. This is how you enforce "staging uses this database URL, production uses that one" without duplicating your pipeline definition.

```mermaid
flowchart TD
    I["Instance Variables\n(GitLab admin panel)\nAvailable to all projects\non this GitLab instance"]
    G["Group Variables\n(Group > Settings > CI/CD)\nShared across all projects\nin the group — set once,\ninherited everywhere"]
    P["Project Variables\n(Project > Settings > CI/CD)\nProject-specific secrets\nCan override group variables\nwith the same name"]
    J["Job-level variables:\nblock in .gitlab-ci.yml\nPlaintext only — use for\nnon-secret configuration"]

    I --> G --> P --> J

    style I fill:#e8f4f8,stroke:#2196F3
    style G fill:#e8f5e9,stroke:#4CAF50
    style P fill:#fff3e0,stroke:#FF9800
    style J fill:#fce4ec,stroke:#E91E63
```

**Override rules:**
- A Project variable with the same name as a Group variable wins.
- A job-level `variables:` block with the same name as a Project variable wins.
- Protected variables are only injected into jobs running on protected branches or tags — they are invisible to feature branches.
- Masked variables are never printed in job logs, even if your script explicitly tries to `echo` them.

---

## Challenge 1 — Create project variables and test masking

**Goal:** Create `DOCKER_REGISTRY_USER`, `DOCKER_REGISTRY_PASSWORD`, and `API_KEY` as GitLab CI project variables, and verify that masked variables cannot be leaked to logs.

### Steps

1. Navigate to your `lumio-api` project: **Settings > CI/CD > Variables**. Expand the Variables section and click **Add variable**.

2. Create the following variables:

| Key | Value | Masked | Protected | Notes |
|---|---|---|---|---|
| `DOCKER_REGISTRY_USER` | `lumio-ci` | No | Yes | Username is not sensitive, but only needed on protected branches |
| `DOCKER_REGISTRY_PASSWORD` | `<your-registry-token>` | **Yes** | Yes | Masked + protected: only on main/release branches |
| `API_KEY` | `lm_live_xxxxxxxxxxxx` | **Yes** | No | Masked but not protected: needed on all branches for integration tests |

> **Masking requirement:** GitLab can only mask a variable if its value is at least 8 characters and contains no newlines. Values that do not meet these requirements cannot be masked — use a base64-encoded version or a Vault reference instead.

3. Create a test job to verify masking behavior:

```yaml
# .gitlab-ci.yml — masking verification job
test-masking:
  stage: test
  script:
    - echo "Testing masked variable behavior"
    - echo "API_KEY value is: $API_KEY"              # Masked: will show [MASKED]
    - echo "Docker user is: $DOCKER_REGISTRY_USER"  # Not masked: will show lumio-ci
    - printenv | grep -E "API_KEY|DOCKER"           # Try to leak via printenv
```

**Expected output (job log):**
```
$ echo "Testing masked variable behavior"
Testing masked variable behavior
$ echo "API_KEY value is: $API_KEY"
API_KEY value is: [MASKED]
$ echo "Docker user is: $DOCKER_REGISTRY_USER"
Docker user is: lumio-ci
$ printenv | grep -E "API_KEY|DOCKER"
API_KEY=[MASKED]
DOCKER_REGISTRY_USER=lumio-ci
DOCKER_REGISTRY_PASSWORD=[MASKED]
```

4. Now try to run the same job from a feature branch. Verify that `DOCKER_REGISTRY_PASSWORD` (which is protected) is **not injected at all** on non-protected branches:

```yaml
test-protected-check:
  stage: test
  script:
    - |
      if [ -z "$DOCKER_REGISTRY_PASSWORD" ]; then
        echo "DOCKER_REGISTRY_PASSWORD is not available on this branch (expected for non-protected branches)"
      else
        echo "DOCKER_REGISTRY_PASSWORD is available"
      fi
```

**Expected output on `feat/phase-5` branch:**
```
$ if [ -z "$DOCKER_REGISTRY_PASSWORD" ]; then
>   echo "DOCKER_REGISTRY_PASSWORD is not available on this branch (expected for non-protected branches)"
> else
>   echo "DOCKER_REGISTRY_PASSWORD is available"
> fi
DOCKER_REGISTRY_PASSWORD is not available on this branch (expected for non-protected branches)
```

**Expected output on `main` branch:**
```
DOCKER_REGISTRY_PASSWORD is available
```

> **Security implication:** Protected variables prevent a scenario where a developer opens a merge request with a Jenkinsfile that exfiltrates `$STRIPE_SECRET_KEY` to an external server. On GitLab, that script would run on a non-protected branch where `STRIPE_SECRET_KEY` is simply not injected.

---

## Challenge 2 — Migrate `withCredentials` from Jenkins

**Goal:** Translate the Jenkins `withCredentials([usernamePassword(...)])` block to the GitLab CI equivalent, and verify that credentials never appear in logs.

### The Jenkins original

```groovy
// jenkins/jobs/lumio-api/Jenkinsfile — credential injection
pipeline {
  agent any
  stages {
    stage('Push Image') {
      steps {
        withCredentials([
          usernamePassword(
            credentialsId: 'docker-registry-credentials',
            usernameVariable: 'REGISTRY_USER',
            passwordVariable: 'REGISTRY_PASS'
          )
        ]) {
          sh '''
            echo "$REGISTRY_PASS" | docker login registry.lumio.io \
              --username "$REGISTRY_USER" \
              --password-stdin
            docker push registry.lumio.io/lumio-api:${BUILD_NUMBER}
          '''
        }
        // Credentials are unset after the block closes
      }
    }
    stage('Notify') {
      steps {
        withCredentials([string(credentialsId: 'webhook-url', variable: 'WEBHOOK_URL')]) {
          sh "curl -s -X POST $WEBHOOK_URL -d '{\"text\": \"Build ${BUILD_NUMBER} pushed\"}'"
        }
      }
    }
  }
}
```

### The GitLab CI equivalent

```yaml
# .gitlab-ci.yml — credential injection via CI/CD variables
# Variables DOCKER_REGISTRY_USER, DOCKER_REGISTRY_PASSWORD, WEBHOOK_URL
# are defined in Settings > CI/CD > Variables (masked + protected)

stages:
  - build
  - push
  - notify

default:
  image: docker:26

push-image:
  stage: push
  services:
    - docker:26-dind
  variables:
    DOCKER_TLS_CERTDIR: "/certs"
  script:
    # Variables are injected by GitLab — no explicit credential block needed
    - echo "$DOCKER_REGISTRY_PASSWORD" | docker login $CI_REGISTRY
        --username "$DOCKER_REGISTRY_USER"
        --password-stdin
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA
  # Credentials are NEVER in the YAML file
  # They exist only in Settings > CI/CD > Variables

notify:
  stage: notify
  image: alpine:3.19
  when: always
  script:
    # For the lab: echo the notification payload to verify variable injection.
    # In a real pipeline replace the echo with a curl to your webhook endpoint
    # (Slack, Teams, PagerDuty, etc.) using $WEBHOOK_URL from CI/CD variables.
    - |
      echo "Notification payload:"
      echo "  Pipeline : $CI_PIPELINE_URL"
      echo "  Project  : $CI_PROJECT_NAME"
      echo "  Commit   : $CI_COMMIT_SHORT_SHA"
      echo "  Author   : $GITLAB_USER_LOGIN"
      echo "  Status   : $CI_JOB_STATUS"
    # Real webhook call (commented out — requires WEBHOOK_URL variable):
    # apk add --no-cache curl
    # curl -s -X POST "$WEBHOOK_URL" \
    #   --header "Content-Type: application/json" \
    #   --data "{\"text\": \"Pipeline $CI_PIPELINE_ID for $CI_PROJECT_NAME finished ($CI_JOB_STATUS)\"}"
```

**Key differences:**

| Jenkins | GitLab CI |
|---|---|
| `withCredentials([...])` block scopes the variable | Variable is available for the entire job automatically |
| Credential ID references a Jenkins credential store entry | Variable name references a GitLab CI variable |
| Credential type must be declared (`usernamePassword`, `string`, etc.) | All variables are strings; multi-value credentials are split into separate variables |
| Credentials unset after the block | Variables are unset after the job ends |
| Log scrubbing is automatic for values inside the `withCredentials` block | Log scrubbing is automatic for all masked variables |

3. Run the `push-image` job and verify the log scrubbing:

**Expected output (job log):**
```
$ echo "$DOCKER_REGISTRY_PASSWORD" | docker login $CI_REGISTRY --username "$DOCKER_REGISTRY_USER" --password-stdin
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.

Login Succeeded
$ docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA
The push refers to repository [registry.gitlab.com/lumio/lumio-api]
a3f2d91c4b88: Pushed
...
2.4.1-a3f2d91c: digest: sha256:abc123... size: 1234
```

Notice that `$DOCKER_REGISTRY_PASSWORD` never appears — it is passed via stdin and the variable value is masked in any log line where GitLab detects it.

The `notify` job demonstrates that `$WEBHOOK_URL` (if set as a masked variable) would also never appear in the log — GitLab replaces it with `[MASKED]` anywhere it is detected, including inside curl arguments.

---

## Challenge 3 — Environment-scoped variables

**Goal:** Configure `DATABASE_URL` with different values for `staging` and `production` environments using GitLab's `environment_scope`.

### Steps

1. Navigate to **Settings > CI/CD > Variables**. Add two variables with the same key but different scopes:

| Key | Value | Environment scope | Masked | Protected |
|---|---|---|---|---|
| `DATABASE_URL` | `postgres://lumio:dev-pass@staging-db.internal:5432/lumio_staging` | `staging` | Yes | No |
| `DATABASE_URL` | `postgres://lumio:prod-pass@prod-db.internal:5432/lumio_production` | `production` | Yes | Yes |

The `*` wildcard scope means "all environments not matched by a more specific scope". You can also use `staging/*` to match any sub-environment like `staging/us-east-1`.

2. Write the pipeline that deploys to both environments, referencing the same variable name:

```yaml
stages:
  - build
  - deploy

build:
  stage: build
  image: node:20-alpine
  script:
    - npm ci && npm run build
  artifacts:
    paths: [dist/]

deploy-staging:
  stage: deploy
  image: node:20-alpine
  environment:
    name: staging
    url: https://staging.lumio.io
  script:
    - echo "Deploying to staging"
    - echo "Database: $DATABASE_URL"    # Gets staging value
    - npm run db:migrate
    - npm run deploy
  rules:
    - if: $CI_COMMIT_BRANCH == "main"

deploy-production:
  stage: deploy
  image: node:20-alpine
  environment:
    name: production
    url: https://app.lumio.io
  when: manual
  script:
    - echo "Deploying to production"
    - echo "Database: $DATABASE_URL"    # Gets production value
    - npm run db:migrate
    - npm run deploy
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
```

3. Trigger a pipeline on `main`. In the `deploy-staging` job log, verify that the staging `DATABASE_URL` is injected (masked in the output):

**Expected output — deploy-staging log:**
```
$ echo "Deploying to staging"
Deploying to staging
$ echo "Database: $DATABASE_URL"
Database: [MASKED]
$ npm run db:migrate
Migrating database at staging-db.internal...
Applied 2 pending migrations.
```

**Expected output — deploy-production log (after manual trigger):**
```
$ echo "Deploying to production"
Deploying to production
$ echo "Database: $DATABASE_URL"
Database: [MASKED]
$ npm run db:migrate
Migrating database at prod-db.internal...
No pending migrations.
```

4. Verify that the correct database is being used by checking the migration output hostname — it will differ between staging and production in the migration runner logs, even though the variable name is identical in the YAML.

> **Environment scope vs. branch-based conditionals:** You could achieve similar results with `rules: - if: $CI_COMMIT_BRANCH == "main"` and two different variable names. Environment scoping is better because it decouples the variable value from the pipeline logic — the YAML says "use `$DATABASE_URL`" and the environment controls which value is injected. This mirrors the principle of least privilege: the pipeline code does not need to know the credential value, only the credential name.

---

## Challenge 4 — Group variables for shared secrets

**Goal:** Move `SLACK_WEBHOOK_URL` and `SENTRY_DSN` to Group variables so all Lumio projects inherit them without copy-pasting.

### The problem with project-level duplication

If you set `SLACK_WEBHOOK_URL` in `lumio-api`, `lumio-frontend`, and `lumio-worker` separately, you have three copies to rotate when the webhook URL changes. The same applies to `SENTRY_DSN`, `DATADOG_API_KEY`, and any other cross-cutting observability credential.

### Steps

1. Navigate to your GitLab group (e.g., `lumio` group). Go to **Settings > CI/CD > Variables**.

2. Add the following group-level variables:

| Key | Value | Masked | Protected | Notes |
|---|---|---|---|---|
| `WEBHOOK_URL` | `https://your-notification-endpoint/...` | Yes | No | Notification webhook for all branches (Slack, Teams, PagerDuty, etc.) |
| `SENTRY_DSN` | `https://xxx@o123.ingest.sentry.io/456` | No | No | Not sensitive, just configuration |
| `DATADOG_API_KEY` | `dxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx` | Yes | No | Metrics for all services |

> **No notification service configured?** Use a free request-inspection tool like [webhook.site](https://webhook.site) to get a temporary URL — it captures and displays incoming HTTP requests so you can verify the curl payload without a real integration.

3. Remove the `WEBHOOK_URL` variable from individual project settings (if you set it there earlier). Verify it is gone from **lumio-api > Settings > CI/CD > Variables**.

4. Run any pipeline in `lumio-api` that uses `$WEBHOOK_URL`. It should still work, now inheriting the group variable:

**Expected output — notify job:**
```
Fetching variables...
  Group variables: WEBHOOK_URL [masked], SENTRY_DSN, DATADOG_API_KEY [masked]
  Project variables: DOCKER_REGISTRY_USER, DOCKER_REGISTRY_PASSWORD [masked], API_KEY [masked]

$ echo "Notification payload:"
Notification payload:
  Pipeline : https://gitlab.com/lumio4615817/lumio-api/-/pipelines/123
  Project  : lumio-api
  Commit   : a3f2d91c
  Author   : cloudxscalr
  Status   : success
Job succeeded
```

5. Test the override behavior: add a project-level variable `SENTRY_DSN` with a different value in `lumio-api`. Then run a pipeline and verify the project value overrides the group value:

```yaml
debug-vars:
  stage: test
  script:
    - echo "SENTRY_DSN=$SENTRY_DSN"
    # If project-level SENTRY_DSN exists, it will override the group-level one
```

**Expected output:**
```
$ echo "SENTRY_DSN=$SENTRY_DSN"
SENTRY_DSN=https://yyy@o123.ingest.sentry.io/789   ← project-level value wins
```

6. Verify the inheritance order works correctly for all three Lumio services by running the `debug-vars` job in each project:

| Project | `SLACK_WEBHOOK_URL` source | `SENTRY_DSN` source |
|---|---|---|
| lumio-api | Group variable | Project variable (override) |
| lumio-frontend | Group variable | Group variable |
| lumio-worker | Group variable | Group variable |

> **Important note:** Sub-group variables are inherited by child groups and projects. Instance variables (set by GitLab admins) are available to all projects on the instance. The override order is always: **job-level `variables:`** > **project** > **group** > **parent group** > **instance**.

---

## Challenge 5 — HashiCorp Vault integration (advanced)

**Goal:** Configure the GitLab CI `secrets:` keyword to fetch a secret from HashiCorp Vault using JWT authentication, replacing the Jenkins `withCredentials([vaultSecret(...)])` pattern.

### Before You Begin — Running a Vault Dev Server

This challenge requires a running HashiCorp Vault instance reachable by GitLab SaaS shared runners. The fastest path is a local Vault dev server exposed publicly via ngrok.

#### 1. Start the dev server

Dev mode runs Vault entirely in memory — no TLS, no storage backend, no unsealing needed. Data is lost when the process stops. It is only suitable for local testing.

You can run the dev server either with Docker (no installation needed) or with the Vault CLI directly.

---

**Option A — Docker (recommended, no installation needed)**

If you have Docker running, this is the fastest path:

```bash
docker run --rm --name vault-dev \
  -p 8200:8200 \
  -e VAULT_DEV_ROOT_TOKEN_ID=root \
  -e VAULT_DEV_LISTEN_ADDRESS=0.0.0.0:8200 \
  -e VAULT_DISABLE_MLOCK=true \
  --entrypoint vault \
  hashicorp/vault:latest \
  server -dev
```

The `--entrypoint vault` flag bypasses the image's wrapper shell script (`docker-entrypoint.sh`), which calls `setcap` to grant the Vault binary `IPC_LOCK` capability at startup. That `setcap` call requires `CAP_SETFCAP`, which Docker Desktop on macOS and most restricted Linux hosts do not allow. Bypassing it and passing `server -dev` directly avoids the permission error entirely. `VAULT_DISABLE_MLOCK=true` then tells Vault not to attempt memory locking at runtime either. The container runs in the foreground — keep this terminal open.

In a **second terminal**, set environment variables so subsequent CLI commands work:

```bash
export VAULT_ADDR='http://127.0.0.1:8200'
export VAULT_TOKEN="root"
```

You do not need the Vault CLI installed to run the server — Docker pulls the image automatically. You will need the CLI (or use `docker exec`) to run the configuration commands in steps 1–2 of the challenge. To run Vault CLI commands through Docker instead of installing locally:

```bash
# Equivalent to: vault status
docker exec vault-dev vault status

# Equivalent to: vault secrets list
docker exec vault-dev vault secrets list

# Pass environment variables when using docker exec
docker exec -e VAULT_TOKEN=root vault-dev vault auth enable jwt
```

---

**Option B — Vault CLI (install locally)**

**macOS (Homebrew):**
```bash
brew tap hashicorp/tap
brew install hashicorp/tap/vault
vault version   # Vault v1.x.x
```

**Linux:**
```bash
wget -O - https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install vault
```

**Windows:** Download the binary from [developer.hashicorp.com/vault/downloads](https://developer.hashicorp.com/vault/downloads) and add it to your `PATH`.

Open a dedicated terminal and run:

```bash
vault server -dev -dev-root-token-id="root"
```

You will see output like:

```
==> Vault server configuration:

             Api Address: http://127.0.0.1:8200
                     Cgo: disabled
         Cluster Address: https://127.0.0.1:8201
               Log Level: info
                   Mlock: supported: false, enabled: false
           Recovery Mode: false
                 Storage: inmem
                 Version: Vault v1.16.x

==> Vault server started! Log data will stream in below:

WARNING! dev mode is enabled! In this mode, Vault runs entirely in-memory
and starts unsealed with a single unseal key. The root token is already
authenticated to the CLI, so you can immediately begin using Vault.
Root Token: root
```

> **Do not close this terminal** — the dev server runs in the foreground. Keep it open for the rest of the challenge.

---

#### 2. Configure your shell environment

In a **second terminal** (or if using Docker, in any terminal after starting the container), set the Vault address and token so the CLI commands in the challenge steps work without extra flags:

```bash
export VAULT_ADDR='http://127.0.0.1:8200'
export VAULT_TOKEN="root"
```

Verify the server is responding:

```bash
vault status
```

Expected output:
```
Key             Value
---             -----
Seal Type       shamir
Initialized     true
Sealed          false
Total Shares    1
Threshold       1
Version         1.16.x
Storage Type    inmem
Cluster Name    vault-cluster-...
Cluster ID      ...
HA Enabled      false
```

`Sealed: false` confirms the dev server is ready.

#### 3. Enable the KV v2 secrets engine

The dev server enables a KV v1 engine at `secret/` by default. The GitLab CI `secrets:` keyword expects KV v2. Re-enable it:

```bash
# Disable the default v1 mount
vault secrets disable secret/

# Enable KV v2 at the same path
vault secrets enable -path=secret kv-v2

# Confirm
vault secrets list
```

Expected output includes:
```
Path          Type     ...
----          ----     ---
secret/       kv       ...
```

#### 4. Expose the dev server with ngrok

GitLab SaaS shared runners run on GitLab's infrastructure and cannot reach `localhost` on your machine. You must expose the dev server on a public HTTPS URL.

Install ngrok from [ngrok.com/download](https://ngrok.com/download), then in a **third terminal**:

```bash
ngrok http 8200
```

ngrok will print a forwarding URL:

```
Forwarding  https://a1b2-203-0-113-42.ngrok-free.app -> http://localhost:8200
```

Copy the HTTPS URL (e.g. `https://a1b2-203-0-113-42.ngrok-free.app`). This is your `VAULT_SERVER_URL` for the rest of the challenge.

> **Free ngrok accounts** get a randomly assigned URL each session. If you restart ngrok, the URL changes — update the `VAULT_SERVER_URL` variable in GitLab each time.

#### 5. Set the GitLab CI variable

Navigate to your `lumio-api` project: **Settings > CI/CD > Variables**. Add:

| Key | Value | Masked | Protected |
|---|---|---|---|
| `VAULT_SERVER_URL` | `https://a1b2-203-0-113-42.ngrok-free.app` | No | No |

GitLab's `secrets:` keyword reads `VAULT_SERVER_URL` automatically — you do not reference it explicitly in the YAML.

#### 6. Verify end-to-end connectivity

Before configuring JWT auth, confirm the runner can reach your Vault server:

```yaml
# Add this temporary job to .gitlab-ci.yml
vault-connectivity-check:
  stage: test
  image: alpine:3.19
  script:
    - apk add --no-cache curl
    - curl -fs "$VAULT_SERVER_URL/v1/sys/health" | head -c 200
```

Expected output (Vault health endpoint returns JSON):
```json
{"initialized":true,"sealed":false,"standby":false,...}
```

If this job passes, your runner can reach the dev server and you are ready to proceed with JWT configuration below.

> **Dev server limitations:**
> - Data is **ephemeral** — all secrets and configuration are lost when the process stops.
> - The root token (`root`) has unrestricted access — this is fine for a lab but never acceptable in production.
> - For persistent testing across sessions, run Vault with a file storage backend instead of dev mode.

#### 7. Create the S3 bucket

The pipeline in this challenge deploys to `s3://lumio-frontend-prod`. Create the bucket before running the pipeline.

**AWS CLI:**
```bash
# Set your region
export AWS_REGION="eu-west-3"   # Change to match your Phase 3 ECR region

# Create the bucket
aws s3api create-bucket \
  --bucket lumio-frontend-prod \
  --region $AWS_REGION \
  --create-bucket-configuration LocationConstraint=$AWS_REGION

# Block all public access (private bucket — only the IAM user can write to it)
aws s3api put-public-access-block \
  --bucket lumio-frontend-prod \
  --public-access-block-configuration \
    "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"
```

> **us-east-1 note:** If your region is `us-east-1`, omit the `--create-bucket-configuration` flag — it is not accepted for that region:
> ```bash
> aws s3api create-bucket --bucket lumio-frontend-prod --region us-east-1
> ```

**AWS Console:**
1. Navigate to **S3 > Create bucket**
2. Bucket name: `lumio-frontend-prod`
3. Region: same as your ECR region from Phase 3
4. Block all public access: enabled (default)
5. Click **Create bucket**

**Verify the bucket exists:**
```bash
aws s3 ls | grep lumio-frontend-prod
# Expected: 2024-xx-xx xx:xx:xx lumio-frontend-prod
```

**IAM permissions required:**

The IAM user you created in Phase 3 needs S3 permissions. Add the following inline policy in **IAM > Users > lumio-ci > Add permissions > Attach policies directly > Create inline policy**:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "LumioS3Deploy",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:DeleteObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::lumio-frontend-prod",
        "arn:aws:s3:::lumio-frontend-prod/*"
      ]
    }
  ]
}
```

The `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` stored in Vault (step 2 below) must belong to this IAM user.

---

### The Jenkins original

```groovy
// Jenkins with HashiCorp Vault plugin
pipeline {
  agent any
  stages {
    stage('Deploy') {
      steps {
        withCredentials([
          vaultSecret(
            path: 'secret/lumio/production/aws',
            secretValues: [
              [envVar: 'AWS_ACCESS_KEY_ID', vaultKey: 'access_key'],
              [envVar: 'AWS_SECRET_ACCESS_KEY', vaultKey: 'secret_key']
            ]
          )
        ]) {
          sh 'aws s3 sync dist/ s3://lumio-frontend-prod/'
        }
      }
    }
  }
}
```

### Configure Vault JWT authentication

1. In Vault, enable the JWT auth method and configure it to trust your GitLab instance:

```bash
# On your Vault server
vault auth enable jwt

vault write auth/jwt/config \
  jwks_url="https://gitlab.com/-/jwks" \
  bound_issuer="https://gitlab.com"

# Create a policy for lumio-api
vault policy write lumio-api - <<EOF
path "secret/data/lumio/production/aws" {
  capabilities = ["read"]
}
EOF

# Create a role that allows lumio-api's CI jobs to use this policy
vault write auth/jwt/role/lumio-api \
  role_type="jwt" \
  policies="lumio-api" \
  token_explicit_max_ttl=60 \
  user_claim="sub" \
  bound_claims_type="glob" \
  bound_claims="{\"project_path\": \"lumio4615817/lumio-api\", \"ref_type\": \"branch\", \"ref\": \"main\"}"
```

2. Store the secret in Vault:

```bash
vault kv put secret/lumio/production/aws \
  access_key="AKIAIOSFODNN7EXAMPLE" \
  secret_key="wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
```

3. Configure the GitLab CI pipeline to fetch the secret using the `secrets:` keyword:

```yaml
# .gitlab-ci.yml — Vault integration via secrets:
deploy-production:
  stage: deploy
  image: amazon/aws-cli:2.15.0
  environment:
    name: production
  secrets:
    AWS_ACCESS_KEY_ID:
      vault: lumio/production/aws/access_key@secret   # path/key@mount
      token: $VAULT_TOKEN   # Or use JWT auth (see below)
    AWS_SECRET_ACCESS_KEY:
      vault: lumio/production/aws/secret_key@secret
  script:
    - aws s3 sync dist/ s3://lumio-frontend-prod/
    # AWS_ACCESS_KEY_ID and AWS_SECRET_ACCESS_KEY are injected from Vault
    # They are automatically masked in logs
```

4. For JWT authentication (recommended over token), confirm the Vault server URL variable is set in **Settings > CI/CD > Variables** (you set `VAULT_SERVER_URL` to your ngrok URL in the setup section above). Add the remaining variables:

| Key | Value | Notes |
|---|---|---|
| `VAULT_SERVER_URL` | `https://<your-ngrok-subdomain>.ngrok-free.app` | Set during dev server setup above |
| `VAULT_AUTH_PATH` | `jwt` | Auth method path |
| `VAULT_AUTH_ROLE` | `lumio-api` | Role name configured in step 1 |

```yaml
deploy-production:
  stage: deploy
  id_tokens:
    VAULT_ID_TOKEN:
      aud: $VAULT_SERVER_URL   # Must match the bound_issuer configured in Vault
  secrets:
    AWS_ACCESS_KEY_ID:
      vault:
        engine:
          name: kv-v2
          path: secret
        path: lumio/production/aws
        field: access_key
  script:
    - aws s3 sync dist/ s3://lumio-frontend-prod/
```

**JWT authentication flow:**

```mermaid
sequenceDiagram
    participant GL as GitLab Runner
    participant V as HashiCorp Vault
    participant S as Secret Store

    GL->>GL: Generate JWT token (id_tokens:)
    GL->>V: POST /auth/jwt/login (JWT token + role)
    V->>V: Validate JWT signature against GitLab JWKS
    V->>V: Check bound_claims (project_path, ref, ref_type)
    V->>GL: Return short-lived Vault token (TTL: 60s)
    GL->>V: GET /secret/data/lumio/production/aws
    V->>S: Fetch secret
    S->>V: Return secret value
    V->>GL: Return secret (injected as env var)
    GL->>GL: Run job script (secret available as env var)
```

**Expected output — deploy-production job log:**
```
Authenticating to Vault...
  Using JWT token for authentication
  Role: lumio-api
  Vault token TTL: 60s

Fetching secrets from Vault...
  AWS_ACCESS_KEY_ID: fetched from secret/lumio/production/aws (field: access_key) ✓
  AWS_SECRET_ACCESS_KEY: fetched from secret/lumio/production/aws (field: secret_key) ✓

$ aws s3 sync dist/ s3://lumio-frontend-prod/
upload: dist/index.html to s3://lumio-frontend-prod/index.html
...
Completed 147 files
Job succeeded
```

> **Comparison:** Jenkins requires a plugin (`hashicorp-vault-plugin`) and stores the Vault token or AppRole credentials in the Jenkins credential store. GitLab CI's `secrets:` keyword + `id_tokens:` uses short-lived JWT tokens issued per-job, with no long-lived credential needed in GitLab at all. The JWT is scoped to the specific project and branch by Vault's `bound_claims`, which is more fine-grained than anything Jenkins can enforce natively.

---

## Challenge 6 — Security audit: from 14 exposed credentials to 0

**Goal:** Use the GitLab API to audit all variables in the `lumio-api` project and produce a compliance report showing that no credentials remain in the codebase.

### Steps

1. Audit all project variables via the GitLab API:

```bash
# List all project CI/CD variables
curl --header "PRIVATE-TOKEN: $GITLAB_API_TOKEN" \
  "https://gitlab.com/api/v4/projects/$PROJECT_ID/variables" | \
  jq '.[] | {key: .key, masked: .masked, protected: .protected, environment_scope: .environment_scope}'
```

**Expected output:**
```json
{
  "key": "DOCKER_REGISTRY_USER",
  "masked": false,
  "protected": true,
  "environment_scope": "*"
}
{
  "key": "DOCKER_REGISTRY_PASSWORD",
  "masked": true,
  "protected": true,
  "environment_scope": "*"
}
{
  "key": "API_KEY",
  "masked": true,
  "protected": false,
  "environment_scope": "*"
}
{
  "key": "DATABASE_URL",
  "masked": true,
  "protected": false,
  "environment_scope": "staging"
}
{
  "key": "DATABASE_URL",
  "masked": true,
  "protected": true,
  "environment_scope": "production"
}
```

2. Scan the `.gitlab-ci.yml` and any included templates for hard-coded credentials using `git log`:

```bash
# Scan git history for potential credential patterns
git log --all -p | grep -iE "(password|secret|key|token|credential)\s*[:=]\s*['\"]?[A-Za-z0-9+/]{16,}" | \
  grep -v "^\+" | \
  head -20
```

**Expected output after clean migration:**
```
(no output — no hard-coded credentials found in git history)
```

3. Run a security scan job in the pipeline to verify no secrets are embedded:

```yaml
credential-audit:
  stage: test
  image: trufflesecurity/trufflehog:latest
  script:
    - trufflehog git file://. --only-verified --json 2>/dev/null | \
        jq '. | select(.FoundAt != null)' | \
        tee /tmp/credential-findings.json
    - |
      COUNT=$(cat /tmp/credential-findings.json | wc -l)
      if [ "$COUNT" -gt 0 ]; then
        echo "FAIL: Found $COUNT potential credentials in codebase"
        cat /tmp/credential-findings.json
        exit 1
      else
        echo "PASS: No credentials found in codebase"
      fi
  artifacts:
    reports:
      # Custom report visible in the Security tab
      sast: gl-sast-report.json
    when: always
```

**Expected output:**
```
$ trufflehog git file://. --only-verified --json 2>/dev/null | ...

PASS: No credentials found in codebase
Job succeeded
```

4. Produce the final migration audit table:

### Migration audit — Jenkins credentials to GitLab CI variables

| Jenkins Credential ID | Type | Was exposed | GitLab CI Variable | Masked | Protected | Env scope |
|---|---|---|---|---|---|---|
| `docker-registry-credentials` | usernamePassword | No (credential store) | `DOCKER_REGISTRY_USER` + `DOCKER_REGISTRY_PASSWORD` | Yes | Yes | `*` |
| `webhook-url` | string | No (credential store) | `WEBHOOK_URL` | Yes | No | `*` (group) |
| `staging-db-url` | string | **Yes (plain text in XML)** | `DATABASE_URL` | Yes | No | `staging` |
| `prod-db-url` | string | **Yes (plain text in XML)** | `DATABASE_URL` | Yes | Yes | `production` |
| `stripe-api-key` | string | **Yes (plain text in Jenkinsfile)** | `STRIPE_API_KEY` | Yes | Yes | `production` |
| `aws-access-key` | usernamePassword | **Yes (plain text in XML)** | Vault: `secret/lumio/production/aws` | N/A (Vault) | N/A (Vault) | `production` |
| `sentry-dsn` | string | No (credential store) | `SENTRY_DSN` | No | No | `*` (group) |
| `datadog-api-key` | string | No (credential store) | `DATADOG_API_KEY` | Yes | No | `*` (group) |
| `github-token` | string | **Yes (plain text in XML)** | `GITHUB_TOKEN` | Yes | Yes | `*` |
| `npm-registry-token` | string | No (credential store) | `NPM_TOKEN` | Yes | No | `*` |
| `kubernetes-config` | file | No (credential store) | Vault: `secret/lumio/k8s/kubeconfig` | N/A (Vault) | N/A (Vault) | `production` |
| `lighthouse-api-key` | string | No (credential store) | `LIGHTHOUSE_API_KEY` | Yes | No | `*` |
| `terraform-cloud-token` | string | **Yes (env var in XML)** | `TF_TOKEN_app_terraform_io` | Yes | Yes | `*` |
| `pagerduty-integration-key` | string | No (credential store) | `PAGERDUTY_INTEGRATION_KEY` | Yes | No | `*` (group) |

**Summary:**
- Jenkins credentials audited: 14
- Credentials that were exposed (plain text): 6
- GitLab CI variables created (masked + protected): 8
- Secrets moved to Vault: 2 (AWS credentials, Kubernetes config)
- Group-level variables created: 4
- **Credentials remaining in codebase: 0**

---

## Outcome

After completing Phase 5, all 14 Jenkins credentials have been migrated to the GitLab CI variable hierarchy with appropriate security controls:

| Metric | Jenkins (before) | GitLab CI (after) |
|---|---|---|
| Credentials in plain text | 6 of 14 | 0 of 14 |
| Credential scoping | Job-level `withCredentials` block | Project/Group/Instance variable hierarchy |
| Environment scoping | None (manual conditional logic) | Native `environment_scope` enforcement |
| Branch protection | None | Protected variables (only injected on protected branches) |
| Log masking | Yes (inside `withCredentials` block only) | Yes (all masked variables, everywhere) |
| Shared credentials | Copy-pasted across jobs | Group variables (single source of truth) |
| Rotation surface | Jenkins credential store + XML files | GitLab UI + Vault (for high-value secrets) |

Every credential now has a clear answer to three questions: *Who can see it? Which branches can use it? Which environment does it belong to?*

---

[Back to main README](../README.md) | [Previous: Phase 4 — Parallel Stages](../phase-4-parallel-stages/README.md) | [Next: Phase 6 — Docker Build and Registry](../phase-6-docker-registry/README.md)
