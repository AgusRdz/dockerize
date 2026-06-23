---
description: >-
  Generate Docker dev/prod configuration or audit an existing Docker setup.
  Modes: generate dev environment (hot reload, source mounts, debug ports),
  generate production config (multi-stage, non-root, pinned, health checks),
  audit existing Dockerfile/docker-compose for security and best-practice
  issues. Optionally generates a Makefile with Docker shortcuts and package
  manager wrappers (npm, composer, artisan, dotnet, go, pip, etc.).
argument-hint: "[audit | dev | prod | both | <stack>]"
allowed-tools: [Read, Write, Edit, Glob, Grep, Bash]
---

# /dockerize

Run the `dockerize` skill for the current project.

Arguments passed by the user: $ARGUMENTS

- No arguments → skill auto-detects stack and asks which mode (audit / dev / prod / both)
- `audit` → skip to audit mode regardless of stack
- `dev` / `prod` / `both` → skip mode question, generate the requested config
- Stack name (`node`, `dotnet`, `laravel`, `go`, `python`, `bun`, `deno`, `angular`, `symfony`, `php`) → skip stack detection, use named stack
