# Docker in Jenkins CI/CD Pipeline: Complete Guide

## The Concept: What and Why

### The Problem We're Solving

**Without CI/CD Docker Pipeline:**
- Developer builds Docker image locally
- Developer manually pushes to Docker Hub
- Multiple developers = inconsistent processes
- No automated testing before image is pushed
- No version tracking by Git commit
- Risk of broken code being deployed in containers

**With CI/CD Docker Pipeline:**
- Every code push automatically triggers build
- Tests run before Docker image creation
- Only passing code gets containerized
- Images tagged with Git SHA for traceability
- Complete automation from code to registry
- Audit trail of every deployment

---

## The Pipeline Flow

```
Developer pushes code to GitHub
            ↓
GitHub webhook triggers Jenkins
            ↓
Stage 1: Checkout code from Git
            ↓
Stage 2: Setup dependencies (pip install)
            ↓
Stage 3: Run tests (pytest)
            ↓
    If tests FAIL → Pipeline stops
            ↓
Stage 4: Log into Docker Hub
            ↓
Stage 5: Build Docker image
            ↓
Stage 6: Push image to Docker Hub
            ↓
Docker image is now live and available for deployment
```

---

## Key Concepts Explained

### 1. Docker Tagging Strategy

**Image naming format:**
```
username/repo-name:tag
```

**Example:**
```
sanjeev-thiyagarajan/jenkins-flask-app:a1b2c3d4e5f6
```

Breaking it down:
- `sanjeev-thiyagarajan` = Docker Hub username
- `jenkins-flask-app` = Repository name
- `a1b2c3d4e5f6` = Git SHA (commit hash)

**Why use Git SHA?**
- Each commit gets a unique image identifier
- Can trace exactly which code is in which image
- If there's a production issue, you can pull that specific commit
- No guessing about versioning

**Example workflow:**
```
Commit 1: abc123def456 → Image tagged abc123def456
Commit 2: xyz789abc123 → Image tagged xyz789abc123
Commit 3: def456xyz789 → Image tagged def456xyz789
```

### 2. Docker Login in CI/CD (The Safe Way)

**DON'T do this (insecure):**
```bash
docker login -u $USERNAME -p $PASSWORD
# WRONG! Password appears in logs and history
```

**DO this (secure):**
```bash
echo $PASSWORD | docker login -u $USERNAME --password-stdin
```

**Why?**
- Password is piped through stdin, not visible in command
- Doesn't appear in Jenkins logs
- Doesn't appear in shell history
- This is the official Docker recommended way for CI/CD

### 3. Jenkins Environment Variables

Jenkins provides built-in variables you can use:

- `$BUILD_NUMBER` - Sequential build number
- `$BUILD_ID` - Timestamp-based build ID
- `$GIT_COMMIT` - Full Git commit SHA (what we use)
- `$GIT_BRANCH` - Current Git branch
- `$WORKSPACE` - Jenkins working directory
- `$JOB_NAME` - Name of the Jenkins job

---

## Installation & Setup Steps

### Step 1: Install Docker on Jenkins Server

```bash
# Ubuntu/Linux
sudo apt-get update
sudo apt-get install docker.io

# Add Jenkins user to docker group (so Jenkins can run Docker commands)
sudo usermod -aG docker jenkins

# Restart Jenkins for group changes to take effect
sudo systemctl restart jenkins

# Verify Docker is installed
docker --version
```

**Why add Jenkins to docker group?**
Jenkins runs as the `jenkins` user. Without being in the docker group, it won't have permission to run Docker commands.

### Step 2: Create Docker Hub Repository

**In Docker Hub:**
1. Log in to hub.docker.com
2. Click **Create Repository**
3. Repository name: `jenkins-flask-app`
4. Visibility: **Public** (or Private if preferred)
5. Click **Create**

**Result:** Your image repository is now ready at:
```
docker.io/YOUR_USERNAME/jenkins-flask-app
```

### Step 3: Create Docker Hub Credentials in Jenkins

**In Jenkins UI:**
1. Click **Manage Jenkins** → **Manage Credentials**
2. Click **Global** (under Stores scoped to Jenkins)
3. Click **Add Credentials**

**Configure:**
- Kind: **Username with password**
- Username: `your-docker-hub-username`
- Password: `your-docker-hub-password` (or access token)
- ID: `docker-hub-credentials`
- Click **Create**

**Note:** Best practice is to use a Docker Hub access token, not your actual password.

---

## Complete Jenkinsfile with Docker

```groovy
pipeline {
    agent any
    
    environment {
        // Set up variables for Docker image tagging
        IMAGE_NAME = "sanjeev-thiyagarajan/jenkins-flask-app"
        IMAGE_TAG = "${IMAGE_NAME}:${GIT_COMMIT.take(8)}"
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo '========== Checking out code from Git =========='
                checkout scm
            }
        }
        
        stage('Setup') {
            steps {
                echo '========== Installing dependencies =========='
                sh '''
                    pip install -r requirements.txt
                '''
            }
        }
        
        stage('Test') {
            steps {
                echo '========== Running pytest =========='
                sh '''
                    pytest -v
                '''
            }
        }
        
        stage('Build Docker Image') {
            steps {
                echo '========== Building Docker image =========='
                sh '''
                    docker build -t ${IMAGE_TAG} .
                '''
                
                echo '========== Verifying image was created =========='
                sh '''
                    docker images | grep ${IMAGE_NAME}
                '''
            }
        }
        
        stage('Push to Docker Hub') {
            steps {
                echo '========== Logging into Docker Hub =========='
                withCredentials([usernamePassword(
                    credentialsId: 'docker-hub-credentials',
                    usernameVariable: 'USERNAME',
                    passwordVariable: 'PASSWORD'
                )]) {
                    sh '''
                        echo $PASSWORD | docker login -u $USERNAME --password-stdin
                        echo "========== Login successful =========="
                    '''
                }
                
                echo '========== Pushing image to Docker Hub =========='
                sh '''
                    docker push ${IMAGE_TAG}
                    echo "========== Image pushed successfully =========="
                    echo "Image location: ${IMAGE_TAG}"
                '''
            }
        }
    }
    
    post {
        success {
            echo '========== Pipeline completed successfully =========='
            echo "Docker image is available at: ${IMAGE_TAG}"
        }
        failure {
            echo '========== Pipeline failed =========='
            echo "Tests or build failed. Docker image was not pushed."
        }
        always {
            // Logout from Docker after pipeline completes
            sh '''
                docker logout || true
            '''
        }
    }
}
```

---

## Understanding Each Stage

### Stage 1: Checkout
```groovy
checkout scm
```
- Pulls your code from GitHub
- Sets `$GIT_COMMIT` variable automatically
- Ready for testing and building

### Stage 2: Setup
```sh
pip install -r requirements.txt
```
- Installs all Flask dependencies
- Matches what your Dockerfile will do
- Earlier detection of dependency issues

### Stage 3: Test
```sh
pytest -v
```
- Runs all automated tests
- `-v` flag for verbose output
- **Pipeline stops here if tests fail**
- No broken code gets containerized

### Stage 4: Build Docker Image
```sh
docker build -t ${IMAGE_TAG} .
```
- Builds Docker image with tag: `username/repo:commit-hash`
- The `.` means use Dockerfile in current directory
- `${IMAGE_TAG}` is environment variable we defined

**Verifying the image:**
```sh
docker images | grep ${IMAGE_NAME}
```
- Shows that image was successfully created
- Useful for debugging

### Stage 5: Push to Docker Hub
```sh
echo $PASSWORD | docker login -u $USERNAME --password-stdin
```
- Logs into Docker Hub securely
- Password piped to stdin (not visible in logs)

```sh
docker push ${IMAGE_TAG}
```
- Uploads image to Docker Hub
- Image is now available for deployment

---

## Example Dockerfile (for reference)

Your Dockerfile should look something like this:

```dockerfile
FROM python:3.9-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install -r requirements.txt

COPY . .

EXPOSE 5000

CMD ["python", "app.py"]
```

This is what Jenkins uses when it runs `docker build`.

---

## Environment Variables Reference

In your Jenkinsfile, you can use:

```groovy
// Git-related
${GIT_COMMIT}           // Full commit SHA (40 characters)
${GIT_COMMIT.take(8)}   // First 8 characters (short SHA)
${GIT_BRANCH}           // Branch name

// Build-related
${BUILD_NUMBER}         // Sequential build number
${BUILD_ID}             // Timestamp
${JOB_NAME}             // Jenkins job name

// Custom environment
${WORKSPACE}            // Where Jenkins checks out code
```

**Example tagging strategies:**

```groovy
// Strategy 1: Short Git SHA (recommended)
IMAGE_TAG = "${IMAGE_NAME}:${GIT_COMMIT.take(8)}"
// Result: myuser/myapp:a1b2c3d4

// Strategy 2: Build number
IMAGE_TAG = "${IMAGE_NAME}:build-${BUILD_NUMBER}"
// Result: myuser/myapp:build-42

// Strategy 3: Timestamp
IMAGE_TAG = "${IMAGE_NAME}:${BUILD_ID}"
// Result: myuser/myapp:1701234567890

// Strategy 4: Branch + commit (comprehensive)
IMAGE_TAG = "${IMAGE_NAME}:${GIT_BRANCH}-${GIT_COMMIT.take(8)}"
// Result: myuser/myapp:main-a1b2c3d4
```

---

## Complete Workflow Example

**Scenario: Developer pushes code fix**

```
1. Developer fixes a bug in app.py
   └─ Commit message: "Fix login bug"

2. Developer does: git push origin main
   └─ GitHub sends webhook to Jenkins

3. Jenkins receives notification
   └─ Starts build pipeline

4. Stage 1 - Checkout: Code pulled
   └─ GIT_COMMIT = "abc123def456xyz..."

5. Stage 2 - Setup: Dependencies installed
   └─ All Flask libraries ready

6. Stage 3 - Test: pytest runs
   └─ All tests pass ✓

7. Stage 4 - Build: Docker image created
   └─ Image tagged as: myuser/flask-app:abc123de

8. Stage 5 - Push: Image uploaded to Docker Hub
   └─ Image available at hub.docker.com

9. Developers/DevOps can now pull image:
   └─ docker pull myuser/flask-app:abc123de

10. Deploy to production:
    └─ Run container in Kubernetes, Docker Swarm, etc.
```

---

## What Happens If Tests Fail

```
1. Developer pushes broken code
2. Jenkins starts pipeline
3. Tests run → FAIL
4. Pipeline stops immediately
5. Docker image is NOT built
6. Docker image is NOT pushed
7. Jenkins sends notification to developer
8. Docker Hub still has the previous working image
9. Production is not affected
```

**Result:** Broken code never reaches Docker Hub or production.

---

## Interview Tips

**When asked about this:**

"I implemented a CI/CD pipeline that integrates Docker with Jenkins. Here's how it works:

1. **Automated trigger**: Every code push to GitHub triggers Jenkins automatically via webhook

2. **Quality gates**: Before we even think about Docker, the code is tested with pytest. If tests fail, we stop immediately—no broken code gets containerized

3. **Docker integration**: Once tests pass, Jenkins uses Docker to build an image, tagged with the Git commit SHA. This gives us complete traceability

4. **Secure login**: Jenkins logs into Docker Hub securely using piped credentials, not visible in logs

5. **Automated push**: The image is automatically pushed to Docker Hub, making it immediately available for deployment

6. **Key insight**: We're just automating what we were doing manually—`docker build` and `docker push`. The difference is now it's fast, reliable, and triggered automatically.

The benefits:
- No manual Docker commands needed
- Complete audit trail (which commit = which image)
- Broken code never gets containerized
- Developers just push code—everything else is automated
- Production always has reliable images"

---

## Common Issues & Solutions

| Issue | Cause | Solution |
|-------|-------|----------|
| "Docker command not found" | Jenkins user can't run Docker | Add jenkins to docker group: `sudo usermod -aG docker jenkins` |
| "Docker login failed" | Wrong credentials | Verify Docker Hub username/password in Jenkins credentials |
| "Push denied" | Wrong image name | Ensure IMAGE_NAME matches your Docker Hub repo: `username/repo-name` |
| "Permission denied" | Jenkins doesn't have docker group permission | Restart Jenkins after adding to group |
| "Image not found locally" | Build stage failed silently | Check build stage output for errors |

---

## Summary of Key Takeaways

1. **Use Git SHA for versioning** - Provides complete traceability
2. **Test before Docker** - Only containerize passing code
3. **Secure Docker login** - Use `echo $PASSWORD | docker login --password-stdin`
4. **Automate everything** - Remove manual Docker commands
5. **Environment variables** - Use Jenkins variables for dynamic tagging
6. **Jenkins needs Docker** - Install Docker on Jenkins server and add Jenkins to docker group
7. **Credentials management** - Store Docker Hub credentials securely in Jenkins