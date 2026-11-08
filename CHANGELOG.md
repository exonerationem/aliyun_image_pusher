# 更新日志 (Changelog)

## [v2.1.0] - 2026-11-08 (当前版本)

> 通知通道改为 **Server酱 微信推送**，移除从未配置过的飞书 / 企微 Webhook。同步逻辑一行未动。
> **已合并**：PR #3 squash 合并为 `7c7d67a`（2026-11-08）。

### 🚀 新增 (Added)
- **Server酱 通知**：任务结束（`if: always()`）发送一条 Markdown 消息，**每次运行都推**，标题按 job 异常 / 有失败 / 有新增 / 全跳过四种情况动态切换；正文含新增与失败镜像清单、统计、时间戳和可点击的 GitHub 日志链接。
- **通知正文精简**：`SYNC_RESULT` 只把 `✨` 新增与 `🚨` 失败转成列表项，`⏭️` 跳过项和占位文案一律过滤；过滤用 `LC_ALL=C` 按字节白名单匹配，避免 4 字节 emoji 在多字节 locale 下漏匹配。完整明细仍留在 `SYNC_RESULT` 供日志排查，统计行照旧显示跳过数；全跳过时正文清单为空。
- **推送明细可核对**：通知步骤在 Action 日志打印「📋 通知明细（仅新增 / 失败）」，不打印 SendKey / URL。
- **Turbo / Server酱³ 自动适配**：按 `SERVERCHAN_SENDKEY` 前缀选择 `https://sctapi.ftqq.com/{key}.send` 或 `https://{uid}.push.ft07.com/send/{key}.send`，uid 从 `sctp<uid>t…` 解析并做数字校验。
- **通知成功判定**：以返回 JSON 的 `code == 0` 为准。旧飞书逻辑只看 curl 退出码，HTTP 200 但业务失败会被误判为「发送成功」；现在失败重试 3 次后只发 `::warning::`，不影响 job 结论。
- **job 异常告警**：新增 `JOB_STATUS` 判断，工作流非 `success` 时推送「🚨 同步任务异常终止」，补齐同步步骤崩溃时的告警盲区。

### 🔧 变更 (Changed)
- **移除飞书 / 企微渠道**：删除 job 级 `WEBHOOK_URL` 与 `msg_type: text` payload 构造代码。该 Secret 从未配置，无实际影响。
- **凭据最小暴露**：`SERVERCHAN_SENDKEY` 只放在通知 step 级 `env`，不在 job 级 env 或 shell 脚本里做 `${{ }}` 插值；SendKey 不打印、curl 错误输出丢弃。
- **正文改 Markdown**：用 `--data-urlencode` 发送 `title` / `desp`，正确处理中文、emoji、换行与链接；`SYNC_RESULT` 中仅 `✨` / `🚨` 转成列表项。不再按字节截断标题（避免切坏中文）。
- 计数变量改用 shell 环境变量加 `${X:-0}` 兜底，去掉 `${{ env.X }}` 插值，减少脚本源码回显与注入面。

### 🐛 修复 (Fixed)
- 修正 `docker.yaml` 的 cron 注释：实际为 UTC 10:00 / 22:00（北京 18:00 / 06:00），旧注释误写 UTC 00:00 / 北京 08:00。

### 📝 文档 (Docs)
- README：Secrets 表用 `SERVERCHAN_SENDKEY` 替换 `WEBHOOK_URL`；通知章节改写为 Server酱（两种 SendKey、Markdown 正文、5 条/天额度）；移除已修正的「定时注释有误」已知限制。
- README（合并后补充）：明确正文只列新增 / 失败、跳过项仅在统计体现；新增「每日失败巡检」章节，说明自动化巡检如何兜住「假绿」。
- README 通知预览改用**真实 Server酱 截图** `doc/serverchan-notify.jpg`（1190×1105，103 KB），替换重写前的旧 Webhook 通知外链图与纯文字示例；图片随仓库走，不再依赖 GitHub `user-attachments` 附件。
- 巡检 automation prompt 同步更新：移除已废弃的「飞书 Webhook 通知步骤」排查项，改为 Server酱；删除 rsshub `blob type invalid` 的「长期已知问题」标注（PR #2 已修复，复发按回归处理）；补充通知明细日志的核对方式。

### ✅ 验证记录
- 本地以 curl / jq stub 模拟 6 种场景全部通过：Turbo 与 ³ 地址生成、中文标题完整、日志链接存在、`code == 0` 成功、业务失败重试 3 次、未配置 / 格式异常时安全跳过、job 失败标题。
- YAML 解析通过（8 个 step），全部 `run` 块 `bash -n` 通过。
- 过滤逻辑本地脚本 3 场景通过：混合输入只留 1 条 `✨` + 1 条 `🚨`、全跳过输出为空、占位文案被过滤。
- 分支 dispatch **`42463572879`** = `success`（6m50s，9 步骤全绿）：`成功: 0, 跳过: 71, 失败: 0`，日志明细输出「（本次无新增或失败项）」，`✅ Server酱 通知发送成功 (第 1 次)`，真实微信到达且正文无跳过清单。
- PR #3 合并后 main 与分支 `docker.yaml` 逐行一致（`git diff` 为空）。

### ⚠️ 已知限制
- Server酱 免费版 5 条/天；每天 2 次定时约占 2 条额度。
- 未配置 `SERVERCHAN_SENDKEY` 时静默跳过微信通知，不影响同步。
- 「假绿」仍在：单个镜像失败不 `exit 1`，判断真实结果请看日志汇总行（每日巡检已自动检查该行）。
- 全跳过时通知正文的镜像清单为空，属预期行为；跳过数量看统计行。

---

## [v2.0.0] - 2026-11-08

> **同步引擎重构 + 稳定性修复。** 本版把镜像传输从 `docker pull/tag/push` 换成 `skopeo` 仓库直传，并修复了一个会让整个 job 崩溃的磁盘问题；同时补齐 zstd 层兼容、凭据隔离与 runner 环境固定。行为上有两处变化见「变更」。

### 🐛 修复 (Fixed)

- **修复磁盘写满导致 runner 崩溃**：删除 `easimon/maximize-build-space`。该 action 的 `root-reserve-mb: 2048` 会把根分区 `/` 压到 2 GB，而 runner 的 `_diag`、`$RUNNER_TEMP`、`$GITHUB_WORKSPACE` 都在 `/` 上，最终连诊断日志都写不进去，抛出 `System.IO.IOException: No space left on device` 并使 Worker 进程未捕获异常退出。现在只删除预装工具链（dotnet / android / ghc / CodeQL / boost），**不再重分区**。
- **修复 rsshub 同步失败 `unknown: blob type invalid`**：ACR 个人版不接受 zstd 压缩层。旧版经 Docker daemon 推送时会顺带重压成 gzip，改成 skopeo 直传后这层隐式转换消失，`diygod/rsshub` 与 `:chromium-bundled` 因此失败。现在按源清单 `layers[].mediaType` 自动检测 zstd 源，仅对这类镜像用 `quay.io/skopeo/stable:v1.22.3` 加 `--dest-force-compress-format --dest-compress-format gzip --format v2s2` 转换；**按层格式识别，不硬编码镜像名**。
- **修复 Docker Hub 凭据外泄风险**：新增 `hub_creds()` 按 registry 白名单分发，PAT 只发给 Docker Hub（无域名前缀 / `docker.io` / `index.docker.io` / `registry-1.docker.io`）；`ghcr.io`、`quay.io` 等第三方源保持匿名。此前无条件 `--src-creds` 会把凭据发给所有源。
- **修复通知步骤空变量报错**：`SUCCESS_COUNT` / `FAILURE_COUNT` / `SKIP_COUNT` 加 `|| 0` 兜底，同步步骤中途崩溃时不再出现 `integer expression expected`。
- **修复并行检查函数的环境变量继承**：`hub_creds` 改为 `export -f`，`xargs -P 4` 子进程可正常调用。

### 🚀 新增 (Added)

- **磁盘水位熔断**：每个镜像同步前执行 `df -P /`，根盘使用率 ≥90% 时打印 `::error::` 并主动 `exit 1`，避免 runner 再次因写日志失败而整体崩溃。
- **并发保护**：`concurrency: aliyun-image-sync` + `cancel-in-progress: false`，定时与手动任务不会互相打断，后者排队等待。
- **Docker Hub 认证**：可选 `DOCKERHUB_USER` / `DOCKERHUB_TOKEN`（Personal Access Token），避免匿名拉取被限流；未配置时自动退化为匿名。
- **运行器环境可观测性**：预检输出 `ImageOS` / `ImageVersion`、`lsb_release -ds`、Docker 版本、`df -h` / `df -i` 及 skopeo 临时目录所在磁盘。
- **临时目录管理**：临时 blob 优先落到独立设备 `/mnt`，否则回退 `$RUNNER_TEMP/skopeo-tmp`；每轮无论成败都清理，作业结束再统一 `docker system prune -af`。

### 🔧 变更 (Changed)

- **同步引擎改为 `skopeo copy` 仓库直传**：用 `--dest-creds` 认证，不再 `docker pull` → `docker tag` → `docker push`，不落 Docker 存储、不依赖 Docker daemon 做中转，也不需要 `docker login`。这是根治磁盘占用的主因。
- **运行器固定为 `ubuntu-24.04`**、`actions/checkout@v7`：规避 `ubuntu-latest` 灰度迁移到 Ubuntu 26.04 期间的系统漂移，同时消除 Node.js 20 弃用警告。
- **重试与超时**：`--retry-times 3`、单镜像 `timeout 900`；失败镜像不再残留完整镜像层。
- **保留创建时间作为增量判据**：曾评估改为比较 digest，实证发现 ACR 上目标 manifest 会因单架构抽取、OCI ↔ schema2 转换、zstd ↔ gzip 重压缩而变化，直接比较会每轮全量重推，故维持 `Created` 时间比较（digest / 镜像大小仍作为解析失败时的兜底）。
- **仅同步单架构**：`--override-os linux --override-arch <arch>`，默认 amd64；可用 `--platform` 逐行指定其他架构（**须写在行首**）。

### 📝 文档更正 (Docs)

- 更正命名规则：实际为**取源镜像最后一段**（`ponphil/litepan:beta` → `.../litepan:beta`），旧 README 描述的「`组织名_镜像名`」与代码不符；跨命名空间同名与 `--platform` 才会加前缀。
- 更正定时时间：`0 10,22 * * *` 是 UTC 10:00 / 22:00（北京 18:00 / 06:00），旧注释写的「UTC 00:00 / 北京 08:00」有误。
- 更正目标仓库性质：ACR 目标为**公开仓库**，不是私有仓库。
- 更正 v1.2.0 中「支持 `main` 与 `dev` 双分支触发」：当前 `on.push.branches` 只有 `main`。
- 补充 Secrets 清单（新增两条可选的 Docker Hub 凭据）、`images.txt` 写法、zstd 处理原理与已知限制。

### ✅ 验证记录

| 项目 | 结果 |
| --- | --- |
| PR #1（磁盘 / skopeo / runner 固定） | 已合并 `bcce879`；分支验证 `42445192616`、`42447109283` 均 success |
| 推送链路自证 | run `42450806602`，`hello-world:latest` 写入 ACR 成功（`成功: 1`），测试镜像随后移除 |
| zstd 转换测试 tag | run `42457614058`，两个测试 tag 均 9 层 gzip / 0 层 zstd，正式 tag 未被覆盖 |
| PR #2（zstd → gzip） | 已合并 `09e89db`；分支端到端 `42458090082` = `成功: 4, 跳过: 67, 失败: 0` |
| main 正式验证 | run `42458913854` = success，`成功: 0, 跳过: 71, 失败: 0`（幂等，无全量重推） |

### ⚠️ 已知限制

- **「假绿」**：单个镜像失败只累加计数、不 `exit 1`，job 结论仍可能是 `success`；需看日志末尾的 `成功/跳过/失败` 汇总行。
- `WEBHOOK_URL` 未配置时通知步骤会跳过，不影响同步。（该通道已在 v2.1.0 移除，改为 Server酱）
- zstd 镜像额外依赖 `quay.io`；不可达时退回原路径，仅这类镜像会失败。
- 创建时间判据在上游「同 tag 重推但固定 `Created`」时会漏同步。

---

> 以下为历史记录。其中命名规则、双分支触发等描述已在 v2.0.0 更正。

## [v1.2.0] - 2026-01-27

### 🚀 新增 (Added)
- **运维自动化告警**：集成飞书 Webhook 通知。任务执行后自动发送精简的富文本通知卡片。
- **智能内容分发**：根据任务结果（成功/失败/无更新）动态切换通知内容，拒绝信息骚扰。
- **中文超链接支持**：通知内直接集成"查看详情"中文短链接，点击即跳转至 GitHub Action 执行日志，不再显示冗长的 URL。

### 🔧 优化 (Changed)
- **JSON 排版修复**：重构了 Payload 构造逻辑，彻底消除飞书通知中出现的 JSON 源码杂质，实现极致简洁的视觉效果。
- **多分支触发**：优化工作流配置，支持 `main` 与 `dev` 双分支触发，便于开发与生产环境隔离调试。
- **文档重构**：更新 README.md，增加详细的命名转换规则说明及 v1.2.0 新特性介绍。

---

## [v1.1.0] - 2026-01-23 (初始发布)

### 🚀 新增 (Added)
- **核心同步功能**：利用 GitHub Actions 实现从 Docker Hub 到阿里云 ACR 的自动镜像转存。
- **智能命名空间隔离**：自动将镜像映射为 `组织名_镜像名` 格式，彻底解决同名覆盖痛点。
- **官方镜像标准化**：识别官方镜像并统一归类至 `library_` 命名空间。
- **增量同步机制**：集成 `skopeo` 探测，自动跳过已存在镜像，节省构建时间。
- **重型镜像支持**：通过自动清理磁盘空间，支持 GB 级别的大型镜像同步。
