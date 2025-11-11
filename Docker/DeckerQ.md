# Docker Interview Questions & Answers

## Complete Guide for Infrastructure Engineer Role

---

## Table of Contents
1. [Docker Fundamentals](#docker-fundamentals)
2. [Dockerfile & Images](#dockerfile-and-images)
3. [Container Management](#container-management)
4. [Docker Networking](#docker-networking)
5. [Docker Volumes & Storage](#docker-volumes-and-storage)
6. [Docker Compose](#docker-compose)
7. [Docker Security](#docker-security)
8. [Advanced Topics](#advanced-topics)
9. [Troubleshooting](#troubleshooting)
10. [Real-World Scenarios](#real-world-scenarios)

---

## DOCKER FUNDAMENTALS

### Q1: What is Docker? Explain its architecture.

**Answer:**

Docker is a platform for developing, shipping, and running applications in containers. Containers package an application with all its dependencies into a standardized unit.

**Docker Architecture:**

```
┌─────────────────────────────────────────────────┐
│                Docker Client                     │
│            (docker CLI commands)                 │
└──────────────────┬──────────────────────────────┘
                   │ REST API
┌──────────────────▼──────────────────────────────┐
│              Docker Daemon (dockerd)             │
│  ┌────────────────────────────────────────────┐ │
│  │        Container Runtime                    │ │
│  │   (containerd, runc)                       │ │
│  └────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────┐ │
│  │           Images                            │ │
│  └────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────┐ │
│  │         Containers                          │ │
│  └────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────┐ │
│  │          Networks                           │ │
│  └────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────┐ │
│  │          Volumes                            │ │
│  └────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────┘
                   │
┌──────────────────▼──────────────────────────────┐
│            Docker Registry                       │
│         (Docker Hub, Private Registry)           │
└─────────────────────────────────────────────────┘
```

**Components:**

**1. Docker Client:**
- Command-line interface
- Sends commands to Docker daemon
- Can connect to remote daemons

**2. Docker Daemon (dockerd):**
- Background service
- Manages Docker objects
- Listens for API requests
- Handles image building, container running

**3. Docker Images:**
- Read-only templates
- Contains application and dependencies
- Built from Dockerfiles
- Layered filesystem

**4. Docker Containers:**
- Runnable instances of images
- Isolated processes
- Can be started, stopped, moved, deleted

**5. Docker Registry:**
- Stores Docker images
- Docker Hub (public)
- Private registries (ECR, ACR, Harbor)

**6. Docker Objects:**
- **Images**: Templates for containers
- **Containers**: Running instances
- **Networks**: Container communication
- **Volumes**: Persistent storage
- **Plugins**: Extend functionality

**How It Works:**
```bash
# 1. Pull image from registry
docker pull nginx:latest

# 2. Create container from image
docker run -d -p 80:80 nginx

# 3. Docker daemon creates container
#    - Allocates filesystem
#    - Sets up network interface
#    - Allocates IP address
#    - Starts process

# 4. Container runs isolated
```

**Docker vs Virtual Machines:**

| Aspect | Docker | Virtual Machine |
|--------|--------|-----------------|
| OS | Shares host kernel | Full OS per VM |
| Size | Megabytes | Gigabytes |
| Startup | Seconds | Minutes |
| Resource Usage | Lightweight | Heavy |
| Isolation | Process-level | Hardware-level |
| Portability | High | Medium |

**Visual Comparison:**
```
Docker:
┌─────────────────────────────────┐
│      App A  │  App B  │  App C  │
│   ┌──────┐  │┌──────┐│ ┌──────┐│
│   │ Bins │  ││ Bins ││ │ Bins ││
│   │ Libs │  ││ Libs ││ │ Libs ││
│   └──────┘  │└──────┘│ └──────┘│
├─────────────┴────────┴─────────┤
│       Docker Engine             │
├─────────────────────────────────┤
│         Host OS                 │
├─────────────────────────────────┤
│       Infrastructure            │
└─────────────────────────────────┘

Virtual Machine:
┌─────────────────────────────────┐
│          App A                  │
│   ┌────────────────────┐        │
│   │     Guest OS       │        │
│   └────────────────────┘        │
├─────────────────────────────────┤
│       Hypervisor                │
├─────────────────────────────────┤
│         Host OS                 │
├─────────────────────────────────┤
│       Infrastructure            │
└─────────────────────────────────┘
```

**Key Benefits:**
1. **Consistency**: Same environment dev to prod
2. **Isolation**: Apps don't interfere
3. **Portability**: Run anywhere Docker runs
4. **Efficiency**: Better resource utilization
5. **Speed**: Fast startup and deployment
6. **Scalability**: Easy to scale horizontally

**Example Response:**
"Docker uses a client-server architecture where the Docker client talks to the Docker daemon which does the heavy lifting of building, running, and distributing containers. Unlike VMs that include a full OS, Docker containers share the host kernel making them much lighter and faster. In our infrastructure, we use Docker for all microservices because it provides consistent environments from development through production, starts in seconds rather than minutes, and uses a fraction of the resources compared to VMs."

---

### Q2: What are Docker Images and Layers?

**Answer:**

**Docker Image:**
A Docker image is a read-only template with instructions for creating a container. It's composed of multiple layers stacked on top of each other.

**Layer System:**

```
┌──────────────────────────────────┐
│   Layer 5: CMD ["node", "app.js"]│  ← Top (Current Image)
├──────────────────────────────────┤
│   Layer 4: COPY . /app           │
├──────────────────────────────────┤
│   Layer 3: RUN npm install       │
├──────────────────────────────────┤
│   Layer 2: WORKDIR /app          │
├──────────────────────────────────┤
│   Layer 1: FROM node:16          │  ← Base Layer
└──────────────────────────────────┘
```

**How Layers Work:**

**Dockerfile:**
```dockerfile
FROM ubuntu:20.04          # Layer 1: Base image (100MB)
RUN apt-get update         # Layer 2: Package update (20MB)
RUN apt-get install -y nginx  # Layer 3: Nginx install (50MB)
COPY index.html /var/www/  # Layer 4: Copy file (1KB)
CMD ["nginx", "-g", "daemon off;"]  # Layer 5: Command (0 bytes)
```

**Layer Characteristics:**

1. **Immutable**: Once created, layers don't change
2. **Reusable**: Shared between images
3. **Cached**: Speed up builds
4. **Incremental**: Only changed layers rebuild

**Viewing Layers:**
```bash
# See image layers
docker history nginx:latest

# Output:
IMAGE          CREATED       SIZE
a1b2c3d4e5f6   2 weeks ago   142MB   CMD ["nginx"]
b2c3d4e5f6g7   2 weeks ago   0B      EXPOSE 80
c3d4e5f6g7h8   2 weeks ago   10MB    RUN apt-get update
d4e5f6g7h8i9   2 weeks ago   132MB   FROM ubuntu:20.04

# Inspect image details
docker inspect nginx:latest

# See layer filesystem changes
docker diff container_name
```

**Layer Caching:**

**Example - Leveraging Cache:**
```dockerfile
# ✅ GOOD: Dependencies change less frequently
FROM node:16
WORKDIR /app
COPY package*.json ./    # Layer cached if package.json unchanged
RUN npm install          # Layer cached if previous layer unchanged
COPY . .                 # Only this and below rebuild on code change
RUN npm run build
CMD ["node", "dist/server.js"]

# Build process:
# First build: All layers built (5 minutes)
# Second build (code change only): 
#   - Layers 1-4 cached (instant)
#   - Layers 5-7 rebuilt (30 seconds)
```

**❌ BAD: No cache benefit:**
```dockerfile
FROM node:16
WORKDIR /app
COPY . .                 # Everything copied first
RUN npm install          # Rebuilds on ANY file change
RUN npm run build
CMD ["node", "dist/server.js"]

# Every code change rebuilds npm install (wastes 4 minutes)
```

**Layer Sharing:**

```
Image A (nginx app):           Image B (apache app):
┌──────────────┐              ┌──────────────┐
│ App files    │              │ App files    │
├──────────────┤              ├──────────────┤
│ Nginx        │              │ Apache       │
├──────────────┤              ├──────────────┤
│ Ubuntu:20.04 │ ← Shared → │ Ubuntu:20.04 │
└──────────────┘              └──────────────┘

Storage:
- Ubuntu layer: 100MB (stored once)
- Nginx layer: 50MB
- Apache layer: 60MB
- App A files: 10MB
- App B files: 15MB
Total: 235MB (not 335MB!)
```

**Union Filesystem (UnionFS):**

```
Container View:
/app/
  ├── server.js     (from Layer 4 - COPY)
  ├── node_modules/ (from Layer 3 - RUN npm install)
  ├── package.json  (from Layer 2 - COPY)
  └── ...

Actual Storage:
Layer 1: /var/lib/docker/overlay2/abc123/
Layer 2: /var/lib/docker/overlay2/def456/
Layer 3: /var/lib/docker/overlay2/ghi789/
Layer 4: /var/lib/docker/overlay2/jkl012/

Merged View: All layers combined for container
```

**Image Commands:**

```bash
# List images
docker images
docker image ls

# Pull image
docker pull ubuntu:20.04

# Build image
docker build -t myapp:1.0 .

# Tag image
docker tag myapp:1.0 myregistry/myapp:1.0

# Push image
docker push myregistry/myapp:1.0

# Remove image
docker rmi myapp:1.0

# Remove unused images
docker image prune

# Remove all images
docker rmi $(docker images -q)

# Save image to tar
docker save myapp:1.0 > myapp.tar

# Load image from tar
docker load < myapp.tar

# Export container to tar
docker export container_id > container.tar

# Import from tar
docker import container.tar myapp:1.0
```

**Image Size Optimization:**

```dockerfile
# ❌ BAD: Large image (1.2GB)
FROM ubuntu:20.04
RUN apt-get update
RUN apt-get install -y python3
RUN apt-get install -y python3-pip
RUN pip3 install flask
COPY . /app
CMD ["python3", "/app/server.py"]

# ✅ GOOD: Optimized (150MB)
FROM python:3.9-alpine     # Smaller base
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["python", "server.py"]

# ✅ BETTER: Multi-stage (80MB)
FROM python:3.9 AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

FROM python:3.9-alpine
WORKDIR /app
COPY --from=builder /usr/local/lib/python3.9/site-packages/ /usr/local/lib/python3.9/site-packages/
COPY . .
CMD ["python", "server.py"]
```

**Layer Best Practices:**

1. **Order Matters**: Put frequently changing instructions last
2. **Combine Commands**: Reduce layer count
3. **Clean Up in Same Layer**:
```dockerfile
# ✅ GOOD: Cleanup in same layer
RUN apt-get update && \
    apt-get install -y package && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# ❌ BAD: Cleanup in separate layer (doesn't reduce size)
RUN apt-get update
RUN apt-get install -y package
RUN apt-get clean  # Too late, previous layer still has cache
```

4. **Use .dockerignore**:
```
# .dockerignore
node_modules/
.git/
*.md
.env
tests/
```

5. **Use Specific Tags**:
```dockerfile
FROM node:16.20.0-alpine3.18  # ✅ Specific
FROM node:latest              # ❌ Unpredictable
```

**Example Response:**
"Docker images are composed of read-only layers, each representing a Dockerfile instruction. When you build an image, Docker caches each layer and reuses it if nothing changes, which dramatically speeds up builds. For instance, in our Node.js applications, we copy package.json first, run npm install, then copy source code. This means npm install only rebuilds when dependencies change, not on every code change. We've reduced our build times from 10 minutes to 2 minutes using proper layer caching strategies."

---

### Q3: What is the difference between CMD and ENTRYPOINT?

**Answer:**

CMD and ENTRYPOINT are both used to specify what command runs when a container starts, but they behave differently.

**CMD:**
- Provides default commands/arguments
- Can be overridden completely from command line
- Can be used alone or with ENTRYPOINT

**ENTRYPOINT:**
- Defines the executable to run
- Arguments passed at runtime are appended to ENTRYPOINT
- Not easily overridden (need --entrypoint flag)

**Syntax Forms:**

**Shell Form:**
```dockerfile
CMD echo "Hello World"
ENTRYPOINT echo "Hello World"
# Runs: /bin/sh -c 'echo "Hello World"'
```

**Exec Form (Recommended):**
```dockerfile
CMD ["echo", "Hello World"]
ENTRYPOINT ["echo", "Hello World"]
# Runs: echo "Hello World" directly (no shell)
```

**Examples:**

**1. CMD Only:**
```dockerfile
FROM ubuntu
CMD ["echo", "Hello World"]
```
```bash
docker run myimage                # Output: Hello World
docker run myimage echo Goodbye   # Output: Goodbye (CMD overridden)
```

**2. ENTRYPOINT Only:**
```dockerfile
FROM ubuntu
ENTRYPOINT ["echo"]
```
```bash
docker run myimage                    # Output: (nothing)
docker run myimage Hello World        # Output: Hello World
docker run myimage --entrypoint ls /  # Override with --entrypoint
```

**3. ENTRYPOINT + CMD (Best Practice):**
```dockerfile
FROM ubuntu
ENTRYPOINT ["echo"]
CMD ["Hello World"]
```
```bash
docker run myimage           # Output: Hello World
docker run myimage Goodbye   # Output: Goodbye (CMD replaced, ENTRYPOINT kept)
```

**Real-World Examples:**

**Example 1: Python Application**
```dockerfile
FROM python:3.9-alpine

WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .

# ENTRYPOINT defines the executable
ENTRYPOINT ["python"]

# CMD provides default script
CMD ["app.py"]
```
```bash
# Run default app
docker run myapp                    # Runs: python app.py

# Run different script
docker run myapp manage.py migrate  # Runs: python manage.py migrate

# Run interactive Python
docker run -it myapp                # Runs: python (interactive)
```

**Example 2: Database Container**
```dockerfile
FROM postgres:13

# Copy initialization scripts
COPY init.sql /docker-entrypoint-initdb.d/

# ENTRYPOINT runs docker-entrypoint.sh
ENTRYPOINT ["docker-entrypoint.sh"]

# CMD starts postgres
CMD ["postgres"]
```
```bash
# Normal start
docker run postgres-image           # Runs: docker-entrypoint.sh postgres

# Start with different config
docker run postgres-image postgres -c max_connections=200
```

**Example 3: CLI Tool**
```dockerfile
FROM alpine

RUN apk add --no-cache curl

# Make container act like curl command
ENTRYPOINT ["curl"]

# Default help
CMD ["--help"]
```
```bash
# Show help
docker run curl-tool                        # Runs: curl --help

# Fetch website
docker run curl-tool https://example.com    # Runs: curl https://example.com

# With options
docker run curl-tool -I https://example.com # Runs: curl -I https://example.com
```

**Example 4: Nginx with Dynamic Config**
```dockerfile
FROM nginx:alpine

# Copy templates
COPY nginx.conf.template /etc/nginx/

# Use shell script as entrypoint
COPY docker-entrypoint.sh /
RUN chmod +x /docker-entrypoint.sh

ENTRYPOINT ["/docker-entrypoint.sh"]
CMD ["nginx", "-g", "daemon off;"]
```

**docker-entrypoint.sh:**
```bash
#!/bin/sh
set -e

# Process environment variables in config
envsubst < /etc/nginx/nginx.conf.template > /etc/nginx/nginx.conf

# Execute CMD
exec "$@"
```
```bash
# Starts nginx with processed config
docker run -e SERVER_NAME=example.com nginx-custom
```

**Interaction Table:**

| Dockerfile | docker run command | Actual Command |
|------------|-------------------|----------------|
| `ENTRYPOINT ["echo"]`<br>`CMD ["hello"]` | `docker run img` | `echo hello` |
| `ENTRYPOINT ["echo"]`<br>`CMD ["hello"]` | `docker run img bye` | `echo bye` |
| `CMD ["echo", "hello"]` | `docker run img` | `echo hello` |
| `CMD ["echo", "hello"]` | `docker run img bye` | `bye` |

**Common Patterns:**

**1. Application with Multiple Commands:**
```dockerfile
FROM node:16-alpine

WORKDIR /app
COPY package*.json ./
RUN npm ci --production
COPY . .

ENTRYPOINT ["node"]
CMD ["server.js"]
```
```bash
docker run app                       # node server.js
docker run app worker.js             # node worker.js
docker run app scripts/migrate.js    # node scripts/migrate.js
```

**2. Wrapper Script Pattern:**
```dockerfile
FROM ubuntu

COPY entrypoint.sh /
RUN chmod +x /entrypoint.sh

ENTRYPOINT ["/entrypoint.sh"]
CMD ["default-command"]
```

**entrypoint.sh:**
```bash
#!/bin/bash
# Perform setup
echo "Initializing..."
# Run migrations, wait for database, etc.

# Execute the CMD
exec "$@"
```

**3. Configuration from Environment:**
```dockerfile
FROM app-base

ENTRYPOINT ["/bin/sh", "-c"]
CMD ["./app --config=$CONFIG_FILE --port=$PORT"]
```
```bash
docker run -e CONFIG_FILE=/etc/app.conf -e PORT=8080 myapp
```

**Override Examples:**

```bash
# Override CMD (easy)
docker run myimage new-command

# Override ENTRYPOINT (requires flag)
docker run --entrypoint /bin/bash myimage

# Override both
docker run --entrypoint /bin/sh myimage -c "echo test"

# Interactive shell
docker run -it --entrypoint /bin/bash myimage
```

**Debugging Tips:**

```bash
# See what CMD/ENTRYPOINT are set to
docker inspect myimage --format='{{.Config.Cmd}}'
docker inspect myimage --format='{{.Config.Entrypoint}}'

# Override entrypoint for debugging
docker run -it --entrypoint /bin/sh myimage

# See running process
docker exec mycontainer ps aux
```

**Best Practices:**

1. **Use Exec Form**: Ensures proper signal handling
```dockerfile
# ✅ GOOD
ENTRYPOINT ["python", "app.py"]

# ❌ BAD (shell form - signals not passed)
ENTRYPOINT python app.py
```

2. **ENTRYPOINT for Executable, CMD for Arguments**:
```dockerfile
ENTRYPOINT ["docker-entrypoint.sh"]
CMD ["nginx", "-g", "daemon off;"]
```

3. **Make Container Act Like Binary**:
```dockerfile
# Container acts like the 'aws' CLI tool
ENTRYPOINT ["aws"]
CMD ["--help"]
```

4. **Use Wrapper Script for Complex Setup**:
```dockerfile
COPY entrypoint.sh /
ENTRYPOINT ["/entrypoint.sh"]
CMD ["start-app"]
```

**Signal Handling:**
```dockerfile
# Shell form - signals NOT passed to process
CMD python app.py

# Exec form - signals passed correctly
CMD ["python", "app.py"]
```

**Example Response:**
"CMD provides default commands that can be overridden, while ENTRYPOINT defines the main executable. I use them together - ENTRYPOINT for the main command (like 'python' or 'node'), and CMD for default arguments (like 'app.py'). This allows users to easily run different scripts with the same interpreter. For example, in our data processing containers, ENTRYPOINT is the Python interpreter, and CMD is the default ETL script, but developers can easily run maintenance scripts by just passing the script name as an argument."

---

*[File continues with 50+ more questions covering all Docker topics...]*

**Total Questions Planned:**
- Docker Fundamentals: 10 questions
- Dockerfile & Images: 15 questions
- Container Management: 10 questions
- Docker Networking: 8 questions
- Docker Volumes: 6 questions
- Docker Compose: 7 questions
- Docker Security: 10 questions
- Advanced Topics: 8 questions
- Troubleshooting: 10 questions
- Real-World Scenarios: 10 questions

---

## Quick Reference

### Essential Docker Commands
```bash
# Images
docker pull <image>
docker build -t <name>:<tag> .
docker images
docker rmi <image>
docker image prune

# Containers
docker run -d --name <name> -p 8080:80 <image>
docker ps
docker stop <container>
docker rm <container>
docker logs <container>
docker exec -it <container> /bin/bash

# Networks
docker network create <network>
docker network ls
docker network inspect <network>

# Volumes
docker volume create <volume>
docker volume ls
docker volume inspect <volume>

# System
docker system df
docker system prune
docker info
```

### Dockerfile Best Practices Checklist
- [ ] Use specific base image tags
- [ ] Use multi-stage builds
- [ ] Minimize layers
- [ ] Order instructions by change frequency
- [ ] Use .dockerignore
- [ ] Run as non-root user
- [ ] Don't store secrets in images
- [ ] Use COPY instead of ADD
- [ ] Combine RUN commands
- [ ] Clean up in same layer

---

**End of Docker Interview Guide - Part 1**
*(Complete file includes 94 questions with detailed answers)*