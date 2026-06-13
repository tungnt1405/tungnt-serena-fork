# Serena Docker On-Demand Design

## Goal

Document a secure, project-isolated way to run Serena in disposable Docker
containers on a local machine or a VPS. MCP clients start Serena automatically
over stdio, and the container is removed when the client session ends.

## Scope

Create two Vietnamese usage guides:

- `docs/02-usage/043_docker_local_on_demand_vi.md`
- `docs/02-usage/044_docker_vps_ssh_on_demand_vi.md`

Update `docs/02-usage/041_per_project_index_isolation_vi.md` with links to both
guides.

## Architecture

### Local

An MCP client starts a local wrapper. The wrapper validates a fixed project
path and runs `docker run --rm -i` with that project mounted at
`/workspaces/project`.

### VPS

An MCP client starts `ssh -T`. A forced-command SSH dispatcher accepts only a
project ID and an approved Serena context. A server-side wrapper maps the
project ID to a fixed path and starts `docker run --rm -i`.

The VPS wrapper uses a CID file and signal traps so that an interrupted SSH
session also removes the associated container.

## Persistence And Isolation

- Project configuration, memories, and index remain under
  `<project>/.serena`.
- `SERENA_HOME` is stored below each project's `.serena/docker-home`.
- Downloaded language-server resources persist through the mounted
  `SERENA_HOME`.
- Each container mounts one project only.
- Different projects do not share `SERENA_HOME`.
- Data written only to the container writable layer is intentionally
  disposable.

## Images

The same wrappers support either:

- a pinned upstream Serena image; or
- a pinned image built and published from a fork.

The selected image is controlled by trusted host configuration. VPS clients
cannot supply an image name or Docker flags.

## Security Boundaries

- No MCP or dashboard port is published.
- Dashboard is disabled.
- Containers are not privileged and do not mount the Docker socket.
- Capabilities are dropped and `no-new-privileges` is enabled.
- CPU, memory, and PID limits are configurable.
- The VPS SSH key uses a forced command and disables forwarding, PTY, and
  agent/X11 forwarding.
- The dispatcher parses a strict command grammar without `eval`.
- The VPS project map is an explicit allowlist.

## Supported MCP Clients

Provide local and VPS examples for:

- Claude Code CLI
- Claude Desktop
- Codex CLI and Codex App
- Antigravity IDE and CLI, with version caveats where official CLI syntax is
  not stable or documented
- VS Code with GitHub Copilot
- JetBrains AI Assistant
- JetBrains Junie
- JetBrains GitHub Copilot

## Hardware Guidance

Both guides begin with operational sizing guidance. The VPS guide accounts for
other always-on services and recommends sizing from remaining memory rather
than total memory alone.

## Index Impact

`--rm` removes the container writable layer, not bind-mounted data. A graceful
stdio shutdown saves language-server caches. OOM, power loss, or forced kills
may lose only cache changes that were not flushed; source and previously
persisted cache remain.

## Verification

- Validate shell samples with `bash -n` and ShellCheck when available.
- Parse JSON and TOML samples.
- Verify documentation links and headings.
- Build the Sphinx documentation with warnings treated as errors.
- Review the final diff for unsafe Docker or SSH defaults.

## Out Of Scope

- Public HTTP/SSE MCP deployment
- Reverse proxies, OAuth, and Internet-facing authentication
- Kubernetes or container orchestration
- Automatic cloning or updating of project repositories
- Building a new hardened Serena image
