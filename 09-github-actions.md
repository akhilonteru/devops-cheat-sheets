# GitHub Actions Cheat Sheet

> CI/CD workflow syntax and common patterns — triggers, jobs, contexts, actions, secrets, matrices, artifacts, and reusable workflows.

---

## Table of Contents

- [Workflow File Anatomy](#1-workflow-file-anatomy)
- [Triggers (`on`)](#2-triggers-on)
- [Jobs, Runners & Steps](#3-jobs-runners--steps)
- [Contexts & Expressions](#4-contexts--expressions)
- [Secrets, Variables & Permissions](#5-secrets-variables--permissions)
- [Matrix Builds](#6-matrix-builds)
- [Services & Containers](#7-services--containers)
- [Artifacts & Caching](#8-artifacts--caching)
- [Reusable Workflows](#9-reusable-workflows)
- [Complete Example Workflows](#10-complete-example-workflows)
- [Useful Built-in Commands](#11-useful-built-in-commands)

---

## 1. Workflow File Anatomy

File: `.github/workflows/ci.yml`

```yaml
name: CI

on:
  push:
    branches: [main, develop]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
      - run: npm ci
      - run: npm test
```

## 2. Triggers (`on`)

```yaml
on:
  push:
    branches: [main]
    tags: ['v*.*.*']
    paths: ['src/**', 'package.json']          # Only when these change
    paths-ignore: ['docs/**']
  pull_request:
    branches: [main]
    types: [opened, synchronize, reopened, ready_for_review]
  schedule:
    - cron: '0 2 * * *'                       # Daily 2 AM UTC
  workflow_dispatch:                           # Manual trigger w/ inputs
    inputs:
      environment:
        type: choice
        options: [dev, staging, prod]
        default: dev
      dry_run:
        type: boolean
        default: true
  workflow_call:                               # Reusable by other workflows
    inputs:
      version: { required: true, type: string }
    secrets:
      token: { required: true }
  release:
    types: [published]
  repository_dispatch:                         # External API-triggered
    types: [deploy-webhook]
```

## 3. Jobs, Runners & Steps

```yaml
jobs:
  test:
    runs-on: ubuntu-latest                    # ubuntu-latest | windows-latest | macos-latest | [self-hosted, linux, gpu]
    name: Unit tests
    timeout-minutes: 30
    needs: [lint]                             # Run after these jobs
    if: github.ref == 'refs/heads/main'
    env:
      CI: true
    outputs:
      image_tag: ${{ steps.meta.outputs.tag }}
    concurrency:
      group: deploy-${{ github.ref }}
      cancel-in-progress: true
    continue-on-error: true

    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0                      # Full history (for versioning/sonar)
          ref: ${{ github.head_ref }}

      - name: Setup
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
          cache-dependency-path: package-lock.json

      - name: Run tests
        run: |
          npm ci
          npm run test:coverage
        env:
          NODE_ENV: test
        working-directory: ./app
        shell: bash
        continue-on-error: false
```

**Step outputs to job outputs:**
```yaml
    steps:
      - id: meta
        run: echo "tag=v${{ github.run_number }}" >> "$GITHUB_OUTPUT"
    outputs:
      tag: ${{ steps.meta.outputs.tag }}
```

## 4. Contexts & Expressions

| Context | Contains |
| --- | --- |
| `github` | `ref`, `sha`, `actor`, `event_name`, `repository`, `event` payload |
| `env` | Workflow/job/step env vars |
| `vars` | Repository/organization/environment variables |
| `secrets` | Encrypted secrets |
| `jobs` | Outputs of other jobs |
| `steps` | Outputs, outcome of steps (`success()`, `failure()`) |
| `matrix` | Matrix values |
| `inputs` | Workflow dispatch / reusable workflow inputs |
| `runner` | `os`, `arch`, `temp`, `tool_cache` |

```yaml
if: github.event_name == 'push' && github.ref == 'refs/heads/main'
if: startsWith(github.ref, 'refs/tags/v')
if: contains(github.event.head_commit.message, '[skip ci]') == false
if: failure() && steps.tests.outcome == 'failure'
run: echo "Deploying ${{ inputs.environment }} by ${{ github.actor }}"
run: echo "${{ toJSON(github) }}"          # Debug a context
```

## 5. Secrets, Variables & Permissions

```yaml
on: push
permissions:
  contents: read               # Minimal default — grant only what's needed
  packages: write
  id-token: write              # For OIDC cloud auth (no stored keys!)
  pull-requests: write

env:
  REGISTRY: ghcr.io

jobs:
  deploy:
    steps:
      - run: echo "${{ secrets.DATABASE_PASSWORD }}" | base64
      - run: echo "${{ vars.APP_DOMAIN }}"

      # Environments add protection rules + separate secrets
      - uses: actions/checkout@v4
      - name: Deploy to environment
        run: ./deploy.sh
        environment: production
        env:
          SSH_KEY: ${{ secrets.DEPLOY_SSH_KEY }}
```

**OIDC → cloud (no long-lived keys):**
```yaml
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789:role/gha-deploy
          aws-region: us-east-1
```

## 6. Matrix Builds

```yaml
jobs:
  test:
    strategy:
      fail-fast: false
      max-parallel: 4
      matrix:
        os: [ubuntu-latest, windows-latest]
        node: [18, 20]
        include:
          - os: macos-latest
            node: 20
        exclude:
          - os: windows-latest
            node: 18
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
      - run: npm ci && npm test
```

## 7. Services & Containers

```yaml
jobs:
  integration-test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: testpass
        ports: ['5432:5432']
        options: >-
          --health-cmd pg_isready
          --health-interval 5s
          --health-timeout 5s
          --health-retries 5
      redis:
        image: redis:7-alpine
        ports: ['6379:6379']
    container: node:20-alpine             # All steps run inside this container
    steps:
      - uses: actions/checkout@v4
      - run: npm run test:integration
```

## 8. Artifacts & Caching

```yaml
      - name: Cache dependencies
        uses: actions/cache@v4
        with:
          path: ~/.npm
          key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}
          restore-keys: |
            ${{ runner.os }}-npm-

      - name: Cache Docker layers
        uses: actions/cache@v4
        with:
          path: /tmp/.buildx-cache
          key: ${{ runner.os }}-buildx-${{ github.sha }}
          restore-keys: ${{ runner.os }}-buildx-

      - uses: actions/upload-artifact@v4
        with:
          name: dist-${{ matrix.node }}
          path: dist/
          retention-days: 7
          if-no-files-found: error

      - uses: actions/download-artifact@v4
        with:
          name: dist-20
          path: ./dist
```

**Pre-configured caches:** `actions/setup-node@v4` with `cache: npm`, `setup-python@v5` with `cache: pip`, `setup-java@v4` with `cache: maven` / `cache: gradle`.

## 9. Reusable Workflows

```yaml
# .github/workflows/build-reusable.yml
on:
  workflow_call:
    inputs:
      image_name: { required: true, type: string }
      push: { type: boolean, default: false }
    secrets:
      registry_token: { required: true }
    outputs:
      image_digest:
        value: ${{ jobs.docker.outputs.digest }}

jobs:
  docker:
    runs-on: ubuntu-latest
    outputs:
      digest: ${{ steps.build.outputs.digest }}
    steps:
      - uses: actions/checkout@v4
      - uses: docker/build-push-action@v6
        id: build
        with:
          context: .
          push: ${{ inputs.push }}
          tags: ${{ inputs.image_name }}:latest
```

```yaml
# .github/workflows/ci.yml
jobs:
  docker:
    uses: ./.github/workflows/build-reusable.yml
    with:
      image_name: ghcr.io/org/app
      push: ${{ github.ref == 'refs/heads/main' }}
    secrets:
      registry_token: ${{ secrets.GITHUB_TOKEN }}
```

## 10. Complete Example Workflows

**Docker build & push to GHCR:**
```yaml
name: Release
on:
  push:
    tags: ['v*']
permissions:
  contents: read
  packages: write
jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/metadata-action@v5
        id: meta
        with:
          images: ghcr.io/${{ github.repository }}
          tags: |
            type=semver,pattern={{version}}
            type=sha
      - uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

**Kubernetes deploy:**
```yaml
      - uses: azure/k8s-set-context@v4           # or aws-actions/amazon-eks... 
        with:
          kubeconfig: ${{ secrets.KUBECONFIG }}
      - run: |
          kubectl set image deploy/app app=ghcr.io/org/app:${{ github.sha }} -n prod
          kubectl rollout status deploy/app -n prod --timeout=300s
```

## 11. Useful Built-in Commands

```yaml
      - run: echo "RESULT=success" >> "$GITHUB_ENV"      # Set env for later steps
      - run: echo "TAG=v1.0.0" >> "$GITHUB_OUTPUT"       # Set step output
      - run: echo "my-key=value" >> "$GITHUB_STEP_SUMMARY"  # Add to job summary
      - run: echo "$HOME/.local/bin" >> "$GITHUB_PATH"   # Add to PATH
      - run: |
          {
            echo '```'
            cat test-output.txt
            echo '```'
          } >> "$GITHUB_STEP_SUMMARY"
```

| Action | Purpose |
| --- | --- |
| `actions/checkout@v4` | Clone repo |
| `actions/setup-node@v4` | Node.js + npm cache |
| `actions/setup-python@v5` | Python + pip cache |
| `actions/setup-java@v4` | JDK + Maven/Gradle cache |
| `actions/cache@v4` | Generic caching |
| `actions/upload-artifact@v4` / `download-artifact@v4` | Share files between jobs |
| `docker/setup-buildx-action@v3` / `login-action@v3` / `build-push-action@v6` | Docker builds |
| `docker/metadata-action@v5` | Smart image tags |
| `github/codeql-action/*` | Security scanning |
| `EnricoMi/publish-unit-test-result-action` | Test result annotations |

---
