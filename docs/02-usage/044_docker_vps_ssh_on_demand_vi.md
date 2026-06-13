# Triển khai Serena bằng Docker trên VPS qua SSH theo nhu cầu

Tài liệu này hướng dẫn dùng một VPS chứa nhiều project và nhiều dịch vụ Docker,
trong khi Serena chỉ chạy khi MCP client cần:

- không chạy container Serena 24/7;
- không mở MCP, SSE, HTTP hoặc dashboard port ra Internet;
- MCP giao tiếp qua `stdio`;
- SSH chỉ chuyển luồng stdio đến wrapper đã allowlist;
- wrapper tạo `docker run --rm -i`;
- khi phiên kết thúc, container tự xóa;
- project, memories, index và language-server resources vẫn tồn tại trên VPS.

Mỗi container chỉ được mount một project. Một hoặc hai project có thể chạy song
song mà không dùng chung active project hay cache.

## Cấu hình phần cứng khuyến nghị

Với VPS đang chạy thêm dịch vụ 24/7, phải tính theo RAM còn lại sau peak load
của các dịch vụ đó:

| Tình huống | RAM VPS khuyến nghị | Cách vận hành |
| --- | ---: | --- |
| Repo nhỏ, dịch vụ nền nhẹ | 2 GB | Một Serena container, theo dõi OOM |
| VPS cá nhân chạy nhiều dịch vụ | Từ 4 GB | Một Serena container ổn định hơn |
| Thỉnh thoảng chạy hai project | Cân nhắc từ 8 GB | Hai container với limit riêng |
| Repo lớn hoặc language server nặng | 8-16 GB hoặc hơn | Đo peak sau initial indexing |

Quy tắc lập ngân sách:

```text
RAM có thể cấp cho Serena =
RAM tổng VPS
- peak RAM của dịch vụ 24/7
- 20-30% dự phòng cho kernel, Docker và SSH
```

Ví dụ VPS 4 GB có các dịch vụ khác dùng peak 1.5 GB:

```text
4 GB - 1.5 GB - 1 GB dự phòng = khoảng 1.5 GB cho Serena
```

Khi đó bắt đầu với `SERENA_MEMORY_LIMIT=1400m`, theo dõi rồi điều chỉnh.

Swap giúp giảm nguy cơ OOM đột ngột nhưng không thay thế RAM. Index trên swap
nhiều sẽ chậm rõ rệt. Disk nên là SSD/NVMe và cần chừa ít nhất 20 GB cho image,
language servers, dependency và `.serena/cache`; repo lớn cần nhiều hơn.

Kiểm tra trước khi triển khai:

```bash
free -h
swapon --show
df -h
docker stats --no-stream
docker system df
```

## Điều kiện bắt buộc: agent và Serena phải nhìn cùng working tree

Serena trên VPS thao tác file tại VPS. Nhiều coding agent lại có file, edit và
shell tool riêng chạy trên máy của client.

Không cấu hình theo mô hình này:

```text
Agent built-in tools -> local clone project-a
Serena MCP          -> VPS clone project-a
```

Hai clone có thể cùng commit ban đầu nhưng sẽ khác ngay sau lần edit đầu tiên.
Agent có thể đọc file local rồi yêu cầu Serena sửa file remote, tạo kết quả sai
hoặc khó kiểm soát.

Chọn một trong hai mô hình an toàn:

### Mô hình A: remote workspace, khuyến nghị cho coding agent

- Claude Code/Codex CLI chạy trong SSH shell trên VPS; hoặc
- VS Code dùng Remote SSH; hoặc
- JetBrains dùng Remote Development; hoặc
- client có remote workspace/executor chính thức.

Agent built-in tools và Serena đều nhìn cùng `/srv/serena/projects/<id>`.
Trong mô hình này MCP command gọi `/usr/local/bin/run-serena` trực tiếp trên
VPS; không cần SSH lồng thêm.

### Mô hình B: MCP stdio trực tiếp qua SSH

Client local gọi:

```bash
ssh -T serena@vps.example.com run-serena project-a desktop-app
```

Mô hình này phù hợp với:

- Claude Desktop hoặc client không có coding filesystem riêng;
- agent được cấu hình chỉ dùng Serena cho file remote;
- quy trình chỉ phân tích remote project và không chỉnh một local clone khác.

Nếu coding agent vẫn dùng built-in file/shell tools trên local clone, không dùng
mô hình B.

## Kiến trúc VPS

```text
MCP client
    |
    | stdio, không public TCP port
    v
ssh -T serena@vps
    |
    v
/usr/local/bin/serena-ssh-dispatch
    |
    | project ID + context đã kiểm tra
    v
/usr/local/bin/run-serena
    |
    v
docker run --rm -i
    |
    +-- một project allowlist
    +-- một persistent SERENA_HOME
    `-- resource/security limits
```

Trong remote workspace, bỏ lớp SSH đầu tiên:

```text
Remote MCP client -> /usr/local/bin/run-serena -> Docker
```

## Chuẩn bị VPS

Yêu cầu:

- Linux VPS có Docker Engine;
- OpenSSH server;
- project đã được clone dưới một root do admin quản lý;
- image Serena đã được ghim tag/digest;
- user SSH riêng cho Serena.

Tạo user trước:

```bash
sudo useradd --create-home --shell /bin/bash serena
sudo passwd -l serena
sudo usermod -aG docker serena
```

Sau đó tạo cấu trúc với parent directory do `root` quản lý:

```bash
sudo install -d -o root -g serena -m 0750 /srv/serena
sudo install -d -o root -g serena -m 0750 /srv/serena/projects
sudo install -d -o root -g serena -m 0770 /srv/serena/homes
sudo install -d -o root -g serena -m 0770 /srv/serena/run
```

Ví dụ:

```text
/srv/serena/
├── projects/
│   ├── project-a/
│   │   └── .serena/
│   └── project-b/
│       └── .serena/
├── homes/
│   ├── project-a/
│   └── project-b/
└── run/
```

Quyền thành viên nhóm `docker` gần tương đương quyền root trên host. Không cấp
SSH shell tự do cho key MCP. Phần forced command bên dưới là ranh giới bắt buộc,
không phải hardening tùy chọn.

Chuẩn bị quyền:

```bash
sudo chown -R serena:serena /srv/serena/projects/project-a
sudo chown -R serena:serena /srv/serena/projects/project-b
sudo install -d -o serena -g serena -m 0750 /srv/serena/homes/project-a
sudo install -d -o serena -g serena -m 0750 /srv/serena/homes/project-b
```

## Cấu hình image và resource limits

Tạo `/etc/serena-docker.env`, owner `root:root`, mode `0644` nếu không chứa
secret:

```bash
SERENA_IMAGE=ghcr.io/oraios/serena:PINNED_VERSION
SERENA_MEMORY_LIMIT=1400m
SERENA_CPU_LIMIT=1.5
SERENA_PIDS_LIMIT=512
SERENA_NETWORK_MODE=bridge
```

Hoặc dùng image fork đã publish:

```bash
SERENA_IMAGE=registry.example.com/serena-fork:PINNED_VERSION
```

Image là cấu hình do admin kiểm soát. Client không được truyền image name.

`bridge` cho phép Serena/language server tải dependency. Nó không phải network
sandbox tuyệt đối: container vẫn có outbound access và có thể tiếp cận host/LAN
tùy firewall. Sau khi đã chuẩn bị đủ dependency và xác nhận project không cần
outbound network, có thể thử:

```bash
SERENA_NETWORK_MODE=none
```

Không đặt `none` mặc định cho mọi project vì một số language server, package
manager hoặc build tool cần mạng.

## Project allowlist

Tạo `/etc/serena-projects.conf`:

```text
# project-id|absolute-path|rw-or-ro
project-a|/srv/serena/projects/project-a|rw
project-b|/srv/serena/projects/project-b|rw
docs-readonly|/srv/serena/projects/docs|ro
```

Yêu cầu:

- file do `root` sở hữu;
- project ID chỉ gồm chữ thường, số, `.`, `_`, `-`;
- path là absolute path;
- mode chỉ là `rw` hoặc `ro`;
- không có hai dòng trùng project ID.

Thiết lập quyền:

```bash
sudo chown root:root /etc/serena-projects.conf /etc/serena-docker.env
sudo chmod 0644 /etc/serena-projects.conf /etc/serena-docker.env
```

Client chỉ truyền `project-a`; wrapper mới quyết định host path và mount mode.

## Wrapper chạy Serena

Tạo `/usr/local/bin/run-serena`:

```bash
#!/usr/bin/env bash
set -euo pipefail

usage() {
    echo "Usage: run-serena PROJECT_ID CONTEXT" >&2
    exit 64
}

[[ $# -eq 2 ]] || usage

project_id=$1
context=$2

[[ "$project_id" =~ ^[a-z0-9][a-z0-9._-]*$ ]] || {
    echo "Invalid project ID" >&2
    exit 65
}

case "$context" in
    claude-code|desktop-app|codex|antigravity|vscode|jb-ai-assistant|junie|jb-copilot-plugin)
        ;;
    *)
        echo "Unsupported Serena context: $context" >&2
        exit 65
        ;;
esac

config_file=/etc/serena-projects.conf
env_file=/etc/serena-docker.env

[[ -r "$config_file" && -r "$env_file" ]] || {
    echo "Serena server configuration is not readable" >&2
    exit 78
}

project_dir=
mount_mode=

while IFS='|' read -r configured_id configured_path configured_mode; do
    [[ -n "$configured_id" ]] || continue
    [[ "$configured_id" == \#* ]] && continue

    if [[ "$configured_id" == "$project_id" ]]; then
        project_dir=$configured_path
        mount_mode=$configured_mode
        break
    fi
done < "$config_file"

[[ -n "$project_dir" ]] || {
    echo "Project is not allowlisted: $project_id" >&2
    exit 77
}

[[ -d "$project_dir" ]] || {
    echo "Project directory does not exist" >&2
    exit 66
}

resolved_project_dir=$(realpath -e "$project_dir")

[[ "$resolved_project_dir" == /srv/serena/projects/* ]] || {
    echo "Allowlisted path is outside /srv/serena/projects" >&2
    exit 78
}

project_dir=$resolved_project_dir

case "$mount_mode" in
    rw|ro)
        ;;
    *)
        echo "Invalid mount mode in allowlist" >&2
        exit 78
        ;;
esac

[[ -f "$project_dir/.serena/project.yml" ]] || {
    echo "Missing .serena/project.yml for $project_id" >&2
    exit 66
}

# This file is root-controlled and contains only KEY=VALUE assignments.
# shellcheck disable=SC1090
source "$env_file"

: "${SERENA_IMAGE:?SERENA_IMAGE is missing}"
: "${SERENA_MEMORY_LIMIT:=1400m}"
: "${SERENA_CPU_LIMIT:=1.5}"
: "${SERENA_PIDS_LIMIT:=512}"
: "${SERENA_NETWORK_MODE:=bridge}"

home_dir="/srv/serena/homes/$project_id"
mkdir -p "$home_dir"

cidfile="/srv/serena/run/${project_id}.$$.cid"

cleanup() {
    status=$?
    trap - EXIT HUP INT TERM

    if [[ -s "$cidfile" ]]; then
        container_id=$(<"$cidfile")
        docker rm -f "$container_id" >/dev/null 2>&1 || true
    fi

    rm -f "$cidfile"
    exit "$status"
}

trap cleanup EXIT HUP INT TERM

docker_args=(
    run
    --rm
    -i
    --init
    --cidfile="$cidfile"
    --label=serena.on-demand=true
    --label="serena.project=$project_id"
    --cap-drop=ALL
    --security-opt=no-new-privileges:true
    --memory="$SERENA_MEMORY_LIMIT"
    --cpus="$SERENA_CPU_LIMIT"
    --pids-limit="$SERENA_PIDS_LIMIT"
    --network="$SERENA_NETWORK_MODE"
    --env=SERENA_HOME=/workspaces/serena-home
    --env=SERENA_USAGE_REPORTING=false
    --volume="$home_dir:/workspaces/serena-home:rw"
    --workdir=/workspaces/project
)

if [[ "$mount_mode" == "rw" ]]; then
    docker_args+=(--volume="$project_dir:/workspaces/project:rw")
else
    docker_args+=(
        --volume="$project_dir:/workspaces/project:ro"
        --volume="$project_dir/.serena:/workspaces/project/.serena:rw"
    )
fi

docker "${docker_args[@]}" "$SERENA_IMAGE" \
    serena start-mcp-server \
    --transport=stdio \
    --context="$context" \
    --project=/workspaces/project \
    --enable-web-dashboard=false \
    --open-web-dashboard=false
```

Cài đặt:

```bash
sudo chown root:root /usr/local/bin/run-serena
sudo chmod 0755 /usr/local/bin/run-serena
```

### Vì sao dùng CID file và trap?

`--rm` xử lý tốt khi Serena exit bình thường. CID file và trap bổ sung cleanup
khi:

- SSH bị ngắt;
- client crash;
- shell nhận `HUP`, `INT` hoặc `TERM`;
- Docker command trả lỗi sau khi container đã được tạo.

Nếu cả VPS hoặc Docker daemon crash, kiểm tra container còn sót sau khi host
khởi động:

```bash
docker ps -a --filter label=serena.on-demand=true
```

## SSH forced-command dispatcher

Tạo `/usr/local/bin/serena-ssh-dispatch`:

```bash
#!/usr/bin/env bash
set -euo pipefail

original_command=${SSH_ORIGINAL_COMMAND:-}

if [[ "$original_command" =~ ^run-serena[[:space:]]+([a-z0-9][a-z0-9._-]*)[[:space:]]+(claude-code|desktop-app|codex|antigravity|vscode|jb-ai-assistant|junie|jb-copilot-plugin)$ ]]; then
    project_id=${BASH_REMATCH[1]}
    context=${BASH_REMATCH[2]}
    exec /usr/local/bin/run-serena "$project_id" "$context"
fi

echo "Rejected SSH command" >&2
exit 77
```

Cài đặt:

```bash
sudo chown root:root /usr/local/bin/serena-ssh-dispatch
sudo chmod 0755 /usr/local/bin/serena-ssh-dispatch
```

Dispatcher:

- không dùng `eval`;
- không chạy shell fragment do client cung cấp;
- không nhận path, image hoặc Docker flag;
- chỉ chấp nhận đúng grammar `run-serena PROJECT_ID CONTEXT`.

## Cấu hình SSH key

Tạo key riêng trên máy client:

```bash
ssh-keygen -t ed25519 -f "$HOME/.ssh/serena_vps" -C "serena-mcp"
```

Trên VPS, thêm public key vào `/home/serena/.ssh/authorized_keys` trên một dòng:

```text
restrict,command="/usr/local/bin/serena-ssh-dispatch" ssh-ed25519 PUBLIC_KEY_MATERIAL serena-mcp
```

`restrict` trên OpenSSH hiện đại tắt forwarding, agent forwarding, X11 và PTY.
Nếu OpenSSH cũ không hỗ trợ `restrict`, dùng:

```text
command="/usr/local/bin/serena-ssh-dispatch",no-port-forwarding,no-agent-forwarding,no-X11-forwarding,no-pty ssh-ed25519 PUBLIC_KEY_MATERIAL serena-mcp
```

Đặt quyền:

```bash
sudo install -d -o serena -g serena -m 0700 /home/serena/.ssh
sudo chown serena:serena /home/serena/.ssh/authorized_keys
sudo chmod 0600 /home/serena/.ssh/authorized_keys
```

Tạo alias client trong `~/.ssh/config`:

```text
Host serena-vps
    HostName vps.example.com
    User serena
    IdentityFile ~/.ssh/serena_vps
    IdentitiesOnly yes
    RequestTTY no
    ServerAliveInterval 30
    ServerAliveCountMax 3
```

Test command hợp lệ:

```bash
ssh -T serena-vps run-serena project-a desktop-app
```

Command sẽ mở MCP stdio và chờ client. Dùng `Ctrl+C` để kết thúc test.

Test command bị từ chối:

```bash
ssh -T serena-vps "docker ps"
ssh -T serena-vps "run-serena /etc codex"
ssh -T serena-vps "run-serena unknown-project codex"
```

## `--rm` và dữ liệu index trên VPS

| Dữ liệu | Persistent path trên VPS | Sau khi container xóa |
| --- | --- | --- |
| Source | `/srv/serena/projects/<id>/` | Giữ nguyên |
| Project config | `/srv/serena/projects/<id>/.serena/project.yml` | Giữ nguyên |
| Memories | `/srv/serena/projects/<id>/.serena/memories/` | Giữ nguyên |
| Symbol cache/index | `/srv/serena/projects/<id>/.serena/cache/` | Giữ nguyên |
| Global config/log | `/srv/serena/homes/<id>/` | Giữ nguyên |
| Language server resources | `/srv/serena/homes/<id>/language_servers/` | Giữ nguyên |
| Container writable layer | Không mount | Bị xóa |

Shutdown bình thường sẽ flush cache. OOM, mất điện hoặc kill cưỡng chế có thể
làm mất phần cache chưa flush, nhưng không xóa source hay cache cũ.

## Cấu hình MCP: remote workspace

Đây là cách khuyến nghị cho coding agent. MCP client đang chạy trên VPS hoặc
trong remote workspace và gọi wrapper trực tiếp.

Command chung:

```bash
/usr/local/bin/run-serena project-a CONTEXT
```

Không dùng project path từ client; wrapper lấy path từ allowlist.

### Claude Code CLI trên VPS

```bash
claude mcp add --scope project serena -- \
    /usr/local/bin/run-serena project-a claude-code
```

Kiểm tra:

```bash
claude mcp list
```

### Codex CLI trên VPS

```bash
codex mcp add serena -- \
    /usr/local/bin/run-serena project-a codex
```

Hoặc `.codex/config.toml`:

```toml
[mcp_servers.serena]
command = "/usr/local/bin/run-serena"
args = ["project-a", "codex"]
startup_timeout_sec = 120
tool_timeout_sec = 600
```

### VS Code Remote SSH

Trong `.vscode/mcp.json` của remote workspace:

```json
{
  "servers": {
    "serena": {
      "type": "stdio",
      "command": "/usr/local/bin/run-serena",
      "args": [
        "project-a",
        "vscode"
      ]
    }
  }
}
```

Extension host và MCP server phải cùng chạy phía remote.

### JetBrains Remote Development

Dùng path wrapper trên remote host. Ví dụ JetBrains AI Assistant:

```json
{
  "mcpServers": {
    "serena": {
      "command": "/usr/local/bin/run-serena",
      "args": [
        "project-a",
        "jb-ai-assistant"
      ]
    }
  }
}
```

Đảm bảo MCP process được backend remote khởi chạy, không phải UI client local.

## Cấu hình MCP: client local gọi SSH stdio

Các ví dụ dưới đây dùng:

```text
SSH alias: serena-vps
Project ID: project-a
```

### Claude Code CLI

Chỉ dùng khi Claude Code không chỉnh một local clone khác:

```bash
claude mcp add --scope project serena-remote -- \
    ssh -T serena-vps run-serena project-a claude-code
```

`.mcp.json` tương đương:

```json
{
  "mcpServers": {
    "serena-remote": {
      "command": "ssh",
      "args": [
        "-T",
        "serena-vps",
        "run-serena",
        "project-a",
        "claude-code"
      ],
      "timeout": 600000
    }
  }
}
```

### Claude Desktop

Claude Desktop phù hợp với mô hình này vì context `desktop-app` cung cấp bộ
tool Serena đầy đủ:

```json
{
  "mcpServers": {
    "serena-project-a": {
      "command": "ssh",
      "args": [
        "-T",
        "serena-vps",
        "run-serena",
        "project-a",
        "desktop-app"
      ]
    }
  }
}
```

Thoát hoàn toàn app để dừng MCP subprocess và container.

### Codex CLI

Chỉ dùng khi Codex không chỉnh local clone:

```bash
codex mcp add serena-remote -- \
    ssh -T serena-vps run-serena project-a codex
```

TOML:

```toml
[mcp_servers.serena-remote]
command = "ssh"
args = ["-T", "serena-vps", "run-serena", "project-a", "codex"]
startup_timeout_sec = 120
tool_timeout_sec = 600
```

### Codex App

Codex App dùng cùng `config.toml` với Codex CLI. Dùng entry TOML phía trên.
Không mở local project khác rồi để built-in tools và remote Serena cùng chỉnh
hai working tree khác nhau.

### Antigravity IDE và Antigravity CLI

```json
{
  "mcpServers": {
    "serena-project-a": {
      "command": "ssh",
      "args": [
        "-T",
        "serena-vps",
        "run-serena",
        "project-a",
        "antigravity"
      ]
    }
  }
}
```

Dùng config MCP stdio của phiên bản đang cài. Nếu Antigravity CLI có config
riêng, thêm cùng `command` và `args`; không tự chuyển sang HTTP.

Ưu tiên remote workspace nếu Antigravity có built-in file tools.

### VS Code với GitHub Copilot

Khuyến nghị dùng VS Code Remote SSH như phần trước. Nếu vẫn dùng client local
chỉ để truy vấn remote project:

```json
{
  "servers": {
    "serena-remote": {
      "type": "stdio",
      "command": "ssh",
      "args": [
        "-T",
        "serena-vps",
        "run-serena",
        "project-a",
        "vscode"
      ]
    }
  }
}
```

Không dùng entry này để sửa song song một local clone.

### JetBrains AI Assistant

```json
{
  "mcpServers": {
    "serena-remote": {
      "command": "ssh",
      "args": [
        "-T",
        "serena-vps",
        "run-serena",
        "project-a",
        "jb-ai-assistant"
      ]
    }
  }
}
```

Ưu tiên JetBrains Remote Development để IDE index, built-in tools và Serena
cùng nhìn project trên VPS.

### JetBrains Junie

Thêm vào đúng một trong hai nơi, không thêm cả hai:

- `<project>/.junie/mcp/mcp.json`
- `~/.junie/mcp/mcp.json`

```json
{
  "mcpServers": {
    "serena-remote": {
      "command": "ssh",
      "args": [
        "-T",
        "serena-vps",
        "run-serena",
        "project-a",
        "junie"
      ]
    }
  }
}
```

### JetBrains GitHub Copilot

Settings / Tools / GitHub Copilot / Model Context Protocol:

```json
{
  "servers": {
    "serena-remote": {
      "type": "stdio",
      "command": "ssh",
      "args": [
        "-T",
        "serena-vps",
        "run-serena",
        "project-a",
        "jb-copilot-plugin"
      ]
    }
  }
}
```

Ưu tiên remote backend. Subagent do IDE tạo có thể không được kế thừa MCP.

## Chạy hai project đồng thời

Tạo hai MCP entries:

```text
serena-project-a -> run-serena project-a CONTEXT
serena-project-b -> run-serena project-b CONTEXT
```

Hai container:

- có project mount riêng;
- có `/srv/serena/homes/<id>` riêng;
- không dùng port;
- không dùng chung active project;
- có cùng limit mặc định trừ khi admin tách wrapper/config.

Theo dõi:

```bash
docker stats --no-stream
docker ps --filter label=serena.on-demand=true
```

Nếu RAM không đủ, chỉ enable một MCP entry tại một thời điểm hoặc tăng VPS.

## Kiểm tra bảo mật và vận hành

### Không có port Serena public

```bash
sudo ss -lntp
docker ps --format 'table {{.Names}}\t{{.Ports}}'
```

Container Serena không được có host port mapping.

### Project isolation

Trong lúc `project-a` đang chạy:

```bash
container_id=$(docker ps -q --filter label=serena.project=project-a)
docker inspect "$container_id" --format '{{json .Mounts}}'
```

Không được thấy path của `project-b`.

### Security options và limits

```bash
docker inspect "$container_id" --format \
    'memory={{.HostConfig.Memory}} nano_cpus={{.HostConfig.NanoCpus}} pids={{.HostConfig.PidsLimit}} privileged={{.HostConfig.Privileged}}'

docker inspect "$container_id" --format \
    'cap_drop={{json .HostConfig.CapDrop}} security_opt={{json .HostConfig.SecurityOpt}}'
```

`privileged` phải là `false`; `CapDrop` phải có `ALL`; security options phải có
`no-new-privileges`.

### Container tự xóa

Đóng MCP client hoặc ngắt command test, sau đó:

```bash
docker ps -a --filter label=serena.project=project-a
ls -la /srv/serena/run
```

Không còn container hoặc CID file của phiên đã đóng.

### Index còn tồn tại

```bash
du -sh /srv/serena/projects/project-a/.serena
find /srv/serena/projects/project-a/.serena/cache -maxdepth 2 -type f | head
du -sh /srv/serena/homes/project-a
```

### Audit SSH

```bash
sudo journalctl -u ssh --since today
```

Không log secret hoặc nội dung source. Nếu cần audit project usage, wrapper có
thể thêm:

```bash
logger -t serena-mcp "start project=$project_id context=$context user=$USER"
```

## Cập nhật và rollback image

Admin cập nhật `/etc/serena-docker.env`, không để client chọn image:

```bash
sudoedit /etc/serena-docker.env
docker pull ghcr.io/oraios/serena:PINNED_VERSION
```

Container đang chạy tiếp tục dùng image cũ. Phiên mới dùng cấu hình mới.

Nếu có lỗi:

1. đổi lại tag/digest cũ;
2. khởi động phiên MCP mới;
3. kiểm tra log trong `/srv/serena/homes/<id>/logs`;
4. chỉ xóa cache khi đã xác nhận cache không tương thích.

Không xóa toàn bộ `.serena` như bước rollback đầu tiên vì memories và project
config cũng nằm ở đó.

## Xử lý sự cố

### SSH báo command bị từ chối

Kiểm tra command đúng dạng:

```text
run-serena project-a codex
```

Project ID và context phân biệt chữ hoa/thường.

### Docker permission denied

Kiểm tra user:

```bash
id serena
sudo -u serena docker version
```

Sau khi thêm group, cần tạo SSH session mới.

### Container còn sót sau crash

```bash
docker ps -a --filter label=serena.on-demand=true
docker rm -f CONTAINER_ID
```

Chỉ xóa container có label Serena; không prune toàn bộ Docker host đang chạy
nhiều dịch vụ.

### OOM

```bash
journalctl -k | grep -i oom
docker inspect CONTAINER_ID --format '{{.State.OOMKilled}}'
```

Giảm số project đồng thời, tăng limit nếu host còn RAM, hoặc tăng cấu hình VPS.

### Index lại từ đầu ở mỗi phiên

Kiểm tra quyền ghi:

```bash
sudo -u serena test -w /srv/serena/projects/project-a/.serena
sudo -u serena test -w /srv/serena/homes/project-a
```

Kiểm tra `SERENA_HOME` trong `docker inspect` và log shutdown của phiên trước.

## Checklist VPS

- [ ] Đã tính RAM từ peak load của dịch vụ 24/7.
- [ ] Image được ghim tag hoặc digest.
- [ ] Project config là allowlist root-owned.
- [ ] Client không truyền path, image hoặc Docker flags.
- [ ] SSH key dùng forced command và không có PTY/forwarding.
- [ ] Không public MCP/dashboard port.
- [ ] Không mount Docker socket vào container.
- [ ] Mỗi project có `.serena` và `SERENA_HOME` riêng.
- [ ] Wrapper có CID file và cleanup trap.
- [ ] Agent built-in tools và Serena nhìn cùng working tree.
- [ ] Đã test command hợp lệ và command bị từ chối.
- [ ] Đã test container tự xóa và index còn tồn tại.
- [ ] Đã test resource limit trước khi chạy hai project.
