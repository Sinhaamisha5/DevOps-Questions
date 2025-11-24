# Jenkins Multi-Branch Pipeline Setup Guide

## Overview
This guide covers setting up a code quality pipeline in Jenkins using multi-branch pipelines with GitHub integration.

---

## Part 1: GitHub Access Token Setup

### Why Use an Access Token?
- Prevents hitting GitHub's rate limits when accessing repositories
- Required for publishing tags and releases via GitHub API
- Even public repositories benefit from authentication to avoid stricter rate limits

### Steps to Create Token
1. Click your GitHub user icon → **Settings**
2. Navigate to **Developer Settings** (bottom of sidebar)
3. Select **Personal Access Tokens** → **Tokens (classic)**
4. Click **Generate new token** → **Generate new token (classic)**
5. Configure token:
   - **Name**: `Jenkins` (or any descriptive name)
   - **Scopes**: Select `repo` (full control over private repositories)
   - Optional: Add `admin:repo_hook` for webhook control
6. **Generate token** and copy immediately (won't be shown again)

### Add Token to Jenkins
1. Go to Jenkins → **Credentials**
2. Click **Add Credentials**
3. Configure:
   - **Type**: Username with password
   - **Username**: `Jenkins` (any value works)
   - **Password**: Paste the GitHub access token
   - **ID**: `GitHub-access-token`
4. Click **Create**

---

## Part 2: Pipeline Organization

### Creating a Folder Structure
Organize related pipelines into folders for better management.

**Steps:**
1. Jenkins Dashboard → **New Item**
2. Enter name: `final-project`
3. Select type: **Folder**
4. Click **OK**
5. Optional: Add display name
6. Click **Save**

---

## Part 3: Multi-Branch Pipeline Setup

### What is a Multi-Branch Pipeline?
A pipeline type that:
- Automatically discovers and runs on multiple branches
- Handles pull requests natively
- Provides branch-specific build history
- Scans repository for changes automatically

### Creating the Pipeline
1. Inside your folder → **New Item**
2. Name: `code-quality`
3. Type: **Multibranch Pipeline**
4. Click **OK**

### Configuration Settings

#### Branch Sources
- **Source**: GitHub
- **Credentials**: Select `GitHub-access-token`
- **Repository URL**: Paste your GitHub repository URL

#### Behaviors - Discover Branches
Choose: **Exclude branches that are also filed as PRs**

**Why?** When a PR exists for a branch, you don't need separate branch builds since the PR build covers it.

**Options:**
- Discover all branches
- Discover only branches filed as PRs
- Exclude branches filed as PRs ✓ (recommended)

#### Behaviors - Discover Pull Requests
Choose: **Merging the pull request with current target branch revision**

**Why?** Tests the code as it will look after merging, catching integration issues early.

**Options:**
- Current pull request revision
- Merging with target branch ✓ (recommended)

#### Script Path
Set to: `Jenkinsfile-code-quality`

This tells Jenkins which Jenkinsfile to execute.

#### Scan Repository Triggers
- Select: **Periodically if not otherwise run**
- Interval: `2 minutes` (for demos; use longer intervals in production)

**Production Recommendation:** 15-30 minutes or use webhooks instead

---

## Part 4: Jenkinsfile Structure

### Basic Structure
```groovy
pipeline {
    agent any
    
    stages {
        stage('Print Environment') {
            steps {
                sh 'printenv'
            }
        }
        
        stage('PR Info') {
            when {
                changeRequest target: 'main'
            }
            steps {
                echo "Pull Request Number: ${env.CHANGE_ID}"
            }
        }
        
        stage('Setup') {
            steps {
                sh 'poetry install --with dev'
            }
        }
        
        stage('Test') {
            steps {
                sh 'poetry run pytest'
            }
        }
    }
}
```

### Key Concepts

#### Conditional Stages
```groovy
when {
    changeRequest target: 'main'
}
```
- `changeRequest`: Built-in function that evaluates to true for PRs
- `target: 'main'`: Only runs when PR targets the main branch

#### Environment Variables
Access PR and branch information:
- `${env.CHANGE_ID}`: Pull request number
- `${env.CHANGE_BRANCH}`: Source branch of PR
- `${env.CHANGE_TARGET}`: Target branch of PR
- `${env.BRANCH_NAME}`: Current branch name
- `${env.GIT_COMMIT}`: Git commit hash

---

## Part 5: Excluding Branches

### Why Exclude Main Branch?
- Separate pipelines for different purposes
- Code quality pipeline: For feature branches and PRs
- Release pipeline: For main branch commits only
- Keeps pipeline logic clean and focused

### How to Exclude Main Branch
1. Pipeline Configuration → **Branch Sources**
2. Scroll to behaviors section
3. Click **Add** → **Filter by name (with wildcards)**
4. Select **Exclude**
5. Enter: `main`
6. Click **Save**

**Result:** Pipeline ignores all commits directly to main branch.

---

## Part 6: Pipeline Workflows

### Workflow 1: Push to Main Branch
```bash
git add .
git commit -m "Update version"
git push origin main
```
**Expected Behavior:** No build triggered (main branch excluded)

### Workflow 2: Feature Branch Development
```bash
# Create and switch to new branch
git checkout -b feature-one

# Make changes
# Edit files...

# Commit and push
git add .
git commit -m "Add new feature"
git push origin feature-one
```
**Expected Behavior:** 
- Jenkins detects new branch
- Build triggered automatically
- Tests run on feature-one branch

### Workflow 3: Pull Request
```bash
# After pushing feature branch
# Go to GitHub → Compare & Pull Request
# Create PR targeting main branch
```
**Expected Behavior:**
- Jenkins detects new PR
- Separate PR build created
- PR-specific stages execute
- Tests run on merged result (PR + target branch)

---

## Part 7: Understanding Build Outputs

### Environment Variables in Console Output
When viewing console output, look for:

```
CHANGE_BRANCH=feature-one
CHANGE_TARGET=main
CHANGE_ID=31
BRANCH_NAME=PR-31
GIT_COMMIT=abc123...
```

### PR-Specific Information
- **Change Branch**: Source branch of the PR
- **Change Target**: Destination branch (usually main)
- **Change ID**: GitHub PR number
- **Branch Name**: Shows "PR-##" format for pull request builds

### Stage Execution
- **Skipped stages**: Shown when conditions not met
- **PR-only stages**: Execute only for pull requests
- **Standard stages**: Run for all builds (setup, test)

---

## Key Concepts Summary

### Multi-Branch Pipeline Benefits
1. **Automatic branch discovery** - No manual pipeline creation per branch
2. **Native PR support** - Built-in PR detection and handling
3. **Branch isolation** - Each branch maintains separate build history
4. **Conditional execution** - Different logic for PRs vs. regular commits

### Best Practices
1. **Use access tokens** - Avoid rate limiting
2. **Organize with folders** - Group related pipelines
3. **Exclude appropriately** - Separate concerns (code quality vs. release)
4. **Scan responsibly** - Balance freshness with resource usage
5. **Test merged code** - Use "merging with target" option for PRs

### Pipeline Separation Strategy
- **Code Quality Pipeline**: Feature branches + PRs (excludes main)
- **Release Pipeline**: Main branch only (covered in next section)
- **Benefit**: Clear separation of concerns, easier maintenance

---

---

## Part 8: Release Pipeline Setup

### Pipeline Type
Unlike the code quality pipeline, the release pipeline uses a **standard Jenkins pipeline** (not multi-branch).

### Why Standard Pipeline?
- Only needs to run on the main branch
- Triggered by commits to main (usually from merged PRs)
- Simpler configuration for single-branch workflow

### Creating the Release Pipeline
1. Inside `final-project` folder → **New Item**
2. Name: `release`
3. Type: **Pipeline** (not multibranch)
4. Click **OK**

### Basic Configuration

#### Build Triggers
- Enable: **GitHub hook trigger for GITScm polling**
- Allows GitHub webhooks to trigger builds automatically

#### Pipeline Configuration
- **Definition**: Pipeline script from SCM
- **SCM**: Git
- **Repository URL**: Your GitHub repository URL
- **Branch**: `*/main` (only main branch)
- **Script Path**: `Jenkinsfile-release`

---

## Part 9: Critical Jenkins Configuration for Release Pipeline

### Problem: Missing Git Tags
By default, Jenkins doesn't fetch git tags, which are essential for the release pipeline.

### Solution: Advanced Clone Behaviors
1. In pipeline configuration → **Additional Behaviours**
2. Click **Add** → **Advanced clone behaviours**
3. Enable: **Fetch tags** ✓

**Why?** The pipeline needs tags to determine if a release has already been created.

### Problem: Detached HEAD State
Jenkins checks out code in detached HEAD state, preventing commits.

### Solution: Checkout to Local Branch
1. **Additional Behaviours** → **Add**
2. Select: **Check out to specific local branch**
3. **Branch name**: `main`

**Why?** The pipeline needs to commit version changes and tags back to the repository.

---

## Part 10: Understanding the Release Pipeline Flow

### The Double-Run Pattern
The release pipeline runs **twice** for each merge to main:

```
Merge PR → Commit to main → Build #1 (No tag)
                                ↓
                         Create release + tag
                                ↓
                         New commit to main → Build #2 (Has tag)
                                                    ↓
                                            Build & Deploy
```

### Build #1: Create Release (No Tag Present)
**Sequence:**
1. Check for git tag → None found
2. Install dependencies
3. **Create Release Stage** executes:
   - Calculate next version using semantic versioning
   - Create new commit with version bump
   - Add git tag to commit
   - Publish release to GitHub
4. **Build & Deploy Stage** skipped (no tag yet)

### Build #2: Deploy Release (Tag Present)
**Sequence:**
1. Check for git tag → Tag found!
2. Install dependencies
3. **Create Release Stage** skipped (tag exists)
4. **Build & Deploy Stage** executes:
   - Login to Docker registry
   - Build Docker image with version tag
   - Push image to registry
   - Deploy to Kubernetes cluster

---

## Part 11: Release Pipeline Jenkinsfile Structure

### Complete Flow
```groovy
pipeline {
    agent any
    
    stages {
        stage('Check for Tag') {
            steps {
                script {
                    // Check if current commit has a tag
                    def tag = sh(
                        script: 'git tag --contains',
                        returnStdout: true
                    ).trim()
                    
                    // Set environment variable
                    env.GIT_TAG = tag ?: ''
                    echo "Git Tag: ${env.GIT_TAG}"
                }
            }
        }
        
        stage('Install Dependencies') {
            steps {
                sh 'poetry install'
            }
        }
        
        stage('Create Release') {
            when {
                expression { env.GIT_TAG == '' }
            }
            steps {
                script {
                    // Calculate next version
                    sh 'poetry run semantic-release version'
                    
                    // Create release, tag, and commit
                    sh 'poetry run semantic-release publish'
                    
                    echo "Published new tag"
                }
            }
        }
        
        stage('Build and Deploy') {
            when {
                expression { env.GIT_TAG != '' }
            }
            steps {
                script {
                    // Login to Docker
                    sh 'docker login ...'
                    
                    // Build image with version tag
                    sh "docker build -t myapp:${env.GIT_TAG} ."
                    
                    // Push to registry
                    sh "docker push myapp:${env.GIT_TAG}"
                    
                    // Deploy to Kubernetes
                    sh "kubectl apply -f deployment.yaml"
                }
            }
        }
    }
}
```

### Key Commands Explained

#### Check for Git Tag
```bash
git tag --contains
```
- Returns tag name if current commit is tagged
- Returns empty string if no tag exists

#### Calculate Next Version
```bash
poetry run semantic-release version
```
- Analyzes commit messages (conventional commits)
- Determines next version number based on changes
- Updates version in `pyproject.toml`

#### Publish Release
```bash
poetry run semantic-release publish
```
- Creates new commit with version changes
- Adds git tag to commit
- Creates GitHub release with changelog
- Uses GitHub API (requires access token)

---

## Part 12: Semantic Versioning with Conventional Commits

### Version Format: MAJOR.MINOR.PATCH
Example: `2.20.0` → `2.21.0`

### Commit Types and Version Impact

| Commit Type | Example | Version Change | Description |
|-------------|---------|----------------|-------------|
| `feat:` | `feat: added feature two` | 2.20.0 → 2.21.0 | New feature (MINOR bump) |
| `fix:` | `fix: corrected bug in login` | 2.20.0 → 2.20.1 | Bug fix (PATCH bump) |
| `BREAKING CHANGE:` | `feat!: redesigned API` | 2.20.0 → 3.0.0 | Breaking change (MAJOR bump) |
| `docs:` | `docs: updated README` | No change | Documentation only |
| `chore:` | `chore: updated dependencies` | No change | Maintenance task |

### Current Example
- **Starting version**: 2.20.0
- **Commit message**: `feat: added feature two`
- **Result**: Version bumped to 2.21.0 (MINOR increment)

---

## Part 13: Complete End-to-End Workflow

### Step-by-Step Process

#### 1. Create Feature Branch
```bash
git checkout main
git pull origin main
git checkout -b feature-two
```

#### 2. Make Changes
```bash
# Edit files
echo "v2 with feature two changes" >> app.py

git add .
git commit -m "feat: added feature two"
git push origin feature-two
```

**What Happens:**
- Code Quality pipeline detects new branch
- Build triggered for `feature-two`
- Tests run automatically
- ✓ Tests pass

#### 3. Create Pull Request
Go to GitHub:
- Click **Compare & pull request**
- **Title**: `feat: added feature two` (conventional commit format!)
- **Description**: Describe changes
- Click **Create pull request**

**What Happens:**
- Code Quality pipeline detects new PR
- PR build triggered (tests merged code)
- Shows as PR-32 in Jenkins
- ✓ Tests pass on merged result

#### 4. Merge Pull Request
On GitHub PR page:
- Review changes
- Click **Merge pull request**
- Confirm merge

**What Happens (Build #1):**
1. Commit to main triggers Release pipeline
2. Check for tag → None found
3. Install dependencies
4. Create Release stage executes:
   - Calculate version: 2.20.0 → 2.21.0
   - Create commit with version bump
   - Add tag `2.21.0` to commit
   - Publish release to GitHub
5. Pipeline ends

**What Happens (Build #2):**
1. New commit (with tag) triggers Release pipeline again
2. Check for tag → Found `2.21.0`!
3. Install dependencies
4. Create Release stage skipped
5. Build and Deploy stage executes:
   - Build Docker image: `myapp:2.21.0`
   - Push to Docker registry
   - Deploy to Kubernetes cluster
6. ✓ Application updated in production

#### 5. Verify Deployment
```bash
# Check Kubernetes deployment
kubectl get pods

# Check application version
curl https://your-app.com
# Output: "to-do app version two with feature two changes"
```

**Verify on GitHub:**
- Go to **Code** → **Tags**
- See new tag: `2.21.0`
- Go to **Releases**
- See new release with automated changelog

---

## Part 14: Pipeline Interaction Diagram

```
┌─────────────────────────────────────────────────────────┐
│  Developer Workflow                                      │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  1. Create feature branch (feature-two)                 │
│     ↓                                                    │
│  2. Make changes + commit                               │
│     ↓                                                    │
│  3. Push to GitHub                                      │
│     ↓                                                    │
│  ┌──────────────────────────────────┐                  │
│  │  CODE QUALITY PIPELINE           │                  │
│  │  - Detects new branch            │                  │
│  │  - Runs tests on feature-two     │                  │
│  │  - ✓ Tests pass                  │                  │
│  └──────────────────────────────────┘                  │
│     ↓                                                    │
│  4. Create Pull Request (PR-32)                         │
│     ↓                                                    │
│  ┌──────────────────────────────────┐                  │
│  │  CODE QUALITY PIPELINE           │                  │
│  │  - Detects new PR                │                  │
│  │  - Tests merged code (PR + main) │                  │
│  │  - ✓ Tests pass                  │                  │
│  └──────────────────────────────────┘                  │
│     ↓                                                    │
│  5. Merge PR to main                                    │
│                                                          │
└─────────────────────────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────────────┐
│  Release Pipeline - Build #1 (No Tag)                   │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  1. Triggered by merge to main                          │
│  2. Check for git tag → NONE                            │
│  3. Install dependencies                                │
│  4. CREATE RELEASE STAGE:                               │
│     - semantic-release version → 2.21.0                 │
│     - semantic-release publish                          │
│       • Create commit with version bump                 │
│       • Add git tag "2.21.0"                            │
│       • Publish GitHub release                          │
│  5. Skip Build & Deploy (no tag yet)                    │
│     ↓                                                    │
│  New commit to main created!                            │
│                                                          │
└─────────────────────────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────────────┐
│  Release Pipeline - Build #2 (Tag Found!)               │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  1. Triggered by new commit (tagged)                    │
│  2. Check for git tag → FOUND "2.21.0"                  │
│  3. Install dependencies                                │
│  4. Skip Create Release (already done)                  │
│  5. BUILD & DEPLOY STAGE:                               │
│     - docker login                                      │
│     - docker build -t myapp:2.21.0                      │
│     - docker push myapp:2.21.0                          │
│     - kubectl apply -f deployment.yaml                  │
│  6. ✓ Production deployment complete                    │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

---

## Part 15: Key Configuration Summary

### Code Quality Pipeline
- **Type**: Multi-branch pipeline
- **Runs on**: All branches + PRs (except main)
- **Branch exclusion**: `main`
- **Trigger**: Push to any branch or PR creation
- **Purpose**: Test code before merging

### Release Pipeline
- **Type**: Standard pipeline
- **Runs on**: `main` branch only
- **Special configs**: 
  - Fetch tags enabled
  - Checkout to local branch (main)
- **Trigger**: Merge to main
- **Purpose**: Version, release, and deploy

---

## Part 16: Troubleshooting Release Pipeline

### Issue: "git tag --contains" Returns Nothing
**Cause**: Jenkins not fetching tags
**Solution**: Enable "Fetch tags" in Advanced clone behaviours

### Issue: Cannot Create Commits
**Cause**: Detached HEAD state
**Solution**: Enable "Check out to specific local branch" → main

### Issue: Pipeline Runs Only Once
**Cause**: GitHub webhook not configured or second commit not triggering
**Solution**: 
- Verify webhook is active in GitHub settings
- Check Jenkins GitHub hook trigger is enabled
- Ensure semantic-release successfully creates second commit

### Issue: Wrong Version Number
**Cause**: Commit message doesn't follow conventional commits
**Solution**: Use proper format:
- `feat:` for features (minor bump)
- `fix:` for fixes (patch bump)
- `BREAKING CHANGE:` for major changes

### Issue: Docker/Kubernetes Deployment Fails
**Cause**: Missing credentials or incorrect configuration
**Solution**:
- Verify Docker registry credentials in Jenkins
- Confirm Kubernetes config is accessible
- Check deployment.yaml is correct

---

## Part 17: Best Practices for Release Pipeline

### 1. Commit Message Discipline
Always use conventional commit format:
```
feat: add user authentication
fix: resolve login timeout issue
docs: update API documentation
chore: upgrade dependencies
```

### 2. Protect Main Branch
Configure GitHub branch protection:
- Require pull request reviews
- Require status checks to pass (Code Quality pipeline)
- No direct pushes to main

### 3. Tag Management
- Never manually create tags (let semantic-release handle it)
- Don't delete tags from GitHub (breaks versioning history)
- Use semantic versioning consistently

### 4. Monitoring
Watch for:
- Both builds completing successfully
- Version numbers incrementing correctly
- GitHub releases created with proper changelogs
- Production deployments succeeding

### 5. Rollback Strategy
If deployment fails:
```bash
# Revert to previous version
kubectl rollout undo deployment/myapp

# Or deploy specific version
kubectl set image deployment/myapp myapp=myapp:2.20.0
```

---

## Complete Project Summary

### Pipeline Architecture
Two complementary pipelines working together:

**Code Quality Pipeline**
- ✓ Tests feature branches
- ✓ Tests pull requests
- ✓ Prevents bad code from reaching main
- ✓ Runs on every branch (except main)

**Release Pipeline**
- ✓ Automates versioning
- ✓ Creates releases and tags
- ✓ Builds production Docker images
- ✓ Deploys to Kubernetes
- ✓ Runs only on main branch

### Technologies Used
- **Jenkins**: CI/CD orchestration
- **GitHub**: Source control and releases
- **Poetry**: Python dependency management
- **Semantic Release**: Automated versioning
- **Docker**: Containerization
- **Kubernetes**: Container orchestration
- **Conventional Commits**: Standardized commit messages

### Benefits of This Architecture
1. **Separation of Concerns**: Code quality separate from releases
2. **Automation**: No manual version bumps or tag creation
3. **Consistency**: Semantic versioning enforced automatically
4. **Traceability**: Every release linked to commits and PRs
5. **Safety**: Multiple test stages before production
6. **Documentation**: Automatic changelog generation

### Final Workflow Summary
```
Feature Branch → Tests → PR → Tests → Merge → 
    → Version Bump → Tag → Build → Deploy → Production ✓
```

---

## Common Issues & Solutions

### Issue: Rate Limiting
**Solution**: Use GitHub access token with proper scopes

### Issue: Duplicate Builds
**Solution**: Use "Exclude branches that are also filed as PRs"

### Issue: PR Tests Don't Match Production
**Solution**: Use "Merging with target branch" discovery option

### Issue: Too Many Scans
**Solution**: Increase scan interval or use webhooks

### Issue: Main Branch Still Building
**Solution**: Verify branch exclusion filter is configured correctly