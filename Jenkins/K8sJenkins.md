# Kubernetes & Jenkins CI/CD Pipeline: Complete Guide

## Table of Contents
1. Kubernetes Fundamentals
2. Key Kubernetes Concepts
3. kubectl Commands
4. kubeconfig File Structure
5. Deployment Workflow
6. k6 Load Testing
7. Jenkins + Kubernetes Integration
8. Complete Jenkinsfile
9. Troubleshooting Guide
10. Interview Case Study

---

## Part 1: Kubernetes Fundamentals

### What is Kubernetes?

Kubernetes (K8s) is an open-source container orchestration platform that automates deployment, scaling, and management of containerized applications.

**Key insight:** Kubernetes isn't just about running containers—it's about managing them at scale with self-healing, auto-scaling, and rolling updates.

### Pods vs Containers

**Container (Docker):**
- Single application instance
- Lightweight, standalone unit

**Pod (Kubernetes):**
- Wrapper around one or more containers
- Smallest deployable unit in Kubernetes
- Containers in a pod share network namespace (same IP address)
- You never deploy containers directly; you deploy pods

**Simple example:**
```
Container: Single Flask app
Pod: Flask app container + logging sidecar container
Deployment: 3 replicas of that pod
```

---

## Part 2: Key Kubernetes Concepts

### 1. Pods

A pod is the smallest Kubernetes resource. It contains one or more containers that need to work together.

**When to use single container vs multiple containers in a pod:**
- **Single container:** Most common case (your Flask app)
- **Multiple containers:** When you need a sidecar (logging, monitoring, proxying)

### 2. Deployments

Deployment is an abstraction layer above pods that provides:
- **Replica management:** Maintain a specific number of pod instances
- **Self-healing:** If a pod crashes, deployment recreates it
- **Rolling updates:** Update to new version gradually without downtime
- **Rollback:** Revert to previous version if issues occur
- **Scaling:** Easily increase or decrease number of replicas

**Why use Deployments instead of Pods directly?**
- Pods are ephemeral (can be deleted)
- Deployments ensure your desired state is always maintained
- You want these intelligence features for production

### 3. Services

Service is an abstraction that exposes your pods to the outside world.

**Types of services:**
- **ClusterIP:** Internal only (default)
- **NodePort:** Accessible on a port on each node
- **LoadBalancer:** Exposes with external load balancer (what we use)

**Why services?**
- Pods are ephemeral; they come and go
- Service provides stable IP/DNS for accessing pods
- Service load balances traffic across all pod replicas
- Users/applications talk to service, not directly to pods

### 4. Contexts

Context binds together:
- A Kubernetes cluster
- A user
- A namespace (optional)

Contexts allow you to switch between different clusters/environments.

---

## Part 3: YAML Configuration Files

### Pod Configuration Example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  labels:
    app: flask
    environment: production
spec:
  containers:
    - name: flask-app
      image: myuser/flask-app:latest
      ports:
        - containerPort: 5000
      resources:
        limits:
          memory: "256Mi"
          cpu: "500m"
```

**Breakdown:**
- `kind: Pod` - Type of Kubernetes resource
- `metadata` - Name and labels (tags for organization)
- `spec` - Configuration for the pod
- `containers` - List of containers in this pod
- `image` - Docker image to use
- `resources` - CPU/memory limits

### Deployment Configuration Example

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flask-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: flask
  template:
    metadata:
      labels:
        app: flask
    spec:
      containers:
        - name: flask-app
          image: sanjeev-thiyagarajan/jenkins-flask-app:abc123de
          ports:
            - containerPort: 5000
          resources:
            limits:
              memory: "256Mi"
              cpu: "500m"
```

**Key points:**
- `replicas: 3` - Keep 3 pod instances running at all times
- `selector` - Which pods does this deployment manage?
- `template` - Pod configuration for deployment to create

### Service Configuration Example

```yaml
apiVersion: v1
kind: Service
metadata:
  name: flask-app-service
spec:
  type: LoadBalancer
  selector:
    app: flask
  ports:
    - protocol: TCP
      port: 80
      targetPort: 5000
```

**Breakdown:**
- `type: LoadBalancer` - Expose externally with load balancer
- `selector` - Which pods to route traffic to
- `port: 80` - External port users connect to
- `targetPort: 5000` - Internal port your app listens on

---

## Part 4: kubectl Commands

### Installation

```bash
# macOS
brew install kubectl

# Ubuntu/Linux
sudo apt-get install kubectl

# Windows
choco install kubernetes-cli

# Verify
kubectl version --client
```

### Essential Commands

**Cluster Information:**
```bash
# Get current context
kubectl config current-context

# List all contexts
kubectl config get-contexts

# Switch to different context (cluster)
kubectl config use-context prod-cluster

# Get nodes in cluster
kubectl get nodes
```

**Deploying Resources:**
```bash
# Apply configuration from file
kubectl apply -f deployment.yaml

# Apply all configs in directory
kubectl apply -f k8s/

# Delete resource
kubectl delete -f deployment.yaml
```

**Getting Information:**
```bash
# List deployments
kubectl get deployment

# List pods
kubectl get pod

# List services
kubectl get service
# Short form: kubectl get svc

# Get detailed info
kubectl describe pod my-pod
```

**Updating Deployments:**
```bash
# Update image for deployment
kubectl set image deployment/flask-app \
  flask-app=sanjeev-thiyagarajan/jenkins-flask-app:v2

# Scale replicas
kubectl scale deployment flask-app --replicas=5

# Rollout status
kubectl rollout status deployment/flask-app

# Rollback to previous version
kubectl rollout undo deployment/flask-app
```

**Debugging:**
```bash
# View pod logs
kubectl logs pod-name

# Get pod status
kubectl get pod pod-name -o yaml

# Execute command in pod
kubectl exec -it pod-name -- /bin/bash
```

**JSONPath Extraction:**
```bash
# Get specific field from resource
kubectl get svc flask-app-service \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'

# Use in environment variable
SERVICE_URL=$(kubectl get svc flask-app-service \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
```

---

## Part 5: kubeconfig File Structure

### Location

Default location: `~/.kube/config`

Can also be specified via:
- Environment variable: `export KUBECONFIG=/path/to/config`
- Command flag: `kubectl --kubeconfig=/path/to/config`

### File Structure

```yaml
apiVersion: v1
kind: Config

# Different clusters
clusters:
  - name: prod-cluster
    cluster:
      server: https://prod-cluster.eks.amazonaws.com
      certificate-authority-data: LS0tLS1CRU... (base64)
  
  - name: staging-cluster
    cluster:
      server: https://staging-cluster.eks.amazonaws.com
      certificate-authority-data: LS0tLS1CRU... (base64)

# Users (credentials)
users:
  - name: user-prod
    user:
      client-certificate-data: LS0tLS1CRU... (base64)
      client-key-data: LS0tLS1CRU... (base64)
  
  - name: user-staging
    user:
      client-certificate-data: LS0tLS1CRU... (base64)
      client-key-data: LS0tLS1CRU... (base64)

# Contexts (binding of cluster + user)
contexts:
  - name: prod
    context:
      cluster: prod-cluster
      user: user-prod
  
  - name: staging
    context:
      cluster: staging-cluster
      user: user-staging

# Current active context
current-context: staging
```

**Three sections explained:**

1. **Clusters** - Where your Kubernetes clusters are located (URLs)
2. **Users** - Credentials to authenticate with clusters
3. **Contexts** - Binds cluster + user together (this is what you switch between)

### Switching Contexts

```bash
# View all contexts
kubectl config get-contexts

# Switch to production
kubectl config use-context prod

# Switch to staging
kubectl config use-context staging

# Verify current context
kubectl config current-context
```

---

## Part 6: k6 Load Testing

### What is k6?

k6 is a load testing tool that simulates user traffic to test application performance under load.

**Why use it?**
- Test before deploying to production
- Ensure application meets performance requirements
- Verify auto-scaling works properly
- Catch performance regressions

### Installation

```bash
# macOS
brew install k6

# Ubuntu/Linux
sudo apt-get install k6

# Windows
choco install k6

# Verify
k6 version
```

### Acceptance Test Script Example

**File: `acceptance-test.js`**

```javascript
import http from 'k6/http';
import { check } from 'k6';

export let options = {
  vus: 10,                    // 10 virtual users
  duration: '10s',            // Test duration: 10 seconds
  thresholds: {
    http_req_duration: ['p(90)<600'], // 90% of requests < 600ms
  },
};

export default function() {
  let response = http.get(`http://${__ENV.SERVICE_URL}:5000/`);
  
  check(response, {
    'status is 200': (r) => r.status === 200,
    'response time < 600ms': (r) => r.timings.duration < 600,
  });
}
```

**Configuration breakdown:**
- `vus: 10` - Simulate 10 concurrent users
- `duration: '10s'` - Run test for 10 seconds
- `thresholds` - Define pass/fail criteria
- `http_req_duration: ['p(90)<600']` - 90th percentile response time must be < 600ms

### Running k6 Test

```bash
# Basic run
k6 run acceptance-test.js

# With environment variable
k6 run -e SERVICE_URL=myapp.example.com acceptance-test.js

# With multiple environment variables
k6 run \
  -e SERVICE_URL=myapp.example.com \
  -e PORT=5000 \
  acceptance-test.js
```

### k6 Output

```
running (10.1s), 0/10 VUs, 50 complete and 0 interrupted transactions
default ✓ [======================================] 10 VUs  10s

     checks...................: 100% ✓ 5000  ✗ 0
     data_received............: 250 kB ✓
     data_sent................: 50 kB ✓
     http_req_blocked.........: avg=1.2ms    min=0.5ms    med=1ms      max=5ms     p(90)=2.1ms   p(95)=2.5ms
     http_req_connecting......: avg=0.5ms    min=0ms      med=0ms      max=2ms     p(90)=1ms     p(95)=1.2ms
     http_req_duration........: avg=45.3ms   min=12ms     med=42ms     max=128ms   p(90)=71.17ms p(95)=85ms
     http_req_receiving.......: avg=2.1ms    min=0.5ms    med=2ms      max=8ms     p(90)=3ms     p(95)=4ms
     http_req_sending.........: avg=0.8ms    min=0.2ms    med=0.8ms    max=3ms     p(90)=1ms     p(95)=1.2ms
     http_req_tls_connecting..: avg=0ms      min=0ms      med=0ms      max=0ms     p(90)=0ms     p(95)=0ms
     http_req_waiting.........: avg=42.4ms   min=10ms     med=40ms     max=125ms   p(90)=69ms    p(95)=82ms
     http_requests............: 5000 in 10.1s, 495/s
```

**Key metrics:**
- `checks` - Pass/fail rate
- `http_req_duration` - Response times (avg, min, max, percentiles)
- `p(90)=71.17ms` - **90th percentile response time** (what we set threshold for)

---

## Part 7: Jenkins Setup for Kubernetes

### Step 1: Install kubectl on Jenkins Server

```bash
# SSH into Jenkins server
ssh jenkins-server

# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Verify
kubectl version --client
```

### Step 2: Create Jenkins Credentials

**In Jenkins UI:**

1. Click **Manage Jenkins** → **Manage Credentials**
2. Click **Global** (under Stores scoped to Jenkins)
3. Click **Add Credentials**

**Add Credential 1 - kubeconfig:**
- Kind: **Secret file**
- File: Select your `~/.kube/config` file
- ID: `kubeconfig-credentials`
- Click **Create**

**Add Credential 2 - Docker Hub (if not already done):**
- Kind: **Username with password**
- Username: `docker-hub-username`
- Password: `docker-hub-password`
- ID: `docker-hub-credentials`
- Click **Create**

**Add Credential 3 - AWS (if using EKS):**
- Kind: **Secret text**
- Secret: `AWS_ACCESS_KEY_ID`
- ID: `aws-access-key`
- Click **Create**

- Kind: **Secret text**
- Secret: `AWS_SECRET_ACCESS_KEY`
- ID: `aws-secret-key`
- Click **Create**

### Step 3: Verify kubectl Access

In Jenkins, create a test pipeline:

```groovy
pipeline {
    agent any
    
    stages {
        stage('Test kubectl') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig-credentials', variable: 'KUBECONFIG_FILE')]) {
                    sh '''
                        export KUBECONFIG=$KUBECONFIG_FILE
                        kubectl get nodes
                        kubectl config current-context
                    '''
                }
            }
        }
    }
}
```

---

## Part 8: Complete Jenkinsfile with Kubernetes

```groovy
pipeline {
    agent any
    
    environment {
        // Docker image configuration
        DOCKER_REGISTRY = "sanjeev-thiyagarajan"
        IMAGE_NAME = "${DOCKER_REGISTRY}/jenkins-flask-app"
        IMAGE_TAG = "${IMAGE_NAME}:${GIT_COMMIT.take(8)}"
        
        // Kubernetes contexts
        STAGING_CONTEXT = "staging"
        PROD_CONTEXT = "prod-us-east-1.eksctl.io"
        
        // Kubernetes deployment info
        DEPLOYMENT_NAME = "flask-app"
        CONTAINER_NAME = "flask-app"
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo '========== Checking out code =========='
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
                echo '========== Running unit tests =========='
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
                
                echo '========== Verifying image =========='
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
                    usernameVariable: 'DOCKER_USERNAME',
                    passwordVariable: 'DOCKER_PASSWORD'
                )]) {
                    sh '''
                        echo $DOCKER_PASSWORD | docker login -u $DOCKER_USERNAME --password-stdin
                        echo "Login successful"
                    '''
                }
                
                echo '========== Pushing image to Docker Hub =========='
                sh '''
                    docker push ${IMAGE_TAG}
                    echo "Image pushed: ${IMAGE_TAG}"
                '''
            }
        }
        
        stage('Deploy to Staging') {
            steps {
                echo '========== Deploying to staging cluster =========='
                withCredentials([
                    file(credentialsId: 'kubeconfig-credentials', variable: 'KUBECONFIG_FILE'),
                    string(credentialsId: 'aws-access-key', variable: 'AWS_ACCESS_KEY_ID'),
                    string(credentialsId: 'aws-secret-key', variable: 'AWS_SECRET_ACCESS_KEY')
                ]) {
                    sh '''
                        export KUBECONFIG=$KUBECONFIG_FILE
                        export AWS_DEFAULT_REGION=us-east-1
                        
                        echo "========== Fixing kubeconfig permissions =========="
                        chmod 600 $KUBECONFIG_FILE
                        
                        echo "========== Switching to staging context =========="
                        kubectl config use-context ${STAGING_CONTEXT}
                        kubectl config current-context
                        
                        echo "========== Applying Kubernetes configs =========="
                        kubectl apply -f k8s/
                        
                        echo "========== Updating deployment image =========="
                        kubectl set image deployment/${DEPLOYMENT_NAME} \
                            ${CONTAINER_NAME}=${IMAGE_TAG} \
                            --record
                        
                        echo "========== Waiting for rollout =========="
                        kubectl rollout status deployment/${DEPLOYMENT_NAME}
                    '''
                }
            }
        }
        
        stage('Acceptance Tests - Staging') {
            steps {
                echo '========== Running k6 acceptance tests =========='
                withCredentials([file(credentialsId: 'kubeconfig-credentials', variable: 'KUBECONFIG_FILE')]) {
                    sh '''
                        export KUBECONFIG=$KUBECONFIG_FILE
                        
                        echo "========== Getting staging service URL =========="
                        SERVICE_URL=$(kubectl get svc flask-app-service \
                            -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
                        
                        echo "Service URL: $SERVICE_URL"
                        
                        echo "========== Running k6 load test =========="
                        k6 run -e SERVICE_URL=$SERVICE_URL acceptance-test.js
                    '''
                }
            }
        }
        
        stage('Deploy to Production') {
            when {
                branch 'main'
            }
            steps {
                echo '========== Deploying to production cluster =========='
                withCredentials([
                    file(credentialsId: 'kubeconfig-credentials', variable: 'KUBECONFIG_FILE'),
                    string(credentialsId: 'aws-access-key', variable: 'AWS_ACCESS_KEY_ID'),
                    string(credentialsId: 'aws-secret-key', variable: 'AWS_SECRET_ACCESS_KEY')
                ]) {
                    sh '''
                        export KUBECONFIG=$KUBECONFIG_FILE
                        export AWS_DEFAULT_REGION=us-east-1
                        
                        echo "========== Fixing kubeconfig permissions =========="
                        chmod 600 $KUBECONFIG_FILE
                        
                        echo "========== Switching to production context =========="
                        kubectl config use-context ${PROD_CONTEXT}
                        kubectl config current-context
                        
                        echo "========== Updating deployment image =========="
                        kubectl set image deployment/${DEPLOYMENT_NAME} \
                            ${CONTAINER_NAME}=${IMAGE_TAG} \
                            --record
                        
                        echo "========== Waiting for rollout =========="
                        kubectl rollout status deployment/${DEPLOYMENT_NAME}
                        
                        echo "========== Deployment complete =========="
                    '''
                }
            }
        }
    }
    
    post {
        always {
            echo '========== Cleaning up Docker login =========='
            sh 'docker logout || true'
        }
        success {
            echo '========== Pipeline completed successfully =========='
        }
        failure {
            echo '========== Pipeline failed =========='
        }
    }
}
```

---

## Part 9: Common Issues and Troubleshooting

### Issue 1: Permission Denied on kubeconfig

**Error:**
```
kubectl config use-context: permission denied
```

**Cause:** Jenkins doesn't have write permissions on kubeconfig file.

**Solution:**
```bash
chmod 600 $KUBECONFIG_FILE
```

**Why:** `kubectl config use-context` needs to update the file to mark current context.

### Issue 2: kubectl Command Not Found

**Error:**
```
command not found: kubectl
```

**Cause:** kubectl not installed on Jenkins server.

**Solution:**
```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

### Issue 3: Cannot Connect to Cluster

**Error:**
```
Unable to connect to the server: dial tcp: lookup on <IP>: no such host
```

**Cause:** kubeconfig file has incorrect cluster information or network issue.

**Solution:**
```bash
# Verify kubeconfig
kubectl config view

# Test connectivity
kubectl get nodes

# Check AWS credentials (if using EKS)
aws sts get-caller-identity
```

### Issue 4: Image Pull Backoff

**Error:**
```
Failed to pull image "myrepo/myimage:tag": rpc error
```

**Cause:** Docker image doesn't exist at tag, or credentials incorrect.

**Solution:**
1. Verify image was pushed: `docker push myrepo/myimage:tag`
2. Verify image name matches deployment: `kubectl get deployment -o yaml`
3. If private registry, verify image pull secrets configured

### Issue 5: Service URL Not Available

**Error:**
```
EXTERNAL-IP is <pending>
```

**Cause:** Load balancer not provisioned yet.

**Solution:**
```bash
# Wait a few minutes
kubectl get svc flask-app-service --watch

# Get URL once ready
kubectl get svc flask-app-service \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

---

## Part 10: Interview Case Study

### "How I Implemented a Multi-Cluster CI/CD Pipeline with Kubernetes and Jenkins"

#### The Situation

I was working at a company that was transitioning from manual Docker deployments to a Kubernetes-based infrastructure on AWS EKS. The challenge was that we had:

- Multiple Kubernetes clusters (staging and production)
- Manual deployment processes that were error-prone
- No automated testing between code commit and production
- No way to quickly rollback if issues occurred
- Developers spending time on deployment tasks instead of development

The goal was to build a fully automated CI/CD pipeline that would safely deploy code to staging first, run acceptance tests, and then deploy to production—all triggered by a simple git push.

#### The Approach I Took

**Step 1: Understanding the Requirements**

First, I had to understand the Kubernetes architecture:
- What are pods vs deployments vs services?
- How do contexts work for switching between clusters?
- How does kubectl authenticate with different clusters?

I spent time learning that pods are Kubernetes abstractions around containers, deployments manage multiple pod replicas with self-healing, and services expose pods to the outside world. Contexts allow kubectl to switch between different clusters.

**Step 2: Setting Up Infrastructure**

I created two EKS clusters in AWS:
- Staging cluster for testing
- Production cluster for live traffic

Each cluster had a kubeconfig file that contained cluster information, user credentials, and contexts. I realized that Jenkins would need secure access to this kubeconfig file without exposing sensitive credentials.

**Step 3: Creating Kubernetes Manifests**

I created YAML files for:
- **Deployment:** Specifying 3 replicas of our Flask application, using Docker images with Git SHA tags
- **Service:** LoadBalancer service to expose the application with an external IP

I tagged Docker images with Git commit SHA so each deployment could be traced back to a specific commit.

**Step 4: Building the Jenkins Pipeline**

The pipeline had these stages:

1. **Checkout:** Pull code from GitHub
2. **Test:** Run unit tests with pytest
3. **Build:** Create Docker image
4. **Push:** Upload to Docker Hub
5. **Deploy to Staging:** Use kubectl to switch context and deploy
6. **Acceptance Tests:** Run k6 load tests to verify performance
7. **Deploy to Production:** Only if on main branch, switch to prod context and deploy

**Step 5: The Critical Issue - kubeconfig Permissions**

During the first run, the pipeline failed with: "permission denied" when trying to run `kubectl config use-context`.

I initially thought it was a credential issue, but after investigating the logs with `ls -la`, I discovered that the kubeconfig file only had read permissions. The `kubectl config use-context` command needs to write to the file to update the current context pointer.

**Solution:** I added `chmod 600 $KUBECONFIG_FILE` before using kubectl. This gave Jenkins write permissions.

**Step 6: Acceptance Testing**

After deploying to staging, I integrated k6 load testing to ensure performance requirements were met:
- 10 virtual users
- 10-second duration
- Threshold: 90% of requests must complete in under 600ms

The k6 script extracted the service URL using kubectl JSONPath and tested against the actual staging deployment. This caught performance regressions before production deployment.

**Step 7: Multi-Cluster Management**

For production deployment, I:
- Used `kubectl config use-context` to switch from staging to production context
- Used `kubectl set image` to update the deployment image
- Used `kubectl rollout status` to wait for the new pods to be ready

This ensured smooth deployments across multiple clusters.

#### The Implementation

**Key technical decisions:**

1. **Git SHA tagging:** Every Docker image tagged with commit SHA for complete traceability
2. **Test before deploy:** Unit tests run before Docker build; acceptance tests run before prod
3. **Secure credentials:** Kubeconfig and Docker credentials stored in Jenkins credentials manager
4. **Staging first:** Always deploy to staging, run acceptance tests, then promote to prod
5. **kubectl context switching:** Automate cluster selection instead of manual switching

#### The Results

The new pipeline delivered significant improvements:

- **Deployment time:** 2-3 minutes end-to-end (previously 20-30 minutes manual)
- **Safety:** Broken code never reaches production because tests block deployment
- **Visibility:** Complete audit trail showing which commit deployed when
- **Reliability:** Kubernetes self-healing means no manual pod restarts
- **Testing:** Every deployment validated with load tests before going live
- **Scalability:** Easy to add new clusters or change number of replicas

#### The Business Impact

- Developers deploy 5-10 times per day instead of once per week
- Zero production incidents from bad deployments (caught in staging)
- Faster feature releases to customers
- Ops team spends less time on manual deployments
- Easy rollback if issues occur (just revert commit and push)

#### Key Learnings

1. **Understand your infrastructure:** Knowing the difference between pods, deployments, and services helped me design a robust solution
2. **Debugging is critical:** The permission issue taught me to check logs carefully and verify file permissions
3. **Automation at every step:** From building Docker images to switching clusters—everything should be automated
4. **Testing between environments:** Staging tests catch issues before production
5. **Git as source of truth:** Every change tracked, every deployment traceable to a commit

#### What I'd Do Differently

If I were to do this again, I would:
1. Use Helm charts for more complex deployments
2. Implement GitOps practices (declarative state)
3. Add more comprehensive monitoring and alerting
4. Use separate service accounts for different pipeline stages
5. Implement canary deployments for gradual rollouts

#### Interview Tips for This Story

- **Start with the problem:** "We had manual, error-prone deployments..."
- **Show your learning:** "I had to understand pods vs deployments..."
- **Highlight the debugging:** "When it failed, I checked permissions..."
- **Emphasize the results:** "Deployment time went from 30 minutes to 3 minutes..."
- **Be honest about challenges:** "The kubeconfig permission issue took me a while to debug..."
- **Show architectural thinking:** "I chose to deploy to staging first for safety..."
- **Mention traceability:** "Every image tagged with Git SHA for complete auditability..."

#### Likely Follow-up Questions

**Q: How would you handle a failed deployment?**
A: "Kubernetes deployments have built-in rollback. I'd use `kubectl rollout undo` to revert to the previous version. Or if it's a code issue, the developer would fix the code, push to Git, and the pipeline automatically re-runs."

**Q: How do you manage secrets in Kubernetes?**
A: "Kubernetes has Secrets for sensitive data, though I'd recommend using a secrets manager like AWS Secrets Manager or HashiCorp Vault. The kubeconfig file and Docker credentials are stored in Jenkins credentials manager, not in code."

**Q: What about blue-green or canary deployments?**
A: "The current pipeline does rolling updates. For blue-green, I'd create two separate deployments and switch traffic. For canary, I'd use a tool like Flagger or Argo Rollouts to gradually shift traffic to new version while monitoring metrics."

**Q: How do you monitor the pipeline?**
A: "Jenkins logs every stage. For production monitoring, I'd use CloudWatch or Prometheus/Grafana to monitor pod health, resource usage, and application metrics. Any issues trigger alerts to the team."

---

## Summary of Key Takeaways

1. **Pods are Kubernetes abstractions** - Deploy pods (with deployments), not containers
2. **Deployments provide intelligence** - Self-healing, scaling, rolling updates, rollback
3. **Services expose pods** - Users talk to services, not directly to pods
4. **Contexts switch clusters** - One kubeconfig file manages multiple clusters
5. **Git SHA tags images** - Complete traceability from commit to container
6. **Test before deploy** - Unit tests → Docker build → Acceptance tests → Production
7. **Automate everything** - From building images to switching clusters
8. **Debug systematically** - Check logs, verify permissions, test connectivity
9. **Staging is your safety net** - Always validate in staging before production
10. **Kubernetes + Jenkins = Powerful** - Automated, reliable, traceable deployments at scale