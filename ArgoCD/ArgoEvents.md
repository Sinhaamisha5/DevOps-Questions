# Argo Events: Complete Documentation

## Table of Contents
1. [What is Argo Events](#what-is-argo-events)
2. [Use Cases](#use-cases)
3. [When to Use Argo Events](#when-to-use-argo-events)
4. [How Argo Events Works](#how-argo-events-works)
5. [Why Use Argo Events](#why-use-argo-events)
6. [Installation Guide](#installation-guide)
7. [Core Concepts](#core-concepts)
8. [Practical Example: Webhook to Workflow](#practical-example-webhook-to-workflow)
9. [Advanced Configuration](#advanced-configuration)
10. [Best Practices](#best-practices)

---

## What is Argo Events

Argo Events is an event-driven workflow automation framework designed for Kubernetes. It enables you to listen to external events from various sources and trigger actions within your Kubernetes cluster in response to those events. Argo Events acts as a bridge between the external world and your Kubernetes infrastructure, converting external happenings into orchestrated Kubernetes actions.

### Core Capabilities

Argo Events can:
- Convert external events into Kubernetes object creation and modification
- Trigger Argo Workflows automatically based on events
- Launch serverless workloads in response to external signals
- Integrate with multiple event sources seamlessly
- Provide reliable, scalable event processing within Kubernetes

### Event Sources Supported

Argo Events can consume events from:
- **Webhooks**: HTTP POST requests from external systems
- **Cloud Storage**: AWS S3 notifications
- **Messaging Queues**: AWS SQS, AWS SNS
- **Pub/Sub Systems**: Google Cloud Pub/Sub
- **Scheduled Events**: CRON expressions for time-based triggers
- **Registry Events**: Container registry notifications (Docker Hub, ECR, GCR)
- **And many more**: GitHub webhooks, GitLab events, Kafka, NATS, etc.

---

## Use Cases

### 1. CI/CD Pipeline Automation
Trigger Argo Workflows automatically when code is pushed to a Git repository, enabling continuous integration and deployment without manual intervention.

**Example**: A GitHub webhook triggers a build workflow whenever code is merged to the main branch.

### 2. Event-Driven Data Processing
Process data automatically as soon as files are uploaded to cloud storage or messages arrive in a queue.

**Example**: When an image is uploaded to S3, automatically trigger a machine learning pipeline for image processing or analysis.

### 3. Monitoring and Alerting Automation
Respond to monitoring alerts by automatically triggering remediation workflows.

**Example**: When a Prometheus alert fires, trigger a workflow to scale up resources or restart problematic pods.

### 4. Scheduled Task Execution
Run batch jobs, cleanup tasks, or maintenance workflows on a schedule using CRON expressions.

**Example**: Daily backup workflows triggered at midnight, or weekly database maintenance tasks.

### 5. Microservices Integration
Decouple microservices by triggering asynchronous workflows in response to events from other services.

**Example**: When a new user signs up, trigger workflows for sending welcome emails, creating user records, and setting up initial permissions.

### 6. Multi-Cloud Orchestration
Coordinate complex workflows across multiple cloud providers by responding to events from different cloud platforms.

**Example**: AWS S3 events trigger workflows that process data on Google Cloud, store results in Azure, and update on-premise systems.

### 7. Serverless Function Invocation
Trigger serverless functions (AWS Lambda, Google Cloud Functions) in response to Kubernetes events or external signals.

---

## When to Use Argo Events

### Appropriate Scenarios

Use Argo Events when you need to:
- **React to external events**: Your workflow depends on external signals or conditions that occur unpredictably
- **Eliminate manual triggers**: Automate workflow initiation that currently requires manual intervention
- **Build event-driven architectures**: Design systems where components communicate through events
- **Integrate with cloud services**: Connect Kubernetes workflows with cloud provider services and notifications
- **Implement real-time processing**: Process events as they occur rather than on a fixed schedule
- **Create complex automation chains**: Chain multiple workflows or actions based on event conditions

### Not Appropriate For

You should consider alternatives when:
- **Simple scheduling is sufficient**: If you only need to run tasks on a schedule, use Kubernetes CronJobs
- **Direct API calls are simpler**: For simple synchronous operations, direct API calls may be more straightforward
- **Real-time requirements are extreme**: If you need sub-millisecond latency, consider lower-level event streaming solutions
- **Your infrastructure is not Kubernetes-based**: Argo Events is designed specifically for Kubernetes

---

## How Argo Events Works

### Architecture Overview

Argo Events consists of three main components:

1. **EventSource**: Listens to events from external sources and exposes them
2. **EventBus**: Acts as a message bus that receives events from EventSources and delivers them to Sensors
3. **Sensor**: Watches for specific events and triggers actions (workflows, Kubernetes objects, etc.)

### Event Flow

```
External Event → EventSource → EventBus → Sensor → Action (Create Workflow/Object)
```

### Step-by-Step Process

**Step 1: EventSource Initialization**
- You define an EventSource resource that specifies which external source to listen to
- The EventSource controller creates a pod that runs a listener for that event source
- The listener exposes the events through a service

**Step 2: Event Reception**
- External systems send events to the EventSource
- The EventSource pod receives and validates these events
- Events are published to the EventBus

**Step 3: EventBus Distribution**
- The EventBus (a message broker) receives events from all EventSources
- It acts as a central hub for event distribution
- Multiple Sensors can subscribe to events through the EventBus

**Step 4: Sensor Trigger**
- Sensors watch the EventBus for specific events they're configured to monitor
- When a matching event arrives, the Sensor detects it
- The Sensor evaluates any dependencies and conditions

**Step 5: Action Execution**
- Once conditions are met, the Sensor executes its trigger
- This typically involves creating a Kubernetes resource (usually an Argo Workflow)
- The workflow then runs to completion

### Example: Webhook Event Flow

```
1. External system sends POST request to webhook endpoint
   ↓
2. EventSource webhook listener receives the request
   ↓
3. EventSource publishes event to EventBus
   ↓
4. Sensor subscribes to EventBus and receives the event
   ↓
5. Sensor creates an Argo Workflow resource
   ↓
6. Workflow controller picks up the Workflow and executes it
   ↓
7. Workflow completes and results are available
```

---

## Why Use Argo Events

### Key Advantages

#### 1. **Kubernetes-Native Integration**
Argo Events is built specifically for Kubernetes, making it seamless to integrate with your existing Kubernetes infrastructure, RBAC, and tooling.

#### 2. **Event Source Variety**
Support for numerous event sources means you can build workflows triggered by virtually any external event without custom code.

#### 3. **Decoupling**
EventSources and Sensors decouple event producers from event consumers, making your system more flexible and maintainable.

#### 4. **Scalability**
Built on Kubernetes principles, Argo Events scales horizontally and handles high event volumes efficiently.

#### 5. **Reliability**
Leverages Kubernetes' reliability features and provides message persistence through various EventBus backends.

#### 6. **Cost Efficiency**
Runs within your Kubernetes cluster alongside other workloads, eliminating need for additional event processing infrastructure.

#### 7. **Open Source & Community Driven**
Part of the Argo project ecosystem, with active community support and continuous improvements.

#### 8. **Complex Event Processing**
Sensors can process multiple events, apply filters, and create sophisticated event correlations before triggering actions.

#### 9. **Integration with Argo Workflows**
Seamless integration with Argo Workflows for sophisticated workflow orchestration and pipeline management.

#### 10. **GitOps Compatible**
All resources (EventSource, Sensor, EventBus) are standard Kubernetes CRDs, enabling GitOps workflows and version control.

---

## Installation Guide

### Prerequisites

- Kubernetes cluster (v1.19 or later)
  - Options: minikube, kind, EKS, GKE, AKS, or any managed Kubernetes service
- `kubectl` configured to access your cluster
- Helm (optional, but recommended)
- Internet connectivity to download manifests

### Step 1: Install Argo Workflows

Argo Events works best alongside Argo Workflows for workflow orchestration.

#### Set Version Variable

```bash
ARGO_WORKFLOWS_VERSION="v3.7.0"
```

#### Create Namespace and Deploy

```bash
kubectl create namespace argo
kubectl apply -n argo -f "https://github.com/argoproj/argo-workflows/releases/download/${ARGO_WORKFLOWS_VERSION}/quick-start-minimal.yaml"
```

#### Verify Installation

```bash
kubectl get pods -n argo
kubectl get svc -n argo
```

You should see `argo-controller-manager` and `argo-server` pods running.

### Step 2: Install Argo CLI

#### For Linux

Create and run the installation script:

```bash
#!/bin/bash
ARGO_OS="linux"
ARGO_VERSION="v3.7.0"

# Download the binary
curl -sLO "https://github.com/argoproj/argo-workflows/releases/download/${ARGO_VERSION}/argo-${ARGO_OS}-amd64.gz"

# Unzip
gunzip "argo-${ARGO_OS}-amd64.gz"

# Make binary executable
chmod +x "argo-${ARGO_OS}-amd64"

# Move binary to path
sudo mv "./argo-${ARGO_OS}-amd64" /usr/local/bin/argo

# Test installation
argo version
```

Save this as `install_argo.sh`, make it executable, and run it:

```bash
chmod +x install_argo.sh
./install_argo.sh
```

#### For macOS

```bash
curl -sLO "https://github.com/argoproj/argo-workflows/releases/download/v3.7.0/argo-darwin-amd64.gz"
gunzip argo-darwin-amd64.gz
chmod +x argo-darwin-amd64
sudo mv argo-darwin-amd64 /usr/local/bin/argo
argo version
```

### Step 3: Install Argo Events

#### Create Namespace

```bash
kubectl create namespace argo-events
```

#### Deploy Argo Events Core

```bash
kubectl apply -f https://raw.githubusercontent.com/argoproj/argo-events/stable/manifests/install.yaml
```

#### Install Validating Webhook (Optional but Recommended)

```bash
kubectl apply -f https://raw.githubusercontent.com/argoproj/argo-events/stable/manifests/install-validating-webhook.yaml
```

#### Deploy EventBus

The EventBus is required for Sensors and EventSources to communicate:

```bash
kubectl apply -n argo-events -f https://raw.githubusercontent.com/argoproj/argo-events/stable/examples/eventbus/native.yaml
```

### Step 4: Verify Installation

```bash
kubectl get all -n argo-events
kubectl get pods -n argo-events
```

You should see:
- `argo-events-controller` pod
- EventBus-related pods
- Service resources

### Step 5: Configure RBAC for Workflows

```bash
# Sensor RBAC
kubectl apply -n argo-events -f https://raw.githubusercontent.com/argoproj/argo-events/master/examples/rbac/sensor-rbac.yaml

# Workflow RBAC
kubectl apply -n argo-events -f https://raw.githubusercontent.com/argoproj/argo-events/master/examples/rbac/workflow-rbac.yaml
```

### Step 6: Access Argo Workflows UI

Port-forward the Argo server:

```bash
kubectl -n argo port-forward svc/argo-server 2746:2746 --address=0.0.0.0 &
```

Open your browser to `https://<your-instance-ip>:2746` (note: use HTTPS, not HTTP).

---

## Core Concepts

### EventSource

An EventSource is a Kubernetes Custom Resource Definition (CRD) that listens to a specific event source and exposes those events to the EventBus.

**Key Characteristics:**
- Runs as a pod within the cluster
- Exposes a service for receiving external events (in case of webhooks)
- Validates incoming events
- Publishes validated events to the EventBus
- Can define multiple event listeners on different ports/endpoints

**Example EventSource Manifest:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: EventSource
metadata:
  name: webhook
  namespace: argo-events
spec:
  service:
    ports:
      - port: 12000
        targetPort: 12000
  webhook:
    example:
      port: "12000"
      endpoint: /example
      method: POST
```

### EventBus

The EventBus is a message bus that facilitates communication between EventSources and Sensors.

**Key Characteristics:**
- Receives events from all EventSources
- Distributes events to interested Sensors
- Provides message persistence and ordering
- Can use different backends (native, NATS, Kafka, etc.)
- Manages event filtering and routing

**Deployment Types:**
- **Native EventBus**: Built-in, suitable for small to medium deployments
- **NATS EventBus**: For high throughput and reliability
- **Kafka EventBus**: For complex event processing and long-term retention

### Sensor

A Sensor is a CRD that watches the EventBus for specific events and triggers actions when those events occur.

**Key Characteristics:**
- Subscribes to specific events from the EventBus
- Defines dependencies and conditions for triggering
- Executes triggers when conditions are met
- Can create Kubernetes resources or invoke webhooks
- Supports multiple triggers from a single Sensor

**Example Sensor Manifest:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Sensor
metadata:
  name: webhook
  namespace: argo-events
spec:
  template:
    serviceAccountName: operate-workflow-sa
  dependencies:
    - name: test-dep
      eventSourceName: webhook
      eventName: example
  triggers:
    - template:
        name: webhook-workflow-trigger
        k8s:
          operation: create
          source:
            resource:
              apiVersion: argoproj.io/v1alpha1
              kind: Workflow
              metadata:
                generateName: webhook-
              spec:
                entrypoint: whalesay
                templates:
                  - name: whalesay
                    container:
                      image: alpine:3.16
                      command: ["sh", "-c"]
                      args: ["echo 'Event triggered workflow'"]
```

---

## Practical Example: Webhook to Workflow

This example demonstrates the complete workflow: setting up a webhook EventSource that triggers an Argo Workflow via a Sensor.

### Step 1: Create the EventSource

Save this as `event-source.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: EventSource
metadata:
  name: webhook
  namespace: argo-events
spec:
  service:
    ports:
      - port: 12000
        targetPort: 12000
  webhook:
    example:
      port: "12000"
      endpoint: /example
      method: POST
```

Apply it:

```bash
kubectl apply -f event-source.yaml
```

Verify the EventSource pod and service were created:

```bash
kubectl get pods -n argo-events
kubectl get svc -n argo-events
```

### Step 2: Create the Sensor

Save this as `webhook-sensor.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Sensor
metadata:
  name: webhook
  namespace: argo-events
spec:
  template:
    serviceAccountName: operate-workflow-sa
  dependencies:
    - name: test-dep
      eventSourceName: webhook
      eventName: example
  triggers:
    - template:
        name: webhook-workflow-trigger
        k8s:
          operation: create
          source:
            resource:
              apiVersion: argoproj.io/v1alpha1
              kind: Workflow
              metadata:
                generateName: webhook-
              spec:
                entrypoint: whalesay
                arguments:
                  parameters:
                    - name: message
                      value: hello world
                templates:
                  - name: whalesay
                    inputs:
                      parameters:
                        - name: message
                    container:
                      image: alpine:3.16
                      command: ["sh", "-c"]
                      args: ["echo \"{{inputs.parameters.message}}\"; sleep 1"]
```

Apply it:

```bash
kubectl apply -f webhook-sensor.yaml
```

Verify the Sensor was created:

```bash
kubectl get sensors -n argo-events
kubectl describe sensor webhook -n argo-events
```

### Step 3: Expose the EventSource

Port-forward the EventSource service:

```bash
kubectl -n argo-events port-forward svc/webhook-eventsource-svc 12000:12000 --address=0.0.0.0 &
```

If using a cloud provider, open firewall rules for port 12000.

### Step 4: Trigger the Workflow

Send a webhook request using curl:

```bash
curl -d '{"message":"this is my first webhook"}' \
  -H "Content-Type: application/json" \
  -X POST http://<your-instance-ip>:12000/example
```

Replace `<your-instance-ip>` with your actual instance IP address.

### Step 5: Verify Workflow Execution

Check if the workflow was created:

```bash
kubectl get wf -n argo-events
```

Describe the workflow:

```bash
kubectl describe wf <workflow-name> -n argo-events
```

View the workflow logs:

```bash
kubectl logs -l workflows.argoproj.io/workflow=<workflow-name> -n argo-events
```

View in the Argo UI:

Open `https://<your-instance-ip>:2746` in your browser and navigate to the argo-events namespace.

---

## Advanced Configuration

### 1. Webhook Security with TLS

Enable secure webhooks by providing certificates:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: EventSource
metadata:
  name: webhook-secure
  namespace: argo-events
spec:
  webhook:
    secure-example:
      port: "13000"
      endpoint: "/secure"
      method: "POST"
      serverCertSecret:
        name: webhook-cert
        key: cert.pem
      serverKeySecret:
        name: webhook-cert
        key: key.pem
```

Create the secret with your certificate:

```bash
kubectl create secret generic webhook-cert \
  --from-file=cert.pem=path/to/cert.pem \
  --from-file=key.pem=path/to/key.pem \
  -n argo-events
```

### 2. Multiple Event Listeners on Same EventSource

Run multiple HTTP servers on different ports:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: EventSource
metadata:
  name: multi-webhook
  namespace: argo-events
spec:
  service:
    ports:
      - port: 12000
        targetPort: 12000
      - port: 12001
        targetPort: 12001
  webhook:
    github:
      port: "12000"
      endpoint: /github
      method: POST
    gitlab:
      port: "12001"
      endpoint: /gitlab
      method: POST
```

### 3. Event Filtering in Sensors

Filter events before triggering:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Sensor
metadata:
  name: filtered-sensor
  namespace: argo-events
spec:
  template:
    serviceAccountName: operate-workflow-sa
  dependencies:
    - name: github-event
      eventSourceName: webhook
      eventName: github
      filters:
        data:
          - path: action
            type: string
            value:
              - opened
              - synchronize
  triggers:
    - template:
        name: pr-workflow
        k8s:
          operation: create
          source:
            resource:
              apiVersion: argoproj.io/v1alpha1
              kind: Workflow
              metadata:
                generateName: github-pr-
              spec:
                entrypoint: test
                templates:
                  - name: test
                    container:
                      image: alpine:3.16
                      command: ["echo"]
                      args: ["Running tests for PR"]
```

### 4. Multiple Triggers from One Sensor

Execute multiple actions on a single event:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Sensor
metadata:
  name: multi-trigger-sensor
  namespace: argo-events
spec:
  template:
    serviceAccountName: operate-workflow-sa
  dependencies:
    - name: deployment-event
      eventSourceName: webhook
      eventName: deploy
  triggers:
    - template:
        name: build-trigger
        k8s:
          operation: create
          source:
            resource:
              apiVersion: argoproj.io/v1alpha1
              kind: Workflow
              metadata:
                generateName: build-
              spec:
                entrypoint: build
                templates:
                  - name: build
                    container:
                      image: alpine:3.16
                      command: ["echo"]
                      args: ["Building..."]
    - template:
        name: test-trigger
        k8s:
          operation: create
          source:
            resource:
              apiVersion: argoproj.io/v1alpha1
              kind: Workflow
              metadata:
                generateName: test-
              spec:
                entrypoint: test
                templates:
                  - name: test
                    container:
                      image: alpine:3.16
                      command: ["echo"]
                      args: ["Testing..."]
```

### 5. AWS S3 Event Source

Trigger workflows on S3 bucket events:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: EventSource
metadata:
  name: s3-events
  namespace: argo-events
spec:
  s3:
    mybucket:
      bucket:
        name: my-bucket
      events:
        - s3:ObjectCreated:*
      filter:
        prefix: uploads/
        suffix: .csv
      roleARN: arn:aws:iam::123456789012:role/my-role
      region: us-east-1
```

### 6. CRON Schedule Events

Trigger workflows on schedule:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: EventSource
metadata:
  name: cron-schedule
  namespace: argo-events
spec:
  calendar:
    daily:
      schedule: "0 0 * * *"
      timezone: "UTC"
    hourly:
      schedule: "0 * * * *"
      timezone: "America/New_York"
```

---

## Best Practices

### 1. **RBAC and Security**

Always use proper service accounts and RBAC:

```bash
# Create a dedicated service account
kubectl create serviceaccount sensor-sa -n argo-events

# Create a role with minimal required permissions
kubectl create role workflow-creator -n argo-events \
  --verb=create,get,list,watch \
  --resource=workflows

# Bind the role to the service account
kubectl create rolebinding sensor-workflow-creator \
  --clusterrole=workflow-creator \
  --serviceaccount=argo-events:sensor-sa \
  -n argo-events
```

### 2. **Use EventBus for Scalability**

Deploy an appropriate EventBus backend based on your needs:

```yaml
# For high-throughput scenarios, use NATS EventBus
apiVersion: argoproj.io/v1alpha1
kind: EventBus
metadata:
  name: default
  namespace: argo-events
spec:
  nats:
    native: {}
```

### 3. **Implement Event Filtering**

Always filter events at the Sensor level to reduce unnecessary workflow executions:

```yaml
filters:
  data:
    - path: body.status
      type: string
      value:
        - success
        - completed
```

### 4. **Version Control Everything**

Store all EventSource and Sensor manifests in Git for GitOps workflows:

```bash
git init argo-events-config
cd argo-events-config
git add *.yaml
git commit -m "Initial Argo Events configuration"
git push origin main
```

### 5. **Monitor EventSource and Sensor Pods**

Set up monitoring and alerting for your EventSource and Sensor pods:

```bash
# Check pod status
kubectl get pods -n argo-events -w

# Check pod logs for errors
kubectl logs -n argo-events -l app=event-source

# Describe for detailed error information
kubectl describe pod <pod-name> -n argo-events
```

### 6. **Handle Webhook Retries**

Design your workflows to be idempotent to handle potential duplicate events:

```yaml
templates:
  - name: process-data
    retryStrategy:
      limit: 3
      backoff:
        duration: 5s
        factor: 2
    container:
      image: myimage:latest
      command: ["/app/process.sh"]
```

### 7. **Use Meaningful Names and Labels**

Add descriptive labels and annotations:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: EventSource
metadata:
  name: github-webhook
  namespace: argo-events
  labels:
    app: ci-cd
    source: github
  annotations:
    description: "Listens for GitHub push events"
    owner: devops-team
```

### 8. **Test Webhook Payloads Locally**

Always test webhook payloads before deploying to production:

```bash
# Create a local listener
nc -l 12000

# Send test payload
curl -d '{"test":"data"}' \
  -H "Content-Type: application/json" \
  -X POST http://localhost:12000/example
```

### 9. **Document Your EventSources and Sensors**

Create documentation explaining the purpose of each EventSource and Sensor:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: EventSource
metadata:
  name: deployment-webhook
  namespace: argo-events
  annotations:
    description: "Receives deployment notifications from CI/CD platform"
    owner: "DevOps Team"
    runbook: "https://wiki.company.com/deployment-events"
```

### 10. **Use Namespacing for Multi-Tenant Setups**

Separate EventSources and Sensors by namespace for different teams:

```bash
kubectl create namespace team-a-events
kubectl create namespace team-b-events

# Deploy EventBus in each namespace
kubectl apply -n team-a-events -f eventbus-native.yaml
kubectl apply -n team-b-events -f eventbus-native.yaml
```

---

## Troubleshooting

### EventSource Pod Not Starting

```bash
# Check pod status
kubectl describe pod <eventsource-pod> -n argo-events

# Check logs
kubectl logs <eventsource-pod> -n argo-events

# Check RBAC permissions
kubectl auth can-i create workflows --as=system:serviceaccount:argo-events:default -n argo-events
```

### Webhook Not Triggering Workflows

```bash
# Verify EventBus is running
kubectl get eventbus -n argo-events
kubectl get pods -n argo-events | grep eventbus

# Check Sensor logs
kubectl logs -l app=sensor -n argo-events

# Verify Sensor dependencies
kubectl get sensor <sensor-name> -n argo-events -o yaml
```

### Unable to Connect to Webhook Endpoint

```bash
# Verify port-forward is active
kubectl get port-forward -n argo-events

# Check security groups/firewalls
nc -zv <instance-ip> 12000

# Restart port-forward if needed
kubectl -n argo-events port-forward svc/webhook-eventsource-svc 12000:12000 --address=0.0.0.0
```

---

## Conclusion

Argo Events provides a powerful, Kubernetes-native solution for event-driven workflow automation. By enabling you to react to external events and trigger complex automation chains, it significantly reduces manual intervention and enables sophisticated, scalable event-driven architectures within your Kubernetes clusters.

The combination of EventSources, EventBus, and Sensors creates a flexible framework that integrates with virtually any external system, making Argo Events an essential tool for modern cloud-native environments.