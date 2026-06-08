# Docker — The Complete Field Guide

> "Build once. Run anywhere. Debug forever."

---

## Table of Contents

1. [Core Concepts](#core-concepts)
2. [Images](#images)
3. [Containers](#containers)
4. [Volumes](#volumes)
5. [Networking](#networking)
6. [Docker Compose](#docker-compose)
7. [Dockerfile Deep Dive](#dockerfile-deep-dive)
8. [Multi-stage Builds](#multi-stage-builds)
9. [Registry & Distribution](#registry--distribution)
10. [Security](#security)
11. [Performance & Debugging](#performance--debugging)
12. [Cleanup](#cleanup)
13. [Production Patterns](#production-patterns)

---

## Core Concepts

```
Image       — Blueprint (read-only layers). Built from Dockerfile.
Container   — Running instance of an image. Isolated process.
Layer       — Filesystem change stored as a delta. Shared across images.
Volume      — Persistent storage. Lives outside the container.
Network     — Virtual network connecting containers.
Registry    — Image storage server (Docker Hub, ECR, GCR, etc.)
```

### How layers work
```
FROM ubuntu:22.04           ← base layer (read-only)
RUN apt-get install nginx   ← new layer (read-only)
COPY index.html /var/www/   ← new layer (read-only)
# Container run → adds a thin writable layer on top
# Union filesystem (overlay2) merges all layers
```

---

## Images

```bash
# Search & pull
docker search nginx
docker pull nginx
docker pull nginx:1.25-alpine
docker pull ubuntu:22.04

# List
docker images
docker images -a             include intermediate layers
docker images --filter "dangling=true"   untagged images

# Inspect
docker inspect nginx
docker inspect --format '{{.Config.Env}}' nginx
docker history nginx         show layer history
docker image inspect nginx --format '{{json .Config}}'

# Tag
docker tag nginx:latest myregistry.io/nginx:1.25
docker tag abc123def456 myapp:v1.2.0

# Remove
docker rmi nginx
docker rmi nginx:1.25-alpine
docker image prune           remove dangling images
docker image prune -a        remove all unused images

# Save/load (for air-gapped environments)
docker save nginx | gzip > nginx.tar.gz
docker load < nginx.tar.gz
docker export container | gzip > container.tar.gz
docker import container.tar.gz
```

---

## Containers

### Running Containers
```bash
docker run nginx
docker run -d nginx                      detached (background)
docker run -d -p 8080:80 nginx           port mapping (host:container)
docker run -d --name webserver nginx     name the container
docker run -it ubuntu bash               interactive terminal
docker run --rm ubuntu echo "hello"      auto-remove when done
docker run -e ENV_VAR=value nginx        environment variable
docker run -e ENV_VAR nginx              inherit from host
docker run --env-file .env nginx         from file

# Resource limits
docker run -d --memory="512m" nginx               memory limit
docker run -d --cpus="1.5" nginx                  CPU limit
docker run -d --memory="512m" --memory-swap="1g"  swap limit

# Volumes
docker run -v /host/path:/container/path nginx    bind mount
docker run -v myvolume:/data nginx                named volume
docker run --mount source=myvol,target=/data nginx  (modern syntax)
docker run --mount type=tmpfs,target=/tmp nginx   tmpfs (RAM)

# Networking
docker run --network mynet nginx
docker run --network host nginx          share host networking
docker run --network none nginx          no network

# Security
docker run --read-only nginx             read-only filesystem
docker run --user 1001:1001 nginx        non-root user
docker run --cap-drop ALL --cap-add NET_BIND_SERVICE nginx
docker run --security-opt no-new-privileges nginx
```

### Managing Containers
```bash
docker ps                    list running containers
docker ps -a                 include stopped containers
docker ps -q                 just IDs
docker ps --format "table {{.ID}}\t{{.Names}}\t{{.Status}}\t{{.Ports}}"

docker start container
docker stop container        graceful (SIGTERM + timeout)
docker kill container        immediate (SIGKILL)
docker restart container
docker pause container
docker unpause container

docker rm container
docker rm -f container       force remove running container
docker rm $(docker ps -aq)   remove ALL containers

# Execute in running container
docker exec container command
docker exec -it container bash       interactive
docker exec -it container sh         if no bash
docker exec -u root container bash   as root

# Copy files
docker cp container:/path/to/file .  copy from container
docker cp file.txt container:/path/  copy to container

# Inspect
docker inspect container
docker inspect --format '{{.NetworkSettings.IPAddress}}' container
docker top container                 running processes
docker stats                         live resource usage
docker stats container               specific container
docker diff container                filesystem changes

# Logs
docker logs container
docker logs -f container             follow
docker logs --tail 100 container     last 100 lines
docker logs --since 5m container     last 5 minutes
docker logs --timestamps container
docker logs container 2>&1 | grep ERROR
```

---

## Volumes

```bash
docker volume create myvolume
docker volume ls
docker volume inspect myvolume
docker volume rm myvolume
docker volume prune              remove all unused volumes

# Named volume (managed by Docker, persists across containers)
docker run -v myvolume:/data nginx

# Bind mount (maps host directory directly)
docker run -v /host/path:/container/path nginx
docker run -v $(pwd):/app node                    current dir

# Read-only mount
docker run -v myvolume:/data:ro nginx

# tmpfs (RAM, not persisted)
docker run --tmpfs /tmp nginx
```

---

## Networking

```bash
# Network management
docker network create mynet
docker network create --driver bridge mynet
docker network create --driver overlay mynet      Swarm
docker network ls
docker network inspect mynet
docker network rm mynet
docker network prune

# Connect/disconnect
docker network connect mynet container
docker network disconnect mynet container

# Types
bridge      default; containers can communicate by name
host        shares host network stack (Linux only)
none        no networking
overlay     multi-host (Swarm)
macvlan     assign real network interface

# Container DNS
# On a user-defined bridge, containers resolve each other by name
docker network create app-net
docker run -d --name db --network app-net postgres
docker run -d --name api --network app-net myapi
# api can reach db at hostname "db"

# Expose vs Publish
EXPOSE 80         in Dockerfile: document the port (doesn't open it)
-p 8080:80        publish: actually maps host port to container port
-p 80             publish to random host port
-P                publish ALL exposed ports to random host ports
```

---

## Docker Compose

### docker-compose.yml
```yaml
version: "3.9"

services:
  web:
    build:
      context: .
      dockerfile: Dockerfile
      args:
        BUILD_ENV: production
    image: myapp:latest
    container_name: myapp_web
    restart: unless-stopped
    ports:
      - "8080:80"
    environment:
      - NODE_ENV=production
      - DB_HOST=db
    env_file:
      - .env
    volumes:
      - ./uploads:/app/uploads
      - static_files:/app/static
    networks:
      - frontend
      - backend
    depends_on:
      db:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/health"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 10s
    deploy:
      resources:
        limits:
          cpus: "0.5"
          memory: 512M
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  db:
    image: postgres:15-alpine
    restart: unless-stopped
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: myapp
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    volumes:
      - db_data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    networks:
      - backend
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U myapp"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    command: redis-server --appendonly yes --requirepass "${REDIS_PASSWORD}"
    volumes:
      - redis_data:/data
    networks:
      - backend

  nginx:
    image: nginx:alpine
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./certs:/etc/nginx/certs:ro
      - static_files:/usr/share/nginx/html:ro
    networks:
      - frontend
    depends_on:
      - web

volumes:
  db_data:
  redis_data:
  static_files:

networks:
  frontend:
  backend:
    internal: true              no internet access

secrets:
  db_password:
    file: ./secrets/db_password.txt
```

### Compose Commands
```bash
docker compose up              start everything (foreground)
docker compose up -d           start detached
docker compose up --build      rebuild images first
docker compose up web          start specific service
docker compose up --scale web=3   scale service

docker compose down            stop + remove containers
docker compose down -v         also remove volumes
docker compose down --rmi all  also remove images

docker compose ps              list containers
docker compose logs            all logs
docker compose logs -f web     follow specific service
docker compose exec web bash   shell in running container
docker compose run web bash    new one-off container

docker compose build           build all images
docker compose build web       build specific image
docker compose pull            pull latest images
docker compose push            push to registry

docker compose config          validate + display config
docker compose top             processes in each service
docker compose pause / unpause
docker compose restart web
```

---

## Dockerfile Deep Dive

```dockerfile
# ============================================
# Best-practice Dockerfile
# ============================================

# Use specific version tags, never :latest
FROM node:20.10-alpine3.18 AS base

# Metadata (OCI standard)
LABEL maintainer="team@example.com"
LABEL org.opencontainers.image.title="My App"
LABEL org.opencontainers.image.version="1.0.0"
LABEL org.opencontainers.image.source="https://github.com/org/repo"

# Set working directory (creates if doesn't exist)
WORKDIR /app

# Create non-root user early
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# Copy dependency manifests FIRST (better cache)
COPY package.json package-lock.json ./

# Install dependencies (cached until package.json changes)
RUN npm ci --only=production && npm cache clean --force

# Copy source code AFTER deps (cache optimization)
COPY --chown=appuser:appgroup . .

# Build
RUN npm run build

# Switch to non-root
USER appuser

# Document the port
EXPOSE 3000

# Use ENTRYPOINT + CMD for flexible invocation
ENTRYPOINT ["node"]
CMD ["dist/server.js"]

# Health check
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD node healthcheck.js || exit 1
```

### Key Directives
```dockerfile
FROM        base image
ARG         build-time variable (not in final image)
ENV         environment variable (persists in image)
RUN         execute command (creates layer)
COPY        copy files from build context
ADD         like COPY but handles URLs + auto-extracts archives
WORKDIR     set working directory
USER        set user
EXPOSE      document port (doesn't publish)
VOLUME      declare volume mount point
CMD         default command (overridable)
ENTRYPOINT  fixed command (CMD appends to it)
HEALTHCHECK define health check
SHELL       override default shell
STOPSIGNAL  signal to stop container
ONBUILD     trigger for child images
```

### ARG vs ENV
```dockerfile
ARG BUILD_VERSION          # available only during build
ENV APP_VERSION=$BUILD_VERSION  # available at runtime too

# Build: docker build --build-arg BUILD_VERSION=1.2.3 .
# NEVER use ARG/ENV for secrets — they appear in image history
```

### Cache optimization order
```dockerfile
# Order from least to most frequently changed:
# 1. Base image
# 2. System dependencies (apt-get)
# 3. App dependencies (package.json)
# 4. Configuration files
# 5. Application code
```

---

## Multi-stage Builds

Drastically reduce final image size by not including build tools:

```dockerfile
# ============================================
# Go application
# ============================================
FROM golang:1.21-alpine AS builder
WORKDIR /build
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o server .

FROM scratch AS final          # empty image!
COPY --from=builder /build/server /server
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
EXPOSE 8080
USER 65534:65534               # nobody user in scratch
ENTRYPOINT ["/server"]

# ============================================
# Node.js application
# ============================================
FROM node:20-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci

FROM node:20-alpine AS build
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

FROM node:20-alpine AS production
ENV NODE_ENV=production
WORKDIR /app
RUN addgroup -S app && adduser -S app -G app
COPY --from=build --chown=app:app /app/dist ./dist
COPY --from=deps --chown=app:app /app/node_modules ./node_modules
USER app
EXPOSE 3000
CMD ["node", "dist/index.js"]

# ============================================
# Python application
# ============================================
FROM python:3.12-slim AS builder
RUN pip install poetry
WORKDIR /app
COPY pyproject.toml poetry.lock ./
RUN poetry export -f requirements.txt --output requirements.txt --without-hashes

FROM python:3.12-slim AS final
WORKDIR /app
COPY --from=builder /app/requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY --chown=1000:1000 . .
USER 1000
EXPOSE 8000
CMD ["gunicorn", "app:app", "-w", "4", "-b", "0.0.0.0:8000"]
```

---

## Registry & Distribution

```bash
# Docker Hub
docker login
docker login -u username -p password     non-interactive
docker logout

docker tag myapp:latest username/myapp:1.0.0
docker push username/myapp:1.0.0
docker push username/myapp:latest

# Private registry
docker login registry.company.com
docker tag myapp registry.company.com/team/myapp:1.0.0
docker push registry.company.com/team/myapp:1.0.0

# AWS ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  123456789.dkr.ecr.us-east-1.amazonaws.com

# GCR
gcloud auth configure-docker gcr.io
docker tag myapp gcr.io/project-id/myapp:latest
docker push gcr.io/project-id/myapp:latest

# Building with metadata
docker build \
  --build-arg BUILD_DATE=$(date -u +'%Y-%m-%dT%H:%M:%SZ') \
  --build-arg VCS_REF=$(git rev-parse --short HEAD) \
  --tag myapp:$(git rev-parse --short HEAD) \
  .
```

---

## Security

```bash
# Scan images for vulnerabilities
docker scout cves nginx          # Docker Scout
trivy image nginx                # Trivy (open source)
grype nginx                      # Grype

# Security best practices checklist:
# ✓ Use non-root user (USER directive)
# ✓ Use read-only filesystem (--read-only)
# ✓ Drop capabilities (--cap-drop ALL)
# ✓ Use no-new-privileges
# ✓ Scan base images regularly
# ✓ Pin image versions (not :latest)
# ✓ Don't store secrets in images or Dockerfiles
# ✓ Use .dockerignore
# ✓ Multi-stage builds (no build tools in prod)

# .dockerignore
cat > .dockerignore << 'EOF'
.git
.gitignore
.dockerignore
node_modules
npm-debug.log
Dockerfile
docker-compose*.yml
*.md
.env
.env.*
coverage/
dist/
EOF

# Docker secrets (Compose)
docker secret create my_secret ./secret_file
docker service create --secret my_secret myimage

# Check container as root is running?
docker inspect container --format '{{.Config.User}}'

# Drop capabilities
docker run --cap-drop ALL --cap-add NET_BIND_SERVICE nginx

# Read-only with allowed write paths
docker run --read-only \
  --tmpfs /tmp \
  --tmpfs /run \
  -v data:/var/lib/data \
  myapp
```

---

## Performance & Debugging

```bash
# Stats
docker stats                     all containers
docker stats --no-stream         snapshot (no follow)
docker stats --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}"

# Events
docker events                    real-time events
docker events --filter event=die     just crashes

# Debugging a running container
docker exec -it container bash
docker exec -it container sh    # if no bash
docker attach container         # attach to process stdout/stdin
docker logs container --follow

# Debugging a stopped container
docker commit stopped-container debug-image
docker run -it debug-image bash

# Inspect filesystem
docker run --rm -it --pid=container:mycontainer \
  --net=container:mycontainer \
  --cap-add sys_admin \
  ubuntu nsenter -t 1 -m -u -i -n sh

# Network debugging in container
docker run --rm -it \
  --network container:mycontainer \
  nicolaka/netshoot             # has all network tools

# Image size analysis
docker image inspect nginx --format='{{.Size}}'
dive nginx                      # explore layers interactively

# Check what changed
docker diff mycontainer
```

---

## Cleanup

```bash
# Containers
docker rm $(docker ps -aq)              remove all stopped
docker container prune                  same but safer
docker container prune -f               no confirmation

# Images
docker rmi $(docker images -q)          remove all
docker image prune                      dangling images
docker image prune -a                   all unused

# Volumes
docker volume prune                     unused volumes

# Networks
docker network prune

# Everything (nuclear option)
docker system prune                     stopped containers + dangling images + unused networks
docker system prune -a                  also: all unused images
docker system prune -a --volumes        also: volumes

# Check disk usage
docker system df
docker system df -v                     verbose
```

---

## Production Patterns

### Health checks
```dockerfile
# Web server
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
  CMD curl -f http://localhost/health || exit 1

# Database
HEALTHCHECK --interval=10s --timeout=5s --retries=5 \
  CMD pg_isready -U postgres || exit 1
```

### Graceful shutdown
```dockerfile
STOPSIGNAL SIGTERM
# In your app, handle SIGTERM to drain connections before exit
```

### Init process
```bash
# Use tini as init to properly handle signals and zombies
FROM node:20-alpine
RUN apk add --no-cache tini
ENTRYPOINT ["/sbin/tini", "--"]
CMD ["node", "server.js"]

# Or: docker run --init myimage
```

### Environment-specific configs
```yaml
# docker-compose.override.yml (dev only, git-ignored)
services:
  web:
    build:
      target: development
    volumes:
      - .:/app                 # live reload
    environment:
      - DEBUG=true
    command: npm run dev
```

### Logging
```bash
# Production: use syslog or external log driver
docker run --log-driver=syslog --log-opt syslog-address=udp://loghost:514 nginx
docker run --log-driver=awslogs --log-opt awslogs-group=/myapp/prod nginx

# In compose:
logging:
  driver: "json-file"
  options:
    max-size: "10m"
    max-file: "5"
```
