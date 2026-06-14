# Serena Docker On-Demand Documentation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use `subagent-driven-development` (recommended) or `executing-plans` to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add complete Vietnamese guides for disposable local and VPS Serena Docker deployments with persistent indexes and client-specific MCP configuration.

**Architecture:** MCP clients launch a trusted stdio wrapper. Local wrappers call Docker directly; VPS clients call a forced-command SSH dispatcher which invokes an allowlisted Docker wrapper. Bind mounts preserve project state and language-server resources while `--rm` removes only the disposable container layer.

**Tech Stack:** Markdown, Bash, Docker, OpenSSH, MCP stdio, JSON, TOML, Sphinx/Jupyter Book.

---

### Task 1: Document Local Docker On-Demand Deployment

**Files:**
- Create: `docs/02-usage/043_docker_local_on_demand_vi.md`

- [x] Add hardware guidance before installation instructions.
- [x] Explain the stdio and `docker run --rm -i` lifecycle.
- [x] Provide a complete local wrapper with validation, persistent mounts,
      security controls, and resource limits.
- [x] Explain read-write and read-only project mounts.
- [x] Document exactly which index and cache data survives `--rm`.
- [x] Add local MCP examples for every supported client.
- [x] Add verification and troubleshooting commands.

### Task 2: Document VPS Docker Over SSH

**Files:**
- Create: `docs/02-usage/044_docker_vps_ssh_on_demand_vi.md`

- [x] Add VPS hardware guidance that accounts for existing services.
- [x] Document project and state directory layout.
- [x] Provide a strict project allowlist.
- [x] Provide a server wrapper using `--cidfile` and cleanup traps.
- [x] Provide a forced-command SSH dispatcher without `eval`.
- [x] Document a restricted `authorized_keys` entry.
- [x] Add VPS MCP examples for every supported client.
- [x] Add concurrent-project, verification, operations, and troubleshooting
      sections.

### Task 3: Link The Guides From The Isolation Guide

**Files:**
- Modify: `docs/02-usage/041_per_project_index_isolation_vi.md`

- [x] Add both Docker guides to the table of contents.
- [x] Add a short Docker deployment section that routes readers to the local
      or VPS guide without duplicating their content.
- [x] Preserve the existing per-project isolation recommendations.

### Task 4: Validate Documentation

**Files:**
- Verify: `docs/02-usage/043_docker_local_on_demand_vi.md`
- Verify: `docs/02-usage/044_docker_vps_ssh_on_demand_vi.md`
- Verify: `docs/02-usage/041_per_project_index_isolation_vi.md`

- [x] Extract and validate Bash wrappers with `bash -n`.
- [x] Run ShellCheck when installed.
- [x] Parse representative JSON and TOML configuration samples.
- [x] Search for placeholders and unsafe public-listener examples.
- [x] Run `uv run poe doc-build`.
- [x] Review the final diff against the approved design.

### Task 5: Commit The Documentation

**Files:**
- Stage all files listed above plus the design, plan, and status files.

- [x] Confirm `git status --short` contains only intended changes.
- [x] Commit with message `docs: add on-demand Docker deployment guides`.
