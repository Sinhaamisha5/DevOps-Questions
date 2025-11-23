# OPA Conftest: Docker Security Best Practices Guide

## Table of Contents
1. [Overview](#overview)
2. [Interview Case Study](#interview-case-study)
3. [What is OPA & Conftest?](#what-is-opa--conftest)
4. [Docker Security Best Practices](#docker-security-best-practices)
5. [Installation](#installation)
6. [Rego Language Basics](#rego-language-basics)
7. [Creating Security Policies](#creating-security-policies)
8. [Integration with Jenkins](#integration-with-jenkins)
9. [Practical Examples](#practical-examples)
10. [Common Interview Questions](#common-interview-questions)

---

## Overview

OPA Conftest is a powerful tool for enforcing security best practices in Docker containerization through policy-as-code. This comprehensive guide covers implementing container security scanning in your CI/CD pipeline using OPA Conftest alongside Trivy.

---

## Interview Case Study

### "How did you implement container security best practices in your Dockerfile?"

**Challenge:**
Our development team was creating Docker images without consistent security standards. We had no enforcement mechanism for:
- Using non-root users
- Avoiding the `latest` tag
- Preventing secrets in environment variables
- Using trusted base images
- Following Docker best practices

This meant insecure images were being deployed to production consistently.

**Solution:**
I implemented OPA Conftest to statically analyze every Dockerfile in our CI/CD pipeline before building images.

**Technical Approach:**

1. **Policy Creation:** Developed Rego-based security policies to check 8+ Docker best practices
2. **Pipeline Integration:** Added OPA Conftest as a mandatory gate in Jenkins before Docker build
3. **Automated Remediation:** Provided developers with clear error messages and fix guidance

**Results:**

- **100% compliance** with security policies in all new Dockerfiles
- **Zero container drift** - consistent security standards across 50+ microservices
- **Reduced manual reviews** by 70% - no more ad-hoc security checks
- **Prevented 8 security issues** that would have reached production

**Real Example from Organization:**

Our initial Dockerfile had three policy violations:
```dockerfile
FROM openjdk:17    # ❌ Non-trusted base image
RUN apt-get upgrade  # ❌ Upgrading packages
ADD app.jar /app/   # ❌ Using ADD instead of COPY
# ❌ No USER specified (runs as root)
```

After fixing:
```dockerfile
FROM public.ecr.aws/docker/library/openjdk:21-ea-17-oraclelinux7
RUN apt-get update && apt-get install -y --no-install-recommends git
RUN groupadd -r k8s-pipeline && useradd -r -g k8s-pipeline k8s-pipeline
COPY app.jar /home/k8s-pipeline/app.jar
USER k8s-pipeline
```

**Verification:**
```bash
# Before: 3/8 policies failed
# After: 8/8 policies passed
```

**Key Metrics:**
- **Policy pass rate:** 0% → 100%
- **Security vulnerabilities related to Dockerfile:** 8 → 0
- **Manual security review time:** 2 hours/image → 0
- **Developer time to fix violations:** 15-30 minutes average

**Business Value:**
- Compliance with CIS Docker Benchmark
- Reduced attack surface
- Faster security audits
- Improved team security awareness

---

## What is OPA & Conftest?

### Open Policy Agent (OPA)

OPA is an open-source, general-purpose policy engine that unifies policy enforcement across your stack.

**Key Characteristics:**
- Language-agnostic policy enforcement
- Written in Rego (a declarative language)
- Can be applied to any JSON/YAML configuration
- Decouples policy from application logic

### Conftest

Conftest is a utility built on top of OPA that helps you write tests against structured configuration data using static analysis.

**Supports:**
- Dockerfiles
- Kubernetes manifests
- Terraform code
- CloudFormation templates
- JSON/YAML configurations
- Any structured configuration data

### How They Work Together

```
Dockerfile
    ↓
Conftest (utility)
    ↓
OPA Engine (processes Rego policies)
    ↓
Policy Results (pass/fail)
    ↓
CI/CD Pipeline (block or allow)
```

---

## Docker Security Best Practices

### 1. Use Specific Base Image Tags (Not `latest`)

**Bad:**
```dockerfile
FROM python:latest
```

**Good:**
```dockerfile
FROM python:3.11-slim-bullseye
```

**Why:** `latest` tag is unpredictable and can introduce breaking changes or vulnerabilities.

### 2. Use Minimal Base Images

**Bad:**
```dockerfile
FROM ubuntu:22.04
```

**Good:**
```dockerfile
FROM alpine:3.18
```

**Why:** Smaller images = fewer packages = smaller attack surface. Alpine is only 5MB vs Ubuntu's 70MB.

### 3. Use COPY Instead of ADD

**Bad:**
```dockerfile
ADD https://example.com/file.tar.gz /app/
ADD ./local.tar.gz /app/
```

**Good:**
```dockerfile
COPY ./app.jar /app/
RUN curl -O https://example.com/file.tar.gz
```

**Why:** COPY is explicit and preferred for local file copying. ADD is more complex with automatic extraction.

### 4. Run Containers as Non-Root User

**Bad:**
```dockerfile
FROM openjdk:17
COPY app.jar /app/
CMD ["java", "-jar", "app.jar"]
# Runs as root by default
```

**Good:**
```dockerfile
FROM openjdk:17
RUN groupadd -r appuser && useradd -r -g appuser appuser
COPY app.jar /app/
USER appuser
CMD ["java", "-jar", "app.jar"]
```

**Why:** Root compromise = full container compromise. Non-root limits damage.

### 5. Avoid Storing Secrets in Environment Variables

**Bad:**
```dockerfile
ENV DATABASE_PASSWORD=supersecret123
ENV API_KEY=sk-12345678
```

**Good:**
```dockerfile
# Use Docker secrets or external secret management
# Pass at runtime, don't embed in image
```

**Why:** Secrets in Dockerfiles are visible in image history and can be extracted.

### 6. Don't Use `apt-get upgrade`

**Bad:**
```dockerfile
RUN apt-get update && apt-get upgrade -y
```

**Good:**
```dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends git
```

**Why:** Upgrade can break base image, introduce compatibility issues, and bloat images.

### 7. Use Trusted/Official Base Images

**Bad:**
```dockerfile
FROM randomuser/openjdk:17
```

**Good:**
```dockerfile
FROM public.ecr.aws/docker/library/openjdk:17
FROM docker.io/library/openjdk:17
```

**Why:** Official images are maintained, scanned, and secure.

### 8. Minimize Layers and Use Multi-Stage Builds

**Bad:**
```dockerfile
FROM maven:3.8.1
COPY . /src
RUN cd /src && mvn clean package
CMD ["java", "-jar", "target/app.jar"]
```

**Good:**
```dockerfile
FROM maven:3.8.1 as builder
COPY . /src
RUN cd /src && mvn clean package

FROM openjdk:17-slim
COPY --from=builder /src/target/app.jar /app/
USER appuser
CMD ["java", "-jar", "/app/app.jar"]
```

**Why:** Multi-stage builds reduce final image size by excluding build tools.

---

## Installation

### Option 1: Install as Binary (Linux)

```bash
# Download the latest release
wget https://github.com/open-policy-agent/conftest/releases/download/v0.45.0/conftest_0.45.0_Linux_x86_64.tar.gz

# Extract
tar xzf conftest_0.45.0_Linux_x86_64.tar.gz

# Move to PATH
sudo mv conftest /usr/local/bin/

# Verify installation
conftest --version
```

### Option 2: Install via Package Manager (macOS)

```bash
brew install conftest
```

### Option 3: Use Docker Image

```bash
docker run --rm -v $(pwd):/workspace openpolicyagent/conftest:latest test Dockerfile -p policy.rego
```

### Option 4: Install on Ubuntu/Debian

```bash
sudo apt-get update
sudo apt-get install -y conftest
```

---

## Rego Language Basics

Rego is OPA's declarative language for writing policies. Here are the basics:

### Basic Syntax

```rego
# Simple rule that allows if condition is met
allow {
    input.user.role == "admin"
}

# Rule that denies
deny {
    input.age < 18
}

# Rule with multiple conditions (AND logic)
safe_image {
    input.image != "latest"
    input.user != "root"
}

# Rule with OR logic
acceptable_user {
    input.user == "admin"
} {
    input.user == "appuser"
}
```

### Common Functions

```rego
# String operations
contains(string, substring)          # Check if string contains substring
startswith(string, prefix)           # Check if string starts with prefix
endswith(string, suffix)             # Check if string ends with suffix
split(string, delimiter)             # Split string

# Array operations
in(array)                            # Check if value is in array
count(array)                         # Get array length

# Logical operators
and                                  # AND
or                                   # OR
not                                  # NOT
```

### Example Policy Rules

```rego
# Check if base image is not latest
package main

deny[msg] {
    image := input.from[0].base_image
    image == "latest"
    msg := "Image tag should be explicit and not latest"
}

# Check if running as root
deny[msg] {
    not input.user
    msg := "User should be specified and not be root"
}

# Check for secrets in environment variables
deny[msg] {
    env := input.env[_]
    contains(lower(env.key), "password")
    msg := sprintf("Password found in environment variable: %s", [env.key])
}
```

---

## Creating Security Policies

### Complete Rego Policy File: `opa-docker-security.rego`

```rego
package main

# Define policies
deny[msg] {
    input.from[0].base_image == "latest"
    msg := "Do not use 'latest' tag for base image"
}

deny[msg] {
    contains(input.from[0].base_image, "/")
    not contains(input.from[0].base_image, ":")
    msg := "Base image should be from trusted registry with explicit tag"
}

deny[msg] {
    input.run[_].cmd_shell contains "apt-get upgrade"
    msg := "Do not upgrade packages using apt-get upgrade. Use apt-get install instead"
}

deny[msg] {
    input.add[_]
    msg := "Use COPY instead of ADD for local files"
}

deny[msg] {
    not input.user
    msg := "Container must specify a USER which is not root"
}

deny[msg] {
    input.user == "root"
    msg := "Container should not run as root user"
}

deny[msg] {
    input.user == "0"
    msg := "Container should not run as UID 0 (root)"
}

deny[msg] {
    input.run[_].cmd_shell contains "sudo"
    msg := "Do not use sudo within Dockerfile"
}

deny[msg] {
    input.env[_].value
    contains(lower(input.env[_].key), "password")
    msg := sprintf("Secrets should not be stored in environment variable: %s", [input.env[_].key])
}

deny[msg] {
    input.env[_].value
    contains(lower(input.env[_].key), "secret")
    msg := sprintf("Secrets should not be stored in environment variable: %s", [input.env[_].key])
}

deny[msg] {
    input.env[_].value
    contains(lower(input.env[_].key), "token")
    msg := sprintf("Tokens should not be stored in environment variable: %s", [input.env[_].key])
}
```

### Policy Explanation

| Policy | Check | Why It Matters |
|--------|-------|----------------|
| No `latest` tag | Base image tag is explicit | `latest` can introduce breaking changes |
| Trusted base images | Image is from official registry | Reduces malicious image risk |
| No `apt-get upgrade` | Prevents package upgrades | Upgrades can break compatibility |
| Use COPY, not ADD | Instructions don't use ADD | COPY is explicit and preferred |
| Specify USER | Container has non-root user | Limits damage from container compromise |
| No UID 0 | USER is not 0 or "root" | Root user has full container control |
| No sudo | Commands don't use sudo | Unnecessary elevation in containers |
| No secrets in ENV | Environment variables don't contain secrets | Secrets visible in image history |

---

## Integration with Jenkins

### Add OPA Conftest to Pipeline

Add this to your `Jenkinsfile` in the **Vulnerability Scan - Docker** stage:

```groovy
stage('Vulnerability Scan - Docker') {
  steps {
    parallel(
      "Dependency Scan": {
        sh "mvn dependency-check:check"
      },
      "Trivy Scan": {
        sh "bash trivy-docker-image-scan.sh"
      },
      "OPA Conftest": {
        sh """
          docker run --rm -v $(pwd):/workspace \
            openpolicyagent/conftest:latest test \
            -p opa-docker-security.rego \
            Dockerfile
        """
      }
    )
  }
}
```

### Updated Jenkinsfile

```groovy
pipeline {
  agent any
  
  options {
    buildDiscarder(logRotator(numToKeepStr: '5'))
  }
  
  stages {
    stage('Git Checkout') {
      steps {
        checkout scm
      }
    }
    
    stage('Unit Tests') {
      steps {
        sh "mvn clean test"
      }
    }
    
    stage('Vulnerability Scan - Docker') {
      steps {
        parallel(
          "Dependency Scan": {
            sh "mvn dependency-check:check"
          },
          "Trivy Scan": {
            sh "bash trivy-docker-image-scan.sh"
          },
          "OPA Conftest": {
            sh """
              docker run --rm -v $(pwd):/workspace \
                openpolicyagent/conftest:latest test \
                -p opa-docker-security.rego \
                Dockerfile
            """
          }
        )
      }
    }
    
    stage('Docker Build') {
      steps {
        sh "docker build -t myapp:${BUILD_NUMBER} ."
      }
    }
    
    stage('Docker Push') {
      steps {
        sh "docker push myapp:${BUILD_NUMBER}"
      }
    }
  }
}
```

---

## Practical Examples

### Example 1: Vulnerable Dockerfile

**Dockerfile (FAILS policies):**
```dockerfile
FROM python:latest

ENV DATABASE_PASSWORD=mypassword123
ENV API_KEY=sk-1234567890

RUN apt-get update && apt-get upgrade -y

ADD https://example.com/requirements.txt /app/
COPY app.py /app/

WORKDIR /app
CMD ["python", "app.py"]
```

**Policy Violations:**
```
FAIL - Do not use 'latest' tag for base image
FAIL - Secrets should not be stored in environment variable: DATABASE_PASSWORD
FAIL - Secrets should not be stored in environment variable: API_KEY
FAIL - Do not upgrade packages using apt-get upgrade
FAIL - Use COPY instead of ADD for local files
FAIL - Container must specify a USER which is not root
```

**Fixed Dockerfile (PASSES all policies):**
```dockerfile
FROM python:3.11-slim

RUN apt-get update && apt-get install -y --no-install-recommends curl

RUN groupadd -r appuser && useradd -r -g appuser appuser

COPY requirements.txt /app/
COPY app.py /app/

WORKDIR /app
USER appuser

CMD ["python", "app.py"]
```

**Policy Results:**
```
✓ All 8 policies passed
```

### Example 2: Java Application Dockerfile

**Before:**
```dockerfile
FROM openjdk:17
COPY target/app.jar /app.jar
ENTRYPOINT ["java", "-jar", "/app.jar"]
```

**After:**
```dockerfile
FROM public.ecr.aws/docker/library/openjdk:21-ea-17-oraclelinux7

# Create non-root user
RUN groupadd -r pipeline && useradd -r -g pipeline pipeline

# Copy application
COPY target/app.jar /home/pipeline/app.jar

# Set ownership
RUN chown -R pipeline:pipeline /home/pipeline

# Use non-root user
USER pipeline

# Run application
ENTRYPOINT ["java", "-jar", "/home/pipeline/app.jar"]
```

### Example 3: Multi-Stage Build

**Secure Multi-Stage Dockerfile:**
```dockerfile
# Stage 1: Build
FROM maven:3.8.1 AS builder
COPY . /src
RUN cd /src && mvn clean package -DskipTests

# Stage 2: Runtime
FROM openjdk:21-ea-17-oraclelinux7

# Create user
RUN groupadd -r appuser && useradd -r -g appuser appuser

# Copy from builder
COPY --from=builder /src/target/app.jar /app/app.jar

# Set working directory
WORKDIR /app

# Use non-root user
USER appuser

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
  CMD java -cp app.jar com.example.HealthCheck || exit 1

# Run
ENTRYPOINT ["java", "-jar", "app.jar"]
```

**Verification:**
```bash
conftest test -p opa-docker-security.rego Dockerfile
# Output: 8 tests passed
```

---

## Verifying Container Security

### Verify Non-Root User in Running Container

```bash
# Get the pod
kubectl get pods

# Execute command inside container
kubectl exec -it <pod-name> -- id

# Expected output (non-root user):
# uid=100(k8s-pipeline) gid=101(k8s-pipeline) groups=101(k8s-pipeline)

# NOT expected (root user):
# uid=0(root) gid=0(root) groups=0(root)
```

### Check Docker Image Layers

```bash
# Inspect image
docker image inspect myapp:1.0

# Check user in config
docker inspect myapp:1.0 | grep -A 5 User

# Expected: "User": "k8s-pipeline"
```

---

## Common Interview Questions

### Q1: "Why use OPA Conftest for Docker security?"

**Answer:**
"I use OPA Conftest because it provides:

1. **Policy as Code** - Security policies are version-controlled and reviewable
2. **Early Detection** - Catches issues before image build, saving time and resources
3. **Consistency** - Enforces same security standards across all Dockerfiles
4. **Flexibility** - Easy to add/modify policies as security requirements evolve
5. **Automation** - Integrates seamlessly in CI/CD without manual reviews
6. **Auditability** - Policy violations are logged and traceable

It complements Trivy (vulnerability scanning) by enforcing best practices proactively."

### Q2: "How do you handle false positives or legitimate exceptions?"

**Answer:**
"I handle exceptions in two ways:

1. **Comment Out Policies** - For specific policies that don't apply to your organization, comment them out in the Rego file. For example, we commented out the trusted base image check because we already validate images with Trivy.

2. **Add Conftest Ignore** - Create `.conftest.rego` to allow specific exceptions:

```rego
# Allow specific images with justification
allow[msg] {
    input.from[0].base_image == "legacy-app:1.0"
    msg := "Legacy image approved by security team - expires 2024-12-31"
}
```

3. **Conditional Policies** - Write policies that check context:

```rego
deny[msg] {
    input.from[0].base_image == "latest"
    input.labels["exception"] != "approved"
    msg := "latest tag not allowed unless exception=approved"
}
```

All exceptions are documented with expiration dates and business justification."

### Q3: "How do you manage policy updates across multiple teams?"

**Answer:**
"We use a centralized policy repository:

1. **Central Repository** - Store `opa-docker-security.rego` in shared organization repository
2. **Version Control** - Tag policy versions (v1.0, v1.1, etc.)
3. **Communication** - Send PR notifications when policies change
4. **Gradual Rollout** - Phase in new policies with warning period
5. **Training** - Update team documentation when policies change

For example, when we added the 'no secrets in ENV' policy, we:
- Created a GitHub issue explaining the change
- Sent team notification 2 weeks before enforcement
- Provided examples of how to fix violations
- Only then enforced it in CI/CD"

### Q4: "What's the difference between OPA Conftest and Trivy?"

**Answer:**
"Great question. They serve different purposes:

| Aspect | OPA Conftest | Trivy |
|--------|--------------|-------|
| **Purpose** | Enforce best practices | Find vulnerabilities |
| **Scope** | Dockerfile structure | Base image + dependencies |
| **Method** | Static analysis of config | Scan CVE database |
| **Stage** | Before build | Before push |
| **Example** | Detects root user in Dockerfile | Finds CVE-2023-XXXX in OpenJDK |

We use both - Conftest prevents bad practices, Trivy finds actual vulnerabilities. Together they provide comprehensive security."

### Q5: "Can you write a custom policy for your specific use case?"

**Answer:**
"Absolutely. For example, we needed a policy ensuring all Docker images are tagged with team name for cost allocation. Here's the custom policy:

```rego
deny[msg] {
    not input.labels.team
    msg := "Image must have a 'team' label for cost tracking"
}

deny[msg] {
    team := input.labels.team
    not team in [\"backend\", \"frontend\", \"data\", \"infra\"]
    msg := sprintf(\"Unknown team label: %s. Use: backend, frontend, data, or infra\", [team])
}
```

This ensures:
- Every image has team label
- Only approved teams can build images
- Cost tracking is automated

We run this alongside standard policies for defense-in-depth."

### Q6: "How do you track and report on policy compliance?"

**Answer:**
"We track compliance through multiple mechanisms:

1. **Build Pipeline Metrics:**
   - Failed builds due to OPA Conftest violations
   - Pass rate over time (target: 100%)

2. **Weekly Reports:**
   - Number of policy violations found
   - Time to remediate
   - Most common violations

3. **Dashboard:**
   - Real-time compliance visualization
   - Trends over months
   - Team-wise compliance breakdown

4. **Alerts:**
   - Slack notification if compliance drops below 95%
   - Escalation if violation not fixed in 24 hours

For example, 3 months after implementation:
- Month 1: 78% compliance (lots of violations)
- Month 2: 92% compliance (team learning)
- Month 3: 100% compliance (best practice established)

This data helps justify security tool investments to leadership."

### Q7: "What happens when OPA Conftest fails in your pipeline?"

**Answer:**
"When OPA Conftest detects violations:

1. **Immediate Feedback:**
   - Build stops before Docker build
   - Developer sees detailed error message
   - Specific policy rule that failed is shown

2. **Developer Actions:**
   - Review the Dockerfile
   - Fix the violation (e.g., change FROM to specific tag)
   - Commit and push
   - Pipeline automatically re-runs

3. **Escalation:**
   - If developer thinks policy is wrong, they comment on PR
   - Security team reviews exception request
   - If approved, add temporary exception with justification and expiration date
   - Exception must be approved before merge

4. **Tracking:**
   - All violations and exceptions logged in Jira
   - Monthly review of patterns

Example: Developer used `python:latest`. Policy failed. Developer changed to `python:3.11-slim`. Policy passed. Pipeline continues. Image built and deployed safely."

### Q8: "How do you ensure developers understand and follow Dockerfile best practices?"

**Answer:**
"I use a multi-pronged approach:

1. **Documentation:**
   - Created runbook: 'Dockerfile Security Best Practices'
   - Included policy explanations and fixes
   - Added before/after examples

2. **Training:**
   - 30-minute workshop on Docker security
   - Showed real examples from our codebase
   - Practiced fixing policy violations

3. **Real-time Feedback:**
   - OPA Conftest error messages are clear and actionable
   - Include 'how to fix' guidance in each policy message
   - Link to documentation

4. **Champions Program:**
   - Designated Docker security champions in each team
   - They help teammates fix violations

5. **Incentives:**
   - Highlighted teams with 100% compliance
   - Featured in monthly security newsletter

Result: Compliance went from 78% to 100% in 3 months through awareness and enablement."

---

## Best Practices Summary

### Policy Writing
- Keep policies focused and simple
- One security concern per policy
- Clear, actionable error messages
- Document the 'why' for each policy

### Integration
- Run before Docker build (fail-fast approach)
- Run in parallel with other security scans (Trivy, Dependency-Check)
- Integrate with code review process
- Provide fix guidance automatically

### Compliance
- Start permissive, gradually enforce
- Allow exceptions with expiration dates
- Track violations and exceptions
- Review policies quarterly

### DevOps Culture
- Make policies visible and understandable
- Train developers on security practices
- Celebrate compliance improvements
- Involve security in policy decisions

---

## Troubleshooting

### Issue: Conftest command not found

**Solution:**
```bash
# Check if Docker is available
docker --version

# Use Docker image instead
docker run --rm -v $(pwd):/workspace \
  openpolicyagent/conftest:latest test \
  -p opa-docker-security.rego \
  Dockerfile
```

### Issue: Policy file not found

**Solution:**
```bash
# Ensure policy file is in working directory
ls -la opa-docker-security.rego

# Or specify full path
conftest test -p /path/to/opa-docker-security.rego Dockerfile
```

### Issue: All policies pass but shouldn't

**Solution:**
```bash
# Verify policy syntax
conftest test -p policy.rego Dockerfile -v

# Check if input is being parsed correctly
# Add debug output to policy
```

### Issue: False positives in policies

**Solution:**
```bash
# Add conditional checks
deny[msg] {
    condition1
    condition2
    condition3  # Add more conditions to reduce false positives
}

# Or use exceptions
```

---

## Resources

- **OPA Documentation:** https://www.openpolicyagent.org/docs/
- **Conftest Repository:** https://github.com/open-policy-agent/conftest
- **Docker Best Practices:** https://docs.docker.com/develop/dev-best-practices/
- **CIS Docker Benchmark:** https://www.cisecurity.org/cis-benchmarks/

---

## Summary

OPA Conftest enables **policy-as-code** security enforcement for Docker images through:

1. **Preventive Controls** - Catch issues before deployment
2. **Consistency** - Same standards across all Dockerfiles
3. **Automation** - No manual security reviews needed
4. **Auditability** - Policy compliance is traceable
5. **Flexibility** - Easy to customize for organization needs

Combined with Trivy vulnerability scanning, OPA Conftest provides comprehensive container security covering both best practices and actual vulnerabilities.

Implement OPA Conftest today to shift left security in your container pipeline!