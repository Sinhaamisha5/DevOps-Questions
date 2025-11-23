# Docker Layers & Union Filesystem - Complete Explanation

## The Problem Docker Solves

Imagine you have 5 Docker images, each needs Ubuntu. Without smart storage:
```
Image 1: Ubuntu (100MB) + App1 (20MB) = 120MB
Image 2: Ubuntu (100MB) + App2 (20MB) = 120MB
Image 3: Ubuntu (100MB) + App3 (20MB) = 120MB
Image 4: Ubuntu (100MB) + App4 (20MB) = 120MB
Image 5: Ubuntu (100MB) + App5 (20MB) = 120MB

Total storage needed: 600MB
```

**500MB wasted on duplicate Ubuntu!**

Docker's solution: **Share common layers**

---

## What is a Layer?

A layer is a **read-only snapshot** created by each instruction in a Dockerfile.

### Example Dockerfile:

```dockerfile
FROM ubuntu:20.04              # Layer 1
RUN apt-get update             # Layer 2
RUN apt-get install -y nginx   # Layer 3
COPY index.html /var/www/html/ # Layer 4
CMD ["nginx", "-g", "daemon off;"]
```

**This creates 4 layers:**

```
Layer 4: index.html file
Layer 3: Nginx application
Layer 2: Updated packages
Layer 1: Ubuntu base OS
```

---

## How Layers Work

### Building Image Step by Step:

**Step 1: FROM ubuntu:20.04**
```
Layer 1 (Size: 100MB)
├── bin/
├── lib/
├── etc/
└── (Ubuntu OS files)
```

**Step 2: RUN apt-get update**
```
Layer 1 (100MB - from before)
├── (Ubuntu OS files)

+ Layer 2 (Size: 10MB)
└── Updated package lists
```

**Step 3: RUN apt-get install -y nginx**
```
Layers 1 + 2 (from before)

+ Layer 3 (Size: 50MB)
└── nginx/ (application files)
```

**Step 4: COPY index.html /var/www/html/**
```
Layers 1 + 2 + 3 (from before)

+ Layer 4 (Size: 1MB)
└── index.html
```

**Final Image = Layer 1 + Layer 2 + Layer 3 + Layer 4 = 161MB**

---

## Layer Sharing - The Magic!

### Two Different Images Using Same Base:

**Image A: Nginx Web Server**
```dockerfile
FROM ubuntu:20.04              # Layer 1
RUN apt-get update             # Layer 2
RUN apt-get install -y nginx   # Layer 3 (Nginx)
COPY index.html /var/www/html/ # Layer 4
```

**Image B: Apache Web Server**
```dockerfile
FROM ubuntu:20.04              # Layer 1 (SAME!)
RUN apt-get update             # Layer 2 (SAME!)
RUN apt-get install -y apache2 # Layer 3 (Apache)
COPY index.html /var/www/html/ # Layer 4 (Different)
```

### Storage on Disk:

```
Layer 1 (ubuntu:20.04): 100MB ← Used by BOTH images
Layer 2 (apt-get update): 10MB ← Used by BOTH images
Layer 3a (nginx): 50MB
Layer 3b (apache2): 60MB
Layer 4a (index A): 1MB
Layer 4b (index B): 1MB

Total: 222MB (NOT 322MB!)
Saved: 100MB by sharing Layer 1!
```

### Visual Representation:

```
Image A                  Image B
┌─────────────┐         ┌─────────────┐
│ Layer 4a    │         │ Layer 4b    │
│ (1MB)       │         │ (1MB)       │
├─────────────┤         ├─────────────┤
│ Layer 3a    │         │ Layer 3b    │
│ (50MB Nginx)│         │ (60MB Apache)
├─────────────┤         ├─────────────┤
│ Layer 2     │←────────→│ Layer 2     │
│ (10MB)      │ SHARED  │ (10MB)      │
├─────────────┤         ├─────────────┤
│ Layer 1     │←────────→│ Layer 1     │
│ (100MB)     │ SHARED  │ (100MB)     │
└─────────────┘         └─────────────┘
```

---

## Real-World Example: 3 Different Apps

### App 1: Node.js API
```dockerfile
FROM node:18-slim          # Layer 1: 400MB
RUN npm install express    # Layer 2: 50MB
COPY app.js .              # Layer 3: 5MB
```
**Total Image Size: 455MB**

### App 2: Flask API
```dockerfile
FROM python:3.11-slim      # Layer 1: 150MB
RUN pip install flask      # Layer 2: 30MB
COPY app.py .              # Layer 3: 2MB
```
**Total Image Size: 182MB**

### App 3: Java API
```dockerfile
FROM openjdk:17-slim       # Layer 1: 400MB
COPY app.jar .             # Layer 2: 100MB
```
**Total Image Size: 500MB**

### Storage Without Sharing:
```
App 1: 455MB
App 2: 182MB
App 3: 500MB
Total: 1137MB
```

### Storage WITH Sharing:

```
node:18-slim (400MB) - used by App 1
python:3.11-slim (150MB) - used by App 2
openjdk:17-slim (400MB) - used by App 3

Layer 2a (npm): 50MB - App 1 only
Layer 2b (pip): 30MB - App 2 only
Layer 2c (jar): 100MB - App 3 only

Layer 3a: 5MB - App 1 only
Layer 3b: 2MB - App 2 only

Total: 1137MB
(Layers are different, so less sharing)
```

### But if 2 apps used same base:

```dockerfile
# App 1: Frontend
FROM node:18-slim          # Layer 1: 400MB
RUN npm install react      # Layer 2a: 60MB
COPY index.js .            # Layer 3a: 1MB

# App 2: Backend
FROM node:18-slim          # Layer 1: 400MB (SHARED!)
RUN npm install express    # Layer 2b: 50MB
COPY server.js .           # Layer 3b: 2MB
```

**Storage:**
```
node:18-slim: 400MB ← Both use this!
npm react: 60MB
npm express: 50MB
index.js: 1MB
server.js: 2MB
Total: 513MB (saved 100MB!)
```

---

## Union Filesystem (UnionFS) - How Docker Views Layers

### The Problem:
Layers are **separate files on disk**. How does Docker show them as one filesystem to the container?

### The Solution: Union Filesystem

UnionFS **stacks layers on top of each other** and presents them as a single filesystem.

### Example:

**Layer 1 (Base Ubuntu) - Read-Only**
```
/
├── bin/
├── etc/
├── lib/
└── usr/
```

**Layer 2 (apt-get update) - Read-Only**
```
/
└── var/lib/apt/
    └── lists/ (updated)
```

**Layer 3 (nginx install) - Read-Only**
```
/
├── etc/nginx/
├── var/www/html/
└── usr/local/nginx/
```

**Layer 4 (your app) - Read-Only**
```
/
├── app/
│   ├── index.html
│   └── config.conf
```

### Container's View (Union):
```
/
├── app/
│   ├── index.html       (from Layer 4)
│   └── config.conf      (from Layer 4)
├── bin/                 (from Layer 1)
├── etc/
│   ├── nginx/           (from Layer 3)
│   └── (other)          (from Layer 1)
├── lib/                 (from Layer 1)
├── usr/
│   ├── local/nginx/     (from Layer 3)
│   └── (other)          (from Layer 1)
└── var/
    ├── www/html/        (from Layer 3)
    └── lib/apt/lists/   (from Layer 2)
```

**Everything looks like ONE filesystem!**

---

## How Docker Stores Layers - Overlay2

Docker uses a storage driver called **overlay2** (on Linux).

### On Your Computer:

```
/var/lib/docker/overlay2/

├── abc123de456/ (Layer 1: Ubuntu)
│   ├── diff/
│   │   ├── bin/
│   │   ├── etc/
│   │   └── lib/
│   └── ...
│
├── def789gh012/ (Layer 2: apt update)
│   ├── diff/
│   │   └── var/lib/apt/lists/
│   └── ...
│
├── ghi456jk789/ (Layer 3: Nginx)
│   ├── diff/
│   │   ├── etc/nginx/
│   │   └── var/www/html/
│   └── ...
│
├── jkl012mn345/ (Layer 4: App)
│   ├── diff/
│   │   └── app/index.html
│   └── ...
│
└── pqr678st901/ (Container - Read-Write Layer!)
    ├── diff/
    │   └── (changes made inside container)
    └── ...
```

**Each layer is in a separate folder, but UnionFS merges them!**

---

## Read-Only vs Read-Write

### Image Layers: Read-Only ✓
```
Layer 1 (Ubuntu)  - Can't change
Layer 2 (apt)     - Can't change
Layer 3 (Nginx)   - Can't change
Layer 4 (App)     - Can't change
```

**Why?** So multiple containers can share the same image layers!

### Container Layer: Read-Write
```
Container Layer - CAN change
(modifications made inside container)
```

---

## Example: What Happens When Container Modifies a File

### Scenario:
You have a file from Layer 3 (Nginx) that you want to modify inside the container.

### Without Copy-on-Write:
- Container directly modifies Layer 3
- Other containers using Layer 3 see the change
- DISASTER!

### Docker's Copy-on-Write:
```
Layer 3 (Nginx) - Original Read-Only
├── etc/nginx/nginx.conf

Container Layer - Read-Write
├── etc/nginx/nginx.conf (COPY from Layer 3)
   └── (modifications stored here)
```

**Result:**
- Original file stays safe in Layer 3
- Container gets its own copy in Container Layer
- Other containers unaffected!

---

## Layer Caching - Why Build Speed Matters

### Dockerfile:
```dockerfile
FROM ubuntu:20.04              # Step 1
RUN apt-get update             # Step 2
RUN apt-get install -y python  # Step 3
COPY app.py .                  # Step 4
RUN pip install flask          # Step 5
```

### First Build:
```
Step 1: Build Layer 1 (ubuntu) ✓
Step 2: Build Layer 2 (apt update) ✓
Step 3: Build Layer 3 (python) ✓
Step 4: Build Layer 4 (app.py) ✓
Step 5: Build Layer 5 (flask) ✓
Total time: 5 minutes
```

### Second Build (no changes):
```
Step 1: Layer 1 exists ✓ Use cache (instant!)
Step 2: Layer 2 exists ✓ Use cache (instant!)
Step 3: Layer 3 exists ✓ Use cache (instant!)
Step 4: Layer 4 exists ✓ Use cache (instant!)
Step 5: Layer 5 exists ✓ Use cache (instant!)
Total time: 1 second
```

### Third Build (only app.py changed):
```
Step 1-3: Use cache ✓ (instant!)
Step 4: File changed! ✗ Rebuild Layer 4
Step 5: Layer 5 depends on Layer 4 ✗ Rebuild Layer 5
Total time: 1 minute (only rebuilds from changed layer onward)
```

**This is why order matters!**

```dockerfile
# GOOD - Put things that change least at the top
FROM python:3.11-slim
RUN pip install flask           # Rarely changes
COPY requirements.txt .         # Rarely changes
RUN pip install -r requirements.txt
COPY app.py .                   # Changes frequently
```

```dockerfile
# BAD - Rebuilds everything
FROM python:3.11-slim
COPY app.py .                   # Changes frequently
RUN pip install flask           # Rebuilt unnecessarily
```

---

## Visualizing Layer Size

### Command to see layers:

```bash
docker image inspect myapp:1.0

# Output shows each layer's size
```

### Example Output:
```
Layers:
  sha256:abc123... Size: 100MB (Ubuntu)
  sha256:def456... Size: 10MB  (apt update)
  sha256:ghi789... Size: 50MB  (Nginx)
  sha256:jkl012... Size: 1MB   (app files)

Total Uncompressed Size: 161MB
```

---

## Real-World Impact

### Scenario: Deploy 100 microservices

**Without layer sharing:**
```
100 images × 150MB = 15,000MB (15GB!)
```

**With layer sharing (all use same base):**
```
Base layer: 100MB
100 × app-specific layers: ~50MB each

Total: 100MB + (100 × 50MB) = 5,100MB (5GB!)
Saved: 10GB!
```

**On a company's Docker registry:**
```
10 projects × 100 services × 150MB = 150GB
With sharing: 50GB
Saved: 100GB!
```

---

## Key Takeaways

1. **Every Dockerfile instruction creates a layer**
2. **Layers are read-only and shareable**
3. **Multiple images can share the same layers**
4. **Union Filesystem merges layers into one view**
5. **Docker stores layers separately and merges them**
6. **Containers get a read-write layer on top**
7. **Copy-on-Write prevents container changes from affecting the image**
8. **Layer caching speeds up rebuilds**
9. **Order matters - put stable things first**
10. **Massive storage savings through layer sharing**