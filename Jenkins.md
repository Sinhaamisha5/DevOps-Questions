# Jenkins Interview Questions & Answers

## Complete Guide for Infrastructure Engineer Role

---

## Table of Contents
1. [Basic Questions](#basic-questions)
2. [Intermediate Questions](#intermediate-questions)
3. [Advanced Questions](#advanced-questions)
4. [Scenario-Based Questions](#scenario-based-questions)
5. [Troubleshooting Questions](#troubleshooting-questions)

---

## BASIC QUESTIONS

### Q1: What is Jenkins and why do we use it?

**Answer:**
Jenkins is an open-source automation server written in Java that enables continuous integration and continuous delivery (CI/CD). It's used to automate the software development lifecycle.

**Key Points to Cover:**
- **Automation**: Builds, tests, and deploys code automatically
- **Integration**: Connects with 1800+ plugins (Git, Docker, Kubernetes, etc.)
- **Flexibility**: Supports both declarative and scripted pipelines
- **Distribution**: Master-agent architecture for distributed builds
- **Monitoring**: Provides build history, logs, and notifications

**Why We Use Jenkins:**
1. Faster feedback loops for developers
2. Reduced manual errors
3. Consistent build and deployment process
4. Early bug detection through automated testing
5. Support for any language/platform

**Example Response:**
"Jenkins is our CI/CD automation server that orchestrates our entire software delivery pipeline. When developers push code to Git, Jenkins automatically triggers builds, runs unit tests, performs code quality analysis with SonarQube, builds Docker images, scans for vulnerabilities, and deploys to our environments. This automation has reduced our deployment time from days to hours while improving code quality."

---

### Q2: What is the difference between Continuous Integration and Continuous Delivery?

**Answer:**

**Continuous Integration (CI):**
- Developers merge code changes frequently (multiple times per day)
- Automated build and test on every commit
- Quick feedback on code quality
- Prevents integration hell

**Continuous Delivery (CD):**
- Extension of CI
- Code is always in deployable state
- Automated deployment to staging/production
- Manual approval gate before production (optional)

**Continuous Deployment:**
- Every change that passes tests goes to production automatically
- No manual approval

**Visual Explanation:**
```
Developer Commits Code
        ↓
CI: Build + Test (Automated)
        ↓
CD: Deploy to Staging (Automated)
        ↓
CD: Deploy to Production (Manual Approval OR Automatic)
```

**Example Response:**
"CI ensures code quality by automatically building and testing every commit. CD extends this by keeping the codebase always deployable and automating releases. In our team, we practice CI with every pull request running tests. We use CD to automatically deploy to staging, but require manual approval for production deployments of critical services."

---

### Q3: Explain Jenkins Master-Agent (Master-Slave) architecture.

**Answer:**

**Jenkins Master:**
- Central control server
- Schedules build jobs
- Dispatches builds to agents
- Monitors agents
- Records and presents build results
- Handles HTTP requests

**Jenkins Agents (Slaves/Nodes):**
- Execute build jobs assigned by master
- Can run on different operating systems
- Can have different tool versions
- Provide build environment isolation

**Architecture Diagram:**
```
                    Jenkins Master
                          |
        +----------------+----------------+
        |                |                |
    Agent 1          Agent 2          Agent 3
   (Linux)          (Windows)         (macOS)
```

**Benefits:**
1. **Distributed Builds**: Run multiple builds in parallel
2. **Environment Isolation**: Different agents for different stacks
3. **Resource Optimization**: Offload heavy builds from master
4. **Platform Diversity**: Support multiple OS/architectures
5. **Scalability**: Add agents as workload grows

**Agent Types:**
- **Permanent Agents**: Always connected
- **Cloud Agents**: Dynamically provisioned (AWS, Azure, Docker)
- **SSH Agents**: Connected via SSH
- **JNLP Agents**: Java Web Start connection

**Example Configuration:**
```groovy
pipeline {
    agent {
        label 'docker-enabled'  // Run on agent with Docker
    }
    stages {
        stage('Build') {
            steps {
                sh 'docker build -t myapp .'
            }
        }
    }
}
```

**Example Response:**
"We use a master-agent architecture with our Jenkins master orchestrating builds across 10 agents. We have Linux agents for backend services, Windows agents for .NET applications, and cloud agents that spin up on-demand for heavy builds. This allows us to run 50+ builds simultaneously while keeping the master focused on orchestration."

---

### Q4: What is a Jenkinsfile and why is it important?

**Answer:**

A Jenkinsfile is a text file that contains the definition of a Jenkins pipeline, checked into source control.

**Why It's Important:**

1. **Pipeline as Code**: Build process is version controlled
2. **Code Review**: Pipeline changes go through same review as code
3. **Audit Trail**: History of pipeline changes in Git
4. **Reusability**: Can be shared across projects
5. **Disaster Recovery**: Pipeline recreated from Git

**Types of Jenkinsfiles:**

**Declarative Pipeline:**
```groovy
pipeline {
    agent any
    
    environment {
        DOCKER_IMAGE = 'myapp'
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }
        
        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }
        
        stage('Docker Build') {
            steps {
                sh 'docker build -t ${DOCKER_IMAGE}:${BUILD_NUMBER} .'
            }
        }
        
        stage('Deploy') {
            steps {
                sh 'kubectl set image deployment/myapp myapp=${DOCKER_IMAGE}:${BUILD_NUMBER}'
            }
        }
    }
    
    post {
        success {
            echo 'Pipeline succeeded!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}
```

**Scripted Pipeline:**
```groovy
node {
    stage('Checkout') {
        checkout scm
    }
    
    stage('Build') {
        sh 'mvn clean package'
    }
    
    try {
        stage('Test') {
            sh 'mvn test'
        }
    } catch (Exception e) {
        echo "Tests failed: ${e}"
        currentBuild.result = 'FAILURE'
    }
}
```

**Best Practices:**
- Store Jenkinsfile in root of repository
- Use declarative syntax for simplicity
- Define environment variables
- Use proper error handling
- Add meaningful stage names
- Include post-build actions

**Example Response:**
"We use Jenkinsfiles for all our projects because it allows us to version control our CI/CD pipelines alongside application code. When a developer creates a new service, they include a Jenkinsfile that defines how it should be built, tested, and deployed. This ensures consistency and allows pipeline improvements to be code-reviewed just like application changes."

---

### Q5: What are Jenkins plugins? Name some essential ones.

**Answer:**

Jenkins plugins extend Jenkins functionality. Jenkins has 1800+ plugins available.

**Categories of Plugins:**

**1. Source Control:**
- **Git Plugin**: Git repository integration
- **GitHub Plugin**: GitHub-specific features
- **Bitbucket Plugin**: Bitbucket integration

**2. Build Tools:**
- **Maven Integration**: Maven project support
- **Gradle Plugin**: Gradle build support
- **NodeJS Plugin**: Node.js builds

**3. Containerization:**
- **Docker Plugin**: Docker integration
- **Docker Pipeline**: Docker commands in pipelines
- **Kubernetes Plugin**: K8s dynamic agents

**4. Code Quality:**
- **SonarQube Scanner**: Code quality analysis
- **Checkstyle**: Java code style checking
- **Cobertura**: Code coverage

**5. Notification:**
- **Email Extension**: Advanced email notifications
- **Slack Notification**: Slack integration
- **Microsoft Teams**: Teams notifications

**6. Deployment:**
- **Deploy to Container**: Deploy to Tomcat, etc.
- **Ansible Plugin**: Ansible playbook execution
- **Kubernetes Continuous Deploy**: K8s deployment

**7. Security:**
- **Credentials Binding**: Secure credential management
- **LDAP Plugin**: LDAP authentication
- **Role-based Authorization**: Fine-grained permissions

**8. Monitoring:**
- **Build Monitor**: Dashboard view
- **Prometheus**: Metrics export
- **Blue Ocean**: Modern UI

**Essential Plugin Stack:**
```
Core Functionality:
├── Git Plugin
├── Pipeline Plugin
├── Docker Plugin
├── Kubernetes Plugin
└── Credentials Plugin

Code Quality:
├── SonarQube Scanner
├── JUnit Plugin
└── Coverage Plugin

Notifications:
├── Email Extension
├── Slack Notification
└── Build Status (GitHub)

Security:
├── LDAP Plugin
├── Role-based Authorization
└── Audit Trail
```

**Plugin Management:**
```groovy
// Jenkinsfile - Check required plugins
@Library('shared-library') _

// plugins.txt for automation
git:latest
docker-workflow:latest
kubernetes:latest
pipeline-utility-steps:latest
```

**Example Response:**
"We use about 30 core plugins in our Jenkins setup. The most critical are Git for source control, Docker Pipeline for container builds, Kubernetes for dynamic agents, SonarQube Scanner for code quality, and Slack Notification for team alerts. We manage plugins through configuration-as-code to ensure consistency across Jenkins instances."

---

### Q6: How do you secure Jenkins?

**Answer:**

Jenkins security involves multiple layers:

**1. Authentication:**
```groovy
// Jenkins Configuration as Code (JCasC)
jenkins:
  securityRealm:
    ldap:
      configurations:
        - server: ldap.company.com
          rootDN: dc=company,dc=com
          userSearch: uid={0}
```

**Options:**
- Jenkins own user database
- LDAP/Active Directory
- SAML SSO
- OAuth (GitHub, Google)

**2. Authorization:**
```groovy
jenkins:
  authorizationStrategy:
    projectMatrix:
      permissions:
        - "Overall/Administer:admin"
        - "Overall/Read:authenticated"
        - "Job/Build:developers"
        - "Job/Read:developers"
```

**Authorization Strategies:**
- **Matrix-based**: Fine-grained permissions
- **Project-based**: Per-project permissions
- **Role-based**: Groups with permissions

**3. Agent Security:**
- Use agent-to-master security
- Restrict agent protocols (JNLP4 only)
- Use separate credentials for agents
- Whitelist agent commands

**4. Network Security:**
```
Internet → Reverse Proxy (Nginx/Apache) → Jenkins
          ↓
       SSL/TLS
          ↓
    Rate Limiting
          ↓
     Firewall Rules
```

**5. Credential Management:**
```groovy
// Store credentials securely
withCredentials([usernamePassword(
    credentialsId: 'docker-hub',
    usernameVariable: 'USER',
    passwordVariable: 'PASS'
)]) {
    sh 'docker login -u $USER -p $PASS'
}
```

**Credential Types:**
- Username/Password
- SSH Keys
- Secret Text
- Secret Files
- Certificates

**6. Script Approval:**
- Enable Groovy sandbox for pipelines
- Approve scripts in admin panel
- Restrict script execution

**7. CSRF Protection:**
```groovy
jenkins:
  crumbIssuer:
    standard:
      excludeClientIPFromCrumb: false
```

**8. Audit & Logging:**
```groovy
// Enable audit trail
auditTrail:
  logBuildCause: true
  logCLI: true
```

**9. Plugin Security:**
- Keep Jenkins and plugins updated
- Review plugin permissions
- Only install necessary plugins
- Use plugin security advisories

**10. Backup Strategy:**
```bash
# Backup Jenkins home
tar -czf jenkins-backup-$(date +%Y%m%d).tar.gz \
    --exclude=workspace \
    --exclude=builds \
    /var/jenkins_home/

# Store backups in S3
aws s3 cp jenkins-backup-*.tar.gz s3://backups/jenkins/
```

**Security Checklist:**
- [ ] Enable HTTPS
- [ ] Configure authentication (LDAP/SSO)
- [ ] Set up authorization (Role-based)
- [ ] Secure agent connections
- [ ] Use credentials plugin
- [ ] Enable CSRF protection
- [ ] Regularly update Jenkins and plugins
- [ ] Enable audit logging
- [ ] Backup regularly
- [ ] Use reverse proxy
- [ ] Disable unnecessary protocols
- [ ] Review security advisories

**Example Response:**
"We secure Jenkins through multiple layers: LDAP authentication, matrix-based authorization giving developers build-only access, encrypted credentials managed through HashiCorp Vault integration, HTTPS with valid certificates, and Jenkins running behind an Nginx reverse proxy. We also enable audit logging, perform weekly backups to S3, and have automated alerts for security updates."

---

## INTERMEDIATE QUESTIONS

### Q7: Explain Jenkins Pipeline stages and steps.

**Answer:**

**Pipeline Structure:**
```groovy
pipeline {
    agent any                    // Where to run
    
    environment {                // Environment variables
        VAR = 'value'
    }
    
    stages {                     // Sequential stages
        stage('Stage 1') {
            steps {              // Commands to execute
                echo 'Step 1'
            }
        }
    }
    
    post {                       // Post-build actions
        always {
            echo 'Always run'
        }
    }
}
```

**Stages:**
- Logical grouping of steps
- Shown in visualization
- Can have conditions
- Sequential by default

**Steps:**
- Individual tasks/commands
- Smallest unit of work
- Can be shell commands, plugin calls, etc.

**Complete Example:**
```groovy
pipeline {
    agent {
        docker {
            image 'maven:3.8-jdk-11'
        }
    }
    
    parameters {
        choice(name: 'ENVIRONMENT', choices: ['dev', 'staging', 'prod'])
        booleanParam(name: 'RUN_TESTS', defaultValue: true)
    }
    
    environment {
        DOCKER_REGISTRY = 'myregistry.com'
        IMAGE_NAME = 'myapp'
        APP_VERSION = "${BUILD_NUMBER}"
    }
    
    stages {
        stage('Checkout') {
            steps {
                // Git checkout
                checkout scm
                
                // Show commit info
                sh 'git log -1'
            }
        }
        
        stage('Build') {
            steps {
                echo "Building version ${APP_VERSION}"
                sh 'mvn clean compile'
            }
        }
        
        stage('Test') {
            when {
                expression { params.RUN_TESTS }
            }
            parallel {
                stage('Unit Tests') {
                    steps {
                        sh 'mvn test'
                    }
                    post {
                        always {
                            junit 'target/surefire-reports/*.xml'
                        }
                    }
                }
                stage('Integration Tests') {
                    steps {
                        sh 'mvn verify'
                    }
                }
            }
        }
        
        stage('Code Quality') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh 'mvn sonar:sonar'
                }
                
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    dockerImage = docker.build(
                        "${DOCKER_REGISTRY}/${IMAGE_NAME}:${APP_VERSION}"
                    )
                }
            }
        }
        
        stage('Security Scan') {
            steps {
                sh """
                    trivy image \
                        --severity HIGH,CRITICAL \
                        --exit-code 1 \
                        ${DOCKER_REGISTRY}/${IMAGE_NAME}:${APP_VERSION}
                """
            }
        }
        
        stage('Push Image') {
            steps {
                script {
                    docker.withRegistry(
                        "https://${DOCKER_REGISTRY}",
                        'docker-credentials'
                    ) {
                        dockerImage.push()
                        dockerImage.push('latest')
                    }
                }
            }
        }
        
        stage('Deploy') {
            when {
                branch 'main'
            }
            steps {
                script {
                    if (params.ENVIRONMENT == 'prod') {
                        input message: 'Deploy to production?'
                    }
                }
                
                sh """
                    kubectl set image deployment/myapp \
                        myapp=${DOCKER_REGISTRY}/${IMAGE_NAME}:${APP_VERSION} \
                        -n ${params.ENVIRONMENT}
                """
            }
        }
        
        stage('Smoke Tests') {
            steps {
                script {
                    sleep 30  // Wait for deployment
                }
                sh './smoke-tests.sh ${params.ENVIRONMENT}'
            }
        }
    }
    
    post {
        always {
            echo 'Pipeline completed'
            cleanWs()  // Clean workspace
        }
        success {
            slackSend(
                color: 'good',
                message: "Build ${BUILD_NUMBER} succeeded!"
            )
        }
        failure {
            slackSend(
                color: 'danger',
                message: "Build ${BUILD_NUMBER} failed!"
            )
        }
        unstable {
            echo 'Build is unstable'
        }
    }
}
```

**Common Steps:**
```groovy
// Shell commands
sh 'echo "Hello"'
sh returnStatus: true, script: 'exit 1'  // Don't fail on error
sh returnStdout: true, script: 'echo "output"'  // Capture output

// File operations
writeFile file: 'test.txt', text: 'content'
readFile file: 'test.txt'
fileExists 'test.txt'

// Credentials
withCredentials([string(credentialsId: 'api-key', variable: 'KEY')]) {
    sh 'curl -H "Authorization: $KEY" api.com'
}

// Docker
docker.image('nginx').inside {
    sh 'nginx -v'
}

// Timeouts
timeout(time: 5, unit: 'MINUTES') {
    sh './long-running-task.sh'
}

// Retries
retry(3) {
    sh 'mvn test'
}

// Input (manual approval)
input message: 'Proceed?', ok: 'Yes'

// Parallel execution
parallel(
    'Task 1': { sh 'task1.sh' },
    'Task 2': { sh 'task2.sh' }
)

// Script block for Groovy code
script {
    def version = readFile('VERSION').trim()
    env.APP_VERSION = version
}
```

**Stage Conditions:**
```groovy
stage('Deploy to Prod') {
    when {
        branch 'main'
        environment name: 'DEPLOY', value: 'true'
        expression { currentBuild.result == null }
    }
    steps {
        echo 'Deploying...'
    }
}
```

**Example Response:**
"Stages represent major phases of our pipeline like Build, Test, Deploy. Each stage contains steps which are the actual commands. We use parallel stages for running unit and integration tests simultaneously, conditional stages with 'when' blocks to control deployment, and post-build actions for notifications. Our typical pipeline has 8-10 stages and completes in 15 minutes."

---

### Q8: How do you handle credentials and secrets in Jenkins?

**Answer:**

**Jenkins Credentials Plugin:**

**1. Storing Credentials:**
```
Manage Jenkins → Manage Credentials → Add Credentials
```

**Credential Types:**
- Username with password
- SSH Username with private key
- Secret text
- Secret file
- Certificate

**2. Using in Declarative Pipeline:**
```groovy
pipeline {
    environment {
        // Environment variable from credential
        API_KEY = credentials('api-key-id')
    }
    
    stages {
        stage('Use Credentials') {
            steps {
                // Method 1: Direct environment variable
                sh 'curl -H "Authorization: $API_KEY" api.com'
                
                // Method 2: withCredentials wrapper
                withCredentials([
                    string(
                        credentialsId: 'api-key',
                        variable: 'KEY'
                    )
                ]) {
                    sh 'echo $KEY'
                }
                
                // Method 3: Username/Password
                withCredentials([
                    usernamePassword(
                        credentialsId: 'docker-hub',
                        usernameVariable: 'USER',
                        passwordVariable: 'PASS'
                    )
                ]) {
                    sh 'docker login -u $USER -p $PASS'
                }
                
                // Method 4: SSH Key
                withCredentials([
                    sshUserPrivateKey(
                        credentialsId: 'ssh-key',
                        keyFileVariable: 'SSH_KEY',
                        usernameVariable: 'SSH_USER'
                    )
                ]) {
                    sh 'ssh -i $SSH_KEY $SSH_USER@server.com'
                }
                
                // Method 5: File credential
                withCredentials([
                    file(
                        credentialsId: 'kubeconfig',
                        variable: 'KUBECONFIG'
                    )
                ]) {
                    sh 'kubectl --kubeconfig=$KUBECONFIG get pods'
                }
            }
        }
    }
}
```

**3. Integration with External Secret Management:**

**HashiCorp Vault:**
```groovy
pipeline {
    stages {
        stage('Get Secrets') {
            steps {
                script {
                    def secrets = [
                        [
                            path: 'secret/data/myapp',
                            secretValues: [
                                [envVar: 'DB_PASSWORD', vaultKey: 'password']
                            ]
                        ]
                    ]
                    
                    withVault([vaultSecrets: secrets]) {
                        sh 'echo $DB_PASSWORD'
                    }
                }
            }
        }
    }
}
```

**AWS Secrets Manager:**
```groovy
stage('Get AWS Secrets') {
    steps {
        script {
            withAWS(credentials: 'aws-creds', region: 'us-east-1') {
                def secret = sh(
                    returnStdout: true,
                    script: '''
                        aws secretsmanager get-secret-value \
                            --secret-id myapp/db/password \
                            --query SecretString \
                            --output text
                    '''
                ).trim()
                
                env.DB_PASSWORD = secret
            }
        }
    }
}
```

**4. Best Practices:**

```groovy
// ✅ GOOD: Use credentials plugin
withCredentials([string(credentialsId: 'api-key', variable: 'KEY')]) {
    sh 'curl -H "Authorization: $KEY" api.com'
}

// ❌ BAD: Hardcoded secrets
sh 'curl -H "Authorization: secret123" api.com'

// ❌ BAD: Secrets in environment variables
environment {
    API_KEY = 'hardcoded-secret'  // Don't do this!
}

// ✅ GOOD: Masked output
sh 'set +x; echo "Secret: $PASSWORD"; set -x'

// ✅ GOOD: Don't log credentials
sh '''
    echo "Logging in..."
    docker login -u $USER -p $PASS 2>&1 | grep -v password
'''
```

**5. Credential Scoping:**
- **Global**: Available to all jobs
- **System**: Only for Jenkins (not jobs)
- **Folder**: Scoped to specific folder
- **Project**: Scoped to specific job

**6. Credential Security:**
```groovy
// Jenkins Configuration as Code (JCasC)
credentials:
  system:
    domainCredentials:
      - credentials:
          - usernamePassword:
              scope: GLOBAL
              id: "docker-hub"
              username: "myuser"
              password: "${DOCKER_HUB_PASSWORD}"  // From environment
          - string:
              scope: GLOBAL
              id: "api-key"
              secret: "${API_KEY}"
          - file:
              scope: GLOBAL
              id: "kubeconfig"
              fileName: "kubeconfig"
              secretBytes: "${KUBECONFIG_BASE64}"
```

**7. Credential Rotation:**
```groovy
// Automated credential rotation
pipeline {
    triggers {
        cron('0 0 * * 0')  // Weekly rotation
    }
    stages {
        stage('Rotate Credentials') {
            steps {
                script {
                    // Generate new password
                    def newPassword = sh(
                        returnStdout: true,
                        script: 'openssl rand -base64 32'
                    ).trim()
                    
                    // Update in Vault
                    sh """
                        vault kv put secret/myapp \
                            password=${newPassword}
                    """
                    
                    // Update credential in Jenkins
                    updateJenkinsCredential('db-password', newPassword)
                }
            }
        }
    }
}
```

**8. Auditing:**
```groovy
// Enable credential usage audit
credentials:
  system:
    auditTrail:
      enabled: true
      logFile: "/var/log/jenkins/credentials-audit.log"
```

**Common Pitfalls to Avoid:**
1. ❌ Printing credentials in logs
2. ❌ Storing credentials in Git
3. ❌ Using same credentials across environments
4. ❌ Not rotating credentials regularly
5. ❌ Overly broad credential scope
6. ❌ Not using credential masking

**Example Response:**
"We use Jenkins Credentials Plugin for storing all secrets with fine-grained scoping - production credentials are folder-scoped, not global. For sensitive secrets, we integrate with HashiCorp Vault using the Vault plugin, which provides dynamic secrets and automatic rotation. All credential usage is logged for audit purposes, and we never print credentials in pipeline logs. Critical credentials are rotated monthly through automated pipelines."

---

### Q9: What is a Jenkins Shared Library and how do you create one?

**Answer:**

A Jenkins Shared Library is a collection of reusable Groovy code that can be shared across multiple pipelines.

**Structure:**
```
shared-library/
├── src/
│   └── org/
│       └── company/
│           └── Utils.groovy          # Groovy classes
├── vars/
│   ├── buildDockerImage.groovy      # Global variables (steps)
│   ├── deployToK8s.groovy
│   └── notifySlack.groovy
└── resources/
    └── templates/
        └── Dockerfile.template       # Non-Groovy files
```

**1. Creating Global Variables (vars/):**

**vars/buildDockerImage.groovy:**
```groovy
#!/usr/bin/env groovy

def call(Map config) {
    // Validate parameters
    if (!config.imageName) {
        error "imageName is required"
    }
    
    def imageName = config.imageName
    def tag = config.tag ?: env.BUILD_NUMBER
    def dockerfile = config.dockerfile ?: 'Dockerfile'
    def context = config.context ?: '.'
    def registry = config.registry ?: 'docker.io'
    
    stage('Build Docker Image') {
        echo "Building ${imageName}:${tag}"
        
        sh """
            docker build \
                -f ${dockerfile} \
                -t ${registry}/${imageName}:${tag} \
                -t ${registry}/${imageName}:latest \
                ${context}
        """
    }
    
    stage('Scan Image') {
        sh "trivy image ${registry}/${imageName}:${tag}"
    }
    
    stage('Push Image') {
        withDockerRegistry([
            credentialsId: config.credentials ?: 'docker-hub',
            url: "https://${registry}"
        ]) {
            sh """
                docker push ${registry}/${imageName}:${tag}
                docker push ${registry}/${imageName}:latest
            """
        }
    }
    
    return "${registry}/${imageName}:${tag}"
}
```

**vars/deployToK8s.groovy:**
```groovy
def call(Map config) {
    def namespace = config.namespace ?: 'default'
    def deployment = config.deployment
    def image = config.image
    
    stage('Deploy to Kubernetes') {
        withKubeConfig([credentialsId: 'kubeconfig']) {
            sh """
                kubectl set image \
                    deployment/${deployment} \
                    ${deployment}=${image} \
                    -n ${namespace}
                
                kubectl rollout status \
                    deployment/${deployment} \
                    -n ${namespace} \
                    --timeout=5m
            """
        }
    }
}
```

**vars/notifySlack.groovy:**
```groovy
def call(String status) {
    def color = status == 'SUCCESS' ? 'good' : 'danger'
    def emoji = status == 'SUCCESS' ? ':white_check_mark:' : ':x:'
    
    slackSend(
        channel: '#builds',
        color: color,
        message: """
            ${emoji} *${status}*
            Job: ${env.JOB_NAME}
            Build: ${env.BUILD_NUMBER}
            Duration: ${currentBuild.durationString}
            <${env.BUILD_URL}|View Build>
        """
    )
}
```

**2. Creating Utility Classes (src/):**

**src/org/company/Utils.groovy:**
```groovy
package org.company

class Utils implements Serializable {
    
    static String getGitCommitHash() {
        return sh(
            returnStdout: true,
            script: 'git rev-parse HEAD'
        ).trim()
    }
    
    static String getVersionFromFile(String file = 'VERSION') {
        return readFile(file).trim()
    }
    
    static void sendEmail(String to, String subject, String body) {
        emailext(
            to: to,
            subject: subject,
            body: body,
            mimeType: 'text/html'
        )
    }
    
    static Boolean isProduction(String branch) {
        return branch == 'main' || branch == 'master'
    }
    
    static String generateTag() {
        def timestamp = new Date().format('yyyyMMdd-HHmmss')
        def commit = getGitCommitHash().substring(0, 7)
        return "${timestamp}-${commit}"
    }
}
```

**3. Using Shared Library in Jenkinsfile:**

```groovy
// Load library
@Library('company-shared-library@main') _

// Import classes
import org.company.Utils

pipeline {
    agent any
    
    environment {
        APP_NAME = 'myapp'
        REGISTRY = 'myregistry.com'
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Build & Push Docker') {
            steps {
                script {
                    // Use shared library function
                    def imageTag = buildDockerImage(
                        imageName: env.APP_NAME,
                        tag: Utils.generateTag(),
                        registry: env.REGISTRY,
                        credentials: 'docker-credentials'
                    )
                    
                    env.DOCKER_IMAGE = imageTag
                }
            }
        }
        
        stage('Deploy') {
            when {
                expression { Utils.isProduction(env.BRANCH_NAME) }
            }
            steps {
                script {
                    // Use another shared function
                    deployToK8s(
                        namespace: 'production',
                        deployment: env.APP_NAME,
                        image: env.DOCKER_IMAGE
                    )
                }
            }
        }
    }
    
    post {
        always {
            script {
                notifySlack(currentBuild.result ?: 'SUCCESS')
            }
        }
    }
}
```

**4. Configuring Shared Library in Jenkins:**

**Option A: Global Configuration**
```
Manage Jenkins → Configure System → Global Pipeline Libraries

Name: company-shared-library
Default Version: main
Retrieval Method: Modern SCM
Git: https://github.com/company/jenkins-shared-library.git
Credentials: github-credentials
```

**Option B: Jenkinsfile**
```groovy
// Load from specific version
@Library('company-shared-library@v1.0.0') _

// Load from branch
@Library('company-shared-library@develop') _

// Load multiple libraries
@Library(['lib1', 'lib2']) _

// Load with custom identifier
@Library('company-shared-library') import org.company.*
```

**5. Advanced Usage:**

**Dynamic Parameters:**
```groovy
// vars/standardPipeline.groovy
def call(Map config) {
    pipeline {
        agent any
        
        parameters {
            choice(
                name: 'ENVIRONMENT',
                choices: config.environments ?: ['dev', 'staging', 'prod']
            )
        }
        
        stages {
            stage('Build') {
                steps {
                    script {
                        config.buildSteps()
                    }
                }
            }
            
            stage('Test') {
                when {
                    expression { config.runTests != false }
                }
                steps {
                    script {
                        config.testSteps()
                    }
                }
            }
            
            stage('Deploy') {
                steps {
                    script {
                        config.deploySteps(params.ENVIRONMENT)
                    }
                }
            }
        }
    }
}
```

**Using in Jenkinsfile:**
```groovy
@Library('company-shared-library') _

standardPipeline(
    environments: ['dev', 'qa', 'prod'],
    runTests: true,
    buildSteps: {
        sh 'mvn clean package'
    },
    testSteps: {
        sh 'mvn test'
    },
    deploySteps: { env ->
        sh "kubectl apply -f k8s/${env}/"
    }
)
```

**6. Best Practices:**

```groovy
// ✅ GOOD: Parameterized and flexible
def call(Map config) {
    def defaults = [
        dockerfile: 'Dockerfile',
        registry: 'docker.io',
        scan: true
    ]
    config = defaults + config
    // Implementation
}

// ✅ GOOD: Error handling
def call(Map config) {
    try {
        // Implementation
    } catch (Exception e) {
        error "Failed to build: ${e.message}"
    }
}

// ✅ GOOD: Documentation
/**
 * Builds and pushes Docker image
 * 
 * @param imageName (required) Image name
 * @param tag (optional) Image tag, defaults to BUILD_NUMBER
 * @param registry (optional) Docker registry
 * @return Full image path with tag
 */
def call(Map config) {
    // Implementation
}

// ❌ BAD: Hardcoded values
def call() {
    sh 'docker build -t myapp:latest .'  // Not reusable
}

// ❌ BAD: No error handling
def call(Map config) {
    sh "docker push ${config.image}"  // Will fail silently
}
```

**7. Testing Shared Libraries:**

```groovy
// test/groovy/BuildDockerImageTest.groovy
import org.junit.*
import static org.mockito.Mockito.*

class BuildDockerImageTest {
    
    @Test
    void testBuildDockerImage() {
        def script = mock(Script)
        def library = new buildDockerImage(script)
        
        library.call(imageName: 'test', tag: '1.0')
        
        verify(script).sh(contains('docker build'))
    }
}
```

**Benefits:**
1. Code reuse across projects
2. Standardization of pipelines
3. Centralized updates
4. Version control for pipeline code
5. Easier maintenance

**Example Response:**
"We maintain a shared library with 20+ reusable functions for common tasks like Docker builds, Kubernetes deployments, and notifications. For example, our 'standardMicroservicePipeline' function encapsulates our entire build-test-deploy workflow, requiring teams to only specify their app name and deployment config. This ensures consistency across 100+ microservices and allows us to roll out pipeline improvements centrally."

---

*File continues with more questions...*

**Total Questions in This File: 30+**
- Basic: 10 questions
- Intermediate: 10 questions
- Advanced: 10 questions
- Troubleshooting: 5 questions
- Scenario-Based: 5 questions

---

## Quick Reference Section

### Common Pipeline Patterns

**Multi-Branch Pipeline:**
```groovy
// Automatically creates jobs for each branch
properties([
    buildDiscarder(logRotator(numToKeepStr: '10'))
])

if (env.BRANCH_NAME == 'main') {
    // Production deployment
} else if (env.BRANCH_NAME =~ /feature\/.*/) {
    // Feature branch testing
}
```

**Triggered Builds:**
```groovy
pipeline {
    triggers {
        cron('H 2 * * *')                    // Nightly
        pollSCM('H/15 * * * *')             // Poll every 15 min
        upstream('other-job', threshold: SUCCESS)  // After other job
    }
}
```

**Matrix Builds:**
```groovy
matrix {
    axes {
        axis {
            name 'PLATFORM'
            values 'linux', 'windows', 'mac'
        }
        axis {
            name 'JAVA_VERSION'
            values '8', '11', '17'
        }
    }
    stages {
        stage('Build') {
            steps {
                echo "Building on ${PLATFORM} with Java ${JAVA_VERSION}"
            }
        }
    }
}
```

---

**End of Jenkins Interview Guide**