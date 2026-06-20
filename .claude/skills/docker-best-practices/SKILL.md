---
name: docker-best-practices
description: Shipping lean, safe containers — multi-stage builds, Compose setups, image slimming, and security defaults
---

# Working with Docker

## Multi-stage builds

```dockerfile
FROM node:22-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --only=production

FROM node:22-alpine AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:22-alpine AS runtime
WORKDIR /app
RUN addgroup -g 1001 -S appgroup && adduser -S appuser -u 1001 -G appgroup
COPY --from=deps /app/node_modules ./node_modules
COPY --from=build /app/dist ./dist
COPY --from=build /app/package.json ./
USER appuser
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=3s CMD wget -qO- http://localhost:3000/healthz || exit 1
CMD ["node", "dist/server.js"]
```

Keep dependency installs in their own stage, apart from the build, so the final image carries nothing but what it needs to run.

## The same idea in Python

```dockerfile
FROM python:3.12-slim AS builder
WORKDIR /app
RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

FROM python:3.12-slim
WORKDIR /app
RUN useradd --create-home appuser
COPY --from=builder /opt/venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"
COPY . .
USER appuser
CMD ["gunicorn", "app:create_app()", "-b", "0.0.0.0:8000", "-w", "4"]
```

## Docker Compose

```yaml
services:
  api:
    build:
      context: .
      dockerfile: Dockerfile
      target: runtime
    ports:
      - "3000:3000"
    environment:
      - DATABASE_URL=postgres://user:pass@db:5432/app
      - REDIS_URL=redis://cache:6379
    depends_on:
      db:
        condition: service_healthy
      cache:
        condition: service_started
    restart: unless-stopped
    deploy:
      resources:
        limits:
          memory: 512M

  db:
    image: postgres:16-alpine
    volumes:
      - pgdata:/var/lib/postgresql/data
    environment:
      POSTGRES_DB: app
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user -d app"]
      interval: 5s
      timeout: 3s
      retries: 5

  cache:
    image: redis:7-alpine
    command: redis-server --maxmemory 128mb --maxmemory-policy allkeys-lru

volumes:
  pgdata:
```

## .dockerignore

```
node_modules
.git
.env*
*.md
docker-compose*.yml
.github
coverage
dist
```

A `.dockerignore` is non-optional: it shrinks the build context and keeps secrets from sneaking into an image.

## Slimming images

```bash
# Check image size breakdown
docker history --human --no-trunc <image>

# Use dive for layer analysis
dive <image>

# Multi-arch build
docker buildx build --platform linux/amd64,linux/arm64 -t registry/app:1.0 --push .
```

Fold related `RUN` steps together to cut layer count, and order instructions from least- to most-frequently-changing so the cache survives longer.

## What to avoid

- Letting the container process run as root
- Using `ADD` where `COPY` would do — `ADD` quietly unpacks tarballs and fetches URLs
- Baking secrets into `ENV` lines in the Dockerfile
- Floating base images (`FROM node:latest`) instead of pinning them
- Skipping `.dockerignore` and dragging a huge context into every build
- Shipping production images that still carry dev dependencies

## Before you ship

- [ ] A multi-stage build keeps the build and runtime stages distinct
- [ ] A non-root user is created and selected with `USER`
- [ ] Base images are pinned to a concrete tag (e.g. `node:22-alpine`)
- [ ] `.dockerignore` drops `.git`, `node_modules`, and `.env`
- [ ] A `HEALTHCHECK` is defined
- [ ] The production image has no build tools or dev dependencies left in it
- [ ] Compose services wait on each other with health-conditioned `depends_on`
- [ ] Secrets arrive via build secrets or runtime mounts, never `ENV` in the Dockerfile
