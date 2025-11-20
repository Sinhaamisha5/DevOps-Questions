# Docker Compose - Complete Reference Guide

> **A comprehensive guide covering all Docker Compose concepts, features, and best practices**
> 
> Perfect for DevOps engineers, developers, and interview preparation

---

## Table of Contents

1. [Introduction](#introduction)
2. [What is Docker Compose?](#what-is-docker-compose)
3. [Basic Structure](#basic-structure)
4. [Core Components](#core-components)
5. [Service Configuration](#service-configuration)
6. [Networking](#networking)
7. [Volumes & Storage](#volumes--storage)
8. [Environment Variables](#environment-variables)
9. [Security Features](#security-features)
10. [Advanced Features](#advanced-features)
11. [Commands Reference](#commands-reference)
12. [Best Practices](#best-practices)
13. [Production vs Development](#production-vs-development)
14. [Troubleshooting](#troubleshooting)
15. [Interview Questions](#interview-questions)
16. [Real-World Examples](#real-world-examples)

---

## Introduction

Docker Compose is a tool for defining and running multi-container Docker applications. With Compose, you use a YAML file to configure your application's services, networks, and volumes. Then, with a single command, you create and start all the services from your configuration.

### Key Benefits

- **Simplicity**: Define complex multi-container applications in a single file
- **Reproducibility**: Same configuration works across all environments
- **Isolation**: Services run in isolated containers
- **Scalability**: Easy to scale services up or down
- **Version Control**: Configuration as code

### Use Cases

- Development environments
- Automated testing
- Single-host deployments
- Microservices architecture
- CI/CD pipelines

---

## What is Docker Compose?

### Simple Explanation
Docker Compose is like a recipe book for running multiple containers together. Instead of running 20 separate `docker run` commands, you write one YAML file and start everything with one command.

### Technical Definition
Docker Compose is a tool for defining and running multi-container Docker applications using a declarative YAML configuration file. It orchestrates the creation, networking, and lifecycle management of multiple containers as a single application stack.

### Compose V1 vs V2

| Feature | Compose V1 | Compose V2 |
|---------|-----------|-----------|
| Command | `docker-compose` | `docker compose` |
| Written in | Python | Go (integrated with Docker CLI) |
| Performance | Slower | Faster |
| Version field | Required | Optional |
| Status | Deprecated | Current |

**Migration Command:**
```bash
# Old (V1)
docker-compose up

# New (V2)
docker compose up
```

---

## Basic Structure

### Minimal Compose File

```yaml
services:
  web:
    image: nginx:latest
    ports:
      - "80:80"
```

### Complete Structure

```yaml
version: '3.8'  # Optional in Compose V2

# Reusable configuration blocks
x-logging: &default-logging
  driver: json-file
  options:
    max-size: "10m"
    max-file: "3"

# Networks
networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge

# Volumes
volumes:
  db-data:
  cache-data:

# Configs (Swarm mode)
configs:
  my-config:
    file: ./config.json

# Secrets (Swarm mode)
secrets:
  db-password:
    file: ./db_password.txt

# Services
services:
  web:
    image: nginx:latest
    # ... service configuration ...
    
  api:
    build: ./api
    # ... service configuration ...
```

---

## Core Components

### 1. Services

Services are the containers that make up your application.

```yaml
services:
  web:
    image: nginx:alpine
    container_name: my-web-server
    restart: unless-stopped
    ports:
      - "8080:80"
    environment:
      - ENV=production
    networks:
      - frontend
```

**Key Points:**
- Each service is a container
- Can be built from Dockerfile or pulled from registry
- Services can depend on other services
- Automatically get DNS names matching service names

---

### 2. Networks

Networks enable communication between containers.

```yaml
networks:
  # Default bridge network
  default:
    driver: bridge
  
  # Custom networks
  frontend:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16
  
  backend:
    driver: bridge
    internal: true  # No external access
  
  # External network (created outside compose)
  external-net:
    external: true
    name: my-pre-existing-network
```

**Network Drivers:**
- `bridge`: Default, for single-host networking
- `host`: Remove network isolation, use host networking
- `overlay`: Multi-host networking (Swarm)
- `macvlan`: Assign MAC addresses to containers
- `none`: Disable networking

**Service Discovery Example:**
```yaml
services:
  web:
    image: nginx
    networks:
      - frontend
  
  api:
    image: myapi
    networks:
      - frontend
      - backend
    # Can reach web at: http://web:80
  
  db:
    image: postgres
    networks:
      - backend
    # Cannot reach web (different network)
```

---

### 3. Volumes

Volumes provide persistent storage and data sharing.

```yaml
volumes:
  # Named volume (Docker-managed)
  postgres-data:
    driver: local
  
  # Volume with driver options
  nfs-data:
    driver: local
    driver_opts:
      type: nfs
      o: addr=192.168.1.100,rw
      device: ":/path/to/data"
  
  # External volume
  existing-volume:
    external: true
    name: my-existing-volume

services:
  db:
    image: postgres
    volumes:
      # Named volume
      - postgres-data:/var/lib/postgresql/data
      
      # Bind mount
      - ./config:/etc/postgresql/conf.d:ro
      
      # Anonymous volume
      - /var/lib/postgresql/logs
      
      # tmpfs mount (in-memory)
      - type: tmpfs
        target: /tmp
        tmpfs:
          size: 100M
```

**Volume Types:**

| Type | Syntax | Managed By | Use Case |
|------|--------|------------|----------|
| Named | `volume-name:/path` | Docker | Production data |
| Bind | `./host:/container` | User | Development, config |
| Anonymous | `/container/path` | Docker | Temporary data |
| tmpfs | `type: tmpfs` | Memory | Cache, sensitive data |

**Volume Flags:**
- `:ro` - Read-only
- `:rw` - Read-write (default)
- `:z` - Shared SELinux label
- `:Z` - Private SELinux label
- `:nocopy` - Don't copy data from container

---

## Service Configuration

### Image vs Build

```yaml
services:
  # Use pre-built image
  nginx:
    image: nginx:1.21-alpine
  
  # Build from Dockerfile
  app:
    build:
      context: ./app
      dockerfile: Dockerfile
      args:
        NODE_VERSION: 18
        BUILD_DATE: "2024-01-01"
      cache_from:
        - myapp:latest
      target: production
      labels:
        version: "1.0"
  
  # Build and tag
  api:
    image: myregistry/api:v1
    build:
      context: ./api
```

---

### Ports

```yaml
services:
  web:
    ports:
      # HOST:CONTAINER
      - "8080:80"
      
      # Multiple ports
      - "443:443"
      - "8081:8080"
      
      # Random host port
      - "3000"
      
      # Specify protocol
      - "6379:6379/tcp"
      - "53:53/udp"
      
      # Bind to specific interface
      - "127.0.0.1:8080:80"
      
      # Range of ports
      - "9000-9003:9000-9003"
```

**expose vs ports:**
```yaml
services:
  app:
    expose:
      - "3000"  # Only accessible to linked services
    ports:
      - "3000:3000"  # Accessible from host
```

---

### Environment Variables

```yaml
services:
  app:
    environment:
      # Simple assignment
      - NODE_ENV=production
      
      # From host environment
      - DEBUG
      
      # With default value
      - PORT=${PORT:-3000}
      
      # Required (error if not set)
      - API_KEY=${API_KEY:?API_KEY is required}
    
    # Or use env_file
    env_file:
      - .env
      - .env.local
```

**.env file:**
```bash
NODE_ENV=production
DATABASE_URL=postgres://user:pass@db:5432/mydb
API_KEY=secret123
PORT=3000
```

**Variable Substitution:**
- `${VAR}` - Simple substitution
- `${VAR:-default}` - Use default if not set or empty
- `${VAR-default}` - Use default only if not set
- `${VAR:?error}` - Error if not set or empty
- `${VAR?error}` - Error if not set

---

### Command & Entrypoint

```yaml
services:
  # Override command
  app:
    image: node:18
    command: npm start
    # Or array format
    command: ["npm", "start"]
  
  # Override entrypoint
  custom:
    image: myapp
    entrypoint: /app/custom-entrypoint.sh
    # Or array format
    entrypoint: ["/app/custom-entrypoint.sh"]
  
  # Entrypoint with command
  combined:
    image: myapp
    entrypoint: ["python"]
    command: ["app.py", "--verbose"]
    # Results in: python app.py --verbose
```

**Difference:**
- **ENTRYPOINT**: The main executable
- **CMD**: Default arguments to the entrypoint

---

### Depends On

```yaml
services:
  web:
    image: nginx
    depends_on:
      api:
        condition: service_started
      db:
        condition: service_healthy
  
  api:
    image: myapi
    depends_on:
      - db
      - cache
  
  db:
    image: postgres
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
  
  cache:
    image: redis
```

**Conditions:**
- `service_started` - Wait until service starts
- `service_healthy` - Wait until health check passes
- `service_completed_successfully` - Wait until service exits with 0

**Important:** `depends_on` only controls startup order, not readiness!

---

### Health Checks

```yaml
services:
  web:
    image: nginx
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
  
  db:
    image: postgres
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $$POSTGRES_USER"]
      interval: 10s
      timeout: 5s
      retries: 5
  
  # Disable healthcheck from image
  no-health:
    image: someimage
    healthcheck:
      disable: true
```

**Test Types:**
- `CMD` - Direct command
- `CMD-SHELL` - Shell command (with shell features)

**Health Status:**
- `starting` - Initial grace period
- `healthy` - Check passed
- `unhealthy` - Check failed after retries

---

### Restart Policies

```yaml
services:
  # Never restart
  once:
    image: myapp
    restart: no
  
  # Always restart
  always:
    image: myapp
    restart: always
  
  # Restart on failure
  on-failure:
    image: myapp
    restart: on-failure
    # Or with max attempts
    restart: on-failure:5
  
  # Restart unless manually stopped
  unless-stopped:
    image: myapp
    restart: unless-stopped
```

**Policies:**
- `no` - Never restart (default)
- `always` - Always restart
- `on-failure` - Restart on error
- `on-failure:n` - Restart n times on error
- `unless-stopped` - Always restart unless manually stopped

---

### Resource Limits

```yaml
services:
  app:
    image: myapp
    deploy:
      resources:
        limits:
          cpus: '0.50'      # 50% of one CPU
          memory: 512M
        reservations:
          cpus: '0.25'      # Guaranteed minimum
          memory: 256M
  
  # Legacy format (still works)
  legacy:
    image: myapp
    mem_limit: 512m
    mem_reservation: 256m
    cpus: 0.5
    cpu_shares: 512
```

**Units:**
- Memory: `b`, `k`, `m`, `g` (bytes, kilobytes, megabytes, gigabytes)
- CPU: Decimal (0.5 = 50% of one core)

---

### Logging

```yaml
# Reusable logging config
x-logging: &default-logging
  driver: json-file
  options:
    max-size: "10m"
    max-file: "3"
    tag: "{{.Name}}"

services:
  app:
    image: myapp
    logging: *default-logging
  
  custom-logging:
    image: myapp
    logging:
      driver: syslog
      options:
        syslog-address: "tcp://192.168.0.42:123"
  
  no-logging:
    image: myapp
    logging:
      driver: none
```

**Logging Drivers:**
- `json-file` - Default, JSON logs
- `syslog` - Syslog protocol
- `journald` - Systemd journal
- `gelf` - Graylog Extended Format
- `fluentd` - Fluentd logging
- `awslogs` - AWS CloudWatch Logs
- `splunk` - Splunk logging
- `none` - Disable logging

---

### Labels

```yaml
services:
  web:
    image: nginx
    labels:
      # Documentation
      com.example.description: "Web server"
      com.example.department: "Engineering"
      com.example.version: "1.0.0"
      
      # Traefik routing
      traefik.enable: "true"
      traefik.http.routers.web.rule: "Host(`example.com`)"
      traefik.http.services.web.loadbalancer.server.port: "80"
      
      # Prometheus monitoring
      prometheus.io/scrape: "true"
      prometheus.io/port: "9090"
      prometheus.io/path: "/metrics"
```

**Uses:**
- Documentation and metadata
- Service discovery (Traefik, Consul)
- Monitoring (Prometheus)
- Organization and filtering

---

## Advanced Features

### 1. Profiles

```yaml
services:
  # Always runs
  web:
    image: nginx
  
  # Only with 'debug' profile
  debug-tools:
    image: nicolaka/netshoot
    profiles:
      - debug
    command: sleep infinity
  
  # Only with 'test' profile
  test-runner:
    image: pytest
    profiles:
      - testing
    command: pytest tests/
  
  # Multiple profiles
  monitoring:
    image: prometheus
    profiles:
      - monitoring
      - production
```

**Usage:**
```bash
# Default services only
docker compose up

# With debug profile
docker compose --profile debug up

# Multiple profiles
docker compose --profile debug --profile monitoring up
```

---

### 2. Extension Fields & YAML Anchors

```yaml
# Extension fields (x- prefix)
x-logging: &default-logging
  driver: json-file
  options:
    max-size: "10m"
    max-file: "3"

x-healthcheck: &default-healthcheck
  interval: 30s
  timeout: 10s
  retries: 3
  start_period: 40s

x-deploy: &default-deploy
  resources:
    limits:
      memory: 512M
      cpus: '0.5'

# Common service template
x-app-template: &app-template
  restart: unless-stopped
  logging: *default-logging
  networks:
    - app-network
  deploy: *default-deploy

services:
  # Reuse template
  app1:
    <<: *app-template
    image: app1:latest
    healthcheck:
      <<: *default-healthcheck
      test: ["CMD", "curl", "-f", "http://localhost/health"]
  
  app2:
    <<: *app-template
    image: app2:latest
    healthcheck:
      <<: *default-healthcheck
      test: ["CMD", "wget", "--spider", "http://localhost/ping"]
```

**YAML Features:**
- `&anchor` - Create anchor
- `*anchor` - Reference anchor
- `<<: *anchor` - Merge anchor (objects only)

---

### 3. Multiple Compose Files

**File Structure:**
```
project/
├── docker-compose.yml          # Base configuration
├── docker-compose.override.yml # Auto-loaded overrides
├── docker-compose.dev.yml      # Development overrides
├── docker-compose.prod.yml     # Production overrides
└── docker-compose.test.yml     # Testing overrides
```

**docker-compose.yml (Base):**
```yaml
services:
  web:
    image: nginx
    volumes:
      - ./html:/usr/share/nginx/html
```

**docker-compose.override.yml (Auto-loaded):**
```yaml
services:
  web:
    ports:
      - "8080:80"
    environment:
      - DEBUG=true
```

**docker-compose.prod.yml:**
```yaml
services:
  web:
    deploy:
      replicas: 3
      resources:
        limits:
          memory: 512M
    environment:
      - DEBUG=false
```

**Usage:**
```bash
# Development (auto-loads override)
docker compose up

# Production
docker compose -f docker-compose.yml -f docker-compose.prod.yml up

# Testing
docker compose -f docker-compose.yml -f docker-compose.test.yml up

# Multiple overrides
docker compose -f docker-compose.yml -f prod.yml -f monitoring.yml up
```

---

### 4. Build Arguments vs Environment Variables

```yaml
services:
  app:
    build:
      context: .
      args:
        # Build-time variables
        NODE_VERSION: 18
        BUILD_DATE: "2024-01-01"
        GIT_COMMIT: ${GIT_COMMIT}
    environment:
      # Runtime variables
      - NODE_ENV=production
      - DATABASE_URL=postgres://db:5432/mydb
      - API_KEY=${API_KEY}
```

**Dockerfile:**
```dockerfile
# Build args (used during build)
ARG NODE_VERSION=16
FROM node:${NODE_VERSION}

ARG BUILD_DATE
ARG GIT_COMMIT
LABEL build_date=${BUILD_DATE}
LABEL git_commit=${GIT_COMMIT}

# Environment variables (available at runtime)
ENV NODE_ENV=production
ENV PORT=3000
```

**Key Differences:**

| Feature | Build Args | Environment Variables |
|---------|-----------|----------------------|
| Available | Build time | Runtime |
| Defined with | ARG | ENV |
| Set in compose | build.args | environment |
| Persisted | No | Yes |
| Use case | Versions, build config | App configuration |

---

### 5. Configs & Secrets

**Configs (non-sensitive):**
```yaml
configs:
  nginx-config:
    file: ./nginx.conf
  app-config:
    external: true

services:
  web:
    image: nginx
    configs:
      - source: nginx-config
        target: /etc/nginx/nginx.conf
        mode: 0440
```

**Secrets (sensitive):**
```yaml
secrets:
  db-password:
    file: ./db_password.txt
  api-key:
    external: true

services:
  app:
    image: myapp
    secrets:
      - db-password
      - source: api-key
        target: /run/secrets/api_key
        mode: 0400
```

**Accessing secrets in container:**
```bash
# Secrets are mounted as files
cat /run/secrets/db-password
```

**Best Practices:**
- Use secrets for passwords, keys, certificates
- Never use environment variables for secrets
- Set restrictive permissions (0400, 0440)
- Use external secrets in production

---

## Security Features

### 1. User & Permissions

```yaml
services:
  # Run as non-root user
  app:
    image: myapp
    user: "1000:1000"  # UID:GID
  
  # Run as named user
  app2:
    image: myapp
    user: appuser
  
  # Read-only root filesystem
  secure-app:
    image: myapp
    read_only: true
    tmpfs:
      - /tmp
      - /var/run
```

---

### 2. Linux Capabilities

```yaml
services:
  # Drop all capabilities
  minimal:
    image: myapp
    cap_drop:
      - ALL
  
  # Add specific capabilities
  network-app:
    image: myapp
    cap_drop:
      - ALL
    cap_add:
      - NET_ADMIN
      - NET_RAW
  
  # Common patterns
  web-server:
    image: nginx
    cap_drop:
      - ALL
    cap_add:
      - CHOWN
      - DAC_OVERRIDE
      - SETGID
      - SETUID
      - NET_BIND_SERVICE
```

**Common Capabilities:**
- `NET_ADMIN` - Network configuration
- `NET_RAW` - Raw sockets
- `SYS_ADMIN` - System administration
- `CHOWN` - Change file ownership
- `NET_BIND_SERVICE` - Bind to ports < 1024

---

### 3. Security Options

```yaml
services:
  app:
    image: myapp
    security_opt:
      # Prevent privilege escalation
      - no-new-privileges:true
      
      # AppArmor profile
      - apparmor:docker-default
      
      # Seccomp profile
      - seccomp:./seccomp-profile.json
      
      # SELinux label
      - label:type:container_runtime_t
```

---

### 4. Privileged Mode (Avoid!)

```yaml
services:
  # ⚠️ DANGEROUS - Avoid unless absolutely necessary
  dind:
    image: docker:dind
    privileged: true
    # Only for Docker-in-Docker, testing, etc.
```

**Why avoid:**
- Gives container almost all host capabilities
- Can access all host devices
- Bypasses many security features
- Security risk in production

---

### 5. Resource Constraints

```yaml
services:
  app:
    image: myapp
    
    # Memory limits
    mem_limit: 512m
    mem_reservation: 256m
    mem_swappiness: 0
    
    # CPU limits
    cpus: 0.5
    cpu_shares: 512
    cpu_quota: 50000
    cpu_period: 100000
    
    # PID limit
    pids_limit: 100
    
    # Ulimits
    ulimits:
      nproc: 65535
      nofile:
        soft: 20000
        hard: 40000
    
    # OOM killer adjustment
    oom_score_adj: -500  # Less likely to be killed
```

---

## Networking Advanced

### Network Modes

```yaml
services:
  # Default bridge
  default-net:
    image: nginx
    networks:
      - default
  
  # Host network (no isolation)
  host-net:
    image: nginx
    network_mode: host
  
  # Share network with another container
  shared-net:
    image: sidecar
    network_mode: "service:main-app"
  
  # No network
  isolated:
    image: batch-job
    network_mode: none
  
  # Container network
  linked:
    image: tools
    network_mode: "container:main-app"
```

---

### Custom Network Configuration

```yaml
networks:
  frontend:
    driver: bridge
    driver_opts:
      com.docker.network.bridge.name: frontend-bridge
    ipam:
      driver: default
      config:
        - subnet: 172.20.0.0/16
          gateway: 172.20.0.1
  
  backend:
    driver: bridge
    internal: true  # No external access
    ipam:
      config:
        - subnet: 172.21.0.0/16
  
  # Overlay network (Swarm)
  overlay-net:
    driver: overlay
    attachable: true
```

---

### DNS & Host Configuration

```yaml
services:
  app:
    image: myapp
    
    # Custom DNS servers
    dns:
      - 8.8.8.8
      - 8.8.4.4
    
    # DNS search domains
    dns_search:
      - example.com
      - internal.example.com
    
    # Custom /etc/hosts entries
    extra_hosts:
      - "api.example.com:192.168.1.100"
      - "db.example.com:192.168.1.101"
      - "host.docker.internal:host-gateway"
    
    # Custom hostname
    hostname: my-app-server
    domainname: example.com
```

---

### IPv6 Support

```yaml
networks:
  ipv6-net:
    enable_ipv6: true
    ipam:
      config:
        - subnet: 2001:db8::/64
          gateway: 2001:db8::1

services:
  app:
    image: myapp
    networks:
      ipv6-net:
        ipv6_address: 2001:db8::10
```

---

## Advanced Configuration

### 1. Init System

```yaml
services:
  app:
    image: myapp
    init: true  # Use tini as PID 1
```

**Benefits:**
- Proper signal handling
- Zombie process reaping
- Graceful shutdown

---

### 2. Stop Configuration

```yaml
services:
  app:
    image: myapp
    stop_signal: SIGTERM     # Signal to send
    stop_grace_period: 30s   # Wait time before SIGKILL
```

---

### 3. Platform Specification

```yaml
services:
  # Multi-arch builds
  app:
    image: myapp
    platform: linux/amd64
    # Options: linux/amd64, linux/arm64, linux/arm/v7
```

---

### 4. Device Access

```yaml
services:
  # GPU access
  ml-app:
    image: tensorflow
    runtime: nvidia
    devices:
      - /dev/nvidia0:/dev/nvidia0
  
  # USB device
  hardware-app:
    image: myapp
    devices:
      - /dev/ttyUSB0:/dev/ttyUSB0
  
  # Block device
  storage-app:
    image: myapp
    devices:
      - /dev/sda:/dev/xvda:rwm
```

---

### 5. tmpfs Mounts

```yaml
services:
  app:
    image: myapp
    tmpfs:
      - /tmp
      - /run:size=100M,mode=1777
```

---

### 6. Shared Memory

```yaml
services:
  chrome:
    image: selenium/chrome
    shm_size: '2gb'  # Increase /dev/shm size
```

---

### 7. System Configuration

```yaml
services:
  app:
    image: myapp
    
    # Kernel parameters
    sysctls:
      net.core.somaxconn: 1024
      net.ipv4.tcp_syncookies: 0
    
    # Control groups
    cgroup_parent: my-custom-cgroup
    
    # CPU settings
    cpu_count: 2
    cpu_percent: 50
```

---

### 8. Namespace Sharing

```yaml
services:
  main-app:
    image: myapp
  
  # Share PID namespace
  debug:
    image: debug-tools
    pid: "service:main-app"
  
  # Share IPC namespace
  sidecar:
    image: sidecar
    ipc: "service:main-app"
  
  # Share network namespace
  proxy:
    image: nginx
    network_mode: "service:main-app"
```

---

### 9. Interactive Mode

```yaml
services:
  # For debugging
  debug:
    image: alpine
    stdin_open: true  # -i flag
    tty: true         # -t flag
    command: sh
```

---

### 10. Working Directory

```yaml
services:
  app:
    image: myapp
    working_dir: /app/subdir
```

---

## Commands Reference

### Basic Commands

```bash
# Start services
docker compose up
docker compose up -d                    # Detached mode
docker compose up --build              # Rebuild images
docker compose up --force-recreate     # Force recreate containers
docker compose up --no-deps web        # Start without dependencies

# Stop services
docker compose down
docker compose down -v                 # Remove volumes
docker compose down --rmi all         # Remove images
docker compose stop                    # Stop without removing
docker compose kill                    # Force stop
```

### Build Commands

```bash
# Build images
docker compose build
docker compose build --no-cache       # Fresh build
docker compose build --pull           # Pull latest base images
docker compose build web              # Build specific service
docker compose build --parallel       # Parallel builds
```

### Service Management

```bash
# Start/Stop/Restart
docker compose start
docker compose start web
docker compose stop
docker compose restart
docker compose pause                  # Pause services
docker compose unpause               # Resume services
```

### Scaling

```bash
# Scale services
docker compose up -d --scale web=3
docker compose up -d --scale worker=5 --scale api=2
```

### Information & Logs

```bash
# View status
docker compose ps
docker compose ps -a                  # Include stopped
docker compose top                    # Show processes
docker compose images                 # List images

# Logs
docker compose logs
docker compose logs -f                # Follow logs
docker compose logs --tail=100       # Last 100 lines
docker compose logs web              # Specific service
docker compose logs --since 30m      # Last 30 minutes
docker compose logs --timestamps     # Show timestamps
```

### Execution

```bash
# Execute in running container
docker compose exec web sh
docker compose exec -it web bash
docker compose exec -u root web sh   # As root user

# Run new container
docker compose run web ls
docker compose run --rm test         # Run and remove
docker compose run --no-deps web     # Without dependencies
docker compose run -e DEBUG=1 web    # With env var
```

### Configuration

```bash
# Validate and view config
docker compose config
docker compose config --services     # List services
docker compose config --volumes      # List volumes
docker compose config --quiet        # Validate only
docker compose config --resolve-image-digests  # Show image digests
```

### Profiles

```bash
# Use profiles
docker compose --profile debug up
docker compose --profile test --profile debug up
```

### Multiple Files

```bash
# Use multiple compose files
docker compose -f docker-compose.yml -f prod.yml up
docker compose -f base.yml -f override.yml -f monitoring.yml up
```

### Cleanup

```bash
# Remove stopped containers
docker compose rm
docker compose rm -f                 # Force remove

# Clean everything
docker compose down -v --rmi all
```

### Pull Images

```bash
# Pull images
docker compose pull
docker compose pull web              # Specific service
docker compose pull --ignore-pull-failures
```

### Version & Help

```bash
# Version
docker compose version

# Help
docker compose --help
docker compose up --help
```

---

## Best Practices

### 1. File Organization

```yaml
# Use extension fields for reusable config
x-logging: &logging
  driver: json-file
  options:
    max-size: "10m"

x-healthcheck: &healthcheck
  interval: 30s
  timeout: 10s
  retries: 3

# Clear service organization
services:
  # Frontend tier
  web:
    <<: *app-defaults
    
  # Backend tier
  api:
    <<: *app-defaults
    
  # Data tier
  db:
    <<: *db-defaults
```

### 2. Environment Management

```bash
# .env file (DO NOT COMMIT)
DATABASE_PASSWORD=secret123
API_KEY=abc123

# .env.example (COMMIT THIS)
DATABASE_PASSWORD=changeme
API_KEY=your_api_key_here
```

```yaml
# docker-compose.yml
services:
  app:
    environment:
      - DATABASE_PASSWORD=${DATABASE_PASSWORD:?required}
      - API_KEY=${API_KEY}
```

### 3. Security Checklist

- [ ] Run containers as non-root user
- [ ] Use secrets for sensitive data
- [ ] Drop unnecessary capabilities
- [ ] Enable read-only root filesystem
- [ ] Set resource limits
- [ ] Use health checks
- [ ] Enable security options (no-new-privileges)
- [ ] Avoid privileged mode
- [ ] Use specific image tags (not :latest)
- [ ] Scan images for vulnerabilities

### 4. Production Readiness

```yaml
services:
  app:
    image: myapp:1.2.3           # ✅ Specific version
    # image: myapp:latest        # ❌ Avoid
    
    restart: unless-stopped      # ✅ Auto-restart
    
    deploy:
      resources:
        limits:
          memory: 512M           # ✅ Set limits
          cpus: '0.5'
    
    healthcheck:                 # ✅ Health checks
      test: ["CMD", "curl", "-f", "http://localhost/health"]
      interval: 30s
      timeout: 10s
      retries: 3
    
    logging:                     # ✅ Log management
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
    
    user: "1000:1000"           # ✅ Non-root
    read_only: true             # ✅ Read-only filesystem
    
    security_opt:               # ✅ Security hardening
      - no-new-privileges:true
    
    cap_drop:                   # ✅ Drop capabilities
      - ALL
```

### 5. Development Best Practices

```yaml
services:
  app:
    build:
      context: .
      target: development        # Use multi-stage build
    
    volumes:
      - ./src:/app/src:delegated # Live reload
      - node_modules:/app/node_modules  # Named volume for deps
    
    environment:
      - DEBUG=true
      - HOT_RELOAD=true
    
    ports:
      - "3000:3000"             # Direct port access
```

### 6. Performance Optimization

```yaml
# Layer caching optimization
services:
  app:
    build:
      context: .
      cache_from:
        - myapp:latest
        - myapp:dev
    
    # Volume performance
    volumes:
      - ./src:/app/src:delegated     # macOS/Windows
      - node_modules:/app/node_modules  # Named volume
```

### 7. Naming Conventions

```yaml
# Clear, descriptive names
services:
  frontend-web:              # ✅ Clear
    image: nginx
  
  backend-api:               # ✅ Clear
    image: myapi
  
  postgres-db:               # ✅ Clear
    image: postgres

# Avoid
  web:                       # ❌ Too generic
  app:                       # ❌ Too generic
  db:                        # ❌ Too generic
```

### 8. Documentation

```yaml
# Add comments for complex configurations
services:
  app:
    image: myapp
    # Increased memory for batch processing
    mem_limit: 2g
    
    # Custom DNS for internal service discovery
    dns:
      - 10.0.0.2
    
    # Healthcheck monitors background worker
    healthcheck:
      test: ["CMD", "python", "health_check.py"]
      interval: 30s
```

### 9. .dockerignore

```
# .dockerignore
node_modules
.git
.env
.env.*
*.log
.DS_Store
coverage
.vscode
```

### 10. Version Control

```bash
# Commit
✅ docker-compose.yml
✅ docker-compose.override.yml.example
✅ .env.example
✅ README.md

# Don't commit
❌ .env
❌ docker-compose.override.yml (if contains local config)
❌ *.log
```

---

## Production vs Development

### Development Setup

```yaml
# docker-compose.yml
services:
  web:
    build:
      context: .
      target: development
    volumes:
      - ./src:/app/src:delegated
      - node_modules:/app/node_modules
    environment:
      - NODE_ENV=development
      - DEBUG=true
      - HOT_RELOAD=true
    ports:
      - "3000:3000"
    command: npm run dev

  db:
    image: postgres:14
    ports:
      - "5432:5432"
    volumes:
      - ./data/dev:/var/lib/postgresql/data
    environment:
      - POSTGRES_PASSWORD=devpass
```

### Production Setup

```yaml
# docker-compose.prod.yml
services:
  web:
    image: myregistry/web:1.2.3
    restart: unless-stopped
    deploy:
      replicas: 3
      resources:
        limits:
          memory: 512M
          cpus: '0.5'
    environment:
      - NODE_ENV=production
      - DEBUG=false
    # No ports exposed (behind load balancer)
    healthcheck:
      test: ["CMD", "wget", "--spider", "http://localhost/health"]
      interval: 30s
      timeout: 10s
      retries: 3
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
    security_opt:
      - no-new-privileges:true
    read_only: true
    tmpfs:
      - /tmp

  db:
    image: postgres:14
    restart: unless-stopped
    volumes:
      - postgres-data:/var/lib/postgresql/data
    environment:
      - POSTGRES_PASSWORD=${DB_PASSWORD}
    deploy:
      resources:
        limits:
          memory: 2G
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  postgres-data:
    driver: local
```

### Comparison Table

| Aspect | Development | Production |
|--------|-------------|------------|
| **Image Source** | Built locally | Pre-built, tagged |
| **Volumes** | Bind mounts (code) | Named volumes (data) |
| **Ports** | Exposed to host | Internal only |
| **Restart Policy** | `no` | `unless-stopped` |
| **Resource Limits** | None | Enforced |
| **Environment** | DEBUG=true | DEBUG=false |
| **Health Checks** | Optional | Required |
| **Logging** | Default | Configured + rotation |
| **Security** | Relaxed | Hardened |
| **Secrets** | .env file | External secrets |

---

## Troubleshooting

### Common Issues

#### 1. Container Won't Start

```bash
# Check logs
docker compose logs service-name
docker compose logs -f service-name

# Check if port is already in use
netstat -tulpn | grep PORT
lsof -i :PORT

# Inspect container
docker compose ps
docker inspect CONTAINER_ID
```

#### 2. Network Issues

```bash
# Test connectivity
docker compose exec web ping api
docker compose exec web curl api:8080

# Check networks
docker network ls
docker network inspect PROJECT_default

# DNS resolution
docker compose exec web nslookup api
docker compose exec web cat /etc/hosts
```

#### 3. Volume Issues

```bash
# List volumes
docker volume ls

# Inspect volume
docker volume inspect PROJECT_volume-name

# Check permissions
docker compose exec service ls -la /mount/path
```

#### 4. Build Issues

```bash
# Fresh build
docker compose build --no-cache

# Pull latest base images
docker compose build --pull

# Check Dockerfile
docker compose config
```

#### 5. Performance Issues

```bash
# Check resource usage
docker stats

# Check logs for errors
docker compose logs --tail=100

# Verify health
docker compose ps
```

### Debug Commands

```bash
# Get shell in container
docker compose exec service sh
docker compose exec -u root service bash

# Run diagnostics
docker compose run --rm service python debug.py

# Check environment
docker compose exec service env

# Test without dependencies
docker compose up --no-deps service
```

### Health Check Debugging

```bash
# View health status
docker compose ps

# Manual health check
docker compose exec service curl -f http://localhost/health

# View health check logs
docker inspect --format='{{json .State.Health}}' CONTAINER_ID | jq
```

---

## Real-World Examples

### Example 1: LAMP Stack

```yaml
version: '3.8'

services:
  apache:
    image: php:8.1-apache
    ports:
      - "80:80"
    volumes:
      - ./src:/var/www/html
    depends_on:
      - mysql
    networks:
      - lamp-network

  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: rootpass
      MYSQL_DATABASE: mydb
      MYSQL_USER: user
      MYSQL_PASSWORD: password
    volumes:
      - mysql-data:/var/lib/mysql
    networks:
      - lamp-network

  phpmyadmin:
    image: phpmyadmin/phpmyadmin
    ports:
      - "8080:80"
    environment:
      PMA_HOST: mysql
      PMA_PORT: 3306
    depends_on:
      - mysql
    networks:
      - lamp-network

networks:
  lamp-network:

volumes:
  mysql-data:
```

### Example 2: MERN Stack

```yaml
version: '3.8'

services:
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    ports:
      - "3000:3000"
    environment:
      - REACT_APP_API_URL=http://localhost:5000
    volumes:
      - ./frontend/src:/app/src
      - node_modules_frontend:/app/node_modules
    depends_on:
      - backend

  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    ports:
      - "5000:5000"
    environment:
      - MONGO_URI=mongodb://mongo:27017/mydb
      - JWT_SECRET=${JWT_SECRET}
    volumes:
      - ./backend/src:/app/src
      - node_modules_backend:/app/node_modules
    depends_on:
      - mongo

  mongo:
    image: mongo:6
    ports:
      - "27017:27017"
    volumes:
      - mongo-data:/data/db
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: password

volumes:
  mongo-data:
  node_modules_frontend:
  node_modules_backend:
```

### Example 3: Microservices with Observability

```yaml
version: '3.8'

x-logging: &default-logging
  driver: json-file
  options:
    max-size: "10m"
    max-file: "3"

services:
  # Frontend
  web:
    image: nginx:alpine
    ports:
      - "80:80"
    depends_on:
      - api
      - auth
    logging: *default-logging

  # API Gateway
  api:
    build: ./api-gateway
    environment:
      - AUTH_SERVICE=http://auth:8080
      - USER_SERVICE=http://users:8080
      - ORDER_SERVICE=http://orders:8080
    depends_on:
      - auth
      - users
      - orders
    logging: *default-logging

  # Auth Service
  auth:
    build: ./auth-service
    environment:
      - DATABASE_URL=postgres://postgres:password@postgres-auth:5432/auth
      - REDIS_URL=redis://redis:6379
    depends_on:
      - postgres-auth
      - redis
    logging: *default-logging

  # Users Service
  users:
    build: ./users-service
    environment:
      - DATABASE_URL=postgres://postgres:password@postgres-users:5432/users
    depends_on:
      - postgres-users
    logging: *default-logging

  # Orders Service
  orders:
    build: ./orders-service
    environment:
      - DATABASE_URL=postgres://postgres:password@postgres-orders:5432/orders
      - KAFKA_BROKER=kafka:9092
    depends_on:
      - postgres-orders
      - kafka
    logging: *default-logging

  # Databases
  postgres-auth:
    image: postgres:14
    environment:
      POSTGRES_DB: auth
      POSTGRES_PASSWORD: password
    volumes:
      - postgres-auth-data:/var/lib/postgresql/data

  postgres-users:
    image: postgres:14
    environment:
      POSTGRES_DB: users
      POSTGRES_PASSWORD: password
    volumes:
      - postgres-users-data:/var/lib/postgresql/data

  postgres-orders:
    image: postgres:14
    environment:
      POSTGRES_DB: orders
      POSTGRES_PASSWORD: password
    volumes:
      - postgres-orders-data:/var/lib/postgresql/data

  # Cache
  redis:
    image: redis:alpine
    volumes:
      - redis-data:/data

  # Message Queue
  kafka:
    image: confluentinc/cp-kafka:latest
    environment:
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092

  zookeeper:
    image: confluentinc/cp-zookeeper:latest
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181

  # Observability
  prometheus:
    image: prom/prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus

  grafana:
    image: grafana/grafana
    ports:
      - "3001:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana-data:/var/lib/grafana
    depends_on:
      - prometheus

  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "16686:16686"
      - "14268:14268"

volumes:
  postgres-auth-data:
  postgres-users-data:
  postgres-orders-data:
  redis-data:
  prometheus-data:
  grafana-data:
```

### Example 4: Development Environment with Hot Reload

```yaml
version: '3.8'

services:
  # Next.js Frontend
  frontend:
    build:
      context: ./frontend
      target: development
    volumes:
      - ./frontend:/app
      - /app/node_modules
      - /app/.next
    environment:
      - NODE_ENV=development
    ports:
      - "3000:3000"
    command: npm run dev

  # Node.js Backend
  backend:
    build:
      context: ./backend
      target: development
    volumes:
      - ./backend:/app
      - /app/node_modules
    environment:
      - NODE_ENV=development
      - DATABASE_URL=postgres://postgres:password@db:5432/dev
    ports:
      - "4000:4000"
    command: npm run dev
    depends_on:
      - db

  # Database
  db:
    image: postgres:14
    environment:
      POSTGRES_PASSWORD: password
      POSTGRES_DB: dev
    ports:
      - "5432:5432"
    volumes:
      - postgres-data:/var/lib/postgresql/data

  # Database UI
  adminer:
    image: adminer
    ports:
      - "8080:8080"
    depends_on:
      - db

volumes:
  postgres-data:
```

---

## Interview Questions & Answers

### Q1: What is Docker Compose?

**Answer:**
"Docker Compose is a tool for defining and running multi-container Docker applications using a YAML configuration file. It allows me to define services, networks, and volumes in a single file and manage the entire application lifecycle with simple commands like `docker compose up` and `docker compose down`. It's particularly useful for development environments and single-host deployments."

---

### Q2: Explain the difference between `docker-compose up` and `docker-compose start`

**Answer:**
"`docker compose up` creates and starts containers from scratch based on the compose file. It reads the configuration and applies any changes. `docker compose start` only starts existing stopped containers without recreating them. I use `up` for initial deployment or when configuration changes, and `start` to restart stopped services without rebuilding."

---

### Q3: How do containers communicate in Docker Compose?

**Answer:**
"Containers on the same Docker network communicate using service names as hostnames. Docker provides built-in DNS resolution. For example, if I have a 'web' service and an 'api' service, the web container can reach the API at `http://api:8080`. This service discovery is automatic when services are on the same network."

---

### Q4: What's the purpose of depends_on?

**Answer:**
"`depends_on` controls startup order but doesn't guarantee readiness. It ensures dependent services start first, but the dependency might not be fully initialized. For production, I use health checks with `condition: service_healthy` to wait for services to be actually ready. For example:
```yaml
depends_on:
  db:
    condition: service_healthy
```
This waits for the database health check to pass before starting the application."

---

### Q5: How do you handle secrets in Docker Compose?

**Answer:**
"For development, I use `.env` files with proper `.gitignore`. For production, I use Docker secrets (in Swarm mode) or integrate with external secret managers like HashiCorp Vault or AWS Secrets Manager. I never commit secrets to version control and avoid using environment variables for sensitive data. Instead, I mount secrets as files:
```yaml
secrets:
  db-password:
    external: true
services:
  app:
    secrets:
      - db-password
```
The secret is available at `/run/secrets/db-password` in the container."

---

### Q6: Explain resource limits in Docker Compose

**Answer:**
"Resource limits prevent containers from consuming excessive resources. I set them using the deploy section:
```yaml
deploy:
  resources:
    limits:
      memory: 512M
      cpus: '0.5'
    reservations:
      memory: 256M
      cpus: '0.25'
```
Limits are the maximum allowed, reservations are guaranteed minimums. This is crucial for production stability and prevents one container from affecting others."

---

### Q7: What are the differences between volumes and bind mounts?

**Answer:**
"Named volumes are Docker-managed and portable, stored in Docker's storage directory. I use them for production data:
```yaml
volumes:
  postgres-data:
services:
  db:
    volumes:
      - postgres-data:/var/lib/postgresql/data
```

Bind mounts link host directories directly and are great for development:
```yaml
services:
  app:
    volumes:
      - ./src:/app/src
```

Named volumes survive container removal, support drivers for remote storage, and are more performant on Mac/Windows."

---

### Q8: How do you use multiple compose files?

**Answer:**
"I use multiple compose files for different environments. The base file contains common configuration, and environment-specific files override or extend it:
```bash
# Development (auto-loads override)
docker compose up

# Production
docker compose -f docker-compose.yml -f docker-compose.prod.yml up
```

Docker Compose merges the files in order, with later files overriding earlier ones. This keeps configuration DRY and manageable across environments."

---

### Q9: Explain Docker Compose profiles

**Answer:**
"Profiles allow conditional service activation. Services with profiles only start when explicitly requested:
```yaml
services:
  app:
    image: myapp  # Always runs
  debug:
    image: debug-tools
    profiles: [debug]  # Only with --profile debug
```
I use this for optional services like debugging tools, test runners, or monitoring that shouldn't run by default but are available when needed."

---

### Q10: What security best practices do you follow with Docker Compose?

**Answer:**
"I follow several security practices:
1. Run containers as non-root users
2. Drop unnecessary Linux capabilities
3. Use secrets instead of environment variables for sensitive data
4. Set resource limits to prevent DoS
5. Use read-only root filesystems where possible
6. Enable health checks
7. Use specific image tags, not :latest
8. Enable security options like no-new-privileges
9. Scan images for vulnerabilities
10. Keep base images updated

Example:
```yaml
services:
  app:
    user: '1000:1000'
    read_only: true
    cap_drop: [ALL]
    security_opt:
      - no-new-privileges:true
```"

---

### Q11: How do you debug issues in Docker Compose?

**Answer:**
"I follow a systematic approach:
1. Check service status: `docker compose ps`
2. View logs: `docker compose logs -f service-name`
3. Exec into container: `docker compose exec service sh`
4. Test connectivity: `docker compose exec web ping api`
5. Check environment: `docker compose exec service env`
6. Validate config: `docker compose config`
7. Review health checks: `docker inspect container-id`
8. Check networks: `docker network inspect network-name`

I also use debugging services with the same network to diagnose issues without modifying the main application."

---

### Q12: Explain build args vs environment variables

**Answer:**
"Build args are only available during image building, set in Dockerfile with ARG. Environment variables are available at runtime, set with ENV. In compose:
```yaml
services:
  app:
    build:
      args:
        NODE_VERSION: 18    # Build time only
    environment:
      - NODE_ENV=production  # Runtime
```

I use build args for versions and build metadata that don't need to be in the running container. Environment variables configure the application at runtime and can be changed without rebuilding."

---

### Q13: What's the difference between expose and ports?

**Answer:**
"`expose` documents which ports the container listens on but doesn't publish them to the host. It only makes ports accessible to linked services:
```yaml
expose:
  - '3000'  # Only other containers can access
```

`ports` actually publishes ports to the host machine:
```yaml
ports:
  - '8080:3000'  # Host:Container - accessible from host
```

I use `expose` for documentation and internal services, `ports` for services that need external access."

---

### Q14: How do you optimize Docker Compose for development?

**Answer:**
"For development, I optimize for fast feedback:
1. Use bind mounts for hot reload
2. Use named volumes for dependencies to avoid rebuilding
3. Expose ports for direct access
4. Enable debug mode
5. Use build targets for development-specific steps

Example:
```yaml
services:
  app:
    build:
      target: development
    volumes:
      - ./src:/app/src:delegated
      - node_modules:/app/node_modules
    environment:
      - DEBUG=true
    ports:
      - '3000:3000'
```"

---

### Q15: What's your approach to production deployment with Compose?

**Answer:**
"While Docker Compose is primarily for development and single-host deployments, for production I:
1. Use pre-built, tagged images (not build)
2. Set restart policies (unless-stopped)
3. Configure resource limits
4. Implement health checks
5. Set up proper logging with rotation
6. Use external secrets management
7. Enable security hardening
8. Use specific image versions

For true production at scale, I'd migrate to Kubernetes or Docker Swarm for orchestration, high availability, and rolling updates. Compose is excellent for smaller deployments or as a foundation before scaling."

---

## Summary Cheat Sheet

### Essential Commands
```bash
docker compose up -d              # Start detached
docker compose down              # Stop and remove
docker compose logs -f           # Follow logs
docker compose ps                # List containers
docker compose exec service sh   # Shell into container
docker compose build            # Build images
docker compose pull             # Pull images
docker compose restart          # Restart services
```

### Key Concepts
- **Services**: Containers that make up your application
- **Networks**: Enable container communication
- **Volumes**: Persistent storage
- **Depends_on**: Control startup order
- **Health checks**: Monitor service health
- **Profiles**: Conditional service activation

### Best Practices Checklist
- [ ] Use specific image versions
- [ ] Set resource limits
- [ ] Implement health checks
- [ ] Run as non-root user
- [ ] Use secrets for sensitive data
- [ ] Enable restart policies
- [ ] Configure logging
- [ ] Use .dockerignore
- [ ] Version control compose files
- [ ] Document complex configurations

---

## Additional Resources

### Official Documentation
- [Docker Compose Documentation](https://docs.docker.com/compose/)
- [Compose File Reference](https://docs.docker.com/compose/compose-file/)
- [Docker Security](https://docs.docker.com/engine/security/)

### Tools
- [Docker Desktop](https://www.docker.com/products/docker-desktop)
- [Portainer](https://www.portainer.io/) - Container management UI
- [Lazydocker](https://github.com/jesseduffield/lazydocker) - Terminal UI

### Validators
- `docker compose config` - Validate compose file
- [Compose Validator](https://www.composerize.com/) - Online validator

---

## Conclusion

Docker Compose is an essential tool for modern application development and deployment. This guide covers everything from basic concepts to advanced features and production best practices.

**Key Takeaways:**
1. Compose simplifies multi-container applications
2. YAML configuration is declarative and version-controlled
3. Service discovery is automatic within networks
4. Security and resource management are critical
5. Different configurations for development vs production
6. Proper monitoring and health checks ensure reliability

Master these concepts, and you'll be well-prepared for any Docker Compose scenario in development, production, or interviews.

---

**Document Version:** 1.0  
**Last Updated:** 2024  
**Author:** DevOps Learning Resources

---

*This document is a comprehensive reference. Bookmark it and refer back as needed!*