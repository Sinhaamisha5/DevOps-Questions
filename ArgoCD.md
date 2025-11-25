# GitOps & ArgoCD: Comprehensive Study Guide

## Table of Contents
1. [Introduction to GitOps](#introduction-to-gitops)
2. [Why GitOps](#why-gitops)
3. [GitOps Principles](#gitops-principles)
4. [How GitOps Works](#how-gitops-works)
5. [GitOps vs Traditional CI/CD](#gitops-vs-traditional-cicd)
6. [Popular GitOps Tools](#popular-gitops-tools)
7. [ArgoCD Deep Dive](#argocd-deep-dive)
8. [ArgoCD Architecture](#argocd-architecture)
9. [Installation Methods](#installation-methods)
10. [Complex Scenarios & Solutions](#complex-scenarios--solutions)
11. [Multi-Cluster Deployment](#multi-cluster-deployment)
12. [Security in ArgoCD](#security-in-argocd)
13. [Interview Questions & Case Study](#interview-questions--case-study)
14. [Best Practices](#best-practices)

---

## Introduction to GitOps

GitOps is a modern approach to infrastructure and application delivery that uses Git as the single source of truth for both infrastructure and application configurations. It brings DevOps best practices (version control, code review, CI/CD) to infrastructure provisioning.

**Core Concept:** Infrastructure and application states are declared in Git repositories, and automated systems keep the cluster state synchronized with the Git repository.

---

## Why GitOps

### Problems Without GitOps

1. **No Tracking Mechanism**: When you manually update Kubernetes resources, after 10 days you cannot track what changed, why it changed, or who changed it.

2. **No Versioning**: Unlike source code which is always versioned and tagged in Git with proper CI mechanisms, CD deployments lack proper versioning.

3. **Manual Drift**: Configuration drift occurs when resources are manually modified (through kubectl, UI dashboards, or scripts).

4. **Inconsistent Deployments**: Without version control, it's impossible to reproduce environments or rollback to previous states.

5. **Audit Trail Missing**: No clear history of who deployed what and when.

6. **Manual Changes Go Unnoticed**: A DevOps engineer might update a ConfigMap manually, and after 10-15 days when the application crashes, traditional shell scripts or Ansible won't detect these manual changes.

### How GitOps Solves These

- **Complete Audit Trail**: Every change is tracked in Git with commit history, PR approvals, and timestamps
- **Version Control**: All configurations are versioned and can be rolled back
- **Consistency**: Infrastructure and application state matches Git state automatically
- **Security**: Only GitOps controllers can modify cluster resources (principle of least privilege)
- **Transparency**: Everyone can see what's deployed and when

---

## GitOps Principles

### 1. **Declarative**
- Infrastructure and application configurations are declared in Git (as YAML manifests)
- You specify the desired state, not the steps to achieve it
- Example: Instead of "run these commands", you say "this is what the cluster should look like"

### 2. **Version Controlled & Immutable**
- All configurations are stored in Git repositories
- Changes follow Git workflow (branches, commits, pull requests)
- Every change is auditable and reversible
- Git becomes the single source of truth

### 3. **Pulled Automatically**
- GitOps controllers continuously pull the desired state from Git
- **Pull-based approach** (not push): Controllers watch Git and pull changes automatically
- This is different from traditional CI/CD where systems push changes

### 4. **Continuously Reconciled**
- GitOps controllers continuously compare:
  - **Desired state** (what's in Git)
  - **Actual state** (what's running in the cluster)
- If they differ, the controller reconciles them back to the desired state
- This is continuous because it happens 24/7, not just at deployment time

---

## How GitOps Works

```
┌─────────────────┐
│  Git Repository │
│  (Source Truth) │
└────────┬────────┘
         │
         │ 1. Developer pushes YAML manifest
         │    (with PR review & approval)
         │
         ▼
┌─────────────────┐
│  GitOps         │
│  Controller     │ 3. Watches Git repository
│  (e.g., ArgoCD) │ 4. Watches Kubernetes cluster
└────────┬────────┘ 5. Reconciles when diff exists
         │
         │ 2. Controller reads & applies changes
         │
         ▼
┌─────────────────┐
│ Kubernetes      │
│ Cluster         │
│ (Actual State)  │
└─────────────────┘
```

**Step-by-Step Flow:**

1. Developer commits YAML manifest to Git (with PR approval)
2. GitOps controller reads the manifest from Git
3. Controller continuously watches the Git repository for changes
4. Controller continuously watches the Kubernetes cluster state
5. When Git state differs from cluster state, controller reconciles by updating resources
6. Only the GitOps controller manages resources - nobody else should directly modify them

---

## GitOps vs Traditional CI/CD

| Aspect | Traditional CI/CD | GitOps |
|--------|-------------------|--------|
| **Deployment Model** | Push-based (CI/CD pushes changes) | Pull-based (Controllers pull changes) |
| **Source of Truth** | Multiple (code repo, CI/CD config, manual changes) | Git repository only |
| **Manual Changes** | Not detected or tracked | Automatically reverted |
| **Audit Trail** | Limited | Complete (Git history) |
| **Rollback** | Complex, requires re-running pipelines | Simple Git revert |
| **Configuration Drift** | Common problem | Automatically corrected |
| **Deployment Frequency** | Scheduled or manual | Automatic, continuous |
| **Security** | Push credentials needed everywhere | Pull-based, fewer exposed credentials |
| **Multi-env Deployment** | Separate pipelines per environment | Single Git repo, multiple branches/directories |

### Traditional Pipeline Example:
```
Dev writes code → Git → Jenkins builds → Tests → Sonarqube → Docker push → Shell script/Ansible deploys
                                                                                    (Last stage only)
Problem: If someone manually modifies ConfigMap after 10 days, shell script won't detect it
```

### GitOps Pipeline Example:
```
Dev writes YAML → Git (PR + approval) → ArgoCD watches → ArgoCD syncs → K8s
                                                                           ↑
                                                                           |
                                                    ArgoCD detects manual changes
                                                    and reverts to Git state
```

---

## Popular GitOps Tools

1. **ArgoCD** (Most Popular)
   - Focuses on Kubernetes
   - Excellent UI and CLI
   - Active community
   - Suitable for beginners to advanced users

2. **Flux CD**
   - Lightweight and flexible
   - GitOps by design
   - Good for GitOps purists

3. **JenkinsX**
   - Built on Jenkins
   - Opinionated workflow
   - Good for teams already using Jenkins

4. **Spinnaker**
   - Multi-cloud deployment
   - Complex but powerful
   - Better for sophisticated deployment pipelines

---

## ArgoCD Deep Dive

### What is ArgoCD?

ArgoCD is a declarative GitOps continuous delivery tool for Kubernetes. It automates the deployment of applications to Kubernetes clusters by maintaining synchronization between Git repositories and Kubernetes clusters.

### Is GitOps a Kubernetes Controller?

**Yes, partially.** GitOps tools like ArgoCD are implemented using Kubernetes controllers, but they're more sophisticated than typical controllers. They:

- Use **Custom Resource Definitions (CRDs)** to define Application resources
- Implement **reconciliation loops** (watching and comparing states)
- Act as **operators** that manage the entire deployment lifecycle
- Use **webhooks** and **polling** to detect changes

ArgoCD's Application controller watches Git and Kubernetes cluster simultaneously, making it a specialized controller that bridges Git and Kubernetes.

### Key ArgoCD Concepts

**Application:** A CRD that defines:
- Which Git repository to watch
- Which Kubernetes cluster to deploy to
- How to sync changes
- Health checks and rollback policies

**Sync Status:** Comparison between Git state and cluster state
- **Synced**: Git and cluster match
- **OutOfSync**: Differences exist
- **Unknown**: Controller cannot determine status

**Health Status:** Whether deployed resources are healthy
- **Healthy**: All resources running correctly
- **Progressing**: Resources being deployed
- **Degraded**: Issues detected
- **Unknown**: Status cannot be determined

---

## ArgoCD Architecture

### Components

```
┌────────────────────────────────────────────────────────────┐
│                    ArgoCD Components                       │
└────────────────────────────────────────────────────────────┘

┌──────────────────┐
│   Repo Server    │  - Connects to Git repositories
│                  │  - Fetches manifests (Kubernetes YAML)
│                  │  - Supports: Helm, Kustomize, Ksonnet
│                  │  - Caches manifest data in Redis
└────────┬─────────┘
         │
         │ 1. Pulls manifests from Git
         │
         ▼
┌──────────────────┐
│   API Server     │  - REST API for UI/CLI
│   (+ Dex)        │  - Handles authentication & SSO
│   + Dex OIDC     │  - Manages users & permissions
│                  │  - Webhook integration
└────────┬─────────┘
         │
         │ 2. Serves UI/CLI requests
         │
         ▼
┌──────────────────────────┐
│  Application Controller  │  - Watches Git repository
│  (Core Reconciler)       │  - Watches Kubernetes cluster
│                          │  - Compares desired vs actual state
│                          │  - Triggers sync when differences found
└────────┬─────────────────┘
         │
         │ 3. Gets actual cluster state
         │
         ▼
┌──────────────────┐
│  Kubernetes      │
│  Cluster         │
│  (Target)        │
└──────────────────┘

┌──────────────────┐
│     Redis        │  - In-memory cache store
│                  │  - Stores manifest data
│                  │  - Caches resource information
│                  │  - Improves performance
└──────────────────┘

┌──────────────────┐
│   Dex Server     │  - OpenID Connect (OIDC) provider
│                  │  - Enables SSO integration
│                  │  - Supports: GitHub, GitLab, Google, SAML
│                  │  - Provides secure authentication
└──────────────────┘
```

### Component Details

**Repo Server**
- Connects to Git repositories
- Clones or syncs repositories
- Parses Kubernetes manifests
- Supports multiple manifest formats:
  - Plain Kubernetes YAML
  - Helm charts (renders template + values)
  - Kustomize (overlays)
  - JsonNet (declarative configuration)
- Results are cached in Redis for performance

**Application Controller**
- Core reconciliation engine
- Runs continuous reconciliation loops
- Watches for changes in:
  - Git repository (every 3 minutes by default)
  - Kubernetes cluster (via watch API)
- Compares desired state (Git) with actual state (cluster)
- Applies changes only when necessary
- Supports different sync policies (automatic, manual, etc.)

**API Server**
- Exposes REST API for UI and CLI
- Handles user authentication and authorization
- Integrates with Dex for SSO
- Manages webhooks for Git events
- Provides application management endpoints

**Dex Server**
- OpenID Connect (OIDC) provider built into ArgoCD
- Enables Single Sign-On (SSO)
- Supports multiple identity providers:
  - GitHub OAuth
  - GitLab OAuth
  - Google OAuth
  - SAML
  - LDAP
- Provides secure credential exchange
- Issues tokens for authentication

**Redis**
- In-memory data store
- Caches Git manifest information
- Stores stateless session data
- Improves performance by reducing Git repository access
- Can be stateful or stateless depending on setup

**UI**
- Web-based dashboard
- Real-time status visualization
- Manual sync trigger
- Application management
- User-friendly for non-CLI users

**CLI**
- Command-line interface
- Programmatic interaction with ArgoCD
- Useful for automation and scripting
- Better for experienced users

---

## Installation Methods

### 1. **YAML Installation**
Most direct method using Kubernetes manifest files

```bash
# Create ArgoCD namespace
kubectl create namespace argocd

# Apply official ArgoCD manifests
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Verify installation
kubectl get pods -n argocd
```

**Pros:** Simple, no dependencies, official manifests
**Cons:** Manual updates required, less customizable

### 2. **Helm Installation**
Using Helm charts for more customization

```bash
# Add ArgoCD Helm repository
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

# Install ArgoCD using Helm
helm install argocd argo/argo-cd -n argocd --create-namespace

# With custom values
helm install argocd argo/argo-cd -n argocd -f values.yaml
```

**Pros:** Highly customizable, easy to upgrade, values-driven config
**Cons:** Slightly more complex, requires Helm knowledge

### 3. **Operator Installation**
Using ArgoCD Operator for enterprise deployments

```bash
# Install ArgoCD Operator
kubectl apply -f https://operatorhub.io/install/argocd-operator.yaml

# Create ArgoCD instance using CRD
kubectl apply -f argocd-instance.yaml
```

**Pros:** Kubernetes-native, easier lifecycle management, automatic updates
**Cons:** Requires operator pattern knowledge, relatively new

### Post-Installation Setup

```bash
# Verify all pods are running
kubectl get pods -n argocd

# Get initial password (username: admin)
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# Expose ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443
# or create NodePort service
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'

# Access UI: https://localhost:8080
```

---

## Complex Scenarios & Solutions

### Scenario 1: Admission Controller Conflicts

**Problem:** Git is the single source of truth, but admission controllers automatically inject resource requests/limits or add sidecars. Should these be removed since they're not in Git?

**Solution:**
1. **Include admission changes in Git**: Commit the mutated resources to Git after first deployment
2. **Webhook validation**: Use validating webhooks instead of mutating webhooks where possible
3. **Exclude from sync**: Configure ArgoCD to ignore certain fields using `.argocd-ignore`
4. **Custom sync options**: Use `ignoreDifferences` in Application CRD

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
spec:
  ignoreDifferences:
  - group: apps
    kind: Deployment
    jsonPointers:
    - /spec/template/spec/containers/0/resources
```

**Best Practice:** Admission controllers and ArgoCD should be designed to work together. The cluster's mutation logic should be:
- Idempotent (safe to apply multiple times)
- Documented in Git
- Reflected in your Git manifests

### Scenario 2: Existing Resources in Cluster

**Problem:** When adopting GitOps, there are already resources in the cluster that aren't in Git. Will GitOps delete them?

**Solution:**

ArgoCD won't delete resources by default. You have control through **sync policies**:

1. **Default behavior (Selective Sync):**
   - Only syncs resources defined in Git
   - Ignores resources not in Git
   - Manual resources remain untouched

2. **Automatic Pruning (Dangerous!):**
   - Can be enabled but not recommended initially
   ```yaml
   syncPolicy:
     automated:
       prune: true  # Deletes resources in cluster but not in Git
   ```

3. **Safe Adoption Strategy:**
   - First, sync without pruning
   - Gradually migrate resources to Git
   - Test each component
   - Only enable pruning after full migration

4. **Moving Existing Resources:**
   ```bash
   # Export existing resource to Git
   kubectl get deployment myapp -o yaml > myapp.yaml
   
   # Commit to Git
   git add myapp.yaml
   git commit -m "Move deployment to GitOps"
   
   # ArgoCD will recognize it and manage it
   ```

**Best Practice:** Adopt GitOps gradually. Don't enable `prune: true` until you're confident all resources are managed.

---

## Multi-Cluster Deployment

### Use Case
In organizations with multiple environments (Dev, QA, Pre-prod, Prod), you want to:
- Deploy the same application version across all clusters
- Update versions once and propagate to all clusters
- Maintain consistency while respecting environment differences
- Manage from a centralized location

### ArgoCD Deployment Models

#### 1. **Standalone Model**

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Dev Env    │     │  QA Env      │     │  Prod Env    │
├──────────────┤     ├──────────────┤     ├──────────────┤
│  ArgoCD      │     │  ArgoCD      │     │  ArgoCD      │
│  Instance    │     │  Instance    │     │  Instance    │
└──────────────┘     └──────────────┘     └──────────────┘
       │                    │                     │
       └────────┬───────────┴─────────┬───────────┘
                │                     │
         ┌──────▼──────┐       ┌──────▼──────┐
         │ Dev Git     │       │ Prod Git    │
         │ Repo        │       │ Repo        │
         └─────────────┘       └─────────────┘
```

**Characteristics:**
- One ArgoCD instance per environment
- Each instance watches its own Git repository (or branch)
- Independent deployments
- More resource-intensive
- Easier to isolate environments

**Advantages:**
- Full independence per environment
- Easier to manage different GitOps configurations
- Better security isolation
- Failure in one doesn't affect others

**Disadvantages:**
- Redundant resource consumption
- Complex to manage at scale
- Synchronization between clusters harder
- More operational overhead

#### 2. **Hub-and-Spoke Model** (Recommended for Multi-Cluster)

```
                    ┌─────────────────┐
                    │   Hub Cluster   │
                    │  (Central)      │
                    │                 │
                    │ ┌─────────────┐ │
                    │ │  ArgoCD     │ │
                    │ │  Central    │ │
                    │ └──────┬──────┘ │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
         ┌────▼────┐    ┌────▼────┐    ┌────▼────┐
         │  Dev    │    │   QA    │    │  Prod   │
         │ Cluster │    │ Cluster │    │ Cluster │
         │(Spoke)  │    │(Spoke)  │    │(Spoke)  │
         │         │    │         │    │         │
         │ K8s API │    │ K8s API │    │ K8s API │
         └─────────┘    └─────────┘    └─────────┘
              ▲              ▲              ▲
              │              │              │
              └──────────────┬──────────────┘
                             │
                    ┌────────▼────────┐
                    │ Git Repository  │
                    │ (Single Source  │
                    │  of Truth)      │
                    └─────────────────┘
```

**Characteristics:**
- One central ArgoCD in hub cluster
- Other clusters register as "spoke" clusters
- All clusters controlled from hub
- Single Git repository (or branches)
- Cost-effective resource usage

**Advantages:**
- Single control plane for all clusters
- Reduced resource consumption
- Easier to manage at scale
- Centralized auditing and monitoring
- Simplified secrets management
- One Git repo as source of truth

**Disadvantages:**
- Hub cluster is critical (single point of failure)
- Complex networking requirements
- Requires strong RBAC configuration
- Harder to isolate environments if one team manages all
- More complex troubleshooting

### Setup Walkthrough: Hub-Spoke Model with AWS EKS

**Prerequisites:**
- 3 EKS clusters: hub, dev-spoke, qa-spoke
- kubectl access to all clusters
- ArgoCD CLI installed

**Step 1: Install ArgoCD on Hub Cluster**

```bash
# Switch to hub cluster context
kubectl config use-context hub-cluster

# Create ArgoCD namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Verify installation
kubectl get pods -n argocd
```

**Step 2: Configure Secure Mode**

```bash
# Edit ArgoCD ConfigMap for secure mode
kubectl edit configmap argocd-cmd-params-cm -n argocd

# Ensure these settings:
# application.instanceLabelKey: argocd.argoproj.io/instance
# server.disable.auth: "false"  # Authentication enabled
# server.insecure: "false"       # HTTPS required

# Edit service to expose via NodePort
kubectl edit svc argocd-server -n argocd
# Change spec.type from ClusterIP to NodePort
```

**Step 3: Retrieve Initial Credentials**

```bash
# Get admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# Get NodePort service info
kubectl get svc argocd-server -n argocd
# Note the NodePort (e.g., 30000)

# Access UI: https://your-hub-node-ip:NodePort
```

**Step 4: Register Spoke Clusters**

```bash
# Download ArgoCD CLI
curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x argocd

# Login to ArgoCD (from hub cluster)
./argocd login your-hub-node-ip:NodePort --username admin --password <password> --insecure

# Get service account token from spoke cluster
kubectl config use-context dev-spoke-cluster
TOKEN=$(kubectl -n default get secret $(kubectl -n default get secret -o name | grep argocd) -o jsonpath='{.data.token}' | base64 --decode)

# Get API server
API_SERVER=$(kubectl cluster-info | grep 'Kubernetes master' | awk '/https/ {print $NF}')

# Register spoke cluster with hub ArgoCD
./argocd cluster add dev-spoke-cluster \
  --server https://your-hub-node-ip:NodePort \
  --auth-token <admin-token>

# Repeat for QA spoke cluster
kubectl config use-context qa-spoke-cluster
./argocd cluster add qa-spoke-cluster \
  --server https://your-hub-node-ip:NodePort \
  --auth-token <admin-token>

# Verify registered clusters
./argocd cluster list
```

**Step 5: Create Applications for Each Spoke Cluster**

```bash
# Create dev application
./argocd app create dev-app \
  --repo https://github.com/yourorg/gitops-configs.git \
  --path ./dev \
  --dest-server https://dev-spoke-cluster-api-server \
  --dest-namespace default

# Create QA application
./argocd app create qa-app \
  --repo https://github.com/yourorg/gitops-configs.git \
  --path ./qa \
  --dest-server https://qa-spoke-cluster-api-server \
  --dest-namespace default

# View applications
./argocd app list
```

**Step 6: Set Sync Policy**

```bash
# Enable automatic sync for applications
./argocd app set dev-app --sync-policy automated --auto-prune --self-heal
./argocd app set qa-app --sync-policy automated --auto-prune --self-heal

# Or manually via UI: Applications > Select > Sync Details > Auto Sync
```

**Step 7: Test Deployment**

```bash
# Push changes to Git
git commit -am "Update version to 1.2.3"
git push origin main

# Monitor sync in hub ArgoCD UI
# You'll see applications transitioning to "Synced" state

# Verify deployments in spoke clusters
kubectl config use-context dev-spoke-cluster
kubectl get deployments -n default

kubectl config use-context qa-spoke-cluster
kubectl get deployments -n default
```

### Testing Scenarios

**Test 1: Manual Change Detection**
```bash
# SSH to dev spoke cluster
kubectl config use-context dev-spoke-cluster

# Make manual change to a resource
kubectl set image deployment/myapp myapp=myapp:1.1.0

# Watch ArgoCD detect the drift
# Within 3-5 minutes, ArgoCD will revert it to Git state
kubectl get pods  # You'll see it being updated back to 1.2.3
```

**Test 2: Version Update Across All Clusters**
```bash
# Update version in Git for all environments
# Edit dev/deployment.yaml and qa/deployment.yaml
image: myapp:1.3.0  # Updated version

# Commit and push
git commit -am "Bump version to 1.3.0 across all environments"
git push origin main

# Monitor UI - both dev-app and qa-app will sync
# Check spoke clusters to confirm new version is deployed
```

---

## Security in ArgoCD

### Securing ArgoCD Installation

#### 1. **HTTPS/TLS Configuration**

```bash
# Create TLS certificate
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt -subj "/CN=argocd.example.com"

# Create secret
kubectl create secret tls argocd-tls --cert=tls.crt --key=tls.key -n argocd

# Update service to use HTTPS
kubectl patch svc argocd-server -n argocd -p '{"spec":{"tls":{"cert":"tls.crt","key":"tls.key"}}}'
```

#### 2. **Ingress with TLS**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-ingress
  namespace: argocd
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - argocd.example.com
    secretName: argocd-tls-secret
  rules:
  - host: argocd.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: argocd-server
            port:
              number: 443
```

#### 3. **Authentication & Authorization**

**RBAC Configuration:**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
data:
  policy.default: role:readonly
  policy.csv: |
    p, role:admin, *, *, *, allow
    p, role:readonly, applications, get, *, allow
    p, role:dev, applications, *, dev-*, allow
    g, your-github-team, role:dev
    g, admin-team, role:admin
```

**OIDC/SSO Setup:**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
data:
  url: https://argocd.example.com
  oidc.config: |
    name: GitHub
    issuer: https://token.actions.githubusercontent.com
    clientID: your-github-app-id
    clientSecret: $github-client-secret
    requestedScopes:
    - openid
    - profile
    - email
    - groups
```

#### 4. **Secret Management**

**Don't Store Secrets in Git!** Use:

- **Sealed Secrets**: Encrypt secrets before committing
- **External Secrets Operator**: Fetch from AWS Secrets Manager, HashiCorp Vault
- **Bitnami Sealed Secrets**: Kubernetes-native secret encryption

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: my-secret
  namespace: default
spec:
  encryptedData:
    password: AgBvB8CxL1L8F9N2E...  # Encrypted value
  template:
    type: Opaque
```

#### 5. **Network Policies**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: argocd-netpol
  namespace: argocd
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: argocd
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector: {}
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
  egress:
  - to:
    - namespaceSelector: {}
    - podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
  - to:
    - podSelector: {}
```

#### 6. **RBAC for Cluster Access**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: argocd-application-controller
rules:
- apiGroups: ['*']
  resources: ['*']
  verbs: ['*']
- nonResourceURLs: ['*']
  verbs: ['*']
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: argocd-application-controller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: argocd-application-controller
subjects:
- kind: ServiceAccount
  name: argocd-application-controller
  namespace: argocd
```

---

## Interview Questions & Case Study

### Interview Questions

#### Beginner Level

1. **What is GitOps and how is it different from traditional CI/CD?**
   - *Answer Framework*: GitOps uses Git as single source of truth with pull-based deployments, automatically detecting and correcting drift. Traditional CI/CD uses push-based approach without continuous reconciliation.

2. **Explain the four principles of GitOps.**
   - *Answer Framework*: Declarative (desired state in YAML), Versioned (Git tracked), Pulled (controllers pull changes), Reconciled (continuous comparison).

3. **What is the difference between push-based and pull-based deployment?**
   - *Answer Framework*: Push: CI/CD system deploys (requires credentials everywhere). Pull: Cluster controller fetches from Git (more secure, pulls at intervals).

What is configuration drift and how does GitOps prevent it?

Answer Framework: Configuration drift is when actual state diverges from desired state. GitOps prevents it through continuous reconciliation - controllers constantly compare Git vs cluster and revert any unauthorized changes.


What components does ArgoCD have and what does each do?

Answer Framework: Repo Server (fetches from Git), Application Controller (reconciles), API Server (handles auth/webhooks), Dex (SSO), Redis (cache), UI/CLI (interfaces).



Intermediate Level

How does ArgoCD handle a scenario where an admission controller mutates resources after ArgoCD deploys them?

Answer Framework: This can cause OutOfSync status. Solutions include: (1) including mutations in Git manifests, (2) using ignoreDifferences in Application CRD, (3) designing webhooks to be idempotent and documented.


You're adopting GitOps for a cluster with existing resources not in Git. What's your strategy?

Answer Framework: Adopt gradually. First disable pruning (default), export existing resources to Git, test each component, gradually enable pruning as resources move to Git. Don't enable pruning prematurely.


Explain the difference between standalone and hub-spoke ArgoCD deployment models.

Answer Framework: Standalone: One ArgoCD per cluster, independent management. Hub-spoke: Central ArgoCD (hub) manages multiple clusters (spokes), more efficient at scale, but hub is single point of failure.


How would you implement multi-cluster deployments where you deploy a single version across Dev, QA, Prod?

Answer Framework: Use hub-spoke model. Central ArgoCD in hub watches single Git repo. Spoke clusters register with hub. Create applications pointing to same Git repo but different paths (dev/, qa/, prod/) targeting respective spoke clusters.


What security measures should you implement for ArgoCD?

Answer Framework: HTTPS/TLS, RBAC with policy.csv, OIDC/SSO integration, never store secrets in Git (use Sealed Secrets/External Secrets), network policies, least privilege service accounts, secret scanning in Git repos.


How do you prevent secrets from being committed to Git in ArgoCD?

Answer Framework: Use Sealed Secrets (Bitnami) to encrypt secrets before commit, External Secrets Operator to fetch from vaults, .gitignore for sensitive files, pre-commit hooks to scan for secrets, branch protection rules.



Advanced Level

Design a multi-cluster, multi-tenant GitOps architecture where different teams manage different applications.

Answer Framework: Use hub-spoke + AppProject CRD. Each team gets AppProject with RBAC restrictions. Teams have folders in Git (team-a/, team-b/). Applications deployed to specific namespaces. Implement network policies between tenants. Use RBAC policy.csv to enforce boundaries.


A team manually modified a database connection string in a running deployment (not in Git). How would ArgoCD react and what's the recovery process?

Answer Framework: If self-heal enabled: ArgoCD detects drift within 3-5 mins, reverts to Git state (overwrites manual change). Recovery: Update Git with new connection string, commit, push. ArgoCD syncs changes back.


How would you handle GitOps for stateful applications like databases that require specific ordering of deployments?

Answer Framework: Use Helm charts with ordering/dependencies, or separate Applications with manual sync policies. Use InitContainers for dependency checks. Consider using operators for complex stateful apps (Postgres Operator, MySQL Operator). Argo Workflows for complex sequencing.


Design a disaster recovery strategy using GitOps. Cluster crashes - how do you recover?

Answer Framework: Everything is in Git. Spin up new cluster, install ArgoCD, register with hub (if hub-spoke), point to same Git repo. Applications automatically deploy within minutes. Git is source of truth - instant recovery.


How would you implement canary/blue-green deployments with ArgoCD?

Answer Framework: Use Argo Rollouts (complementary tool) for canary/blue-green. Define rollout strategy in Git manifests. ArgoCD deploys Rollout resources, Argo Rollouts controller manages traffic shifting. Monitor metrics, automatic rollback on failures.




Interview Ready Case Study
Case Study: E-Commerce Platform Multi-Environment Deployment
Company Background:
TechCart is a growing e-commerce platform with microservices architecture. They currently use Jenkins CI/CD with manual Kubernetes deployments. Issues:

Manual changes lost after weeks
Configuration drift causes mysterious failures
No clear audit trail for production incidents
Deploying same version across 3 environments takes 2-3 hours
Team members sometimes bypass deployment pipeline

Current Architecture:

3 Kubernetes clusters: Development, Staging, Production
AWS EKS
8 microservices (API, Frontend, Payment, Inventory, Auth, etc.)
Currently 12 team members
Multiple teams: Platform, Services, QA

Requirements:

Deploy same version across all 3 environments within 5 minutes
Clear audit trail for all deployments
Automatic rollback if manual changes detected
Team-based access control (Platform team manages core infra, Service team manages app)
Disaster recovery: Recover from cluster failure in < 10 minutes
Multi-cluster management with single control plane

Your Task: Design the GitOps solution using ArgoCD
Solution Walkthrough
Architecture Design:
                    ┌─────────────────┐
                    │  EKS - Hub      │
                    │  Region: us-east│
                    │                 │
                    │ ┌─────────────┐ │
                    │ │  ArgoCD     │ │
                    │ │  Central    │ │
                    │ │  (HA mode)  │ │
                    │ └──────┬──────┘ │
                    │   All clusters  │
                    │   managed from  │
                    │   hub           │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
         ┌────▼────┐    ┌────▼────┐    ┌────▼────┐
         │   Dev   │    │ Staging │    │  Prod   │
         │ EKS     │    │   EKS   │    │   EKS   │
         │us-east2 │    │ us-west1│    │us-central
         └─────────┘    └─────────┘    └─────────┘
              ▲              ▲              ▲
              └──────────────┬──────────────┘
                             │
          ┌──────────────────┴──────────────────┐
          │                                     │
     ┌────▼──────────────┐          ┌──────────▼─────┐
     │ GitOps Git Repo   │          │   Secrets Repo │
     │ (Public)          │          │   (Private)    │
     ├───────────────────┤          ├────────────────┤
     │ /dev              │          │ Sealed Secrets │
     │  /microservices   │          │ (Encrypted)    │
     │  /platform        │          │                │
     │ /staging          │          │ OAuth tokens   │
     │  /microservices   │          │ DB passwords   │
     │  /platform        │          │ API keys       │
     │ /prod             │          └────────────────┘
     │  /microservices   │
     │  /platform        │
     │ /applications     │
     │  (per env)        │
     └───────────────────┘
Git Repository Structure:
gitops-techwart/
├── dev/
│   ├── namespace.yaml
│   ├── microservices/
│   │   ├── api-service.yaml
│   │   ├── frontend-service.yaml
│   │   ├── payment-service.yaml
│   │   └── ...
│   ├── platform/
│   │   ├── monitoring.yaml
│   │   ├── logging.yaml
│   │   └── networking.yaml
│   └── application-dev.yaml
│
├── staging/
│   ├── namespace.yaml
│   ├── microservices/
│   ├── platform/
│   └── application-staging.yaml
│
├── prod/
│   ├── namespace.yaml
│   ├── microservices/
│   ├── platform/
│   └── application-prod.yaml
│
├── argocd-setup/
│   ├── argocd-cm.yaml (config)
│   ├── argocd-rbac.yaml (policy.csv)
│   ├── argocd-secret.yaml (OIDC)
│   └── network-policy.yaml
│
├── sealed-secrets/
│   └── (encrypted secrets)
│
├── README.md
└── .gitignore
ArgoCD Applications Structure:
yaml# 1. Application for Dev
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: techwart-dev
  namespace: argocd
spec:
  project: development-team
  source:
    repoURL: https://github.com/techwart/gitops-techwart.git
    targetRevision: main
    path: dev
  destination:
    server: https://dev-eks-cluster-api
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true

---
# 2. Application for Staging
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: techwart-staging
  namespace: argocd
spec:
  project: development-team
  source:
    repoURL: https://github.com/techwart/gitops-techwart.git
    targetRevision: main
    path: staging
  destination:
    server: https://staging-eks-cluster-api
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true

---
# 3. Application for Prod (Manual Sync for Safety)
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: techwart-prod
  namespace: argocd
spec:
  project: production-team
  source:
    repoURL: https://github.com/techwart/gitops-techwart.git
    targetRevision: main
    path: prod
  destination:
    server: https://prod-eks-cluster-api
    namespace: default
  syncPolicy:
    syncOptions:
    - CreateNamespace=true
    # Manual sync for production - requires approval
RBAC Configuration:
yamlapiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
data:
  policy.default: role:readonly
  policy.csv: |
    # Define roles
    p, role:admin, *, *, *, allow
    p, role:platform-admin, applications, *, platform-*, allow
    p, role:platform-readonly, applications, get, platform-*, allow
    p, role:service-admin, applications, *, microservices-*, allow
    p, role:service-readonly, applications, get, microservices-*, allow
    p, role:devops, *, *, *, allow
    p, role:qa-readonly, applications, get, *, allow
    
    # Assign users to roles
    g, platform-team, role:platform-admin
    g, service-team, role:service-admin
    g, devops-team, role:devops
    g, qa-team, role:qa-readonly
    g, cto@techwart.com, role:admin
```

**Deployment Workflow - How it Works:**
```
Developer commits code
         │
         ▼
┌────────────────────┐
│ Git Webhook        │
│ Triggers GitHub    │
│ Action             │
└────────┬───────────┘
         │
         ▼
┌────────────────────────────┐
│ GitHub Actions             │
│ - Build Docker image       │
│ - Run tests                │
│ - Push to ECR              │
│ - Update Image tag in YAML │
│ - Create PR                │
└────────┬───────────────────┘
         │
         ▼
┌────────────────────────────┐
│ PR Review & Approval       │
│ Code review + approval     │
│ Merge to main branch       │
└────────┬───────────────────┘
         │
         ▼
┌────────────────────────────┐
│ Hub ArgoCD Detects Change  │
│ Polls Git every 3 minutes  │
│ or Webhook triggers        │
└────────┬───────────────────┘
         │
         ▼
┌────────────────────────────────────┐
│ ArgoCD Syncs All Environments      │
│ - Dev: Auto sync (5 seconds)       │
│ - Staging: Auto sync (5 seconds)   │
│ - Prod: Manual approval required   │
│   (Dev/Staging verified first)     │
└────────┬───────────────────────────┘
         │
         ▼
┌────────────────────────────────────┐
│ Applications Deployed Across All   │
│ 3 Clusters Simultaneously          │
│ (All within 2 minutes)             │
└────────────────────────────────────┘
```

**Scenario: Manual Change Detection**
```
DevOps Engineer manually updates ConfigMap in production
                     │
                     ▼
         ┌──────────────────────────┐
         │ Change made outside Git  │
         │ (e.g., via kubectl)      │
         │ Cluster state changes    │
         └────────────┬─────────────┘
                      │
                      ▼
         ┌──────────────────────────┐
         │ ArgoCD reconciliation    │
         │ loop runs (3-5 mins)     │
         │ Detects drift            │
         └────────────┬─────────────┘
                      │
         ┌────────────┴────────────┐
         │                         │
         ▼                         ▼
    ┌────────────┐         ┌──────────────┐
    │ Self-Heal  │         │ Revert change│
    │ Enabled    │         │ to Git state │
    └────────────┘         └──────────────┘
         │
         ▼
    ┌──────────────────────┐
    │ ConfigMap restored   │
    │ to Git version       │
    │ Alert sent to team   │
    │ Audit log created    │
    └──────────────────────┘
```

**Disaster Recovery Scenario:**
```
Production cluster goes down (hardware failure, network issue)
                     │
                     ▼
         ┌──────────────────────────┐
         │ Old Prod Cluster Lost    │
         │ ArgoCD Hub Still Running │
         └────────────┬─────────────┘
                      │
         ┌────────────┴────────────────┐
         │ Recovery Steps             │
         │                            │
         ▼                            ▼
    ┌────────────────┐         ┌──────────────┐
    │ 1. Provision   │         │ 2. Register  │
    │    New Prod    │         │    cluster   │
    │    EKS Cluster │         │    with hub  │
    │ (Template from │         │    ArgoCD    │
    │  Terraform)    │         └──────────────┘
    │ Time: 3 mins   │                │
    └────────────────┘                ▼
                      ┌────────────────────────┐
                      │ 3. ArgoCD Auto Deploys │
                      │    from Git            │
                      │ - All microservices    │
                      │ - Platform components  │
                      │ - Secrets (via Sealed  │
                      │   Secrets Operator)    │
                      │ Time: 5-7 minutes      │
                      └────────────────────────┘
                                │
                                ▼
                      ┌────────────────────────┐
                      │ Total Recovery: ~10 min│
                      │ All apps running       │
                      │ From Git source truth  │
                      └────────────────────────┘
Implementation Steps - Detailed
Step 1: Set Up Git Repository
bash# Clone and structure
git clone https://github.com/techwart/gitops-techwart.git
cd gitops-techwart

# Create directory structure
mkdir -p {dev,staging,prod}/{microservices,platform}
mkdir -p {argocd-setup,sealed-secrets}

# Commit
git add .
git commit -m "GitOps repo structure"
git push origin main
Step 2: Install ArgoCD on Hub Cluster (HA Mode)
bash# Switch to hub cluster
kubectl config use-context hub-cluster

# Create namespace
kubectl create namespace argocd

# Install ArgoCD with HA setup
helm repo add argo https://argoproj.github.io/argo-helm
helm install argocd argo/argo-cd \
  -n argocd \
  --set server.replicas=2 \
  --set repo.replicas=2 \
  --set controller.replicas=2 \
  --set redis.enabled=true \
  -f argocd-values.yaml
Step 3: Configure RBAC & OIDC
bash# Apply RBAC ConfigMap
kubectl apply -f argocd-rbac.yaml -n argocd

# Apply OIDC Configuration (for GitHub OAuth)
kubectl apply -f argocd-oidc-secret.yaml -n argocd
Step 4: Register Spoke Clusters
bash# Install ArgoCD CLI
curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x argocd

# Login to hub
./argocd login hub-argocd.example.com --username admin --password <password>

# Register spoke clusters
./argocd cluster add dev-cluster --name dev-eks
./argocd cluster add staging-cluster --name staging-eks
./argocd cluster add prod-cluster --name prod-eks

# Verify
./argocd cluster list
Step 5: Create Applications
bash# Create from YAML
kubectl apply -f applications/ -n argocd

# Or via CLI
./argocd app create techwart-dev \
  --repo https://github.com/techwart/gitops-techwart.git \
  --path dev \
  --dest-server https://dev-eks-api \
  --dest-namespace default \
  --sync-policy automated
Step 6: Deploy and Test
bash# Update application version in Git
echo "image: api-service:v1.2.3" >> dev/microservices/api-service.yaml
git add .
git commit -m "Update API version to 1.2.3"
git push origin main

# Watch ArgoCD sync
./argocd app sync techwart-dev
./argocd app watch techwart-dev

# Verify across clusters
kubectl config use-context dev-cluster
kubectl get deployments -n default

Best Practices
1. Git Repository Organization

Organize by environment: /dev, /staging, /prod
Organize by component: /microservices, /platform, /infrastructure
Keep secrets separate in private repos or use Sealed Secrets
Use .gitignore for sensitive files

2. Application Structure

One Application CRD per environment
Use appropriate sync policies (auto for dev, manual for prod)
Enable selfHeal for automatic drift correction
Set prune: true carefully after validation

3. Secrets Management

NEVER commit plain-text secrets to Git
Use Sealed Secrets or External Secrets Operator
Store secret encryption keys securely
Rotate secrets regularly
Separate secrets repository (private) if needed

4. Deployment Strategy

Use manual sync for production
Implement approval workflow for prod changes
Test in dev/staging before production
Use GitOps + Argo Rollouts for canary deployments
Monitor and alert on sync failures

5. Monitoring & Observability

Monitor ArgoCD itself (logs, metrics)
Alert on OutOfSync status
Track deployment frequency and lead time
Use ArgoCD Notifications for alerts
Implement distributed tracing for microservices

6. Scale Considerations

Use hub-spoke model for 3+ clusters
Implement resource quotas per project
Use AppProject for multi-tenancy
Consider Argo ApplicationSet for dynamic application generation
Use repository caching for large repos

7. Security Hardening

Use TLS/HTTPS everywhere
Implement RBAC with principle of least privilege
Use OIDC/SSO for authentication
Network policies between namespaces
Regular security audits
Scan Git repos for secrets (using git-secrets, talisman)

8. Disaster Recovery

Keep Git repo as backup (everything is there)
Document recovery procedures
Test recovery regularly
Use IaC for cluster provisioning (Terraform)
Multi-region setup for critical applications

9. Performance Optimization

Tune reconciliation intervals (default 3 mins)
Use Redis for caching
Implement webhook triggers (faster than polling)
Minimize resource requests for ArgoCD components
Use resource quotas appropriately

10. Team Collaboration

Define clear ownership (who manages what)
Use RBAC for team isolation
Require PR reviews for deployments
Implement change approval workflows
Document deployment procedures
Regular training for team members


Additional Important Topics Not Covered
1. Argo Rollouts

Progressive delivery strategies (canary, blue-green)
Traffic management and analysis
Automated rollbacks based on metrics

2. Argo Workflows

Complex deployment orchestration
Multi-step deployments with dependencies
Conditional workflows

3. ApplicationSet

Generate applications dynamically
Matrix generators for combinations
Git generator for multi-repo patterns

4. Notifications

Slack notifications on sync events
Email alerts
Custom webhooks