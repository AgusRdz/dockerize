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

**Step 1** — Locate and load the skill instructions:

1. Use the Glob tool with pattern `**/dockerize/skills/dockerize/SKILL.md` to find the skill file.
2. Use the Read tool to read the full content of that file.
3. Follow every instruction in the file exactly, starting from Phase 0. Do not summarize, skip, or improvise any phase.

User arguments: $ARGUMENTS

Argument routing — apply BEFORE Phase 0:
- No arguments → run Phase 0 (auto-detect mode)
- `audit` → jump directly to AUDIT MODE, skip Phase 0 question
- `dev` → jump to GENERATE MODE, skip mode question, generate dev only
- `prod` → jump to GENERATE MODE, skip mode question, generate prod only
- `both` → jump to GENERATE MODE, skip mode question, generate both dev and prod
- Stack name (`node`, `dotnet`, `laravel`, `go`, `python`, `bun`, `deno`, `angular`, `symfony`, `php`) → jump to GENERATE MODE, skip stack detection, use the named stack
- `--makefile` flag (can combine with any of the above) → pre-answer the Makefile question as "yes", skip the prompt for it
- `--no-makefile` flag → pre-answer the Makefile question as "no", skip the prompt for it

Examples:
- `/dockerize` — auto-detect, ask everything
- `/dockerize angular --makefile` — Angular, generate Makefile without asking
- `/dockerize audit` — audit existing setup
- `/dockerize prod` — prod only, still ask about Makefile
