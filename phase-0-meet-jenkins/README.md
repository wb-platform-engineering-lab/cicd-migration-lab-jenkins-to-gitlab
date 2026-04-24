# Phase 0 — Meet Jenkins

> **Tools:** Docker, Jenkins, Configuration as Code (JCasC) | **Cost:** $0

---

## The Problem

You cannot migrate what you do not understand.

Lumio's Jenkins setup has been accumulating for eight years. Before touching a single pipeline, you need to answer five questions:

1. What does the Jenkins setup actually look like when it runs?
2. What jobs exist, what do they do, and how are they structured?
3. Where are credentials stored — and are they safe?
4. What does the shared Groovy library do, job by job?
5. What happens when Jenkins fails in the middle of a build?

The answers to these questions define your migration plan. They also reveal why the migration is necessary.

This phase has no GitLab CI content. It is entirely about understanding the starting point before you change it.

```mermaid
flowchart TD
    A["Phase 0 Goal:\nUnderstand Lumio's Jenkins"] --> B["Challenge 1\nRun Jenkins locally"]
    A --> C["Challenge 2\nExplore existing jobs"]
    A --> D["Challenge 3\nAudit credentials"]
    A --> E["Challenge 4\nMap shared library dependencies"]
    A --> F["Challenge 5\nDocument failure modes"]
    B & C & D & E & F --> G["Outcome:\nMigration backlog created\nRisks identified\nPhase 1 can begin"]
```

---

## Challenge 1 — Start Jenkins Locally with Docker Compose

**Goal:** Run Lumio's Jenkins setup on your machine and verify that all jobs load correctly.

Jenkins Configuration as Code (JCasC) lets you define Jenkins configuration in YAML rather than clicking through the UI. Lumio uses it so the controller state is reproducible — you can spin it up fresh and get the same 154-job setup every time.

### Steps

**1. Clone the lab repository and navigate to the Jenkins directory.**

> **Note:** The `jenkins/` directory contains the starting state files (Docker Compose, JCasC config, Jenkinsfiles). Create it at the repo root before running the commands below — see the [repository structure](../README.md#repository-structure) for the expected layout.

```bash
git clone https://github.com/wb-platform-engineering-lab/cicd-migration-lab-jenkins-to-gitlab.git
cd cicd-migration-lab-jenkins-to-gitlab/jenkins
```

**2. Review the Docker Compose file before starting.**

```yaml
# jenkins/docker-compose.yml
version: "3.8"

services:
  jenkins-controller:
    image: jenkins/jenkins:lts
    container_name: lumio-jenkins
    restart: unless-stopped
    ports:
      - "8080:8080"     # Jenkins web UI
      - "50000:50000"   # JNLP agent connection port
    environment:
      - JAVA_OPTS=-Djenkins.install.runSetupWizard=false
      - CASC_JENKINS_CONFIG=/var/jenkins_home/casc
    volumes:
      - jenkins_home:/var/jenkins_home
      - ./casc:/var/jenkins_home/casc:ro
      - ./jobs:/var/jenkins_home/jobs:ro
      - /var/run/docker.sock:/var/run/docker.sock
    networks:
      - jenkins-net

  jenkins-agent:
    image: jenkins/inbound-agent:latest
    container_name: lumio-jenkins-agent
    restart: unless-stopped
    environment:
      - JENKINS_URL=http://jenkins-controller:8080
      - JENKINS_AGENT_NAME=docker-agent-01
      - JENKINS_SECRET=${AGENT_SECRET:-changeme}
      - JENKINS_AGENT_WORKDIR=/home/jenkins/agent
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - agent_workspace:/home/jenkins/agent
    depends_on:
      - jenkins-controller
    networks:
      - jenkins-net

volumes:
  jenkins_home:
  agent_workspace:

networks:
  jenkins-net:
    driver: bridge
```

**3. Review the JCasC configuration file.**

```yaml
# jenkins/casc/jenkins.yaml
jenkins:
  systemMessage: "Lumio Jenkins — managed by JCasC. Do not edit via UI."
  numExecutors: 0  # No builds on controller
  agentProtocols:
    - "JNLP4-connect"

  securityRealm:
    local:
      allowsSignup: false
      users:
        - id: "admin"
          password: "admin"
        - id: "lumio-readonly"
          password: "readonly"

  authorizationStrategy:
    loggedInUsersCanDoAnything:
      allowAnonymousRead: false

  clouds:
    - docker:
        name: "docker-local"
        dockerApi:
          dockerHost:
            uri: "unix:///var/run/docker.sock"
        templates:
          - labelString: "docker-agent"
            dockerTemplateBase:
              image: "jenkins/agent:latest-jdk17"
            remoteFs: "/home/jenkins/agent"
            connector:
              attach:
                user: "jenkins"

credentials:
  system:
    domainCredentials:
      - credentials:
          - usernamePassword:
              scope: GLOBAL
              id: "ecr-push-credentials"
              username: "AWS"
              password: "${ECR_PUSH_PASSWORD}"
              description: "AWS ECR push credentials"
          - string:
              scope: GLOBAL
              id: "slack-webhook-url"
              secret: "${SLACK_WEBHOOK_URL}"
              description: "Slack incoming webhook"
          - usernamePassword:
              scope: GLOBAL
              id: "lumio-db-staging"
              username: "lumio_app"
              password: "staging_db_password_plain"   # <-- plain text, flagged in audit
              description: "Staging database credentials"

tool:
  git:
    installations:
      - name: "Default"
        home: "git"
  nodejs:
    installations:
      - name: "NodeJS-18"
        version: "18.19.0"

unclassified:
  location:
    url: "http://localhost:8080/"
    adminAddress: "devops@lumio.io"
  globalLibraries:
    libraries:
      - name: "lumio-shared"
        retriever:
          modernSCM:
            scm:
              git:
                remote: "https://github.com/lumio/jenkins-shared-library.git"
                credentialsId: "github-token"
```

**4. Start Jenkins.**

```bash
docker compose up -d
```

**5. Wait for Jenkins to become ready.** The first boot takes 60–90 seconds while plugins are loaded from the image.

```bash
docker compose logs -f jenkins-controller | grep -E "(Jenkins is fully up|ERROR|SEVERE)"
```

Expected output when Jenkins is ready:

```
lumio-jenkins  | 2024-01-15 09:23:41.847+0000 [id=36] INFO    jenkins.InitReactorRunner$1#onAttained: Completed initialization
lumio-jenkins  | 2024-01-15 09:23:42.103+0000 [id=1]  INFO    hudson.lifecycle.Lifecycle#onReady: Jenkins is fully up and running
```

**6. Verify Jenkins is accessible.**

```bash
curl -s -o /dev/null -w "%{http_code}" http://admin:admin@localhost:8080/api/json
```

Expected output:

```
200
```

**7. Check that the jobs loaded from JCasC.**

```bash
curl -s http://admin:admin@localhost:8080/api/json?tree=jobs\[name\] | python3 -m json.tool
```

Expected output:

```json
{
    "jobs": [
        { "name": "lumio-api" },
        { "name": "lumio-frontend" },
        { "name": "lumio-worker" },
        { "name": "infrastructure" }
    ]
}
```

**8. Open the Jenkins UI** at [http://localhost:8080](http://localhost:8080) — log in with `admin` / `admin`.

---

## Challenge 2 — Explore the Existing Jobs

**Goal:** Understand the structure and purpose of Lumio's 154 jobs without running any of them.

### Steps

**1. List all jobs across all folders using the Jenkins API.**

```bash
curl -s "http://admin:admin@localhost:8080/api/json?tree=jobs[name,jobs[name,jobs[name]]]" \
  | python3 -m json.tool
```

Expected output (abbreviated):

```json
{
    "jobs": [
        {
            "name": "lumio-api",
            "jobs": [
                { "name": "lumio-api-build" },
                { "name": "lumio-api-deploy-staging" },
                { "name": "lumio-api-deploy-prod" },
                { "name": "lumio-api-db-migrate" }
            ]
        },
        {
            "name": "lumio-frontend",
            "jobs": [
                { "name": "lumio-frontend-build" },
                { "name": "lumio-frontend-e2e" }
            ]
        },
        {
            "name": "lumio-worker",
            "jobs": [
                { "name": "lumio-worker-build" }
            ]
        },
        {
            "name": "infrastructure",
            "jobs": [
                { "name": "infra-terraform-plan" },
                { "name": "infra-terraform-apply" },
                { "name": "infra-nightly-cleanup" }
            ]
        }
    ]
}
```

**2. Identify the three categories of jobs.** Based on the folder structure, jobs fall into:

| Category | Folder | Job count | Purpose |
|---|---|---|---|
| Application | `lumio-api`, `lumio-frontend`, `lumio-worker` | ~130 | Build, test, push Docker images, deploy |
| Infrastructure | `infrastructure` | ~15 | Terraform, nightly cleanup |
| Utilities | _(scattered)_ | ~9 | Notifications, cache warmup, health checks |

**3. Read the main Jenkinsfile for `lumio-api`.**

```bash
cat jenkins/jobs/lumio-api/Jenkinsfile
```

```groovy
@Library('lumio-shared@main') _

pipeline {
    agent { label 'docker-agent' }

    environment {
        APP_NAME    = 'lumio-api'
        ECR_REPO    = '123456789.dkr.ecr.eu-west-1.amazonaws.com/lumio-api'
        NODE_ENV    = 'test'
    }

    stages {
        stage('Install') {
            steps {
                sh 'npm ci --prefer-offline'
            }
        }

        stage('Lint') {
            steps {
                sh 'npm run lint'
            }
        }

        stage('Test') {
            steps {
                runTests(
                    reportDir: 'coverage',
                    junitFile: 'test-results/junit.xml'
                )
            }
        }

        stage('Build Docker') {
            when { anyOf { branch 'main'; branch 'release/*' } }
            steps {
                buildDocker(
                    imageName: env.ECR_REPO,
                    tag:       env.GIT_COMMIT[0..7]
                )
            }
        }

        stage('Deploy Staging') {
            when { branch 'main' }
            steps {
                deployToEnv(
                    app:   env.APP_NAME,
                    env:   'staging',
                    image: "${env.ECR_REPO}:${env.GIT_COMMIT[0..7]}"
                )
            }
        }
    }

    post {
        always {
            notifySlack(status: currentBuild.result ?: 'SUCCESS')
        }
        failure {
            mail to: 'devops@lumio.io',
                 subject: "FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                 body: "Pipeline failed. Check: ${env.BUILD_URL}"
        }
    }
}
```

**4. Identify the shared library calls.** The Jenkinsfile uses four abstractions from the `lumio-shared` library:

| Call | File in shared library | What it does |
|---|---|---|
| `@Library('lumio-shared@main') _` | N/A — import declaration | Loads the library from git |
| `runTests(...)` | `vars/runTests.groovy` | Runs Jest, publishes JUnit results |
| `buildDocker(...)` | `vars/buildDocker.groovy` | Docker build + ECR push |
| `deployToEnv(...)` | `vars/deployToEnv.groovy` | Parameterized deploy to staging/prod |
| `notifySlack(...)` | `vars/notifySlack.groovy` | Posts build result to Slack |

**5. Note what you cannot see from the Jenkinsfile alone.** Each shared library call hides complexity. You will need to read the Groovy files in Challenge 4 to understand what is actually running.

---

## Challenge 3 — Audit Jenkins Credentials

**Goal:** Identify all credentials stored in Jenkins and flag the ones that are a security risk.

### Steps

**1. List all credentials via the Jenkins API.**

```bash
curl -s \
  "http://admin:admin@localhost:8080/credentials/store/system/domain/_/api/json?pretty=true&depth=2"
```

Expected output (abbreviated):

```json
{
  "credentials": [
    {
      "id": "ecr-push-credentials",
      "typeName": "Username with password",
      "description": "AWS ECR push credentials",
      "displayName": "ecr-push-credentials"
    },
    {
      "id": "slack-webhook-url",
      "typeName": "Secret text",
      "description": "Slack incoming webhook",
      "displayName": "slack-webhook-url"
    },
    {
      "id": "lumio-db-staging",
      "typeName": "Username with password",
      "description": "Staging database credentials",
      "displayName": "lumio-db-staging"
    },
    {
      "id": "lumio-db-prod",
      "typeName": "Username with password",
      "description": "Production database credentials",
      "displayName": "lumio-db-prod"
    },
    {
      "id": "github-token",
      "typeName": "Secret text",
      "description": "GitHub personal access token",
      "displayName": "github-token"
    }
  ]
}
```

**2. Pull the raw job config XML for a job suspected of using plain-text credentials.**

```bash
curl -s http://admin:admin@localhost:8080/job/lumio-api/job/lumio-api-deploy-prod/config.xml
```

Look for patterns like this in the output — plain-text passwords directly in job configuration:

```xml
<hudson.model.PasswordParameterDefinition>
  <name>DB_PASSWORD</name>
  <defaultValue>prod_lumio_v2_secure123</defaultValue>   <!-- ⚠ plain text -->
</hudson.model.PasswordParameterDefinition>
```

**3. Write a script to scan all job configs for plain-text credential patterns.**

```bash
#!/bin/bash
# scan-credentials.sh — finds plain-text secrets in Jenkins job XML configs

JENKINS_URL="http://admin:admin@localhost:8080"
FLAGGED=0

jobs=$(curl -s "${JENKINS_URL}/api/json?tree=jobs[name,jobs[name]]" \
  | python3 -c "
import json, sys
data = json.load(sys.stdin)
for folder in data['jobs']:
    for job in folder.get('jobs', []):
        print(f\"{folder['name']}/{job['name']}\")
")

for job in $jobs; do
  config=$(curl -s "${JENKINS_URL}/job/${job//\//\/job\/}/config.xml")
  if echo "$config" | grep -qE '<defaultValue>[^<]{8,}</defaultValue>'; then
    echo "⚠  FLAGGED: $job — potential plain-text credential in defaultValue"
    ((FLAGGED++))
  fi
done

echo ""
echo "Scan complete. $FLAGGED jobs flagged."
```

```bash
chmod +x scan-credentials.sh
./scan-credentials.sh
```

Expected output:

```
⚠  FLAGGED: lumio-api/lumio-api-deploy-prod — potential plain-text credential in defaultValue
⚠  FLAGGED: lumio-api/lumio-api-db-migrate — potential plain-text credential in defaultValue
⚠  FLAGGED: lumio-frontend/lumio-frontend-build — potential plain-text credential in defaultValue
⚠  FLAGGED: infrastructure/infra-terraform-apply — potential plain-text credential in defaultValue

Scan complete. 14 jobs flagged.
```

**4. Record the audit findings.** These 14 jobs will be specifically addressed in **Phase 5 — Secrets and Variables**, where each credential will be moved to a masked, protected GitLab CI variable.

> **Security note:** The Jenkins credential store encrypts credentials at rest in `credentials.xml` on disk. However, the *default value fields in parameterized builds* are stored unencrypted in `config.xml`. This is a well-known Jenkins security gap. GitLab CI variables with the `masked` flag prevent secrets from appearing in job logs entirely.

---

## Challenge 4 — Map the Shared Library Dependencies

**Goal:** Read every `.groovy` file in the shared library and create a Jenkins-to-GitLab CI concept map.

### Steps

**1. List all vars in the shared library.**

```bash
ls jenkins/shared-library/vars/
```

```
buildDocker.groovy
deployToEnv.groovy
notifySlack.groovy
runTests.groovy
```

**2. Read `buildDocker.groovy` in full.**

```groovy
// jenkins/shared-library/vars/buildDocker.groovy
import com.lumio.DockerUtils
import com.lumio.AwsUtils

def call(Map config = [:]) {
    def imageName  = config.imageName ?: error("buildDocker: imageName is required")
    def tag        = config.tag       ?: env.GIT_COMMIT[0..7]
    def dockerfile = config.dockerfile ?: 'Dockerfile'
    def buildArgs  = config.buildArgs ?: [:]
    def push       = config.push      ?: true

    stage("Docker Build: ${imageName}:${tag}") {
        AwsUtils.ecrLogin(this)  // runs: aws ecr get-login-password | docker login

        def buildArgStr = buildArgs.collect { k, v -> "--build-arg ${k}=${v}" }.join(' ')

        sh """
            docker build \
              -f ${dockerfile} \
              -t ${imageName}:${tag} \
              -t ${imageName}:latest \
              ${buildArgStr} \
              .
        """

        if (push) {
            sh "docker push ${imageName}:${tag}"
            sh "docker push ${imageName}:latest"
        }
    }

    DockerUtils.cleanupLocalImage(this, imageName, tag)
    return "${imageName}:${tag}"
}
```

**3. Read `deployToEnv.groovy` in full.**

```groovy
// jenkins/shared-library/vars/deployToEnv.groovy
import com.lumio.AwsUtils

def call(Map config = [:]) {
    def app    = config.app    ?: error("deployToEnv: app is required")
    def env    = config.env    ?: error("deployToEnv: env is required")
    def image  = config.image  ?: error("deployToEnv: image is required")
    def region = config.region ?: 'eu-west-1'

    def ALLOWED_ENVS = ['staging', 'production']
    if (!ALLOWED_ENVS.contains(env)) {
        error("deployToEnv: env must be one of ${ALLOWED_ENVS}, got '${env}'")
    }

    stage("Deploy ${app} to ${env}") {
        if (env == 'production') {
            input message: "Deploy ${image} to PRODUCTION?", ok: "Deploy"
        }

        withCredentials([string(credentialsId: "deploy-token-${env}", variable: 'DEPLOY_TOKEN')]) {
            sh """
                curl -X POST \
                  -H "Authorization: Bearer ${DEPLOY_TOKEN}" \
                  -H "Content-Type: application/json" \
                  -d '{"image": "${image}", "app": "${app}"}' \
                  https://deploy.lumio.io/api/v1/deploy/${env}
            """
        }
    }
}
```

**4. Read `runTests.groovy` in full.**

```groovy
// jenkins/shared-library/vars/runTests.groovy

def call(Map config = [:]) {
    def reportDir = config.reportDir ?: 'coverage'
    def junitFile = config.junitFile ?: 'test-results/junit.xml'
    def threshold = config.threshold ?: 80

    stage('Run Tests') {
        try {
            sh 'npm test -- --ci --coverage --reporters=jest-junit'
        } catch (err) {
            currentBuild.result = 'UNSTABLE'
            throw err
        } finally {
            junit junitFile
            publishHTML([
                allowMissing:          false,
                alwaysLinkToLastBuild: true,
                keepAll:               true,
                reportDir:             reportDir,
                reportFiles:           'index.html',
                reportName:            'Coverage Report'
            ])
        }
    }
}
```

**5. Read `notifySlack.groovy` in full.**

```groovy
// jenkins/shared-library/vars/notifySlack.groovy

def call(Map config = [:]) {
    def status  = config.status  ?: 'SUCCESS'
    def channel = config.channel ?: '#ci-notifications'

    def color = [
        SUCCESS:  'good',
        FAILURE:  'danger',
        UNSTABLE: 'warning'
    ][status] ?: 'warning'

    def message = "${env.JOB_NAME} #${env.BUILD_NUMBER} — ${status}\n${env.BUILD_URL}"

    withCredentials([string(credentialsId: 'slack-webhook-url', variable: 'SLACK_URL')]) {
        sh """
            curl -s -X POST ${SLACK_URL} \
              -H 'Content-type: application/json' \
              --data '{"attachments":[{"color":"${color}","text":"${message}"}]}'
        """
    }
}
```

**6. Visualize the dependency graph between shared library components.**

```mermaid
graph TD
    subgraph Jenkinsfiles["Jenkinsfiles (consumers)"]
        JF1["lumio-api/Jenkinsfile"]
        JF2["lumio-frontend/Jenkinsfile"]
        JF3["lumio-worker/Jenkinsfile"]
    end

    subgraph Vars["shared-library/vars/"]
        BD["buildDocker.groovy"]
        DE["deployToEnv.groovy"]
        RT["runTests.groovy"]
        NS["notifySlack.groovy"]
    end

    subgraph Src["shared-library/src/com/lumio/"]
        DU["DockerUtils.groovy\n(ecrLogin, cleanupLocalImage)"]
        AU["AwsUtils.groovy\n(ecrLogin, assumeRole)"]
        SU["SecretsUtils.groovy\n(getSecret from AWS SM)"]
    end

    JF1 --> BD & DE & RT & NS
    JF2 --> BD & RT & NS
    JF3 --> BD & DE & NS

    BD --> DU & AU
    DE --> AU & SU
    RT --> NS
```

**7. Create the Jenkins concept → GitLab CI equivalent table.**

| Jenkins concept | Groovy file | GitLab CI equivalent | Phase |
|---|---|---|---|
| `@Library('lumio-shared')` | _import_ | `include: project: lumio/lumio-ci-templates` | Phase 3 |
| `buildDocker()` | `vars/buildDocker.groovy` | `include: templates/docker-build.yml` + `extends: .docker-build` | Phase 3 |
| `deployToEnv()` | `vars/deployToEnv.groovy` | `include: templates/deploy.yml` + `environment:` + `when: manual` | Phase 7 |
| `runTests()` | `vars/runTests.groovy` | `artifacts: reports: junit:` + coverage report | Phase 2 |
| `notifySlack()` | `vars/notifySlack.groovy` | `after_script:` with `curl` or GitLab Slack integration | Phase 2 |
| `AwsUtils.ecrLogin()` | `src/com/lumio/AwsUtils.groovy` | CI variable `AWS_ACCESS_KEY_ID` + `before_script: aws ecr get-login-password` | Phase 6 |
| `SecretsUtils.getSecret()` | `src/com/lumio/SecretsUtils.groovy` | GitLab-Vault integration or CI variable | Phase 5 |
| `input message: 'Deploy?'` | `vars/deployToEnv.groovy` | `when: manual` on deploy job | Phase 7 |
| `withCredentials([...])` | `vars/deployToEnv.groovy` | Masked CI variable injected automatically | Phase 5 |
| `publishHTML(...)` | `vars/runTests.groovy` | `artifacts: paths: [coverage/]` + pages or link in MR | Phase 2 |

---

## Challenge 5 — Document Jenkins Failure Modes

**Goal:** Observe what happens when Jenkins fails mid-build, and understand why this matters for the migration.

### Steps

**1. Start a long-running build to simulate in-flight work.**

Trigger a manual build from the Jenkins UI or via the API:

```bash
curl -X POST \
  "http://admin:admin@localhost:8080/job/lumio-api/job/lumio-api-build/build" \
  --data-urlencode json=''
```

Wait 10–15 seconds until the build is executing (you will see it running in the UI).

**2. Kill the Jenkins controller container mid-build.**

```bash
docker stop lumio-jenkins
```

Expected output:

```
lumio-jenkins
```

**3. Observe what the agent does when the controller disappears.**

```bash
docker compose logs jenkins-agent --tail=20
```

Expected output:

```
lumio-jenkins-agent  | WARNING: Connection was broken: java.io.IOException: Unexpected termination of the channel
lumio-jenkins-agent  | INFO: Attempting to reconnect in 10 seconds
lumio-jenkins-agent  | INFO: Attempting to reconnect in 20 seconds
lumio-jenkins-agent  | INFO: Attempting to reconnect in 40 seconds
lumio-jenkins-agent  | WARNING: Failed to connect to http://jenkins-controller:8080 after 3 attempts
```

The agent retries indefinitely. The build result is lost.

**4. Restart Jenkins.**

```bash
docker compose up -d jenkins-controller
docker compose logs -f jenkins-controller | grep "Jenkins is fully up"
```

**5. Check the status of the in-flight build.**

In the Jenkins UI, navigate to the `lumio-api-build` job. Observe that the build that was running now shows as:

- Status: **ABORTED** (if the controller caught the shutdown signal)
- Status: **(no result)** / stuck "building" — if the kill was abrupt

The build queue may show jobs in a `Waiting` or `Blocked` state that require manual intervention to clear.

**6. Compare with GitLab CI behavior.**

| Scenario | Jenkins behavior | GitLab CI behavior |
|---|---|---|
| Controller goes down mid-build | Build result is lost; agent reconnects indefinitely | GitLab.com is SaaS — controller never goes down for you |
| Agent disconnects mid-build | Build marked failed or stuck | Job is automatically retried (configurable with `retry:`) |
| Queue after restart | Jobs may be stuck; manual clear needed | No controller restart needed (SaaS); self-managed auto-recovers |
| Build history after failure | Build may show no result | Job always has a final state; logs are preserved |
| Single point of failure | Jenkins controller is SPOF | No SPOF on GitLab.com; self-managed can use HA |

**7. Note the operational burden this creates.** Every time someone at Lumio had to restart Jenkins, an engineer had to:

1. Check which builds were in-flight
2. Manually re-trigger any that were lost
3. Confirm deployments that may or may not have completed
4. Notify teams whose pipelines were silently broken

This is part of what the migration is eliminating.

---

## Outcome

By the end of Phase 0, you have a full picture of what Lumio has and what needs to change:

| Problem identified | Severity | Migration phase |
|---|---|---|
| Jenkins controller is a single point of failure | High | Resolved by moving to GitLab CI (Phase 1) |
| 14 jobs store credentials in plain text | Critical | Phase 5 — Secrets and Variables |
| Shared library is 12,000 lines of Groovy understood by 2 people | High | Phase 3 — Shared Libraries → Templates |
| Killing the controller mid-build loses build history | Medium | Resolved by GitLab CI's SaaS model (Phase 1) |
| 47 plugins installed, 11 with pending security advisories | High | Resolved by decommissioning Jenkins (Phase 12) |
| No concept mapping between Jenkins and GitLab CI exists | Medium | Phase 2 — Pipeline Anatomy |
| Shared library calls hide complexity from pipeline authors | Medium | Phase 3 — Shared Libraries → Templates |
| New engineers cannot onboard without reading Groovy | Medium | Resolved by YAML pipelines (Phase 2+) |
| Manual approval in `deployToEnv()` is not visible in UI | Medium | Phase 7 — Environments and Deployment Gates |
| Agent-controller connection is fragile | Low | Phase 9 — GitLab Runners |

The migration backlog is now defined. Phase 1 begins the GitLab setup.

---

[Back to main README](../README.md) | [Next: Phase 1 — GitLab Foundations](../phase-1-gitlab-foundations/README.md)
