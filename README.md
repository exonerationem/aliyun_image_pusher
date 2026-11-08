# Aliyun Image Pusher

> 基于 GitHub Actions 的镜像中转工具（Fork 优化版）

把 Docker Hub、`ghcr.io` 等公共镜像**自动同步到阿里云容器镜像服务（ACR）**，解决国内拉取慢、拉取失败的问题。

同步完全在 GitHub 托管 runner 上完成：用 `skopeo` 在源 registry 与目标 registry 之间**直传**，不经过 Docker daemon、不落 Docker 存储、不需要自己的服务器或 NAS。当前 `images.txt` 维护 71 个镜像，目标为 ACR **公开仓库**。

## ✨ 当前特性

| 特性 | 说明 |
| --- | --- |
| 仓库直传 | `skopeo copy` 端到端直传 + `--dest-creds` 认证，不再 `docker pull / tag / push` |
| 增量同步 | 先并行（`xargs -P 4`）比对源与目标，仅同步需要更新的镜像 |
| 磁盘保护 | 只删除 runner 预装工具链腾空间、**不重分区**；每个镜像同步前检查根盘水位，≥90% 主动熔断 |
| zstd 自适应 | ACR 个人版不接受 zstd 层，按源清单 `layers[].mediaType` 自动识别并强制转 gzip，不硬编码镜像名 |
| 凭据隔离 | Docker Hub PAT 只发给 Docker Hub，`ghcr.io` 等第三方源保持匿名 |
| 固定环境 | `runs-on: ubuntu-24.04` + `actions/checkout@v7`，规避系统镜像漂移与 Node 20 弃用警告 |
| 通知 | 可选飞书 / 企微 Webhook，含新增 / 失败镜像清单与日志跳转链接 |

## 🚀 快速开始

### 1. 配置 Secrets

在 `Settings → Secrets and variables → Actions → New repository secret` 中添加：

| 名称 | 必填 | 说明 |
| --- | --- | --- |
| `ALIYUN_REGISTRY` | ✅ | ACR 地址，如 `registry.cn-beijing.aliyuncs.com` |
| `ALIYUN_NAME_SPACE` | ✅ | 命名空间，如 `wangboyu-images` |
| `ALIYUN_REGISTRY_USER` | ✅ | ACR 用户名 |
| `ALIYUN_REGISTRY_PASSWORD` | ✅ | ACR 访问令牌（临时密码，非账号登录密码） |
| `DOCKERHUB_USER` | 可选 | Docker Hub 用户名 |
| `DOCKERHUB_TOKEN` | 可选 | Docker Hub **Personal Access Token**，建议 `Repo Read-only` + 设置到期日 |
| `WEBHOOK_URL` | 可选 | 飞书 / 企微机器人 Webhook 地址 |

不配 Docker Hub 凭据也能运行，但匿名拉取容易撞 Docker Hub 限流。两条 PAT 建议成对配置；工作流会按 registry 白名单分发凭据，**不会**把它发给 `ghcr.io` 等第三方源。

> ⚠️ PAT 只写进 Secrets，不要写进 `images.txt`、`docker.yaml` 或提交记录；GitHub 会在日志中自动打码。

### 2. 编辑镜像列表

修改 `images.txt`，每行一个镜像，支持 `#` 注释、第三方 registry、以及指定架构：

```text
# 官方镜像
mariadb
postgres:17-alpine

# Docker Hub 组织镜像
jxxghp/moviepilot-v3
ponphil/litepan:beta

# 第三方 registry（匿名访问）
ghcr.io/homarr-labs/homarr

# 指定架构：--platform 必须写在行首，镜像名放在最后
--platform=linux/arm64 linuxserver/transmission
```

说明：

- 不写 tag 时目标名也不带 `:latest` 字面，Docker 会按约定解析为 `latest`。
- `@sha256:` 摘要会被去掉，以 tag 为准。
- **`--platform` 必须写在镜像名之前**。脚本用 `awk '{print $NF}'` 取镜像名，若写成 `nginx:latest --platform=linux/arm64`，镜像名会被误解析成 `--platform=linux/arm64`，整行同步失败。

### 3. 触发同步

| 方式 | 触发条件 | 时间 |
| --- | --- | --- |
| 定时 | `schedule: '0 10,22 * * *'` | 每天 **UTC 10:00 / 22:00**（北京 **18:00 / 06:00**） |
| 自动 | push 到 `main` 且**只有 `images.txt` 变更** | 提交后立即 |
| 手动 | `Actions → Docker Image Sync to Aliyun → Run workflow` | 随时 |

⚠️ **修改 `.github/workflows/docker.yaml` 本身不会触发 CI**（`on.push.paths` 只含 `images.txt`）。改完工作流必须手动 `Run workflow` 验证，否则会出现「提交完没反应」。

同一时间只会跑一个同步任务（`concurrency: aliyun-image-sync`，`cancel-in-progress: false`）：定时任务运行中手动触发会**排队**，不会打断正在跑的任务。

## 📝 命名转换规则

目标地址统一为：

```text
$ALIYUN_REGISTRY/$ALIYUN_NAME_SPACE/<镜像名:tag>
```

**取源镜像最后一段**（最后一个 `/` 之后的部分），不带组织名前缀：

| 源镜像（`images.txt`） | ACR 目标 |
| --- | --- |
| `jxxghp/moviepilot-v3` | `registry.cn-beijing.aliyuncs.com/wangboyu-images/moviepilot-v3` |
| `ponphil/litepan:beta` | `.../litepan:beta` |
| `postgres:17-alpine` | `.../postgres:17-alpine` |
| `mariadb` | `.../mariadb` |
| `ghcr.io/homarr-labs/homarr` | `.../homarr` |

两个例外，都会加前缀：

- **跨命名空间同名**：两个不同组织出现同名镜像时，自动加 `<命名空间>_` 前缀避免覆盖（当前清单检测结果为 0 个重名）。
- **指定平台**：`--platform=linux/arm64` 会加 `linux_arm64_` 前缀。

> 早期 README 把规则描述为「`组织名_镜像名`」，与实际代码不符，已在本版更正。

## ⚙️ 同步判定与工作原理

1. **预处理**：扫描 `images.txt`，解析命名空间、平台、目标名，检测重名冲突。
2. **并行检查**（4 并发）：用 `skopeo inspect` 读源与目标清单。
   - 目标不存在 → 需要同步
   - **优先比较 `Created` 创建时间**：源比目标新 → 同步；相同或更旧 → 跳过
   - 时间解析失败时回退比较 digest，再回退比较镜像大小
3. **串行同步**：`skopeo copy --override-os linux --override-arch <arch> --dest-creds …`，重试 3 次、单镜像超时 900 秒；每轮无论成败都清理临时 blob。

**为什么用创建时间而不是 digest？** 同步时会从多架构 index 中只抽取 `linux/amd64` 子镜像；且 ACR 上的历史镜像经旧版 `docker pull/tag/push` 推送后，manifest 会被 Docker daemon 重写成 Docker schema2（OCI ↔ schema2、`zstd` ↔ `gzip` 都会改变层 digest）。因此**源与目标的 manifest digest 天然不相等**，直接比较会导致每轮全量重推。`Created` 来自 config blob，跨 registry 转换保持不变，是这里可用的稳定判据。

代价：若上游对同一 tag 重推但固定了 `Created`（可复现构建 / `SOURCE_DATE_EPOCH`），会漏同步。可定期手动强制重推，或改用比较 `config.digest` 的方案（`rootfs.diff_ids` 同样不受压缩格式影响）。

## 🧩 zstd 压缩层处理

ACR 个人版上传 zstd 层会返回 `unknown: blob type invalid`。上游 `diygod/rsshub` 等镜像使用 `tar+zstd` 层；旧版 `docker pull/tag/push` 会由 Docker daemon 顺带重压成 gzip，改成 skopeo 直传后这层隐式转换消失，因此需要显式处理。

工作流按源清单的 `layers[].mediaType` **自动检测**：仅当源含 zstd 层时，才拉取 `quay.io/skopeo/stable:v1.22.3` 并加 `--dest-force-compress-format --dest-compress-format gzip --format v2s2`；其余镜像仍走系统自带的 skopeo 1.13.3 原路径。转 gzip 不改变 `config.digest` / `Created` / `rootfs.diff_ids`，因此不会破坏上面的增量判据。

> 系统 apt 版 skopeo 1.13.3 的 `--dest-compress-format` 不会重压已压缩层，`--dest-force-compress-format` 要到 1.21+ 才提供，所以这一步必须用新版。若 `quay.io` 不可达会打印警告并退回原路径，仅影响 zstd 镜像。

## 🔔 通知预览

配置 `WEBHOOK_URL` 后，任务结束（`if: always()`）发送精简文本卡片，按结果切换标题，并附新增 / 失败镜像清单与 GitHub 日志链接：

> ## **Docker镜像同步：成功**
>
> ✅ 本次新增同步镜像：2 个
> ✨ moviepilot-v3
> ✨ litepan:beta

![通知预览](https://github.com/user-attachments/assets/36e35447-31a4-4e3e-85d4-ca9005d32424)

未配置时打印 `⚠️ 未配置 Webhook URL，跳过通知`，不影响同步。

## ⚠️ 已知限制

- **「假绿」**：单个镜像失败只累加失败计数、不会 `exit 1`，job 结论仍可能是 `success`。判断真实结果请看日志末尾的 `成功: N, 跳过: N, 失败: N` 汇总行。
- **只同步单架构**：默认 `linux/amd64`；需要 arm64 请在 `images.txt` 用 `--platform` 逐行指定（注意写在行首）。
- **失败重试有上限**：3 次失败后跳过该镜像，等下一轮再试。
- **额外网络依赖**：仅 zstd 镜像会拉 `quay.io/skopeo/stable:v1.22.3`。
- **定时注释有误**：`docker.yaml` 里那行注释写的是「UTC 00:00 / 北京 08:00」，实际 cron 为 `0 10,22 * * *`，即 UTC 10:00 / 22:00（北京 18:00 / 06:00）。以本 README 为准。

## 📅 更新日志

见 [CHANGELOG.md](CHANGELOG.md)。

## 📄 许可证

[MIT](LICENSE)
