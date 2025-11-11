# CI/CD & DevOps Interview Questions & Answers

## Complete Guide for Infrastructure Engineer Role

---

## Table of Contents
1. [CI/CD Fundamentals](#cicd-fundamentals)
2. [Build Automation](#build-automation)
3. [Deployment Strategies](#deployment-strategies)
4. [GitOps & Version Control](#gitops-and-version-control)
5. [Monitoring & Observability](#monitoring-and-observability)
6. [DevOps Culture & Practices](#devops-culture-and-practices)

---

## CI/CD FUNDAMENTALS

### Q1: What is CI/CD? Explain the difference between Continuous Integration, Continuous Delivery, and Continuous Deployment.

**Answer:**

**Continuous Integration (CI):**
Developers frequently merge code changes into a central repository, followed by automated builds and tests.

**Key Practices:**
- Commit code daily (or multiple times per day)
- Automated build triggers on every commit
- Automated testing (unit, integration)
- Fast feedback (< 10 minutes)
- Fix broken builds immediately

**Example CI Pipeline:**
```
Developer Commits Code
        ↓
Git Webhook Triggers Build
        ↓
Checkout Code
        ↓
Run Linters (Code Quality)
        ↓
Compile/Build Application
        ↓
Run Unit Tests
        ↓
Run Integration Tests
        ↓
Code Coverage Analysis
        ↓
Security Scanning (SAST)
        ↓
Build Artifacts/Docker Image
        ↓
✅ CI Complete - Code is Verified
```

**Continuous Delivery (CD):**
Extension of CI where code is automatically built, tested, and prepared for production release, but deployment requires manual approval.

**Key Practices:**
- Automated deployment to staging
- Manual approval gate before production
- Always in deployable state
- Deployment can happen any time

**Example CD Pipeline:**
```
CI Pipeline Complete
        ↓
Deploy to Dev Environment (Automated)
        ↓
Run Smoke Tests
        ↓
Deploy to Staging (Automated)
        ↓
Run E2E Tests
        ↓
Run Performance Tests
        ↓
Security Scanning (DAST)
        ↓
Manual Approval Gate
        ↓
Deploy to Production (Manual Trigger)
        ↓
✅ CD Complete - In Production
```

**Continuous Deployment:**
Every change that passes automated tests is automatically deployed to production without manual intervention.

**Key Practices:**
- Fully automated pipeline
- No manual approval gates
- High confidence in automated testing
- Feature flags for risk mitigation
- Automated rollback on failure

**Example Continuous Deployment Pipeline:**
```
CI Pipeline Complete
        ↓
Deploy to Staging (Automated)
        ↓
Run All Automated Tests
        ↓
Automated Canary Deployment (5% traffic)
        ↓
Monitor Metrics (Error Rate, Latency)
        ↓
Gradual Rollout (25% → 50% → 100%)
        ↓
✅ Fully Deployed to Production (No Human Intervention)
```

**Comparison Table:**

| Aspect | CI | Continuous Delivery | Continuous Deployment |
|--------|----|--------------------|----------------------|
| **Build & Test** | Automated | Automated | Automated |
| **Deploy to Staging** | Manual | Automated | Automated |
| **Deploy to Production** | Manual | Manual Approval | Automated |
| **Frequency** | Every commit | On demand | Every passing commit |
| **Risk** | Low | Medium | Higher (mitigated by tests) |
| **Maturity Required** | Basic | Intermediate | Advanced |

**Visual Flow:**
```
┌─────────────┐
│   Code      │
│   Commit    │
└──────┬──────┘
       │
       ▼
┌─────────────────────────────────────┐
│  Continuous Integration (CI)         │
│  • Build                             │
│  • Test                              │
│  • Code Quality                      │
└──────┬──────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────┐
│  Continuous Delivery (CD)            │
│  • Deploy to Staging                 │
│  • Automated Tests                   │
│  • Manual Approval Gate              │
│  • Deploy to Production              │
└──────┬──────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────┐
│  Continuous Deployment               │
│  • Fully Automated                   │
│  • No Manual Gates                   │
│  • Instant Production Deployment     │
└─────────────────────────────────────┘
```

**Real-World Example:**

**Netflix (Continuous Deployment):**
- Deploys thousands of times per day
- Automated canary analysis
- Automated rollback on metrics regression
- Feature flags for risk control

**Our Implementation:**
```yaml
# .github/workflows/cicd.yml
name: CI/CD Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  # Continuous Integration
  ci:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Run Linters
        run: npm run lint
      
      - name: Build
        run: npm run build
      
      - name: Unit Tests
        run: npm test
      
      - name: Integration Tests
        run: npm run test:integration
      
      - name: Code Coverage
        run: npm run coverage
      
      - name: SonarQube Scan
        run: sonar-scanner
      
      - name: Build Docker Image
        run: docker build -t myapp:${{ github.sha }} .
      
      - name: Security Scan
        run: trivy image myapp:${{ github.sha }}

  # Continuous Delivery
  cd-staging:
    needs: ci
    if: github.ref == 'refs/heads/develop'
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to Staging
        run: |
          kubectl set image deployment/myapp \
            myapp=myapp:${{ github.sha }} \
            -n staging
      
      - name: Smoke Tests
        run: npm run test:smoke -- --env=staging
  
  # Continuous Deployment (Production)
  cd-production:
    needs: ci
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - name: Canary Deployment (10%)
        run: ./deploy-canary.sh 10
      
      - name: Monitor Metrics
        run: ./monitor-canary.sh --duration=10m
      
      - name: Gradual Rollout
        run: |
          ./deploy-canary.sh 25
          sleep 300
          ./deploy-canary.sh 50
          sleep 300
          ./deploy-canary.sh 100
```

**Benefits:**

**CI Benefits:**
- Early bug detection
- Reduced integration problems
- Better code quality
- Faster development cycle

**CD Benefits:**
- Faster time to market
- Reduced deployment risk
- Consistent deployment process
- Better reliability

**Continuous Deployment Benefits:**
- Fastest feedback from users
- Highest deployment frequency
- Smallest change batches (easier to debug)
- True DevOps culture

**Challenges:**

**CI Challenges:**
- Test suite must be fast (<10 min)
- Requires discipline to fix broken builds
- Need good test coverage

**CD Challenges:**
- Requires robust automated testing
- Need good monitoring
- Database migrations complexity
- Configuration management

**Continuous Deployment Challenges:**
- Requires highest test confidence
- Need feature flags
- Requires excellent monitoring
- Cultural shift needed

**Example Response:**
"CI ensures code quality by running automated builds and tests on every commit, giving developers fast feedback. Continuous Delivery extends this by automating deployments to staging and preparing for production, but requires manual approval for production release. Continuous Deployment goes further by automatically deploying every passing change to production without human intervention. In our team, we practice Continuous Delivery for our core banking services that require compliance reviews, but use Continuous Deployment for our internal tools where we have high test confidence and can iterate rapidly."

---

### Q2: What makes a good CI/CD pipeline?

**Answer:**

A good CI/CD pipeline should be **fast, reliable, secure, and provide clear feedback**.

**Key Characteristics:**

**1. Speed (Fast Feedback):**
```
Ideal Pipeline Timing:
├── Linting: 1-2 minutes
├── Build: 2-5 minutes
├── Unit Tests: 3-5 minutes
├── Integration Tests: 5-10 minutes
├── Security Scanning: 2-3 minutes
└── Total: < 20 minutes

If > 30 minutes:
- Developers stop waiting for feedback
- Slows down development
- Reduces CI adoption
```

**Speed Optimization Techniques:**
```yaml
# Parallel Stages
stages:
  - name: Tests
    parallel:
      - unit-tests
      - integration-tests
      - security-scan

# Caching
cache:
  paths:
    - node_modules/
    - .m2/repository/
    - pip-cache/

# Incremental Builds
build:
  script:
    - docker build --cache-from myapp:latest .
```

**2. Reliability (Consistent Results):**
```yaml
# ✅ GOOD: Deterministic
- name: Install Dependencies
  run: npm ci  # Clean install from package-lock.json

# ❌ BAD: Non-deterministic
- name: Install Dependencies
  run: npm install  # Might get different versions
```

**Reliability Practices:**
- Use locked dependency versions
- Clean environments for each build
- Retry flaky tests (but fix root cause)
- Isolated test environments
- Deterministic builds

**3. Security:**
```yaml
security-pipeline:
  stages:
    - name: Secret Detection
      run: gitleaks detect
    
    - name: Dependency Scanning
      run: npm audit --production
    
    - name: SAST (Static Analysis)
      run: sonar-scanner
    
    - name: Container Scanning
      run: trivy image myapp:latest
    
    - name: DAST (Dynamic Analysis)
      run: zap-baseline.py -t https://staging.app.com
    
    - name: License Compliance
      run: license-check
```

**4. Clear Feedback:**
```yaml
notification:
  on_failure:
    - slack:
        channel: "#builds"
        message: |
          ❌ Build Failed: ${BUILD_URL}
          Failed Stage: ${FAILED_STAGE}
          Error: ${ERROR_MESSAGE}
          Commit: ${GIT_COMMIT}
          Author: ${GIT_AUTHOR}
  
  on_success:
    - slack:
        channel: "#builds"
        message: "✅ Build Passed: ${BUILD_URL}"
```

**5. Comprehensive Testing:**
```
Testing Pyramid:
         /\
        /  \    E2E Tests (Few, Slow)
       /────\
      /      \   Integration Tests (Some, Medium)
     /────────\
    /          \  Unit Tests (Many, Fast)
   /────────────\

Ideal Ratio:
- Unit Tests: 70%
- Integration Tests: 20%
- E2E Tests: 10%
```

**6. Infrastructure as Code:**
```yaml
# Pipeline Definition in Code
# .gitlab-ci.yml
stages:
  - build
  - test
  - deploy

build:
  stage: build
  script:
    - docker build -t myapp:${CI_COMMIT_SHA} .
  artifacts:
    paths:
      - dist/

test:
  stage: test
  script:
    - npm test
  coverage: '/Coverage: \d+\.\d+%/'

deploy:
  stage: deploy
  script:
    - kubectl apply -f k8s/
  only:
    - main
```

**7. Artifact Management:**
```yaml
artifacts:
  storage:
    - docker-images: registry.company.com
    - npm-packages: artifactory/npm
    - maven-artifacts: nexus/maven
  
  retention:
    - production: 365 days
    - staging: 90 days
    - development: 30 days
  
  versioning:
    - semantic: v1.2.3
    - commit-hash: abc123f
    - build-number: #1234
```

**8. Environment Parity:**
```yaml
environments:
  dev:
    similar_to: production
    scale: 1x
  
  staging:
    similar_to: production
    scale: 0.5x
    differences:
      - synthetic_data
      - lower_resources
  
  production:
    scale: 100x
    high_availability: true
```

**9. Rollback Capability:**
```bash
# Automated Rollback on Failure
deploy-with-rollback() {
  PREVIOUS_VERSION=$(get-current-version)
  
  deploy-new-version $NEW_VERSION
  
  if ! health-check $NEW_VERSION; then
    echo "Health check failed, rolling back..."
    deploy-previous-version $PREVIOUS_VERSION
    exit 1
  fi
}
```

**10. Observability:**
```yaml
pipeline-metrics:
  collect:
    - build_duration
    - test_duration
    - deploy_duration
    - success_rate
    - failure_rate
    - mean_time_to_recovery
  
  dashboards:
    - grafana: pipeline-metrics
    - jenkins: build-monitor
  
  alerts:
    - build_failure_rate > 10%
    - build_duration > 30min
```

**Complete Pipeline Example:**
```yaml
name: Production-Grade CI/CD

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # Stage 1: Code Quality
  quality:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v3
      
      - name: Lint Code
        run: npm run lint
      
      - name: Check Formatting
        run: npm run format:check
      
      - name: Secret Detection
        uses: gitleaks/gitleaks-action@v2
      
      - name: License Check
        run: npm run license:check

  # Stage 2: Build
  build:
    needs: quality
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2
      
      - name: Cache Docker layers
        uses: actions/cache@v3
        with:
          path: /tmp/.buildx-cache
          key: ${{ runner.os }}-buildx-${{ github.sha }}
      
      - name: Build Docker Image
        uses: docker/build-push-action@v4
        with:
          context: .
          push: false
          tags: ${{ env.IMAGE_NAME }}:${{ github.sha }}
          cache-from: type=local,src=/tmp/.buildx-cache
          cache-to: type=local,dest=/tmp/.buildx-cache-new

  # Stage 3: Test
  test:
    needs: build
    runs-on: ubuntu-latest
    strategy:
      matrix:
        test-type: [unit, integration, e2e]
    steps:
      - uses: actions/checkout@v3
      
      - name: Run ${{ matrix.test-type }} Tests
        run: npm run test:${{ matrix.test-type }}
      
      - name: Upload Coverage
        if: matrix.test-type == 'unit'
        uses: codecov/codecov-action@v3

  # Stage 4: Security
  security:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Run Trivy Scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.IMAGE_NAME }}:${{ github.sha }}
          severity: 'CRITICAL,HIGH'
      
      - name: Run OWASP ZAP
        run: |
          docker run -t owasp/zap2docker-stable \
            zap-baseline.py -t https://staging.app.com

  # Stage 5: Deploy
  deploy:
    needs: [test, security]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - name: Deploy to Production
        uses: azure/k8s-deploy@v4
        with:
          manifests: |
            k8s/deployment.yaml
            k8s/service.yaml
          images: |
            ${{ env.IMAGE_NAME }}:${{ github.sha }}
      
      - name: Wait for Rollout
        run: |
          kubectl rollout status deployment/myapp -n production
      
      - name: Run Smoke Tests
        run: npm run test:smoke -- --env=production
      
      - name: Notify Success
        uses: 8398a7/action-slack@v3
        with:
          status: ${{ job.status }}
          text: 'Deployment to production succeeded!'
          webhook_url: ${{ secrets.SLACK_WEBHOOK }}
```

**Pipeline Anti-Patterns to Avoid:**

**❌ BAD: Flaky Tests**
```yaml
test:
  script: npm test
  retry: 3  # Masking the problem!
```

**✅ GOOD: Fix Flaky Tests**
```yaml
test:
  script: npm test
  # No retries - fix flaky tests properly
```

**❌ BAD: Long-Running Pipeline**
```yaml
# 2 hours to complete
test:
  script:
    - npm run test:all  # Includes everything
```

**✅ GOOD: Parallel & Optimized**
```yaml
test:
  parallel:
    matrix:
      - TEST: [unit, integration]
  script:
    - npm run test:$TEST
```

**❌ BAD: No Versioning**
```yaml
deploy:
  image: myapp:latest  # Which version?
```

**✅ GOOD: Explicit Versions**
```yaml
deploy:
  image: myapp:${GIT_SHA}
  # or
  image: myapp:v1.2.3-build-${BUILD_NUMBER}
```

**Metrics to Track:**

```yaml
cicd-kpis:
  speed:
    - build_time: < 20 minutes
    - deployment_frequency: multiple per day
  
  quality:
    - build_success_rate: > 95%
    - test_coverage: > 80%
    - code_quality_gate: passing
  
  reliability:
    - change_failure_rate: < 15%
    - mean_time_to_recovery: < 1 hour
  
  security:
    - critical_vulnerabilities: 0
    - secrets_exposed: 0
```

**Example Response:**
"A good CI/CD pipeline should complete in under 20 minutes, provide clear feedback on failures, and be reliable enough that developers trust it. We prioritize speed through parallelization - running unit tests, integration tests, and security scans simultaneously. We ensure reliability by using locked dependencies and clean build environments. Security is baked in with automated scanning at every stage. Our pipeline has 95%+ success rate and provides Slack notifications with direct links to failed stages, making debugging fast."

---

*[File continues with more Q&A...]*

## Quick Reference

### CI/CD Best Practices Checklist
- [ ] Build in < 20 minutes
- [ ] Run tests in parallel
- [ ] Use dependency caching
- [ ] Fail fast (quick feedback)
- [ ] Clean build environment
- [ ] Version everything
- [ ] Automated security scanning
- [ ] Clear notifications
- [ ] Rollback capability
- [ ] Monitor pipeline metrics
- [ ] Infrastructure as Code
- [ ] Blue-green or canary deployments
- [ ] Automated testing at all levels
- [ ] Secrets management
- [ ] Artifact retention policy

---

**End of CI/CD Interview Guide**