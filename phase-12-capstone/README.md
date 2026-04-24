# Phase 12 — Capstone: Decommissioning Jenkins

> **Objective:** Complete migration — 154/154 jobs migrated, Jenkins decommissioned | **Cost savings:** elimination of static Jenkins agent infrastructure (~$400–800/month depending on instance sizes) + ~30 hours/month of engineering time previously spent on Jenkins maintenance

---

## This is the end of an era

Eight years ago, someone installed Jenkins on a VM that was supposed to be temporary. That VM became the heartbeat of Lumio's engineering operation. It outlasted two CTOs, one reorg, three office moves, a pandemic, and a Series C fundraise. It built every release. It ran every test. It pushed every Docker image. It was there for all of it.

Now it ends.

Not because Jenkins failed — it did not. It shipped. But the debt accumulated quietly: shared library complexity, credential sprawl, opaque failures, plugin rot, two engineers who understood it deeply and one who just left the company. The system that served Lumio well for eight years became an obstacle to the next eight.

The decommissioning is not a ceremony of contempt. It is a recognition that what you migrate away from made it possible to migrate at all. Jenkins gave Lumio CI/CD when CI/CD was new. GitLab CI gives Lumio CI/CD for what comes next.

```mermaid
gantt
    title Lumio CI/CD Migration Timeline — 10 Weeks
    dateFormat  YYYY-MM-DD
    axisFormat  Week %W

    section Foundation
    Phase 0-1  Setup and first pipeline       :done, p01, 2024-01-08, 7d
    Phase 2-3  Pipeline anatomy + templates   :done, p23, 2024-01-15, 7d

    section Core migration
    Phase 4-5  Parallel stages + secrets      :done, p45, 2024-01-22, 7d
    Phase 6    Docker and registry            :done, p6,  2024-01-29, 7d
    Phase 7    Environments and gates         :done, p7,  2024-02-05, 7d

    section Advanced
    Phase 8    GitLab native features         :done, p8,  2024-02-12, 7d
    Phase 9    Runners                        :done, p9,  2024-02-19, 7d
    Phase 10   GitOps and infrastructure      :done, p10, 2024-02-26, 7d

    section Finalization
    Phase 11   Observability                  :done, p11, 2024-03-04, 7d
    Phase 12   Capstone — decommission        :active, p12, 2024-03-11, 7d

    section Jenkins status
    All 154 jobs active in Jenkins            :crit, j1, 2024-01-08, 14d
    Parallel run — shrinking                  :crit, j2, 2024-01-22, 42d
    Last 2 jobs migrated                      :crit, j3, 2024-03-04, 7d
    Jenkins decommissioned                    :milestone, m1, 2024-03-18, 0d
```

---

## Challenge 1 — Migration Checklist

**Goal:** Create a complete, auditable checklist of every Jenkins job, its GitLab CI equivalent, the parallel run period, and the cutover date.

This document is your contract. Before Jenkins is shut down, every row must be checked.

**Step 1: Create the migration checklist file.**

```markdown
<!-- docs/migration-checklist.md -->
# Lumio CI/CD Migration Checklist

Status: 152/154 jobs migrated | Last updated: 2024-03-11

## How to read this table
- **Parallel run start:** Date when GitLab CI pipeline first ran alongside Jenkins
- **Min successful runs:** GitLab pipeline must pass ≥5 times before cutover
- **Cutover date:** Date Jenkins job was disabled
- **Status:** ✅ Migrated | ⏳ In progress | ❌ Blocked

## lumio-api

| Jenkins job | GitLab CI file | Parallel run start | Successful runs | Cutover | Status |
|---|---|---|---|---|---|
| lumio-api/Jenkinsfile | `.gitlab-ci.yml` (stages: build, test, push) | 2024-01-22 | 47 | 2024-02-05 | ✅ |
| lumio-api/Jenkinsfile.deploy | `.gitlab-ci.yml` (deploy:staging, deploy:production) | 2024-02-05 | 23 | 2024-02-19 | ✅ |
| lumio-api/Jenkinsfile.db-migrate | `.gitlab-ci.yml` (migrate:staging, migrate:production) | 2024-02-05 | 18 | 2024-02-19 | ✅ |

## lumio-frontend

| Jenkins job | GitLab CI file | Parallel run start | Successful runs | Cutover | Status |
|---|---|---|---|---|---|
| lumio-frontend/Jenkinsfile | `.gitlab-ci.yml` (build, lighthouse, s3-deploy) | 2024-01-29 | 38 | 2024-02-12 | ✅ |
| lumio-frontend/Jenkinsfile.e2e | `.gitlab-ci.yml` (e2e:chrome, e2e:firefox) | 2024-03-04 | 2 | — | ⏳ |

## lumio-worker

| Jenkins job | GitLab CI file | Parallel run start | Successful runs | Cutover | Status |
|---|---|---|---|---|---|
| lumio-worker/Jenkinsfile | `.gitlab-ci.yml` (build, test, push, deploy) | 2024-01-29 | 41 | 2024-02-12 | ✅ |

## infrastructure

| Jenkins job | GitLab CI file | Parallel run start | Successful runs | Cutover | Status |
|---|---|---|---|---|---|
| infrastructure/Jenkinsfile.terraform | `infrastructure/.gitlab-ci.yml` | 2024-02-26 | 12 | 2024-03-04 | ✅ |
| infrastructure/Jenkinsfile.nightly | `.gitlab-ci.yml` (scheduled pipeline) | 2024-03-04 | 1 | — | ⏳ |
```

**Step 2: Identify the two remaining jobs.**

Two jobs are still in `⏳ In progress`:
- `lumio-frontend/Jenkinsfile.e2e` — Playwright E2E tests (Challenge 2)
- `infrastructure/Jenkinsfile.nightly` — Nightly cleanup and cost reports (Challenge 3)

These are the last two jobs standing between Lumio and a decommissioned Jenkins instance.

---

## Challenge 2 — Migrate `Jenkinsfile.e2e`

**Goal:** Migrate the Playwright E2E test suite for `lumio-frontend` to GitLab CI, with browser support, artifact collection for failed test screenshots, and HTML report publishing.

**The original `Jenkinsfile.e2e`:**

```groovy
// jenkins/jobs/lumio-frontend/Jenkinsfile.e2e  (BEFORE — do not copy)
pipeline {
    agent { docker { image 'mcr.microsoft.com/playwright:v1.44.0-focal' } }
    stages {
        stage('Install') {
            steps { sh 'npm ci' }
        }
        stage('E2E — Chrome') {
            steps {
                sh 'npx playwright test --browser=chromium --reporter=html'
            }
        }
        stage('E2E — Firefox') {
            steps {
                sh 'npx playwright test --browser=firefox --reporter=html'
            }
        }
    }
    post {
        always {
            publishHTML([
                reportDir: 'playwright-report',
                reportFiles: 'index.html',
                reportName: 'Playwright Report'
            ])
            archiveArtifacts artifacts: 'test-results/**', allowEmptyArchive: true
        }
    }
}
```

**Step 1: Write the GitLab CI equivalent.**

```yaml
# lumio-frontend/.gitlab-ci.yml — E2E section
# Added to the existing .gitlab-ci.yml alongside build and unit test jobs

e2e:chrome:
  stage: test
  image: mcr.microsoft.com/playwright:v1.44.0-jammy    # jammy = Ubuntu 22.04 LTS
  variables:
    BASE_URL: https://staging.lumio.io
    CI: "true"
  script:
    - npm ci
    - npx playwright test --browser=chromium --reporter=html,junit
  artifacts:
    when: always                               # Collect even when tests fail
    expire_in: 7 days
    paths:
      - playwright-report/                     # HTML report
      - test-results/                          # Screenshots, traces, videos
    reports:
      junit: test-results/junit.xml            # Parsed into MR test summary
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == "main"

e2e:firefox:
  stage: test
  image: mcr.microsoft.com/playwright:v1.44.0-jammy
  variables:
    BASE_URL: https://staging.lumio.io
    CI: "true"
  script:
    - npm ci
    - npx playwright test --browser=firefox --reporter=html,junit
  artifacts:
    when: always
    expire_in: 7 days
    paths:
      - playwright-report/
      - test-results/
    reports:
      junit: test-results/junit.xml
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == "main"
  allow_failure: true    # Firefox tests are historically less stable — track but don't block
```

**Step 2: Verify artifacts are collected on failure.**

Intentionally break a test to confirm artifact collection works:

```bash
# Temporarily add a failing assertion to a test
echo "test('intentional failure', async ({ page }) => { expect(1).toBe(2); });" \
  >> tests/example.spec.ts
git add tests/example.spec.ts
git commit -m "test: add intentional failure to verify artifact collection"
git push
```

After the pipeline runs, navigate to the failed job and click **Download artifacts**. You should find:
- `playwright-report/index.html` — full HTML report with screenshots of the failure
- `test-results/` — individual test traces that can be opened with `npx playwright show-trace`

In Jenkins, this required the `publishHTML` plugin, manual archiving config, and the report was buried in the build page. In GitLab, the artifact is downloadable from the pipeline page and the JUnit report is parsed into the MR test summary automatically.

**Step 3: Update the migration checklist.**

Mark `lumio-frontend/Jenkinsfile.e2e` as `✅` once the GitLab pipeline has run successfully 5 times.

---

## Challenge 3 — Migrate `Jenkinsfile.nightly`

**Goal:** Migrate the nightly cleanup and cost report job from a Jenkins cron trigger to a GitLab CI scheduled pipeline, with proper isolation so the cleanup logic only runs on schedule (not on every commit).

**The original `Jenkinsfile.nightly`:**

```groovy
// jenkins/jobs/infrastructure/Jenkinsfile.nightly  (BEFORE — do not copy)
pipeline {
    agent any
    triggers { cron('0 2 * * *') }
    environment {
        CLEANUP_MODE = 'full'
        AWS_REGION   = 'eu-west-1'
    }
    stages {
        stage('Cleanup old images') {
            steps {
                sh './scripts/cleanup-ecr-images.sh --older-than 30d --mode ${CLEANUP_MODE}'
            }
        }
        stage('Cleanup test namespaces') {
            steps {
                sh 'kubectl delete namespace -l env=review --field-selector status.phase=Active'
            }
        }
        stage('Cost report') {
            steps {
                sh './scripts/generate-cost-report.sh --period yesterday'
                slackSend channel: '#lumio-infra', message: "Nightly cost report ready"
            }
        }
    }
}
```

**Step 1: Create the scheduled pipeline in GitLab.**

In the `infrastructure` project, navigate to **Build > Pipeline schedules > New schedule**:

| Field | Value |
|---|---|
| Description | Nightly cleanup and cost report |
| Interval | `0 2 * * *` (daily at 02:00 UTC) |
| Target branch | `main` |
| Variables | `CLEANUP_MODE` = `full` |

Click **Save pipeline schedule**.

**Step 2: Write the GitLab CI pipeline, isolated to schedule runs.**

```yaml
# infrastructure/.gitlab-ci.yml — nightly section
# Add to the existing Terraform pipeline file

# These jobs only run when triggered by a schedule, never on commits.
# The $CI_PIPELINE_SOURCE variable is set to "schedule" by GitLab automatically.

.nightly_rule: &nightly_rule
  rules:
    - if: $CI_PIPELINE_SOURCE == "schedule"

cleanup:ecr-images:
  <<: *nightly_rule
  stage: cleanup
  image: amazon/aws-cli:2.15.0
  script:
    - ./scripts/cleanup-ecr-images.sh --older-than 30d --mode ${CLEANUP_MODE}
  variables:
    AWS_DEFAULT_REGION: eu-west-1

cleanup:review-namespaces:
  <<: *nightly_rule
  stage: cleanup
  image: bitnami/kubectl:1.29
  script:
    - kubectl delete namespace -l env=review --field-selector status.phase=Active --ignore-not-found
  environment:
    name: cleanup
    kubernetes:
      namespace: default

cost:report:
  <<: *nightly_rule
  stage: report
  image: python:3.12-slim
  script:
    - pip install -r scripts/requirements.txt
    - python scripts/generate-cost-report.py --period yesterday --output report.html
    - |
      curl -X POST "${SLACK_WEBHOOK_URL}" \
        -H 'Content-type: application/json' \
        --data "{\"text\": \"Nightly cost report ready — <${CI_PIPELINE_URL}|View pipeline>\"}"
  artifacts:
    name: "cost-report-$(date +%Y-%m-%d)"
    paths:
      - report.html
    expire_in: 30 days

stages:
  - cleanup
  - report
```

**Step 3: Verify schedule isolation.**

Push a regular commit to `main`. Confirm that `cleanup:ecr-images`, `cleanup:review-namespaces`, and `cost:report` do not appear in the pipeline. They should be absent — not skipped, but absent — because their `rules` do not match a push-triggered pipeline.

Navigate to **Build > Pipeline schedules** and click **Run** to manually trigger the schedule. Confirm all three jobs run.

Expected output from `cleanup:review-namespaces`:

```
$ kubectl delete namespace -l env=review --field-selector status.phase=Active --ignore-not-found
namespace "review-feature-invoice-export" deleted
namespace "review-fix-login-timeout" deleted
namespace "review-chore-update-deps" deleted
```

**Step 4: Update the migration checklist.**

Mark `infrastructure/Jenkinsfile.nightly` as `✅`. All 154 jobs are now accounted for.

---

## Challenge 4 — Final Parallel Run Validation

**Goal:** Run an automated validation script that confirms every Jenkins job has a GitLab CI equivalent that has passed at least 5 times. No job may be cut over until this check passes.

**Step 1: Write the validation script.**

```bash
#!/bin/bash
# scripts/validate-migration.sh
# Validates that every listed GitLab CI pipeline has passed >= MIN_RUNS times.
# Exits 1 if any job fails validation.

set -euo pipefail

GITLAB_TOKEN="${GITLAB_TOKEN:?GITLAB_TOKEN is required}"
GITLAB_URL="https://gitlab.com"
MIN_RUNS=5
FAILED=0

declare -A JOBS=(
  ["lumio/lumio-api|main|build"]="build"
  ["lumio/lumio-api|main|test:unit"]="test:unit"
  ["lumio/lumio-api|main|docker:build"]="docker:build"
  ["lumio/lumio-api|main|deploy:staging"]="deploy:staging"
  ["lumio/lumio-api|main|deploy:production"]="deploy:production"
  ["lumio/lumio-frontend|main|build"]="build"
  ["lumio/lumio-frontend|main|e2e:chrome"]="e2e:chrome"
  ["lumio/lumio-frontend|main|e2e:firefox"]="e2e:firefox"
  ["lumio/lumio-worker|main|build"]="build"
  ["lumio/lumio-worker|main|deploy:staging"]="deploy:staging"
  ["lumio/infrastructure|main|plan"]="plan"
  ["lumio/infrastructure|main|apply"]="apply"
  ["lumio/infrastructure|schedule|cleanup:ecr-images"]="cleanup:ecr-images"
  ["lumio/infrastructure|schedule|cost:report"]="cost:report"
)

echo "╔═══════════════════════════════════════════════════════════════╗"
echo "║       Lumio CI/CD Migration — Final Validation Report         ║"
echo "╚═══════════════════════════════════════════════════════════════╝"
echo ""
printf "%-45s %-8s %-10s %s\n" "Job" "Runs" "Min Req" "Status"
printf "%-45s %-8s %-10s %s\n" "---" "----" "-------" "------"

for key in "${!JOBS[@]}"; do
  IFS='|' read -r project ref job_name <<< "$key"
  encoded_project=$(python3 -c "import urllib.parse; print(urllib.parse.quote('${project}', safe=''))")

  successful_runs=$(curl -s \
    --header "PRIVATE-TOKEN: ${GITLAB_TOKEN}" \
    "${GITLAB_URL}/api/v4/projects/${encoded_project}/jobs?scope[]=success&per_page=100" | \
    jq --arg name "$job_name" '[.[] | select(.name == $name)] | length')

  if [ "${successful_runs}" -ge "${MIN_RUNS}" ]; then
    status="✅ PASS"
  else
    status="❌ FAIL (need $((MIN_RUNS - successful_runs)) more)"
    FAILED=$((FAILED + 1))
  fi

  printf "%-45s %-8s %-10s %s\n" "${project}/${job_name}" "${successful_runs}" "${MIN_RUNS}" "${status}"
done

echo ""
if [ "${FAILED}" -eq 0 ]; then
  echo "✅ All jobs validated. Ready for Jenkins decommission."
else
  echo "❌ ${FAILED} job(s) failed validation. Do not decommission Jenkins yet."
  exit 1
fi
```

**Step 2: Run the validation.**

```bash
export GITLAB_TOKEN="glpat-xxxxxxxxxxxxxxxxxxxx"
bash scripts/validate-migration.sh
```

Expected output when all 154 jobs pass:

```
╔═══════════════════════════════════════════════════════════════╗
║       Lumio CI/CD Migration — Final Validation Report         ║
╚═══════════════════════════════════════════════════════════════╝

Job                                           Runs     Min Req    Status
---                                           ----     -------    ------
lumio/lumio-api/build                         47       5          ✅ PASS
lumio/lumio-api/test:unit                     47       5          ✅ PASS
lumio/lumio-api/docker:build                  45       5          ✅ PASS
lumio/lumio-api/deploy:staging                23       5          ✅ PASS
lumio/lumio-api/deploy:production             11       5          ✅ PASS
lumio/lumio-frontend/build                    38       5          ✅ PASS
lumio/lumio-frontend/e2e:chrome               22       5          ✅ PASS
lumio/lumio-frontend/e2e:firefox              20       5          ✅ PASS
lumio/lumio-worker/build                      41       5          ✅ PASS
lumio/lumio-worker/deploy:staging             19       5          ✅ PASS
lumio/infrastructure/plan                     12       5          ✅ PASS
lumio/infrastructure/apply                    7        5          ✅ PASS
lumio/infrastructure/cleanup:ecr-images       7        5          ✅ PASS
lumio/infrastructure/cost:report              7        5          ✅ PASS

✅ All jobs validated. Ready for Jenkins decommission.
```

All 154 jobs pass. The validation script is your sign-off document.

---

## Challenge 5 — Cutover and Decommission

**Goal:** Disable all Jenkins jobs, run 48 hours on GitLab CI only, then permanently decommission the Jenkins instance and calculate the savings.

This is the point of no return. Read the checklist in Challenge 1 one more time. Confirm the validation script in Challenge 4 passed cleanly. Then proceed.

**Step 1: Put Jenkins in quiet mode.**

Quiet mode stops Jenkins from accepting new builds while allowing running builds to finish. It is reversible.

```bash
# Put Jenkins in quiet mode (no new builds accepted)
curl -X POST http://localhost:8080/quietDown \
  --user "admin:${JENKINS_TOKEN}"
```

Verify in the Jenkins UI: a yellow banner appears at the top reading *"Jenkins is going to shut down. No new builds will be executed."*

**Step 2: Wait for all running builds to complete.**

```bash
# Poll until no builds are running
watch -n 10 'curl -s "http://localhost:8080/api/json?tree=jobs[name,lastBuild[building]]" \
  --user "admin:${JENKINS_TOKEN}" | \
  jq "[.jobs[] | select(.lastBuild.building == true)] | length"'
```

When the output is `0`, all builds have finished.

**Step 3: 48-hour monitoring window.**

Start a 48-hour clock. During this window:
- GitLab CI is the only system running pipelines.
- Engineers commit, merge, and deploy as normal.
- The on-call rotation watches the Grafana dashboard from Phase 11.
- If any critical failure occurs that cannot be resolved in GitLab CI within 2 hours, cancel quiet mode and roll back: `curl -X POST http://localhost:8080/cancelQuietDown`.

After 48 hours with a clean green dashboard, proceed.

**Step 4: Stop and remove Jenkins.**

```bash
# Stop the Jenkins container and its agent
docker compose -f jenkins/docker-compose.yml down

# Verify it is stopped
docker ps | grep jenkins
# Expected: (no output)

# Archive the data volume before deletion (keep for 90 days per Lumio's data retention policy)
docker run --rm \
  -v jenkins_home:/data \
  -v $(pwd)/backups:/backup \
  alpine tar czf /backup/jenkins-final-backup-$(date +%Y%m%d).tar.gz /data

# Remove volumes
docker volume rm jenkins_home jenkins_agent_workdir
```

Expected output:

```
[+] Running 3/3
 ✔ Container jenkins-agent  Stopped    0.3s
 ✔ Container jenkins         Stopped    1.2s
 ✔ Network jenkins_default  Removed    0.1s

jenkins_home
jenkins_agent_workdir
```

**Step 5: Calculate the savings.**

```
Static Jenkins infrastructure decommissioned:
- Jenkins controller: 1x c5.xlarge (4 vCPU / 8GB) = $122/month
- Jenkins agents (always-on pool): 3x t3.medium = $96/month
- EBS volumes (100 GB build artifacts): $10/month
- Total infrastructure: $228/month

Engineering time savings:
- Jenkins maintenance (plugin updates, config drift, outage response): 15 hours/month
- Groovy shared library maintenance (the 12,000-line library): 8 hours/month
- Onboarding new engineers to Jenkins: 4 hours/new hire × 2 hires/quarter = 2.7 hours/month
- Total: ~26 hours/month × $120/hour (loaded cost) = $3,120/month

Total monthly savings: $228 (infrastructure) + $3,120 (engineering time) = $3,348/month
Annual savings: $40,176
```

---

## Challenge 6 — Retrospective and Documentation

**Goal:** Write an Architecture Decision Record (ADR) documenting the migration decision, create the final migration summary table, and document the three most important lessons learned.

**Step 1: Write the ADR.**

```markdown
<!-- docs/adr/0001-gitlab-ci-over-jenkins.md -->
# ADR 0001 — Adopt GitLab CI/CD as the primary CI/CD platform

**Date:** 2024-03-18
**Status:** Accepted
**Deciders:** VP of Engineering, Platform Engineering team

## Context

Lumio operated a Jenkins-based CI/CD system for 8 years with 154 jobs, a 12,000-line
Groovy shared library, and 47 plugins. In Q3 2023, a DevEx survey identified CI/CD as
the top developer pain point for the third consecutive quarter. A security audit flagged
14 jobs with plain-text credentials. The two engineers who understood the system deeply
were both considering leaving.

## Decision

Migrate all CI/CD pipelines from Jenkins to GitLab CI/CD using the Parallel Run pattern:
GitLab CI takes over one pipeline at a time, Jenkins is decommissioned only when the
last job is proven stable in GitLab.

## Alternatives considered

| Option | Rejected reason |
|---|---|
| Jenkins modernization (upgrade + plugin audit) | High effort, does not address architectural debt, credentials problem, or developer experience |
| GitHub Actions | Would require migrating from GitLab SCM — double migration, higher risk |
| Tekton/ArgoCD | Higher operational complexity, requires Kubernetes expertise not present on the team |
| CircleCI / BuildKite | SaaS lock-in, cost at Lumio's scale, no self-managed option |

## Consequences

**Positive:**
- YAML pipelines readable by any engineer, no Groovy expertise required
- Credentials stored in GitLab CI variables (masked, protected, env-scoped)
- DORA metrics and pipeline analytics built in, no plugins
- GitLab agent for Kubernetes eliminates cluster-admin credentials in CI
- Terraform state managed by GitLab — no S3 bucket or external credentials

**Negative:**
- 10-week migration effort required continued parallel operation of both systems
- Some Jenkins concepts (like parameterized builds triggered by upstream jobs) required
  rethinking rather than direct translation
- Shared library Groovy abstractions could not be directly ported — required simplification
  (which was ultimately a benefit but required upfront investment)

## Migration result

154/154 jobs migrated. Jenkins decommissioned 2024-03-18. Annual savings: $40,176.
```

**Step 2: The final migration mapping table.**

| Jenkins concept | GitLab CI equivalent | Benefit |
|---|---|---|
| `Jenkinsfile` | `.gitlab-ci.yml` | YAML, version-controlled, no plugin required |
| Shared library (`@Library`) | `include: project:` templates | Simpler, no Groovy, discoverable |
| `agent { docker { image } }` | `image:` | Same concept, simpler syntax |
| `credentials('id')` | CI/CD Variables (masked/protected) | Never in plaintext, env-scoped |
| `when { branch 'main' }` | `rules: - if: $CI_COMMIT_BRANCH` | More expressive, supports regex |
| `parallel { stage... }` | `needs: []` DAG | More flexible fan-out |
| `input 'Deploy?'` | `when: manual` | Same concept, visible in UI |
| `archiveArtifacts` | `artifacts: paths:` | Native, with expiry and reports |
| `junit 'results.xml'` | `artifacts: reports: junit:` | Auto-parsed in MR test summary |
| `cron('0 2 * * *')` | Pipeline schedules | UI-managed, variable injection |
| `post { always }` | `when: always` + `after_script` | Per-job control |
| Blue Ocean pipeline view | GitLab pipeline graph | Faster, no plugin required |
| kubeconfig credential | GitLab Agent for Kubernetes | Outbound-only, no cluster-admin |
| Build metrics (plugin) | Analytics > CI/CD Analytics | Built-in, DORA metrics included |
| Terraform S3 state backend | GitLab HTTP state backend | No S3 bucket, no extra credentials |

**Step 3: Three lessons learned.**

> **Lesson 1: The hardest part is not the YAML — it is the dependency untangling.**
>
> Lumio's Jenkins jobs had implicit dependencies that nobody had documented. Job A triggered job B which waited for job C to complete, managed by a Groovy function buried 400 lines deep in the shared library. Translating to GitLab CI forced making these dependencies explicit in `needs:`. That work was slow and occasionally blocked migration. But the result — an explicit DAG in YAML — is something any engineer can read. The untangling was the point.

> **Lesson 2: Parallel run duration is determined by confidence, not by calendar.**
>
> The original plan was a 4-week migration. It took 10 weeks. The extra time was not wasted — it was spent building confidence that each GitLab CI pipeline produced identical results to its Jenkins equivalent before the Jenkins version was disabled. Rushing the parallel run would have been false economy. The 10-week duration is a feature of the approach, not a failure of execution.

> **Lesson 3: Observability reveals what you did not know you did not know.**
>
> The `integration:database` job had a 23.4% failure rate. Everyone on the team knew it "sometimes failed" — but nobody knew how often. With Jenkins, this number required custom tooling to surface. GitLab CI's job-level analytics made it visible in an afternoon. Three weeks into the migration, a developer fixed the root cause (a race condition in test teardown). It had been flaky for two years.

**Step 4: Update the top-level README.**

Add the migration completion badges to `/README.md`:

```markdown
![Migration Status](https://img.shields.io/badge/Migration-154%2F154%20Complete-brightgreen)
![Jenkins](https://img.shields.io/badge/Jenkins-Decommissioned_2024--03--18-red)
![Annual Savings](https://img.shields.io/badge/Annual_Savings-%2440%2C176-blue)
```

---

## What We Learned — 10 Migration Surprises

The following surprises came from the real work of migration — things that seemed straightforward from the outside but required a different approach in GitLab CI, or the reverse.

| Surprise | Jenkins | GitLab CI | Verdict |
|---|---|---|---|
| Shared library size | 12,000 lines of Groovy, built over 3 years | 400 lines of YAML templates, built during migration | GitLab CI wins — YAML forces simplicity by design |
| Parallel jobs | `parallel {}` block, visually clear | `needs: []` DAG — more powerful but less intuitive at first | Equivalent in outcome, GitLab more expressive for complex DAGs |
| Credentials security | Credentials stored in Jenkins, accessible to any job | Variables scoped by environment, protected, masked | GitLab CI wins — no comparison |
| Nightly job isolation | `cron()` trigger, always-running agent | `rules: - if: $CI_PIPELINE_SOURCE == "schedule"` | GitLab CI wins — clean isolation without a separate job config |
| Manual approval gate | `input 'message'` inside the pipeline — pauses the build | `when: manual` — same concept, visible in pipeline graph | Equivalent; GitLab UI is better for reviewers |
| Terraform state | S3 bucket, IAM credentials required | GitLab HTTP backend, uses CI job token | GitLab CI wins — no external dependencies |
| Artifact browsing | Build > Workspace — manual navigation | Download button per artifact, expiry control | GitLab CI wins — structured, expiring, browsable |
| Pipeline as code portability | Jenkinsfile tied to Jenkins-specific syntax | `.gitlab-ci.yml` is portable YAML | GitLab CI wins — easier for new team members |
| E2E test screenshots on failure | Required `publishHTML` plugin + config | `artifacts: when: always` — three lines of YAML | GitLab CI wins — no plugin, works immediately |
| Deployment audit trail | Build log only, manual reconstruction | Full deployment history per environment with actor, commit, MR | GitLab CI wins — compliance-ready out of the box |

---

## Outcome

Jenkins is gone. All 154 jobs run on GitLab CI.

| Metric | Before (Jenkins) | After (GitLab CI) | Change |
|---|---|---|---|
| Jobs under management | 154 Jenkins jobs | 154 GitLab CI jobs | Neutral |
| CI/CD infrastructure cost | $228/month | $0 (GitLab.com free tier) | -$228/month |
| Engineering maintenance time | ~26 hours/month | ~3 hours/month | -23 hours/month |
| Credential security violations | 14 jobs with plain-text creds | 0 | Fixed |
| DORA tier at migration start | Medium (1 deploy/week) | — | — |
| DORA tier at migration end | — | High (3.2 deploys/day) | Improved |
| Onboarding time for new engineer | 1 week to understand Jenkinsfiles | 1 afternoon to understand `.gitlab-ci.yml` | -4.5 days |
| Flaky test visibility | Unknown | `integration:database` at 23.4% — fixed in week 3 | Fixed |

---

## Congratulations

You have completed the Lumio CI/CD Migration Lab.

You migrated 154 Jenkins jobs to GitLab CI, including: parameterized builds, shared library translations, Docker registry integration, environment-gated deployments, Kubernetes agent setup, Terraform CI/CD, DORA metrics, and a full decommission. You did it using the Parallel Run pattern — keeping the lights on the entire time, never pausing releases.

That is not a tutorial exercise. That is the real work of platform engineering.

The engineers who do this in production are the ones who make it possible for everyone else to ship. The ones who make the pipeline invisible — reliable, fast, and easy enough that developers never have to think about it. The ones who, when it works perfectly, get no credit — because nothing broke.

Now you know how it is done.

---

[Back to main README](../README.md) | [Previous: Phase 11 — Observability](../phase-11-observability/README.md)
