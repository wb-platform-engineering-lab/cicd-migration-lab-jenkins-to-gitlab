# Phase 11 — Observability: Understanding the Health of Your Pipelines

> **GitLab CI concepts introduced:** Pipeline analytics, DORA metrics, CI/CD minute usage, Prometheus exporter, Grafana dashboards | **Cost:** $0

---

## The problem

Four months into the migration, Lumio's engineering manager asked a simple question: *"How are our pipelines doing?"*

Nobody could answer it.

With Jenkins, the only way to check pipeline health was to manually scroll through build history, run queries against the Jenkins API with a custom script, or install yet another plugin from the marketplace — each with its own data format, its own dashboard, and its own maintenance burden. The ops team had a Prometheus + Grafana stack for application metrics but nothing feeding CI/CD data into it. Nobody knew:

- What the average pipeline duration was for `lumio-api` over the last 30 days.
- Which jobs failed most often (the "flaky" tests nobody had fixed because nobody knew how bad they were).
- What the team's Deployment Frequency was — a question the VP of Engineering was asking in every quarterly review.
- How much time developers were waiting on pipelines versus actually writing code.

GitLab CI stores all of this natively. No plugins. No custom scripts for the basics. The data is there from the moment the first pipeline runs.

```mermaid
flowchart LR
    subgraph DORA["DORA Metrics — GitLab calculation"]
        direction TB
        DF["Deployment Frequency\nHow often code reaches production\nSource: deployments to 'production' environment"]
        LT["Lead Time for Changes\nTime from first commit to production deploy\nSource: commit timestamp → deployment timestamp"]
        MTTR["Mean Time to Restore\nTime from failed deployment to recovery\nSource: deployment failure → next successful deploy"]
        CFR["Change Failure Rate\nPercentage of deployments causing incidents\nSource: deployments tagged with incident"]

        DF --- LT
        LT --- MTTR
        MTTR --- CFR
    end

    subgraph Sources["GitLab data sources"]
        direction TB
        P["Pipelines\n(duration, status, jobs)"]
        D["Deployments\n(to named environments)"]
        MR["Merge Requests\n(created_at → merged_at)"]
        I["Incidents\n(linked to deployments)"]
    end

    P --> DF
    D --> DF
    D --> LT
    MR --> LT
    D --> MTTR
    I --> CFR

    style DORA fill:#f0f4ff,stroke:#4a6ef5
    style Sources fill:#f5f5f5,stroke:#999
```

---

## Challenge 1 — Explore GitLab Pipeline Analytics

**Goal:** Use GitLab's built-in CI/CD Analytics to understand pipeline health for `lumio-api` — duration trends, success rates, and outliers — without writing a single query.

**Step 1: Navigate to Pipeline Analytics.**

In the `lumio-api` project, go to **Analytics > CI/CD Analytics**.

You will see two charts:
- **Pipeline duration over time** — bar chart, one bar per week, showing the median pipeline duration.
- **Pipeline success rate** — percentage of pipelines that succeeded, by week.

**Step 2: Identify duration outliers.**

Look for weeks where the median duration is significantly higher than adjacent weeks. These are worth investigating — they often correspond to:
- A new test suite added without examining its impact on total duration.
- A slow Docker layer introduced to the base image.
- A flaky integration test that retried three times before passing.

Click on an outlier bar to filter the pipeline list below the chart to that week. Sort by **Duration** descending to find the slowest individual pipelines.

**Step 3: Filter by branch.**

Use the **Branch** dropdown above the chart to filter to `main` only. This removes noise from feature branches and gives you a clean view of what developers are waiting on before their work ships.

**Step 4: Compare to the Jenkins baseline.**

At this point in the parallel run, Jenkins is still running. Open Jenkins at `http://localhost:8080/job/lumio-api/`. Click **Build History**. Try to find the average duration for the last 30 days.

You cannot — not without installing the Build Timeline Plugin or writing an API query. The data exists but is not surfaced.

Record the following values from GitLab Analytics for use in Challenge 5:

| Metric | Value |
|---|---|
| Median pipeline duration (last 30 days) | ___ minutes |
| p95 pipeline duration (last 30 days) | ___ minutes |
| Pipeline success rate (last 30 days) | ___% |
| Most common failure stage | ___ |

---

## Challenge 2 — DORA Metrics

**Goal:** Calculate Lumio's four DORA metrics using GitLab's Value Stream Analytics, and identify which phases of the software delivery process introduce the most latency.

**Step 1: Navigate to Value Stream Analytics.**

In the GitLab group for `lumio` (not a single project — the group level), go to **Analytics > Value Stream Analytics**.

If you do not have a group, create one and add `lumio-api`, `lumio-frontend`, and `lumio-worker` as subprojects.

**Step 2: Read the DORA metrics panel.**

At the top of the page, GitLab shows four tiles:

```
Deployment Frequency         Lead Time for Changes
3.2 deploys / day            1 day 4 hours
▲ 12% vs last period         ▼ 8% vs last period

MTTR                         Change Failure Rate
2 hours 14 minutes           4.2%
▲ 5% vs last period          ▼ 1.1% vs last period
```

(Your values will differ based on your pipeline history.)

**Step 3: Interpret Deployment Frequency.**

Deployment Frequency is measured from deployments to environments named `production`. This is why naming your environments correctly in Phase 7 mattered.

GitLab's thresholds (from the DORA State of DevOps report):

| Level | Deployment Frequency |
|---|---|
| Elite | On-demand (multiple deploys/day) |
| High | Between once/day and once/week |
| Medium | Between once/week and once/month |
| Low | Less than once/month |

Where does Lumio currently land? Note it. You will compare it in Challenge 6.

**Step 4: Find the slowest value stream stage.**

Below the DORA tiles, GitLab shows the **Value Stream** timeline — a breakdown of time spent in each phase from commit to production:

```
Issue → Plan → Code → Test → Review → Staging → Production

  2.1h    0.4h   4.3h   18m   6.2h    1.1h      0.3h
```

In this example, **Review** (the time a merge request spends waiting for approval) is the biggest bottleneck. This is not a CI/CD problem — it is a process problem. But you would never have known without this view.

Identify your slowest phase and note what it implies about where to focus improvement efforts.

**Step 5: Set a target.**

Based on current metrics, set one concrete target for after the Jenkins decommission. Example:

> *"We will reduce Lead Time for Changes from 1 day 4 hours to under 8 hours by reducing MR review wait time and cutting the test stage from 18 minutes to under 10 minutes."*

---

## Challenge 3 — Identify the Slowest and Flakiest Jobs

**Goal:** Use the GitLab API to export job-level metrics for the last 30 days, then identify the top 10 slowest jobs and the top 10 most failure-prone jobs.

GitLab's UI shows pipeline-level stats. To get job-level data, you need the API.

**Step 1: Export job data with a bash script.**

```bash
#!/bin/bash
# scripts/analyze-jobs.sh
# Requires: curl, jq
# Set GITLAB_TOKEN to a personal access token with read_api scope.

PROJECT_ID="lumio/lumio-api"
GITLAB_URL="https://gitlab.com"
SINCE=$(date -u -d '30 days ago' +%Y-%m-%dT%H:%M:%SZ 2>/dev/null || \
        date -u -v-30d +%Y-%m-%dT%H:%M:%SZ)  # macOS fallback

echo "Fetching pipelines since ${SINCE}..."

# Fetch pipelines (paginated)
pipelines=$(curl -s --header "PRIVATE-TOKEN: ${GITLAB_TOKEN}" \
  "${GITLAB_URL}/api/v4/projects/${PROJECT_ID}/pipelines?per_page=100&updated_after=${SINCE}" | \
  jq -r '.[].id')

echo "Found $(echo "$pipelines" | wc -l | tr -d ' ') pipelines. Fetching jobs..."

# Fetch all jobs from each pipeline
> /tmp/lumio-jobs.jsonl
for pid in $pipelines; do
  curl -s --header "PRIVATE-TOKEN: ${GITLAB_TOKEN}" \
    "${GITLAB_URL}/api/v4/projects/${PROJECT_ID}/pipelines/${pid}/jobs?per_page=100" | \
    jq -c '.[]' >> /tmp/lumio-jobs.jsonl
done

echo "Total job records: $(wc -l < /tmp/lumio-jobs.jsonl)"
```

Run the script:

```bash
export GITLAB_TOKEN="glpat-xxxxxxxxxxxxxxxxxxxx"
bash scripts/analyze-jobs.sh
```

Expected output:

```
Fetching pipelines since 2024-03-22T10:00:00Z...
Found 312 pipelines. Fetching jobs...
Total job records: 4218
```

**Step 2: Analyze duration — top 10 slowest jobs by average.**

```bash
# Average duration per job name, sorted descending
jq -r 'select(.status == "success") | [.name, (.duration | tostring)] | @tsv' \
  /tmp/lumio-jobs.jsonl | \
  awk '{sum[$1]+=$2; count[$1]++} END {for (k in sum) print sum[k]/count[k], k}' | \
  sort -rn | head -10
```

Expected output:

```
847.3  e2e:chrome
612.1  e2e:firefox
418.9  integration:database
204.5  test:unit-coverage
187.2  docker:build-api
143.7  test:unit-api
98.3   docker:build-frontend
72.1   test:lint
54.4   deploy:staging
31.2   security:sast
```

The E2E jobs are the outliers. The `e2e:chrome` job averages 847 seconds (14 minutes). That is a candidate for parallelization (Phase 4 technique) or selective execution (run only on MR, not on every commit).

**Step 3: Analyze flakiness — top 10 jobs by failure rate.**

```bash
# Failure rate per job name
jq -r '[.name, .status] | @tsv' /tmp/lumio-jobs.jsonl | \
  awk '{
    total[$1]++;
    if ($2 == "failed") fail[$1]++
  } END {
    for (k in total) {
      rate = (fail[k]+0) / total[k] * 100;
      printf "%.1f%%\t%d/%d\t%s\n", rate, fail[k]+0, total[k], k
    }
  }' | sort -rn | head -10
```

Expected output:

```
23.4%   73/312   integration:database
12.1%   38/314   e2e:chrome
8.7%    27/310   test:unit-coverage
4.2%    13/309   docker:build-api
3.1%    10/316   security:dast
1.8%     6/334   test:unit-api
0.9%     3/312   deploy:staging
0.6%     2/312   security:sast
0.3%     1/312   test:lint
0.0%     0/312   deploy:production
```

`integration:database` fails almost 1 in 4 times. This is the kind of finding that was invisible in Jenkins — now it has a number attached to it, and someone owns fixing it.

---

## Challenge 4 — Configure Pipeline Failure Alerts

**Goal:** Set up a GitLab webhook that sends a Slack notification whenever a pipeline on `main` fails, so the team is alerted within seconds rather than discovering the failure on their next commit.

**Step 1: Create a Slack incoming webhook.**

In your Slack workspace, go to **Apps > Incoming Webhooks > Add New Webhook to Workspace**. Select the `#lumio-ci-alerts` channel. Copy the webhook URL:

```
https://hooks.slack.com/services/<T_ID>/<B_ID>/<WEBHOOK_TOKEN>
```

**Step 2: Configure the GitLab webhook.**

In `lumio-api`, go to **Settings > Webhooks > Add new webhook**.

- **URL:** `https://hooks.slack.com/services/<T_ID>/<B_ID>/<WEBHOOK_TOKEN>`
- **Trigger:** Check **Pipeline events**
- **Secret token:** (optional but recommended — set to a random string)
- **SSL verification:** Enabled

Click **Add webhook**.

**Step 3: Test the webhook.**

Click **Test > Pipeline events**. GitLab sends a sample payload to Slack. You should see a message appear in `#lumio-ci-alerts`.

The payload GitLab sends looks like this:

```json
{
  "object_kind": "pipeline",
  "object_attributes": {
    "id": 48274819,
    "ref": "main",
    "status": "failed",
    "duration": 184,
    "finished_at": "2024-04-22T10:45:33.000Z"
  },
  "project": {
    "name": "lumio-api",
    "web_url": "https://gitlab.com/lumio/lumio-api"
  },
  "user": {
    "name": "Sarah Chen",
    "username": "s.chen"
  },
  "commit": {
    "id": "a3f8c1b2d4e5f67890abcdef1234567890abcdef",
    "message": "feat: add invoice export endpoint",
    "url": "https://gitlab.com/lumio/lumio-api/-/commit/a3f8c1b2d4e5f67890abcdef1234567890abcdef"
  }
}
```

**Step 4: Filter to failures on `main` only (optional advanced configuration).**

For more control — for example, suppressing notifications for feature branch failures — deploy a simple filter receiver. A minimal approach using a GitLab CI job itself:

```yaml
# In a separate notification-router project or as a job in the pipeline:
notify:failure:
  stage: notify
  image: curlimages/curl:8.7.1
  script:
    - |
      curl -X POST "${SLACK_WEBHOOK_URL}" \
        -H 'Content-type: application/json' \
        --data "{
          \"text\": \":red_circle: Pipeline failed on \`${CI_COMMIT_BRANCH}\`\",
          \"attachments\": [{
            \"color\": \"danger\",
            \"fields\": [
              {\"title\": \"Project\", \"value\": \"${CI_PROJECT_NAME}\", \"short\": true},
              {\"title\": \"Triggered by\", \"value\": \"${GITLAB_USER_NAME}\", \"short\": true},
              {\"title\": \"Commit\", \"value\": \"${CI_COMMIT_SHORT_SHA} — ${CI_COMMIT_TITLE}\"},
              {\"title\": \"Pipeline\", \"value\": \"${CI_PIPELINE_URL}\"}
            ]
          }]
        }"
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      when: on_failure    # Only runs if a previous job failed
```

This approach gives you full control over message format and filtering logic directly in YAML, without an external service.

---

## Challenge 5 — Export Metrics to Prometheus and Build a Grafana Dashboard

**Goal:** Feed GitLab CI metrics into Prometheus and build a Grafana dashboard with four panels: pipeline duration (p50/p95), weekly success rate, top 10 slowest jobs, and daily production deployments.

**Step 1: Understand the GitLab metrics endpoint.**

GitLab exposes a Prometheus-compatible `/metrics` endpoint on the instance. For GitLab.com users, a per-project API-based approach is more practical.

Create a metrics exporter script that runs on a schedule:

```python
#!/usr/bin/env python3
# scripts/gitlab-metrics-exporter.py
# Exposes GitLab CI metrics in Prometheus format on port 9118.
# Run this on a VM or as a cron job that writes to a Prometheus pushgateway.

import requests
import time
from prometheus_client import start_http_server, Gauge, Counter, Histogram

GITLAB_TOKEN = os.environ["GITLAB_TOKEN"]
PROJECT_ID   = "lumio/lumio-api"
GITLAB_URL   = "https://gitlab.com"

pipeline_duration_p50 = Gauge(
    "gitlab_ci_pipeline_duration_p50_seconds",
    "Median pipeline duration in seconds",
    ["project", "ref"]
)
pipeline_success_rate = Gauge(
    "gitlab_ci_pipeline_success_rate",
    "Percentage of successful pipelines (0–1)",
    ["project", "ref"]
)
deployment_count = Counter(
    "gitlab_ci_deployments_total",
    "Total deployments to named environments",
    ["project", "environment"]
)

def collect_metrics():
    headers = {"PRIVATE-TOKEN": GITLAB_TOKEN}
    # Fetch last 100 pipelines on main
    resp = requests.get(
        f"{GITLAB_URL}/api/v4/projects/{PROJECT_ID}/pipelines",
        headers=headers,
        params={"ref": "main", "per_page": 100}
    )
    pipelines = resp.json()
    durations  = [p["duration"] for p in pipelines if p.get("duration")]
    successes  = [p for p in pipelines if p["status"] == "success"]

    if durations:
        durations.sort()
        p50 = durations[len(durations) // 2]
        pipeline_duration_p50.labels(project="lumio-api", ref="main").set(p50)

    if pipelines:
        rate = len(successes) / len(pipelines)
        pipeline_success_rate.labels(project="lumio-api", ref="main").set(rate)

if __name__ == "__main__":
    start_http_server(9118)
    while True:
        collect_metrics()
        time.sleep(60)
```

**Step 2: Add a scrape config to Prometheus.**

```yaml
# prometheus.yml — add this scrape job
scrape_configs:
  - job_name: 'gitlab-ci'
    static_configs:
      - targets: ['metrics-exporter:9118']
    scrape_interval: 60s
```

**Step 3: Create the Grafana dashboard.**

Import this dashboard JSON into Grafana (**Dashboards > Import > Upload JSON**):

```json
{
  "title": "Lumio CI/CD Health",
  "panels": [
    {
      "title": "Pipeline Duration — p50 and p95",
      "type": "timeseries",
      "targets": [
        {
          "expr": "gitlab_ci_pipeline_duration_p50_seconds{project='lumio-api', ref='main'}",
          "legendFormat": "p50"
        },
        {
          "expr": "histogram_quantile(0.95, rate(gitlab_ci_pipeline_duration_bucket[1h]))",
          "legendFormat": "p95"
        }
      ]
    },
    {
      "title": "Pipeline Success Rate (weekly)",
      "type": "stat",
      "targets": [
        {
          "expr": "avg_over_time(gitlab_ci_pipeline_success_rate{ref='main'}[7d]) * 100",
          "legendFormat": "Success rate %"
        }
      ]
    },
    {
      "title": "Top 10 Slowest Jobs (avg duration)",
      "type": "bargauge",
      "targets": [
        {
          "expr": "topk(10, avg by (job_name) (gitlab_ci_job_duration_seconds))",
          "legendFormat": "{{job_name}}"
        }
      ]
    },
    {
      "title": "Production Deployments per Day",
      "type": "timeseries",
      "targets": [
        {
          "expr": "increase(gitlab_ci_deployments_total{environment='production'}[1d])",
          "legendFormat": "Deployments"
        }
      ]
    }
  ]
}
```

**Step 4: Verify the dashboard populates.**

After running the exporter for a few minutes, all four panels should show data. The **Pipeline Success Rate** stat panel is the one to share with stakeholders — it is a single number that captures CI/CD health at a glance.

---

## Challenge 6 — Comparing with Jenkins

**Goal:** Perform the same health analysis on the Jenkins instance (still running in parallel) to document the observability gap and build the business case for completing the migration.

**Step 1: Attempt DORA metrics in Jenkins.**

Jenkins does not have native DORA metric support. To get Deployment Frequency, you would need:
- The **DORA Metrics** plugin (released 2022, 1,200 installs — tiny compared to Jenkins' ecosystem)
- Or a custom script that queries Jenkins' API and a deployment log

Try it. Navigate to **Jenkins > Manage Jenkins > Plugins > Available** and search for "DORA". Install and configure. You will quickly discover that it requires deployments to be tagged in a specific format that none of Lumio's existing jobs use. The plugin is not retroactive.

**Step 2: Attempt pipeline duration analysis in Jenkins.**

The Jenkins API does have job duration data. Try this:

```bash
# Fetch last 100 builds for lumio-api from Jenkins API
curl -s "http://localhost:8080/job/lumio-api/api/json?tree=builds[number,duration,result]{0,100}" \
  --user "admin:${JENKINS_TOKEN}" | \
  jq '[.builds[] | select(.result == "SUCCESS") | .duration] | 
      add / length / 1000 | . * 100 | round / 100'
```

Expected output (if it works):

```
214.3
```

The result is in seconds, but you had to write a jq pipeline to get it. In GitLab, this number is shown in a graph on a dedicated analytics page with no scripting required.

**Step 3: Document the observability comparison.**

| Capability | Jenkins | GitLab CI |
|---|---|---|
| Pipeline duration chart | Plugin required (Build Timeline) | Built-in (Analytics > CI/CD) |
| Pipeline success rate | Plugin required or manual API query | Built-in |
| DORA metrics | Plugin (limited, not retroactive) | Built-in (Value Stream Analytics) |
| Job-level failure rates | Not available out of the box | Available via API with structured data |
| Prometheus export | Plugin (prometheus-plugin — unmaintained since 2022) | Native `/metrics` endpoint |
| MR-level analytics | Not available | Built-in (time from MR open to merge) |
| Per-environment deployment history | Not available | Built-in (Environments page) |

**Step 4: Send the comparison to the engineering manager.**

The data from this phase answers the question that started this lab: *"How are our pipelines doing?"*

Write a one-page summary using the numbers from Challenge 1 (GitLab analytics), Challenge 2 (DORA metrics), and Challenge 3 (flaky jobs). Include:

1. Current Deployment Frequency and DORA tier.
2. The three flakiest jobs and their failure rates.
3. The slowest job and its impact on developer wait time.
4. One recommended action for each finding.

This is the kind of data-driven engineering conversation that was impossible with Jenkins. It is now a 30-minute exercise.

---

## Outcome

After completing Phase 11, Lumio has full visibility into the health of its CI/CD system:

| Before (Jenkins) | After (GitLab CI) |
|---|---|
| No pipeline duration trends | Analytics dashboard with p50/p95 duration by branch and period |
| No DORA metrics without custom plugins | Native DORA metrics in Value Stream Analytics |
| Flaky tests invisible (no aggregate failure rate) | Top flaky jobs identified: `integration:database` at 23.4% failure rate |
| Alerts only if someone checked Jenkins | Slack notification within 30 seconds of a `main` pipeline failure |
| Prometheus integration required unmaintained plugin | Native metrics exporter, Grafana dashboard with 4 panels |
| Observability required deep Jenkins expertise | GitLab analytics accessible to any team member |

**Deliverables from this phase:**
- Grafana dashboard: `Lumio CI/CD Health` (pipeline duration, success rate, top slow jobs, daily deploys)
- Weekly automated CI/CD health summary (scheduled pipeline that posts metrics to Slack)
- Written list of top 5 improvements to prioritize after Jenkins decommission

**Jobs remaining in Jenkins:** `lumio-frontend/Jenkinsfile.e2e`, `infrastructure/Jenkinsfile.nightly`

---

[Back to main README](../README.md) | [Previous: Phase 10 — GitOps and Infrastructure](../phase-10-gitops/README.md) | [Next: Phase 12 — Capstone: Decommission Jenkins](../phase-12-capstone/README.md)
