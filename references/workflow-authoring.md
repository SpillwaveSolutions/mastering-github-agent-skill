# GitHub Actions Workflow Authoring Reference

Comprehensive reference for writing GitHub Actions workflow YAML files.

## Contents

- [Workflow Structure](#workflow-structure)
- [Event Triggers](#event-triggers)
- [Caching Strategies](#caching-strategies)
- [Matrix Builds](#matrix-builds)
- [OIDC Integration](#oidc-integration)
- [Reusable Workflows](#reusable-workflows)
- [Composite Actions](#composite-actions)
- [Security Best Practices](#security-best-practices)
- [Common Patterns](#common-patterns)
- [Expression Reference](#expression-reference)
- [Production Templates](#production-templates)
- [Advanced Patterns](#advanced-patterns)
- [Workflow Checklist](#workflow-checklist)

---

## Workflow Structure

### Basic template

```yaml
name: CI Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest
    timeout-minutes: 15

    steps:
      - uses: actions/checkout@v4
      - name: Build
        run: make build
      - name: Test
        run: make test
```

### Job dependencies

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: make build

  test:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: make test

  deploy:
    needs: [build, test]
    runs-on: ubuntu-latest
    steps:
      - run: make deploy
```

### Concurrency control

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

---

## Event Triggers

### Common triggers

```yaml
on:
  # Push to branches
  push:
    branches: [main, develop]
    paths:
      - 'src/**'
      - 'tests/**'
    paths-ignore:
      - 'docs/**'

  # Pull requests
  pull_request:
    types: [opened, synchronize, reopened]

  # Manual trigger
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment to deploy'
        required: true
        type: choice
        options:
          - staging
          - production

  # Scheduled
  schedule:
    - cron: '0 2 * * *'  # 2 AM daily

  # Reusable workflow
  workflow_call:
    inputs:
      config:
        required: true
        type: string
```

---

## Caching Strategies

### Python/Poetry

```yaml
steps:
  - uses: actions/setup-python@v5
    with:
      python-version: '3.11'
    id: setup_python

  - uses: actions/cache@v4
    with:
      path: |
        ~/.cache/pypoetry
        .venv
      key: poetry-${{ runner.os }}-${{ steps.setup_python.outputs.python-version }}-${{ hashFiles('poetry.lock') }}
      restore-keys: |
        poetry-${{ runner.os }}-${{ steps.setup_python.outputs.python-version }}-

  - uses: snok/install-poetry@v1
    with:
      virtualenvs-in-project: true

  - run: poetry install --no-interaction
```

### Node.js (npm/pnpm/yarn)

```yaml
# npm (built-in caching)
- uses: actions/setup-node@v4
  with:
    node-version: '20'
    cache: 'npm'
- run: npm ci

# pnpm
- uses: pnpm/action-setup@v2
  with:
    version: 8
- uses: actions/setup-node@v4
  with:
    node-version: '20'
    cache: 'pnpm'
- run: pnpm install --frozen-lockfile

# yarn
- uses: actions/setup-node@v4
  with:
    node-version: '20'
    cache: 'yarn'
- run: yarn install --frozen-lockfile
```

### Java (Maven/Gradle)

```yaml
# Maven
- uses: actions/setup-java@v4
  with:
    java-version: '17'
    distribution: 'temurin'
    cache: 'maven'
- run: mvn clean install

# Gradle
- uses: actions/setup-java@v4
  with:
    java-version: '17'
    distribution: 'temurin'
    cache: 'gradle'
- run: ./gradlew build
```

### Docker layer caching

```yaml
- uses: docker/setup-buildx-action@v3

- uses: docker/build-push-action@v5
  with:
    context: .
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

---

## Matrix Builds

### Basic matrix

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest, macos-latest]
    python-version: ['3.9', '3.10', '3.11']

runs-on: ${{ matrix.os }}

steps:
  - uses: actions/setup-python@v5
    with:
      python-version: ${{ matrix.python-version }}
```

### Include/Exclude

```yaml
strategy:
  fail-fast: false
  matrix:
    os: [ubuntu-latest, windows-latest]
    node: [18, 20]
    include:
      - os: ubuntu-latest
        node: 20
        experimental: true
      - os: macos-latest
        node: 20
    exclude:
      - os: windows-latest
        node: 18
```

### Dynamic matrix

```yaml
jobs:
  generate-matrix:
    runs-on: ubuntu-latest
    outputs:
      matrix: ${{ steps.set-matrix.outputs.matrix }}
    steps:
      - uses: actions/checkout@v4
      - id: set-matrix
        run: |
          MATRIX=$(cat .github/test-matrix.json)
          echo "matrix=$MATRIX" >> $GITHUB_OUTPUT

  test:
    needs: generate-matrix
    strategy:
      matrix: ${{ fromJson(needs.generate-matrix.outputs.matrix) }}
    runs-on: ubuntu-latest
    steps:
      - run: echo "Testing ${{ matrix.config }}"
```

---

## OIDC Integration

### AWS

```yaml
permissions:
  id-token: write
  contents: read

steps:
  - uses: aws-actions/configure-aws-credentials@v4
    with:
      role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsRole
      aws-region: us-east-1
      role-session-name: GitHubActions

  - uses: aws-actions/amazon-ecr-login@v2

  - name: Deploy to ECS
    run: |
      aws ecs update-service --cluster my-cluster --service my-service --force-new-deployment
```

### GCP Workload Identity Federation

```yaml
permissions:
  id-token: write
  contents: read

steps:
  - uses: google-github-actions/auth@v2
    with:
      workload_identity_provider: ${{ secrets.WIF_PROVIDER }}
      service_account: ${{ secrets.WIF_SERVICE_ACCOUNT }}

  - uses: google-github-actions/setup-gcloud@v2

  - name: Deploy to Cloud Run
    run: |
      gcloud run deploy my-service \
        --image us-central1-docker.pkg.dev/project/repo/image:${{ github.sha }} \
        --region us-central1
```

---

## Reusable Workflows

### Define reusable workflow

```yaml
# .github/workflows/deploy.yml
name: Reusable Deploy

on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
      image-tag:
        required: true
        type: string
    secrets:
      deploy-token:
        required: true
    outputs:
      deployment-url:
        value: ${{ jobs.deploy.outputs.url }}

jobs:
  deploy:
    runs-on: ubuntu-latest
    outputs:
      url: ${{ steps.deploy.outputs.url }}

    steps:
      - name: Deploy
        id: deploy
        run: |
          echo "Deploying ${{ inputs.image-tag }} to ${{ inputs.environment }}"
          echo "url=https://${{ inputs.environment }}.example.com" >> $GITHUB_OUTPUT
```

### Call reusable workflow

```yaml
jobs:
  deploy-staging:
    uses: ./.github/workflows/deploy.yml
    with:
      environment: staging
      image-tag: ${{ github.sha }}
    secrets:
      deploy-token: ${{ secrets.STAGING_TOKEN }}

  deploy-prod:
    needs: deploy-staging
    uses: ./.github/workflows/deploy.yml
    with:
      environment: production
      image-tag: ${{ github.sha }}
    secrets:
      deploy-token: ${{ secrets.PROD_TOKEN }}
```

---

## Composite Actions

### Create composite action

```yaml
# .github/actions/setup-project/action.yml
name: 'Setup Project'
description: 'Setup Python, Poetry, and dependencies'

inputs:
  python-version:
    description: 'Python version'
    required: false
    default: '3.11'

outputs:
  cache-hit:
    description: 'Whether cache was hit'
    value: ${{ steps.cache.outputs.cache-hit }}

runs:
  using: 'composite'
  steps:
    - uses: actions/setup-python@v5
      with:
        python-version: ${{ inputs.python-version }}

    - uses: actions/cache@v4
      id: cache
      with:
        path: .venv
        key: venv-${{ runner.os }}-${{ inputs.python-version }}-${{ hashFiles('poetry.lock') }}

    - uses: snok/install-poetry@v1
      with:
        virtualenvs-in-project: true

    - run: poetry install
      if: steps.cache.outputs.cache-hit != 'true'
      shell: bash
```

### Use composite action

```yaml
steps:
  - uses: actions/checkout@v4
  - uses: ./.github/actions/setup-project
    with:
      python-version: '3.11'
```

---

## Security Best Practices

### Pin actions to SHA

```yaml
# GOOD: Pinned to commit SHA
- uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11  # v4.1.1

# ACCEPTABLE: Major version
- uses: actions/checkout@v4

# AVOID: Mutable tags
- uses: actions/checkout@main
```

### Minimal permissions

```yaml
permissions:
  contents: read
  pull-requests: write
  id-token: write  # Only for OIDC
```

### Secret scanning

```yaml
- uses: trufflesecurity/trufflehog@main
  with:
    path: ./
    base: main
    head: HEAD
```

---

## Common Patterns

### Path-based conditional execution

```yaml
- uses: dorny/paths-filter@v3
  id: changes
  with:
    filters: |
      backend:
        - 'backend/**'
      frontend:
        - 'frontend/**'

- name: Run backend tests
  if: steps.changes.outputs.backend == 'true'
  run: npm test --prefix backend
```

### Environment-specific deployments

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://example.com

    steps:
      - run: echo "Deploying..."
        env:
          API_KEY: ${{ secrets.PROD_API_KEY }}
```

### Artifact management

```yaml
# Upload
- uses: actions/upload-artifact@v4
  with:
    name: dist
    path: dist/
    retention-days: 7

# Download
- uses: actions/download-artifact@v4
  with:
    name: dist
    path: dist/
```

### Continue on error

```yaml
- name: Risky step
  run: ./might-fail.sh
  continue-on-error: true

- name: Always run cleanup
  if: always()
  run: ./cleanup.sh
```

---

## Expression Reference

### Conditionals

```yaml
if: github.event_name == 'push' && github.ref == 'refs/heads/main'
if: contains(github.event.head_commit.message, '[skip ci]') == false
if: success() && github.event.pull_request.draft == false
if: failure() || cancelled()
if: always()
if: startsWith(github.ref, 'refs/tags/v')
if: contains(github.event.pull_request.labels.*.name, 'deploy')
```

### Built-in functions

```yaml
# JSON
${{ fromJson(needs.job.outputs.matrix) }}
${{ toJson(github.event) }}

# Hash files
${{ hashFiles('**/package-lock.json') }}

# Format strings
${{ format('image-{0}:{1}', matrix.variant, github.sha) }}
```

### Context values

```yaml
${{ github.repository }}         # owner/repo
${{ github.repository_owner }}   # owner
${{ github.sha }}                # commit SHA
${{ github.ref }}                # refs/heads/main
${{ github.ref_name }}           # main
${{ github.actor }}              # user who triggered
${{ github.event_name }}         # push, pull_request, etc
${{ github.run_id }}             # unique run ID
```

### Outputs and secrets

```yaml
${{ needs.build.outputs.version }}
${{ steps.meta.outputs.tags }}
${{ env.NODE_ENV }}
${{ secrets.AWS_ACCESS_KEY_ID }}
${{ vars.API_URL }}
```

---

## Production Templates

### Full CI/CD Pipeline

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

permissions:
  contents: read
  pull-requests: write
  id-token: write

jobs:
  test:
    runs-on: ubuntu-latest
    timeout-minutes: 10

    strategy:
      matrix:
        node-version: [18, 20]

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'
      - run: npm ci
      - run: npm run lint
      - run: npm run test:ci

  build:
    needs: test
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/build-push-action@v5
        with:
          context: .
          push: false
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy:
    needs: build
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: production

    steps:
      - uses: actions/checkout@v4
      - run: echo "Deploy to production"
```

### Python Monorepo

```yaml
name: Python Monorepo CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  changes:
    runs-on: ubuntu-latest
    outputs:
      packages: ${{ steps.filter.outputs.changes }}
    steps:
      - uses: actions/checkout@v4
      - uses: dorny/paths-filter@v3
        id: filter
        with:
          filters: |
            api:
              - 'packages/api/**'
            worker:
              - 'packages/worker/**'

  test:
    needs: changes
    if: ${{ needs.changes.outputs.packages != '[]' }}
    runs-on: ubuntu-latest

    strategy:
      matrix:
        package: ${{ fromJson(needs.changes.outputs.packages) }}

    defaults:
      run:
        working-directory: packages/${{ matrix.package }}

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      - uses: snok/install-poetry@v1
      - run: poetry install
      - run: poetry run pytest
```

---

## Advanced Patterns

### Ephemeral PR Environments

Create isolated environments per pull request with automatic cleanup.

#### Compute Environment Identifier

```yaml
# .github/actions/compute-identifier/action.yml
name: 'Compute Identifier'
description: 'Compute namespaced environment identifier'

outputs:
  identifier:
    description: 'Environment identifier'
    value: ${{ steps.compute.outputs.identifier }}
  is-ephemeral:
    description: 'Whether this is an ephemeral environment'
    value: ${{ steps.compute.outputs.is-ephemeral }}

runs:
  using: 'composite'
  steps:
    - id: compute
      shell: bash
      run: |
        if [ "${{ github.event_name }}" = "pull_request" ]; then
          echo "identifier=pr${{ github.event.number }}" >> $GITHUB_OUTPUT
          echo "is-ephemeral=true" >> $GITHUB_OUTPUT
        elif [ "${{ github.ref_name }}" = "main" ]; then
          echo "identifier=main" >> $GITHUB_OUTPUT
          echo "is-ephemeral=false" >> $GITHUB_OUTPUT
        else
          echo "identifier=${{ github.ref_name }}" >> $GITHUB_OUTPUT
          echo "is-ephemeral=true" >> $GITHUB_OUTPUT
        fi
```

#### Use identifier in workflows

```yaml
jobs:
  setup:
    runs-on: ubuntu-latest
    outputs:
      identifier: ${{ steps.id.outputs.identifier }}
      is-ephemeral: ${{ steps.id.outputs.is-ephemeral }}
    steps:
      - uses: actions/checkout@v4
      - uses: ./.github/actions/compute-identifier
        id: id

  deploy:
    needs: setup
    runs-on: ubuntu-latest
    env:
      RESOURCE_ID: ${{ needs.setup.outputs.identifier }}
    steps:
      - run: |
          echo "Deploying to environment: $RESOURCE_ID"
          # Resources named: myapp-$RESOURCE_ID-*
```

### PR Cleanup on Close

Automatically destroy ephemeral resources when PR is closed/merged.

```yaml
name: Cleanup PR Environment

on:
  pull_request:
    types: [closed]

jobs:
  cleanup:
    runs-on: ubuntu-latest
    env:
      RESOURCE_ID: pr${{ github.event.number }}

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN_CLEANUP }}
          aws-region: us-east-1

      # Destroy in reverse order: app → schemas → foundation
      - name: Destroy application stack
        run: |
          aws cloudformation delete-stack --stack-name "app-${RESOURCE_ID}" || true
          aws cloudformation wait stack-delete-complete --stack-name "app-${RESOURCE_ID}" || true

      - name: Destroy foundation stack
        run: |
          aws cloudformation delete-stack --stack-name "foundation-${RESOURCE_ID}" || true
          aws cloudformation wait stack-delete-complete --stack-name "foundation-${RESOURCE_ID}" || true

      - name: Clean up SSM parameters
        run: |
          PARAMS=$(aws ssm get-parameters-by-path --path "/myapp/${RESOURCE_ID}" --recursive --query 'Parameters[].Name' --output text)
          if [ -n "$PARAMS" ]; then
            aws ssm delete-parameters --names $PARAMS || true
          fi
```

### Release Please Integration

Automate semantic versioning based on conventional commits.

```yaml
name: Release Please

on:
  push:
    branches: [main]

permissions:
  contents: write
  pull-requests: write

jobs:
  release:
    runs-on: ubuntu-latest
    outputs:
      release_created: ${{ steps.release.outputs.release_created }}
      tag_name: ${{ steps.release.outputs.tag_name }}
      version: ${{ steps.release.outputs.version }}

    steps:
      - uses: googleapis/release-please-action@v4
        id: release
        with:
          release-type: node  # or: python, simple, etc.

  # Deploy only when release is created
  deploy:
    needs: release
    if: ${{ needs.release.outputs.release_created }}
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: |
          echo "Deploying version ${{ needs.release.outputs.version }}"
```

#### Monorepo with manifest

```yaml
# release-please-config.json
{
  "packages": {
    "packages/api": {
      "release-type": "python"
    },
    "packages/web": {
      "release-type": "node"
    }
  },
  "linked-versions": true
}
```

### Testing Patterns

#### Python with pytest and coverage

```yaml
name: Python Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: |
          pip install poetry
          poetry install

      - name: Run tests with coverage
        run: |
          poetry run pytest -q --cov=src --cov-report=xml --cov-report=term

      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          files: ./coverage.xml
          fail_ci_if_error: false
```

#### Java/Kotlin with Gradle

```yaml
name: Gradle Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: 'temurin'
          java-version: '17'
          cache: 'gradle'

      - name: Run tests
        run: ./gradlew test check

      - name: Publish test report
        uses: mikepenz/action-junit-report@v4
        if: always()
        with:
          report_paths: '**/build/test-results/test/TEST-*.xml'
```

#### TypeScript with Jest

```yaml
name: TypeScript Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - run: npm ci
      - run: npx tsc --noEmit  # Type check
      - run: npm test -- --coverage

      - uses: codecov/codecov-action@v4
        with:
          files: ./coverage/coverage-final.json
```

### Deployment Status Checks

Wait for infrastructure to be ready before deploying.

```yaml
- name: Wait for stack ready
  run: |
    STACK_NAME="foundation-${RESOURCE_ID}"
    MAX_ATTEMPTS=30
    ATTEMPT=0

    while [ $ATTEMPT -lt $MAX_ATTEMPTS ]; do
      STATUS=$(aws cloudformation describe-stacks \
        --stack-name "$STACK_NAME" \
        --query 'Stacks[0].StackStatus' \
        --output text 2>/dev/null || echo "NOT_FOUND")

      case "$STATUS" in
        CREATE_COMPLETE|UPDATE_COMPLETE)
          echo "Stack ready: $STATUS"
          exit 0
          ;;
        *_IN_PROGRESS)
          echo "Waiting... Status: $STATUS"
          sleep 30
          ;;
        *_FAILED|ROLLBACK_*)
          echo "Stack failed: $STATUS"
          exit 1
          ;;
        NOT_FOUND)
          echo "Stack not found, waiting..."
          sleep 30
          ;;
      esac
      ATTEMPT=$((ATTEMPT + 1))
    done
    echo "Timeout waiting for stack"
    exit 1
```

### Separate Build vs Deploy Roles

Use fine-grained OIDC roles for different workflow phases.

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read

    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          # Build role: ECR push + SSM write only
          role-to-assume: ${{ secrets.AWS_ROLE_ARN_BUILD }}
          aws-region: us-east-1

      - name: Build and push image
        run: |
          docker build -t $ECR_REPO:$VERSION .
          docker push $ECR_REPO:$VERSION
          aws ssm put-parameter --name "/app/version" --value "$VERSION" --overwrite

  deploy:
    needs: build
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read

    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          # Deploy role: CloudFormation + broader permissions
          role-to-assume: ${{ secrets.AWS_ROLE_ARN_DEPLOY }}
          aws-region: us-east-1

      - name: Deploy stack
        run: npx cdk deploy --require-approval never
```

### Ordered Multi-Stack Deployment

Deploy stacks in dependency order with wait checks.

```yaml
name: Deploy All Stacks

on:
  workflow_dispatch:
  push:
    branches: [main]

jobs:
  foundation:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npx cdk deploy FoundationStack --require-approval never

  schemas:
    needs: foundation
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npx cdk deploy SchemasStack --require-approval never

  application:
    needs: schemas
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npx cdk deploy ApplicationStack --require-approval never
```

---

## Workflow Checklist

Before deploying a new workflow, verify:

```
- [ ] Uses concurrency control to prevent parallel runs
- [ ] Has explicit permissions (not default)
- [ ] Actions pinned to SHA or major version
- [ ] Secrets accessed via ${{ secrets.* }}
- [ ] Environment variables use ${{ env.* }} or ${{ vars.* }}
- [ ] Cleanup jobs tolerate failures (|| true)
- [ ] Tests run before deployment
- [ ] PR environments auto-cleanup on close
- [ ] Build and deploy use separate OIDC roles
- [ ] Stacks deploy in dependency order
```
