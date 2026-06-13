# Investigation: Docker Wrapper Working Directory

## Hand-off Brief

1. **What happened.** The documented local and VPS wrappers changed Docker's
   working directory, which prevented the released Serena image entrypoint from
   activating its virtual environment.
2. **Where the case stands.** Complete. The failure was reproduced with both a
   locally built image and the official image, then fixed and verified.
3. **What's needed next.** Keep the image working directory unchanged unless
   the image entrypoint is changed to use an absolute virtual-environment path.

## Case Info

| Field | Value |
| --- | --- |
| Ticket | N/A |
| Date opened | 2026-06-13 |
| Status | Complete |
| Evidence sources | Docker build, official image inspection, wrapper smoke tests, documentation build |

## Problem Statement

The Docker deployment guides needed an end-to-end test on a host without a
local Python installation.

## Confirmed Findings

### The released image requires its default working directory

The official `ghcr.io/oraios/serena:latest` image pulled on 2026-06-13 had
digest `sha256:c9ead14cb70782f9ba475a67a28cfa86e0e3ee4cbc152d2902f859cffbadc800`.
Its entrypoint was:

```text
["/bin/bash","-c","source .venv/bin/activate && $0 $@"]
```

Running it with `--workdir=/workspaces/project` failed before Serena started:

```text
serena: line 1: .venv/bin/activate: No such file or directory
```

The same failure occurred with an image built from this repository's
`Dockerfile`.

### Removing the workdir override fixes the wrappers

Without the workdir override:

- `serena project create --index` indexed two Python files;
- `.serena/cache/python/*.pkl` remained on the host after container removal;
- the documented local wrapper started the stdio MCP server with 22 tools;
- closing stdin shut the server down and `--rm` removed the container;
- the wrapper cleanup trap left no CID file;
- a read-only project mount rejected source writes while the nested `.serena`
  mount remained writable.

### Hardening controls were applied

`docker inspect` confirmed:

```text
CapDrop=["ALL"]
SecurityOpt=["no-new-privileges:true"]
Memory=536870912
NanoCpus=500000000
PidsLimit=128
NetworkMode=none
Mounts=/workspaces/project:false;/workspaces/project/.serena:true;
```

## Conclusion

**Confidence:** High

The failure was caused by the wrapper's Docker working-directory override, not
by Serena project activation or index persistence. Serena already receives the
project path through `--project=/workspaces/project`, so removing `--workdir`
preserves the intended behavior and restores compatibility with the released
image.

## Verification

- Local wrapper extracted directly from the guide: `bash -n` and ShellCheck
  passed.
- VPS wrapper and SSH dispatcher extracted directly from the guide: `bash -n`
  and ShellCheck passed.
- Full Sphinx documentation build in `python:3.13-slim`: passed with warnings
  treated as errors.
