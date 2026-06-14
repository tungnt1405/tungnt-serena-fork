# Serena Workspace-Local Quick Deployment Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use `subagent-driven-development` (recommended) or `executing-plans` to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a complete Vietnamese guide for project-local, disposable Serena Docker deployment across supported MCP clients.

**Architecture:** A single ignored wrapper lives inside each project and derives its project root from its own path. A bootstrap block pins the image, creates or indexes Serena state, and adds clone-local Git excludes without replacing client configuration.

**Tech Stack:** Markdown, Bash, Docker, MCP stdio, JSON, TOML, Sphinx/Jupyter Book.

---

### Task 1: Write The Workspace-Local Guide

**Files:**
- Create: `docs/02-usage/043_docker_workspace_local_quick_deployment_vi.md`

- [x] Add use cases, hardware guidance, and comparison with the shared wrapper.
- [x] Add a Docker-only bootstrap that creates the wrapper and local excludes.
- [x] Add the complete self-locating wrapper with secure defaults.
- [x] Explain initialization, indexing, persistence, read-only and read-write modes.
- [x] Add merge-safe MCP examples for every supported client.
- [x] Add validation, troubleshooting, upgrade, and complete removal procedures.

### Task 2: Link The Guide

**Files:**
- Modify: `docs/02-usage/041_per_project_index_isolation_vi.md`
- Modify: `docs/02-usage/043_docker_local_on_demand_vi.md`

- [x] Link the quick guide from the isolation routing section.
- [x] Link the quick guide from the machine-wide local Docker guide.
- [x] Keep the distinction between project-local and machine-wide wrappers explicit.

### Task 3: Validate With Docker

**Files:**
- Verify: `docs/02-usage/043_docker_workspace_local_quick_deployment_vi.md`

- [x] Extract Bash blocks and run `bash -n`.
- [x] Run ShellCheck in Docker.
- [x] Parse JSON and TOML examples.
- [x] Bootstrap and index a temporary Git project using Docker.
- [x] Start the documented wrapper and confirm persistence and cleanup.
- [x] Run the full documentation build in Docker.
- [x] Run `git diff --check` and review the final diff.

### Task 4: Commit

**Files:**
- Stage the guide, links, design, plan, and status.

- [x] Confirm only intended tracked changes are present.
- [x] Commit with message `docs: add workspace-local Docker quick setup`.
