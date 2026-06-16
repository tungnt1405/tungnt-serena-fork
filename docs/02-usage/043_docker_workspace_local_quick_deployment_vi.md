(docker-workspace-local-quick-deployment)=
# Triển khai nhanh Serena Docker ngay trong project

Tài liệu này dành cho trường hợp cần dùng Serena nhanh trên:

- máy tester;
- máy khách hàng;
- máy cá nhân không dùng để phát triển thường xuyên;
- một project cần smoke test trước khi cài tooling cố định.

Máy host chỉ cần Docker. Không cần cài Python, `uv`, Serena hoặc wrapper dùng
chung toàn máy.

Thiết lập tạo một wrapper riêng tại:

```text
<project>/.serena-local/serena-docker
```

Claude, Codex, Antigravity, VS Code và JetBrains trong project đó cùng gọi
wrapper này. Wrapper tự tìm project root dựa trên vị trí của chính nó, do đó
không chứa absolute project path và không cần sửa logic sau khi tạo.

## Cấu hình phần cứng trước khi triển khai

Tính theo tài nguyên còn khả dụng trong lúc index, không chỉ theo tổng cấu hình
máy:

| Trường hợp | RAM còn khả dụng | CPU | Disk trống |
| --- | ---: | ---: | ---: |
| Repo nhỏ, dùng tạm | 1-2 GB | 1-2 core | Từ 10 GB SSD |
| Dùng ổn định cho một project | Từ 4 GB | 2-4 core | Từ 20 GB SSD |
| Repo lớn hoặc language server nặng | Từ 8 GB | 4 core trở lên | Từ 40 GB SSD/NVMe |

Máy 1 GB hoặc 2 GB RAM vẫn có thể chạy repo nhỏ nếu còn đủ RAM và swap, nhưng
index có thể chậm hoặc bị OOM. Với máy đang chạy nhiều dịch vụ, giảm giới hạn
container không làm giảm nhu cầu thực của language server; hãy theo dõi:

```bash
docker stats --no-stream
docker system df
du -sh .serena
```

## Khi nào dùng cách này

| Cách triển khai | Phạm vi wrapper | Khi nên dùng |
| --- | --- | --- |
| Workspace-local trong tài liệu này | Một project | Tester, máy khách, dùng tạm, smoke test |
| Wrapper tại `~/.local/bin` | Một user, nhiều project | Máy dev cá nhân dùng thường xuyên |
| Wrapper tại `/usr/local/bin` | Toàn máy | Máy dev được quản trị tập trung |
| VPS qua SSH | Nhiều project trên server | Cần dùng từ xa, project nằm trên VPS |

Nếu thường xuyên mở nhiều project trên cùng máy dev, dùng
[Docker local theo nhu cầu](043_docker_local_on_demand_vi.md) sẽ gọn hơn vì chỉ
cần cài wrapper một lần.

## Kiến trúc

```text
MCP client của project
    |
    | stdio
    v
<project>/.serena-local/serena-docker
    |
    | docker run --rm -i
    v
Serena container
    |
    +-- project: <project> -> /workspaces/project
    +-- index:   <project>/.serena/cache
    `-- home:    <project>/.serena/docker-home
```

Khi agent mở MCP server, wrapper tạo container. Khi agent dừng MCP process hoặc
đóng stdin, Serena shutdown và Docker xóa container do có `--rm`. Source,
project config, memories, index, logs và language-server resources nằm trên bind
mount nên vẫn tồn tại cho phiên sau.

## Phạm vi tin cậy

Wrapper là script local do người vận hành tạo, không phải file Serena tự sinh.
Chỉ triển khai trong repository mà bạn tin cậy và kiểm tra nội dung trước khi
chạy.

Không dùng block bootstrap từ một branch, archive hoặc repository không rõ
nguồn. Project-local script có quyền gọi Docker và mount source của project vào
container.

## Kiểm tra trước khi cài

Đi vào đúng project root:

```bash
cd /absolute/path/to/project
```

Kiểm tra Docker:

```bash
docker version
docker info
```

Nếu đây là Git repository, kiểm tra các file Serena/MCP đã được track hay chưa:

```bash
git ls-files \
    .serena \
    .serena-local \
    .mcp.json \
    .codex/config.toml \
    .vscode/mcp.json \
    .junie/mcp/mcp.json
```

`git/info/exclude` chỉ ẩn file chưa được track. Nếu lệnh trên trả về file, local
exclude không ngăn thay đổi của file đó xuất hiện trong `git status`. Khi đó:

- không ghi đè file;
- merge đúng entry MCP cần dùng;
- kiểm tra `git diff` trước khi bàn giao project.

## Bootstrap workspace-local

Block dưới đây:

1. kiểm tra đang đứng ở project root;
2. tải image Serena;
3. lấy immutable registry digest của image;
4. tạo `.serena-local/env`;
5. tạo wrapper dùng chung cho mọi agent trong project;
6. thêm local exclude vào `.git/info/exclude` nếu project dùng Git;
7. tạo hoặc cập nhật index bằng Docker.

Chạy trên macOS hoặc Linux:

```bash
#!/usr/bin/env bash
set -euo pipefail

project_dir=$(pwd -P)
wrapper_dir="$project_dir/.serena-local"
wrapper="$wrapper_dir/serena-docker"

if git rev-parse --show-toplevel >/dev/null 2>&1; then
    git_root=$(git rev-parse --show-toplevel)
    [[ "$git_root" == "$project_dir" ]] || {
        echo "Run bootstrap at Git root: $git_root" >&2
        exit 1
    }
fi

command -v docker >/dev/null 2>&1 || {
    echo "Docker CLI is required" >&2
    exit 1
}

docker info >/dev/null

image_input=${SERENA_IMAGE:-ghcr.io/oraios/serena:latest}
docker pull "$image_input"

pinned_image=$(docker image inspect "$image_input" \
    --format '{{index .RepoDigests 0}}')

[[ "$pinned_image" == *@sha256:* ]] || {
    echo "Could not resolve an immutable image digest" >&2
    exit 1
}

mkdir -p "$wrapper_dir"

languages=${SERENA_LANGUAGES:-}
if [[ ! -f "$project_dir/.serena/project.yml" && -z "$languages" ]]; then
    printf 'Serena languages, comma-separated (example: python,typescript): '
    IFS= read -r languages
fi

if [[ -n "$languages" &&
      ! "$languages" =~ ^[a-z0-9_-]+(,[a-z0-9_-]+)*$ ]]; then
    echo "Invalid SERENA_LANGUAGES: $languages" >&2
    exit 1
fi

if [[ ! -f "$project_dir/.serena/project.yml" && -z "$languages" ]]; then
    echo "SERENA_LANGUAGES is required when creating a new Serena project" >&2
    exit 1
fi

printf '%s\n' \
    "SERENA_IMAGE=$pinned_image" \
    "SERENA_LANGUAGES=$languages" \
    "SERENA_MEMORY_LIMIT=2g" \
    "SERENA_CPU_LIMIT=2" \
    "SERENA_PIDS_LIMIT=512" \
    "SERENA_NETWORK_MODE=bridge" \
    > "$wrapper_dir/env"

cat > "$wrapper" <<'SERENA_WRAPPER'
#!/usr/bin/env bash
set -euo pipefail

usage() {
    cat >&2 <<'USAGE'
Usage:
  serena-docker index
  serena-docker CONTEXT [ro|rw]
USAGE
    exit 64
}

[[ $# -ge 1 && $# -le 2 ]] || usage

script_dir=$(cd -- "$(dirname -- "${BASH_SOURCE[0]}")" && pwd -P)
project_dir=$(cd -- "$script_dir/.." && pwd -P)
env_file="$script_dir/env"

[[ -r "$env_file" ]] || {
    echo "Missing trusted environment file: $env_file" >&2
    exit 78
}

# This file is created locally by the operator and contains trusted KEY=VALUE assignments.
# shellcheck disable=SC1090
source "$env_file"

: "${SERENA_IMAGE:?SERENA_IMAGE is missing}"
: "${SERENA_LANGUAGES:=}"
: "${SERENA_MEMORY_LIMIT:=2g}"
: "${SERENA_CPU_LIMIT:=2}"
: "${SERENA_PIDS_LIMIT:=512}"
: "${SERENA_NETWORK_MODE:=bridge}"

action=$1
mount_mode=${2:-ro}
label_context=$action

if [[ "$action" == "index" ]]; then
    [[ $# -eq 1 ]] || usage
    mount_mode=rw

    if [[ -f "$project_dir/.serena/project.yml" ]]; then
        serena_args=(project index /workspaces/project)
    else
        [[ -n "$SERENA_LANGUAGES" ]] || {
            echo "SERENA_LANGUAGES is required to create project.yml" >&2
            exit 66
        }

        serena_args=(project create --index)
        IFS=',' read -r -a languages <<< "$SERENA_LANGUAGES"

        for language in "${languages[@]}"; do
            [[ "$language" =~ ^[a-z0-9_-]+$ ]] || {
                echo "Invalid Serena language: $language" >&2
                exit 65
            }
            serena_args+=(--language "$language")
        done

        serena_args+=(/workspaces/project)
    fi
else
    case "$action" in
        claude-code|desktop-app|codex|antigravity|vscode|jb-ai-assistant|junie|jb-copilot-plugin)
            ;;
        *)
            echo "Unsupported Serena context: $action" >&2
            exit 65
            ;;
    esac

    case "$mount_mode" in
        ro|rw)
            ;;
        *)
            echo "Mount mode must be ro or rw" >&2
            exit 65
            ;;
    esac

    [[ -f "$project_dir/.serena/project.yml" ]] || {
        echo "Missing $project_dir/.serena/project.yml" >&2
        echo "Run $script_dir/serena-docker index first." >&2
        exit 66
    }

    serena_args=(
        start-mcp-server
        --transport=stdio
        --context="$action"
        --project=/workspaces/project
        --enable-web-dashboard=false
        --open-web-dashboard=false
    )
fi

mkdir -p "$project_dir/.serena"
serena_gitignore="$project_dir/.serena/.gitignore"
touch "$serena_gitignore"

gitignore_tracked=false
if command -v git >/dev/null 2>&1 &&
   git -C "$project_dir" rev-parse --git-dir >/dev/null 2>&1 &&
   git -C "$project_dir" ls-files --error-unmatch \
       .serena/.gitignore >/dev/null 2>&1; then
    gitignore_tracked=true
fi

for pattern in /cache /docker-home /project.local.yml; do
    if ! grep -qxF "$pattern" "$serena_gitignore"; then
        if [[ "$gitignore_tracked" == true ]]; then
            echo "Tracked $serena_gitignore is missing: $pattern" >&2
            echo "Add the rule intentionally or use the machine-wide wrapper." >&2
            exit 78
        fi

        printf '%s\n' "$pattern" >> "$serena_gitignore"
    fi
done

mkdir -p "$project_dir/.serena/docker-home"

docker_args=(
    run
    --rm
    -i
    --init
    --label=serena.on-demand=true
    --label=serena.workspace-local=true
    --label="serena.context=$label_context"
    --cap-drop=ALL
    --security-opt=no-new-privileges:true
    --memory="$SERENA_MEMORY_LIMIT"
    --cpus="$SERENA_CPU_LIMIT"
    --pids-limit="$SERENA_PIDS_LIMIT"
    --network="$SERENA_NETWORK_MODE"
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
cidfile="$runtime_dir/serena-workspace.$$.cid"

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
    serena "${serena_args[@]}"
SERENA_WRAPPER

chmod 0755 "$wrapper"

if git rev-parse --git-dir >/dev/null 2>&1; then
    exclude_file=$(git rev-parse --git-path info/exclude)

    for pattern in \
        "/.serena/" \
        "/.serena-local/" \
        "/.mcp.json" \
        "/.codex/config.toml" \
        "/.vscode/mcp.json" \
        "/.junie/mcp/mcp.json"
    do
        grep -qxF "$pattern" "$exclude_file" ||
            printf '%s\n' "$pattern" >> "$exclude_file"
    done
fi

"$wrapper" index

printf 'Wrapper: %s\n' "$wrapper"
printf 'Image:   %s\n' "$pinned_image"
```

Wrapper tự chạy `serena project index`. Nếu `.serena/project.yml` chưa tồn tại,
bootstrap hỏi danh sách language rồi wrapper gọi `serena project create
--index` với danh sách đó, không dựa vào prompt tương tác bên trong container.
Wrapper cũng thêm `/docker-home` vào `.serena/.gitignore` trước khi index để
language-server resources không bị quét như source của project.

Nếu `.serena/.gitignore` đã được Git track và thiếu một trong các rule
`/cache`, `/docker-home`, `/project.local.yml`, wrapper dừng thay vì sửa file
được track. Hãy review và thêm rule có chủ đích, hoặc dùng wrapper toàn máy nếu
không được phép thay đổi repository.

Có thể đặt trước để bootstrap không hỏi:

```bash
export SERENA_LANGUAGES=python,typescript
```

Sau đó kiểm tra:

```bash
test -x .serena-local/serena-docker
test -f .serena-local/env
test -f .serena/project.yml
find .serena/cache -type f -print
git status --short
```

Nếu project không dùng Git, bootstrap không có nơi để ghi local exclude. Trước
khi zip, upload hoặc bàn giao source, phải tự loại:

```text
.serena/
.serena-local/
```

và các file MCP local đã tạo.

## Wrapper dùng chung hay riêng

Trong cách này wrapper là **riêng cho project**, nhưng dùng chung cho mọi agent
trong project đó:

```text
Claude Code ---------+
Claude Desktop ------+
Codex CLI/App -------+
Antigravity ---------+--> .serena-local/serena-docker
VS Code -------------+
JetBrains -----------+
```

Không tạo một wrapper khác cho từng agent. Agent chỉ truyền context tương ứng:

| Client | Context |
| --- | --- |
| Claude Code | `claude-code` |
| Claude Desktop | `desktop-app` |
| Codex CLI/App | `codex` |
| Antigravity IDE/CLI | `antigravity` |
| VS Code | `vscode` |
| JetBrains AI Assistant | `jb-ai-assistant` |
| JetBrains Junie | `junie` |
| JetBrains GitHub Copilot | `jb-copilot-plugin` |

## Chế độ read-only và read-write

Wrapper mặc định dùng `ro`:

```bash
./.serena-local/serena-docker vscode
```

Project được mount read-only, nhưng `.serena` vẫn read-write để Serena lưu
index, memories và logs. Dùng chế độ này trên máy khách khi chỉ cần phân tích.

Chỉ dùng `rw` khi agent thực sự cần sửa source:

```bash
./.serena-local/serena-docker vscode rw
```

Quyền `rw` cho phép tool Serena sửa file trong project. Nó không thay thế review
diff, branch protection hoặc backup.

## Cấu hình MCP cho tất cả client

Các ví dụ dùng:

```text
<PROJECT_ROOT>/.serena-local/serena-docker
```

Thay `<PROJECT_ROOT>` bằng absolute path thật. Không thay nội dung wrapper.

Nếu file cấu hình đã tồn tại, chỉ merge object/server entry `serena-local`.
Không thay toàn bộ file bằng ví dụ.

### Nguyên tắc cấu hình theo từng project

Với workspace-local deployment, cấu hình MCP cũng nên đặt theo từng project. Mỗi
project có wrapper riêng và wrapper tự suy ra project root từ vị trí của chính
nó:

```text
<PROJECT_ROOT>/.serena-local/serena-docker
```

Vì vậy, nếu đưa entry này vào config global của một agent, agent đó sẽ luôn trỏ
về project chứa wrapper đã hardcode, kể cả khi bạn đang mở project khác. Cách an
toàn là đặt config ở file project-local khi client hỗ trợ:

| Client | Nên cấu hình ở đâu |
| --- | --- |
| Claude Code CLI | `<project>/.mcp.json` hoặc `claude mcp add --scope project` |
| Codex CLI/App | `<project>/.codex/config.toml` |
| VS Code Copilot | `<project>/.vscode/mcp.json` |
| JetBrains Junie | `<project>/.junie/mcp/mcp.json` |
| Antigravity | project/workspace MCP config nếu phiên bản hỗ trợ |
| JetBrains AI Assistant / Copilot | project-scoped MCP config nếu phiên bản hỗ trợ |
| Claude Desktop | chỉ có user-level config; đặt tên server kèm project và xóa khi không dùng |

Không thêm cùng một Serena wrapper vào cả project config và global config của
cùng client. Nếu một IDE chỉ hỗ trợ user-level/global MCP config, đặt tên server
có project name, ví dụ `serena-marketplace`, và kiểm tra kỹ `command` trước khi
dùng trong project khác.

Bootstrap và index chỉ cần chạy một lần cho project:

```bash
cd <PROJECT_ROOT>
./.serena-local/serena-docker index
```

Sau đó IDE/agent sẽ tự start MCP server qua config của chính nó. Không chạy
`./.serena-local/serena-docker <context> ro` bằng tay trừ khi đang smoke test
stdio.

### Claude Code CLI

Đăng ký theo project, mặc định read-only:

```bash
claude mcp add --scope project serena-local -- \
    "<PROJECT_ROOT>/.serena-local/serena-docker" \
    claude-code ro
```

Hoặc merge vào `<project>/.mcp.json`:

```json
{
  "mcpServers": {
    "serena-local": {
      "command": "<PROJECT_ROOT>/.serena-local/serena-docker",
      "args": [
        "claude-code",
        "ro"
      ]
    }
  }
}
```

Kiểm tra:

```bash
claude mcp get serena-local
```

Khi cần chỉnh source, đổi argument cuối sang `rw`.

### Claude Desktop

Claude Desktop dùng config user-level, không nằm trong project. Mở Settings /
Developer / Edit Config và merge vào `claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "serena-local-project": {
      "command": "<PROJECT_ROOT>/.serena-local/serena-docker",
      "args": [
        "desktop-app",
        "ro"
      ]
    }
  }
}
```

Vị trí thường dùng:

- macOS: `~/Library/Application Support/Claude/claude_desktop_config.json`
- Windows host không thuộc phạm vi wrapper Bash này.

Thoát hoàn toàn Claude Desktop rồi mở lại. Khi bỏ project, xóa entry
`serena-local-project` khỏi config global.

### Codex CLI

Ưu tiên cấu hình trong project:

```toml
# <project>/.codex/config.toml

[mcp_servers.serena-local]
enabled = true
command = "<PROJECT_ROOT>/.serena-local/serena-docker"
args = ["codex", "ro"]
startup_timeout_sec = 120
tool_timeout_sec = 600
```

Kiểm tra từ đúng project root:

```bash
cd <PROJECT_ROOT>
codex mcp list
```

Kết quả phải có `serena-local` ở trạng thái `enabled`. Nếu chạy lệnh trên ở
project khác mà vẫn thấy server trỏ về project này, bạn đã cấu hình nhầm vào
global `~/.codex/config.toml`.

Chỉ dùng `codex mcp add` khi bạn thật sự muốn cấu hình user-level cho một máy
chỉ dùng một project Serena:

```bash
codex mcp add serena-local -- \
    "<PROJECT_ROOT>/.serena-local/serena-docker" \
    codex ro
```

Không thêm cùng server vào cả project config và `~/.codex/config.toml`.

### Codex App

Codex App và Codex CLI dùng cùng định dạng `config.toml`. Với nhiều project,
merge block TOML ở trên vào `<project>/.codex/config.toml`, không đặt wrapper
project-local vào global config. Sau đó mở hoặc tạo thread mới từ đúng project.

Codex App không đọc `.vscode/mcp.json`; VS Code chạy được không có nghĩa Codex
đã nhận Serena. Kiểm tra bằng:

```bash
cd <PROJECT_ROOT>
codex mcp list
docker ps --filter label=serena.workspace-local=true
```

`codex mcp list` xác nhận Codex đã nạp config. Container Docker chỉ xuất hiện
sau khi Codex thật sự start MCP server trong thread.

Nếu app giữ MCP process nền, container tiếp tục chạy đến khi app dừng server
hoặc thoát. Đây là vòng đời bình thường của stdio MCP.

### Antigravity IDE và Antigravity CLI

Merge entry stdio sau vào project/workspace MCP config nếu phiên bản Antigravity
hỗ trợ. Nếu chỉ có màn hình global MCP, đặt tên server theo project và tránh
dùng lại entry này khi mở project khác:

```json
{
  "mcpServers": {
    "serena-local": {
      "command": "<PROJECT_ROOT>/.serena-local/serena-docker",
      "args": [
        "antigravity",
        "ro"
      ]
    }
  }
}
```

Antigravity IDE và CLI có thể dùng config khác nhau tùy phiên bản. Giữ nguyên
`command` và `args`; không chuyển sang HTTP và không public port.

### VS Code với GitHub Copilot

Merge vào `<project>/.vscode/mcp.json`:

```json
{
  "servers": {
    "serena-local": {
      "type": "stdio",
      "command": "${workspaceFolder}/.serena-local/serena-docker",
      "args": [
        "vscode",
        "ro"
      ]
    }
  }
}
```

`${workspaceFolder}` giúp config không chứa absolute path. Mở Copilot Chat,
chọn Agent mode, mở Tools và bật `serena-local`.

Nếu workspace có nhiều root, đặt config ở đúng root hoặc dùng absolute path để
tránh mount nhầm project.

### JetBrains AI Assistant

Vào Settings / Tools / AI Assistant / MCP và merge. Ưu tiên project-scoped
configuration nếu IDE hỗ trợ:

```json
{
  "mcpServers": {
    "serena-local": {
      "command": "<PROJECT_ROOT>/.serena-local/serena-docker",
      "args": [
        "jb-ai-assistant",
        "ro"
      ]
    }
  }
}
```

Nếu IDE chỉ hỗ trợ global config, đặt tên server có project name và xóa entry
sau khi kết thúc test.

### JetBrains Junie

Merge vào `<project>/.junie/mcp/mcp.json`:

```json
{
  "mcpServers": {
    "serena-local": {
      "command": "<PROJECT_ROOT>/.serena-local/serena-docker",
      "args": [
        "junie",
        "ro"
      ]
    }
  }
}
```

Không cấu hình đồng thời cùng server trong
`~/.junie/mcp/mcp.json` và project config.

### JetBrains GitHub Copilot

Vào Settings / Tools / GitHub Copilot / Model Context Protocol (MCP) và merge:

```json
{
  "servers": {
    "serena-local": {
      "type": "stdio",
      "command": "<PROJECT_ROOT>/.serena-local/serena-docker",
      "args": [
        "jb-copilot-plugin",
        "ro"
      ]
    }
  }
}
```

Mở Copilot Agent mode và kiểm tra Serena trong tool list.

## Kiểm tra triển khai

### Kiểm tra wrapper

```bash
bash -n .serena-local/serena-docker
./.serena-local/serena-docker index
```

### Kiểm tra MCP stdio

```bash
./.serena-local/serena-docker vscode ro
```

Nếu chạy trực tiếp trong terminal không có MCP client, stdin có thể đóng ngay.
Log vẫn phải cho thấy Serena activate đúng project trước khi shutdown.

### Kiểm tra client nhận đúng project

Mở terminal ở đúng project root rồi kiểm tra client tương ứng:

```bash
cd <PROJECT_ROOT>
codex mcp list
claude mcp get serena-local
```

Với VS Code, kiểm tra file MCP đang nằm trong đúng workspace:

```bash
test -f .vscode/mcp.json
```

Nếu một client vẫn thấy Serena khi đang đứng ở project khác chưa cấu hình
Serena, kiểm tra và gỡ entry global của client đó. Với Codex, entry global nằm ở
`~/.codex/config.toml`; với Claude Desktop, entry nằm trong
`claude_desktop_config.json`.

Container Docker chỉ xác nhận server đã được start, không thay thế kiểm tra
config:

```bash
docker ps --filter label=serena.workspace-local=true
```

### Kiểm tra container tự xóa

Trong lúc agent đang dùng Serena:

```bash
docker ps --filter label=serena.workspace-local=true
```

Sau khi dừng MCP:

```bash
docker ps -a --filter label=serena.workspace-local=true
```

Container của phiên đã đóng không còn xuất hiện.

### Kiểm tra index được giữ

```bash
find .serena/cache -type f -print
du -sh .serena/cache .serena/docker-home
```

Chạy lại agent không xóa các thư mục này. Serena có thể cập nhật cache nếu
source hoặc phiên bản language server thay đổi.

### Kiểm tra Git sạch

```bash
git status --short
git check-ignore -v \
    .serena/project.yml \
    .serena-local/serena-docker \
    .vscode/mcp.json
```

Nếu file đã được track từ trước, `git check-ignore` không làm nó trở thành
untracked. Dùng:

```bash
git diff -- .serena .mcp.json .codex .vscode .junie
```

để kiểm tra thay đổi thực tế.

## Cập nhật image

Bootstrap đã lưu digest vào `.serena-local/env`. Để cập nhật:

```bash
image_input=ghcr.io/oraios/serena:latest
docker pull "$image_input"
pinned_image=$(docker image inspect "$image_input" \
    --format '{{index .RepoDigests 0}}')

sed -i.bak \
    "s|^SERENA_IMAGE=.*|SERENA_IMAGE=$pinned_image|" \
    .serena-local/env
rm -f .serena-local/env.bak

./.serena-local/serena-docker index
```

Trên macOS và Linux, `sed -i.bak` tạo backup rồi lệnh sau xóa backup. Kiểm tra
`git status` và smoke test MCP sau khi đổi image.

## Thay đổi giới hạn tài nguyên

Chỉnh `.serena-local/env`, không chỉnh wrapper:

```bash
SERENA_IMAGE=ghcr.io/oraios/serena@sha256:...
SERENA_LANGUAGES=python,typescript
SERENA_MEMORY_LIMIT=2g
SERENA_CPU_LIMIT=2
SERENA_PIDS_LIMIT=512
SERENA_NETWORK_MODE=bridge
```

Sau khi language server đã được tải đầy đủ và project không cần outbound
network, có thể thử:

```bash
SERENA_NETWORK_MODE=none
```

Không dùng `none` mặc định cho mọi project vì package manager hoặc language
server có thể cần tải dependency.

## Troubleshooting

### `Missing .serena/project.yml`

Chạy:

```bash
./.serena-local/serena-docker index
```

### MCP timeout trong lần đầu

Lần đầu có thể phải tải language server. Tăng startup timeout của client lên
120 giây hoặc cao hơn và kiểm tra:

```bash
find .serena/docker-home/language_servers -type d -print
```

### Container bị OOM

Kiểm tra:

```bash
docker inspect <container-id> --format '{{.State.OOMKilled}}'
docker stats --no-stream
```

Tăng `SERENA_MEMORY_LIMIT` hoặc dừng dịch vụ khác. Máy có tổng RAM 2 GB nhưng
chỉ còn vài trăm MB không phù hợp để index repo lớn.

### Agent không thấy server

Kiểm tra:

```bash
test -x .serena-local/serena-docker
bash -n .serena-local/serena-docker
```

Sau đó kiểm tra absolute path trong config, JSON/TOML syntax và log:

```bash
find .serena/docker-home/logs -type f -print | sort | tail
```

### Container còn sót

`--rm` và cleanup trap xử lý shutdown bình thường. Sau Docker daemon crash hoặc
`SIGKILL`, kiểm tra:

```bash
docker ps -a --filter label=serena.workspace-local=true
```

Một số language server có thể làm Serena chậm shutdown sau `Ctrl+C`. Nếu log
dừng lâu ở bước shutdown, mở terminal khác, xác nhận đúng label/context rồi mới
xóa container. Đóng MCP từ client để stdin đóng thường ổn định hơn ngắt liên
tiếp bằng `Ctrl+C`.

Chỉ xóa container sau khi xác nhận nó thuộc phiên Serena đã chết:

```bash
docker rm -f <container-id>
```

## Gỡ bỏ

1. Xóa entry `serena-local` khỏi từng MCP client.
2. Dừng mọi agent đang giữ MCP process.
3. Kiểm tra không còn container:

```bash
docker ps -a --filter label=serena.workspace-local=true
```

4. Xóa tooling và state local:

```bash
rm -rf .serena-local .serena
```

Lệnh trên xóa cả index, logs, memories và `.serena/project.yml`. Không chạy nếu
project đã dùng hoặc track `.serena` cho mục đích khác.

5. Nếu muốn dọn local exclude, mở file:

```bash
git rev-parse --git-path info/exclude
```

và xóa đúng các dòng đã thêm. Không cần sửa shared `.gitignore`.

## Checklist

- [ ] Đang đứng đúng project root.
- [ ] Repository và block bootstrap được tin cậy.
- [ ] Image đã được lưu bằng digest trong `.serena-local/env`.
- [ ] `.serena/project.yml` và cache đã được tạo.
- [ ] Wrapper qua `bash -n`.
- [ ] MCP config được merge, không ghi đè file hiện hữu.
- [ ] Máy khách dùng `ro` trừ khi thực sự cần sửa source.
- [ ] Không public port, không mount Docker socket.
- [ ] `git status` không có file local bị push nhầm.
- [ ] Container biến mất sau khi đóng agent.
