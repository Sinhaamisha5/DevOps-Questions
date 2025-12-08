# GitHub Actions - Complete Solutions Guide

## 1. Unit + Integration Tests on Pull Requests

Create `.github/workflows/test-on-pr.yml`:

```yaml
name: Run Tests on PR

on:
  pull_request:
    branches: [main, develop]
  push:
    branches: [main]

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run unit tests
        run: npm run test:unit
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage/coverage-final.json
          flags: unittests

  integration-tests:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:14
        env:
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpass
          POSTGRES_DB: testdb
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      
      redis:
        image: redis:7
        ports:
          - 6379:6379
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run integration tests
        env:
          DATABASE_URL: postgresql://testuser:testpass@localhost:5432/testdb
          REDIS_URL: redis://localhost:6379
        run: npm run test:integration
      
      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: integration-test-results
          path: test-results/
```

---

## 2. Matrix Builds for Multiple Versions

Create `.github/workflows/matrix-test.yml`:

```yaml
name: Matrix Build Tests

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  test-node:
    runs-on: ${{ matrix.os }}
    
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        node-version: [16, 18, 20]
        # Exclude specific combinations if needed
        exclude:
          - os: macos-latest
            node-version: 16
      fail-fast: false  # Continue running other jobs if one fails
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Setup Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v3
        with:
          node-version: ${{ matrix.node-version }}
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run tests
        run: npm test
      
      - name: Display Node version
        run: node --version

  test-python:
    runs-on: ${{ matrix.os }}
    
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        python-version: ['3.9', '3.10', '3.11', '3.12']
      fail-fast: false
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Setup Python ${{ matrix.python-version }}
        uses: actions/setup-python@v4
        with:
          python-version: ${{ matrix.python-version }}
      
      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt
          pip install pytest pytest-cov
      
      - name: Run tests
        run: pytest --cov=src tests/
      
      - name: Display Python version
        run: python --version

  # Combined matrix with includes
  test-combined:
    runs-on: ubuntu-latest
    
    strategy:
      matrix:
        include:
          - node-version: 18
            python-version: '3.11'
          - node-version: 20
            python-version: '3.12'
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: ${{ matrix.node-version }}
      
      - name: Setup Python
        uses: actions/setup-python@v4
        with:
          python-version: ${{ matrix.python-version }}
      
      - name: Run tests
        run: |
          npm ci && npm test
          pip install -r requirements.txt && pytest
```

---

## 3. Build and Push Docker Images to Docker Hub

Create `.github/workflows/docker-build-push.yml`:

```yaml
name: Build and Push Docker Image

on:
  push:
    branches: [main]
    tags:
      - 'v*'
  pull_request:
    branches: [main]

env:
  DOCKER_IMAGE: yourusername/your-app-name

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2
      
      - name: Login to Docker Hub
        if: github.event_name != 'pull_request'
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
      
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v4
        with:
          images: ${{ env.DOCKER_IMAGE }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha,prefix={{branch}}-
            type=raw,value=latest,enable={{is_default_branch}}
      
      - name: Build and push
        uses: docker/build-push-action@v4
        with:
          context: .
          file: ./Dockerfile
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=registry,ref=${{ env.DOCKER_IMAGE }}:buildcache
          cache-to: type=registry,ref=${{ env.DOCKER_IMAGE }}:buildcache,mode=max
          platforms: linux/amd64,linux/arm64
      
      - name: Image digest
        run: echo ${{ steps.build-and-push.outputs.digest }}
```

**Setup Instructions:**

1. Create Docker Hub access token at: https://hub.docker.com/settings/security
2. Add to GitHub Secrets:
   - `DOCKERHUB_USERNAME`: Your Docker Hub username
   - `DOCKERHUB_TOKEN`: Your access token

---

## 4. Environment-Specific Deployments with Conditions

Create `.github/workflows/environment-deploy.yml`:

```yaml
name: Environment-Specific Deployment

on:
  push:
    branches:
      - main
      - develop
      - staging
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment to deploy to'
        required: true
        type: choice
        options:
          - development
          - staging
          - production

jobs:
  # Determine which environment to deploy to
  determine-environment:
    runs-on: ubuntu-latest
    outputs:
      environment: ${{ steps.set-env.outputs.environment }}
    
    steps:
      - name: Determine environment
        id: set-env
        run: |
          if [[ "${{ github.event_name }}" == "workflow_dispatch" ]]; then
            echo "environment=${{ github.event.inputs.environment }}" >> $GITHUB_OUTPUT
          elif [[ "${{ github.ref }}" == "refs/heads/main" ]]; then
            echo "environment=production" >> $GITHUB_OUTPUT
          elif [[ "${{ github.ref }}" == "refs/heads/staging" ]]; then
            echo "environment=staging" >> $GITHUB_OUTPUT
          else
            echo "environment=development" >> $GITHUB_OUTPUT
          fi

  # Deploy to Development
  deploy-development:
    runs-on: ubuntu-latest
    needs: determine-environment
    if: needs.determine-environment.outputs.environment == 'development'
    environment:
      name: development
      url: https://dev.yourapp.com
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Deploy to Development
        env:
          API_URL: ${{ secrets.DEV_API_URL }}
          DATABASE_URL: ${{ secrets.DEV_DATABASE_URL }}
        run: |
          echo "Deploying to Development..."
          # Your deployment commands here
          ./scripts/deploy.sh development

  # Deploy to Staging
  deploy-staging:
    runs-on: ubuntu-latest
    needs: determine-environment
    if: needs.determine-environment.outputs.environment == 'staging'
    environment:
      name: staging
      url: https://staging.yourapp.com
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Run tests before staging
        run: npm test
      
      - name: Deploy to Staging
        env:
          API_URL: ${{ secrets.STAGING_API_URL }}
          DATABASE_URL: ${{ secrets.STAGING_DATABASE_URL }}
        run: |
          echo "Deploying to Staging..."
          ./scripts/deploy.sh staging

  # Deploy to Production
  deploy-production:
    runs-on: ubuntu-latest
    needs: determine-environment
    if: needs.determine-environment.outputs.environment == 'production'
    environment:
      name: production
      url: https://yourapp.com
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Run full test suite
        run: npm run test:all
      
      - name: Security scan
        run: npm audit --production
      
      - name: Deploy to Production
        env:
          API_URL: ${{ secrets.PROD_API_URL }}
          DATABASE_URL: ${{ secrets.PROD_DATABASE_URL }}
        run: |
          echo "Deploying to Production..."
          ./scripts/deploy.sh production
      
      - name: Run smoke tests
        run: ./scripts/smoke-tests.sh
      
      - name: Notify team
        if: always()
        uses: 8398a7/action-slack@v3
        with:
          status: ${{ job.status }}
          text: 'Production deployment ${{ job.status }}'
          webhook_url: ${{ secrets.SLACK_WEBHOOK }}

  # Conditional deployment based on file changes
  deploy-api:
    runs-on: ubuntu-latest
    if: contains(github.event.head_commit.modified, 'api/')
    steps:
      - name: Deploy API
        run: echo "API files changed, deploying API..."

  deploy-frontend:
    runs-on: ubuntu-latest
    if: contains(github.event.head_commit.modified, 'frontend/')
    steps:
      - name: Deploy Frontend
        run: echo "Frontend files changed, deploying frontend..."
```

---

## 5. Cache Dependencies

Create `.github/workflows/cache-dependencies.yml`:

```yaml
name: Cache Dependencies

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  # Node.js with npm cache
  build-node-npm:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'  # Built-in caching
      
      - name: Install dependencies
        run: npm ci
      
      - name: Build
        run: npm run build

  # Node.js with yarn cache
  build-node-yarn:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'yarn'
      
      - name: Install dependencies
        run: yarn install --frozen-lockfile
      
      - name: Build
        run: yarn build

  # Node.js with manual cache
  build-node-manual:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
      
      - name: Cache node modules
        uses: actions/cache@v3
        id: cache-node-modules
        with:
          path: node_modules
          key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
          restore-keys: |
            ${{ runner.os }}-node-
      
      - name: Install dependencies
        if: steps.cache-node-modules.outputs.cache-hit != 'true'
        run: npm ci
      
      - name: Build
        run: npm run build

  # Python with pip cache
  build-python:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Setup Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
          cache: 'pip'  # Built-in caching
      
      - name: Install dependencies
        run: pip install -r requirements.txt
      
      - name: Run tests
        run: pytest

  # Python with manual cache
  build-python-manual:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Setup Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      
      - name: Cache pip packages
        uses: actions/cache@v3
        with:
          path: ~/.cache/pip
          key: ${{ runner.os }}-pip-${{ hashFiles('**/requirements.txt') }}
          restore-keys: |
            ${{ runner.os }}-pip-
      
      - name: Install dependencies
        run: pip install -r requirements.txt

  # Docker layer caching
  build-docker:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2
      
      - name: Cache Docker layers
        uses: actions/cache@v3
        with:
          path: /tmp/.buildx-cache
          key: ${{ runner.os }}-buildx-${{ github.sha }}
          restore-keys: |
            ${{ runner.os }}-buildx-
      
      - name: Build Docker image
        uses: docker/build-push-action@v4
        with:
          context: .
          push: false
          cache-from: type=local,src=/tmp/.buildx-cache
          cache-to: type=local,dest=/tmp/.buildx-cache-new,mode=max
      
      # Prevent cache from growing indefinitely
      - name: Move cache
        run: |
          rm -rf /tmp/.buildx-cache
          mv /tmp/.buildx-cache-new /tmp/.buildx-cache

  # Multiple caches example
  build-complex:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
      
      # Cache node_modules
      - name: Cache node modules
        uses: actions/cache@v3
        with:
          path: node_modules
          key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
      
      # Cache build output
      - name: Cache build
        uses: actions/cache@v3
        with:
          path: dist
          key: ${{ runner.os }}-build-${{ github.sha }}
          restore-keys: |
            ${{ runner.os }}-build-
      
      # Cache test coverage
      - name: Cache coverage
        uses: actions/cache@v3
        with:
          path: coverage
          key: ${{ runner.os }}-coverage-${{ github.sha }}
      
      - name: Install dependencies
        run: npm ci
      
      - name: Build
        run: npm run build
      
      - name: Test
        run: npm test
```

---

## 6. Deploy Static Site to GitHub Pages

Create `.github/workflows/github-pages.yml`:

```yaml
name: Deploy to GitHub Pages

on:
  push:
    branches: [main]
  workflow_dispatch:

# Sets permissions of the GITHUB_TOKEN
permissions:
  contents: read
  pages: write
  id-token: write

# Allow one concurrent deployment
concurrency:
  group: "pages"
  cancel-in-progress: true

jobs:
  # Build job
  build:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout
        uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Build
        run: npm run build
        env:
          # Add environment variables if needed
          PUBLIC_URL: https://${{ github.repository_owner }}.github.io/${{ github.event.repository.name }}
      
      - name: Upload artifact
        uses: actions/upload-pages-artifact@v2
        with:
          path: ./dist  # or ./build for React apps

  # Deploy job
  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    
    runs-on: ubuntu-latest
    needs: build
    
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v2

# Alternative: Simple HTML/CSS/JS static site
---
name: Deploy Simple Static Site

on:
  push:
    branches: [main]

permissions:
  contents: write

jobs:
  deploy:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout
        uses: actions/checkout@v3
      
      - name: Deploy to GitHub Pages
        uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./public  # Your static files directory
          publish_branch: gh-pages
          cname: yourdomain.com  # Optional: custom domain

# Alternative: Jekyll site
---
name: Deploy Jekyll Site

on:
  push:
    branches: [main]

permissions:
  contents: read
  pages: write
  id-token: write

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout
        uses: actions/checkout@v3
      
      - name: Setup Ruby
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.1'
          bundler-cache: true
      
      - name: Setup Pages
        uses: actions/configure-pages@v3
      
      - name: Build with Jekyll
        run: bundle exec jekyll build --baseurl "${{ steps.pages.outputs.base_path }}"
        env:
          JEKYLL_ENV: production
      
      - name: Upload artifact
        uses: actions/upload-pages-artifact@v2

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v2
```

**Setup Instructions:**

1. Go to repository **Settings** → **Pages**
2. Under **Source**, select **GitHub Actions**
3. Push to main branch to trigger deployment

---

## 7. Branch Protection Rules with Required Checks

**Step 1: Create workflow** `.github/workflows/required-checks.yml`:

```yaml
name: Required Checks

on:
  pull_request:
    branches: [main, develop]

jobs:
  lint:
    name: Lint Code
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run ESLint
        run: npm run lint
      
      - name: Run Prettier
        run: npm run format:check

  test:
    name: Run Tests
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run tests
        run: npm test -- --coverage
      
      - name: Check coverage threshold
        run: |
          COVERAGE=$(cat coverage/coverage-summary.json | jq '.total.lines.pct')
          if (( $(echo "$COVERAGE < 80" | bc -l) )); then
            echo "Coverage is below 80%"
            exit 1
          fi

  security:
    name: Security Scan
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Run npm audit
        run: npm audit --audit-level=moderate
      
      - name: Run Snyk scan
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: --severity-threshold=high

  build:
    name: Build Application
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Build
        run: npm run build
      
      - name: Check build size
        run: |
          SIZE=$(du -sb dist | cut -f1)
          MAX_SIZE=5242880  # 5MB
          if [ $SIZE -gt $MAX_SIZE ]; then
            echo "Build size exceeds 5MB"
            exit 1
          fi

  # Summary job that depends on all checks
  all-checks-passed:
    name: All Required Checks Passed
    runs-on: ubuntu-latest
    needs: [lint, test, security, build]
    if: always()
    
    steps:
      - name: Check all jobs status
        run: |
          if [[ "${{ needs.lint.result }}" != "success" ]] || \
             [[ "${{ needs.test.result }}" != "success" ]] || \
             [[ "${{ needs.security.result }}" != "success" ]] || \
             [[ "${{ needs.build.result }}" != "success" ]]; then
            echo "One or more required checks failed"
            exit 1
          fi
          echo "All required checks passed!"
```

**Step 2: Configure Branch Protection:**

1. Go to **Settings** → **Branches**
2. Click **Add rule** for `main` branch
3. Configure:
   - ✅ **Require a pull request before merging**
   - ✅ **Require approvals**: 1 or more
   - ✅ **Require status checks to pass before merging**
   - Select required checks:
     - `Lint Code`
     - `Run Tests`
     - `Security Scan`
     - `Build Application`
   - ✅ **Require branches to be up to date before merging**
   - ✅ **Require conversation resolution before merging**
   - ✅ **Do not allow bypassing the above settings**
4. Click **Create**

**Step 3: Create CODEOWNERS file** `.github/CODEOWNERS`:

```
# Global owners
* @team-leads

# Specific directories
/docs/ @documentation-team
/api/ @backend-team
/frontend/ @frontend-team
/.github/ @devops-team

# Specific files
package.json @team-leads
Dockerfile @devops-team
```

---

## 8. Slack Notifications for Build Status

Create `.github/workflows/slack-notifications.yml`:

```yaml
name: Build with Slack Notifications

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Notify build started
        uses: 8398a7/action-slack@v3
        with:
          status: custom
          custom_payload: |
            {
              text: "🚀 Build Started",
              attachments: [{
                color: 'warning',
                text: `Build started for ${process.env.AS_REPO}\nBranch: ${process.env.AS_REF}\nCommit: ${process.env.AS_COMMIT}\nAuthor: ${process.env.AS_AUTHOR}`
              }]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run tests
        run: npm test
      
      - name: Build
        run: npm run build
      
      - name: Notify success
        if: success()
        uses: 8398a7/action-slack@v3
        with:
          status: custom
          custom_payload: |
            {
              text: "✅ Build Successful",
              attachments: [{
                color: 'good',
                text: `Build completed successfully!\nBranch: ${process.env.AS_REF}\nCommit: ${process.env.AS_COMMIT}\nAuthor: ${process.env.AS_AUTHOR}\nDuration: ${{ job.status }}`
              }]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
      
      - name: Notify failure
        if: failure()
        uses: 8398a7/action-slack@v3
        with:
          status: custom
          custom_payload: |
            {
              text: "❌ Build Failed",
              attachments: [{
                color: 'danger',
                text: `Build failed!\nBranch: ${process.env.AS_REF}\nCommit: ${process.env.AS_COMMIT}\nAuthor: ${process.env.AS_AUTHOR}\nView details: ${process.env.AS_WORKFLOW_RUN_URL}`
              }]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}

  # Advanced: Custom Slack notification with more details
  deploy:
    runs-on: ubuntu-latest
    needs: build
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Deploy
        run: |
          echo "Deploying..."
          # Your deployment script
      
      - name: Detailed Slack notification
        if: always()
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "Deployment ${{ job.status }}",
              "blocks": [
                {
                  "type": "header",
                  "text": {
                    "type": "plain_text",
                    "text": "${{ job.status == 'success' && '✅' || '❌' }} Deployment ${{ job.status }}"
                  }
                },
                {
                  "type": "section",
                  "fields": [
                    {
                      "type": "mrkdwn",
                      "text": "*Repository:*\n${{ github.repository }}"
                    },
                    {
                      "type": "mrkdwn",
                      "text": "*Branch:*\n${{ github.ref_name }}"
                    },
                    {
                      "type": "mrkdwn",
                      "text": "*Author:*\n${{ github.actor }}"
                    },
                    {
                      "type": "mrkdwn",
                      "text": "*Commit:*\n<${{ github.event.head_commit.url }}|${{ github.sha }}>"
                    }
                  ]
                },
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "*Commit Message:*\n${{ github.event.head_commit.message }}"
                  }
                },
                {
                  "type": "actions",
                  "elements": [
                    {
                      "type": "button",
                      "text": {
                        "type": "plain_text",
                        "text": "View Workflow"
                      },
                      "url": "${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
                    }
                  ]
                }
              ]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
          SLACK_WEBHOOK_TYPE: INCOMING_WEBHOOK

  # Slack notification for PR events
  pr-notification:
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request'
    
    steps:
      - name: PR opened notification
        if: github.event.action == 'opened'
        uses: 8398a7/action-slack@v3
        with:
          status: custom
          custom_payload: |
            {
              text: "📝 New Pull Request",
              attachments: [{
                color: '#36a64f',
                text: `PR #${{ github.event.pull_request.number }}: ${{ github.event.pull_request.title }}\nAuthor: ${{ github.event.pull_request.user.login }}\n<${{ github.event.pull_request.html_url }}|View PR>`
              }]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

**Setup Instructions for Slack:**

1. Go to https://api.slack.com/apps
2. Create a new app → "From scratch"
3. Enable "Incoming Webhooks"
4. Click "Add New Webhook to Workspace"
5. Copy the webhook URL
6. Add to GitHub Secrets: `SLACK_WEBHOOK`

---

## 9. Reusable Workflows

### Create the Reusable Workflow

**Repository A (Reusable Workflow Repo): `.github/workflows/reusable-build-test.yml`**

```yaml
name: Reusable Build and Test

on:
  workflow_call:
    inputs:
      node-version:
        description: 'Node.js version to use'
        required: false
        type: string
        default: '18'
      
      run-tests:
        description: 'Whether to run tests'
        required: false
        type: boolean
        default: true
      
      environment:
        description: 'Target environment'
        required: false
        type: string
        default: 'development'
      
      build-command:
        description: 'Build command to run'
        required: false
        type: string
        default: 'npm run build'
    
    outputs:
      build-status:
        description: 'Build completion status'
        value: ${{ jobs.build.outputs.status }}
      
      artifact-name:
        description: 'Name of the build artifact'
        value: ${{ jobs.build.outputs.artifact }}
    
    secrets:
      npm-token:
        description: 'NPM authentication token'
        required: false
      
      deploy-key:
        description: 'Deployment key'
        required: false

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      status: ${{ steps.build-step.outputs.status }}
      artifact: build-${{ github.sha }}
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Setup Node.js ${{ inputs.node-version }}
        uses: actions/setup-node@v3
        with:
          node-version: ${{ inputs.node-version }}
          cache: 'npm'
      
      - name: Configure NPM
        if: secrets.npm-token != ''
        run: |
          echo "//registry.npmjs.org/:_authToken=${{ secrets.npm-token }}" > ~/.npmrc
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run linting
        run: npm run lint
        continue-on-error: false
      
      - name: Run tests
        if: inputs.run-tests
        run: npm test
      
      - name: Build application
        id: build-step
        run: |
          ${{ inputs.build-command }}
          echo "status=success" >> $GITHUB_OUTPUT
      
      - name: Upload build artifacts
        uses: actions/upload-artifact@v3
        with:
          name: build-${{ github.sha }}
          path: dist/
          retention-days: 7
      
      - name: Display build info
        run: |
          echo "Environment: ${{ inputs.environment }}"
          echo "Node version: ${{ inputs.node-version }}"
          echo "Build completed successfully!"

  security-scan:
    runs-on: ubuntu-latest
    needs: build
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Run security audit
        run: npm audit --production --audit-level=moderate
      
      - name: Check for vulnerabilities
        run: |
          echo "Security scan completed for ${{ inputs.environment }}"
```

---

### Call the Reusable Workflow from Another Repository

**Repository B (Consumer Repo): `.github/workflows/use-reusable-workflow.yml`**

```yaml
name: Use Reusable Workflow

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  workflow_dispatch:

jobs:
  # Call reusable workflow with default parameters
  call-reusable-default:
    uses: your-org/repo-a/.github/workflows/reusable-build-test.yml@main
    secrets:
      npm-token: ${{ secrets.NPM_TOKEN }}

  # Call reusable workflow with custom parameters
  call-reusable-custom:
    uses: your-org/repo-a/.github/workflows/reusable-build-test.yml@main
    with:
      node-version: '20'
      run-tests: true
      environment: 'production'
      build-command: 'npm run build:prod'
    secrets:
      npm-token: ${{ secrets.NPM_TOKEN }}
      deploy-key: ${{ secrets.DEPLOY_KEY }}

  # Use outputs from reusable workflow
  use-outputs:
    needs: call-reusable-custom
    runs-on: ubuntu-latest
    
    steps:
      - name: Display outputs
        run: |
          echo "Build Status: ${{ needs.call-reusable-custom.outputs.build-status }}"
          echo "Artifact Name: ${{ needs.call-reusable-custom.outputs.artifact-name }}"
      
      - name: Download artifact
        uses: actions/download-artifact@v3
        with:
          name: ${{ needs.call-reusable-custom.outputs.artifact-name }}
      
      - name: Deploy
        run: |
          echo "Deploying artifact..."
          # Your deployment logic here

  # Call multiple times with matrix
  test-matrix:
    strategy:
      matrix:
        node-version: [16, 18, 20]
        environment: [staging, production]
    
    uses: your-org/repo-a/.github/workflows/reusable-build-test.yml@main
    with:
      node-version: ${{ matrix.node-version }}
      environment: ${{ matrix.environment }}
    secrets:
      npm-token: ${{ secrets.NPM_TOKEN }}
```

---

### Advanced Reusable Workflow with Docker

**Reusable Workflow: `.github/workflows/reusable-docker-build.yml`**

```yaml
name: Reusable Docker Build and Push

on:
  workflow_call:
    inputs:
      image-name:
        description: 'Docker image name'
        required: true
        type: string
      
      dockerfile-path:
        description: 'Path to Dockerfile'
        required: false
        type: string
        default: './Dockerfile'
      
      platforms:
        description: 'Target platforms'
        required: false
        type: string
        default: 'linux/amd64'
      
      push-to-registry:
        description: 'Push to registry'
        required: false
        type: boolean
        default: false
    
    outputs:
      image-tag:
        description: 'Docker image tag'
        value: ${{ jobs.docker-build.outputs.tag }}
      
      image-digest:
        description: 'Docker image digest'
        value: ${{ jobs.docker-build.outputs.digest }}
    
    secrets:
      registry-username:
        required: true
      registry-password:
        required: true

jobs:
  docker-build:
    runs-on: ubuntu-latest
    outputs:
      tag: ${{ steps.meta.outputs.tags }}
      digest: ${{ steps.build.outputs.digest }}
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Set up QEMU
        uses: docker/setup-qemu-action@v2
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2
      
      - name: Login to Docker Hub
        if: inputs.push-to-registry
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.registry-username }}
          password: ${{ secrets.registry-password }}
      
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v4
        with:
          images: ${{ inputs.image-name }}
          tags: |
            type=ref,event=branch
            type=sha,prefix={{branch}}-
            type=semver,pattern={{version}}
            type=raw,value=latest,enable={{is_default_branch}}
      
      - name: Build and push
        id: build
        uses: docker/build-push-action@v4
        with:
          context: .
          file: ${{ inputs.dockerfile-path }}
          platforms: ${{ inputs.platforms }}
          push: ${{ inputs.push-to-registry }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
      
      - name: Image digest
        run: echo ${{ steps.build.outputs.digest }}
```

**Call it from consumer repo:**

```yaml
name: Build and Deploy Docker Image

on:
  push:
    branches: [main]

jobs:
  build-docker:
    uses: your-org/workflows-repo/.github/workflows/reusable-docker-build.yml@main
    with:
      image-name: your-username/your-app
      dockerfile-path: './docker/Dockerfile'
      platforms: 'linux/amd64,linux/arm64'
      push-to-registry: true
    secrets:
      registry-username: ${{ secrets.DOCKERHUB_USERNAME }}
      registry-password: ${{ secrets.DOCKERHUB_TOKEN }}

  deploy:
    needs: build-docker
    runs-on: ubuntu-latest
    steps:
      - name: Deploy with image
        run: |
          echo "Deploying image: ${{ needs.build-docker.outputs.image-tag }}"
          echo "Image digest: ${{ needs.build-docker.outputs.image-digest }}"
```

---

## 10. OpenID Connect (OIDC) with AWS

### Step 1: Configure AWS IAM

**Create IAM Identity Provider:**

1. Go to AWS IAM Console → Identity Providers
2. Click "Add Provider"
3. Select "OpenID Connect"
4. Provider URL: `https://token.actions.githubusercontent.com`
5. Audience: `sts.amazonaws.com`
6. Click "Add Provider"

**Create IAM Role with Trust Policy:**

Create file: `github-actions-trust-policy.json`

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::YOUR_ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:your-org/your-repo:*"
        }
      }
    }
  ]
}
```

**More restrictive - specific branch:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::YOUR_ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
          "token.actions.githubusercontent.com:sub": "repo:your-org/your-repo:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

**Create the role:**

```bash
# Create the role
aws iam create-role \
  --role-name GitHubActionsRole \
  --assume-role-policy-document file://github-actions-trust-policy.json

# Attach policies (adjust as needed)
aws iam attach-role-policy \
  --role-name GitHubActionsRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess

aws iam attach-role-policy \
  --role-name GitHubActionsRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonECS_FullAccess
```

---

### Step 2: Create GitHub Actions Workflow

**Basic OIDC Authentication: `.github/workflows/aws-oidc-basic.yml`**

```yaml
name: AWS OIDC Authentication

on:
  push:
    branches: [main]
  workflow_dispatch:

# Required for OIDC
permissions:
  id-token: write   # Required for requesting JWT
  contents: read    # Required for actions/checkout

jobs:
  deploy-to-aws:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsRole
          aws-region: us-east-1
          role-session-name: GitHubActionsSession
      
      - name: Verify AWS Identity
        run: |
          aws sts get-caller-identity
          echo "Successfully authenticated with AWS!"
      
      - name: List S3 Buckets
        run: aws s3 ls
      
      - name: Deploy to S3
        run: |
          aws s3 sync ./dist s3://your-bucket-name --delete
```

---

### Step 3: Advanced OIDC with Multiple AWS Accounts

**Multi-Account Deployment: `.github/workflows/aws-oidc-multi-account.yml`**

```yaml
name: Multi-Account AWS Deployment

on:
  push:
    branches: [main, staging, develop]
  workflow_dispatch:
    inputs:
      environment:
        description: 'Target environment'
        required: true
        type: choice
        options:
          - development
          - staging
          - production

permissions:
  id-token: write
  contents: read

jobs:
  determine-environment:
    runs-on: ubuntu-latest
    outputs:
      environment: ${{ steps.set-env.outputs.environment }}
      role-arn: ${{ steps.set-env.outputs.role-arn }}
      region: ${{ steps.set-env.outputs.region }}
    
    steps:
      - name: Determine environment and role
        id: set-env
        run: |
          if [[ "${{ github.event_name }}" == "workflow_dispatch" ]]; then
            ENV="${{ github.event.inputs.environment }}"
          elif [[ "${{ github.ref }}" == "refs/heads/main" ]]; then
            ENV="production"
          elif [[ "${{ github.ref }}" == "refs/heads/staging" ]]; then
            ENV="staging"
          else
            ENV="development"
          fi
          
          echo "environment=$ENV" >> $GITHUB_OUTPUT
          
          # Set role ARN based on environment
          case $ENV in
            production)
              echo "role-arn=arn:aws:iam::111111111111:role/GitHubActionsProdRole" >> $GITHUB_OUTPUT
              echo "region=us-east-1" >> $GITHUB_OUTPUT
              ;;
            staging)
              echo "role-arn=arn:aws:iam::222222222222:role/GitHubActionsStagingRole" >> $GITHUB_OUTPUT
              echo "region=us-west-2" >> $GITHUB_OUTPUT
              ;;
            development)
              echo "role-arn=arn:aws:iam::333333333333:role/GitHubActionsDevRole" >> $GITHUB_OUTPUT
              echo "region=eu-west-1" >> $GITHUB_OUTPUT
              ;;
          esac

  deploy:
    runs-on: ubuntu-latest
    needs: determine-environment
    environment:
      name: ${{ needs.determine-environment.outputs.environment }}
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Configure AWS Credentials for ${{ needs.determine-environment.outputs.environment }}
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ needs.determine-environment.outputs.role-arn }}
          aws-region: ${{ needs.determine-environment.outputs.region }}
          role-session-name: GHA-${{ github.run_id }}-${{ github.run_number }}
          role-duration-seconds: 3600  # 1 hour
      
      - name: Verify AWS Identity
        run: |
          echo "Environment: ${{ needs.determine-environment.outputs.environment }}"
          echo "Region: ${{ needs.determine-environment.outputs.region }}"
          aws sts get-caller-identity
      
      - name: Build application
        run: |
          npm ci
          npm run build
      
      - name: Deploy to S3
        run: |
          aws s3 sync ./dist s3://app-${{ needs.determine-environment.outputs.environment }} --delete
      
      - name: Invalidate CloudFront
        run: |
          DISTRIBUTION_ID=$(aws cloudfront list-distributions \
            --query "DistributionList.Items[?Comment=='${{ needs.determine-environment.outputs.environment }}'].Id" \
            --output text)
          
          aws cloudfront create-invalidation \
            --distribution-id $DISTRIBUTION_ID \
            --paths "/*"
```

---

### Step 4: OIDC with ECS Deployment

**Deploy to ECS: `.github/workflows/aws-oidc-ecs.yml`**

```yaml
name: Deploy to AWS ECS with OIDC

on:
  push:
    branches: [main]

permissions:
  id-token: write
  contents: read

env:
  AWS_REGION: us-east-1
  ECR_REPOSITORY: my-app
  ECS_SERVICE: my-app-service
  ECS_CLUSTER: my-app-cluster
  ECS_TASK_DEFINITION: .aws/task-definition.json
  CONTAINER_NAME: my-app-container

jobs:
  deploy:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsECSRole
          aws-region: ${{ env.AWS_REGION }}
      
      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2
      
      - name: Build, tag, and push image to Amazon ECR
        id: build-image
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
          echo "image=$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG" >> $GITHUB_OUTPUT
      
      - name: Fill in the new image ID in the Amazon ECS task definition
        id: task-def
        uses: aws-actions/amazon-ecs-render-task-definition@v1
        with:
          task-definition: ${{ env.ECS_TASK_DEFINITION }}
          container-name: ${{ env.CONTAINER_NAME }}
          image: ${{ steps.build-image.outputs.image }}
      
      - name: Deploy Amazon ECS task definition
        uses: aws-actions/amazon-ecs-deploy-task-definition@v1
        with:
          task-definition: ${{ steps.task-def.outputs.task-definition }}
          service: ${{ env.ECS_SERVICE }}
          cluster: ${{ env.ECS_CLUSTER }}
          wait-for-service-stability: true
      
      - name: Verify deployment
        run: |
          echo "Deployment completed successfully!"
          aws ecs describe-services \
            --cluster ${{ env.ECS_CLUSTER }} \
            --services ${{ env.ECS_SERVICE }} \
            --query 'services[0].deployments'
```

---

### Step 5: OIDC with Lambda Deployment

**Deploy Lambda: `.github/workflows/aws-oidc-lambda.yml`**

```yaml
name: Deploy Lambda with OIDC

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  id-token: write
  contents: read

jobs:
  deploy-lambda:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsLambdaRole
          aws-region: us-east-1
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
      
      - name: Install dependencies
        run: |
          cd lambda
          npm ci --production
      
      - name: Package Lambda function
        run: |
          cd lambda
          zip -r function.zip .
      
      - name: Deploy to Lambda
        run: |
          aws lambda update-function-code \
            --function-name my-lambda-function \
            --zip-file fileb://lambda/function.zip
      
      - name: Wait for update to complete
        run: |
          aws lambda wait function-updated \
            --function-name my-lambda-function
      
      - name: Publish new version
        id: publish
        run: |
          VERSION=$(aws lambda publish-version \
            --function-name my-lambda-function \
            --query 'Version' \
            --output text)
          echo "version=$VERSION" >> $GITHUB_OUTPUT
      
      - name: Update alias to new version
        run: |
          aws lambda update-alias \
            --function-name my-lambda-function \
            --name production \
            --function-version ${{ steps.publish.outputs.version }}
      
      - name: Invoke Lambda to test
        run: |
          aws lambda invoke \
            --function-name my-lambda-function \
            --payload '{"test": true}' \
            response.json
          cat response.json
```

---

### Step 6: OIDC with Terraform

**Terraform with OIDC: `.github/workflows/aws-oidc-terraform.yml`**

```yaml
name: Terraform with AWS OIDC

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:
  id-token: write
  contents: read
  pull-requests: write  # For commenting on PRs

jobs:
  terraform:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsTerraformRole
          aws-region: us-east-1
      
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v2
        with:
          terraform_version: 1.6.0
      
      - name: Terraform Format
        id: fmt
        run: terraform fmt -check
        continue-on-error: true
      
      - name: Terraform Init
        id: init
        run: terraform init
      
      - name: Terraform Validate
        id: validate
        run: terraform validate -no-color
      
      - name: Terraform Plan
        id: plan
        if: github.event_name == 'pull_request'
        run: terraform plan -no-color -input=false
        continue-on-error: true
      
      - name: Comment PR with Terraform Plan
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v6
        with:
          script: |
            const output = `#### Terraform Format and Style 🖌\`${{ steps.fmt.outcome }}\`
            #### Terraform Initialization ⚙️\`${{ steps.init.outcome }}\`
            #### Terraform Validation 🤖\`${{ steps.validate.outcome }}\`
            #### Terraform Plan 📖\`${{ steps.plan.outcome }}\`
            
            <details><summary>Show Plan</summary>
            
            \`\`\`terraform
            ${{ steps.plan.outputs.stdout }}
            \`\`\`
            
            </details>
            
            *Pusher: @${{ github.actor }}, Action: \`${{ github.event_name }}\`*`;
            
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: output
            })
      
      - name: Terraform Apply
        if: github.ref == 'refs/heads/main' && github.event_name == 'push'
        run: terraform apply -auto-approve -input=false
```

---

## Complete OIDC Setup Script

**Automated AWS Setup: `setup-aws-oidc.sh`**

```bash
#!/bin/bash

# Variables
GITHUB_ORG="your-org"
GITHUB_REPO="your-repo"
AWS_ACCOUNT_ID="123456789012"
ROLE_NAME="GitHubActionsRole"
REGION="us-east-1"

echo "Setting up AWS OIDC for GitHub Actions..."

# Create OIDC provider
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list 6938fd4d98bab03faadb97b34396831e3780aea1

# Create trust policy
cat > trust-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::${AWS_ACCOUNT_ID}:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:${GITHUB_ORG}/${GITHUB_REPO}:*"
        }
      }
    }
  ]
}
EOF

# Create IAM role
aws iam create-role \
  --role-name ${ROLE_NAME} \
  --assume-role-policy-document file://trust-policy.json \
  --description "Role for GitHub Actions OIDC"

# Attach policies (customize as needed)
aws iam attach-role-policy \
  --role-name ${ROLE_NAME} \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess

echo "OIDC setup complete!"
echo "Role ARN: arn:aws:iam::${AWS_ACCOUNT_ID}:role/${ROLE_NAME}"
echo ""
echo "Add this to your GitHub workflow:"
echo "  role-to-assume: arn:aws:iam::${AWS_ACCOUNT_ID}:role/${ROLE_NAME}"
```

---

## Benefits of OIDC over Access Keys

✅ **No long-lived credentials** - Tokens are short-lived (default 1 hour)
✅ **No credential rotation** - Tokens generated on-demand
✅ **Better security** - No secrets stored in GitHub
✅ **Granular permissions** - Restrict by repo, branch, environment
✅ **Audit trail** - CloudTrail logs show exactly what was done
✅ **Automatic expiration** - Tokens expire automatically

## Troubleshooting OIDC

**Common Issues:**

1. **"Not authorized to perform sts:AssumeRoleWithWebIdentity"**
   - Check trust policy conditions
   - Verify repo name matches exactly
   - Ensure OIDC provider is created

2. **"No OpenIDConnect provider found"**
   - Create OIDC provider in IAM first
   - Use correct provider URL