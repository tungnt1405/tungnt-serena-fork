# Triển khai Serena bằng Docker local theo nhu cầu

Tài liệu này hướng dẫn chạy Serena trong container dùng một lần:

- MCP client tự tạo container khi bắt đầu phiên;
- Serena giao tiếp qua `stdio`, không mở cổng mạng;
- container chạy với `docker run --rm -i`;
- khi MCP process kết thúc, container tự xóa;
- source, cấu hình, memories, index và language-server resources vẫn được giữ
  trên máy host.

Mỗi MCP entry trong tài liệu gắn với đúng một project. Đây là điều kiện quan
trọng để tránh chồng chéo active project, index và cache.

Hướng dẫn này cài một wrapper dùng chung toàn máy tại `~/.local/bin` hoặc
`/usr/local/bin`. Nếu chỉ cần triển khai nhanh cho một project trên máy tester,
máy khách hoặc máy cá nhân dùng tạm, xem
[Docker workspace-local triển khai nhanh](043_docker_workspace_local_quick_deployment_vi.md).
Mô hình đó đặt wrapper trong project, dùng `.git/info/exclude` và không cần cài
tooling toàn máy.

## Cấu hình phần cứng khuyến nghị

Hãy tính theo lượng tài nguyên còn khả dụng trong lúc Serena index project,
không chỉ theo tổng cấu hình máy:

| Mức sử dụng | RAM còn khả dụng | CPU | Disk trống |
| --- | ---: | ---: | ---: |
| Repo nhỏ, một project | 1-2 GB | 1-2 core | Từ 10 GB SSD |
| Sử dụng ổn định | Từ 4 GB | 2-4 core | Từ 20 GB SSD |
| Repo lớn hoặc hai project đồng thời | Từ 8 GB | 4 core trở lên | Từ 40 GB SSD/NVMe |

Nhu cầu thực tế phụ thuộc vào language server, số file, dependency và build
tool. TypeScript monorepo, Java, C#, Rust hoặc project có dependency lớn thường
cần nhiều RAM và disk hơn repo Python nhỏ.

Theo dõi tải thật sau lần index đầu:

```bash
docker stats --no-stream
du -sh /path/to/project/.serena
docker system df
```

## Kiến trúc và vòng đời

```text
MCP client
    |
    | stdio
    v
/usr/local/bin/serena-docker-local
    |
    v
docker run --rm -i
    |
    +-- /workspaces/project
    +-- /workspaces/project/.serena
    `-- /workspaces/project/.serena/docker-home
```

Quy trình sử dụng:

1. MCP client khởi động wrapper.
2. Wrapper kiểm tra project, context và quyền mount.
3. Docker tạo container Serena.
4. Serena activate đúng `/workspaces/project`.
5. Khi MCP client dừng server, stdin đóng và Serena shutdown.
6. Docker xóa container do có `--rm`.
7. Dữ liệu trên bind mount vẫn còn cho phiên sau.

Không cần chạy `docker compose up`, `docker stop` hoặc đăng nhập vào container
để quản lý vòng đời thông thường.

## Chọn image Serena

Wrapper dùng biến `SERENA_IMAGE`. Có thể dùng cùng cấu hình cho:

```bash
# Image phát hành chính thức, ghim tag hoặc digest
export SERENA_IMAGE="ghcr.io/oraios/serena:PINNED_VERSION"

# Hoặc image đã publish từ fork
export SERENA_IMAGE="registry.example.com/serena-fork:PINNED_VERSION"
```

Với môi trường cần khả năng tái lập chặt chẽ, ghim cả digest:

```bash
export SERENA_IMAGE="ghcr.io/oraios/serena:PINNED_VERSION@sha256:PINNED_DIGEST"
```

Không dùng `latest` cho cấu hình vận hành ổn định. Khi đổi image, index project
vẫn nằm trên host; tuy nhiên language server hoặc cache có thể được Serena nâng
cấp hay tạo lại nếu định dạng thay đổi.

## Chuẩn bị project

Project phải có `.serena/project.yml`:

```bash
cd /path/to/project-a
serena project create --index
```

Nếu máy host không cài Serena, có thể tạo cấu hình bằng container tạm sau khi
đã đặt `SERENA_IMAGE`:

```bash
project_dir=$(cd /path/to/project-a && pwd -P)
mkdir -p "$project_dir/.serena/docker-home"

docker run --rm -i \
    --env=SERENA_HOME=/workspaces/project/.serena/docker-home \
    --volume="$project_dir:/workspaces/project:rw" \
    "$SERENA_IMAGE" \
    serena project create --index
```

Sau đó kiểm tra `.serena/project.yml` trước khi đăng ký MCP server.

### Chọn language key

Danh sách language key phụ thuộc vào image Serena đang dùng. Image hiện tại hỗ
trợ các key sau:

```text
al, angular, ansible, bash, clojure, cpp, cpp_ccls, crystal,
csharp, csharp_omnisharp, dart, elixir, elm, erlang, fortran,
fsharp, go, groovy, haskell, haxe, hlsl, html, java, json, julia,
kotlin, lean4, lua, luau, markdown, matlab, msl, nix, ocaml,
pascal, perl, php, php_phpactor, powershell, python, python_jedi,
python_ty, r, rego, ruby, ruby_solargraph, rust, scala, scss,
solidity, svelte, swift, systemverilog, terraform, toml,
typescript, typescript_vts, vue, yaml, zig
```

Các mapping dễ nhầm:

| Dự án/file | Dùng key |
| --- | --- |
| JavaScript, React JS, `.js`, `.jsx` | `typescript` |
| TypeScript, React TS, `.ts`, `.tsx` | `typescript` |
| Angular | `angular` |
| Svelte | `svelte` |
| CSS, SCSS, Sass | `scss` |
| C | `cpp` |

Khi project có nhiều ngôn ngữ, truyền nhiều `--language` hoặc sửa
`.serena/project.yml`:

```yaml
languages:
- python
- typescript
```

Language đầu tiên là default/fallback. Sau khi đổi language list, chạy index lại:

```bash
serena project index /path/to/project-a
```

Không đặt Docker `--workdir` sang project. Image Serena phát hành kích hoạt virtual
environment từ working directory mặc định của image; Serena đã nhận project qua
`--project=/workspaces/project`.

## Cài wrapper local

Tạo `/usr/local/bin/serena-docker-local` hoặc một path khác có trong `PATH`:

```bash
#!/usr/bin/env bash
set -euo pipefail

usage() {
    echo "Usage: serena-docker-local PROJECT_PATH CONTEXT [rw|ro]" >&2
    exit 64
}

[[ $# -eq 2 || $# -eq 3 ]] || usage

project_input=$1
context=$2
mount_mode=${3:-rw}

case "$context" in
    claude-code|desktop-app|codex|antigravity|vscode|jb-ai-assistant|junie|jb-copilot-plugin)
        ;;
    *)
        echo "Unsupported Serena context: $context" >&2
        exit 65
        ;;
esac

case "$mount_mode" in
    rw|ro)
        ;;
    *)
        echo "Mount mode must be rw or ro" >&2
        exit 65
        ;;
esac

[[ -d "$project_input" ]] || {
    echo "Project directory does not exist: $project_input" >&2
    exit 66
}

project_dir=$(cd "$project_input" && pwd -P)
mkdir -p "$project_dir/.serena/docker-home"

[[ -f "$project_dir/.serena/project.yml" ]] || {
    echo "Missing $project_dir/.serena/project.yml" >&2
    echo "Create the Serena project configuration before starting MCP." >&2
    exit 66
}

env_file=${SERENA_DOCKER_ENV_FILE:-"$HOME/.config/serena-docker/env"}
if [[ -r "$env_file" ]]; then
    # This file belongs to the local user and contains trusted KEY=VALUE assignments.
    # shellcheck disable=SC1090
    source "$env_file"
fi

: "${SERENA_IMAGE:?Set SERENA_IMAGE to a pinned Serena image}"

memory_limit=${SERENA_MEMORY_LIMIT:-2g}
cpu_limit=${SERENA_CPU_LIMIT:-2}
pids_limit=${SERENA_PIDS_LIMIT:-512}
network_mode=${SERENA_NETWORK_MODE:-bridge}

docker_args=(
    run
    --rm
    -i
    --init
    --label=serena.on-demand=true
    --cap-drop=ALL
    --security-opt=no-new-privileges:true
    --memory="$memory_limit"
    --cpus="$cpu_limit"
    --pids-limit="$pids_limit"
    --network="$network_mode"
    --env=SERENA_HOME=/workspaces/project/.serena/docker-home
    --env=SERENA_USAGE_REPORTING=false
)

if [[ "$mount_mode" == "rw" ]]; then
    docker_args+=(--volume="$project_dir:/workspaces/project:rw")
else
    docker_args+=(
        --volume="$project_dir:/workspaces/project:ro"
        --volume="$project_dir/.serena:/workspaces/project/.serena:rw"
    )
fi

runtime_dir=${XDG_RUNTIME_DIR:-"/tmp/serena-docker-$UID"}
mkdir -p "$runtime_dir"
cidfile="$runtime_dir/serena.$$.cid"

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

docker "${docker_args[@]}" --cidfile="$cidfile" "$SERENA_IMAGE" \
    serena start-mcp-server \
    --transport=stdio \
    --context="$context" \
    --project=/workspaces/project \
    --enable-web-dashboard=false \
    --open-web-dashboard=false
```

Cấp quyền thực thi:

```bash
sudo chmod 0755 /usr/local/bin/serena-docker-local
```

Nếu không muốn ghi vào `/usr/local/bin`, đặt script tại
`$HOME/.local/bin/serena-docker-local` và dùng absolute path đó trong cấu hình
MCP.

### Biến môi trường vận hành

Khuyến nghị tạo `$HOME/.config/serena-docker/env` để CLI, desktop app và IDE
dùng cùng cấu hình:

```bash
mkdir -p "$HOME/.config/serena-docker"
cat > "$HOME/.config/serena-docker/env" <<'EOF'
SERENA_IMAGE=ghcr.io/oraios/serena:PINNED_VERSION
SERENA_MEMORY_LIMIT=2g
SERENA_CPU_LIMIT=2
SERENA_PIDS_LIMIT=512
SERENA_NETWORK_MODE=bridge
EOF
chmod 0600 "$HOME/.config/serena-docker/env"
```

File được `source`, vì vậy chỉ user hiện tại được sửa nó. Không đặt nội dung lấy
từ project không tin cậy vào file này.

`bridge` cho phép outbound network và có thể truy cập một số dịch vụ trên host
hoặc LAN tùy cấu hình Docker/firewall. Nếu project đã có đủ language server và
dependency, thử `SERENA_NETWORK_MODE=none`. Không dùng `none` mặc định cho mọi
project vì lần khởi động đầu hoặc build tool có thể cần mạng.

## Quyền ghi và user trong container

Image Serena hiện có thể chạy bằng `root` trong container. Dù đã drop
capabilities, process vẫn có thể tạo file thuộc sở hữu `root` trên bind mount
của Linux host.

Không thêm `--user "$(id -u):$(id -g)"` một cách máy móc: image hoặc language
server có thể phụ thuộc vào home, PATH hay file đã cài cho user mặc định. Nếu
muốn chạy non-root, hãy kiểm tra image cụ thể trước:

```bash
docker run --rm "$SERENA_IMAGE" id
docker run --rm --user "$(id -u):$(id -g)" "$SERENA_IMAGE" serena --help
```

Nếu test thành công, có thể thêm vào `docker_args`:

```bash
--user="$(id -u):$(id -g)"
```

Trên macOS và Windows với Docker Desktop, ownership của bind mount được lớp
chia sẻ file xử lý khác Linux.

## Chế độ read-write và read-only

### Read-write

Dùng khi Serena cần sửa source:

```bash
serena-docker-local /path/to/project-a codex rw
```

Toàn bộ project được mount `rw`; `.serena` và index được ghi trực tiếp trong
repo.

### Read-only

Dùng khi chỉ cần phân tích code:

```bash
serena-docker-local /path/to/project-a desktop-app ro
```

Project được mount `ro`, sau đó `.serena` được mount lồng với quyền `rw` để
Serena lưu cache và memories.

Một số language server hoặc build tool cần tạo file trong project, ví dụ build
cache, generated files hoặc dependency directory. Khi đó chế độ `ro` có thể
khởi động thất bại hoặc thiếu symbol. Chuyển sang `rw` hoặc chuẩn bị dependency
trên host trước khi chạy.

## `--rm` ảnh hưởng thế nào đến index?

`--rm` chỉ xóa writable layer và metadata của container. Nó không xóa bind
mount trên host.

| Dữ liệu | Vị trí | Sau khi container bị xóa |
| --- | --- | --- |
| Source | `<project>/` | Giữ nguyên |
| Project config | `<project>/.serena/project.yml` | Giữ nguyên |
| Memories | `<project>/.serena/memories/` | Giữ nguyên |
| Symbol cache/index | `<project>/.serena/cache/` | Giữ nguyên |
| Global config và logs | `<project>/.serena/docker-home/` | Giữ nguyên |
| Language server đã tải | `<project>/.serena/docker-home/language_servers/` | Giữ nguyên |
| File chỉ ghi trong container layer | Không có bind mount | Bị xóa |

Khi stdio đóng bình thường, Serena gọi shutdown và yêu cầu language server lưu
cache. Phiên sau vẫn phải khởi động process language server, nhưng có thể tái sử
dụng cache đã lưu. CID file và cleanup trap cũng dọn container khi wrapper nhận
`HUP`, `INT` hoặc `TERM`.

Nếu container bị OOM, `docker kill`, Docker daemon crash hoặc máy mất điện:

- source và cache đã ghi trước đó vẫn còn;
- phần cache mới chỉ nằm trong memory có thể chưa được flush;
- Serena có thể phải index lại file thay đổi ở phiên tiếp theo;
- đây là suy giảm hiệu năng tạm thời, không phải mất source.

Nếu wrapper bị `SIGKILL` hoặc toàn bộ Docker daemon crash, trap không thể chạy.
Sau khi máy ổn định, kiểm tra container còn sót theo label:

```bash
docker ps -a --filter label=serena.on-demand=true
```

Không dùng `docker run --rm` mà bỏ bind mount `.serena`/project. Khi đó index sẽ
nằm trong container layer và bị xóa cùng container.

## Cấu hình MCP theo client

Các ví dụ dưới đây dùng:

```text
Project: /work/projects/project-a
Wrapper: /usr/local/bin/serena-docker-local
Mode: rw
```

Thay bằng absolute path thật. Mỗi project nên có một MCP entry riêng.

### Claude Code CLI

Đăng ký theo project:

```bash
claude mcp add --scope project serena -- \
    /usr/local/bin/serena-docker-local \
    /work/projects/project-a claude-code rw
```

Hoặc `.mcp.json` trong project:

```json
{
  "mcpServers": {
    "serena": {
      "command": "/usr/local/bin/serena-docker-local",
      "args": [
        "/work/projects/project-a",
        "claude-code",
        "rw"
      ]
    }
  }
}
```

Kiểm tra bằng:

```bash
claude mcp list
claude mcp get serena
```

Trong Claude Code, dùng `/mcp`. Lần tải language server đầu có thể lâu hơn; đặt
`MCP_TIMEOUT=60000` hoặc cao hơn trong môi trường khởi chạy Claude Code nếu
server startup bị timeout.

### Claude Desktop

Mở Settings / Developer / Edit Config và thêm vào
`claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "serena-project-a": {
      "command": "/usr/local/bin/serena-docker-local",
      "args": [
        "/work/projects/project-a",
        "desktop-app",
        "rw"
      ],
      "env": {
        "SERENA_IMAGE": "ghcr.io/oraios/serena:PINNED_VERSION",
        "SERENA_MEMORY_LIMIT": "2g",
        "SERENA_CPU_LIMIT": "2"
      }
    }
  }
}
```

Vị trí thường dùng:

- macOS: `~/Library/Application Support/Claude/claude_desktop_config.json`
- Windows: `%APPDATA%\Claude\claude_desktop_config.json`

Thoát hoàn toàn Claude Desktop rồi mở lại. Đóng cửa sổ nhưng để app chạy nền có
thể chưa dừng MCP process và container.

### Codex CLI

Đăng ký bằng CLI:

```bash
codex mcp add serena-project-a -- \
    /usr/local/bin/serena-docker-local \
    /work/projects/project-a codex rw
```

Hoặc thêm vào `~/.codex/config.toml` hay `.codex/config.toml` của trusted
project:

```toml
[mcp_servers.serena-project-a]
command = "/usr/local/bin/serena-docker-local"
args = ["/work/projects/project-a", "codex", "rw"]
startup_timeout_sec = 120
tool_timeout_sec = 600
```

Kiểm tra bằng `codex mcp --help` và `/mcp` trong Codex TUI.

### Codex App

Codex App và Codex CLI dùng chung `config.toml`. Dùng cấu hình TOML ở phần trên.
Không dựa vào working directory của app; luôn truyền absolute project path.

Từ MCP settings của app, mở `config.toml`, lưu cấu hình rồi tạo phiên mới. Khi
app giữ MCP server chạy nền, container cũng tiếp tục chạy đến lúc app dừng
server hoặc thoát.

### Antigravity IDE và Antigravity CLI

Dùng cấu hình MCP stdio:

```json
{
  "mcpServers": {
    "serena-project-a": {
      "command": "/usr/local/bin/serena-docker-local",
      "args": [
        "/work/projects/project-a",
        "antigravity",
        "rw"
      ]
    }
  }
}
```

Antigravity không phải phiên bản nào cũng hỗ trợ truyền working directory hoặc
cùng một lệnh quản lý MCP trên CLI. Dùng màn hình MCP/config JSON của phiên bản
đang cài và giữ nguyên `command`/`args` ở trên. Vì project đã được truyền trực
tiếp cho wrapper, không cần đổi active project trong một Serena instance.

Nếu Antigravity CLI dùng config riêng, thêm cùng entry stdio vào config đó;
không suy đổi sang HTTP và không chạy một Serena server dùng chung cho nhiều
project.

### VS Code với GitHub Copilot

Khuyến nghị workspace-scoped. Chạy `MCP: Add Server`, chọn `Command (stdio)`,
hoặc thêm vào `.vscode/mcp.json`:

```json
{
  "servers": {
    "serena": {
      "type": "stdio",
      "command": "/usr/local/bin/serena-docker-local",
      "args": [
        "/work/projects/project-a",
        "vscode",
        "rw"
      ]
    }
  }
}
```

Mở Copilot Chat ở Agent mode và kiểm tra Serena trong danh sách tools. Với
workspace khác, tạo entry có project path khác.

### JetBrains AI Assistant

Vào Settings / Tools / AI Assistant / MCP và thêm:

```json
{
  "mcpServers": {
    "serena": {
      "command": "/usr/local/bin/serena-docker-local",
      "args": [
        "/work/projects/project-a",
        "jb-ai-assistant",
        "rw"
      ]
    }
  }
}
```

Ưu tiên project-scoped configuration nếu IDE hỗ trợ để tránh cùng entry được
dùng nhầm cho project khác.

### JetBrains Junie

Thêm vào `<project>/.junie/mcp/mcp.json`:

```json
{
  "mcpServers": {
    "serena": {
      "command": "/usr/local/bin/serena-docker-local",
      "args": [
        "/work/projects/project-a",
        "junie",
        "rw"
      ]
    }
  }
}
```

Có thể dùng global config `~/.junie/mcp/mcp.json`, nhưng không cấu hình đồng
thời cả global và project với cùng server.

### JetBrains GitHub Copilot

Vào Settings / Tools / GitHub Copilot / Model Context Protocol (MCP):

```json
{
  "servers": {
    "serena": {
      "type": "stdio",
      "command": "/usr/local/bin/serena-docker-local",
      "args": [
        "/work/projects/project-a",
        "jb-copilot-plugin",
        "rw"
      ]
    }
  }
}
```

Mở Copilot Agent mode, kiểm tra tool list và cấu hình auto-approval phù hợp.
Subagent do IDE tạo có thể không được kế thừa MCP server.

## Chạy hai project đồng thời

Tạo hai MCP entry có tên và path khác nhau:

```text
serena-project-a -> /work/projects/project-a
serena-project-b -> /work/projects/project-b
```

Mỗi entry tạo container riêng, mount riêng và dùng
`<project>/.serena/docker-home` riêng. Không cần cấp port vì stdio không lắng
nghe trên TCP.

Kiểm tra tài nguyên trước khi chạy project thứ hai:

```bash
docker stats --no-stream
```

## Kiểm tra triển khai

### Kiểm tra wrapper

```bash
bash -n /usr/local/bin/serena-docker-local
SERENA_IMAGE="ghcr.io/oraios/serena:PINNED_VERSION" \
    /usr/local/bin/serena-docker-local \
    /work/projects/project-a codex ro
```

Lệnh thứ hai mở MCP stdio và sẽ chờ client; dùng `Ctrl+C` để kết thúc test.

### Kiểm tra container tự xóa

Trong terminal khác:

```bash
docker ps --filter ancestor="$SERENA_IMAGE"
```

Sau khi đóng MCP client:

```bash
docker ps -a --filter ancestor="$SERENA_IMAGE"
```

Container của phiên đã đóng không còn xuất hiện.

### Kiểm tra index được giữ

```bash
test -f /work/projects/project-a/.serena/project.yml
find /work/projects/project-a/.serena/cache -maxdepth 2 -type f | head
du -sh /work/projects/project-a/.serena
```

Khởi động lại cùng MCP entry và quan sát log trong:

```text
/work/projects/project-a/.serena/docker-home/logs/
```

### Kiểm tra không mở port

```bash
docker inspect "$(docker ps -q --filter ancestor="$SERENA_IMAGE" | head -1)" \
    --format '{{json .NetworkSettings.Ports}}'
```

Kết quả không được có host port đã publish.

## Xử lý sự cố

### MCP client không tìm thấy wrapper

Dùng absolute path trong `command`, không phụ thuộc `PATH` của GUI app.

### MCP startup timeout

Tăng startup timeout của client. Lần đầu Serena có thể tải language server và
index project.

### File trên Linux bị sở hữu bởi root

Kiểm tra khả năng chạy image bằng host UID/GID như phần quyền ghi. Không sửa
ownership hàng loạt trước khi xác định file nào do container tạo.

### Container còn chạy sau khi đóng cửa sổ

Desktop app hoặc IDE có thể chỉ ẩn cửa sổ và vẫn giữ MCP subprocess. Dừng MCP
server trong UI hoặc thoát hoàn toàn ứng dụng.

### Language server phải tải lại mỗi phiên

Kiểm tra:

```bash
du -sh /work/projects/project-a/.serena/docker-home
```

Nếu thư mục không tăng sau lần cài đầu, kiểm tra `SERENA_HOME` và bind mount
trong wrapper.

## Checklist local

- [ ] Image được ghim tag hoặc digest.
- [ ] Mỗi MCP entry dùng một absolute project path.
- [ ] Project có `.serena/project.yml`.
- [ ] `.serena/cache` và `docker-home` nằm trên host.
- [ ] Không publish MCP/dashboard port.
- [ ] Không mount Docker socket hoặc host root.
- [ ] Resource limits phù hợp với RAM còn trống.
- [ ] Đã kiểm tra container biến mất khi MCP process kết thúc.
- [ ] Đã kiểm tra index còn tồn tại ở phiên tiếp theo.
