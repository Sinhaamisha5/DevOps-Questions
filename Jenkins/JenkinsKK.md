# Complete Jenkins CI/CD Master Guide

## Table of Contents
1. [Jenkins Git Checkout & SCM](#jenkins-git-checkout--scm)
2. [Build Triggers in Jenkins](#build-triggers-in-jenkins)
3. [Environment Variables](#environment-variables)
4. [Jenkins Credentials](#jenkins-credentials)
5. [Pipeline Stages - Nested & Parallel](#pipeline-stages---nested--parallel)
6. [Pipeline Options](#pipeline-options)
7. [Parameters in Jenkins](#parameters-in-jenkins)
8. [Manual Approval with Input Step](#manual-approval-with-input-step)
9. [Real-World Pipeline Examples](#real-world-pipeline-examples)
10. [Interview Case Studies](#interview-case-studies)

---

# Jenkins Git Checkout & SCM

## How Jenkins Automatic Checkout Works

By default, in a **Multibranch Pipeline** or any pipeline using SCM (Git) configured in the job, Jenkins automatically performs a **git checkout** before the pipeline starts. This is called the "Lightweight Checkout" or "SCM Checkout" step.

### Why This Happens

```
Jenkins Job Configuration
  ├─ Pipeline Script from SCM ✓
  ├─ Repository URL ✓
  └─ Branch specifier ✓
        ↓
Jenkins automatically does: git clone + git checkout
```

### The Limitation: Can't Customize Default Checkout

The default checkout **cannot be customized**, meaning:
- ❌ Cannot choose a specific branch (it uses configured branch)
- ❌ Cannot change shallow clone depth
- ❌ Cannot pass custom credentials
- ❌ Cannot checkout multiple repositories

### Solution: Disable Default Checkout

Use `skipDefaultCheckout(true)` to disable Jenkins' automatic checkout and create your own explicit checkout stage.

```groovy
pipeline {
    agent any
    
    options {
        skipDefaultCheckout(true)
    }
    
    stages {
        stage('Checkout Code') {
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: '*/release']],
                    userRemoteConfigs: [[
                        url: 'https://github.com/myrepo/app.git',
                        credentialsId: 'git-creds'
                    ]],
                    shallow: true,
                    depth: 5
                ])
            }
        }
        
        stage('Build') {
            steps {
                echo "Building code from release branch"
            }
        }
    }
}
```

### Interview Explanation

"By default, Jenkins automatically performs a git checkout before the pipeline starts. However, this default checkout cannot be customized—I cannot specify a different branch, credentials, or shallow clone depth. So in my pipelines, I use `skipDefaultCheckout(true)` to disable Jenkins' automatic checkout and then create my own explicit checkout stage where I control exactly which branch and repository to clone."

---

# Build Triggers in Jenkins

Build triggers determine **when** a Jenkins job starts. Here are the main types:

## 1. Trigger Another Job After One Job Completes

Used when Job B should start automatically after Job A finishes.

### Implementation

**In Jenkinsfile:**
```groovy
stage('Trigger Next Job') {
    steps {
        build job: 'Job-B', wait: false
    }
}
```

**Parameters:**
- `job: 'Job-B'` - Name of job to trigger
- `wait: false` - Don't wait for Job-B to complete (async)
- `wait: true` - Wait for Job-B to complete (sync)
- `parameters: [string(name: 'VERSION', value: '1.0.0')]` - Pass parameters

### Use Cases
- Multi-stage pipelines
- Nightly builds followed by deployments
- Dependent workflows

---

## 2. Timer Trigger (Build Periodically)

Runs job at a specific time, similar to cron.

### Syntax (Cron Format)

```
H 2 * * *
```

- `H` = Any hour (Jenkins spreads load)
- `2` = 2 AM
- `* * *` = Any day, any month, any day of week

### Common Examples

| Schedule | Meaning |
|----------|---------|
| `H 2 * * *` | Every day at ~2 AM |
| `H H(0-2) * * *` | Between midnight and 2 AM |
| `H/5 * * * *` | Every 5 minutes |
| `0 0 * * 0` | Every Sunday at midnight |
| `0 9-17 * * 1-5` | Every hour, 9 AM to 5 PM, weekdays |

### Use Cases
- Nightly builds
- Weekly reports
- Regular maintenance tasks
- Scheduled security scans

---

## 3. GitHub Hook Trigger (Webhook)

Triggers Jenkins instantly when code is pushed to GitHub.

### Setup

**In GitHub Repository:**
1. Go to **Settings** → **Webhooks**
2. Click **Add webhook**
3. Payload URL: `http://jenkinsserver:8080/github-webhook/`
4. Content type: `application/json`
5. Events: **Just the push event**
6. Click **Add webhook**

**In Jenkins Job:**
- Enable: **GitHub hook trigger for GITScm polling**

### How It Works

```
Developer pushes code
    ↓
GitHub sends webhook POST
    ↓
Jenkins /github-webhook/ receives notification
    ↓
Jenkins pulls code and runs pipeline
    ↓
Build starts instantly (seconds)
```

### Use Cases
- Real-time CI/CD
- Immediate feedback on code changes
- Pull request validation

---

## 4. Poll SCM (Git Polling)

Jenkins **periodically checks** the Git repository for changes.

### Syntax

```
H/5 * * * *
```

Meaning: Check every 5 minutes

### How It Works

```
Every 5 minutes:
  Jenkins runs: git fetch
  Checks for new commits
  If new commits found → triggers build
```

### Difference from GitHub Webhook

| Aspect | Webhook | Poll SCM |
|--------|---------|----------|
| **Trigger** | GitHub pushes event | Jenkins polls |
| **Speed** | Instant | Delayed (polling interval) |
| **Firewall** | Needs inbound webhook | Works behind firewall |
| **Use Case** | Real-time CI | Enterprise, no webhooks |

### Use Cases
- When GitHub webhooks can't be used
- Enterprise environments with firewalls
- Network restrictions

---

## 5. Trigger Build Remotely

Allows triggering Jenkins jobs with an HTTP GET call from external systems.

### Setup

**In Jenkins Job Configuration:**
1. Enable: **Trigger builds remotely (e.g., from scripts)**
2. Provide token: `XYZ123`

### Trigger URL

```
http://jenkinsserver/job/MyJob/build?token=XYZ123
```

### Example with curl

```bash
curl -X POST \
  http://jenkins:8080/job/Deploy/build?token=SECRET123 \
  --user admin:token
```

### With Parameters

```bash
curl -X POST \
  "http://jenkins:8080/job/Deploy/buildWithParameters" \
  --user admin:token \
  --data "VERSION=2.0.0" \
  --data "ENV=prod"
```

### Use Cases
- External monitoring systems
- ChatOps integration
- Custom deployment scripts
- Third-party tool integration

---

## Build Trigger Comparison Table

| Trigger | Speed | Setup | Use Case |
|---------|-------|-------|----------|
| **Upstream Job** | Fast | Easy | Multi-job pipelines |
| **Timer** | Scheduled | Very Easy | Nightly/weekly tasks |
| **GitHub Webhook** | Instant | Medium | Real-time CI/CD |
| **Poll SCM** | Delayed | Easy | No webhook access |
| **Remote Trigger** | Instant | Medium | External systems |

---

# Environment Variables

## Types of Environment Variables

### 1. System Environment Variables

Come from the OS where Jenkins runs.

Examples:
- `PATH`
- `JAVA_HOME`
- `HOME`
- `USER`

Use: Accessing system tools, Java, Python, etc.

### 2. Jenkins-Provided Variables

Automatically provided by Jenkins during every build.

| Variable | Meaning | Example |
|----------|---------|---------|
| `BUILD_ID` | Unique ID | `20240115_123456` |
| `BUILD_NUMBER` | Sequential number | `42` |
| `BUILD_TAG` | Job name + number | `jenkins-my-job-42` |
| `BUILD_URL` | URL to build | `http://jenkins/job/my-job/42/` |
| `JOB_NAME` | Job name | `my-job` |
| `WORKSPACE` | Working directory | `/var/jenkins_home/workspace/my-job` |
| `GIT_COMMIT` | Full commit SHA | `a1b2c3d4e5f6g7h8...` |
| `GIT_BRANCH` | Branch name | `origin/main` |
| `CHANGE_ID` | Pull request number | `42` (Multibranch only) |
| `CHANGE_TITLE` | PR title | `Add new feature` |

### 3. Custom Environment Variables

Defined globally or per-pipeline by you.

#### Globally (All Jobs)

1. **Manage Jenkins** → **Configure System**
2. **Global properties** → **Environment variables**
3. Add your variables

#### Per-Pipeline in Jenkinsfile

```groovy
pipeline {
    environment {
        MY_VAR = 'my-value'
        REGION = 'us-east-1'
        IMAGE_NAME = "myrepo/myapp"
        VERSION = "${GIT_COMMIT.take(8)}"
    }
    
    stages {
        stage('Print') {
            steps {
                echo "Region: ${env.REGION}"
                echo "Image: ${env.IMAGE_NAME}:${env.VERSION}"
            }
        }
    }
}
```

#### Inside a Stage (Scripted)

```groovy
stage('Build') {
    steps {
        script {
            env.BUILD_STATUS = "success"
            env.ARTIFACT_PATH = "${env.WORKSPACE}/build/app.jar"
        }
    }
}
```

## Environment Variables in Real-World Pipelines

### Use Case 1: Docker Image Tagging

```groovy
environment {
    DOCKER_REGISTRY = "myrepo"
    IMAGE_NAME = "${DOCKER_REGISTRY}/myapp"
    IMAGE_TAG = "${IMAGE_NAME}:${GIT_COMMIT.take(8)}"
}

stage('Build') {
    steps {
        sh "docker build -t ${IMAGE_TAG} ."
    }
}
```

### Use Case 2: Conditional Logic Based on Branch

```groovy
stage('Deploy') {
    steps {
        script {
            if (env.GIT_BRANCH == "origin/main") {
                echo "Deploying to production"
            } else {
                echo "Deploying to staging"
            }
        }
    }
}
```

### Use Case 3: Pull Request Validation

```groovy
stage('PR Validation') {
    when {
        expression {
            return env.CHANGE_ID != null
        }
    }
    steps {
        echo "Running PR tests for PR #${env.CHANGE_ID}"
    }
}
```

### Use Case 4: Build Metadata

```groovy
environment {
    BUILD_TIMESTAMP = sh(script: "date +%Y%m%d_%H%M%S", returnStdout: true).trim()
    RELEASE_VERSION = "1.0.0-${BUILD_NUMBER}"
}

stage('Package') {
    steps {
        sh "tar -czf app-${RELEASE_VERSION}.tar.gz ."
    }
}
```

## Best Practices

✔ **Never hard-code secrets** - Use Jenkins Credentials instead
✔ **Keep non-sensitive config in environment{}** - Like region, URLs, image names
✔ **Document all custom variables** - Maintain README with variable explanations
✔ **Use global env vars sparingly** - Prefer pipeline-level to avoid team conflicts
✔ **Mask sensitive values** - Jenkins does this automatically, but never echo secrets

---

# Jenkins Credentials

Jenkins Credentials securely store and manage sensitive data like passwords, API tokens, SSH keys, and certificates.

## Why Jenkins Credentials?

✔ **Security** - Secrets encrypted and masked in logs
✔ **Compliance** - Meets SOX, PCI, ISO requirements
✔ **Centralized** - Multiple pipelines reuse same credentials
✔ **Automated** - Injected automatically at runtime
✔ **Access Control** - Fine-grained permissions

## Credential Scopes

### 1. Global Scope
- Available to all pipelines
- Stored under "Global Credentials"
- Use when secret isn't job-specific
- Most common

### 2. System Scope
- Used internally by Jenkins
- Example: Jenkins connecting to GitHub
- Not typically for pipelines

### 3. User-Specific Scope
- Owned by individual user
- Only that user can use
- Rare in practice

## Types of Jenkins Credentials

| Type | Use Case | Example |
|------|----------|---------|
| **Username + Password** | Docker Hub, Git | docker login |
| **Secret Text** | API keys, tokens | GitHub token, AWS key |
| **SSH Key** | Server access, Git | Private key for deploy |
| **Secret File** | Certificates, configs | Kubeconfig, SSL cert |
| **AWS Credentials** | AWS API access | AWS key + secret |
| **Docker Registry** | Private Docker registries | Docker credentials |

## Accessing Credentials in Pipeline

### Method 1: Using `credentials()` in environment{}

```groovy
pipeline {
    environment {
        API_KEY = credentials('my-api-token')
    }
    
    stages {
        stage('Use Key') {
            steps {
                sh "curl -H 'Authorization: $API_KEY' https://api.example.com"
            }
        }
    }
}
```

✔ Good for simple secrets
✔ Jenkins masks values automatically

**Limitation:** Not suitable for username/password splitting.

### Method 2: Using `withCredentials()` (Recommended)

```groovy
withCredentials([string(credentialsId: 'my-api-token', variable: 'TOKEN')]) {
    sh 'curl -H "Authorization: $TOKEN" https://api.example.com'
}
```

✔ Credentials exist only inside block
✔ Automatically deleted after execution
✔ More secure scope

## Extracting Username & Password from Credentials

### Using `withCredentials()` (Best Method)

```groovy
withCredentials([usernamePassword(
    credentialsId: 'my-creds',
    usernameVariable: 'USER',
    passwordVariable: 'PASS'
)]) {
    echo "Username is: $USER"
    // Password masked in logs
    sh 'curl -u "$USER:$PASS" https://example.com'
}
```

✔ Jenkins splits username and password automatically
✔ Creates separate variables
✔ Password never exposed in logs
✔ **Use this method**

### Using `credentials()` (Not Recommended)

```groovy
environment {
    CREDS = credentials('my-creds')
}

stages {
    stage('Split') {
        steps {
            script {
                def parts = env.CREDS.split(':')
                def user = parts[0]
                def pass = parts[1]
                
                echo "Username: ${user}"
                // password stays in environment - RISKY!
            }
        }
    }
}
```

❌ Must manually split
❌ Password stays in environment
❌ Higher security risk
❌ Don't use this

## Using SSH Key Credentials

```groovy
withCredentials([sshUserPrivateKey(
    credentialsId: 'my-ssh-key',
    keyFileVariable: 'SSH_KEY',
    usernameVariable: 'SSH_USER'
)]) {
    sh '''
        ssh -i $SSH_KEY $SSH_USER@server.example.com "ls -l"
    '''
}
```

What Jenkins provides:
- Temporary file with private key
- SSH username variable
- Key file deleted after build

## Using AWS Credentials

```groovy
withCredentials([aws(
    credentialsId: 'my-aws-creds',
    region: 'us-east-1'
)]) {
    sh '''
        aws s3 ls
        aws ec2 describe-instances
    '''
}
```

## Credentials Setup in Jenkins UI

1. **Manage Jenkins** → **Manage Credentials**
2. Click **Global** (under Stores scoped to Jenkins)
3. Click **Add Credentials**
4. Select Kind:
   - Username with password
   - Secret text
   - SSH Username with private key
   - Secret file
   - AWS Credentials
5. Fill details
6. Give it an ID (used in pipeline)
7. Click **Create**

## Best Practices

✔ **Use least privilege** - Give secrets only to jobs that need them
✔ **Never hard-code** - Always use credentials store
✔ **Don't echo secrets** - Even with masking, avoid printing
✔ **Separate per environment** - `dev-docker-creds`, `prod-docker-creds`
✔ **Rotate regularly** - Change passwords/keys periodically
✔ **Document all** - Track owner, purpose, expiration
✔ **Use SSH over passwords** - More secure, easier for automation
✔ **Use withCredentials()** - More secure scope than credentials()

## Interview Answer (Short Version)

"Jenkins credentials securely store sensitive data like passwords, SSH keys, API tokens, and certificates, which are encrypted and masked in logs. Credentials can be global, system, or user-scoped. For simple values, I use `credentials()` in environment blocks. For SSH keys, files, or AWS keys, I use `withCredentials()` from the credentials binding plugin. Best practices include using least privilege, rotating secrets regularly, avoiding hard-coded sensitive data, and documenting all credentials."

---

# Pipeline Stages - Nested & Parallel

## Nested Stages

Nested stages group related tasks inside one main stage.

### Why Use Nested Stages?

- Group related tasks logically
- Better visualization in Jenkins UI
- Maintain order of execution
- Show hierarchy visually

### Example: Code Quality Stage with Nested Sub-Stages

```groovy
stage('Code Quality') {
    stages {
        stage('Lint') {
            steps {
                sh 'npm run lint'
            }
        }
        
        stage('Unit Tests') {
            steps {
                sh 'npm test'
            }
        }
        
        stage('Security Scan') {
            steps {
                sh 'npm audit'
            }
        }
    }
}
```

Visual representation:
```
Code Quality
 ├── Lint
 ├── Unit Tests
 └── Security Scan
```

### Real-World Nested Examples

**Build Stage:**
```
Build
 ├── Compile
 ├── Package
 └── Upload Artifact
```

**Deploy Stage:**
```
Deploy
 ├── Provision Infrastructure
 ├── Deploy Application
 └── Smoke Tests
```

**Release Stage:**
```
Release
 ├── Bump Version
 ├── Create Git Tag
 └── Publish Release Notes
```

## Parallel Stages

Parallel stages run multiple tasks simultaneously to reduce build time.

### Why Use Parallel?

- Reduce total build time
- Run independent tasks together
- Speed up test suites
- Run builds for multiple environments

### Example: Run Tests in Parallel

```groovy
stage('Tests') {
    parallel {
        stage('Unit Tests') {
            steps {
                sh 'npm run test:unit'
            }
        }
        
        stage('Integration Tests') {
            steps {
                sh 'npm run test:integration'
            }
        }
        
        stage('UI Tests') {
            steps {
                sh 'npm run test:ui'
            }
        }
    }
}
```

Timeline:
```
Without parallel: Unit (5min) → Integration (5min) → UI (5min) = 15 minutes
With parallel: All run together = 5 minutes
```

### Real-World Parallel Examples

**Parallel Testing:**
```groovy
parallel {
    stage('Unit Tests') { steps { ... } }
    stage('Integration Tests') { steps { ... } }
    stage('Security Tests') { steps { ... } }
    stage('UI Tests') { steps { ... } }
}
```

**Build for Multiple Platforms:**
```groovy
parallel {
    stage('Build Linux') { steps { ... } }
    stage('Build Windows') { steps { ... } }
    stage('Build macOS') { steps { ... } }
}
```

**Test Multiple Java Versions:**
```groovy
parallel {
    stage('Java 8') { steps { sh 'java -version' } }
    stage('Java 11') { steps { sh 'java -version' } }
    stage('Java 17') { steps { sh 'java -version' } }
}
```

**Deploy to Multiple Regions:**
```groovy
parallel {
    stage('Deploy us-east-1') { steps { ... } }
    stage('Deploy us-west-2') { steps { ... } }
    stage('Deploy eu-central-1') { steps { ... } }
}
```

**Docker Build + Helm Validation:**
```groovy
parallel {
    stage('Build Docker Image') { steps { ... } }
    stage('Lint Helm Chart') { steps { ... } }
    stage('Scan Container') { steps { ... } }
}
```

## Interview Explanation

"Nested stages are used when I want to group related tasks inside one main stage, such as a 'Code Quality' stage containing lint, tests, and security scans. This improves visualization and maintains logical order. Parallel stages are used when independent tasks can run at the same time—like running unit, integration, and UI tests simultaneously—to reduce overall build time from 20 minutes to 5 minutes. In real-world pipelines, I use parallelism for running test suites, building for multiple platforms, multi-region deployments, and environment validation."

---

# Pipeline Options

`options{}` in Jenkins Pipeline are special settings that modify how the pipeline behaves. They **do not run steps**—they only configure behavior.

## Common Pipeline Options

### 1. Timeout

Stops pipeline if it takes too long.

```groovy
options {
    timeout(time: 10, unit: 'MINUTES')
}
```

**Units:** SECONDS, MINUTES, HOURS

**Use case:** Prevent jobs from hanging forever

### 2. BuildDiscarder (Log Rotation)

Automatically deletes old builds to save disk space.

```groovy
options {
    buildDiscarder(
        logRotator(
            numToKeepStr: '10',
            daysToKeepStr: '30',
            artifactNumToKeepStr: '5',
            artifactDaysToKeepStr: '10'
        )
    )
}
```

**Parameters:**
- `numToKeepStr: '10'` - Keep last 10 builds
- `daysToKeepStr: '30'` - Keep builds from last 30 days
- `artifactNumToKeepStr: '5'` - Keep artifacts from last 5 builds
- `artifactDaysToKeepStr: '10'` - Keep artifacts from last 10 days

**Use case:** Manage disk space on Jenkins server

### 3. SkipDefaultCheckout

Disable Jenkins' automatic git checkout.

```groovy
options {
    skipDefaultCheckout(true)
}
```

**Use case:** Custom checkout with specific branch/credentials

### 4. Timestamps

Add timestamp prefix to each log line.

```groovy
options {
    timestamps()
}
```

**Output:**
```
[2024-01-15 10:30:45] Build started
[2024-01-15 10:30:46] Running tests
```

**Use case:** Easier debugging and performance analysis

### 5. DisableConcurrentBuilds

Prevent multiple builds running simultaneously.

```groovy
options {
    disableConcurrentBuilds()
}
```

**Use case:** Avoid conflicts when accessing shared resources

### 6. Retry

Retry failed stages automatically.

```groovy
options {
    retry(3)
}
```

**Use case:** Flaky tests or transient network failures

### 7. SkipStagesAfterUnstable

Stop pipeline if result is UNSTABLE.

```groovy
options {
    skipStagesAfterUnstable()
}
```

### Complete Example

```groovy
pipeline {
    agent any
    
    options {
        timeout(time: 30, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timestamps()
        disableConcurrentBuilds()
    }
    
    stages {
        stage('Build') {
            steps {
                echo 'Building...'
            }
        }
    }
}
```

---

# Parameters in Jenkins

Parameters allow running a job with dynamic inputs instead of hard-coding values. They make pipelines flexible and reusable.

## When to Use Parameters

- Deploy to different environments (dev/qa/prod)
- Pass artifact versions
- Enable/disable build steps
- Collect user input before build

## Parameter Types

### 1. String Parameter

Single-line text input.

```groovy
parameters {
    string(
        name: 'VERSION',
        defaultValue: '1.0.0',
        description: 'Application version'
    )
}
```

Use case: Version numbers, branch names, custom tags

### 2. Choice Parameter

Dropdown with fixed options.

```groovy
parameters {
    choice(
        name: 'ENV',
        choices: ['dev', 'qa', 'staging', 'prod'],
        description: 'Select environment'
    )
}
```

Use case: Environment selection, deployment strategy

### 3. Boolean Parameter

Checkbox for yes/no.

```groovy
parameters {
    booleanParam(
        name: 'RUN_TESTS',
        defaultValue: true,
        description: 'Run unit tests?'
    )
}
```

Use case: Optional steps, enable/disable features

### 4. Password Parameter

Hidden password input.

```groovy
parameters {
    password(
        name: 'API_KEY',
        defaultValue: '',
        description: 'Enter API key'
    )
}
```

⚠️ Not recommended—use Jenkins Credentials store instead

### 5. Text Parameter

Multi-line text input.

```groovy
parameters {
    text(
        name: 'CONFIG',
        defaultValue: '{}',
        description: 'JSON configuration'
    )
}
```

Use case: JSON payloads, YAML configs, release notes

## Accessing Parameters in Pipeline

Parameters are accessed via `params` object:

```groovy
pipeline {
    parameters {
        string(name: 'VERSION', defaultValue: '1.0.0')
        choice(name: 'ENV', choices: ['dev', 'prod'])
        booleanParam(name: 'RUN_TESTS', defaultValue: true)
    }
    
    stages {
        stage('Deploy') {
            steps {
                echo "Deploying ${params.VERSION} to ${params.ENV}"
                
                script {
                    if (params.RUN_TESTS) {
                        echo "Running tests..."
                    }
                }
            }
        }
    }
}
```

## Running Build with Parameters

### From Jenkins UI

1. Click job name
2. Click **Build with Parameters** (instead of Build)
3. Fill in parameter values
4. Click **Build**

### From CLI

```bash
jenkins-cli build your-job-name -p VERSION=2.0.0
```

### From API (curl)

```bash
curl -X POST http://jenkins/job/Deploy/buildWithParameters \
  --user admin:token \
  --data "ENV=prod" \
  --data "VERSION=1.2.3"
```

## Real-World Parameter Examples

### Example 1: Environment Selection

```groovy
parameters {
    choice(name: 'ENV', choices: ['dev', 'qa', 'staging', 'prod'])
}

stage('Deploy') {
    steps {
        script {
            if (params.ENV == 'prod') {
                input message: "Deploy to PRODUCTION?"
            }
            sh "deploy.sh --env=${params.ENV}"
        }
    }
}
```

### Example 2: Conditional Testing

```groovy
parameters {
    booleanParam(name: 'RUN_FULL_TESTS', defaultValue: false)
}

stage('Test') {
    steps {
        script {
            if (params.RUN_FULL_TESTS) {
                sh 'pytest --full-suite'
            } else {
                sh 'pytest --quick'
            }
        }
    }
}
```

### Example 3: Version & Build Metadata

```groovy
parameters {
    string(name: 'VERSION', defaultValue: '1.0.0')
    string(name: 'BUILD_CANDIDATE', defaultValue: '1')
}

stage('Build') {
    steps {
        sh "docker build -t app:${params.VERSION}-${params.BUILD_CANDIDATE} ."
    }
}
```

---

# Manual Approval with Input Step

The `input` step pauses a pipeline and waits for human approval before continuing. It's essential for controlling risky operations.

## When to Use Input

1. **Before Production Deployment** (most common)
2. **Change Control Approval**
3. **Canary/Blue-Green Promotion**
4. **Dangerous Operations** (DB migration, destroy)
5. **Mid-Pipeline Manual Input**

## Basic Input Syntax

```groovy
stage('Approval') {
    steps {
        input message: "Approve deployment?"
    }
}
```

## Input with Parameters

```groovy
stage('Approval') {
    steps {
        script {
            def userInput = input(
                message: "Approve and provide details",
                parameters: [
                    choice(name: 'ENV', choices: ['dev', 'prod']),
                    string(name: 'VERSION', defaultValue: '1.0.0'),
                    booleanParam(name: 'SKIP_TESTS', defaultValue: false)
                ]
            )
            
            env.SELECTED_ENV = userInput['ENV']
            env.SELECTED_VERSION = userInput['VERSION']
            env.SKIP_TESTS = userInput['SKIP_TESTS']
        }
    }
}
```

## Real-World Examples

### Example 1: Approval Before Production

```groovy
pipeline {
    agent any
    
    stages {
        stage('Build') {
            steps {
                echo 'Building application...'
                sh 'mvn build'
            }
        }
        
        stage('Test') {
            steps {
                echo 'Running tests...'
                sh 'mvn test'
            }
        }
        
        stage('Approval Gate') {
            steps {
                input message: "Deploy to PRODUCTION?", ok: 'Deploy'
            }
        }
        
        stage('Deploy to Production') {
            steps {
                echo 'Deploying to production...'
                sh './deploy-prod.sh'
            }
        }
    }
}
```

### Example 2: Approval with Collected Input

```groovy
stage('Rollback Decision') {
    steps {
        script {
            def rollbackVersion = input(
                message: 'Enter version to rollback to',
                parameters: [
                    string(name: 'ROLLBACK_VERSION', defaultValue: 'v1.0.0')
                ]
            )
            
            env.ROLLBACK_TO = rollbackVersion
        }
    }
}

stage('Execute Rollback') {
    steps {
        sh "kubectl set image deployment/app container=app:${env.ROLLBACK_TO}"
    }
}
```

### Example 3: Approval with Multiple Options

```groovy
stage('Deployment Strategy') {
    steps {
        script {
            def choice = input(
                message: "Select deployment strategy",
                parameters: [
                    choice(
                        name: 'STRATEGY',
                        choices: ['rolling', 'blue-green', 'canary'],
                        description: 'Deployment strategy'
                    )
                ]
            )
            
            env.DEPLOYMENT_STRATEGY = choice
        }
    }
}

stage('Deploy') {
    steps {
        sh "deploy.sh --strategy=${env.DEPLOYMENT_STRATEGY}"
    }
}
```

### Example 4: Skip Tests Option

```groovy
stage('Approval') {
    steps {
        script {
            def input = input(
                message: "Build options",
                parameters: [
                    booleanParam(name: 'SKIP_TESTS', defaultValue: false, description: 'Skip tests?'),
                    booleanParam(name: 'SKIP_SECURITY_SCAN', defaultValue: false)
                ]
            )
            
            env.SKIP_TESTS = input['SKIP_TESTS']
            env.SKIP_SECURITY = input['SKIP_SECURITY_SCAN']
        }
    }
}

stage('Test') {
    when {
        expression { env.SKIP_TESTS != 'true' }
    }
    steps {
        sh 'npm test'
    }
}
```

---

# Real-World Pipeline Examples

## Example 1: AWS Lambda Deployment with SAM

```groovy
pipeline {
    agent any
    
    stages {
        stage('Setup') {
            steps {
                echo '========== Installing Dependencies =========='
                sh 'pip install -r tests/requirements.txt'
            }
        }
        
        stage('Test') {
            steps {
                echo '========== Running Tests =========='
                sh '''
                    cd tests
                    pytest -v test_app.py
                    cd ..
                '''
            }
        }
        
        stage('Build') {
            steps {
                echo '========== Building with SAM =========='
                sh 'sam build -t lambda-app/template.yaml'
            }
        }
        
        stage('Deploy') {
            steps {
                echo '========== Deploying to AWS Lambda =========='
                withCredentials([
                    string(credentialsId: 'AWS-access-key', variable: 'AWS_ACCESS_KEY_ID'),
                    string(credentialsId: 'AWS-secret-key', variable: 'AWS_SECRET_ACCESS_KEY')
                ]) {
                    sh '''
                        export AWS_DEFAULT_REGION=us-east-1
                        sam deploy \
                            -t lambda-app/template.yaml \
                            --stack-name my-lambda-stack \
                            --capabilities CAPABILITY_IAM \
                            --no-confirm-changeset
                    '''
                }
            }
        }
    }
    
    post {
        success {
            echo '✓ Pipeline Succeeded'
        }
        failure {
            echo '✗ Pipeline Failed'
        }
    }
}
```

## Example 2: Docker Build & Push

```groovy
pipeline {
    agent any
    
    environment {
        DOCKER_REGISTRY = "sanjeev-thiyagarajan"
        IMAGE_NAME = "${DOCKER_REGISTRY}/jenkins-flask-app"
        IMAGE_TAG = "${IMAGE_NAME}:${GIT_COMMIT.take(8)}"
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Test') {
            steps {
                sh 'pip install -r requirements.txt && pytest -v'
            }
        }
        
        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build -t ${IMAGE_TAG} .
                    docker images | grep ${IMAGE_NAME}
                '''
            }
        }
        
        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'docker-hub-credentials',
                    usernameVariable: 'DOCKER_USERNAME',
                    passwordVariable: 'DOCKER_PASSWORD'
                )]) {
                    sh '''
                        echo $DOCKER_PASSWORD | docker login -u $DOCKER_USERNAME --password-stdin
                        docker push ${IMAGE_TAG}
                        echo "Image pushed: ${IMAGE_TAG}"
                    '''
                }
            }
        }
    }
}
```

## Example 3: Parameters + Input Approval + Deployment

```groovy
pipeline {
    agent any
    
    parameters {
        string(name: 'VERSION', defaultValue: '1.0.0', description: 'Docker image version')
        choice(name: 'ENV', choices: ['dev', 'staging', 'prod'], description: 'Target environment')
    }
    
    stages {
        stage('Build') {
            steps {
                echo "Building version ${params.VERSION}"
                sh "docker build -t app:${params.VERSION} ."
            }
        }
        
        stage('Test') {
            steps {
                sh 'npm test'
            }
        }
        
        stage('Approval Gate') {
            when {
                expression { params.ENV == 'prod' }
            }
            steps {
                input message: "Deploy ${params.VERSION} to PRODUCTION?"
            }
        }
        
        stage('Deploy') {
            steps {
                echo "Deploying to ${params.ENV}"
                sh "kubectl set image deployment/app container=app:${params.VERSION}"
            }
        }
    }
}
```

## Example 4: Parallel Testing

```groovy
pipeline {
    agent any
    
    stages {
        stage('Build') {
            steps {
                sh 'npm run build'
            }
        }
        
        stage('Tests') {
            parallel {
                stage('Unit Tests') {
                    steps {
                        sh 'npm run test:unit'
                    }
                }
                stage('Integration Tests') {
                    steps {
                        sh 'npm run test:integration'
                    }
                }
                stage('Security Scan') {
                    steps {
                        sh 'npm audit'
                    }
                }
            }
        }
        
        stage('Build Docker') {
            steps {
                sh 'docker build -t myapp .'
            }
        }
    }
}
```

## Example 5: Multi-Branch Pipeline with PR Validation

```groovy
pipeline {
    agent any
    
    stages {
        stage('Print PR Info') {
            when {
                changeRequest target: 'main'
            }
            steps {
                echo "PR #${CHANGE_ID}: ${CHANGE_TITLE}"
                echo "Author: ${CHANGE_AUTHOR}"
            }
        }
        
        stage('Setup') {
            steps {
                sh 'pip install -r requirements.txt'
            }
        }
        
        stage('Lint') {
            steps {
                sh 'pylint src/'
            }
        }
        
        stage('Test') {
            steps {
                sh 'pytest -v'
            }
        }
    }
    
    post {
        success {
            echo "✓ Code Quality Check Passed"
        }
        failure {
            echo "✗ Code Quality Check Failed - Cannot merge"
        }
    }
}
```

---

# Interview Case Studies

## Case Study 1: Two-Pipeline CI/CD with Semantic Versioning

### The Situation

Company had manual deployments causing errors. Code wasn't reviewed before production. No consistent versioning.

### The Problem

- Developers pushed directly to main
- Jenkins built and deployed immediately
- No code review gate
- Broken code reached production
- No version tracking

### The Solution

**Architecture:**
1. **Code Quality Pipeline** - Runs on every PR
2. **Release Pipeline** - Runs on commits to main

**Pipeline 1: Code Quality Pipeline**
```groovy
pipeline {
    agent any
    
    stages {
        stage('Print PR Info') {
            when {
                changeRequest target: 'main'
            }
            steps {
                echo "PR #${CHANGE_ID}: ${CHANGE_TITLE}"
            }
        }
        
        stage('Setup') {
            steps {
                sh 'pip install -r requirements.txt'
            }
        }
        
        stage('Test') {
            steps {
                sh 'pytest -v'
            }
        }
    }
}
```

**Pipeline 2: Release Pipeline (First Run)**
```groovy
stage('Create Release') {
    when {
        expression { env.GIT_TAG == "" }
        branch 'main'
    }
    steps {
        withCredentials([string(credentialsId: 'github-token', variable: 'GITHUB_TOKEN')]) {
            sh '''
                git config user.name "Jenkins"
                git config user.email "jenkins@example.com"
                npx semantic-release
            '''
        }
    }
}
```

**Pipeline 2: Release Pipeline (Second Run - After Tag)**
```groovy
stage('Build Docker Image') {
    when {
        expression { env.GIT_TAG != "" }
    }
    steps {
        sh '''
            docker build -t app:${VERSION} .
            docker push app:${VERSION}
        '''
    }
}
```

### How It Works

```
Developer creates feature branch
    ↓
Opens PR to main
    ↓
CODE QUALITY PIPELINE runs
    - Tests run
    - Results posted to PR
    ↓
Team lead reviews + approves
    ↓
Merge to main
    ↓
RELEASE PIPELINE TRIGGERS (First Run)
    - Detects: no Git tag
    - Runs: semantic-release
    - Creates: Git tag (v1.2.3)
    - Pushes: tag to GitHub
    ↓
WEBHOOK TRIGGERS ANOTHER BUILD
    ↓
RELEASE PIPELINE TRIGGERS (Second Run)
    - Detects: Git tag exists
    - Builds: Docker image
    - Pushes: to Docker Hub
    - Deploys: to Kubernetes
```

### Key Technical Decisions

1. **Conventional commits** - Commits determine version bump
   - `fix:` → PATCH
   - `feat:` → MINOR
   - `feat!:` → MAJOR

2. **Semantic versioning** - Users know what changed

3. **Two-pipeline approach** - Separates code review from deployment

4. **Git SHA tagging** - Every Docker image has commit hash

### Results

- ✓ Zero production incidents from code review
- ✓ Automatic version bumping
- ✓ Complete audit trail
- ✓ Deployment time: 2-3 minutes
- ✓ Developers trust the process

### Interview Points to Emphasize

"I designed a two-pipeline architecture where the code quality pipeline validates every PR, requiring team approval before merge. Once merged, the release pipeline uses semantic-release to automatically determine version bumps based on conventional commits, creates Git tags, and then deploys. The elegant part is the pipeline runs twice—first to create the version tag, then automatically to deploy using that tag. This ensures only reviewed, tested code reaches production with clear versioning."

---

## Case Study 2: Multi-Cluster Kubernetes Deployment

### The Situation

Company had staging and production Kubernetes clusters on AWS EKS. Manual deployments were slow and error-prone. No automated testing between environments.

### The Problem

- 20-30 minute manual deployments
- No acceptance testing before production
- Cluster switching errors
- kubeconfig permissions issues

### The Solution

**Jenkins Pipeline with Multi-Cluster Deployment:**

```groovy
pipeline {
    agent any
    
    environment {
        IMAGE_TAG = "app:${GIT_COMMIT.take(8)}"
        STAGING_CONTEXT = "staging"
        PROD_CONTEXT = "prod-us-east-1.eksctl.io"
    }
    
    stages {
        stage('Build & Push') {
            steps {
                sh 'docker build -t ${IMAGE_TAG} .'
                sh 'docker push ${IMAGE_TAG}'
            }
        }
        
        stage('Deploy to Staging') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig-credentials', variable: 'KUBECONFIG_FILE')]) {
                    sh '''
                        export KUBECONFIG=$KUBECONFIG_FILE
                        chmod 600 $KUBECONFIG_FILE
                        
                        kubectl config use-context ${STAGING_CONTEXT}
                        kubectl set image deployment/app container=app=${IMAGE_TAG}
                        kubectl rollout status deployment/app
                    '''
                }
            }
        }
        
        stage('Run Acceptance Tests') {
            steps {
                sh '''
                    SERVICE_URL=$(kubectl get svc app-service \
                        -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
                    
                    k6 run -e SERVICE_URL=$SERVICE_URL acceptance-test.js
                '''
            }
        }
        
        stage('Approval for Production') {
            steps {
                input message: "Approve deployment to PRODUCTION?"
            }
        }
        
        stage('Deploy to Production') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig-credentials', variable: 'KUBECONFIG_FILE')]) {
                    sh '''
                        export KUBECONFIG=$KUBECONFIG_FILE
                        chmod 600 $KUBECONFIG_FILE
                        
                        kubectl config use-context ${PROD_CONTEXT}
                        kubectl set image deployment/app container=app=${IMAGE_TAG}
                        kubectl rollout status deployment/app
                    '''
                }
            }
        }
    }
}
```

### Key Challenges & Solutions

**Challenge 1: kubeconfig Permissions**
```
Error: kubectl config use-context: permission denied
```

Solution: `chmod 600 $KUBECONFIG_FILE` before use

**Challenge 2: Context Switching**
```groovy
// Get context name
kubectl config get-contexts

// Switch context
kubectl config use-context staging
```

**Challenge 3: Service URL Extraction**
```bash
kubectl get svc app-service \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

### Results

- ✓ Deployment time: 2-3 minutes (from 30 minutes)
- ✓ Staging deployments catch bugs before production
- ✓ Acceptance tests verify performance
- ✓ Complete traceability with Git commit hash
- ✓ Easy rollback using kubectl set image

### Interview Points

"I implemented a multi-cluster Kubernetes deployment pipeline that deploys to staging first, runs k6 load tests to verify performance thresholds, then requires manual approval before deploying to production. The key challenge was managing kubeconfig file permissions—I had to use `chmod 600` before Jenkins could update the current-context pointer. I used `kubectl config use-context` to switch between staging and production clusters, and `kubectl JSONPath` to extract service URLs for testing. This reduced deployment time from 30 minutes to 3 minutes while ensuring only validated code reaches production."

---

## Case Study 3: Docker Image Build & Registry Push

### The Situation

Team was manually building Docker images locally and pushing to Docker Hub. Process was inconsistent. No CI/CD automation.

### The Problem

- Manual docker build and push
- Developers doing different things
- No image versioning strategy
- No audit trail

### The Solution

**Complete CI/CD Pipeline for Docker:**

```groovy
pipeline {
    agent any
    
    environment {
        DOCKER_REGISTRY = "sanjeev-thiyagarajan"
        IMAGE_NAME = "${DOCKER_REGISTRY}/jenkins-flask-app"
        IMAGE_TAG = "${IMAGE_NAME}:${GIT_COMMIT.take(8)}"
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Test') {
            steps {
                sh 'pip install -r requirements.txt'
                sh 'pytest -v'
            }
        }
        
        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build -t ${IMAGE_TAG} .
                    docker images | grep ${IMAGE_NAME}
                '''
            }
        }
        
        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'docker-hub-credentials',
                    usernameVariable: 'DOCKER_USERNAME',
                    passwordVariable: 'DOCKER_PASSWORD'
                )]) {
                    sh '''
                        echo $DOCKER_PASSWORD | docker login -u $DOCKER_USERNAME --password-stdin
                        docker push ${IMAGE_TAG}
                        echo "✓ Image pushed: ${IMAGE_TAG}"
                    '''
                }
            }
        }
    }
    
    post {
        always {
            sh 'docker logout || true'
        }
    }
}
```

### Key Implementation Details

**1. Git SHA Tagging**
```
Image: myrepo/myapp:a1b2c3d4
```
- Unique identifier per commit
- Easy debugging and traceability
- Can deploy any historical version

**2. Secure Docker Login**
```bash
echo $PASSWORD | docker login -u $USERNAME --password-stdin
```
- Password never visible in logs
- Safe for CI/CD environments
- Official Docker recommended method

**3. Credentials Management**
```groovy
withCredentials([usernamePassword(...)])
```
- Credentials injected only in block
- Automatically masked in logs
- Deleted after execution

### Results

- ✓ Fully automated image building
- ✓ Consistent tagging strategy
- ✓ Complete audit trail
- ✓ Secure credential handling
- ✓ Every commit produces deployable image

### Interview Answer

"I built a Jenkins pipeline that automatically builds a Docker image after tests pass, tags it with the Git commit SHA, and pushes to Docker Hub. This ensures every commit that passes tests produces a deployable image with complete traceability. I used Jenkins credentials binding to securely pass Docker credentials—using `echo $PASSWORD | docker login --password-stdin` rather than `docker login -p` to prevent credentials appearing in logs. The image naming strategy of using the commit SHA means we can always roll back to any historical version or debug issues by knowing exactly which code is in which image."

---

## Case Study 4: Lambda Deployment with Approval Gate

### The Situation

Lambda function deployments were ad-hoc. No testing. No approval gates. Production outages from bad deployments.

### The Solution

**Lambda Pipeline with SAM & Approval:**

```groovy
pipeline {
    agent any
    
    parameters {
        booleanParam(name: 'DEPLOY_TO_PROD', defaultValue: false, description: 'Deploy to production?')
    }
    
    stages {
        stage('Setup') {
            steps {
                sh 'pip install -r tests/requirements.txt'
            }
        }
        
        stage('Test') {
            steps {
                sh 'pytest -v test_handler.py'
            }
        }
        
        stage('Build') {
            steps {
                sh 'sam build -t template.yaml'
            }
        }
        
        stage('Approval Gate') {
            when {
                expression { params.DEPLOY_TO_PROD == true }
            }
            steps {
                input message: "⚠️ Deploy to PRODUCTION?", ok: 'Deploy'
            }
        }
        
        stage('Deploy Lambda') {
            when {
                expression { params.DEPLOY_TO_PROD == true }
            }
            steps {
                withCredentials([
                    string(credentialsId: 'AWS-access-key', variable: 'AWS_ACCESS_KEY_ID'),
                    string(credentialsId: 'AWS-secret-key', variable: 'AWS_SECRET_ACCESS_KEY')
                ]) {
                    sh '''
                        export AWS_DEFAULT_REGION=us-east-1
                        sam deploy \
                            -t template.yaml \
                            --stack-name lambda-stack \
                            --capabilities CAPABILITY_IAM \
                            --no-confirm-changeset
                    '''
                }
            }
        }
    }
    
    post {
        success {
            echo '✓ Lambda deployment complete'
        }
    }
}
```

### How to Run

1. Click **Build with Parameters**
2. Check **DEPLOY_TO_PROD** checkbox
3. Click **Build**
4. Tests run automatically
5. If tests pass, Jenkins pauses
6. Manual approval required
7. Click **Proceed** to deploy

### Results

- ✓ No untested code in production
- ✓ Manual approval required
- ✓ Audit trail of who approved what
- ✓ Easy rollback (just redeploy previous version)

---

## Case Study 5: Parameters + Approval + Docker Deployment

### The Situation

Need to deploy different Docker versions to different environments with manual approval.

### The Solution

```groovy
pipeline {
    agent any
    
    parameters {
        string(name: 'DOCKER_IMAGE', defaultValue: 'localhost:5000/my-app:latest', 
               description: 'Docker image to deploy')
        choice(name: 'ENVIRONMENT', choices: ['dev', 'staging', 'prod'], 
               description: 'Target environment')
    }
    
    stages {
        stage('Validate Image') {
            steps {
                sh '''
                    echo "Validating image: ${DOCKER_IMAGE}"
                    docker pull ${DOCKER_IMAGE}
                '''
            }
        }
        
        stage('Run Smoke Tests') {
            steps {
                sh '''
                    docker run --rm ${DOCKER_IMAGE} ./run-tests.sh
                '''
            }
        }
        
        stage('Approval') {
            when {
                expression { params.ENVIRONMENT == 'prod' }
            }
            steps {
                input(
                    message: "Approve deployment to PRODUCTION?",
                    parameters: [
                        choice(name: 'APPROVAL', choices: ['Approve', 'Reject'])
                    ]
                )
            }
        }
        
        stage('Deploy') {
            steps {
                sh '''
                    echo "Deploying ${DOCKER_IMAGE} to ${ENVIRONMENT}"
                    docker run -d ${DOCKER_IMAGE}
                '''
            }
        }
    }
}
```

### Interview Explanation

"I built a parameterized Jenkins pipeline that accepts Docker image name and target environment as inputs. The pipeline validates the image, runs smoke tests, and before production deployments, requires manual approval from the team. This ensures controlled deployments with audit trails showing who approved what deployment. The approval stage is conditional—only required for production, not for dev/staging."

---

## Summary of Interview Stories

1. **Two-Pipeline CI/CD** - PR validation + semantic versioning
2. **Multi-Cluster K8s** - Staging tests + production approval
3. **Docker Registry** - Automated building + secure credentials
4. **Lambda with Approval** - SAM + manual gates
5. **Parameters + Approval** - Dynamic input + production safety

Each story demonstrates:
- ✔ Understanding of CI/CD principles
- ✔ Knowledge of specific tools (Docker, K8s, SAM)
- ✔ Security best practices (credential management)
- ✔ Risk mitigation (testing, approval gates)
- ✔ Automation thinking (reducing manual work)
- ✔ Problem-solving (handling challenges)

---

## Final Interview Tips

**Structure Your Answer:**
1. **Situation** - What was the problem?
2. **Approach** - What did you design?
3. **Implementation** - How did you build it?
4. **Challenges** - What went wrong?
5. **Results** - What was the impact?
6. **Learning** - What did you learn?

**Key Phrases to Use:**
- "I automated..."
- "I implemented..."
- "I designed..."
- "This ensures..."
- "The challenge was..."
- "The solution was..."
- "The result was..."

**Metrics to Mention:**
- Time reduction (30 min → 3 min)
- Error reduction (0 production incidents)
- Deployment frequency (1x per week → 10x per day)
- Team efficiency (time saved per month)

**Be Honest:**
- Mention actual challenges faced
- Explain how you debugged them
- Show learning mindset
- Don't overclaim knowledge