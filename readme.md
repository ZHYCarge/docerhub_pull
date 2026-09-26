下面内容可直接保存为仓库根目录的 `README.md`：

````markdown
# Docker Image Mirror

通过 **GitHub Actions + Skopeo**，定时将第三方公开容器镜像同步到自己的 Docker Hub。

本项目适用于 NAS、服务器或其他只能稳定访问 Docker Hub、无法直接访问 GHCR、Quay、GitLab Container Registry 等第三方镜像仓库的环境。

---

## 功能特点

- 支持同时同步多个公开镜像
- 支持 GHCR、Quay、GitLab Container Registry 等 OCI/Docker Registry
- 使用 Skopeo 在镜像仓库之间直接复制
- 不运行镜像，不执行镜像中的任何程序
- 不通过 Dockerfile 重新构建镜像
- 复制镜像配置、Manifest 和 Layer
- 使用 `--all` 同步上游提供的全部架构
- 支持每日定时同步
- 支持在 GitHub Actions 页面手动同步
- 单个镜像失败不影响其他镜像继续执行
- 同步后自动验证目标镜像是否可以访问

---

## 工作原理

同步流程如下：

```text
公开上游镜像仓库
        │
        │  GitHub Actions 定时运行
        ▼
      Skopeo
        │
        │  直接复制镜像
        ▼
你的 Docker Hub 公开仓库
        │
        │  docker pull
        ▼
   NAS / 服务器
```

例如，上游镜像为：

```text
ghcr.io/example/example-app:latest
```

同步到：

```text
yourname/example-app:latest
```

NAS 最终只需要访问 Docker Hub：

```bash
docker pull yourname/example-app:latest
```

---

## Skopeo 原样同步说明

本项目使用以下命令同步镜像：

```bash
skopeo copy --all \
  docker://SOURCE_IMAGE \
  docker://TARGET_IMAGE
```

同步内容包括：

- 镜像文件系统 Layer
- 镜像配置
- 镜像 Manifest
- 多架构 Manifest List 或 OCI Index
- 上游镜像已有的 Label
- 上游镜像已有的环境变量及启动参数

同步过程中不会：

- 启动或运行上游镜像
- 执行镜像内的脚本或二进制文件
- 使用 Dockerfile 重新构建镜像
- 修改镜像文件系统内容
- 给镜像添加自定义 Label
- 自动更新 Docker Hub Overview
- 自动删除旧 Tag

> “原样同步”表示不重新构建、不修改镜像内容。由于不同镜像仓库对媒体类型、Manifest 或 OCI 索引的处理方式可能不同，目标镜像的顶层 Digest 不保证在所有情况下都与上游完全一致。

---

## 项目结构

```text
docker-image-mirror/
├── .github/
│   └── workflows/
│       └── sync-images.yml
├── images.json
└── README.md
```

文件用途：

| 文件 | 用途 |
|---|---|
| `.github/workflows/sync-images.yml` | GitHub Actions 定时同步工作流 |
| `images.json` | 需要同步的镜像映射清单 |
| `README.md` | 项目使用说明 |

---

## 前置条件

开始配置前，需要准备：

1. 一个 GitHub 账号
2. 一个 GitHub 仓库
3. 一个 Docker Hub 账号或组织
4. 一个具有 Docker Hub 写入权限的 Access Token
5. 提前创建好目标 Docker Hub 公开仓库

本项目中的源镜像必须允许公开匿名拉取。

如果源镜像是私有镜像，需要额外配置源仓库身份认证，本 README 默认不包含私有源镜像方案。

---

## 第一步：创建 GitHub 仓库

创建一个 GitHub 仓库，例如：

```text
docker-image-mirror
```

仓库可以是公开仓库，也可以是私有仓库。

注意：

- 不要把 Docker Hub Token 写进代码或 `images.json`
- 不要把包含敏感信息的配置提交到 GitHub
- Docker Hub Token 应保存在 GitHub Actions Secrets 中

---

## 第二步：创建 Docker Hub Access Token

登录 Docker Hub，进入账户安全设置并创建 Personal Access Token。

Token 至少需要具备：

```text
Read & Write
```

建议：

- 为 GitHub Actions 单独创建 Token
- 不要直接使用 Docker Hub 登录密码
- Token 名称可以设置为 `github-actions-mirror`
- 如果 Token 泄露，应立即撤销并重新创建

---

## 第三步：创建 Docker Hub 目标仓库

建议每个应用使用独立的 Docker Hub 仓库。

例如需要同步：

```text
ghcr.io/immich-app/immich-server:release
ghcr.io/immich-app/immich-machine-learning:release
quay.io/prometheus/node-exporter:latest
```

可以在 Docker Hub 中分别创建：

```text
immich-server
immich-machine-learning
node-exporter
```

如果 Docker Hub 用户名是 `yourname`，最终镜像为：

```text
yourname/immich-server:release
yourname/immich-machine-learning:release
yourname/node-exporter:latest
```

建议将这些目标仓库设置为：

```text
Public
```

虽然部分情况下首次推送可能自动创建仓库，但为了避免权限或平台策略问题，建议提前在 Docker Hub 中手动创建。

---

## 第四步：配置 GitHub Actions Secrets

打开 GitHub 仓库，依次进入：

```text
Settings
→ Secrets and variables
→ Actions
→ New repository secret
```

添加以下两个 Secret：

| Secret 名称 | 内容 |
|---|---|
| `DOCKERHUB_USERNAME` | Docker Hub 用户名或组织名 |
| `DOCKERHUB_TOKEN` | Docker Hub Personal Access Token |

示例：

```text
DOCKERHUB_USERNAME=yourname
DOCKERHUB_TOKEN=dckr_pat_xxxxxxxxxxxxxxxxx
```

请不要创建明文配置文件保存这些值，也不要将 Token 提交到 GitHub。

公开源镜像不需要配置：

```text
SOURCE_REGISTRY_USERNAME
SOURCE_REGISTRY_TOKEN
```

---

## 第五步：配置镜像清单

在仓库根目录创建：

```text
images.json
```

示例内容：

```json
[
  {
    "source": "ghcr.io/immich-app/immich-server:release",
    "target": "immich-server:release",
    "description": "Immich Server 官方公开镜像的自动同步副本"
  },
  {
    "source": "ghcr.io/immich-app/immich-machine-learning:release",
    "target": "immich-machine-learning:release",
    "description": "Immich Machine Learning 官方公开镜像的自动同步副本"
  },
  {
    "source": "quay.io/prometheus/node-exporter:latest",
    "target": "node-exporter:latest",
    "description": "Prometheus Node Exporter 官方公开镜像的自动同步副本"
  }
]
```

### 字段说明

| 字段 | 是否必填 | 说明 |
|---|---:|---|
| `source` | 是 | 上游完整镜像地址，建议包含明确的 Tag |
| `target` | 是 | Docker Hub 目标仓库名称和 Tag，不包含用户名 |
| `description` | 否 | 镜像说明，仅显示在 GitHub Actions 日志中 |

### 名称映射示例

假设：

```text
DOCKERHUB_USERNAME=yourname
```

配置：

```json
{
  "source": "ghcr.io/example/app:v1.2.0",
  "target": "my-app:v1.2.0",
  "description": "Example App 的自动同步镜像"
}
```

最终同步为：

```text
ghcr.io/example/app:v1.2.0
```

到：

```text
yourname/my-app:v1.2.0
```

### 添加多个 Tag

如果同一个应用需要同时同步多个 Tag，应分别配置：

```json
[
  {
    "source": "ghcr.io/example/app:latest",
    "target": "my-app:latest",
    "description": "Example App 最新版本"
  },
  {
    "source": "ghcr.io/example/app:stable",
    "target": "my-app:stable",
    "description": "Example App 稳定版本"
  },
  {
    "source": "ghcr.io/example/app:v2.5.1",
    "target": "my-app:v2.5.1",
    "description": "Example App 固定版本"
  }
]
```

### 配置注意事项

`source` 应填写完整仓库地址：

```text
ghcr.io/owner/image:tag
quay.io/owner/image:tag
registry.gitlab.com/group/project/image:tag
```

`target` 只填写仓库名称和 Tag：

```text
image-name:tag
```

不要在 `target` 中添加用户名：

```text
# 正确
"target": "example-app:latest"

# 错误
"target": "yourname/example-app:latest"
```

Docker Hub 用户名会由工作流自动添加。

建议始终明确填写 Tag，不要省略：

```text
# 推荐
ghcr.io/example/app:latest

# 不推荐
ghcr.io/example/app
```

---

## 第六步：创建 GitHub Actions 工作流

创建文件：

```text
.github/workflows/sync-images.yml
```

完整内容如下：

```yaml
name: Sync Container Images

on:
  # 允许在 GitHub Actions 页面手动执行
  workflow_dispatch:

  # 定时执行
  schedule:
    # GitHub Actions 使用 UTC 时间
    # UTC 19:17 = 北京时间次日 03:17
    - cron: "17 19 * * *"

permissions:
  contents: read

concurrency:
  # 防止手动任务与定时任务同时运行
  group: container-image-sync
  cancel-in-progress: false

jobs:
  prepare:
    name: Prepare sync matrix
    runs-on: ubuntu-latest

    outputs:
      matrix: ${{ steps.matrix.outputs.value }}

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Validate images.json
        run: |
          set -euo pipefail

          # 检查 JSON 语法
          jq empty images.json

          # 检查配置必须为非空数组，并包含 source 和 target
          jq -e '
            type == "array" and
            length > 0 and
            all(
              .[];
              type == "object" and
              (.source | type == "string" and length > 0) and
              (.target | type == "string" and length > 0)
            )
          ' images.json >/dev/null

          # target 只允许填写仓库名和 Tag，不允许携带用户名路径
          jq -e '
            all(
              .[];
              (.target | contains("/") | not)
            )
          ' images.json >/dev/null

      - name: Generate job matrix
        id: matrix
        run: |
          echo "value=$(jq -c '{include: .}' images.json)" >> "$GITHUB_OUTPUT"

  sync:
    name: Sync ${{ matrix.target }}
    needs: prepare
    runs-on: ubuntu-latest

    strategy:
      # 单个镜像失败时，继续同步其他镜像
      fail-fast: false

      # 限制并发数量，降低触发仓库限流的可能性
      max-parallel: 3

      matrix: ${{ fromJSON(needs.prepare.outputs.matrix) }}

    steps:
      - name: Install Skopeo
        run: |
          set -euo pipefail
          sudo apt-get update
          sudo apt-get install --yes skopeo

      - name: Show sync information
        env:
          SOURCE_IMAGE: ${{ matrix.source }}
          TARGET_IMAGE: ${{ secrets.DOCKERHUB_USERNAME }}/${{ matrix.target }}
          DESCRIPTION: ${{ matrix.description }}
        run: |
          echo "Source:      ${SOURCE_IMAGE}"
          echo "Target:      ${TARGET_IMAGE}"
          echo "Description: ${DESCRIPTION:-No description}"

      - name: Copy all image architectures
        env:
          SOURCE_IMAGE: ${{ matrix.source }}
          TARGET_IMAGE: ${{ secrets.DOCKERHUB_USERNAME }}/${{ matrix.target }}
          DOCKERHUB_USERNAME: ${{ secrets.DOCKERHUB_USERNAME }}
          DOCKERHUB_TOKEN: ${{ secrets.DOCKERHUB_TOKEN }}
        run: |
          set -euo pipefail

          skopeo copy \
            --all \
            --retry-times 3 \
            --dest-creds "${DOCKERHUB_USERNAME}:${DOCKERHUB_TOKEN}" \
            "docker://${SOURCE_IMAGE}" \
            "docker://${TARGET_IMAGE}"

      - name: Verify target image
        env:
          TARGET_IMAGE: ${{ secrets.DOCKERHUB_USERNAME }}/${{ matrix.target }}
        run: |
          set -euo pipefail

          skopeo inspect "docker://${TARGET_IMAGE}" >/dev/null
          echo "Successfully verified: ${TARGET_IMAGE}"
```

---

## 定时同步设置

GitHub Actions 的 cron 表达式使用 UTC，而不是北京时间。

当前配置：

```yaml
- cron: "17 19 * * *"
```

表示：

```text
每天 UTC 19:17
每天北京时间次日 03:17
```

常用配置：

| 北京时间 | Cron 表达式 |
|---|---|
| 每天 02:00 | `0 18 * * *` |
| 每天 03:00 | `0 19 * * *` |
| 每天 03:17 | `17 19 * * *` |
| 每天 06:00 | `0 22 * * *` |
| 每天 12:00 | `0 4 * * *` |
| 每 6 小时 | `17 */6 * * *` |
| 每周日 03:00 | `0 19 * * 6` |

建议避开整点，以降低 GitHub Actions 定时任务排队延迟的概率，例如使用：

```yaml
- cron: "17 19 * * *"
```

GitHub Actions 的定时任务并不保证精确到分钟，实际开始时间可能存在延迟。

---

## 手动执行同步

完成配置并提交文件后，可以先手动测试。

打开 GitHub 仓库：

```text
Actions
→ Sync Container Images
→ Run workflow
→ Run workflow
```

工作流会：

1. 检出仓库
2. 验证 `images.json`
3. 根据镜像列表生成任务矩阵
4. 安装 Skopeo
5. 并行同步镜像
6. 验证 Docker Hub 中的目标镜像

---

## 查看同步结果

进入：

```text
GitHub 仓库
→ Actions
→ Sync Container Images
```

绿色表示工作流执行成功。

由于配置了：

```yaml
fail-fast: false
```

当某一个镜像失败时，其他镜像仍会继续同步。

可以点开失败的镜像任务查看具体日志。

同步成功后，也可以在本地或 NAS 上测试：

```bash
docker pull yourname/example-app:latest
```

查看镜像：

```bash
docker image inspect yourname/example-app:latest
```

查看本机架构：

```bash
uname -m
```

常见对应关系：

| `uname -m` 输出 | 容器平台 |
|---|---|
| `x86_64` | `linux/amd64` |
| `aarch64` | `linux/arm64` |
| `armv7l` | `linux/arm/v7` |

---

## 多架构镜像

工作流使用：

```bash
skopeo copy --all
```

如果上游镜像提供多架构 Manifest，Skopeo 会复制上游包含的全部架构，例如：

```text
linux/amd64
linux/arm64
linux/arm/v7
```

因此：

- 不需要配置 NAS 架构
- 不需要安装 QEMU
- 不需要使用 Docker Buildx
- NAS 拉取镜像时会自动选择匹配的平台

如果上游只提供 `linux/amd64`，同步后也不会自动生成 `linux/arm64` 镜像。

Skopeo 只能复制上游已经发布的架构，不能进行跨架构编译或转换。

可以在支持 Docker Manifest 的环境中检查：

```bash
docker buildx imagetools inspect yourname/example-app:latest
```

---

## 在 NAS 上使用

### Docker CLI

```bash
docker pull yourname/example-app:latest
```

启动容器：

```bash
docker run -d \
  --name example-app \
  --restart unless-stopped \
  yourname/example-app:latest
```

实际端口、目录、环境变量等参数请以上游项目文档为准。

### Docker Compose

将上游镜像地址替换为 Docker Hub 镜像地址：

```yaml
services:
  app:
    image: yourname/example-app:latest
    container_name: example-app
    restart: unless-stopped
```

启动：

```bash
docker compose pull
docker compose up -d
```

更新镜像：

```bash
docker compose pull
docker compose up -d
```

### 旧版本 Docker Compose

部分 NAS 可能仍使用：

```bash
docker-compose pull
docker-compose up -d
```

---

## Docker Hub 仓库说明建议

Skopeo 原样同步不会修改镜像配置，因此不能向镜像内添加自定义说明 Label。

建议手动在每个 Docker Hub 仓库中设置 Description 和 Overview。

### Short Description

```text
上游公开容器镜像的自动同步副本，仅用于便捷拉取。
```

### Overview

```markdown
# 镜像说明

这是上游公开容器镜像的自动同步副本，主要用于只能访问 Docker Hub 的 NAS 或服务器。

## 同步方式

- 使用 GitHub Actions 定时同步
- 使用 Skopeo 直接复制镜像
- 不使用 Dockerfile 重新构建
- 不运行镜像内容
- 同步上游提供的多架构 Manifest

## 注意事项

- 软件版权、商标及许可证归上游项目所有
- 本仓库不是上游项目的官方 Docker Hub 仓库
- 镜像同步可能存在时间延迟
- 使用前请查看上游项目文档及许可证
- 本镜像未经过额外安全审计
```

---

## 固定版本建议

不建议在重要环境中只依赖：

```text
latest
```

推荐同步并使用明确版本：

```json
{
  "source": "ghcr.io/example/app:v2.5.1",
  "target": "example-app:v2.5.1",
  "description": "Example App v2.5.1"
}
```

Docker Compose：

```yaml
services:
  app:
    image: yourname/example-app:v2.5.1
```

这样可以避免上游更新 `latest` 后出现不兼容变化。

如需同时跟踪最新版本，可以同时同步固定版本和 `latest`：

```json
[
  {
    "source": "ghcr.io/example/app:v2.5.1",
    "target": "example-app:v2.5.1",
    "description": "固定版本"
  },
  {
    "source": "ghcr.io/example/app:latest",
    "target": "example-app:latest",
    "description": "最新版本"
  }
]
```

---

## 更新镜像列表

增加镜像时，只需：

1. 在 Docker Hub 创建目标公开仓库
2. 编辑 `images.json`
3. 提交修改
4. 手动运行工作流测试
5. 等待之后的定时同步

例如添加一个新镜像：

```json
{
  "source": "quay.io/example/new-app:stable",
  "target": "new-app:stable",
  "description": "New App 稳定版自动同步镜像"
}
```

请确保 JSON 格式正确：

- 对象之间使用英文逗号分隔
- 最后一个对象后面不能添加逗号
- 字符串使用英文双引号
- 文件最外层必须是数组

可以在本地验证：

```bash
jq empty images.json
```

如果没有输出且退出状态为 `0`，说明 JSON 语法正确。

---

## 删除镜像或停止同步

如果不再需要同步某个镜像：

1. 从 `images.json` 中删除对应项目
2. 提交修改
3. 根据需要在 Docker Hub 中手动删除对应 Tag 或仓库

工作流不会自动删除 Docker Hub 上已存在的镜像或 Tag。

也就是说，从 `images.json` 删除配置后：

- 后续不再同步
- Docker Hub 中已经存在的镜像仍会保留
- 如需彻底删除，应到 Docker Hub 手动操作

---

## 并发与限流

工作流默认设置：

```yaml
max-parallel: 3
```

表示最多同时同步 3 个镜像。

如果镜像数量较多，或遇到上游仓库、Docker Hub 限流，可以降低为：

```yaml
max-parallel: 1
```

如果镜像数量少且仓库限制较宽松，也可以提高：

```yaml
max-parallel: 5
```

不建议设置过高，否则可能：

- 触发上游匿名拉取限制
- 触发 Docker Hub API 或上传限制
- 增加 GitHub Actions 网络失败概率

---

## 常见问题

### 1. `unauthorized: authentication required`

可能原因：

- `DOCKERHUB_USERNAME` 配置错误
- `DOCKERHUB_TOKEN` 无效或已过期
- Token 没有写入权限
- 目标仓库属于组织，但账号没有写入权限
- Secret 名称与工作流不一致

处理方法：

1. 检查 GitHub Secrets 名称
2. 重新生成 Docker Hub Token
3. 确认 Token 具有 Read & Write 权限
4. 确认账户能够向目标仓库推送

---

### 2. `denied: requested access to the resource is denied`

可能原因：

- Docker Hub 目标仓库不存在
- 目标仓库不属于当前用户名
- 组织仓库权限不足
- `target` 名称错误
- Docker Hub 用户名大小写或拼写错误

建议提前创建目标仓库并确认权限。

---

### 3. `manifest unknown`

可能原因：

- 上游 Tag 不存在
- 镜像名称拼写错误
- 上游删除了该版本
- 镜像地址未包含正确的 Registry
- 上游镜像并非公开镜像

检查：

```bash
skopeo inspect docker://ghcr.io/example/app:tag
```

或者直接查看上游项目发布文档。

---

### 4. NAS 提示 `no matching manifest for linux/arm64`

说明上游镜像没有提供 NAS 所需的架构。

Skopeo 的 `--all` 只能复制现有架构，不能把 `amd64` 镜像转换成 `arm64`。

请：

- 检查上游是否提供 ARM 版本
- 更换支持当前 NAS 架构的 Tag
- 联系上游项目
- 自行获取源代码并进行跨架构构建

---

### 5. 定时任务没有准时执行

GitHub Actions 的定时任务可能排队，不能保证精确到分钟。

同时应确认：

- 工作流文件位于默认分支
- YAML 格式正确
- 仓库的 Actions 功能已启用
- 定时工作流没有因长期无活动而暂停
- cron 使用的是 UTC 时间

可以使用 `workflow_dispatch` 手动运行进行排查。

---

### 6. 某个镜像失败是否会影响其他镜像？

不会。

工作流使用：

```yaml
fail-fast: false
```

某个镜像失败后，其余镜像仍会继续同步。

整个工作流最终可能显示部分失败，需要在 Actions 页面查看具体失败任务。

---

### 7. 每天同步是否会重复上传所有 Layer？

Skopeo 会检查并复制镜像。目标 Registry 通常能够复用已经存在的 Layer，因此不一定每次都重新上传所有内容。

但每次同步仍会产生：

- 上游 Registry API 请求
- Manifest 查询
- Layer 存在性检查
- Docker Hub API 请求

因此不建议设置过于频繁的同步周期。

对于个人自用，一般建议：

```text
每天一次
```

或：

```text
每 6～12 小时一次
```

---

### 8. 能否同步所有 Tag？

当前配置按 `images.json` 中明确列出的 Tag 逐个同步。

默认不会自动枚举仓库中的全部 Tag，因为：

- 上游可能有大量历史版本
- 会显著增加存储和网络消耗
- 不同 Registry 的 Tag API 和认证方式可能不同
- 容易触发限流
- 可能同步不需要的测试版或旧版本

建议只同步实际使用的 Tag。

---

### 9. 能否自动修改 Docker Hub Overview？

当前工作流不会自动修改 Docker Hub 仓库页面说明。

原因是镜像推送与仓库页面元数据管理是不同操作。建议在 Docker Hub 中手动填写 Overview。

如需自动维护 Overview，需要额外调用 Docker Hub API，并配置额外逻辑和权限。

---

### 10. 能否同步私有源镜像？

可以，但需要：

- 源仓库用户名
- 源仓库 Token
- 对应 Registry 的认证配置
- 在 `skopeo copy` 中增加 `--src-creds`

本项目当前配置针对公开源镜像，不包含私有源认证。

---

## 安全建议

虽然 Skopeo 不会运行同步的镜像，但同步后的镜像仍然包含完整的上游内容。

建议：

- 只同步可信项目发布的官方镜像
- 核对上游 Registry、组织名、镜像名和 Tag
- 优先使用固定版本
- 定期查看上游安全公告
- 不要因为镜像成功同步就默认其绝对安全
- 对重要环境使用镜像扫描工具
- 定期轮换 Docker Hub Token
- 不要将 Token 写进日志、JSON 或仓库文件
- 限制 GitHub Actions 的权限
- 删除不再使用的 Token

本工作流仅申请：

```yaml
permissions:
  contents: read
```

这足以读取仓库配置，同时减少不必要的 GitHub 权限。

---

## 镜像许可证与版权

同步公开镜像不代表可以忽略上游软件许可证。

使用前应确认：

- 上游项目许可证是否允许再分发
- 镜像中是否包含额外软件或资产
- 是否存在商标使用限制
- 是否需要保留版权和许可证声明
- 是否允许将同步仓库公开发布

即使仅供个人使用，也建议保留完整的上游来源说明。

---

## 免责声明

- 本项目仅用于将上游公开容器镜像同步到 Docker Hub。
- 镜像中的软件版权、商标和许可证归对应上游项目及权利人所有。
- 本项目与上游项目不存在官方隶属、合作、担保或授权关系。
- 同步镜像可能与上游存在时间差。
- 本项目不保证镜像的可用性、完整性、安全性或适用性。
- Skopeo 同步成功不代表镜像已经通过安全审计。
- 使用者应自行核实上游来源、许可证、版本和安全风险。
- 对镜像的部署、运行和使用后果由使用者自行承担。

---

## 快速检查清单

首次部署前，请确认：

- [ ] 已创建 GitHub 仓库
- [ ] 已创建 Docker Hub Access Token
- [ ] Token 具有 Read & Write 权限
- [ ] 已配置 `DOCKERHUB_USERNAME`
- [ ] 已配置 `DOCKERHUB_TOKEN`
- [ ] 已提前创建 Docker Hub 目标公开仓库
- [ ] 已创建 `images.json`
- [ ] `images.json` 格式正确
- [ ] `target` 中没有包含 Docker Hub 用户名
- [ ] 已创建 `.github/workflows/sync-images.yml`
- [ ] 已提交所有文件到默认分支
- [ ] 已手动运行一次工作流
- [ ] 已在 NAS 上测试拉取
- [ ] 已为 Docker Hub 仓库填写来源说明
- [ ] 已检查上游许可证和镜像架构

---

## 快速使用示例

假设 Docker Hub 用户名为：

```text
yourname
```

`images.json`：

```json
[
  {
    "source": "ghcr.io/example/example-app:latest",
    "target": "example-app:latest",
    "description": "Example App 自动同步镜像"
  }
]
```

工作流同步关系：

```text
ghcr.io/example/example-app:latest
    ↓
yourname/example-app:latest
```

NAS 拉取：

```bash
docker pull yourname/example-app:latest
```

Docker Compose：

```yaml
services:
  example-app:
    image: yourname/example-app:latest
    container_name: example-app
    restart: unless-stopped
```

启动：

```bash
docker compose pull
docker compose up -d
```

后续 GitHub Actions 会按照 cron 配置定时同步上游镜像。
````
