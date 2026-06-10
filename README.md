# Roboshop Docker Project

This repository contains the Dockerized configuration for the **Roboshop** application, a multi-container microservices application.

## Component Architecture

The application consists of the following components:

- **Frontend**: The user interface. Routes requests to `catalogue`, `user`, and other backend services.
- **MongoDB**: Database store for product catalog details.
- **Redis**: In-memory data store for user sessions and cart details.
- **Catalogue**: Microservice handling catalog-related requests (depends on MongoDB).
- **User**: Microservice managing user profiles and logins (depends on Redis).
- **Cart**: Microservice managing user carts (depends on Redis).
- **Shipping**: Microservice handling shipping calculations.

---

## Networking and Inter-Component Communication

> [!IMPORTANT]
> **Custom Bridge Network Requirement**
> To allow secure and seamless communication between the microservices, a **custom bridge network** must be used.
>
> **Why is this necessary?**
> By default, containers on the default bridge network cannot resolve each other by container/service name via DNS. Using a custom bridge network enables Docker's automatic DNS resolution, allowing containers to connect to each other using their container name (e.g., `mongodb`, `redis`) as the hostname.
>
> In the `compose.yaml` configuration, this custom bridge network is defined under the `networks` block:
> ```yaml
> networks:
>   default:
>     name: roboshop
> ```
> 
> If you are running the containers individually (without Docker Compose), you must first create this network manually before launching the containers:
> ```bash
> docker network create --driver bridge roboshop
> ```
> And connect each container to it:
> ```bash
> docker run --network roboshop ...
> ```

---

## Getting Started

### Install Docker

If you are running on a RHEL 9 EC2 instance, you can install Docker and the required plugins using the following commands:

```bash
sudo dnf -y install dnf-plugins-core
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
sudo dnf install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker ec2-user
```

> [!NOTE]
> After running the commands, you may need to log out and log back in (or run `newgrp docker`) for the group changes to take effect so that you can run Docker commands without `sudo`.

### Using Docker Compose
To build and start the components together:
```bash
docker compose up -d --build
```
# Increase Disk Space on RHEL 9.7 EC2 (LVM)

> **Note:** While creating the EC2 instance, choose **50 GB** as the root volume size instead of the default size. On RHEL 9.x, the filesystem is **not automatically expanded** to use the entire EBS volume, so the following steps are required.

## Verify the current disk layout

```bash
lsblk
df -h
sudo vgs
sudo lvs
```

## Install `growpart`

```bash
sudo dnf install -y cloud-utils-growpart
```

## Extend the root partition

```bash
sudo growpart /dev/nvme0n1 4
```

> Replace `4` with your root partition number if it is different.

## Resize the LVM Physical Volume

```bash
sudo pvresize /dev/nvme0n1p4
```

## Verify free space in the Volume Group

```bash
sudo vgs
```

`VFree` should now show the newly available space.

## Extend the `/var` Logical Volume

Since Docker stores images and containers under `/var/lib/docker`, it's recommended to extend the `/var` logical volume.

Use all available free space:

```bash
sudo lvextend -r -l +100%FREE /dev/RootVG/varVol
```

Or extend by a specific size (example: 20 GB):

```bash
sudo lvextend -r -L +20G /dev/RootVG/varVol
```

## Verify

```bash
df -h
```

The `/var` filesystem should now reflect the increased size.

---

### Why is this required?

Although the EC2 instance is created with a **50 GB EBS volume**, RHEL 9.x initially creates a smaller LVM partition (around 20 GB by default). The remaining space is left unallocated and must be manually extended before Docker workloads can use it.

---

# Container Size Optimization

To ensure fast deployment and minimal resource usage, the container configurations in this repository have been optimized:

* **Minimal Base Images:** Used Alpine-based Docker images (such as `node:20-alpine`) to significantly decrease the overall container image size and reduce the security vulnerability footprint compared to standard base images.
* **Multi-Stage Builds:** Implemented multi-stage builds (e.g., in the `catalogue` service) to separate the build environment from the final execution environment. This ensures that build tools and temporary dependencies are not included in the final production images, resulting in highly lightweight containers.

---

# Troubleshooting & Key Learnings

During the development and containerization of the RoboShop microservices application, I encountered several real-world issues. Below are the major challenges, root causes, and their resolutions.

---

## 1. Docker Build Context Issues

### Problem

Docker failed to locate the `Dockerfile` or application source files during image build.

### Root Cause

The `context` property was not specified correctly in `compose.yaml`, causing Docker to search in the project root.

### Solution

```yaml
build:
  context: ./catalogue
  dockerfile: dockerfile
```

**Learning:** Docker can only access files within the specified build context.

---

## 2. COPY Instruction Failures

### Problem

```
COPY requirements.txt failed
```

### Root Cause

Required files were outside the Docker build context or missing.

### Solution

* Verified project structure.
* Corrected the build context.
* Ensured all required files existed before building.

---

## 3. Shipping Service Build Failure

### Problem

```
Unable to find main class
```

### Root Cause

* Incorrect Dockerfile configuration.
* Application source wasn't copied correctly.

### Solution

```dockerfile
COPY pom.xml .
COPY src ./src
RUN mvn clean package
```

**Learning:** Spring Boot projects require both `pom.xml` and the complete `src` directory.

---

## 4. MySQL Container Initialization Failure

### Problem

```
Database is uninitialized and password option is not specified
```

### Root Cause

The MySQL image expects the root password through Docker Compose environment variables.

### Solution

```yaml
mysql:
  environment:
    MYSQL_ROOT_PASSWORD: Roboshop@1
```

**Learning:** Environment variables defined in `docker-compose.yml` override image configuration and are the recommended approach.

---

## 5. Payment Container Exiting Immediately

### Problem

The payment container exited with status code `0` immediately after startup.

### Root Cause

Missing dependencies and incorrect environment variable configuration.

### Solution

Configured:

* RabbitMQ
* Cart Service
* User Service
* Required environment variables

using Docker Compose.

---

## 6. Frontend Could Not Reach Backend Services

### Problem

```
host not found in upstream "payment"
```

### Root Cause

The backend container was unavailable when Nginx started.

### Solution

* Fixed backend services.
* Rebuilt the frontend image.
* Verified Nginx reverse proxy configuration.

---

## 7. Redis Connectivity Issues

### Problem

```
ECONNREFUSED 127.0.0.1:6379
```

### Root Cause

Application attempted to connect to localhost instead of the Redis container.

### Solution

Configured the Redis hostname as:

```
redis
```

instead of

```
localhost
```

**Learning:** Containers communicate using Docker DNS service names.

---

## 8. MongoDB Connectivity

### Problem

Catalogue service failed health checks.

### Solution

Verified service connectivity using:

```bash
curl http://catalogue:8080/health
```

Expected response:

```json
{
  "app": "OK",
  "mongo": true
}
```

---

## 9. Docker Compose `depends_on`

### Observation

`depends_on` only controls startup order.

It **does not wait** until a service is healthy.

**Learning:** For production deployments, Docker Health Checks should be implemented.

---

## 10. Docker Cache Issues

### Problem

Even after modifying source files, containers continued using older configurations.

### Solution

Performed a complete rebuild:

```bash
docker compose down
docker builder prune -af
docker compose build --no-cache
docker compose up -d
```

**Learning:** Docker aggressively caches image layers.

---

## 11. Container Debugging Commands

Frequently used commands during troubleshooting:

```bash
docker ps
docker ps -a
docker images
docker logs <container>
docker compose logs <service>
docker exec -it <container> sh
docker inspect <container>
```

---

## 12. API Verification Using curl

Verified inter-service communication directly inside containers.

Examples:

```bash
curl http://catalogue:8080/health

curl http://catalogue:8080/products

curl http://cart:8080/add/ruthvik/STAN-1/1
```

This helped isolate frontend issues from backend issues.

---

## 13. Browser Network Debugging

Used Firefox Developer Tools to inspect:

* API requests
* HTTP status codes
* Request headers
* Response payloads

This made it easier to identify backend routing problems.

---

## 14. Docker Networking

Verified communication between containers using Docker DNS.

Example:

```
cart  --->  http://catalogue:8080
```

instead of using IP addresses.

**Learning:** Docker automatically provides service discovery using container names.

---

## 15. Nginx Reverse Proxy

Configured Nginx as an API gateway for all backend services.

Example:

```nginx
location /api/catalogue/ {
    proxy_pass http://catalogue:8080/;
}

location /api/cart/ {
    proxy_pass http://cart:8080/;
}

location /api/payment/ {
    proxy_pass http://payment:8080/;
}
```

---

# Useful Debugging Workflow

Whenever a service failed, I followed the same troubleshooting process:

1. Check container status

```bash
docker ps -a
```

2. View logs

```bash
docker logs <container>
```

3. Verify environment variables

```bash
docker exec -it <container> env
```

4. Enter the container

```bash
docker exec -it <container> sh
```

5. Test service connectivity

```bash
curl http://service-name:port/health
```

6. Inspect browser network requests

7. Rebuild images if configuration changed

```bash
docker compose build --no-cache
docker compose up -d
```

---

# Skills Demonstrated

* Docker & Docker Compose
* Container Size Optimization (Minimal Alpine images, Multi-Stage builds)
* Docker Networking
* Docker Image Creation
* Multi-Container Applications
* Nginx Reverse Proxy
* Spring Boot Containerization
* Python Containerization
* Node.js Containerization
* MongoDB
* MySQL
* Redis
* RabbitMQ
* Linux
* REST API Debugging
* Service Discovery
* Log Analysis
* Browser Network Debugging

---

# Lessons Learned

* Docker build context is one of the most common causes of build failures.
* `depends_on` controls startup order but **does not** guarantee service readiness.
* Docker DNS enables service-to-service communication using container names.
* Browser Developer Tools are invaluable for debugging frontend-backend interactions.
* `docker logs`, `docker exec`, and `curl` are essential tools for troubleshooting microservices.
* Docker image cache can cause outdated configurations to persist; rebuilding with `--no-cache` resolves such issues.
* Effective microservice debugging involves isolating each service and validating connectivity step by step.
* A systematic troubleshooting approach significantly reduces debugging time.

---

# Docker Best Practices

To ensure secure, lightweight, and performant container deployments, follow these Docker best practices:

### 1. Use Specific and Minimal Base Images
* **Avoid `latest`:** Always pin base image versions (e.g., `node:20-alpine` instead of `node`) to ensure build reproducibility.
* **Use Alpine/Slim:** Use minimal distributions like `alpine` or `slim` to reduce the image size, decrease the attack surface, and speed up deployments.

### 2. Leverage Build Cache (Order of Instructions)
* Copy dependency files (e.g., `package.json`, `pom.xml`, `requirements.txt`) first and run installation commands before copying the rest of the application source code.
* This ensures that Docker reuses cached layers for dependencies unless they explicitly change.

```dockerfile
# Good caching practice
COPY package.json .
RUN npm install
COPY . .
```

### 3. Use `.dockerignore` Files
* Exclude unnecessary files and folders (e.g., `node_modules`, `.git`, `dist`, local log files, configuration secrets) from entering the build context.
* This keeps build times fast and prevents confidential local configuration files from leaking into the container.

### 4. Run as a Non-Root User
* By default, Docker containers run with root privileges. For production setups, define and run the container with a non-root user (e.g., the built-in `node` user in Node.js images, or create a custom system user).

```dockerfile
# Example for Node.js Alpine
USER node
```

### 5. Utilize Multi-Stage Builds
* For compiled or built applications (like Java or React apps), use multi-stage builds. Compile artifacts in a heavier builder container, then copy only the finalized assets/jars to a lightweight runner image.

### 6. Avoid Storing Secrets or Sensitive Data in Dockerfiles
* Do not hardcode passwords, API keys, or certificates in the `Dockerfile` or source files.
* Inject sensitive data at runtime using environment variables (`env_file`, `environment` keys in Docker Compose) or Docker Secrets.

### 7. Clean Up Package Manager Caches
* When installing OS dependencies via `apk`, `apt`, or `dnf`, clean up package cache databases to avoid bloating the final image.

```dockerfile
RUN apk add --no-cache curl
```
