# Serena Workspace-Local Quick Deployment Design

## Goal

Document a disposable, project-local Serena Docker setup for tester, customer,
and occasional-use machines without installing Python or a machine-wide
wrapper.

## Users

- Testers who need Serena for one repository.
- Customer machines where permanent developer tooling is undesirable.
- Personal machines that only occasionally need an MCP coding agent.
- Developers who need a quick smoke test before adopting the machine-wide
  setup.

## Architecture

Each project contains one ignored wrapper at
`.serena-local/serena-docker`. Every supported MCP client for that project
calls the same wrapper with its own Serena context and an optional `ro` or `rw`
mount mode.

The wrapper derives the project root from its own location. It starts one
`docker run --rm -i` process per MCP connection, binds the project and its
Serena state, applies resource and privilege limits, and cleans up the
container through a CID file and signal trap.

The bootstrap records project-local artifacts in `.git/info/exclude` instead
of modifying the repository's shared `.gitignore`. Existing MCP configuration
files are never overwritten; users merge the documented server entry.

## Local Artifacts

```text
<project>/.serena-local/serena-docker
<project>/.serena-local/env
<project>/.serena/.gitignore
<project>/.serena/project.yml
<project>/.serena/cache/
<project>/.serena/docker-home/
```

Agent-specific configuration may also exist under `.vscode`, `.codex`,
`.claude`, `.junie`, or another client-specific path. Only the Serena-specific
files are locally excluded.

## Security Defaults

- Stdio only; no MCP or dashboard listener is published.
- `--rm`, `--init`, `--cap-drop=ALL`, and
  `--security-opt=no-new-privileges:true`.
- Memory, CPU, and PID limits.
- Default project mount is read-only.
- Image reference is resolved to a registry digest during bootstrap.
- Languages are declared explicitly when a new project is created, avoiding an
  interactive prompt inside the container.
- `.serena/docker-home` is ignored by Serena indexing.
- No Docker socket mount, privileged mode, host network, or arbitrary command
  evaluation.
- Project-local scripts are treated as trusted local tooling and must not be
  copied blindly into an untrusted repository.

## Git Behavior

For Git repositories, bootstrap appends exact paths to `.git/info/exclude`.
This keeps the setup local to the current clone and avoids a shared
`.gitignore` diff.

Local excludes do not hide already tracked files. The guide must require
`git ls-files` and `git status --short` checks before and after setup.

## Supported Clients

- Claude Code CLI
- Claude Desktop
- Codex CLI
- Codex App
- Antigravity IDE
- Antigravity CLI
- VS Code with GitHub Copilot
- JetBrains AI Assistant
- JetBrains Junie
- JetBrains GitHub Copilot

## Error Handling

- Fail when Docker is unavailable.
- Fail when `.serena/project.yml` is missing at MCP startup.
- Fail instead of modifying a tracked `.serena/.gitignore` that lacks required
  local-state rules.
- Reject unknown contexts and mount modes.
- Preserve the MCP process exit status during cleanup.
- Never replace an existing agent configuration automatically.

## Verification

- Extract wrapper from the guide and run `bash -n`.
- Run ShellCheck in Docker.
- Parse JSON and TOML examples.
- Bootstrap and index a temporary repository using Docker only.
- Start the wrapper in `ro` and `rw` modes.
- Confirm `--rm`, CID cleanup, persistent cache, and negative input handling.
- Build the complete documentation with warnings treated as errors.

## Out Of Scope

- Public HTTP/SSE MCP deployment.
- VPS or SSH deployment.
- Installing a shared wrapper under `~/.local/bin` or `/usr/local/bin`.
- Automatically editing or replacing existing MCP configuration files.
- Supporting Windows containers or PowerShell wrappers.
