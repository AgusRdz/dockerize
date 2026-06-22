---
description: Dockerize the current project with production-grade best practices
argument-hint: [stack-override]
allowed-tools: [Read, Write, Edit, Glob, Grep, Bash]
---

# /dockerize

Containerize the current project using the `dockerize` skill.

Invoke the skill and pass any arguments the user provided:

$ARGUMENTS

The skill will auto-detect the stack. If `$ARGUMENTS` names a specific stack (e.g. `node`, `dotnet`, `laravel`), use that as the detected stack and skip the fingerprinting phase.
