---
name: dockerize
description: >-
  Docker configuration for any project — generate dev/prod setups or audit and
  improve existing ones. Triggers on: "dockerize this", "add Docker", "create
  Dockerfile", "containerize", "set up docker-compose", "audit my Docker setup",
  "improve my Dockerfile", "review my Docker config", "Docker for local dev",
  "hot reload in Docker". Supports .NET Framework, .NET Core/5+, Angular (SSR
  and static), Node.js, Bun, Deno, PHP (Laravel, Symfony, generic), Go, Python.
version: 2.0.0
---

# Dockerize Skill

Three modes: **Audit** an existing Docker setup, generate a **Dev** environment, or generate a **Production** configuration. The goal is never just "a Dockerfile" — it's a complete, working environment for each context.

**Output rules — follow strictly:**
- All output is plain markdown rendered in the terminal. No exceptions.
- Never invoke `artifact-design`, `Artifact`, or any other rendering skill.
- Never create web pages, HTML files, or visual artifacts.
- Findings tables, proposed fixes, and checklists are plain fenced markdown blocks.

---

## Phase 0 — Mode Detection

Run these probes in parallel:

```
Glob: Dockerfile
Glob: Dockerfile.*
Glob: docker-compose*.yml
Glob: .dockerignore
```

**Branch:**

| Condition | Mode |
|-----------|------|
| Any Docker file found | → **Ask**: "Audit and improve existing setup, regenerate, or add missing mode (dev/prod)?" |
| No Docker files found | → **Ask**: "Generate dev config, prod config, or both?" |

When generating both, generate dev first (simpler to validate), then prod.

---

## AUDIT MODE

### A1 — Read Everything Docker-Related

Read all of: `Dockerfile`, `Dockerfile.dev`, `Dockerfile.*`, `docker-compose*.yml`, `.dockerignore`.

Also run stack detection (same as Phase 1 below) — needed to evaluate whether base images match the stack and whether hot-reload tooling is correct.

### A2 — Score Against Rubric

Evaluate every finding. Assign severity. Never skip a finding because it "seems minor".

#### Production Dockerfile Checks

| ID | Severity | Check |
|----|----------|-------|
| P01 | CRITICAL | Final stage runs as root (no `USER` instruction, or `USER root`) |
| P02 | CRITICAL | `.env`, credentials, or secret files are `COPY`'d into any stage |
| P03 | CRITICAL | `ARG` or `ENV` used to pass secrets (visible in `docker history`) |
| P04 | HIGH | Base image uses `latest` or has no tag |
| P05 | HIGH | No multi-stage build — build tools (sdk, compiler, node_modules devDeps) ship in final image |
| P06 | HIGH | No `.dockerignore` file (or one that doesn't exclude `node_modules`, `bin/obj`, `.git`, etc.) |
| P07 | HIGH | `COPY . .` before dependency install (busts layer cache on every source change) |
| P08 | MEDIUM | No `HEALTHCHECK` instruction |
| P09 | MEDIUM | `apt-get install` without `--no-install-recommends` and `rm -rf /var/lib/apt/lists/*` |
| P10 | MEDIUM | `npm install` instead of `npm ci` (non-reproducible) |
| P11 | MEDIUM | `pip install` without `--no-cache-dir` |
| P12 | MEDIUM | `RUN` chains that install then remove in separate layers (wasted space) |
| P13 | MEDIUM | Final image is full SDK/runtime when a slimmer option exists (alpine, slim, distroless) |
| P14 | LOW | `COPY` without `--chown` followed by a separate `RUN chown` (extra layer) |
| P15 | LOW | `WORKDIR` not set (operations against `/`) |
| P16 | LOW | `EXPOSE` missing, wrong port, or mismatches app config |
| P17 | LOW | `ADD` used where `COPY` suffices (ADD has implicit tar extraction and URL fetch) |
| P18 | HIGH | `|| true` or `2>/dev/null` silences failures in `RUN` — broken builds ship silently |

#### Development Docker Checks

| ID | Severity | Check |
|----|----------|-------|
| D01 | HIGH | No source volume mount (code changes require full rebuild — no hot reload) |
| D02 | HIGH | `node_modules` overwritten by volume mount (anonymous volume not defined) |
| D03 | HIGH | Dev dependencies not installed (linters, type checkers, test runners unavailable) |
| D04 | MEDIUM | Hot-reload command not configured (using production `CMD`) |
| D05 | MEDIUM | Debug/inspect port not exposed (no remote debugger support) |
| D06 | MEDIUM | HMR WebSocket port not exposed (browser can't receive hot updates) |
| D07 | MEDIUM | Source watching uses polling but `CHOKIDAR_USEPOLLING` or `--poll` not set (misses changes on bind mounts on some hosts) |
| D08 | LOW | Deno `DENO_DIR` not persisted as named volume (re-downloads deps on every restart) |
| D09 | LOW | Go dev image doesn't include a file watcher (`air`, `reflex`) |

#### docker-compose Checks

| ID | Severity | Check |
|----|----------|-------|
| C01 | CRITICAL | Secrets in `environment:` as plain strings (use `env_file:` or secret refs) |
| C02 | HIGH | No health checks on dependent services (app starts before DB is ready — use `condition: service_healthy`) |
| C03 | HIGH | No `restart:` policy on production services |
| C04 | MEDIUM | Missing `depends_on` — app can start before database |
| C05 | MEDIUM | Database port exposed to host in production (unnecessary attack surface) |
| C06 | LOW | No named volumes for persistent data (anonymous volumes are lost on `down`) |

### A3 — Report Findings

Output the full audit report. For every finding, include:
1. The finding itself (what is wrong and where)
2. The proposed fix (exactly what the change would look like — a concrete diff or the replacement snippet)

Format:

```
## Docker Audit Report

### Production Dockerfile
| ID  | Severity | Finding                                   | Line |
|-----|----------|-------------------------------------------|------|
| P01 | CRITICAL | Final stage has no USER — runs as root    | —    |
| P04 | HIGH     | FROM node:latest — unpinned tag           | 1    |
| P07 | HIGH     | COPY . . on line 8 before npm ci on line 9| 8    |
| P08 | MEDIUM   | No HEALTHCHECK instruction                | —    |

### Development Setup
| ID  | Severity | Finding                                   |
|-----|----------|-------------------------------------------|
| D01 | HIGH     | No Dockerfile.dev or source volume mount  |
| D02 | HIGH     | No anonymous volume for node_modules      |

### docker-compose.yml
| ID  | Severity | Finding                                   | Line |
|-----|----------|-------------------------------------------|------|
| C02 | HIGH     | depends_on db missing condition: service_healthy | 14 |

Score: X/17 production checks passed, Y/9 dev checks passed.

---

### Proposed Fixes

**P01 — Add non-root user (Dockerfile, final stage)**
```diff
+ RUN addgroup -S appgroup && adduser -S appuser -G appgroup
  COPY --from=build /app/dist ./dist
+ USER appuser
```

**P04 — Pin base image tag (Dockerfile, line 1)**
```diff
- FROM node:latest
+ FROM node:22-alpine
```

*(one block per finding)*
```

### A4 — Confirm Before Fixing

**HARD STOP. Do not touch any file yet.**

After presenting the report and proposed fixes, output exactly this prompt and wait for the user's response:

```
---
Ready to apply fixes. How would you like to proceed?

  [1] Fix all findings automatically
  [2] Fix by ID — specify which (e.g. "fix P01 P04")
  [3] Discuss a specific finding before fixing
  [4] Skip — I'll handle these manually

Your choice:
```

Only proceed to apply changes after the user explicitly responds. Never auto-apply.

**Applying fixes (after confirmation):**

- Apply one finding at a time using Edit. After each file is changed, output a one-line summary: `✓ P01 fixed — added non-root user in final stage`.
- When fixing P04 (unpinned tag): read the actual version from `package.json`, `.csproj`, `go.mod`, etc. — never guess.
- When fixing P05 (no multi-stage): rewrite the Dockerfile using the correct template from Phase 4-Prod. Preserve any custom `ENV`, `LABEL`, `ARG`, and `RUN` instructions not in the standard template.
- After all selected fixes are applied, re-run the checks for the fixed findings only and confirm they now pass.

---

## GENERATE MODE

### Phase 1 — Stack Detection

Probe in parallel:

```
Glob: **/*.csproj
Glob: **/*.sln
Glob: angular.json
Glob: package.json
Glob: bun.lockb
Glob: bunfig.toml
Glob: deno.json
Glob: deno.jsonc
Glob: go.mod
Glob: artisan
Glob: symfony.lock
Glob: composer.json
Glob: requirements.txt
Glob: pyproject.toml
Glob: Pipfile
```

**Priority table** (evaluate top-to-bottom; multiple stacks = ask user):

| Signal | Stack |
|--------|-------|
| `*.csproj` with `<TargetFramework>net4` | .NET Framework 4.x |
| `*.csproj` with `<TargetFramework>net[5-9]` or `net8.0` etc. | .NET Core / .NET 5+ |
| `angular.json` | Angular — grep `@angular/ssr` or `outputMode: 'server'` to detect SSR |
| `bun.lockb` or `bunfig.toml` | Bun (takes priority over Node.js) |
| `deno.json` or `deno.jsonc` | Deno |
| `artisan` | Laravel |
| `symfony.lock` | Symfony |
| `composer.json` (no artisan/symfony) | PHP generic |
| `package.json` (no angular/bun/deno) | Node.js |
| `go.mod` | Go |
| `requirements.txt` / `pyproject.toml` / `Pipfile` | Python |

---

### Phase 2 — Version Extraction

Read the config file to pick the pinned base image tag. Never use `latest`.

| Stack | Source | How |
|-------|--------|-----|
| .NET Framework | `.csproj` | `<TargetFrameworkVersion>` → e.g. `v4.8` → `4.8` |
| .NET Core/5+ | `.csproj` | `<TargetFramework>` → `net8.0` → `8.0` |
| Angular / Node.js | `package.json` | `engines.node`, or `.nvmrc`, or `.node-version`; fall back to LTS (22) |
| Bun | `package.json` or `bunfig.toml` | `engines.bun`; fall back to `1.2` |
| Deno | `deno.json` | `version` field; fall back to latest stable |
| PHP | `composer.json` | `require.php`; fall back to `8.3` |
| Go | `go.mod` | `go X.Y` line |
| Python | `pyproject.toml` `python_requires`, or `.python-version` | fall back to `3.13` |

---

### Phase 3 — Plan

Before writing anything, present the plan:

```
Stack   : <stack>
Version : <version>
Mode(s) : Dev / Prod / Both

Files to generate:
  Dockerfile                (prod: multi-stage, non-root, pinned)
  Dockerfile.dev            (dev: full toolchain, source mounted at runtime)
  .dockerignore
  docker-compose.yml        (base: shared services, named volumes, networks)
  docker-compose.override.yml  (dev: source mounts, debug ports, hot-reload cmd)
  docker-compose.prod.yml   (prod: restart policy, resource limits, no dev ports)
  docker/nginx.conf         (if Angular static or PHP)
  docker/supervisord.conf   (if PHP with FPM + Nginx in one container)
  Makefile                  (optional — ask user)

Dev features:
  Hot reload  : <method>
  Debug port  : <port>
  HMR socket  : <port if applicable>

Prod image : <final-image:tag>
Exposed    : <port>
Non-root   : <username>
```

Ask:
- "Should I also generate a `Makefile` with Docker shortcuts and package manager wrappers? (y/n)"

---

### Phase 4-Dev — Development Configuration

**Philosophy**: the dev container is a development machine, not a runtime. Source is never `COPY`'d — it's mounted. The goal is zero rebuild on code change.

#### Non-negotiable dev rules

1. Source code is a **volume mount** (bind mount) — never `COPY`'d in `Dockerfile.dev`
2. **Dev dependencies installed** — `npm install` (not `--omit=dev`), full SDK, debugger, linters
3. **Hot reload command** as `CMD` — never the production start command
4. **Debug port exposed** — so IDEs can attach remotely
5. **HMR WebSocket port exposed** if the framework uses one
6. `CHOKIDAR_USEPOLLING=1` or `--poll` where bind mounts don't trigger inotify (Docker Desktop on Mac/Windows)
7. **Named volume for package caches** — `node_modules`, `~/.nuget`, `GOPATH/pkg`, `DENO_DIR` — so they survive restarts without being overwritten by the source mount

---

#### .NET Core / .NET 5+ — Dev

**Dockerfile.dev:**
```dockerfile
# syntax=docker/dockerfile:1
FROM mcr.microsoft.com/dotnet/sdk:8.0
WORKDIR /app
EXPOSE 8080 5005

# Restore packages separately — mounted source will override /app
COPY src/**/*.csproj ./src/
RUN dotnet restore

RUN addgroup --system appgroup && adduser --system --ingroup appgroup appuser
USER appuser

# Source is mounted at runtime — do not COPY here
CMD ["dotnet", "watch", "run", "--project", "src/<ProjectName>", \
     "--urls", "http://+:8080"]
```

**docker-compose.override.yml additions:**
```yaml
services:
  app:
    build:
      dockerfile: Dockerfile.dev
    volumes:
      - .:/app
      - nuget_cache:/root/.nuget/packages
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - DOTNET_USE_POLLING_FILE_WATCHER=1
    ports:
      - "8080:8080"
      - "5005:5005"    # VS remote debug / vsdbg

volumes:
  nuget_cache:
```

---

#### Angular (Static and SSR) — Dev

Angular's dev server must bind to `0.0.0.0` inside the container. HMR uses a WebSocket on a different port (Vite: 24678, webpack-dev-server: same port as app).

**Dockerfile.dev:**
```dockerfile
# syntax=docker/dockerfile:1
FROM node:22-alpine
WORKDIR /app
EXPOSE 4200 24678

# Install deps — node_modules stays in image; source mounted over /app
COPY package*.json ./
RUN npm install

# Source mounted at runtime
CMD ["npx", "ng", "serve", \
     "--host", "0.0.0.0", \
     "--port", "4200", \
     "--poll", "500", \
     "--live-reload", "true"]
```

**docker-compose.override.yml additions:**
```yaml
services:
  app:
    build:
      dockerfile: Dockerfile.dev
    volumes:
      - .:/app
      - /app/node_modules    # anonymous volume prevents host override
    environment:
      - CHOKIDAR_USEPOLLING=1
    ports:
      - "4200:4200"
      - "24678:24678"    # Vite HMR WebSocket
```

For SSR, also expose port `4000` (or whatever `server.ts` uses) and add it to the `ng serve` command if applicable. In Angular 19 SSR with Vite, the dev server serves both browser and server bundles on port `4200`.

---

#### Node.js — Dev

Check `package.json` `scripts.dev` or `scripts.start:dev`. Common hot-reload tools: `tsx --watch`, `ts-node-dev`, `nodemon`, `vite`.

**Dockerfile.dev:**
```dockerfile
# syntax=docker/dockerfile:1
FROM node:22-alpine
WORKDIR /app
EXPOSE 3000 9229

COPY package*.json ./
RUN npm install

# Source mounted at runtime
CMD ["npm", "run", "dev"]
```

**docker-compose.override.yml additions:**
```yaml
services:
  app:
    build:
      dockerfile: Dockerfile.dev
    volumes:
      - .:/app
      - /app/node_modules
    environment:
      - NODE_ENV=development
      - CHOKIDAR_USEPOLLING=1
    ports:
      - "3000:3000"
      - "9229:9229"    # Node.js --inspect debugger
```

If no `dev` script found, default `CMD` to `["node", "--watch", "--inspect=0.0.0.0:9229", "src/index.js"]`.

---

#### Bun — Dev

**Dockerfile.dev:**
```dockerfile
# syntax=docker/dockerfile:1
FROM oven/bun:1.2-alpine
WORKDIR /app
EXPOSE 3000 6499

COPY package.json bun.lockb* ./
RUN bun install

CMD ["bun", "--watch", "run", "src/index.ts"]
```

**docker-compose.override.yml additions:**
```yaml
services:
  app:
    build:
      dockerfile: Dockerfile.dev
    volumes:
      - .:/app
      - /app/node_modules
    environment:
      - NODE_ENV=development
    ports:
      - "3000:3000"
      - "6499:6499"    # Bun Inspector (--inspect)
```

Check `package.json` `scripts.dev` and use that instead of the hardcoded `CMD` if found.

---

#### Deno — Dev

Deno's `--watch` flag covers hot reload. Cache must be persisted as a named volume or Deno re-downloads deps on every restart.

**Dockerfile.dev:**
```dockerfile
# syntax=docker/dockerfile:1
FROM denoland/deno:2.3.3
WORKDIR /app
EXPOSE 8000

# Pre-cache dependencies — source mounted at runtime
COPY deno.json* deno.lock* ./
RUN deno install --entrypoint main.ts 2>/dev/null || true

CMD ["deno", "run", \
     "--watch", \
     "--allow-net", \
     "--allow-read", \
     "--allow-env", \
     "main.ts"]
```

**docker-compose.override.yml additions:**
```yaml
services:
  app:
    build:
      dockerfile: Dockerfile.dev
    volumes:
      - .:/app
      - deno_cache:/deno-dir
    environment:
      - DENO_DIR=/deno-dir
    ports:
      - "8000:8000"

volumes:
  deno_cache:
```

---

#### Go — Dev

Go has no built-in hot reload. Use [`air`](https://github.com/air-verse/air) — the standard Go live-reload tool.

**Dockerfile.dev:**
```dockerfile
# syntax=docker/dockerfile:1
FROM golang:1.24-alpine
WORKDIR /app
EXPOSE 8080 2345

RUN go install github.com/air-verse/air@v1.61.7 && \
    go install github.com/go-delve/delve/cmd/dlv@v1.24.2

COPY go.mod go.sum ./
RUN go mod download

# Source mounted at runtime — air watches and rebuilds
CMD ["air", "-c", ".air.toml"]
```

Also generate `.air.toml` at project root:
```toml
root = "."
tmp_dir = "tmp"

[build]
cmd = "go build -gcflags='all=-N -l' -o ./tmp/main ./cmd/server"
bin = "./tmp/main"
include_ext = ["go"]
exclude_dir = ["tmp", "vendor"]

[log]
time = true

[color]
main = "magenta"
watcher = "cyan"
build = "yellow"
runner = "green"
```

**docker-compose.override.yml additions:**
```yaml
services:
  app:
    build:
      dockerfile: Dockerfile.dev
    volumes:
      - .:/app
      - go_cache:/go/pkg/mod
    environment:
      - CGO_ENABLED=0
    ports:
      - "8080:8080"
      - "2345:2345"    # Delve remote debugger

volumes:
  go_cache:
```

---

#### PHP — Laravel — Dev

Laravel dev needs: PHP with Xdebug, Composer, artisan serve or php-fpm, and a Vite dev server for frontend assets.

**Dockerfile.dev:**
```dockerfile
# syntax=docker/dockerfile:1
FROM php:8.3-cli-alpine
WORKDIR /var/www/html
EXPOSE 80 9003

RUN apk add --no-cache git unzip && \
    docker-php-ext-install pdo pdo_mysql && \
    pecl install xdebug && docker-php-ext-enable xdebug

COPY --from=composer:2.8 /usr/bin/composer /usr/bin/composer

# Source mounted at runtime
CMD ["php", "artisan", "serve", "--host=0.0.0.0", "--port=80"]
```

Also generate `docker/xdebug.ini`:
```ini
zend_extension=xdebug
xdebug.mode=debug
xdebug.start_with_request=yes
xdebug.client_host=host.docker.internal
xdebug.client_port=9003
xdebug.log=/tmp/xdebug.log
```

**docker-compose.override.yml additions:**
```yaml
services:
  app:
    build:
      dockerfile: Dockerfile.dev
    volumes:
      - .:/var/www/html
      - ./docker/xdebug.ini:/usr/local/etc/php/conf.d/xdebug.ini
      - composer_cache:/root/.composer
    environment:
      - APP_ENV=local
      - APP_DEBUG=true
    ports:
      - "80:80"
      - "9003:9003"    # Xdebug
    extra_hosts:
      - "host.docker.internal:host-gateway"

  vite:
    image: node:22-alpine
    working_dir: /var/www/html
    volumes:
      - .:/var/www/html
      - /var/www/html/node_modules
    command: npm run dev -- --host
    ports:
      - "5173:5173"
      - "24678:24678"    # HMR WebSocket

volumes:
  composer_cache:
```

---

#### PHP — Symfony — Dev

Replace artisan with Symfony's dev server:

```dockerfile
CMD ["php", "bin/console", "server:run", "0.0.0.0:80"]
```

Or use Symfony CLI if available (`symfony server:start --no-tls`). Adjust Xdebug config identically.

---

#### Python — Dev

**Dockerfile.dev:**
```dockerfile
# syntax=docker/dockerfile:1
FROM python:3.13-slim
WORKDIR /app
EXPOSE 8000

COPY requirements*.txt ./
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt

RUN addgroup --system appgroup && adduser --system --ingroup appgroup appuser
# Source mounted at runtime — USER set after install so pip can write to site-packages
USER appuser
CMD ["uvicorn", "main:app", "--reload", "--host", "0.0.0.0", "--port", "8000"]
```

For Django, replace CMD with:
```dockerfile
CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]
```

**docker-compose.override.yml additions:**
```yaml
services:
  app:
    build:
      dockerfile: Dockerfile.dev
    volumes:
      - .:/app
      - pip_cache:/root/.cache/pip
    environment:
      - PYTHONUNBUFFERED=1
      - DEBUG=True
    ports:
      - "8000:8000"

volumes:
  pip_cache:
```

---

### Phase 4-Prod — Production Configuration

**Philosophy**: the production container is a sealed artifact. Minimal image, no dev tools, no source writability, non-root, health-checked.

#### Non-negotiable production rules

1. **Multi-stage** — build stage has full SDK; runtime stage has nothing it doesn't need
2. **Pinned tags** — `image:X.Y-alpine`, never `latest`
3. **Non-root** — `USER` set in final stage; `COPY --chown` to set ownership at copy time
4. **`HEALTHCHECK`** — always present on runtime stage
5. **No secrets** — no `.env`, no hardcoded creds, no `ARG`/`ENV` secrets
6. **Deps before source** — lockfile + manifest first, then `COPY . .`
7. **Smallest viable final image** — alpine > slim > full; distroless when possible (Go)

---

#### .NET Framework 4.x — Prod

> **Warning**: .NET Framework requires Windows containers. Incompatible with Linux hosts and most CI runners. Strongly recommend migrating to .NET 8+ for cross-platform support.

```dockerfile
# syntax=docker/dockerfile:1
FROM mcr.microsoft.com/dotnet/framework/sdk:4.8 AS build
WORKDIR /src
COPY *.sln* ./
COPY src/**/*.csproj ./src/
RUN nuget restore
COPY . .
RUN msbuild /p:Configuration=Release /p:DeployOnBuild=true \
    /p:WebPublishMethod=FileSystem /p:PublishUrl=C:\publish

FROM mcr.microsoft.com/dotnet/framework/aspnet:4.8-windowsservercore-ltsc2022 AS runtime
WORKDIR /inetpub/wwwroot
COPY --from=build C:\publish .
HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
  CMD powershell -command \
    "try { (Invoke-WebRequest http://localhost/health -UseBasicParsing).StatusCode -eq 200 } catch { exit 1 }"
```

---

#### .NET Core / .NET 5+ — Prod

Map `<TargetFramework>` to version. `Microsoft.NET.Sdk.Web` → `aspnet` image; console/worker → `runtime` image.

```dockerfile
# syntax=docker/dockerfile:1
FROM mcr.microsoft.com/dotnet/sdk:8.0-alpine AS build
WORKDIR /src
COPY *.sln* ./
COPY src/**/*.csproj ./src/
RUN dotnet restore
COPY . .
RUN dotnet publish src/<ProjectName>/<ProjectName>.csproj \
    -c Release -o /app/publish \
    --no-restore \
    /p:UseAppHost=false

FROM mcr.microsoft.com/dotnet/aspnet:8.0-alpine AS runtime
WORKDIR /app
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
COPY --from=build --chown=appuser:appgroup /app/publish .
USER appuser
EXPOSE 8080
ENV ASPNETCORE_URLS=http://+:8080 \
    DOTNET_RUNNING_IN_CONTAINER=true
HEALTHCHECK --interval=30s --timeout=10s --start-period=30s --retries=3 \
  CMD wget -q -O- http://localhost:8080/health || exit 1
ENTRYPOINT ["dotnet", "<ProjectName>.dll"]
```

---

#### Angular — Static SPA (Nginx) — Prod

Read `outputPath` from `angular.json` to find the browser output dir.

```dockerfile
# syntax=docker/dockerfile:1
FROM node:22-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN --mount=type=cache,target=/root/.npm npm ci
COPY . .
RUN npm run build

# nginxinc/nginx-unprivileged runs entirely as uid 101 — no root needed, listens on 8080
FROM nginxinc/nginx-unprivileged:1.27-alpine AS runtime
COPY --from=build /app/dist/<project-name>/browser /usr/share/nginx/html
COPY docker/nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
  CMD wget -q -O- http://localhost:8080/health || exit 1
```

> **Why `nginxinc/nginx-unprivileged`**: standard `nginx` master process binds port 80 as root; switching `USER` before it starts crashes the container. The unprivileged variant runs the entire server as uid 101, listens on 8080, and needs zero privilege at any point.

**docker/nginx.conf** (also generate):
```nginx
server {
    listen 8080;
    server_tokens off;
    root /usr/share/nginx/html;
    index index.html;

    gzip on;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml image/svg+xml;

    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    location / {
        try_files $uri $uri/ /index.html;
    }

    location ~* \.(js|css|png|jpg|jpeg|gif|svg|ico|woff2?)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    location /health {
        access_log off;
        return 200 "ok";
        add_header Content-Type text/plain;
    }
}
```

---

#### Angular — SSR — Prod

```dockerfile
# syntax=docker/dockerfile:1
FROM node:22-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN --mount=type=cache,target=/root/.npm npm ci
COPY . .
RUN npm run build

FROM node:22-alpine AS runtime
WORKDIR /app
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
COPY --from=build --chown=appuser:appgroup /app/dist/<project-name> .
USER appuser
EXPOSE 4000
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
  CMD wget -q -O- http://localhost:4000/ || exit 1
CMD ["node", "server/server.mjs"]
```

---

#### Node.js — Prod

3-stage if `build` script exists (TypeScript), 2-stage for plain JS.

```dockerfile
# syntax=docker/dockerfile:1
FROM node:22-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN --mount=type=cache,target=/root/.npm npm ci --omit=dev

FROM node:22-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN --mount=type=cache,target=/root/.npm npm ci
COPY . .
RUN npm run build

FROM node:22-alpine AS runtime
WORKDIR /app
COPY --from=deps --chown=node:node /app/node_modules ./node_modules
COPY --from=build --chown=node:node /app/dist ./dist
USER node
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
  CMD wget -q -O- http://localhost:3000/health || exit 1
CMD ["node", "dist/index.js"]
```

---

#### Bun — Prod

```dockerfile
# syntax=docker/dockerfile:1
FROM oven/bun:1.2-alpine AS deps
WORKDIR /app
COPY bun.lockb* package.json ./
RUN bun install --frozen-lockfile --production

FROM oven/bun:1.2-alpine AS build
WORKDIR /app
COPY bun.lockb* package.json ./
RUN bun install --frozen-lockfile
COPY . .
RUN bun run build

FROM oven/bun:1.2-alpine AS runtime
WORKDIR /app
COPY --from=deps --chown=bun:bun /app/node_modules ./node_modules
COPY --from=build --chown=bun:bun /app/dist ./dist
COPY --chown=bun:bun package.json .
USER bun
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
  CMD wget -q -O- http://localhost:3000/health || exit 1
CMD ["bun", "run", "dist/index.js"]
```

---

#### Deno — Prod

```dockerfile
# syntax=docker/dockerfile:1
FROM denoland/deno:2.3.3 AS runtime
WORKDIR /app
RUN addgroup --system appgroup && adduser --system --ingroup appgroup appuser
COPY --chown=appuser:appgroup deno.json* deno.lock* ./
RUN deno cache --lock=deno.lock main.ts 2>/dev/null || \
    deno install --entrypoint main.ts 2>/dev/null || true
COPY --chown=appuser:appgroup . .
USER appuser
EXPOSE 8000
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
  CMD curl -f http://localhost:8000/health || exit 1
CMD ["deno", "run", "--allow-net", "--allow-read", "--allow-env", "main.ts"]
```

Use minimum `--allow-*` flags. Read `deno.json` `tasks.start` to find the actual entry point.

---

#### Go — Prod

```dockerfile
# syntax=docker/dockerfile:1
FROM golang:1.24-alpine AS build
# TARGETARCH is injected by BuildKit -- correct binary for amd64 AND arm64
ARG TARGETARCH
WORKDIR /src
COPY go.mod go.sum ./
RUN --mount=type=cache,target=/go/pkg/mod go mod download
COPY . .
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    CGO_ENABLED=0 GOOS=linux GOARCH=${TARGETARCH} \
    go build -ldflags="-w -s" -trimpath \
    -o /app/server ./cmd/server

FROM gcr.io/distroless/static-debian12:nonroot AS runtime
COPY --from=build /app/server /server
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
  CMD ["/server", "-healthcheck"]
ENTRYPOINT ["/server"]
```

> If the binary doesn't support `-healthcheck`, use `gcr.io/distroless/base-debian12:nonroot` (has glibc) or `alpine:3.21` (has `wget`). Distroless `HEALTHCHECK` must use exec form — no shell.
>
> **Multi-arch builds**: `docker buildx build --platform linux/amd64,linux/arm64 -t image:tag .` — BuildKit sets `$TARGETARCH` automatically per platform.

Find the build target by checking for `cmd/` subdirectories in the repo. Common: `./cmd/server`, `./cmd/api`, `./main.go`.

---

#### PHP — Laravel — Prod

```dockerfile
# syntax=docker/dockerfile:1
FROM composer:2.8 AS composer
WORKDIR /app
COPY composer.json composer.lock ./
RUN composer install \
    --no-dev \
    --no-autoloader \
    --no-scripts \
    --no-interaction \
    --prefer-dist
COPY . .
RUN composer dump-autoload --optimize --no-dev --no-scripts

FROM php:8.3-fpm-alpine AS runtime
RUN apk add --no-cache nginx supervisor && \
    docker-php-ext-install pdo pdo_mysql opcache && \
    docker-php-ext-enable opcache
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
WORKDIR /var/www/html
COPY --from=composer --chown=appuser:appgroup /app .
RUN chown -R appuser:appgroup storage bootstrap/cache && \
    chmod -R 775 storage bootstrap/cache

# Caches run at container startup (not build time) — they need the real env vars
# and may need DB access. Generate docker/entrypoint.sh (see below).
COPY docker/entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh
COPY docker/nginx.conf /etc/nginx/nginx.conf
COPY docker/php-fpm.conf /usr/local/etc/php-fpm.d/www.conf
COPY docker/supervisord.conf /etc/supervisor/conf.d/supervisord.conf
COPY docker/opcache.ini /usr/local/etc/php/conf.d/opcache.ini
EXPOSE 80
HEALTHCHECK --interval=30s --timeout=10s --start-period=30s --retries=3 \
  CMD wget -q -O- http://localhost/up || exit 1
ENTRYPOINT ["/entrypoint.sh"]
CMD ["/usr/bin/supervisord", "-c", "/etc/supervisor/conf.d/supervisord.conf"]
```

Also generate `docker/nginx.conf`, `docker/supervisord.conf`, `docker/opcache.ini`, `docker/php-fpm.conf`.

**docker/nginx.conf:**
```nginx
events {}
http {
    include mime.types;
    server {
        listen 80;
        root /var/www/html/public;
        index index.php;
        location / {
            try_files $uri $uri/ /index.php?$query_string;
        }
        location ~ \.php$ {
            fastcgi_pass unix:/var/run/php-fpm.sock;
            fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
            include fastcgi_params;
        }
        location /up {
            access_log off;
            fastcgi_pass unix:/var/run/php-fpm.sock;
            fastcgi_param SCRIPT_FILENAME $realpath_root/public/index.php;
            include fastcgi_params;
        }
    }
}
```

**docker/entrypoint.sh** (also generate — runs artisan caches with real env at startup):
```bash
#!/bin/sh
set -e
php artisan config:cache
php artisan route:cache
php artisan view:cache
exec "$@"
```

**docker/supervisord.conf:**
```ini
[supervisord]
nodaemon=true
# supervisord itself stays root so it can manage processes;
# each child process drops to appuser via its own user= setting
user=root

[program:php-fpm]
command=php-fpm -F
user=appuser
autostart=true
autorestart=true
stdout_logfile=/dev/stdout
stdout_logfile_maxbytes=0
stderr_logfile=/dev/stderr
stderr_logfile_maxbytes=0

[program:nginx]
# nginx master binds port 80 as root; workers drop to the user
# defined in nginx.conf (user nginx). For fully rootless, use
# nginxinc/nginx-unprivileged on port 8080 instead.
command=nginx -g "daemon off;"
autostart=true
autorestart=true
stdout_logfile=/dev/stdout
stdout_logfile_maxbytes=0
stderr_logfile=/dev/stderr
stderr_logfile_maxbytes=0
```

**docker/opcache.ini:**
```ini
opcache.enable=1
opcache.memory_consumption=256
opcache.max_accelerated_files=20000
opcache.validate_timestamps=0
opcache.save_comments=1
```

**docker/php-fpm.conf:**
```ini
[www]
user = appuser
group = appgroup
listen = /var/run/php-fpm.sock
listen.owner = appuser
listen.group = appgroup
pm = dynamic
pm.max_children = 20
pm.start_servers = 2
pm.min_spare_servers = 1
pm.max_spare_servers = 3
```

---

#### PHP — Symfony — Prod

Same as Laravel. Replace cache step with:
```dockerfile
RUN php bin/console cache:warmup --env=prod --no-debug
```

Replace health check target with `/` or a dedicated `/health` route.

---

#### Python — Prod

**pip / requirements.txt:**
```dockerfile
# syntax=docker/dockerfile:1
FROM python:3.13-slim AS build
WORKDIR /app
COPY requirements.txt .
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install --prefix=/install -r requirements.txt

FROM python:3.13-slim AS runtime
WORKDIR /app
RUN addgroup --system appgroup && adduser --system --ingroup appgroup appuser
COPY --from=build --chown=appuser:appgroup /install /usr/local
COPY --chown=appuser:appgroup . .
USER appuser
EXPOSE 8000
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')" || exit 1
# gunicorn manages workers; uvicorn handles async I/O per worker
# WEB_CONCURRENCY defaults to (2 * cpus + 1) when unset
CMD ["gunicorn", "main:app", \
     "--worker-class", "uvicorn.workers.UvicornWorker", \
     "--bind", "0.0.0.0:8000", \
     "--workers", "${WEB_CONCURRENCY:-4}"]
```

> **Note**: `gunicorn` and `uvicorn[standard]` must be in `requirements.txt`. Bare `uvicorn` with `--workers` is not a real process manager — if a worker dies, it isn't restarted.

**Poetry (`pyproject.toml` with `[tool.poetry]`):**
Replace build stage with:
```dockerfile
FROM python:3.13-slim AS build
WORKDIR /app
RUN pip install poetry==1.8.3
COPY pyproject.toml poetry.lock ./
RUN poetry export -f requirements.txt --output requirements.txt --without-hashes --without dev
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install --prefix=/install -r requirements.txt
```

---

### Phase 5 — .dockerignore

Always generate. Compose from the common base + stack-specific blocks.

**Common (all stacks):**
```
.git/
.gitignore
.github/
*.md
!README.md
.env
.env.*
!.env.example
.DS_Store
Thumbs.db
Dockerfile*
docker-compose*.yml
.dockerignore
.air.toml
```

**Append for Node.js / Angular / Bun / Deno:**
```
node_modules/
dist/
.angular/
.nuxt/
.next/
coverage/
.nyc_output/
*.log
.cache/
.vite/
```

**Append for .NET:**
```
bin/
obj/
.vs/
.vscode/
*.user
TestResults/
```

**Append for Go:**
```
tmp/
bin/
*.test
```

**Append for PHP:**
```
vendor/
node_modules/
storage/logs/
.phpunit.result.cache
*.log
```

**Append for Python:**
```
__pycache__/
*.pyc
*.pyo
.venv/
venv/
env/
.pytest_cache/
*.egg-info/
dist/
build/
.mypy_cache/
htmlcov/
```

---

### Phase 6 — docker-compose Files

Generate three files. This is the override pattern: `docker compose up` = dev, `docker compose -f docker-compose.yml -f docker-compose.prod.yml up` = prod.

**docker-compose.yml** (base — shared between dev and prod):
```yaml
name: <project-name>

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    env_file:
      - .env
    networks:
      - app_network

networks:
  app_network:
    driver: bridge
```

Add database/cache services here if the stack uses them (Laravel, Symfony, Django, Express with DB). Use `condition: service_healthy` on `depends_on`.

**docker-compose.override.yml** (dev — auto-loaded by `docker compose up`):
Populate from the per-stack additions in Phase 4-Dev. Always includes:
- `dockerfile: Dockerfile.dev`
- Source volume mount
- Debug/HMR ports
- Dev environment variables
- Named volumes for package caches

**docker-compose.prod.yml** (production overrides):
```yaml
services:
  app:
    build:
      dockerfile: Dockerfile
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true    # blocks privilege escalation via setuid binaries
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 512M
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
    ports:
      - "80:8080"    # or stack-appropriate port
```

---

### Phase 7 — Makefile (if requested)

Generate only if the user confirmed in Phase 3. Always generate the full file in one pass — never partial.

#### Structure

Every `Makefile` has:
1. **Header** — project name, usage hint
2. **Common targets** — Docker lifecycle commands that work for every stack
3. **Stack-specific targets** — package manager wrappers, framework CLI, test runners
4. **Help target** — self-documenting via `##` comments (always the default target)

#### Common Base (all stacks)

```makefile
# ==============================================================
# <project-name>
# Usage: make <target> [cmd="..."] [name="..."]
# ==============================================================

.DEFAULT_GOAL := help
.PHONY: help build build-prod up up-d up-prod down restart \
        logs logs-app shell ps clean

COMPOSE       := docker compose
COMPOSE_PROD  := docker compose -f docker-compose.yml -f docker-compose.prod.yml
APP_SERVICE   := app

# Load .env if present (exposes vars to make targets)
-include .env
export

# ── Help ──────────────────────────────────────────────────────
help: ## Show available targets
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | \
	  awk 'BEGIN {FS = ":.*?## "}; {printf "  \033[36m%-22s\033[0m %s\n", $$1, $$2}'

# ── Build ─────────────────────────────────────────────────────
build: ## Build dev images
	$(COMPOSE) build

build-prod: ## Build production images
	$(COMPOSE_PROD) build

build-no-cache: ## Build dev images without cache
	$(COMPOSE) build --no-cache

# ── Lifecycle ─────────────────────────────────────────────────
up: ## Start services — foreground, dev (hot reload)
	$(COMPOSE) up

up-d: ## Start services — background, dev
	$(COMPOSE) up -d

up-prod: ## Start services — background, production
	$(COMPOSE_PROD) up -d

down: ## Stop and remove containers
	$(COMPOSE) down

restart: ## Restart app container
	$(COMPOSE) restart $(APP_SERVICE)

# ── Logs ──────────────────────────────────────────────────────
logs: ## Tail all service logs
	$(COMPOSE) logs -f

logs-app: ## Tail app service logs only
	$(COMPOSE) logs -f $(APP_SERVICE)

# ── Shell ─────────────────────────────────────────────────────
shell: ## Open shell in running app container
	$(COMPOSE) exec $(APP_SERVICE) sh

shell-root: ## Open root shell in running app container
	$(COMPOSE) exec -u root $(APP_SERVICE) sh

# ── Status ────────────────────────────────────────────────────
ps: ## Show running containers
	$(COMPOSE) ps

# ── Cleanup ───────────────────────────────────────────────────
clean: ## Remove containers, volumes, and local images
	$(COMPOSE) down -v --rmi local
```

#### Stack-specific targets — append after the common base

Include only the block(s) matching the detected stack.

---

**Node.js:**
```makefile
# ── npm ───────────────────────────────────────────────────────
.PHONY: npm npm-install npm-ci npm-test npm-lint

npm: ## Run npm command  e.g. make npm cmd="install express"
	$(COMPOSE) exec $(APP_SERVICE) npm $(cmd)

npm-install: ## Install all dependencies (including devDeps)
	$(COMPOSE) exec $(APP_SERVICE) npm install

npm-ci: ## Clean install from lockfile
	$(COMPOSE) exec $(APP_SERVICE) npm ci

npm-test: ## Run tests
	$(COMPOSE) exec $(APP_SERVICE) npm test

npm-lint: ## Run linter
	$(COMPOSE) exec $(APP_SERVICE) npm run lint
```

---

**Angular** (append after Node.js block — Angular projects use both):
```makefile
# ── Angular ───────────────────────────────────────────────────
.PHONY: ng ng-test ng-lint ng-build ng-e2e

ng: ## Run ng command  e.g. make ng cmd="generate component foo"
	$(COMPOSE) exec $(APP_SERVICE) npx ng $(cmd)

ng-test: ## Run unit tests (headless)
	$(COMPOSE) exec $(APP_SERVICE) npx ng test --watch=false --browsers=ChromeHeadless

ng-lint: ## Run ESLint via ng lint
	$(COMPOSE) exec $(APP_SERVICE) npx ng lint

ng-build: ## Production build inside container
	$(COMPOSE) exec $(APP_SERVICE) npx ng build --configuration production

ng-e2e: ## Run e2e tests
	$(COMPOSE) exec $(APP_SERVICE) npx ng e2e
```

---

**Bun:**
```makefile
# ── bun ───────────────────────────────────────────────────────
.PHONY: bun bun-install bun-test bun-lint

bun: ## Run bun command  e.g. make bun cmd="add express"
	$(COMPOSE) exec $(APP_SERVICE) bun $(cmd)

bun-install: ## Install dependencies
	$(COMPOSE) exec $(APP_SERVICE) bun install

bun-test: ## Run tests
	$(COMPOSE) exec $(APP_SERVICE) bun test

bun-lint: ## Run linter
	$(COMPOSE) exec $(APP_SERVICE) bun run lint
```

---

**Deno:**
```makefile
# ── deno ──────────────────────────────────────────────────────
.PHONY: deno deno-test deno-lint deno-fmt deno-check

deno: ## Run deno command  e.g. make deno cmd="task test"
	$(COMPOSE) exec $(APP_SERVICE) deno $(cmd)

deno-test: ## Run tests
	$(COMPOSE) exec $(APP_SERVICE) deno test --allow-all

deno-lint: ## Run linter
	$(COMPOSE) exec $(APP_SERVICE) deno lint

deno-fmt: ## Format source files
	$(COMPOSE) exec $(APP_SERVICE) deno fmt

deno-check: ## Type-check without running
	$(COMPOSE) exec $(APP_SERVICE) deno check main.ts
```

---

**.NET Core / .NET 5+:**
```makefile
# ── dotnet ────────────────────────────────────────────────────
.PHONY: dotnet dotnet-restore dotnet-build dotnet-test dotnet-watch \
        ef-add ef-update ef-revert

dotnet: ## Run dotnet command  e.g. make dotnet cmd="add package Newtonsoft.Json"
	$(COMPOSE) exec $(APP_SERVICE) dotnet $(cmd)

dotnet-restore: ## Restore NuGet packages
	$(COMPOSE) exec $(APP_SERVICE) dotnet restore

dotnet-build: ## Build solution
	$(COMPOSE) exec $(APP_SERVICE) dotnet build

dotnet-test: ## Run tests
	$(COMPOSE) exec $(APP_SERVICE) dotnet test

dotnet-watch: ## Run with hot reload (dev)
	$(COMPOSE) exec $(APP_SERVICE) dotnet watch run

ef-add: ## Add EF migration  e.g. make ef-add name="AddUserTable"
	$(COMPOSE) exec $(APP_SERVICE) dotnet ef migrations add $(name)

ef-update: ## Apply pending EF migrations
	$(COMPOSE) exec $(APP_SERVICE) dotnet ef database update

ef-revert: ## Revert last EF migration
	$(COMPOSE) exec $(APP_SERVICE) dotnet ef migrations remove
```

---

**PHP — Laravel:**
```makefile
# ── composer ──────────────────────────────────────────────────
.PHONY: composer composer-install composer-update composer-dump

composer: ## Run composer command  e.g. make composer cmd="require laravel/horizon"
	$(COMPOSE) exec $(APP_SERVICE) composer $(cmd)

composer-install: ## Install dependencies
	$(COMPOSE) exec $(APP_SERVICE) composer install

composer-update: ## Update dependencies
	$(COMPOSE) exec $(APP_SERVICE) composer update

composer-dump: ## Regenerate autoloader
	$(COMPOSE) exec $(APP_SERVICE) composer dump-autoload -o

# ── artisan ───────────────────────────────────────────────────
.PHONY: artisan migrate migrate-fresh migrate-rollback seed tinker \
        queue schedule horizon test

artisan: ## Run artisan command  e.g. make artisan cmd="make:model Post"
	$(COMPOSE) exec $(APP_SERVICE) php artisan $(cmd)

migrate: ## Run pending migrations
	$(COMPOSE) exec $(APP_SERVICE) php artisan migrate

migrate-fresh: ## Drop all tables, re-migrate, and seed
	$(COMPOSE) exec $(APP_SERVICE) php artisan migrate:fresh --seed

migrate-rollback: ## Rollback last migration batch
	$(COMPOSE) exec $(APP_SERVICE) php artisan migrate:rollback

seed: ## Run database seeders
	$(COMPOSE) exec $(APP_SERVICE) php artisan db:seed

tinker: ## Open Laravel Tinker REPL
	$(COMPOSE) exec $(APP_SERVICE) php artisan tinker

queue: ## Start queue worker (dev)
	$(COMPOSE) exec $(APP_SERVICE) php artisan queue:work

schedule: ## Run scheduler (single run)
	$(COMPOSE) exec $(APP_SERVICE) php artisan schedule:run

horizon: ## Start Laravel Horizon
	$(COMPOSE) exec $(APP_SERVICE) php artisan horizon

test: ## Run PHPUnit tests
	$(COMPOSE) exec $(APP_SERVICE) php artisan test
```

---

**PHP — Symfony:**
```makefile
# ── composer ──────────────────────────────────────────────────
.PHONY: composer composer-install composer-update

composer: ## Run composer command  e.g. make composer cmd="require symfony/http-client"
	$(COMPOSE) exec $(APP_SERVICE) composer $(cmd)

composer-install: ## Install dependencies
	$(COMPOSE) exec $(APP_SERVICE) composer install

composer-update: ## Update dependencies
	$(COMPOSE) exec $(APP_SERVICE) composer update

# ── console ───────────────────────────────────────────────────
.PHONY: console db-migrate db-rollback cache-clear test

console: ## Run symfony console command  e.g. make console cmd="debug:router"
	$(COMPOSE) exec $(APP_SERVICE) php bin/console $(cmd)

db-migrate: ## Run Doctrine migrations
	$(COMPOSE) exec $(APP_SERVICE) php bin/console doctrine:migrations:migrate --no-interaction

db-rollback: ## Rollback last Doctrine migration
	$(COMPOSE) exec $(APP_SERVICE) php bin/console doctrine:migrations:execute --down $(version)

cache-clear: ## Clear Symfony cache
	$(COMPOSE) exec $(APP_SERVICE) php bin/console cache:clear

test: ## Run PHPUnit tests
	$(COMPOSE) exec $(APP_SERVICE) php bin/phpunit
```

---

**Go:**
```makefile
# ── go ────────────────────────────────────────────────────────
.PHONY: go go-test go-test-race go-bench go-lint go-tidy go-vet

go: ## Run go command  e.g. make go cmd="get github.com/gin-gonic/gin"
	$(COMPOSE) exec $(APP_SERVICE) go $(cmd)

go-test: ## Run tests
	$(COMPOSE) exec $(APP_SERVICE) go test ./...

go-test-race: ## Run tests with race detector
	$(COMPOSE) exec $(APP_SERVICE) go test -race ./...

go-bench: ## Run benchmarks
	$(COMPOSE) exec $(APP_SERVICE) go test -bench=. ./...

go-vet: ## Run go vet
	$(COMPOSE) exec $(APP_SERVICE) go vet ./...

go-lint: ## Run golangci-lint
	$(COMPOSE) exec $(APP_SERVICE) golangci-lint run

go-tidy: ## Tidy go.mod and go.sum
	$(COMPOSE) exec $(APP_SERVICE) go mod tidy
```

---

**Python — generic / FastAPI:**
```makefile
# ── pip ───────────────────────────────────────────────────────
.PHONY: pip pip-install pip-freeze pytest lint

pip: ## Run pip command  e.g. make pip cmd="install requests"
	$(COMPOSE) exec $(APP_SERVICE) pip $(cmd)

pip-install: ## Install dependencies from requirements.txt
	$(COMPOSE) exec $(APP_SERVICE) pip install -r requirements.txt

pip-freeze: ## Freeze current packages to requirements.txt
	$(COMPOSE) exec $(APP_SERVICE) pip freeze > requirements.txt

pytest: ## Run tests
	$(COMPOSE) exec $(APP_SERVICE) pytest

lint: ## Run ruff / flake8
	$(COMPOSE) exec $(APP_SERVICE) ruff check . 2>/dev/null || \
	  $(COMPOSE) exec $(APP_SERVICE) flake8 .
```

**Python — Django (append after pip block):**
```makefile
# ── manage ────────────────────────────────────────────────────
.PHONY: manage migrate makemigrations shell-django createsuperuser

manage: ## Run manage.py command  e.g. make manage cmd="showmigrations"
	$(COMPOSE) exec $(APP_SERVICE) python manage.py $(cmd)

migrate: ## Apply Django migrations
	$(COMPOSE) exec $(APP_SERVICE) python manage.py migrate

makemigrations: ## Generate Django migrations
	$(COMPOSE) exec $(APP_SERVICE) python manage.py makemigrations

shell-django: ## Open Django shell
	$(COMPOSE) exec $(APP_SERVICE) python manage.py shell

createsuperuser: ## Create Django superuser interactively
	$(COMPOSE) exec -it $(APP_SERVICE) python manage.py createsuperuser
```

---

#### Windows compatibility note

`make` on Windows requires Git Bash, WSL, or Chocolatey/Scoop. Add a comment at the top of the Makefile:

```makefile
# Windows: run via Git Bash, WSL, or `winget install GnuWin32.Make`
```

---

### Phase 8 — Post-generation Checklist

```
Generated:
  [x] Dockerfile                    (prod)
  [x] Dockerfile.dev                (dev)
  [x] .dockerignore
  [x] docker-compose.yml            (base)
  [x] docker-compose.override.yml   (dev — auto-loaded)
  [x] docker-compose.prod.yml       (prod overrides)
  [x] docker/nginx.conf             (if applicable)
  [x] docker/supervisord.conf       (if applicable)
  [x] .air.toml                     (Go only)
  [x] Makefile                      (if requested)

Replace before using:
  [ ] <ProjectName> → actual project/dll name
  [ ] <project-name> → angular.json project name
  [ ] Port numbers → confirm they match app config

Dev workflow (raw):
  docker compose up           ← starts dev (hot reload, debug ports)
  docker compose down

Dev workflow (Makefile):
  make up                     ← start dev
  make logs                   ← tail all logs
  make shell                  ← shell into app
  make <pkg-manager>-install  ← e.g. make npm-install, make composer-install
  make down                   ← stop everything

Prod workflow (raw):
  docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d

Prod workflow (Makefile):
  make build-prod
  make up-prod

Verify:
  [ ] docker build -t <name> .                       (prod build succeeds)
  [ ] docker build -f Dockerfile.dev -t <name>:dev . (dev build succeeds)
  [ ] make up → edit a source file → confirm hot reload fires without rebuild
  [ ] HEALTHCHECK endpoint returns 200
  [ ] docker run --rm <name> id   (uid should not be 0)
  [ ] Secrets come from .env file, not baked into image
  [ ] make help outputs all available targets
  [ ] Scan image: trivy image <name> or grype <name>
  [ ] Node.js apps: confirm SIGTERM handler exits cleanly (docker stop → no SIGKILL timeout)
```
