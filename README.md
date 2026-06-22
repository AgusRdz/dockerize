# dockerize

> A Claude Code plugin that generates complete Docker environments — dev with hot reload, production-ready secure images, and audits of existing setups.

No more Googling "Dockerfile best practices" for every new project. Drop `/dockerize` in any repo and get a full, working Docker setup tailored to your stack.

## What it does

**Audit** — reads your existing `Dockerfile`, `docker-compose.yml`, and `.dockerignore`, scores every finding by severity (CRITICAL / HIGH / MEDIUM / LOW), and offers to fix them in-place.

**Dev** — generates a dev-first environment: hot reload, source volume mounts, debug ports, HMR WebSocket ports, named volumes for package caches, and polling flags for Docker Desktop on Mac/Windows.

**Prod** — generates slim, secure production images: multi-stage builds, non-root user, pinned tags, health checks, minimal final images, zero dev tooling.

## Installation

```bash
/plugins install github:AgusRdz/dockerize
```

Or from a local clone:

```bash
/plugins install /path/to/dockerize
```

## Usage

```
/dockerize
```

The skill auto-detects your stack and asks whether you want dev, prod, audit, or all of the above. You can skip detection with a stack hint:

```
/dockerize laravel
/dockerize dotnet
/dockerize node
/dockerize go
```

## Output

```
Dockerfile                    ← production: multi-stage, non-root, pinned
Dockerfile.dev                ← dev: full toolchain, source mounted at runtime
.dockerignore                 ← stack-appropriate exclusions
docker-compose.yml            ← base: shared services, networks, named volumes
docker-compose.override.yml   ← dev: auto-loaded by `docker compose up`
docker-compose.prod.yml       ← prod: restart policy, resource limits, logging
docker/                       ← nginx.conf, supervisord.conf, php configs
.air.toml                     ← Go live-reload config
Makefile                      ← optional: Docker shortcuts + package manager wrappers
```

## Dev workflow

```bash
# Without Makefile
docker compose up        # hot reload, debug ports live
docker compose down

# With Makefile
make up                  # start dev
make logs                # tail all logs
make shell               # sh into the app container
make down                # stop everything
```

## Production workflow

```bash
# Without Makefile
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d

# With Makefile
make build-prod
make up-prod
```

## Makefile — package manager wrappers

All commands run inside the container. No local toolchain required.

```bash
# Node.js
make npm cmd="install express"
make npm-install
make npm-test

# Angular
make ng cmd="generate component dashboard"
make ng-test

# Bun
make bun cmd="add hono"
make bun-test

# Composer / Laravel
make composer cmd="require laravel/horizon"
make artisan cmd="migrate"
make migrate-fresh
make tinker
make queue

# Symfony
make console cmd="doctrine:migrations:migrate"
make db-migrate
make cache-clear

# .NET
make dotnet cmd="add package Newtonsoft.Json"
make dotnet-test
make ef-add name="AddUserTable"
make ef-update

# Go
make go cmd="get github.com/gin-gonic/gin"
make go-test
make go-tidy

# Python
make pip cmd="install requests"
make pytest
make manage cmd="migrate"      # Django
make makemigrations            # Django
```

## Supported stacks

| Stack | Dev hot reload | Prod final image |
|-------|---------------|-----------------|
| .NET Framework 4.x | `dotnet watch run` | `aspnet:4.8-windowsservercore` |
| .NET Core / .NET 5+ | `dotnet watch run` | `dotnet/aspnet:X.Y-alpine` |
| Angular (static SPA) | `ng serve --host 0.0.0.0` | `nginx:1.27-alpine` |
| Angular (SSR) | `ng serve` | `node:22-alpine` |
| Node.js | `npm run dev` / `--watch` | `node:22-alpine` |
| Bun | `bun --watch` | `oven/bun:1.2-alpine` |
| Deno | `deno run --watch` | `denoland/deno:X.Y.Z` |
| PHP — Laravel | `artisan serve` + Vite service | `php:8.3-fpm-alpine` + nginx |
| PHP — Symfony | `bin/console server:run` | `php:8.3-fpm-alpine` + nginx |
| Go | `air` (live reload) + `dlv` (debugger) | `distroless/static-debian12:nonroot` |
| Python | `uvicorn --reload` / `runserver` | `python:X.Y-slim` |

## Audit checks

### Production

| Severity | Check |
|----------|-------|
| CRITICAL | Final stage runs as root |
| CRITICAL | `.env` or secrets `COPY`'d into image |
| CRITICAL | Secrets passed via `ARG`/`ENV` (visible in `docker history`) |
| HIGH | Base image uses `latest` or no tag |
| HIGH | No multi-stage build — build tools ship in runtime image |
| HIGH | No `.dockerignore` |
| HIGH | `COPY . .` before dependency install — cache busted on every change |
| MEDIUM | No `HEALTHCHECK` |
| MEDIUM | `npm install` instead of `npm ci` |
| MEDIUM | `apt-get` without `--no-install-recommends` |
| MEDIUM | Fat final image when alpine/slim/distroless is available |

### Dev

| Severity | Check |
|----------|-------|
| HIGH | No source volume mount — changes require full rebuild |
| HIGH | `node_modules` overwritten by bind mount |
| HIGH | Dev dependencies not installed |
| MEDIUM | Production `CMD` used in dev container |
| MEDIUM | Debug port not exposed |
| MEDIUM | HMR WebSocket port not exposed |

### docker-compose

| Severity | Check |
|----------|-------|
| CRITICAL | Secrets as plain `environment:` strings |
| HIGH | No `condition: service_healthy` on database dependency |
| HIGH | No `restart:` policy in production |

## License

MIT
