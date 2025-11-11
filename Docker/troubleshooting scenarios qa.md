# Troubleshooting & Scenario-Based Interview Questions

## Real-World Problem Solving for Infrastructure Engineers

---

## Table of Contents
1. [Jenkins Troubleshooting](#jenkins-troubleshooting)
2. [Docker Troubleshooting](#docker-troubleshooting)
3. [CI/CD Pipeline Issues](#cicd-pipeline-issues)
4. [Performance Problems](#performance-problems)
5. [Security Incidents](#security-incidents)
6. [Scenario-Based Questions](#scenario-based-questions)

---

## JENKINS TROUBLESHOOTING

### Q1: Jenkins build is failing intermittently. How do you debug this?

**Answer:**

**Systematic Debugging Approach:**

**Step 1: Gather Information**
```bash
# Check console output
curl -u user:token \
  https://jenkins.com/job/myapp/123/consoleText

# Check build history
# Look for patterns:
# - Same stage failing?
# - Time of day correlation?
# - Specific agents?
# - After certain commits?
```

**Step 2: Identify Pattern**
```
Analyze last 20 builds:
✅✅❌✅✅✅❌✅✅❌  Pattern?

Check correlations:
- Time: Fails during peak hours? (Resource contention)
- Agent: Always same agent? (Agent-specific issue)
- Stage: Same stage failing? (Stage-specific problem)
- Commit: After specific changes? (Code issue)
```

**Step 3: Common Causes & Solutions**

**Cause 1: Resource Contention**
```groovy
// Problem: Multiple builds competing for resources
// Symptom: "Cannot allocate memory", timeouts

// Solution: Throttle concurrent builds
properties([
    disableConcurrentBuilds(),
    buildDiscarder(logRotator(numToKeepStr: '10'))
])

// Or: Use different agents
agent {
    label 'high-memory-agent'
}
```

**Cause 2: Network Timeouts**
```groovy
// Problem: Dependency downloads timeout intermittently
// Symptom: "Connection timed out", "Read timed out"

// Solution 1: Add retries
retry(3) {
    sh 'npm install'
}

// Solution 2: Increase timeout
timeout(time: 30, unit: 'MINUTES') {
    sh 'mvn clean install'
}

// Solution 3: Use local cache
sh '''
    mvn -Dmaven.repo.local=./.m2/repository \
        clean install
'''
```

**Cause 3: Race Conditions**
```groovy
// Problem: Tests fail when run in parallel
// Symptom: Random test failures

// Solution 1: Disable parallel tests
sh 'npm test -- --runInBand'

// Solution 2: Isolate test data
sh 'npm test -- --testNamePattern="${BUILD_NUMBER}"'

// Solution 3: Use unique ports
environment {
    TEST_PORT = "${8000 + BUILD_NUMBER.toInteger() % 1000}"
}
```

**Cause 4: Flaky Tests**
```groovy
// Problem: Tests pass/fail randomly
// Symptom: "Expected 200, got 503", timing issues

// Temporary Fix (NOT recommended):
retry(3) {
    sh 'npm test'
}

// Proper Fix:
// 1. Identify flaky test
sh 'npm test -- --detectOpenHandles'

// 2. Add proper wait conditions
// Instead of:
sleep(5000)  // ❌ Flaky

// Do:
await waitForCondition(
    () => service.isReady(),
    timeout: 30000
)  // ✅ Reliable
```

**Cause 5: External Service Dependency**
```groovy
// Problem: Builds fail when external service is down
// Symptom: "Connection refused", "Service unavailable"

// Solution: Health check before build
stage('Pre-flight Check') {
    steps {
        script {
            def services = ['database', 'cache', 'api']
            services.each { service ->
                def healthy = sh(
                    returnStatus: true,
                    script: "curl -f http://${service}:8080/health"
                )
                if (healthy != 0) {
                    error "${service} is not healthy"
                }
            }
        }
    }
}
```

**Cause 6: Disk Space**
```bash
# Problem: Agent running out of disk space
# Symptom: "No space left on device"

# Check disk usage
df -h

# Solution: Clean workspace
pipeline {
    options {
        skipDefaultCheckout()
    }
    stages {
        stage('Checkout') {
            steps {
                cleanWs()
                checkout scm
            }
        }
    }
    post {
        always {
            cleanWs()
        }
    }
}
```

**Debugging Tools:**

**1. Enable Debug Logging:**
```groovy
pipeline {
    options {
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }
    stages {
        stage('Debug Build') {
            steps {
                // Verbose output
                sh 'set -x'
                sh 'npm install --verbose'
                
                // Show environment
                sh 'env | sort'
                
                // Show system info
                sh 'df -h'
                sh 'free -m'
                sh 'docker ps -a'
            }
        }
    }
}
```

**2. Reproduce Locally:**
```bash
# Start Jenkins agent container locally
docker run -it --rm \
    -v $(pwd):/workspace \
    jenkins/agent:latest \
    /bin/bash

# Run build commands manually
cd /workspace
npm install
npm test
```

**3. Add Monitoring:**
```groovy
stage('Monitor Build') {
    steps {
        script {
            // Track build duration
            def startTime = System.currentTimeMillis()
            
            sh 'npm test'
            
            def duration = System.currentTimeMillis() - startTime
            echo "Tests completed in ${duration}ms"
            
            if (duration > 300000) {  // 5 minutes
                echo "WARNING: Tests are slow!"
            }
        }
    }
}
```

**4. Capture Artifacts on Failure:**
```groovy
post {
    failure {
        script {
            // Capture system state
            sh '''
                echo "=== System Info ===" > debug.log
                df -h >> debug.log
                free -m >> debug.log
                docker ps -a >> debug.log
                docker logs $(docker ps -lq) >> debug.log 2>&1 || true
            '''
            
            // Archive for investigation
            archiveArtifacts artifacts: 'debug.log,test-results/**/*'
        }
    }
}
```

**Real-World Example:**

**Problem:** Build failing 20% of the time during npm install
```
Investigation:
1. Checked console logs: "ETIMEOUT" errors
2. Pattern: Only during 9am-5pm (peak hours)
3. Root cause: Network saturation from concurrent builds
```

**Solution:**
```groovy
pipeline {
    // Limit concurrent builds
    options {
        throttleJobProperty(
            maxConcurrentPerNode: 2,
            maxConcurrentTotal: 5
        )
    }
    
    stages {
        stage('Install Dependencies') {
            steps {
                // Use local npm cache
                sh '''
                    npm config set cache $(pwd)/.npm-cache
                    npm ci --prefer-offline
                '''
            }
        }
    }
}

// Result: 0% failure rate
```

**Prevention Strategies:**

```groovy
// 1. Resource limits
pipeline {
    agent {
        kubernetes {
            yaml '''
                spec:
                  containers:
                  - name: build
                    resources:
                      requests:
                        memory: "1Gi"
                        cpu: "1"
                      limits:
                        memory: "2Gi"
                        cpu: "2"
            '''
        }
    }
}

// 2. Timeouts
timeout(time: 30, unit: 'MINUTES') {
    sh 'long-running-command'
}

// 3. Health checks
waitUntil {
    script {
        def r = sh(
            returnStatus: true,
            script: 'curl http://localhost:8080/health'
        )
        return r == 0
    }
}

// 4. Proper cleanup
post {
    always {
        sh 'docker-compose down -v'
        cleanWs()
    }
}
```

**Example Response:**
"I start by analyzing build history to identify patterns - whether it's time-based, agent-specific, or stage-specific. Recently we had intermittent npm install failures that only occurred during business hours. By analyzing logs, I found ETIMEOUT errors indicating network congestion. I implemented local npm caching and throttled concurrent builds to 5. This reduced failure rate from 20% to 0%. I also added retry logic with exponential backoff for network operations and set up monitoring alerts for build durations exceeding normal ranges."

---

## DOCKER TROUBLESHOOTING

### Q2: Container keeps crashing with exit code 137. What's the issue?

**Answer:**

**Exit Code 137 = Out of Memory (OOM Killed)**

Exit code 137 means the container was killed by the OS due to memory exhaustion.
- Signal 9 (SIGKILL) = 128 + 9 = 137
- OS kernel OOM killer terminated the process

**Debugging Steps:**

**1. Verify It's OOM:**
```bash
# Check container exit code
docker inspect container_name --format='{{.State.ExitCode}}'
# Output: 137

# Check for OOM in logs
docker logs container_name
# Look for: "Killed" message

# Check system logs
sudo dmesg | grep -i "out of memory"
sudo dmesg | grep -i oom
# Output: "Memory cgroup out of memory: Killed process..."

# Check Docker events
docker events --filter 'event=oom'
```

**2. Check Memory Usage:**
```bash
# Current memory usage
docker stats container_name --no-stream

# Output:
CONTAINER   CPU %   MEM USAGE / LIMIT     MEM %
myapp       50%     1.5GiB / 2GiB        75%

# Memory limit set on container
docker inspect container_name --format='{{.HostConfig.Memory}}'
# Output: 2147483648 (2GB in bytes)
```

**3. Identify Memory Leak:**
```bash
# Monitor memory over time
while true; do
    docker stats container_name --no-stream
    sleep 5
done

# Output showing increasing memory:
# MEM %: 50% → 60% → 70% → 85% → Crashed (137)
```

**Solutions:**

**Solution 1: Increase Memory Limit**
```bash
# If application legitimately needs more memory
docker run -d \
    --name myapp \
    --memory="4g" \              # Hard limit
    --memory-swap="4g" \         # Total memory (RAM + swap)
    --memory-reservation="2g" \  # Soft limit
    myapp:latest

# Docker Compose
version: '3.8'
services:
  app:
    image: myapp:latest
    deploy:
      resources:
        limits:
          memory: 4G
        reservations:
          memory: 2G
```

**Solution 2: Fix Memory Leak**
```javascript
// Node.js Example - Memory Leak
// ❌ BAD: Global array grows indefinitely
const logs = [];
app.use((req, res, next) => {
    logs.push(req.url);  // Memory leak!
    next();
});

// ✅ GOOD: Bounded cache
const LRU = require('lru-cache');
const logs = new LRU({ max: 1000 });
app.use((req, res, next) => {
    logs.set(Date.now(), req.url);
    next();
});
```

**Solution 3: Enable OOM Killer Protection**
```bash
# Adjust OOM score (lower = less likely to be killed)
docker run -d \
    --name myapp \
    --oom-score-adj=-500 \  # Range: -1000 to 1000
    myapp:latest

# Disable OOM killer (risky!)
docker run -d \
    --name myapp \
    --oom-kill-disable \
    --memory="4g" \        # Must set limit!
    myapp:latest
```

**Solution 4: Memory Profiling**

**Node.js:**
```bash
# Start with memory profiling
docker run -d \
    -e NODE_OPTIONS="--max-old-space-size=2048 --heap-prof" \
    myapp:latest

# Use heapdump
npm install heapdump
# In code:
const heapdump = require('heapdump');
heapdump.writeSnapshot('/tmp/heap.heapsnapshot');
```

**Java:**
```bash
# Start with memory profiling
docker run -d \
    -e JAVA_OPTS="-Xmx2g -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp" \
    myapp:latest

# Analyze heap dump
docker cp container_name:/tmp/java_pid123.hprof ./
# Use Eclipse MAT or VisualVM to analyze
```

**Python:**
```python
# Use memory_profiler
from memory_profiler import profile

@profile
def memory_intensive_function():
    data = [i for i in range(10000000)]
    return data

# Run with:
# python -m memory_profiler script.py
```

**Solution 5: Optimize Application**
```javascript
// Example: Process large files in streams
// ❌ BAD: Loads entire file in memory
const fs = require('fs');
const data = fs.readFileSync('large-file.txt', 'utf8');
// OOM if file > available memory

// ✅ GOOD: Stream processing
const fs = require('fs');
const stream = fs.createReadStream('large-file.txt');
stream.on('data', (chunk) => {
    processChunk(chunk);  // Process in chunks
});
```

**Monitoring & Alerts:**

**Prometheus + Grafana:**
```yaml
# Alert on high memory
groups:
- name: docker_alerts
  rules:
  - alert: HighContainerMemory
    expr: |
      container_memory_usage_bytes / container_spec_memory_limit_bytes > 0.9
    for: 5m
    annotations:
      summary: "Container {{ $labels.name }} using >90% memory"
```

**Docker Stats Export:**
```bash
# Monitor all containers
docker stats --format "table {{.Name}}\t{{.MemUsage}}\t{{.MemPerc}}"
```

**Prevention:**

**1. Set Memory Limits Always:**
```yaml
# Kubernetes
resources:
  limits:
    memory: "2Gi"
  requests:
    memory: "1Gi"

# Docker Compose
deploy:
  resources:
    limits:
      memory: 2G
```

**2. Use Health Checks:**
```dockerfile
# Dockerfile
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1
```

**3. Implement Graceful Shutdown:**
```javascript
// Handle SIGTERM for graceful shutdown
process.on('SIGTERM', async () => {
    console.log('SIGTERM received, shutting down gracefully');
    
    // Stop accepting new requests
    server.close();
    
    // Close database connections
    await db.close();
    
    // Flush caches
    cache.clear();
    
    process.exit(0);
});
```

**Common Scenarios:**

**Scenario 1: Memory Leak in Cache**
```javascript
// Problem: Cache grows unbounded
const cache = {};
app.get('/user/:id', (req, res) => {
    if (!cache[req.params.id]) {
        cache[req.params.id] = fetchUser(req.params.id);
    }
    res.json(cache[req.params.id]);
});

// Solution: Use LRU cache with max size
const LRU = require('lru-cache');
const cache = new LRU({ max: 1000, maxAge: 1000 * 60 * 5 });
```

**Scenario 2: Large Batch Processing**
```python
# Problem: Loading entire dataset in memory
data = database.query("SELECT * FROM large_table")  # OOM!
process(data)

# Solution: Process in batches
batch_size = 1000
offset = 0
while True:
    batch = database.query(
        f"SELECT * FROM large_table LIMIT {batch_size} OFFSET {offset}"
    )
    if not batch:
        break
    process(batch)
    offset += batch_size
```

**Scenario 3: Event Listener Leak**
```javascript
// Problem: Event listeners not removed
app.get('/subscribe', (req, res) => {
    eventEmitter.on('data', (data) => {
        res.write(data);  // Listener never removed!
    });
});

// Solution: Remove listeners
app.get('/subscribe', (req, res) => {
    const handler = (data) => res.write(data);
    eventEmitter.on('data', handler);
    
    req.on('close', () => {
        eventEmitter.removeListener('data', handler);
    });
});
```

**Example Response:**
"Exit code 137 indicates the container was killed by the OS due to out-of-memory. I would first verify by checking dmesg logs for OOM killer messages and docker stats to see memory usage trends. Then I'd investigate whether it's a legitimate need for more memory or a memory leak. Recently, we had a Node.js service with this issue - I used heapdump to capture memory snapshots and found an unbounded cache growing. We implemented an LRU cache with a 1000-item limit, which solved the issue. I also added monitoring alerts for when containers exceed 80% memory usage."

---

*[File continues with more troubleshooting scenarios...]*

## SCENARIO-BASED QUESTIONS

### S1: Your production deployment just failed and customers are reporting errors. Walk me through your response.

**Answer:**

**Incident Response Framework:**

**Phase 1: Immediate Response (0-5 minutes)**

```bash
# 1. Acknowledge the incident
echo "Incident detected at $(date)" | tee -a incident.log

# 2. Check system status
kubectl get pods -n production
docker ps -a
systemctl status myapp

# 3. Quick triage - Is it widespread?
curl https://api.company.com/health
# Check monitoring dashboards
# Check error rates in Grafana/Datadog

# 4. Communicate
slack_notify "#incidents" "🚨 Production issue detected. Investigating..."

# 5. Decide: Rollback or Fix Forward?
```

**Decision Tree:**
```
Is it a critical outage?
├── Yes → Immediate Rollback
│   └── Can we rollback safely?
│       ├── Yes → Execute rollback
│       └── No → Fix forward with hotfix
└── No → Fix forward
    └── Deploy fix to canary first
```

**Phase 2: Rollback (5-10 minutes)**

```bash
# Option 1: Kubernetes Rollback
kubectl rollout undo deployment/myapp -n production

# Verify rollback
kubectl rollout status deployment/myapp -n production

# Option 2: Roll back to previous version
kubectl set image deployment/myapp \
    myapp=myregistry/myapp:v1.2.3 \
    -n production

# Option 3: Scale down bad pods, scale up good ones
kubectl scale deployment/myapp-new --replicas=0 -n production
kubectl scale deployment/myapp-old --replicas=10 -n production
```

**Phase 3: Verification (10-15 minutes)**

```bash
# Check health
for i in {1..60}; do
    curl -f https://api.company.com/health && break
    sleep 5
done

# Check error rates
curl "http://grafana/api/dashboards/error-rate?from=now-15m"

# Run smoke tests
./smoke-tests.sh production

# Verify customer reports stopped
slack_notify "#incidents" "✅ Rollback complete. Monitoring..."
```

**Phase 4: Root Cause Analysis (After Stabilization)**

```bash
# Collect logs
kubectl logs deployment/myapp -n production --previous > failed-deployment.log

# Check what changed
git diff v1.2.3 v1.2.4

# Review deployment events
kubectl get events -n production --sort-by='.lastTimestamp'

# Check monitoring
# - What metrics spiked?
# - When did it start?
# - Were there warnings?
```

**Phase 5: Post-Mortem**

```markdown
# Incident Report: Production Deployment Failure

## Timeline
- 14:00: Deployment started (v1.2.4)
- 14:05: Error rate spiked from 0.1% to 15%
- 14:07: Incident detected
- 14:10: Rollback initiated
- 14:12: Rollback complete
- 14:15: System stable

## Impact
- Duration: 15 minutes
- Affected users: ~5% (est. 10,000 users)
- Failed requests: ~50,000
- Revenue impact: $5,000 (estimated)

## Root Cause
Database migration in v1.2.4 added new NOT NULL column
without default value, causing INSERT statements to fail.

## Why It Wasn't Caught
- Staging database already had test data
- Migration worked fine on staging
- Didn't test with production-like empty tables

## Action Items
1. [ ] Add migration testing with empty database (Owner: Alice, Due: 2 days)
2. [ ] Improve canary deployment (gradual rollout) (Owner: Bob, Due: 1 week)
3. [ ] Add automated rollback on error rate threshold (Owner: Charlie, Due: 1 week)
4. [ ] Update runbook with this scenario (Owner: All, Due: Today)

## Lessons Learned
- Always test migrations with production-like data
- Canary deployments would have limited impact
- Monitoring caught it quickly
- Rollback process worked well
```

**Communication Strategy:**

```
Incident Start:
├── Internal: Slack #incidents
│   "🚨 Production issue detected. Error rate at 15%. Investigating."
├── Status Page: Update
│   "We're experiencing elevated error rates. Investigating."
└── Customer Support: Alert
    "Heads up: Customers may report errors. We're working on it."

Rollback Complete:
├── Internal: Slack #incidents
│   "✅ Rollback complete. System stable. Error rate back to 0.1%."
├── Status Page: Update
│   "Issue resolved. All systems operational."
└── Customer Support: Update
    "Issue resolved. Monitor for residual reports."

Post-Incident:
├── Internal: Post-Mortem Meeting
├── Status Page: Detailed Explanation
└── If major: Customer Email (Apology + Explanation)
```

**Prevention for Next Time:**

```yaml
# 1. Automated Canary Deployment
deployment:
  strategy:
    canary:
      steps:
        - setWeight: 10    # 10% traffic
        - pause: {duration: 5m}
        - analysis:
            metrics:
              - name: error-rate
                threshold: 1%
                action: rollback
        - setWeight: 50
        - pause: {duration: 5m}
        - setWeight: 100

# 2. Automated Rollback
monitoring:
  rules:
    - alert: HighErrorRate
      expr: error_rate > 5%
      for: 2m
      actions:
        - trigger_rollback
        - notify_oncall

# 3. Better Testing
test:
  - unit_tests
  - integration_tests
  - e2e_tests
  - migration_tests:  # New!
      - test_with_empty_db
      - test_with_production_snapshot
```

**Example Response:**
"First, I'd immediately check monitoring dashboards to assess severity and scope. If it's critical, I'd execute a rollback without hesitation - we have kubectl rollout undo automation. While the rollback is happening, I'd notify the team via Slack and update our status page. Once stable, I'd gather logs from the failed deployment, analyze what changed, and start a post-mortem. In a recent incident, we deployed code that failed on production data but passed staging. We rolled back in 3 minutes and later implemented better staging data practices plus automated canary deployments that would have caught it at 10% traffic."

---

**End of Troubleshooting & Scenarios Guide**