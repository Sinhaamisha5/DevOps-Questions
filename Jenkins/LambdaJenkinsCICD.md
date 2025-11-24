# Lambda in Jenkins CI/CD Pipeline

## The Concept: What and Why

### What is AWS Lambda?

AWS Lambda is a **serverless compute service** that runs your code without you needing to provision or manage servers. You upload your code, specify when it should run (via events, schedules, or API calls), and Lambda handles the scaling automatically.

### Why Lambda in a Jenkins Pipeline?

The Jenkins-Lambda integration creates a **fully automated deployment pipeline** for serverless applications. Instead of manually deploying code to Lambda every time you make changes, Jenkins automatically:

1. **Tests your code** (pytest validates functionality)
2. **Builds it** (SAM build packages everything)
3. **Deploys it to Lambda** (SAM deploy pushes to AWS)

This is the essence of CI/CD (Continuous Integration/Continuous Deployment) applied to serverless functions.

### The Three Key Commands in the Pipeline

**SAM Build:**
```bash
sam build -t lambda-app/template.yaml
```
- Prepares your Lambda function code for deployment
- Resolves dependencies
- Creates an optimized package ready for AWS

**AWS Credentials:**
```bash
export AWS_ACCESS_KEY_ID=<your-key>
export AWS_SECRET_ACCESS_KEY=<your-secret>
```
- Authenticates Jenkins with your AWS account
- These must be exact environment variable names that AWS CLI/SAM CLI expects
- Stored securely in Jenkins credentials manager

**SAM Deploy:**
```bash
sam deploy -t template.yaml
```
- Actually pushes your Lambda function to AWS
- Updates the function code in production
- Returns the endpoint URL so you can immediately test

---

## The Complete Flow: What Happens Behind the Scenes

```
You push code to GitHub
           ↓
GitHub webhook triggers Jenkins
           ↓
Jenkins checks out your code
           ↓
pip install requirements.txt (from test folder)
           ↓
pytest runs all tests
           ↓
If tests pass → SAM build packages everything
           ↓
SAM deploy updates Lambda in AWS
           ↓
Your function is live in production
           ↓
You get a URL to test immediately
```

### Why This Matters

**Without this pipeline:**
- You'd manually run `sam deploy` every time
- You might forget to test before deploying
- You could deploy broken code to production
- No audit trail of who deployed what

**With this pipeline:**
- Every code change is automatically tested
- Only passing code reaches production
- Complete audit trail (Jenkins logs everything)
- Developers just push to GitHub—deployment is automatic

---

## Interview Story: "The Lambda Deployment That Saved the Day"

Here's a compelling way to tell this in an interview:

---

### The Setup (Start here)

"In my previous role, I was working on a microservices architecture where we had several AWS Lambda functions handling critical API operations. The challenge we faced was that every time a developer made a change to a Lambda function, the deployment process was completely manual—someone would SSH into a build machine, run SAM CLI commands, and hope nothing went wrong in production.

### The Problem (Show you understand real pain)

We had a situation where one developer deployed a Lambda function without running the full test suite first. Turns out, there was a missing import statement that only our integration tests would have caught. That function went live and started throwing errors for a few minutes, affecting customer requests. It wasn't a disaster, but it was a wake-up call.

So I volunteered to set up a CI/CD pipeline that would make this impossible to happen again.

### The Solution (This is where you shine)

I built a Jenkins pipeline that uses AWS SAM (Serverless Application Model) to automate the entire deployment process. Here's the flow I implemented:

First, whenever a developer commits code to our GitHub repository, a webhook automatically triggers a Jenkins build. Jenkins then:

1. **Checks out the latest code** from GitHub
2. **Installs dependencies** from our test requirements file—this is important because I specifically used the test requirements, not production requirements, to ensure we have all the testing libraries
3. **Runs the full pytest suite** to validate the code works as expected
4. **Builds the Lambda package** using SAM build, which resolves all dependencies and creates an optimized deployment package
5. **Deploys to AWS** using SAM deploy with our AWS credentials stored securely in Jenkins' credentials manager

The credentials piece was critical—I stored the AWS access key and secret key as separate Jenkins secrets with environment variables named exactly as AWS expects them: `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`. This way, the SAM CLI can automatically authenticate.

### The Impact (Show business value)

After setting this up, we saw immediate improvements:

- **Zero manual deployments**: Developers just push to Git and forget about it
- **Faster feedback**: If tests fail, the developer knows within minutes, not hours
- **Confidence in production**: Every line of code in production has passed automated tests
- **Audit trail**: Jenkins logs show exactly when changes were deployed and by whom
- **Speed**: We went from 20-minute manual deployments to 5-minute automated ones

We even tested it that week by intentionally committing code with a bug—and sure enough, the pipeline caught it before it ever reached production. The tests failed, the deployment was blocked, and the developer fixed the issue before re-pushing.

### The Lesson (This is what you learned)

The key insight I gained is that **automation isn't just about speed—it's about removing human error from critical processes**. By automating the build, test, and deploy steps, we transformed deployment from a risky manual process into a reliable, repeatable, auditable system.

It also reinforced how important it is to understand each tool's requirements—knowing that AWS environment variables had to be *exactly* those names, understanding why we used the test requirements file instead of production requirements, knowing when to use SAM build vs. SAM deploy—those details matter.

---

## Key Points to Emphasize in Your Interview

1. **You understand the problem it solves**: Manual deployments are error-prone and slow
2. **You know the tools**: SAM, Jenkins, AWS credentials, pytest
3. **You understand the flow**: Checkout → Test → Build → Deploy
4. **You think about security**: Credentials stored securely, not hardcoded
5. **You measure impact**: Speed, reliability, and error reduction
6. **You learn from mistakes**: Show how the pipeline catches real bugs
7. **You think holistically**: It's not just automation—it's reducing human error and building confidence

---

## Technical Details Worth Mentioning

If they dig deeper, you can explain:

- **Why SAM?** It's specifically designed for serverless applications and handles CloudFormation templates
- **Why test requirements separately?** Because you don't want pytest and testing libraries in production Lambda code
- **Why GitHub webhook?** It's the trigger—when code is pushed, Jenkins automatically runs without manual intervention
- **The AWS credentials flow**: How Jenkins environment variables map to AWS CLI expectations
- **Versioning**: How the pipeline enabled you to deploy "version 2" instantly without manual steps



# Complete Lambda CI/CD Pipeline: Step-by-Step Interview Guide

## Part 1: Understanding SAM

### What is AWS SAM?

**SAM stands for AWS Serverless Application Model.** It's a framework that makes it easier to build, test, and deploy serverless applications (Lambda functions, API Gateway, DynamoDB, etc.).

Think of it as a **simplified wrapper around CloudFormation** specifically designed for serverless. Instead of writing complex CloudFormation templates, you write simpler YAML templates that SAM understands, and it handles the heavy lifting of converting that into AWS resources.

### SAM vs Manual AWS Deployment

**Without SAM (Manual way):**
- Write CloudFormation template (verbose, complex)
- Package code manually
- Upload to S3
- Deploy via CloudFormation
- Multiple manual steps

**With SAM (Automated way):**
- Write simple SAM template
- Run `sam build` (one command)
- Run `sam deploy` (one command)
- Done!

SAM essentially abstracts away the complexity so you can focus on your Lambda function code.

---

## Part 2: Overall Steps I Performed

Here's what I did to build the complete CI/CD pipeline:

```
STEP 1: Set up the development environment locally
        ↓
STEP 2: Create Lambda function with SAM template
        ↓
STEP 3: Create test suite with pytest
        ↓
STEP 4: Create requirements.txt files
        ↓
STEP 5: Push code to GitHub
        ↓
STEP 6: Set up Jenkins server
        ↓
STEP 7: Configure AWS credentials in Jenkins
        ↓
STEP 8: Create Jenkins pipeline (Jenkinsfile)
        ↓
STEP 9: Set up GitHub webhook for auto-triggering
        ↓
STEP 10: Test the entire pipeline end-to-end
```

---

## Part 3: Installation Steps

### Prerequisites Installation

#### 1. Install Python and pip
```bash
# macOS
brew install python3

# Ubuntu/Linux
sudo apt-get update
sudo apt-get install python3 python3-pip

# Windows
# Download from python.org

# Verify installation
python3 --version
pip3 --version
```

#### 2. Install AWS SAM CLI
```bash
# macOS
brew tap aws/tap
brew install aws-sam-cli

# Ubuntu/Linux (using pip)
pip3 install aws-sam-cli

# Windows (using chocolatey)
choco install aws-sam-cli

# Verify installation
sam --version
```

#### 3. Install AWS CLI
```bash
# macOS
brew install awscli

# Ubuntu/Linux
pip3 install awscli

# Windows
choco install awscliv2

# Verify installation
aws --version
```

#### 4. Install Docker (Required for SAM)
```bash
# macOS
brew install docker

# Ubuntu/Linux
sudo apt-get install docker.io
sudo usermod -aG docker $USER

# Windows
# Download Docker Desktop from docker.com

# Verify installation
docker --version
```

#### 5. Install Git
```bash
# macOS
brew install git

# Ubuntu/Linux
sudo apt-get install git

# Windows
choco install git

# Verify installation
git --version
```

#### 6. Install Jenkins (on your Jenkins server)
```bash
# Ubuntu/Linux
sudo apt-get update
sudo apt-get install openjdk-11-jdk
wget -q -O - https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo apt-key add -
sudo sh -c 'echo deb https://pkg.jenkins.io/debian-stable binary/ > /etc/apt/sources.list.d/jenkins.list'
sudo apt-get update
sudo apt-get install jenkins

# Start Jenkins
sudo systemctl start jenkins
sudo systemctl enable jenkins

# Access Jenkins at http://localhost:8080
```

#### 7. Install pytest (for testing)
```bash
pip3 install pytest
pip3 install pytest-cov  # optional, for coverage reports

# Verify installation
pytest --version
```

---

## Part 4: Detailed Steps I Took (Interview Walkthrough)

### STEP 1: Set Up Development Environment

**What I did:**
```bash
# Create project directory
mkdir lambda-project
cd lambda-project

# Initialize git repository
git init

# Create Python virtual environment
python3 -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Create project structure
mkdir lambda-app
mkdir tests
```

**Why:** Virtual environment isolates project dependencies so they don't conflict with system Python or other projects.

---

### STEP 2: Create Lambda Function with SAM Template

**Created `lambda-app/template.yaml`:**
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31

Globals:
  Function:
    Timeout: 20
    MemorySize: 128
    Runtime: python3.9

Resources:
  MyLambdaFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: app.lambda_handler
      CodeUri: .
      Events:
        ApiEvent:
          Type: Api
          Properties:
            RestApiId: !Ref MyApi
            Path: /hello
            Method: get
  
  MyApi:
    Type: AWS::Serverless::Api
    Properties:
      StageName: prod

Outputs:
  MyLambdaFunctionUrl:
    Description: "Lambda Function URL"
    Value: !Sub "https://${MyApi}.execute-api.${AWS::Region}.amazonaws.com/prod/hello"
```

**Created `lambda-app/app.py`:**
```python
import json

def lambda_handler(event, context):
    """
    Lambda handler function
    """
    return {
        'statusCode': 200,
        'body': json.dumps({
            'message': 'Hello World version 2'
        })
    }
```

**Why:** The template.yaml defines what AWS resources to create. The app.py is the actual Lambda function code that runs.

---

### STEP 3: Create Test Suite with pytest

**Created `tests/test_app.py`:**
```python
import sys
import json
sys.path.insert(0, '../lambda-app')

from lambda_app import app

def test_lambda_handler_returns_200():
    """Test that lambda returns status code 200"""
    event = {}
    context = None
    response = app.lambda_handler(event, context)
    assert response['statusCode'] == 200

def test_lambda_handler_returns_hello_world():
    """Test that lambda returns hello world message"""
    event = {}
    context = None
    response = app.lambda_handler(event, context)
    body = json.loads(response['body'])
    assert 'message' in body
    assert 'Hello World' in body['message']

def test_lambda_handler_response_format():
    """Test response structure"""
    event = {}
    context = None
    response = app.lambda_handler(event, context)
    assert 'statusCode' in response
    assert 'body' in response
    assert isinstance(response['body'], str)
```

**Why:** Tests validate the Lambda function works before deployment. If tests fail, the pipeline stops and prevents bad code from reaching production.

---

### STEP 4: Create Requirements Files

**Created `requirements.txt` (Production):**
```
requests==2.28.0
boto3==1.26.0
```

**Created `tests/requirements.txt` (Testing):**
```
pytest==7.2.0
pytest-cov==4.0.0
requests==2.28.0
boto3==1.26.0
```

**Why:** Separate requirements files because testing libraries (pytest) aren't needed in production Lambda code. This keeps the deployment package smaller.

---

### STEP 5: Push Code to GitHub

```bash
# Create .gitignore
echo "venv/
*.pyc
.DS_Store
.pytest_cache/
__pycache__/
build/
.aws-sam/" > .gitignore

# Add all files to git
git add .

# Commit
git commit -m "Initial Lambda project setup"

# Add remote repository
git remote add origin https://github.com/YOUR_USERNAME/lambda-project.git

# Push to GitHub
git branch -M main
git push -u origin main
```

**Why:** GitHub is the source of truth. Jenkins watches this repository and triggers builds on every push.

---

### STEP 6: Set Up Jenkins Server

```bash
# If not already installed
sudo systemctl start jenkins
sudo systemctl enable jenkins

# Access Jenkins
# Open browser to http://your-jenkins-server:8080

# First time setup:
# 1. Get initial admin password
sudo cat /var/lib/jenkins/secrets/initialAdminPassword

# 2. Install suggested plugins
# 3. Create admin user
# 4. Configure Jenkins URL
```

**Why:** Jenkins is the automation orchestrator. It watches GitHub and automatically runs our build pipeline.

---

### STEP 7: Configure AWS Credentials in Jenkins

**Navigate to Jenkins:**
1. Click **Manage Jenkins**
2. Click **Manage Credentials**
3. Click **Global** (under Stores scoped to Jenkins)
4. Click **Add Credentials**

**Create First Credential (AWS Access Key):**
- Kind: **Secret text**
- Secret: `YOUR_AWS_ACCESS_KEY_ID` (from AWS console)
- ID: `AWS-access-key`
- Click **Create**

**Create Second Credential (AWS Secret Key):**
- Kind: **Secret text**
- Secret: `YOUR_AWS_SECRET_ACCESS_KEY` (from AWS console)
- ID: `AWS-secret-key`
- Click **Create**

**Why:** These credentials let Jenkins authenticate with AWS. They're stored securely in Jenkins, not hardcoded in files.

---

### STEP 8: Create Jenkins Pipeline (Jenkinsfile)

**Created `Jenkinsfile` in root of repository:**

```groovy
pipeline {
    agent any
    
    stages {
        stage('Setup') {
            steps {
                echo '========== Installing Dependencies =========='
                sh '''
                    pip install -r tests/requirements.txt
                '''
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
                sh '''
                    sam build -t lambda-app/template.yaml
                '''
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
            echo '========== Pipeline Succeeded =========='
        }
        failure {
            echo '========== Pipeline Failed =========='
        }
    }
}
```

**Stage Breakdown:**

1. **Setup Stage**: Installs testing dependencies
2. **Test Stage**: Runs pytest to validate Lambda function
3. **Build Stage**: Runs `sam build` to package Lambda code
4. **Deploy Stage**: Runs `sam deploy` to push to AWS (only if tests pass)

**Why:** The Jenkinsfile defines the entire pipeline. Each stage is a checkpoint—if any stage fails, the pipeline stops.

---

### STEP 9: Create Jenkins Job

**In Jenkins UI:**

1. Click **New Item**
2. Enter job name: `Lambda-Pipeline`
3. Select **Pipeline**
4. Click **OK**

**Configure Pipeline:**
1. Under **Build Triggers**, check **GitHub hook trigger for GITScm polling**
2. Under **Pipeline**:
   - Definition: **Pipeline script from SCM**
   - SCM: **Git**
   - Repository URL: `https://github.com/YOUR_USERNAME/lambda-project.git`
   - Branch: `*/main`
   - Script Path: `Jenkinsfile`
3. Click **Save**

**Why:** This tells Jenkins to:
- Watch the GitHub repository
- Automatically trigger when code is pushed
- Execute the steps defined in Jenkinsfile

---

### STEP 10: Set Up GitHub Webhook

**In GitHub Repository:**

1. Go to **Settings** → **Webhooks**
2. Click **Add webhook**
3. Payload URL: `http://your-jenkins-server:8080/github-webhook/`
4. Content type: `application/json`
5. Events: Select **Just the push event**
6. Click **Add webhook**

**Why:** When you push code to GitHub, it sends a notification to Jenkins, which immediately triggers the build pipeline.

---

## Part 5: What Happens When You Push Code

```
1. Developer commits and pushes to GitHub
                    ↓
2. GitHub sends webhook to Jenkins
                    ↓
3. Jenkins receives notification
                    ↓
4. Jenkins pulls latest code from GitHub
                    ↓
5. Stage 1 - Setup: pip install -r tests/requirements.txt
                    ↓
6. Stage 2 - Test: pytest validates Lambda function
                    ↓
   If tests FAIL → Pipeline stops, developer gets notification
                    ↓
7. Stage 3 - Build: sam build packages the code
                    ↓
8. Stage 4 - Deploy: sam deploy pushes to AWS
                    ↓
9. Lambda function in AWS is updated
                    ↓
10. Jenkins returns function URL
                    ↓
11. Function is immediately live and testable
```

---

## Part 6: How SAM Build and Deploy Work

### SAM Build Process
```bash
sam build -t lambda-app/template.yaml
```

**What happens:**
1. Reads your template.yaml
2. Identifies Lambda function code location
3. Installs dependencies from requirements.txt
4. Copies code to `.aws-sam/build/` directory
5. Creates an optimized package ready for AWS

**Output:** `.aws-sam/build/` folder with everything AWS needs

### SAM Deploy Process
```bash
sam deploy -t lambda-app/template.yaml --stack-name my-lambda-stack --capabilities CAPABILITY_IAM
```

**What happens:**
1. Takes the built package from `.aws-sam/build/`
2. Authenticates to AWS using AWS credentials
3. Creates a CloudFormation stack (named `my-lambda-stack`)
4. Uploads code to S3
5. Creates/updates Lambda function
6. Creates API Gateway endpoint
7. Returns the function URL

**Output:** Live Lambda function at endpoint URL

---

## Interview Tips When Answering This

**When they ask "Walk me through the steps":**

1. Start with SAM definition: "First, I explained what SAM is—it's a framework that simplifies serverless deployment"

2. Outline the 10 steps: "I took a phased approach: local development, writing tests, setting up Jenkins, configuring credentials..."

3. Explain the flow: "When a developer pushes code, GitHub sends a webhook to Jenkins, which automatically runs..."

4. Show you understand each component:
   - Template.yaml: "This defines AWS resources"
   - app.py: "This is the actual Lambda function logic"
   - tests/requirements.txt: "Testing libraries for development only"
   - Jenkinsfile: "This orchestrates the entire pipeline"

5. Emphasize the outcome: "The result was fully automated deployment—developers just push code, and if tests pass, it's live in AWS within minutes"

6. Talk about benefits: "No more manual deployments, zero human error in deployment process, complete audit trail"

**Key phrases to use:**
- "SAM simplified the deployment process"
- "Automated testing before deployment"
- "Credentials stored securely in Jenkins"
- "GitHub webhook triggers Jenkins automatically"
- "Complete audit trail in Jenkins logs"
- "Developers can focus on code, not deployment logistics"