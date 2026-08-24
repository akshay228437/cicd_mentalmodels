# GitHub Actions Generic YAML Structure

## Full Generic Structure

```text
.github/
└── workflows/
    ├── ci.yml
    ├── cd.yml
    ├── release.yml
    └── reusable-deploy.yml
```

## Workflow Structure

```text
Workflow
│
├── name
├── on
│   ├── push
│   ├── pull_request
│   ├── workflow_dispatch
│   ├── schedule
│   ├── workflow_call
│   └── workflow_run
│
├── permissions
├── env
├── concurrency
│
└── jobs
    │
    ├── lint
    ├── test
    ├── build
    │   ├── matrix
    │   ├── cache
    │   └── artifacts
    │
    ├── security
    ├── integration-test
    │   └── services
    │
    ├── deploy-dev
    ├── deploy-staging
    │   └── environment
    │
    ├── deploy-production
    │   ├── environment
    │   ├── approval
    │   └── secrets
    │
    └── notify
```

## Full YAML Example

```yaml
# .github/workflows/generic.yml

name: Generic CI/CD Workflow

# ============================================================
# 1. WORKFLOW TRIGGERS
# ============================================================
on:

  # Run on pushes to selected branches
  push:
    branches:
      - main
      - develop
      - "release/**"
    paths:
      - "**.js"
      - "**.ts"
      - "src/**"

  # Run on pull requests
  pull_request:
    branches:
      - main
      - develop
    types:
      - opened
      - synchronize
      - reopened

  # Manual execution
  workflow_dispatch:
    inputs:
      environment:
        description: "Deployment environment"
        required: true
        type: choice
        options:
          - development
          - staging
          - production
        default: staging

      debug:
        description: "Enable debug mode"
        required: false
        type: boolean
        default: false

      version:
        description: "Version to deploy"
        required: false
        type: string

  # Scheduled execution
  schedule:
    - cron: "0 2 * * *"

  # Run after another workflow completes
  # workflow_run:
  #   workflows: ["Build"]
  #   types:
  #     - completed

  # Run when a release is created
  # release:
  #   types:
  #     - published


# ============================================================
# 2. GLOBAL PERMISSIONS
# ============================================================
permissions:
  contents: read
  actions: read
  checks: read
  pull-requests: read
  # packages: write
  # deployments: write
  # id-token: write


# ============================================================
# 3. GLOBAL ENVIRONMENT VARIABLES
# ============================================================
env:
  APP_NAME: "my-application"
  NODE_ENV: "test"
  CI: true

  # Non-sensitive configuration
  LOG_LEVEL: "info"

  # Never hard-code secrets here.
  # Use secrets instead:
  # API_KEY: ${{ secrets.API_KEY }}


# ============================================================
# 4. CONCURRENCY
# ============================================================
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true


# ============================================================
# 5. JOBS
# ============================================================
jobs:

  # ==========================================================
  # JOB 1: LINT
  # ==========================================================
  lint:

    name: Lint
    runs-on: ubuntu-latest

    permissions:
      contents: read

    # environment: development

    env:
      NODE_ENV: development

    timeout-minutes: 10

    # if: github.event_name != 'pull_request'

    steps:

      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: "22"
          cache: npm

      - name: Install dependencies
        run: npm ci

      - name: Run lint
        run: npm run lint


  # ==========================================================
  # JOB 2: TEST
  # ==========================================================
  test:

    name: Test
    runs-on: ubuntu-latest

    needs:
      - lint

    steps:

      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: "22"
          cache: npm

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm test

      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-results
          path: |
            test-results/
            coverage/
          retention-days: 7


  # ==========================================================
  # JOB 3: BUILD
  # ==========================================================
  build:

    name: Build
    runs-on: ubuntu-latest

    needs:
      - lint
      - test

    outputs:
      version: ${{ steps.version.outputs.version }}

    steps:

      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: "22"
          cache: npm

      - name: Install dependencies
        run: npm ci

      - name: Generate version
        id: version
        run: |
          VERSION="${GITHUB_SHA::7}"
          echo "version=$VERSION" >> "$GITHUB_OUTPUT"

      - name: Build application
        run: npm run build

      - name: Upload build artifact
        uses: actions/upload-artifact@v4
        with:
          name: application-build
          path: |
            dist/
            build/
          retention-days: 30


  # ==========================================================
  # JOB 4: MATRIX BUILD
  # ==========================================================
  matrix-test:

    name: Test on Node ${{ matrix.node-version }}
    runs-on: ubuntu-latest

    strategy:

      fail-fast: false

      matrix:
        node-version:
          - "20"
          - "22"

        # os:
        #   - ubuntu-latest
        #   - windows-latest
        #   - macos-latest

        # include:
        #   - node-version: "22"
        #     experimental: true

        # exclude:
        #   - node-version: "20"
        #     os: windows-latest

    steps:

      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}

      - name: Install
        run: npm ci

      - name: Test
        run: npm test


  # ==========================================================
  # JOB 5: DATABASE / SERVICE CONTAINER EXAMPLE
  # ==========================================================
  integration-test:

    name: Integration Tests
    runs-on: ubuntu-latest

    needs: build

    services:

      postgres:
        image: postgres:16
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: testdb

        ports:
          - 5432:5432

        options: >-
          --health-cmd "pg_isready -U test -d testdb"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

      redis:
        image: redis:7
        ports:
          - 6379:6379

    env:
      DATABASE_URL: postgres://test:test@localhost:5432/testdb
      REDIS_URL: redis://localhost:6379

    steps:

      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: "22"
          cache: npm

      - name: Install
        run: npm ci

      - name: Run integration tests
        run: npm run test:integration


  # ==========================================================
  # JOB 6: SECURITY / DEPENDENCY CHECK
  # ==========================================================
  security:

    name: Security Scan
    runs-on: ubuntu-latest

    needs: build

    permissions:
      contents: read
      security-events: write

    steps:

      - name: Checkout
        uses: actions/checkout@v4

      - name: Dependency audit
        run: npm audit --audit-level=high

      # Example third-party security scanner:
      # - name: Run security scanner
      #   uses: <security-action>
      #   with:
      #     ...


  # ==========================================================
  # JOB 7: DEPLOY TO DEVELOPMENT
  # ==========================================================
  deploy-development:

    name: Deploy Development
    runs-on: ubuntu-latest

    needs:
      - build
      - integration-test

    environment:
      name: development
      # url: https://dev.example.com

    if: github.ref == 'refs/heads/develop'

    steps:

      - name: Download build
        uses: actions/download-artifact@v4
        with:
          name: application-build
          path: dist/

      - name: Deploy
        run: |
          echo "Deploying to development..."
          # ./scripts/deploy.sh development


  # ==========================================================
  # JOB 8: DEPLOY TO STAGING
  # ==========================================================
  deploy-staging:

    name: Deploy Staging
    runs-on: ubuntu-latest

    needs:
      - build
      - integration-test
      - security

    environment:
      name: staging
      # url: https://staging.example.com

    if: |
      github.ref == 'refs/heads/main' ||
      github.event_name == 'workflow_dispatch'

    steps:

      - name: Download build
        uses: actions/download-artifact@v4
        with:
          name: application-build
          path: dist/

      - name: Deploy to staging
        env:
          DEPLOY_TOKEN: ${{ secrets.DEPLOY_TOKEN }}
        run: |
          echo "Deploying to staging..."
          # ./scripts/deploy.sh staging


  # ==========================================================
  # JOB 9: PRODUCTION DEPLOYMENT
  # ==========================================================
  deploy-production:

    name: Deploy Production
    runs-on: ubuntu-latest

    needs:
      - deploy-staging

    environment:
      name: production
      # url: https://example.com

    if: |
      github.ref == 'refs/heads/main' &&
      github.event_name == 'push'

    permissions:
      contents: read
      deployments: write
      id-token: write

    steps:

      - name: Download build
        uses: actions/download-artifact@v4
        with:
          name: application-build
          path: dist/

      - name: Deploy production
        env:
          DEPLOY_TOKEN: ${{ secrets.PRODUCTION_DEPLOY_TOKEN }}
        run: |
          echo "Deploying to production..."
          # ./scripts/deploy.sh production


  # ==========================================================
  # JOB 10: NOTIFICATION
  # ==========================================================
  notify:

    name: Notify
    runs-on: ubuntu-latest

    if: always()

    needs:
      - lint
      - test
      - build
      - deploy-development
      - deploy-staging
      - deploy-production

    steps:

      - name: Workflow summary
        run: |
          echo "Workflow: ${{ github.workflow }}"
          echo "Repository: ${{ github.repository }}"
          echo "Branch: ${{ github.ref }}"
          echo "Commit: ${{ github.sha }}"
          echo "Status: ${{ job.status }}"

      # Example Slack/Teams notification
      # - name: Send notification
      #   uses: <notification-action>
      #   with:
      #     webhook-url: ${{ secrets.SLACK_WEBHOOK_URL }}

```

## Common GitHub Actions Features / Options

### Job Condition Examples

```yaml
if: github.ref == 'refs/heads/main'

if: github.event_name == 'pull_request'

if: startsWith(github.ref, 'refs/tags/')

if: success()

if: failure()

if: always()

if: cancelled()
```

### Step Condition Examples

```yaml
- name: Production only
  if: github.ref == 'refs/heads/main'
  run: echo "Production"
```

## Common Contexts

### GitHub Context

```yaml
${{ github.repository }}
${{ github.ref }}
${{ github.sha }}
${{ github.actor }}
${{ github.event_name }}
${{ github.workspace }}
${{ github.run_id }}
${{ github.run_number }}
```

### Environment

```yaml
${{ env.APP_NAME }}
```

### Secrets

```yaml
${{ secrets.API_KEY }}
${{ secrets.DEPLOY_TOKEN }}
```

### Variables

```yaml
${{ vars.ENVIRONMENT }}
```

### Job Outputs

```yaml
${{ needs.build.outputs.version }}
```

### Matrix

```yaml
${{ matrix.node-version }}
```

### Steps

```yaml
${{ steps.version.outputs.version }}
```

## Reusable Workflow Example

A reusable workflow can use:

```yaml
on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string

    secrets:
      deploy_token:
        required: true
```

Then call it using:

```yaml
jobs:
  deploy:
    uses: ./.github/workflows/deploy.yml
    with:
      environment: production
    secrets:
      deploy_token: ${{ secrets.DEPLOY_TOKEN }}
```

## Reusable Workflow / External Workflow Call

```yaml
jobs:
  deploy:
    uses: organization/repository/.github/workflows/deploy.yml@main
    with:
      environment: production
    secrets: inherit
```

## Composite / Custom Action Example

```yaml
steps:
  - name: Run custom action
    uses: ./.github/actions/my-action
    with:
      environment: production
```

## Docker Example

```yaml
- name: Build Docker image
  run: |
    docker build \
      --tag my-app:${{ github.sha }} \
      .

- name: Login to registry
  uses: docker/login-action@v3
  with:
    registry: ghcr.io
    username: ${{ github.actor }}
    password: ${{ secrets.GITHUB_TOKEN }}

- name: Push image
  run: |
    docker push ghcr.io/${{ github.repository }}:${{ github.sha }}
```

## Artifact Examples

### Upload

```yaml
- uses: actions/upload-artifact@v4
  with:
    name: artifact-name
    path: ./dist
```

### Download

```yaml
- uses: actions/download-artifact@v4
  with:
    name: artifact-name
    path: ./dist
```

## Cache Example

```yaml
- uses: actions/cache@v4
  with:
    path: ~/.cache/my-tool
    key: ${{ runner.os }}-my-tool-${{ hashFiles('**/lockfile') }}
    restore-keys: |
      ${{ runner.os }}-my-tool-
```

## Shell Options

```yaml
defaults:
  run:
    shell: bash
    working-directory: ./app
```

## Default Working Directory

```yaml
defaults:
  run:
    working-directory: ./src
```

## Self-Hosted Runner Example

```yaml
runs-on:
  - self-hosted
  - linux
  - x64
```

## Multiple Runner Options

```yaml
runs-on: ${{ matrix.os }}

strategy:
  matrix:
    os:
      - ubuntu-latest
      - windows-latest
      - macos-latest
```

## Environment / Approval

```yaml
environment:
  name: production
  url: https://example.com
```

Configure required reviewers, deployment branches, environment secrets, and environment variables in:

```text
Repository
  -> Settings
  -> Environments
```

## Tag / Release Deployment

```yaml
if: startsWith(github.ref, 'refs/tags/v')
```

## Branch Condition

```yaml
if: github.ref == 'refs/heads/main'
```

## File / Path Condition

```yaml
on:
  push:
    paths:
      - "src/**"
      - "package.json"
      - "package-lock.json"
```

## Manual Input Example

```yaml
github.event.inputs.environment
```

Or, with typed `workflow_dispatch` inputs:

```yaml
inputs.environment
inputs.debug
inputs.version
```

## Important Security Notes

1. Never commit passwords, API keys, tokens, or private keys.

2. Use:

```yaml
${{ secrets.SECRET_NAME }}
```

3. Give `GITHUB_TOKEN` only the permissions required.

4. Pin third-party actions to trusted versions/SHAs where appropriate.

5. Use GitHub Environments for production deployments.

6. Use environment secrets for environment-specific secrets.

7. Be careful when running untrusted pull-request code with privileged secrets.

8. Prefer OIDC (`id-token: write`) for cloud authentication instead of long-lived cloud credentials.
