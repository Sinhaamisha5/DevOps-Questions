# Docker Projects Documentation

## 1. Build a custom Docker image with multi-stage builds

**Answer:**  
"I use multi-stage builds to reduce the final image size by separating the build environment from the runtime. For example, when building a Go application, I first compile the binary in a larger image with all build tools, then copy only the binary into a minimal image like alpine for deployment."

**Example Dockerfile snippet:**

```dockerfile
# Build stage
FROM golang:1.21 AS builder
WORKDIR /app
COPY . .
RUN go build -o myapp

# Runtime stage
FROM alpine:3.18
WORKDIR /app
COPY --from=builder /app/myapp .
CMD ["./myapp"]
Case Study:
"In my last project, we had a Node.js app with heavy dev dependencies. Using multi-stage builds, we reduced the image from 1.2GB to 250MB, improving deployment speed and security."

2. Run a MySQL container with persistent volumes and init scripts
Answer:
"I usually attach a volume to persist data across container restarts and use the /docker-entrypoint-initdb.d folder for schema or seed scripts."

Example:

bash
Copy code
docker run -d \
  --name mysql-db \
  -e MYSQL_ROOT_PASSWORD=rootpass \
  -v mysql_data:/var/lib/mysql \
  -v ./init.sql:/docker-entrypoint-initdb.d/init.sql \
  mysql:8.0
Case Study:
"For an e-commerce project, we ran MySQL with a persistent volume so the database persisted even after container updates. We also had multiple SQL scripts to initialize product categories and users automatically."

3. Configure Docker Compose for a web app + DB + cache
Answer:
"I use Docker Compose to orchestrate multiple services, making it easy to spin up the whole stack with one command."

docker-compose.yml snippet:

yaml
Copy code
version: '3.9'
services:
  web:
    build: .
    ports:
      - "8080:8080"
    environment:
      - DB_HOST=db
      - REDIS_HOST=cache
    depends_on:
      - db
      - cache
  db:
    image: mysql:8
    environment:
      MYSQL_ROOT_PASSWORD: rootpass
    volumes:
      - db_data:/var/lib/mysql
  cache:
    image: redis:alpine

volumes:
  db_data:
Case Study:
"In a recent project, we used Compose to deploy a Flask app, MySQL, and Redis locally. This setup allowed developers to replicate the production stack quickly."

4. Optimize Dockerfile to reduce image size
Answer:
"I optimize images by using lightweight base images like Alpine, cleaning up package caches, and removing unnecessary files."

Example Dockerfile snippet:

dockerfile
Copy code
FROM python:3.12-alpine
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["python", "app.py"]
Case Study:
"In a microservices project, we reduced the image size of a service from 800MB to 80MB using Alpine and --no-cache-dir, which also sped up CI/CD builds."

5. Write a healthcheck in Dockerfile for your app
Answer:
"I use the HEALTHCHECK instruction to allow Docker to monitor container health and restart unhealthy containers automatically."

Example:

dockerfile
Copy code
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s \
  CMD curl -f http://localhost:8080/health || exit 1
Case Study:
"For a Node.js API, we used a healthcheck endpoint. Kubernetes used it to restart pods automatically when the service was unresponsive."

6. Set up Docker networking between multiple containers
Answer:
"Docker provides user-defined networks so containers can communicate using container names as hostnames."

Example:

bash
Copy code
docker network create my-network
docker run -d --name db --network my-network mysql
docker run -d --name web --network my-network my-web-app
Case Study:
"In a microservice setup, we created a bridge network so all backend services could talk to the database using service names, which simplified configuration."

7. Configure environment variables and secrets in containers
Answer:
"I use environment variables for configuration and Docker secrets for sensitive data like passwords."

Example:

yaml
Copy code
services:
  web:
    image: myapp
    environment:
      - API_URL=https://api.example.com
    secrets:
      - db_password

secrets:
  db_password:
    file: ./db_password.txt
Case Study:
"In production, we used Docker secrets for DB passwords to avoid committing them to the repo, which improved security compliance."

8. Create a private Docker registry and push images
Answer:
"I can set up a private registry using the official registry image and push custom images for internal use."

Example:

bash
Copy code
docker run -d -p 5000:5000 --name registry registry:2
docker tag myapp localhost:5000/myapp
docker push localhost:5000/myapp
Case Study:
"For an internal CI/CD pipeline, we hosted a private registry, which allowed developers to pull images quickly without relying on Docker Hub."

9. Use Docker logs to troubleshoot container issues
Answer:
"I use docker logs for runtime debugging and docker exec to investigate the container state."

Example:

bash
Copy code
docker logs -f web
docker exec -it web sh
Case Study:
"During a production outage, I discovered a misconfigured environment variable by inspecting logs, which allowed us to fix the container without redeploying."

10. Run a containerized app with resource limits
Answer:
"I use CPU and memory limits to prevent a single container from overloading the host."

Example:

bash
Copy code
docker run -d --name myapp --memory="512m" --cpus="1.0" myapp
Case Study:
"In a shared staging environment, we set memory and CPU limits to prevent runaway processes from affecting other services."