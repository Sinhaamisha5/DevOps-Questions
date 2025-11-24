# Advanced Jenkins Pipeline: Pull Requests, Semantic Versioning & Release Management

## Table of Contents
1. Pipeline Architecture Overview
2. Git Workflow Strategy
3. Pull Request Pipeline (Code Quality)
4. Release Pipeline (Versioning & Deployment)
5. Semantic Versioning Explained
6. Conventional Commits
7. Complete Jenkinsfile Implementation
8. GitHub Token Setup
9. Two-Pipeline Workflow Diagram
10. Interview Case Study

---

## Part 1: Pipeline Architecture Overview

### Traditional Single Pipeline (What We Did Before)
```
Code Push to main → Jenkins Build → Test → Docker Build → Push → Deploy
```

**Problem:** No code review, no quality gate, risky to deploy directly to production.

### Advanced Two-Pipeline Architecture (This Project)

```
Developer creates feature branch
         ↓
Developer opens Pull Request (PR)
         ↓
CODE QUALITY PIPELINE TRIGGERS
  - Checkout code
  - Run tests
  - Check code quality
  - Results posted to PR
         ↓
Human review & approval (Team Lead)
         ↓
Merge to main branch
         ↓
RELEASE PIPELINE TRIGGERS
  - Bump semantic version
  - Create Git tag & release
  - Triggers ANOTHER build
         ↓
Second run of Release Pipeline
  - Detects Git tag exists
  - Builds Docker image
  - Tags with version + commit SHA
  - Pushes to Docker Hub
  - Deploys to Kubernetes
```

**Key difference:** Two separate pipelines with human approval in between.

---

## Part 2: Git Workflow Strategy

### Branch Strategy

```
main (production-ready code)
  ↑
  └─ feature-x (developer's feature branch)
     - Multiple commits
     - Tests passing locally
     - Ready for review
```

### Workflow Steps

**Step 1: Developer Creates Feature Branch**
```bash
git checkout -b feature-x
# Make changes
git add .
git commit -m "feat: add new login feature"
# Continue making commits
git commit -m "fix: handle edge case in login"
# Push branch
git push origin feature-x
```

**Step 2: Open Pull Request (PR)**
- On GitHub, create PR from `feature-x` → `main`
- This triggers Code Quality Pipeline

**Step 3: Review & Approval**
- Tests run automatically
- Team lead reviews code + test results
- Team lead approves PR

**Step 4: Merge to Main**
- PR merged to main branch
- This triggers Release Pipeline

**Step 5: Release Pipeline Actions**
- Bumps version automatically
- Creates Git tag
- Creates GitHub release
- Tags trigger another build

**Step 6: Deployment**
- Second run of Release Pipeline
- Detects tag exists
- Builds, tags, and pushes Docker image
- Deploys to Kubernetes

---

## Part 3: Semantic Versioning

### Version Format: MAJOR.MINOR.PATCH

Example: `1.2.3`

### What Each Number Means

#### MAJOR Version (First Number: 1)
- Increment for **breaking changes**
- Users MUST review their code for compatibility
- Examples:
  - API endpoint removed
  - Data structure fundamentally changed
  - Database migration required
  - Incompatible with previous version

**Example:** `1.2.3` → `2.0.0`

#### MINOR Version (Middle Number: 2)
- Increment for **new features** (non-breaking)
- Backward compatible
- Safe to upgrade in production
- Examples:
  - New API endpoint added
  - New optional parameter
  - New feature doesn't affect existing code
  - Can still use old way of doing things

**Example:** `1.2.3` → `1.3.0`

#### PATCH Version (Last Number: 3)
- Increment for **bug fixes** and **security patches**
- Smallest changes
- Safest to upgrade
- Examples:
  - Bug fix
  - Security vulnerability patched
  - Performance improvement
  - Minor code cleanup

**Example:** `1.2.3` → `1.2.4`

### Versioning Timeline Example

```
Release 1.0.0 - Initial release
         ↓
1.0.1 - Bug fix
         ↓
1.1.0 - New feature (non-breaking)
         ↓
1.1.1 - Security patch
         ↓
1.2.0 - Another new feature
         ↓
2.0.0 - Major redesign (breaking changes)
```

### Why Semantic Versioning?

Users can determine if they can safely upgrade:
- Upgrading patch: Always safe
- Upgrading minor: Usually safe (non-breaking)
- Upgrading major: Requires careful review and possible code changes

---

## Part 4: Conventional Commits

### Commit Message Format

```
type: description
```

### Commit Types & Version Impact

#### Type: `fix`
```
fix: resolve login button not responding
```
- **Version bump:** PATCH (last number)
- **Example:** 1.2.3 → 1.2.4
- Used for: Bug fixes, small corrections

#### Type: `feat`
```
feat: add dark mode toggle
```
- **Version bump:** MINOR (middle number)
- **Example:** 1.2.3 → 1.3.0
- Used for: New features (non-breaking)

#### Type: `feat` with `!`
```
feat!: redesign authentication system
```
OR
```
feat(auth)!: new auth flow

BREAKING CHANGE: old login endpoints removed
```
- **Version bump:** MAJOR (first number)
- **Example:** 1.2.3 → 2.0.0
- Used for: Major breaking changes

### Other Commit Types (Don't Affect Version)

```
docs: update README
refactor: reorganize code structure
style: format code
test: add unit tests
chore: update dependencies
ci: modify CI configuration
```

### Examples

```bash
# Bug fix → Patch version
git commit -m "fix: handle null pointer in user lookup"

# New feature → Minor version
git commit -m "feat: add export to CSV functionality"

# Breaking change → Major version
git commit -m "feat!: remove deprecated API endpoints"

# With scope
git commit -m "feat(payment): add stripe integration"
```

---

## Part 5: Code Quality Pipeline

### Purpose
- Triggered on every **PR to main** or **push to non-main branches**
- Tests code quality before review
- Blocks merging if tests fail

### Environment Variables

```groovy
environment {
    // Change_id is set by Jenkins Multibranch Pipeline plugin
    // It contains the PR number
}
```

### Pipeline Code

```groovy
pipeline {
    agent any
    
    stages {
        stage('Print PR Info') {
            when {
                changeRequest target: 'main'
            }
            steps {
                echo "========== Pull Request Info =========="
                echo "Pull Request Number: ${CHANGE_ID}"
                echo "PR Title: ${CHANGE_TITLE}"
                echo "PR Author: ${CHANGE_AUTHOR}"
                echo "Source Branch: ${CHANGE_BRANCH}"
                echo "Target Branch: ${CHANGE_TARGET}"
            }
        }
        
        stage('Setup') {
            steps {
                echo "========== Installing Dependencies =========="
                sh '''
                    pip install -r requirements.txt
                '''
            }
        }
        
        stage('Test') {
            steps {
                echo "========== Running Tests =========="
                sh '''
                    pytest -v --tb=short
                '''
            }
        }
    }
    
    post {
        success {
            echo "✓ Code Quality Pipeline Passed"
        }
        failure {
            echo "✗ Code Quality Pipeline Failed - PR cannot be merged"
        }
    }
}
```

### How It Triggers

**On PR Creation:**
```
GitHub → Webhook → Jenkins
         ↓
Create PR from feature-x → main
         ↓
Multibranch Pipeline detects PR
         ↓
Code Quality Pipeline runs
```

**Jenkins automatically comments on PR with results**

---

## Part 6: Release Pipeline - Semantic Versioning in Action

### How It Works

**First Run (After merge to main):**
1. Detects no Git tag on commit
2. Runs `semantic-release publish`
3. Creates new commit with version bump
4. Creates Git tag (v1.0.1, v1.1.0, etc.)
5. Pushes tag to GitHub
6. **Pipeline ends here**

**Webhook triggers another build** (because commit was pushed to main)

**Second Run (Same pipeline, now tag exists):**
1. Detects Git tag exists
2. Builds Docker image with tag
3. Pushes to Docker Hub
4. Deploys to Kubernetes

### Release Pipeline Code

```groovy
pipeline {
    agent any
    
    environment {
        // Docker configuration
        DOCKER_REGISTRY = "sanjeev-thiyagarajan"
        IMAGE_NAME = "${DOCKER_REGISTRY}/jenkins-flask-app"
        IMAGE_TAG = "${IMAGE_NAME}:${GIT_COMMIT.take(8)}"
        
        // GitHub token for tagging and release
        GITHUB_TOKEN = credentials('github-token')
        
        // Kubernetes configuration
        KUBECONFIG_FILE = credentials('kubeconfig-credentials')
        DEPLOYMENT_NAME = "flask-app"
        CONTAINER_NAME = "flask-app"
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo "========== Checking out code =========="
                checkout scm
            }
        }
        
        stage('Get Git Tag') {
            steps {
                echo "========== Checking for Git tag =========="
                script {
                    // Check if current commit has a tag
                    def tagOutput = sh(
                        script: 'git describe --exact-match --tags ${GIT_COMMIT} 2>/dev/null || echo ""',
                        returnStdout: true
                    ).trim()
                    
                    env.GIT_TAG = tagOutput
                    echo "Current Git Tag: ${env.GIT_TAG}"
                }
            }
        }
        
        stage('Setup') {
            steps {
                echo "========== Installing Dependencies =========="
                sh '''
                    pip install -r requirements.txt
                '''
            }
        }
        
        stage('Create Release') {
            when {
                // Only run if there's NO tag on this commit
                expression {
                    return env.GIT_TAG == ""
                }
                branch 'main'
            }
            steps {
                echo "========== Creating new version with semantic-release =========="
                sh '''
                    # Configure git for semantic-release
                    git config user.name "Jenkins"
                    git config user.email "jenkins@example.com"
                    
                    # Run semantic-release to bump version, create tag, and release
                    npx semantic-release
                '''
            }
        }
        
        stage('Build Docker Image') {
            when {
                // Only run if there IS a tag on this commit
                expression {
                    return env.GIT_TAG != ""
                }
            }
            steps {
                echo "========== Building Docker image =========="
                script {
                    // Get version from Git tag
                    def version = env.GIT_TAG.replaceFirst(/^v/, '')
                    env.VERSION = version
                    env.IMAGE_WITH_VERSION = "${IMAGE_NAME}:${version}"
                    
                    sh '''
                        docker build -t ${IMAGE_TAG} .
                        docker tag ${IMAGE_TAG} ${IMAGE_WITH_VERSION}
                        docker images | grep ${IMAGE_NAME}
                    '''
                }
            }
        }
        
        stage('Push to Docker Hub') {
            when {
                expression {
                    return env.GIT_TAG != ""
                }
            }
            steps {
                echo "========== Logging into Docker Hub =========="
                withCredentials([usernamePassword(
                    credentialsId: 'docker-hub-credentials',
                    usernameVariable: 'DOCKER_USERNAME',
                    passwordVariable: 'DOCKER_PASSWORD'
                )]) {
                    sh '''
                        echo $DOCKER_PASSWORD | docker login -u $DOCKER_USERNAME --password-stdin
                        echo "Login successful"
                    '''
                }
                
                echo "========== Pushing image to Docker Hub =========="
                sh '''
                    docker push ${IMAGE_TAG}
                    docker push ${IMAGE_WITH_VERSION}
                    echo "Images pushed successfully"
                '''
            }
        }
        
        stage('Deploy to Kubernetes') {
            when {
                expression {
                    return env.GIT_TAG != ""
                }
            }
            steps {
                echo "========== Deploying to Kubernetes =========="
                withCredentials([
                    file(credentialsId: 'kubeconfig-credentials', variable: 'KUBECONFIG_FILE'),
                    string(credentialsId: 'aws-access-key', variable: 'AWS_ACCESS_KEY_ID'),
                    string(credentialsId: 'aws-secret-key', variable: 'AWS_SECRET_ACCESS_KEY')
                ]) {
                    sh '''
                        export KUBECONFIG=$KUBECONFIG_FILE
                        export AWS_DEFAULT_REGION=us-east-1
                        
                        # Fix permissions
                        chmod 600 $KUBECONFIG_FILE
                        
                        # Update deployment with new image
                        kubectl set image deployment/${DEPLOYMENT_NAME} \
                            ${CONTAINER_NAME}=${IMAGE_WITH_VERSION} \
                            --record
                        
                        # Wait for rollout
                        kubectl rollout status deployment/${DEPLOYMENT_NAME}
                        
                        echo "Deployment complete with version: ${VERSION}"
                    '''
                }
            }
        }
    }
    
    post {
        always {
            sh 'docker logout || true'
        }
        success {
            echo "✓ Release Pipeline Completed Successfully"
        }
        failure {
            echo "✗ Release Pipeline Failed"
        }
    }
}
```

---

## Part 7: GitHub Token Setup

### Why We Need It

GitHub API operations require authentication:
- Create Git tags
- Create releases
- Add comments to PRs
- Merge PRs (if automated)

### Creating GitHub Personal Access Token

**Step 1: Go to GitHub Settings**
1. Click your profile icon (top right)
2. Select **Settings**
3. Scroll down to **Developer settings**
4. Click **Personal access tokens** → **Tokens (classic)**

**Step 2: Generate New Token**
1. Click **Generate new token**
2. Give it a name: `jenkins-ci-token`
3. Set expiration: 90 days (rotate regularly)

**Step 3: Select Scopes**
Check these permissions:
- `repo` (Full control of private repositories)
  - `repo:status` - Access commit status
  - `repo_deployment` - Access deployments
  - `public_repo` - Access public repositories
  - `repo:invite` - Accept pull requests
- `admin:repo_hook` (Full control of repository hooks)
- `write:packages` (Upload packages to GitHub Package Registry)

**Step 4: Generate Token**
- Click **Generate token**
- **Copy the token immediately** (won't show again)
- Store securely

### Adding Token to Jenkins

**In Jenkins:**
1. Click **Manage Jenkins** → **Manage Credentials**
2. Click **Global** → **Add Credentials**
3. Kind: **Secret text**
4. Secret: `<paste your GitHub token>`
5. ID: `github-token`
6. Click **Create**

### Using in Pipeline

```groovy
environment {
    GITHUB_TOKEN = credentials('github-token')
}
```

---

## Part 8: Environment Variables Reference

### Multibranch Pipeline Variables

When using Multibranch Pipeline plugin:

| Variable | Value | Example |
|----------|-------|---------|
| `CHANGE_ID` | Pull request number | `42` |
| `CHANGE_TITLE` | PR title | `Add new feature` |
| `CHANGE_AUTHOR` | PR creator | `john-dev` |
| `CHANGE_BRANCH` | Source branch | `feature-x` |
| `CHANGE_TARGET` | Target branch | `main` |

### Git Variables

| Variable | Value | Example |
|----------|-------|---------|
| `GIT_COMMIT` | Full commit SHA | `a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6` |
| `GIT_BRANCH` | Current branch | `main` or `feature-x` |
| `GIT_URL` | Repository URL | `https://github.com/user/repo.git` |

### Custom Variables

```groovy
environment {
    SHORT_SHA = "${GIT_COMMIT.take(8)}"           // First 8 chars
    VERSION = "${env.GIT_TAG ?: 'snapshot'}"      // Tag or default
    IMAGE_TAG = "myrepo/image:${SHORT_SHA}"
}
```

---

## Part 9: Troubleshooting

### Issue: `semantic-release` command not found

**Solution:**
```bash
npm install -g semantic-release @semantic-release/cli
```

Or add to requirements/package in your project.

### Issue: GitHub token permission denied

**Solution:**
1. Verify token has correct scopes
2. Check token hasn't expired
3. Verify credential ID matches in pipeline
4. Test token manually: `curl -H "Authorization: token YOUR_TOKEN" https://api.github.com/user`

### Issue: Git tag not detected on second run

**Cause:** Tag hasn't been pushed to remote

**Solution:**
```bash
# In pipeline after creating tag
git push origin ${GIT_TAG}
```

### Issue: Docker image tagged twice

**Why it happens:** You tag with both commit SHA and version

**Result:**
```
myrepo/image:a1b2c3d4
myrepo/image:1.2.3
```

Both point to same image, just different tags.

### Issue: Deployment fails but Docker push succeeded

**Cause:** Kubernetes context or credentials issue

**Solution:**
```bash
# Check context
kubectl config current-context

# Check kubeconfig permissions
ls -la $KUBECONFIG_FILE

# Fix permissions
chmod 600 $KUBECONFIG_FILE
```

---

## Part 10: Complete Workflow Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                     DEVELOPER WORKFLOW                      │
└─────────────────────────────────────────────────────────────┘

1. Developer creates branch
   ├─ git checkout -b feature-x
   ├─ Makes changes
   └─ git push origin feature-x

2. Opens Pull Request (GitHub UI)
   └─ Requests merge to main

3. CODE QUALITY PIPELINE TRIGGERS
   ├─ Webhook: PR created → Jenkins
   ├─ Pipeline runs tests
   ├─ Results posted to PR comment
   └─ Status: ✓ Passing or ✗ Failing

4. Code Review
   ├─ Team lead reviews code
   ├─ Team lead reviews test results
   └─ Team lead approves (or requests changes)

5. Merge to Main (GitHub UI)
   ├─ PR merged
   └─ Branch deleted

6. RELEASE PIPELINE TRIGGERS (First Run)
   ├─ Webhook: commit pushed to main → Jenkins
   ├─ Checks for Git tag: None found
   ├─ Runs: npx semantic-release publish
   │  ├─ Analyzes conventional commits
   │  ├─ Determines version bump
   │  ├─ Creates tag (e.g., v1.2.3)
   │  ├─ Creates GitHub release
   │  ├─ Pushes commit + tag to GitHub
   │  └─ Pipeline ends
   └─ Webhook triggers another build

7. RELEASE PIPELINE TRIGGERS (Second Run)
   ├─ Webhook: tag pushed to main → Jenkins
   ├─ Checks for Git tag: Found v1.2.3
   ├─ Builds Docker image
   ├─ Tags: myrepo/image:abc123de AND myrepo/image:1.2.3
   ├─ Pushes both tags to Docker Hub
   ├─ Deploys to Kubernetes
   │  ├─ Updates deployment image
   │  ├─ Waits for rollout
   │  └─ New version live
   └─ Pipeline complete

8. Production Running
   └─ Users accessing v1.2.3

┌─────────────────────────────────────────────────────────────┐
│                   KEY BRANCHING POINTS                      │
└─────────────────────────────────────────────────────────────┘

Code Quality Pipeline:
├─ Triggers on: PR creation, push to non-main branches
├─ Blocks: Merge if tests fail
└─ Ends with: Comment on PR

Release Pipeline (First Run):
├─ Triggers on: Push to main
├─ Creates: Version tag
└─ Ends: Creates another commit

Release Pipeline (Second Run):
├─ Triggers on: Tag pushed
├─ Creates: Docker image, deployment
└─ Ends: Live in production
```

---

## Part 11: Interview Case Study

### "How I Implemented a Two-Pipeline CI/CD Architecture with Semantic Versioning"

#### The Situation

The company I was working with had a critical problem: code was being deployed to production without proper review. Developers would push directly to main, Jenkins would build and deploy immediately, and sometimes broken code reached production. Additionally, there was no consistent versioning system, making it impossible to track which version was running or to do hotfixes.

The goals were:
1. Require code review before production deployment
2. Automate version bumping based on commit types
3. Maintain clear version history with Git tags
4. Create reproducible deployments tied to specific versions

#### The Approach

**Understanding the Requirements**

First, I had to learn semantic versioning and how it applies to CI/CD:
- MAJOR for breaking changes
- MINOR for new features
- PATCH for bug fixes

This made versioning predictable and meaningful for users.

**Architecture Design**

I designed a two-pipeline system:

1. **Code Quality Pipeline** - Runs on every PR
   - Tests code before review
   - Posts results to PR
   - Blocks merge if tests fail

2. **Release Pipeline** - Runs on commits to main
   - Two-stage execution:
     - First run: Creates version tag via semantic-release
     - Second run: Builds and deploys with version tag

**Integration with Conventional Commits**

I enforced conventional commit messages:
```
fix: bug fixes → PATCH version bump
feat: new features → MINOR version bump
feat!: breaking changes → MAJOR version bump
```

The semantic-release tool analyzes these messages and automatically bumps versions.

**Implementation - The Two-Run Pipeline Pattern**

The release pipeline was clever - it runs twice:

**First Run:** No tag exists
```
1. Merge happens
2. Pipeline detects: no Git tag
3. Runs: npx semantic-release publish
4. Creates: new commit + tag
5. End of pipeline
```

**Webhook triggers again** because commit was pushed

**Second Run:** Tag now exists
```
1. Pipeline detects: Git tag exists
2. Builds Docker image
3. Tags with both: commit SHA + version
4. Pushes to Docker Hub
5. Deploys to Kubernetes
6. End of pipeline
```

**Setting Up Credentials**

I needed several credentials:
- GitHub token (for creating tags/releases)
- Docker credentials (for pushing images)
- kubeconfig (for Kubernetes deployment)
- AWS credentials (for EKS authentication)

**The Challenge: Version Detection Logic**

The trickiest part was detecting whether a Git tag existed:

```groovy
script {
    def tagOutput = sh(
        script: 'git describe --exact-match --tags ${GIT_COMMIT} 2>/dev/null || echo ""',
        returnStdout: true
    ).trim()
    
    env.GIT_TAG = tagOutput
}
```

Then I used this in the `when` clause:

```groovy
when {
    expression {
        return env.GIT_TAG != ""  // Only deploy if tag exists
    }
}
```

**Handling Double Tagging**

Docker images tagged with two tags pointing to same image:
```
myrepo/image:abc123de      (commit SHA)
myrepo/image:1.2.3          (semantic version)
```

This gave us two ways to reference the same image - for debugging and for user-facing releases.

#### The Implementation Steps

**Step 1: Code Quality Pipeline**
```
PR created → Webhook → Code Quality Pipeline
  - Install deps
  - Run tests
  - Comment results on PR
  - Blocks merge if tests fail
```

**Step 2: Merge and Release (First Run)**
```
PR approved → Merge to main → Webhook → Release Pipeline
  - Detect: no tag
  - Run semantic-release
  - Creates tag + release
  - Pushes to GitHub
```

**Step 3: Second Trigger**
```
Tag pushed → Webhook → Release Pipeline again
  - Detect: tag exists
  - Build Docker image
  - Push with version tag
  - Deploy to K8s
```

#### The Results

**Measurable Impact:**

1. **Code Quality:** Zero production incidents from code not being tested
2. **Version Clarity:** Always know exactly which version is running
3. **Automation:** Version bumping fully automated - no manual decisions
4. **Traceability:** Every Docker image tagged with commit SHA and version
5. **Rollback:** Easy to redeploy any previous version by Git tag
6. **Consistency:** Conventional commits enforced consistent commit messages

**Timeline:**

- **Before:** 1-2 hours to manually deploy, errors in production, unclear versions
- **After:** 2-3 minutes automatic deployment, zero errors, clear versioning

#### Key Technical Decisions

1. **Two-pipeline approach:** Separates testing from deployment
2. **Semantic versioning:** Makes version meanings clear to users
3. **Conventional commits:** Automates version decisions based on code changes
4. **Double tagging:** Images tagged with both commit SHA and version number
5. **Two-run release pipeline:** Elegant way to handle versioning then deployment

#### What I Learned

1. **Semantic versioning is powerful** - It communicates meaning to users about what changed
2. **Conventional commits enforce discipline** - Forces developers to be clear about what they changed
3. **Two-stage pipelines are elegant** - Semantic-release triggering another build is clean pattern
4. **Guard deployments with reviews** - Human approval prevents disasters
5. **Versioning matters** - Being able to say "we're running v1.2.3" is valuable

#### Challenges I Faced

1. **Git tag detection:** Figuring out how to detect if a commit had a tag
   - Solution: `git describe --exact-match --tags`

2. **Version extraction from tag:** Getting version from tag like "v1.2.3"
   - Solution: `env.GIT_TAG.replaceFirst(/^v/, '')`

3. **GitHub token permissions:** Getting right scopes for semantic-release
   - Solution: Added `repo` and `admin:repo_hook` scopes

4. **Double triggering:** Understanding why pipeline runs twice
   - Solution: This was actually the intended design - first creates version, second deploys

#### Follow-Up Questions You Might Get

**Q: What if someone makes a PR without conventional commit messages?**
A: "The semantic-release tool would still run, but wouldn't detect version bump type. We enforce this with branch protection rules - commit message must match conventional format before merge allowed."

**Q: How do you handle emergency hotfixes?**
A: "You'd create a hotfix branch from a release tag, fix the issue, commit with `fix:` prefix, merge to main. Semantic-release detects it as patch version, tags appropriately, and deploys automatically."

**Q: Can you revert a deployment?**
A: "Yes, two ways: (1) Git revert creates new commit, triggers new build with higher version. (2) Manually deploy previous version using its Git tag: `kubectl set image deployment/app container=myrepo/image:v1.2.2`"

**Q: What about pre-release versions?**
A: "Semantic-release supports pre-releases with suffixes like v1.2.3-beta.1. Useful for staging deployments or alpha/beta testing."

**Q: How do you handle failed deployments?**
A: "Pipeline stops immediately. We check logs, fix issue, new commit triggers everything again. The version was already created, so we just retry. If it's a critical production issue, we can manually rollback to previous version's Docker image."

---

## Summary: Key Takeaways

1. **Two pipelines:** Code Quality (on PR) + Release (on main)
2. **Semantic versioning:** MAJOR.MINOR.PATCH with clear meaning
3. **Conventional commits:** Auto-determine version bump type
4. **Human approval:** Code review required before production
5. **Two-run pattern:** First run creates version, second deploys
6. **Double tagging:** Both commit SHA and version tag Docker images
7. **GitHub API integration:** Token needed for tagging and releases
8. **Multibranch pipeline:** Automatically handles multiple branches
9. **Traceability:** Version tied to exact Git commit
10. **Safety:** No broken code reaches production; all versions tracked