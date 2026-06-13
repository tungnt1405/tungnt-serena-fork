# Hướng dẫn dùng Serena theo từng dự án để index không chồng chéo

Tài liệu này mô tả cách cấu hình và vận hành Serena khi làm việc với nhiều dự án, sao cho index, cache, memories và active project không bị lẫn giữa các repository.

Các ví dụ command trong tài liệu này dùng Linux/macOS shell (`/bin/bash`). Nếu dùng Windows PowerShell, chỉ cần đổi path kiểu `/path/to/project-a` sang `E:\path\to\project-a` và dùng các lệnh PowerShell tương đương như `Get-ChildItem`, `Get-Content`, `Test-Path`.

## Mục lục

1. [Cài đặt Serena để dùng bình thường](#cài-đặt-serena-để-dùng-bình-thường)
2. [Biến môi trường và file `.env`](#biến-môi-trường-và-file-env)
3. [Dùng repo source cho dev](#dùng-repo-source-cho-dev)
4. [Triển khai Docker local hoặc VPS](#triển-khai-docker-local-hoặc-vps)
5. [Nguyên tắc chính](#nguyên-tắc-chính)
6. [Thiết lập một project mới](#thiết-lập-một-project-mới)
7. [Khởi động Serena đúng project](#khởi-động-serena-đúng-project)
8. [Không dùng chung một Serena instance cho nhiều project độc lập](#không-dùng-chung-một-serena-instance-cho-nhiều-project-độc-lập)
9. [Cấu hình nơi lưu `.serena`](#cấu-hình-nơi-lưu-serena)
10. [Đặt tên project](#đặt-tên-project)
11. [Monorepo và multi-package](#monorepo-và-multi-package)
12. [Git worktree](#git-worktree)
13. [JetBrains backend](#jetbrains-backend)
14. [Checklist cho mỗi project](#checklist-cho-mỗi-project)
15. [Dấu hiệu đang bị chồng chéo index](#dấu-hiệu-đang-bị-chồng-chéo-index)
16. [Cách xử lý khi nghi bị nhầm project](#cách-xử-lý-khi-nghi-bị-nhầm-project)
17. [Mẫu cấu hình an toàn](#mẫu-cấu-hình-an-toàn)
18. [Tóm tắt nhanh](#tóm-tắt-nhanh)

(cài-đặt-serena-để-dùng-bình-thường)=
## Cài đặt Serena để dùng bình thường

Nếu bạn chỉ muốn dùng Serena như một tool đã cài vào máy, cài qua `uv tool`:

```bash
uv tool install -p 3.13 serena-agent
```

Sau khi cài xong, command `serena` phải có trong shell:

```bash
serena --help
```

Khởi tạo cấu hình global một lần. Lệnh này chạy ở đâu cũng được, không cần chạy trong project:

```bash
serena init
```

Nếu bạn dùng JetBrains backend:

```bash
serena init -b JetBrains
```

Sau bước này mới đi vào từng repo để tạo/index project:

```bash
cd /path/to/project-a
serena project create --index
```

Khi muốn cập nhật bản đã cài qua `uv tool`:

```bash
uv tool upgrade serena-agent
```

(biến-môi-trường-và-file-env)=
## Biến môi trường và file `.env`

Serena không tự động đọc file `.env`. File `.env` chỉ là quy ước của shell/tooling để bạn nạp environment variables trước khi chạy `serena`.

Có ba loại cấu hình cần phân biệt:

| Loại cấu hình | Đặt ở đâu | Dùng cho việc gì |
| --- | --- | --- |
| Global Serena config | `$SERENA_HOME/serena_config.yml`, mặc định là `~/.serena/serena_config.yml` | Cấu hình chung như dashboard, backend, ignored paths, project data location |
| Project Serena config | `<project>/.serena/project.yml` | Cấu hình riêng của repo: `project_name`, `languages`, ignored paths, read-only, workspace folders |
| Environment variables / `.env` | Nơi bạn tự chọn, thường là project root hoặc thư mục vận hành | Tắt usage reporting, đổi `SERENA_HOME`, truyền secret/API key cho provider ngoài, hoặc biến build của language server |

Vì vậy `.env` không thay thế cho `.serena/project.yml`. Nếu bạn muốn Serena nhớ project, languages, ignore paths hoặc nơi lưu cache, hãy chỉnh `.serena/project.yml` hoặc `serena_config.yml`, không chỉnh `.env`.

### Nên đặt `.env` ở đâu?

Nếu biến áp dụng cho đúng repo đang chạy Serena, đặt `.env` ở root của repo đó:

```bash
/path/to/project-a/.env
/path/to/project-a/.serena/project.yml
```

Sau đó khi làm việc với project:

```bash
cd /path/to/project-a
set -a
source .env
set +a
serena start-mcp-server --context codex --project-from-cwd
```

Nếu biến áp dụng cho mọi project trên máy, bạn có thể export trong shell profile (`~/.bashrc`, `~/.zshrc`) hoặc tạo một file riêng như `~/.config/serena/env` rồi source nó trước khi chạy Serena.

### Các biến môi trường Serena hỗ trợ trực tiếp

| Biến | Khi nào dùng | Ví dụ |
| --- | --- | --- |
| `SERENA_HOME` | Đổi nơi lưu global config, global memories, logs của Serena. Nếu không đặt, mặc định là `~/.serena`. | `SERENA_HOME="$HOME/.config/serena"` |
| `SERENA_USAGE_REPORTING` | Tắt usage reporting ra ngoài. Đặt `false` để không gửi usage ping. | `SERENA_USAGE_REPORTING=false` |
| `ANTHROPIC_API_KEY` | Chỉ cần nếu bạn đổi `token_count_estimator` sang chế độ dùng Anthropic API. Mặc định `CHAR_COUNT` không cần biến này. | `ANTHROPIC_API_KEY="..."` |

Ví dụ `.env` an toàn cho đa số project:

```bash
SERENA_USAGE_REPORTING=false
```

Ví dụ nếu muốn tách global Serena home cho một workspace:

```bash
SERENA_HOME="$HOME/.serena-work"
SERENA_USAGE_REPORTING=false
```

### Biến liên quan language server hoặc môi trường build

Các biến này không phải config chung của Serena, nhưng language server hoặc toolchain có thể dùng khi Serena khởi động chúng:

| Biến | Dùng khi nào |
| --- | --- |
| `JAVA_HOME` | Java/JDTLS/Groovy/Scala hoặc project cần JDK cụ thể |
| `MATLAB_PATH` | Project dùng MATLAB language server |
| `MATLAB_EXTENSION_PATH` | Dùng MATLAB extension đã cài sẵn thay vì để Serena tải extension |
| `TERRAFORM_CLI_PATH` | Project Terraform cần chỉ rõ binary `terraform` |
| `GOFLAGS` | Project Go cần build flags/tags từ environment |
| `PATH` | Serena và language servers kế thừa `PATH`; dùng để tìm `node`, `npm`, `java`, `terraform`, `go`, v.v. |

Nếu biến chỉ phục vụ một repo cụ thể, đặt trong `.env` của repo đó. Nếu biến là toolchain mặc định cho cả máy, đặt trong shell profile.

### Chạy một lần không cần file `.env`

Nếu chỉ muốn đặt biến cho một lần chạy:

```bash
SERENA_USAGE_REPORTING=false serena start-mcp-server --context codex --project-from-cwd
```

Nếu cần nhiều biến:

```bash
SERENA_HOME="$HOME/.serena-work" \
SERENA_USAGE_REPORTING=false \
serena start-mcp-server --context codex --project /path/to/project-a
```

### Docker Compose `.env`

Nếu dùng Docker Compose, file `.env` cạnh `compose.yaml` mặc định được Compose dùng để nội suy biến trong `compose.yaml`, ví dụ:

```bash
SERENA_PORT=9121
SERENA_DASHBOARD_PORT=24282
```

Các biến trên chỉ dùng để map port trong Compose. Nếu muốn truyền biến runtime vào container, cần khai báo trong `environment:` của `compose.yaml`, ví dụ:

```yaml
environment:
  - SERENA_DOCKER=1
  - SERENA_USAGE_REPORTING=false
```

Không commit `.env` nếu có secret. Repo này đã ignore `.env`, nhưng vẫn nên kiểm tra trước khi commit:

```bash
git status --short
```

(dùng-repo-source-cho-dev)=
## Dùng repo source cho dev

Nếu bạn clone repo Serena để phát triển hoặc test fork, không cần `uv tool install`. Chạy Serena trực tiếp từ source bằng `uv run`.

Clone lần đầu:

```bash
git clone https://github.com/oraios/serena
cd serena
uv run serena --help
```

Khởi tạo config global một lần nếu chưa làm:

```bash
uv run serena init
```

Chạy Serena từ source cho chính repo đang đứng:

```bash
uv run serena start-mcp-server --context codex --project-from-cwd
```

Nếu bạn đang ở một project khác nhưng muốn dùng Serena từ source repo này:

```bash
uv run --directory /path/to/serena serena start-mcp-server --context codex --project /path/to/project-a
```

Khi pull code mới cho repo dev:

```bash
cd /path/to/serena
git pull
uv sync
uv run serena --help
```

Nếu lockfile hoặc dependency đổi, `uv sync` sẽ cập nhật môi trường dev. Nếu bạn dùng bản cài qua `uv tool`, `git pull` repo source không ảnh hưởng đến command global `serena`; lúc đó phải dùng `uv tool upgrade serena-agent` hoặc chạy bằng `uv run --directory /path/to/serena serena ...`.

(triển-khai-docker-local-hoặc-vps)=
## Triển khai Docker local hoặc VPS

Nếu muốn container Serena chỉ tồn tại trong thời gian MCP client đang dùng, xem
hai hướng dẫn riêng:

- [Docker local theo nhu cầu](043_docker_local_on_demand_vi.md): MCP client tự
  chạy `docker run --rm -i`; đóng MCP process thì container tự xóa, còn
  `.serena`, memories, index và language-server resources vẫn nằm trên host.
- [Docker VPS qua SSH theo nhu cầu](044_docker_vps_ssh_on_demand_vi.md): nhiều
  project được allowlist trên VPS, mỗi phiên MCP tự tạo một container riêng và
  không public MCP/dashboard port.

Hai mô hình đều giữ nguyên nguyên tắc một container/Serena instance chỉ phục vụ
một project. Không mount một thư mục cha chứa nhiều repo nếu container chỉ cần
làm việc với một repo.

(nguyên-tắc-chính)=
## Nguyên tắc chính

Serena làm việc theo mô hình project-based. Mỗi project nên có:

- một project root riêng, thường là root của Git repository;
- một file cấu hình riêng tại `.serena/project.yml`;
- một vùng dữ liệu riêng cho Serena, mặc định là `.serena/` nằm trong project;
- một active project rõ ràng khi MCP server khởi động.

Nếu giữ mặc định, cache/index của Serena sẽ nằm trong từng repo:

```text
project-a/.serena/cache/
project-b/.serena/cache/
```

Đây là cách an toàn nhất để tránh chồng chéo.

(thiết-lập-một-project-mới)=
## Thiết lập một project mới

Đi vào đúng root của project trước:

```bash
cd /path/to/project-a
serena project create --index
```

Lệnh này tạo `.serena/project.yml` và index project ngay sau khi tạo.

Nếu project đã có `.serena/project.yml`, chỉ cần index:

```bash
cd /path/to/project-a
serena project index
```

Sau đó kiểm tra nhanh:

```bash
ls -la .serena
cat .serena/project.yml
```

Bạn nên thấy ít nhất:

```text
.serena/
  project.yml
  project.local.yml
  memories/
  cache/
```

`cache/` có thể chỉ xuất hiện sau khi index hoặc sau lần đầu dùng tool symbol.

(khởi-động-serena-đúng-project)=
## Khởi động Serena đúng project

### Cách khuyến nghị cho một project cố định

Truyền đường dẫn project rõ ràng khi khởi động MCP server:

```bash
serena start-mcp-server --context codex --project /path/to/project-a
```

Với Claude Code:

```bash
serena start-mcp-server --context claude-code --project /path/to/project-a
```

Với ChatGPT:

```bash
serena start-mcp-server --context chatgpt --project /path/to/project-a
```

Dùng `--project <path>` khi bạn muốn chắc chắn server chỉ làm việc với đúng project đó.

### Cách thuận tiện cho CLI agent

Nếu agent được mở từ trong thư mục project, dùng:

```bash
serena start-mcp-server --context codex --project-from-cwd
```

`--project-from-cwd` sẽ đi ngược lên các thư mục cha và chọn boundary gần nhất có:

- `.serena/project.yml`, hoặc
- `.git`.

Nếu đang ở trong git worktree lồng trong project khác, boundary gần nhất sẽ được ưu tiên, giúp tránh bị nhầm sang project cha.

Điều kiện quan trọng: phải mở agent từ đúng project hoặc thư mục con của project. Không nên mở agent từ một folder tổng hợp chứa nhiều repo nếu bạn không muốn folder tổng hợp đó trở thành project.

(không-dùng-chung-một-serena-instance-cho-nhiều-project-độc-lập)=
## Không dùng chung một Serena instance cho nhiều project độc lập

Serena MCP server là stateful. Một instance chỉ có một active project tại một thời điểm.

Nếu nhiều agent cùng làm trên cùng một project, có thể dùng chung một HTTP server:

```bash
serena start-mcp-server --transport streamable-http --port 9121 --project /path/to/project-a
```

Nếu nhiều agent làm trên nhiều project khác nhau, hãy chạy mỗi project một Serena instance riêng. Cách đơn giản nhất là để từng MCP client spawn server bằng stdio với `--project` hoặc `--project-from-cwd`.

Không nên dùng một HTTP server duy nhất rồi liên tục switch qua lại giữa các project độc lập, vì active project, language server và cache runtime dễ gây nhầm lẫn trong phiên làm việc.

(cấu-hình-nơi-lưu-serena)=
## Cấu hình nơi lưu `.serena`

Mặc định trong `serena_config.yml`:

```yaml
project_serena_folder_location: "$projectDir/.serena"
```

Nên giữ cấu hình này nếu không có nhu cầu đặc biệt. Mỗi repo sẽ tự quản lý data Serena của nó.

Nếu muốn lưu metadata ở một nơi trung tâm, phải đảm bảo đường dẫn có thành phần tách biệt từng project:

```yaml
project_serena_folder_location: "/home/you/serena-metadata/$projectFolderName/.serena"
```

Cách này vẫn có rủi ro nếu nhiều repo có cùng tên folder, vì cả hai có thể cùng map vào một thư mục metadata. Để an toàn hơn, dùng mặc định `$projectDir/.serena`.

Tuyệt đối tránh cấu hình kiểu này:

```yaml
project_serena_folder_location: "/home/you/serena-metadata/.serena"
```

Cấu hình trên khiến mọi project dùng chung một folder `.serena`, rất dễ gây chồng chéo `project.yml`, memories và cache.

(đặt-tên-project)=
## Đặt tên project

Trong `.serena/project.yml`, `project_name` là tên dùng để kích hoạt project bằng tên:

```yaml
project_name: "project-a"
```

Nên đặt tên rõ ràng và duy nhất, vì nếu nhiều project trùng `project_name`, Serena sẽ yêu cầu tham chiếu bằng đường dẫn.

Khuyến nghị:

```yaml
project_name: "company-api"
project_name: "company-web"
project_name: "serena-fork"
```

Không nên để nhiều repo cùng một tên chung chung:

```yaml
project_name: "app"
project_name: "backend"
project_name: "repo"
```

(monorepo-và-multi-package)=
## Monorepo và multi-package

Nếu các package cần được đọc và sửa cùng nhau trong một tác vụ, hãy coi thư mục monorepo là một Serena project:

```text
workspace/
  .serena/project.yml
  packages/
    api/
    web/
    shared/
```

Với TypeScript, nếu project cần cross-package references, có thể khai báo `additional_workspace_folders` trong `.serena/project.yml`:

```yaml
additional_workspace_folders:
  - packages/api
  - packages/web
  - packages/shared
```

Chỉ thêm các folder thật sự cần symbol/reference cross-package, vì mỗi workspace folder có thể làm tăng thời gian startup và indexing.

Nếu các repo độc lập và ít khi cần sửa chung trong một task, dùng mỗi repo một Serena project riêng thay vì gom tất cả vào một project lớn.

(git-worktree)=
## Git worktree

Với git worktree, mỗi worktree nên được xem như một project riêng nếu bạn muốn index/cache tách biệt:

```bash
cd /path/to/worktrees/feature-a
serena project create --index

cd /path/to/worktrees/feature-b
serena project create --index
```

Khi agent khởi động trong worktree, dùng:

```bash
serena start-mcp-server --context codex --project-from-cwd
```

Serena sẽ chọn boundary gần nhất có `.serena/project.yml` hoặc `.git`, nên worktree gần nhất sẽ thắng project cha.

(jetbrains-backend)=
## JetBrains backend

Nếu dùng JetBrains backend, việc indexing code do IDE xử lý. Khi đó `serena project index` không phải bước quan trọng cho symbol cache LSP.

Vẫn phải đảm bảo project root trong Serena khớp với folder đang mở trong IDE:

```bash
serena start-mcp-server --language-backend JetBrains --project /path/to/project-a
```

Nếu IDE đang mở `/path/to/project-a`, Serena cũng nên activate đúng `/path/to/project-a`, không phải parent folder hay subfolder.

(checklist-cho-mỗi-project)=
## Checklist cho mỗi project

Trước khi dùng Serena cho một repo mới:

1. Đi vào đúng root repo.
2. Chạy `serena project create --index`.
3. Kiểm tra `.serena/project.yml`.
4. Đặt `project_name` rõ ràng, duy nhất.
5. Đảm bảo `languages` chỉ gồm những ngôn ngữ cần Serena hỗ trợ symbolic tools.
6. Giữ `project_serena_folder_location: "$projectDir/.serena"` nếu không có lý do đặc biệt.
7. Khởi động MCP server bằng `--project <path>` hoặc `--project-from-cwd`.
8. Không dùng chung một HTTP Serena server cho các project khác nhau.

(dấu-hiệu-đang-bị-chồng-chéo-index)=
## Dấu hiệu đang bị chồng chéo index

Có thể đang activate nhầm project nếu thấy:

- tool symbol trả về file của repo khác;
- memories nói về project khác;
- dashboard hiện active project không đúng path hiện tại;
- `.serena/project.yml` nằm ở parent folder thay vì repo mong muốn;
- các agent làm việc trên project khác nhau nhưng kết nối cùng một HTTP endpoint;
- global `project_serena_folder_location` trỏ tất cả project vào cùng một folder.

(cách-xử-lý-khi-nghi-bị-nhầm-project)=
## Cách xử lý khi nghi bị nhầm project

Kiểm tra active project trong dashboard hoặc bằng tool Serena nếu client có hỗ trợ.

Kiểm tra thư mục hiện tại:

```bash
pwd
git rev-parse --show-toplevel
test -f .serena/project.yml && echo "exists" || echo "missing"
```

Nếu `.serena/project.yml` không nằm ở root mong muốn, tạo project config tại đúng root:

```bash
cd /path/to/correct-project
serena project create --index
```

Sau đó khởi động lại MCP server bằng đường dẫn rõ ràng:

```bash
serena start-mcp-server --context codex --project /path/to/correct-project
```

Nếu đã cấu hình `project_serena_folder_location` thành một đường dẫn chung, sửa lại về:

```yaml
project_serena_folder_location: "$projectDir/.serena"
```

Sau đó index lại từng project từ đúng root của nó:

```bash
cd /path/to/project-a
serena project index

cd /path/to/project-b
serena project index
```

(mẫu-cấu-hình-an-toàn)=
## Mẫu cấu hình an toàn

`.serena/project.yml` tối thiểu:

```yaml
project_name: "project-a"
languages:
  - typescript
ignore_all_files_in_gitignore: true
ignored_paths: []
read_only: false
additional_workspace_folders: []
```

Global `serena_config.yml` nên giữ:

```yaml
project_serena_folder_location: "$projectDir/.serena"
```

MCP command nên dùng:

```bash
serena start-mcp-server --context codex --project /path/to/project-a
```

Hoặc, nếu agent luôn được mở trong đúng repo:

```bash
serena start-mcp-server --context codex --project-from-cwd
```

(tóm-tắt-nhanh)=
## Tóm tắt nhanh

- Mỗi repo một `.serena/project.yml`.
- Mỗi repo một `.serena/cache/`.
- Mỗi project nên có `project_name` duy nhất.
- Dùng `--project <path>` để chắc chắn không activate nhầm.
- Dùng `--project-from-cwd` chỉ khi agent được mở từ đúng repo.
- Không trỏ nhiều project vào cùng một `project_serena_folder_location`.
- Không dùng chung một Serena HTTP instance cho nhiều project độc lập.
