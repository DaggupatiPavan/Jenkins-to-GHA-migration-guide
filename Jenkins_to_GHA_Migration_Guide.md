# Jenkins to GitHub Actions Migration Guide

This guide is designed for software engineers with Jenkins experience who are transitioning to GitHub Actions (GHA). It covers the fundamental conceptual shifts, syntax mappings, and advanced patterns like reusable workflows and custom actions.

## Table of Contents
1.  [Conceptual Mapping: Jenkins vs. GitHub Actions](#1-conceptual-mapping-jenkins-vs-github-actions)
2.  [The Basics: Syntax & Structure](#2-the-basics-syntax--structure)
3.  [Intermediate: Triggers, Secrets, and Artifacts](#3-intermediate-triggers-secrets-and-artifacts)
4.  [Advanced: Reusability & Custom Actions](#4-advanced-reusability--custom-actions)
5.  [Deep Dive: Migrating Shared Libraries](#5-deep-dive-migrating-shared-libraries)
6.  [Decision Guide: Reusable Workflows vs. Custom Actions](#6-decision-guide-reusable-workflows-vs-custom-actions)
7.  [Comprehensive Feature Mapping (Reference)](#7-comprehensive-feature-mapping-reference)
8.  [The "Missing" Pieces (Common Patterns)](#8-the-missing-pieces-common-patterns)
9.  [Advanced Infrastructure: ARC (Actions Runner Controller)](#9-advanced-infrastructure-arc-actions-runner-controller)
10. [GHA Native "Superpowers" (New Capabilities)](#10-gha-native-superpowers-new-capabilities)
11. [Migration Strategy & Best Practices](#11-migration-strategy--best-practices)

---

## 1. Conceptual Mapping: Jenkins vs. GitHub Actions

The biggest hurdle is often terminology. Here is how Jenkins concepts map to GitHub Actions:

| Jenkins Concept | GitHub Actions Concept | Key Differences |
| :--- | :--- | :--- |
| **Pipeline** | **Workflow** | Defined in `.github/workflows/*.yaml`. |
| **Node / Agent** | **Runner** | Can be GitHub-hosted (Ubuntu, Windows, Mac) or Self-hosted. |
| **Stage** | **Job** | **Crucial:** GHA Jobs run in **parallel** by default. You must use `needs:` to enforce ordering. |
| **Step** | **Step** | Steps within a job run sequentially on the same runner. |
| **Shared Library** | **Reusable Workflow / Custom Action** | Reusable logic is handled via `uses:` keyword. |
| **Plugin** | **Action** | Most plugins are replaced by "Actions" from the Marketplace. |
| **Jenkinsfile** | **YAML Workflow** | GHA uses YAML, not Groovy. |

---

## 2. The Basics: Syntax & Structure

### Jenkinsfile (Declarative)
```groovy
pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                sh 'npm install'
                sh 'npm run build'
            }
        }
        stage('Test') {
            steps {
                sh 'npm test'
            }
        }
    }
}
```

### GitHub Actions Workflow (`.github/workflows/main.yaml`)
```yaml
name: CI Pipeline

on: [push] # Trigger

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4 # Required to fetch code
      - name: Install & Build
        run: |
          npm install
          npm run build

  test:
    needs: build # Enforces sequential order (Build -> Test)
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run Tests
        run: npm test
```

> [!IMPORTANT]
> **Workspace Persistence**: In Jenkins, the workspace persists between stages on the same node. In GHA, **each Job runs on a fresh runner**. If Job B needs files from Job A, Job A must `upload-artifact` and Job B must `download-artifact`.

---

## 3. Intermediate: Triggers, Secrets, and Artifacts

### Triggers (`on:`)
Jenkins relies on polling or webhooks. GHA is event-driven.

*   **Push/PR**:
    ```yaml
    on:
      push:
        branches: [ "main" ]
      pull_request:
        branches: [ "main" ]
    ```
*   **Cron (Scheduled)**:
    ```yaml
    on:
      schedule:
        - cron: '0 0 * * *' # Midnight daily
    ```
*   **Manual (Build with Parameters)**:
    ```yaml
    on:
      workflow_dispatch:
        inputs:
          logLevel:
            description: 'Log level'
            required: true
            default: 'warning'
    ```

### Secrets
**Jenkins**: Credentials Binding Plugin.
**GHA**: Repository/Organization Secrets.

Usage:
```yaml
steps:
  - name: Deploy
    env:
      API_KEY: ${{ secrets.PROD_API_KEY }}
    run: ./deploy.sh
```

### Artifacts (Passing data between jobs)
Since jobs are isolated, use artifacts to pass build outputs.

**Job 1: Build**
```yaml
- uses: actions/upload-artifact@v4
  with:
    name: build-output
    path: dist/
```

**Job 2: Deploy**
```yaml
- uses: actions/download-artifact@v4
  with:
    name: build-output
    path: dist/
```

---

## 4. Advanced: Reusability & Custom Actions

This is where GHA shines. Instead of complex Groovy Shared Libraries, you have two main options:

### A. Reusable Workflows (`workflow_call`)
Best for: **Orchestrating entire pipelines** (e.g., a standard "Deploy to K8s" pipeline used by 50 microservices).

**The Template (called workflow):** `.github/workflows/deploy-template.yaml`
```yaml
name: Reusable Deploy
on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
    secrets:
      token:
        required: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying to ${{ inputs.environment }}"
```

**The Caller:**
```yaml
jobs:
  prod-deploy:
    uses: ./.github/workflows/deploy-template.yaml
    with:
      environment: production
    secrets:
      token: ${{ secrets.DEPLOY_TOKEN }}
```

### B. Custom Actions (Composite)
Best for: **Encapsulating steps** (e.g., "Setup Python + Install Private Deps"). Replaces utility functions in Jenkins Shared Libs.

**File Structure:**
```
.github/actions/setup-app/
  action.yml
```

**`action.yml`**:
```yaml
name: 'Setup App'
description: 'Sets up Node and installs deps'
inputs:
  node-version:
    required: true
    default: '18'
runs:
  using: "composite"
  steps:
    - uses: actions/setup-node@v4
      with:
        node-version: ${{ inputs.node-version }}
    - run: npm ci
      shell: bash
```

**Usage in Workflow:**
```yaml
steps:
  - uses: ./.github/actions/setup-app
    with:
      node-version: '20'
```

---

## 5. Deep Dive: Migrating Shared Libraries

This section addresses the most complex part of migration: moving from Groovy Shared Libraries to GHA patterns.

### Scenario A: Simple Utility Functions
**Jenkins (`vars/logInfo.groovy`):**
```groovy
def call(String message) {
    echo "INFO: ${message}"
}
```

**GitHub Actions:** Use a **Composite Action**.
Create `.github/actions/log-info/action.yml`:
```yaml
name: 'Log Info'
inputs:
  message:
    required: true
runs:
  using: "composite"
  steps:
    - run: echo "INFO: ${{ inputs.message }}"
      shell: bash
```

### Scenario B: Complex Logic with Maps (The "Config" Pattern)
**Jenkins:** Passing a map to a function.
```groovy
// vars/deployApp.groovy
def call(Map config) {
    if (config.dryRun) { ... }
    sh "deploy --target ${config.target}"
}

// Jenkinsfile
deployApp(target: 'prod', dryRun: true)
```

**GitHub Actions:** GHA inputs are **strings**. You cannot pass a map directly.
**Solution:** Pass a JSON string and parse it, OR break it into individual inputs.

**Option 1: Individual Inputs (Recommended for simple maps)**
```yaml
inputs:
  target:
    required: true
  dryRun:
    type: boolean
    default: false
```

**Option 2: JSON Object (For complex/nested data)**
Use `fromJSON` to parse the input.

**Caller:**
```yaml
with:
  config: '{"target": "prod", "dryRun": true}'
```

**Action (`action.yml`):**
```yaml
inputs:
  config:
    required: true
runs:
  using: "composite"
  steps:
    - run: |
        TARGET=${{ fromJSON(inputs.config).target }}
        echo "Deploying to $TARGET"
      shell: bash
```

### Scenario C: Pipeline Wrappers (`call(body)`)
**Jenkins:** Wrapping the entire pipeline to enforce standards.
```groovy
// vars/standardPipeline.groovy
def call(body) {
    pipeline {
        agent any
        stages {
            stage('Setup') { ... }
            body() // Execute user logic
        }
    }
}
```

**GitHub Actions:** Use **Reusable Workflows** with `jobs`.
You cannot "wrap" a workflow like in Jenkins, but you can define a standard workflow that calls other workflows or actions.

**Standard Workflow (`.github/workflows/standard.yaml`):**
```yaml
name: Standard Pipeline
on:
  workflow_call:
    inputs:
      image:
        required: true
        type: string

jobs:
  setup:
    runs-on: ubuntu-latest
    steps: ...

  build:
    needs: setup
    runs-on: ubuntu-latest
    steps: ...
```

**Caller:**
```yaml
jobs:
  ci:
    uses: ./.github/workflows/standard.yaml
    with:
      image: 'node:18'
```

### Handling "Global Variables"
**Jenkins:** `env.MY_GLOBAL = 'value'` available everywhere.
**GHA:**
1.  **Job Level:** Use `env:` at the top of the workflow.
2.  **Step to Step:** Use `$GITHUB_ENV`.
    ```bash
    echo "MY_VAR=value" >> $GITHUB_ENV
    ```
3.  **Job to Job:** Use `outputs`.
    **Job 1:**
    ```yaml
    outputs:
      my-output: ${{ steps.step1.outputs.test }}
    steps:
      - id: step1
        run: echo "test=hello" >> $GITHUB_OUTPUT
    ```
    **Job 2:**
    ```yaml
    needs: job1
    steps:
      - run: echo ${{ needs.job1.outputs.my-output }}
    ```

---

## 6. Decision Guide: Reusable Workflows vs. Custom Actions

When migrating from Jenkins Shared Libraries, choosing between **Reusable Workflows** and **Custom Actions** can be confusing. Use this guide to decide.

### Rule of Thumb
*   **Use a Custom Action (Composite)** if you want to **encapsulate a sequence of steps** that run *inside* a job (like a function call).
    *   *Think: "I need a smarter `npm install` or `setup-env` step."*
*   **Use a Reusable Workflow** if you want to **standardize an entire job or pipeline** (like a template).
    *   *Think: "I need a standard 'Build & Push Docker' pipeline that every team uses."*

### Comparative Examples

#### Example 1: Slack Notification
**Goal:** Send a notification at the end of a build.
**Jenkins:** `notifySlack(message: 'Success')`

*   **Decision:** **Custom Action**. It's a single step running inside a job.
*   **Implementation:** `.github/actions/slack-notify/action.yml`
    ```yaml
    runs:
      using: "composite"
      steps:
        - run: curl -X POST -d ...
          shell: bash
    ```
*   **Usage:**
    ```yaml
    steps:
      - uses: ./.github/actions/slack-notify
    ```

#### Example 2: Standard Java Build
**Goal:** Checkout, setup Java, build with Maven, cache dependencies.
**Jenkins:** `standardJavaBuild()`

*   **Decision:** **Reusable Workflow**. It defines the entire "Build" job structure.
*   **Implementation:** `.github/workflows/java-build.yaml`
    ```yaml
    on: workflow_call
    jobs:
      build:
        runs-on: ubuntu-latest
        steps:
          - uses: actions/checkout@v4
          - uses: actions/setup-java@v4
          - run: mvn clean install
    ```
*   **Usage:**
    ```yaml
    jobs:
      build-app:
        uses: ./.github/workflows/java-build.yaml
    ```

#### Example 3: Setup Environment
**Goal:** Configure AWS credentials, setup Node.js, and install global tools.
**Jenkins:** `setupEnv()` called inside a stage.

*   **Decision:** **Custom Action (Composite)**. It's a bundle of steps used *within* a job.
*   **Implementation:** `.github/actions/setup-env/action.yml`
    ```yaml
    runs:
      using: "composite"
      steps:
        - uses: aws-actions/configure-aws-credentials@v4
        - uses: actions/setup-node@v4
    ```

#### Example 4: Multi-Stage Deployment
**Goal:** Deploy to Dev, wait for approval, deploy to Prod.
**Jenkins:** `deployPipeline(service: 'foo')`

*   **Decision:** **Reusable Workflow**. It orchestrates multiple jobs and their dependencies.
*   **Implementation:** `.github/workflows/deploy-pipeline.yaml`
    ```yaml
    on: workflow_call
    jobs:
      deploy-dev: ...
      deploy-prod:
        needs: deploy-dev
        environment: production
        ...
    ```

---

## 7. Comprehensive Feature Mapping (Reference)

A quick reference for mapping common Jenkins features to GitHub Actions.

| Feature | Jenkins Syntax | GitHub Actions Syntax | Notes |
| :--- | :--- | :--- | :--- |
| **Parallel Execution** | `parallel { stage('A'){...} stage('B'){...} }` | **Default behavior**. Jobs run in parallel unless `needs:` is specified. | Use `matrix` for running the same job with different configs. |
| **Post Actions** | `post { always { ... } }` | `if: always()` on a step. | GHA doesn't have a "post-job" block. You add steps with conditional `if`. |
| **Timeouts** | `options { timeout(time: 1, unit: 'HOURS') }` | `timeout-minutes: 60` | Can be set at Job or Step level. |
| **Locking** | `lock('my-resource')` | `concurrency: group: my-resource` | `concurrency` can also cancel in-progress runs (`cancel-in-progress: true`). |
| **Conditionals** | `when { branch 'main' }` | `if: github.ref == 'refs/heads/main'` | `if` can be used at Job or Step level. |
| **Parameters** | `parameters { string(name: 'FOO') }` | `on: workflow_dispatch: inputs: FOO:` | Accessed via `${{ inputs.FOO }}`. |
| **Tools** | `tools { nodejs 'node-18' }` | `uses: actions/setup-node@v4` | GHA uses "Setup Actions" to install tools on the fly. |
| **Environment** | `environment { FOO = 'bar' }` | `env: FOO: bar` | Can be set at Workflow, Job, or Step level. |
| **Input Request** | `input message: 'Proceed?'` | **Environments** (Protection Rules) | Use GitHub Environments to enforce manual approval before a job runs. |

### Detailed Examples

#### 1. Post Actions (Cleanup/Notification)
**Jenkins:**
```groovy
post {
    always { echo 'Done' }
    failure { notifySlack() }
}
```
**GitHub Actions:**
```yaml
steps:
  - run: ./build.sh
  - run: echo "Done"
    if: always() # Runs even if build fails
  - uses: ./.github/actions/slack-notify
    if: failure() # Runs only if previous steps failed
```

#### 2. Concurrency (Preventing Overlap)
**Jenkins:** `disableConcurrentBuilds()`
**GitHub Actions:**
```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true # Optional: kill old run if new push comes
```

#### 3. Matrix Builds (Replacing Loops)
**Jenkins:**
```groovy
def axes = ['linux', 'windows']
axes.each { os -> stage(os) { node(os) { ... } } }
```
**GitHub Actions:**
```yaml
jobs:
  test:
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest]
    runs-on: ${{ matrix.os }}
    steps: ...
```

---

## 8. The "Missing" Pieces (Common Patterns)

These are frequent Jenkins patterns that don't have a direct 1:1 keyword but are easy to solve.

### 1. Docker Agents (`agent { docker ... }`)
**Jenkins:**
```groovy
agent {
    docker { image 'node:18' }
}
```
**GitHub Actions:** Use `container`.
```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    container: node:18
    steps:
      - run: node -v
```

### 2. Working Directory (`dir('subdir')`)
**Jenkins:**
```groovy
dir('backend') {
    sh 'mvn install'
}
```
**GitHub Actions:** Use `working-directory`.
```yaml
steps:
  - run: mvn install
    working-directory: ./backend
```
*Note: You can also set `defaults.run.working-directory` at the Job level.*

### 3. Retries (`retry(3)`)
**Jenkins:** `retry(3) { ... }`
**GitHub Actions:**
*   **Built-in:** None for steps (only for network requests in some actions).
*   **Solution:** Use a third-party action or a shell loop.
    ```yaml
    uses: nick-fields/retry@v3
    with:
      timeout_minutes: 10
      max_attempts: 3
      command: npm test
    ```

### 4. Triggering Other Pipelines (`build job: 'foo'`)
**Jenkins:** `build job: 'deploy-service'`
**GitHub Actions:**
*   **Option A (Downstream):** Use `on: workflow_run` in the *target* workflow.
    ```yaml
    # In 'deploy-service' workflow
    on:
      workflow_run:
        workflows: ["CI Pipeline"]
        types: [completed]
    ```
*   **Option B (Manual Trigger):** Use `repository_dispatch` or `workflow_dispatch` via API.
    ```yaml
    - run: gh workflow run deploy.yml -f env=prod
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    ```

### 5. Scripted Pipeline (Complex Groovy)
**Jenkins:**
```groovy
node {
    def files = findFiles(glob: '**.xml')
    files.each { f -> ... complex logic ... }
}
```
**GitHub Actions:** **Do not try to write complex logic in YAML.**
*   **Strategy:** Move logic to a Python or Bash script.
*   **Step:**
    ```yaml
    - run: python scripts/process_files.py
    ```

---

## 9. Advanced Infrastructure: ARC (Actions Runner Controller)

For enterprise-scale migrations, managing static VMs for self-hosted runners becomes a bottleneck. **ARC** is the Kubernetes-native solution.

### What is ARC?
ARC is a Kubernetes operator that orchestrates self-hosted runners on your K8s cluster. It automatically scales runners up and down based on the number of pending workflow jobs.

### When to use ARC?
| Feature | Static VM Runners | ARC (Kubernetes) |
| :--- | :--- | :--- |
| **Scaling** | Manual (Add/Remove VMs) | **Auto-scaling** (0 to N) |
| **Environment** | Persistent (Risk of dirty workspace) | **Ephemeral** (Fresh pod per job) |
| **Cost** | High (Idle VMs pay 24/7) | **Low** (Scale to 0 when idle) |
| **Maintenance** | High (Patching OS, tools) | **Low** (Update container image) |

**Use ARC if:**
*   You have high-volume CI/CD loads.
*   You need strict isolation (ephemeral containers).
*   You are already running workloads on Kubernetes.

### Example Configuration (ScaleSet)
Modern ARC uses "Runner Scale Sets".

**1. Helm Chart Installation:**
```bash
helm install arc \
    oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set-controller \
    --namespace arc-systems
```

**2. Define a Scale Set (`runner-set.yaml`):**
```yaml
apiVersion: actions.github.com/v1alpha1
kind: AutoscalingRunnerSet
metadata:
  name: k8s-runners
spec:
  githubConfigUrl: "https://github.com/my-org"
  githubConfigSecret: "pre-defined-secret"
  minRunners: 0
  maxRunners: 50
  template:
    spec:
      containers:
        - name: runner
          image: ghcr.io/actions/actions-runner:latest
          command: ["/home/runner/run.sh"]
```

**3. Usage in Workflow:**
```yaml
jobs:
  build:
    runs-on: k8s-runners # Matches the 'name' in AutoscalingRunnerSet
    steps:
      - run: echo "Running on K8s!"
```

---

## 10. GHA Native "Superpowers" (New Capabilities)

GitHub Actions offers modern features that Jenkins lacks or requires complex plugins for.

### 1. Environments (Manual Approvals & Gates)
In Jenkins, you use `input`. In GHA, you use **Environments**.
1.  Go to Repo Settings -> Environments -> Create "production".
2.  Add "Required Reviewers".
3.  **Workflow:**
    ```yaml
    jobs:
      deploy:
        environment: production # Pauses here for approval
        runs-on: ubuntu-latest
        steps: ...
    ```

### 2. OIDC (Keyless Authentication)
Stop managing long-lived AWS/Azure keys in Jenkins Credentials. Use OpenID Connect (OIDC).
**AWS Example:**
```yaml
permissions:
  id-token: write # Required for OIDC
  contents: read

steps:
  - uses: aws-actions/configure-aws-credentials@v4
    with:
      role-to-assume: arn:aws:iam::123456789012:role/my-github-role
      aws-region: us-east-1
```
*No secrets needed! GitHub signs a token, AWS verifies it.*

### 3. Caching (`actions/cache`)
Speed up builds by caching dependencies (node_modules, .m2, etc.).
```yaml
steps:
  - uses: actions/cache@v4
    with:
      path: ~/.npm
      key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
      restore-keys: |
        ${{ runner.os }}-node-
  - run: npm ci
```

### 4. Debug Mode
Troubleshoot easily without adding `echo` statements.
*   Set Secret `ACTIONS_STEP_DEBUG` = `true`.
*   Re-run the job. You will see verbose debug logs for every step.

---

## 11. Migration Strategy & Best Practices

1.  **Start with "Leaf" Jobs**: Migrate simple CI jobs (lint, unit test) first.
2.  **Don't "Lift and Shift"**: Don't try to replicate Jenkins logic 1:1. Jenkins often has complex logic because it *lacks* features GHA has native support for (like matrix builds).
3.  **Matrix Builds**: Replace `for` loops in Jenkins with Matrix strategies.
    ```yaml
    strategy:
      matrix:
        node: [14, 16, 18]
    ```
4.  **Security**:
    *   Use `permissions:` block to limit GITHUB_TOKEN scope.
    *   Pin actions to a commit SHA for immutable security: `uses: actions/checkout@a5ac7...`
5.  **Self-Hosted Runners**: If you need access to private VPC resources, install the GHA Runner agent on your existing Jenkins nodes (or new VMs) and register them with GitHub.

---

## Summary Checklist for Migration
- [ ] **Audit**: List all Jenkins plugins used. Find GHA equivalents.
- [ ] **Secrets**: Migrate credentials to GitHub Secrets.
- [ ] **Convert**: Rewrite Jenkinsfiles to YAML.
- [ ] **Optimize**: Refactor loops to Matrix, shared steps to Composite Actions.
- [ ] **Secure**: Apply least-privilege permissions.
