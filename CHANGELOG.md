# 更新日志 (Changelog)

## [v2.2.2] - 2026-11-10 (当前版本)

> **每日失败巡检从 WorkBuddy 本机搬到 NAS 青龙面板**，不再依赖 PC 开机；同步、触发与通知逻辑一行未动。

### ✨ 新增 (Added)
- 青龙脚本 `github_patrol.sh`（与 `github_dispatch.sh` 并列）：每天北京时间 **07:00**（cron `0 7 * * *`，容器为 CST）检查过去 30 小时的 Actions run，判断三类异常：① 结论非 `success`，或 run 卡在 `in_progress` / `queued` 超过 1 小时；② **「假绿」**——结论 `success` 但日志汇总「成功: N, 跳过: N, 失败: N」中失败数 > 0；③ **定时丢失**——窗口内 `workflow_dispatch` / `schedule` 的 run 少于 2 条（排除 `push`）。
- 异常时用 **Server酱** 推微信 Markdown 报告（北京时间、窗口统计、异常明细、关键错误、run 链接与修复建议），并 `exit 1` 让青龙任务标红；全部正常时**静默**。Turbo / ³ 自动适配，按返回 `code == 0` 判定、失败重试 3 次。
- 巡检自身异常（缺 PAT、GitHub API 不可达、PAT 过期）同样告警，避免「巡检静默失效」。支持 `PATROL_DRY_RUN` / `PATROL_TEST` 演练，以及 `GITHUB_PROXY`、`PATROL_WINDOW_HOURS`、`PATROL_MIN_RUNS`、`PATROL_ALLOW_NO_LOG` 可调。

### 🔧 变更 (Changed)
- 青龙任务 **1373**「GitHub 镜像同步触发」（每天 06:17 / 18:17）保持不变；新增任务 **1375**「GitHub 镜像同步巡检」（`task github_patrol.sh`，每天 07:00）。
- 青龙「环境变量」新增 `SERVERCHAN_SENDKEY`（与 GitHub Secret 同值，用于巡检告警）；`GITHUB_PAT` 与触发脚本共用。未配置 SendKey 时降级为「报告只写青龙日志 + 任务标红」。
- WorkBuddy 本机 automation「镜像同步失败巡检」(`02371fae-…`) 置为 **PAUSED**（保留配置可随时恢复），避免与青龙重复巡检。
- 巡检实现由「本机 `gh` CLI + 会话内读日志分析」改为「青龙容器内 Python3 + curl + GitHub REST API」的确定性脚本：日志走 `actions/jobs/{id}/logs` 的 302 纯文本，不解压 zip；PAT 通过 `curl -K` 配置文件传入，不出现在命令行参数与日志中。

### 📝 文档 (Docs)
- README 巡检章节改写为青龙面板执行，删除「由 WorkBuddy 客户端在本机执行、关机不运行」的说明；已知限制改为「巡检依赖 NAS/青龙在线与 PAT 有效期」。

### ✅ 验证记录
- 青龙容器能力实测：`python3` / `jq` / `curl` 齐全，时区 `CST +0800`；`api.github.com` 与 `productionresultssa12.blob.core.windows.net` 均直连 200，能读到 job 日志纯文本中的汇总行。
- 正常路径实跑（北京 14:47:42）：窗口内 6 条 run 全部 `success`、日志汇总失败数均为 0 → `✅ 巡检通过`，**未推微信**（成功静默），耗时 28 秒。
- 异常路径演练（`PATROL_MIN_RUNS=99` + `PATROL_DRY_RUN=1`）：正确报出「定时丢失」，报告含窗口统计、6 条 run 明细与修复建议，且未推送、未标红。
- 修正一处误报：job 日志会回显 workflow 脚本源码，注释里也含 `blob type invalid` 等字样；关键错误扫描现只对「确实有失败」的 run 执行。
- 校验：`bash -n`、Python `py_compile`、LF 换行、无明文凭据。

### ⚠️ 已知限制
- 巡检依赖 NAS / 青龙容器在线与 `GITHUB_PAT` 有效期；PAT 过期时巡检会告警（前提是 `SERVERCHAN_SENDKEY` 已配置）。
- 青龙面板若全局开启「任务失败通知」，异常时会额外收到一条面板通知，可在通知设置里排除该任务或忽略。
- 手动 `Run workflow` 与面板触发同为 `workflow_dispatch`，无法从 API 区分，手动补跑可能掩盖一次定时丢失。

---

## [v2.2.1] - 2026-11-10

> **恢复 GitHub `schedule` 作每周一次兜底**，同时大幅简化 README。青龙面板仍是每天两次的主触发，两种定时**任选其一或同时开启均可**。

### 🔧 变更 (Changed)
- `docker.yaml` 恢复 `on.schedule`，频次由每天两次（`17 10,22 * * *`）降为**每周一次** `17 3 * * 3`（UTC 周三 03:17 = 北京周三 11:17），避开青龙 06:17 / 18:17 两个窗口与整点；只作青龙整周失效时的最低限度兜底。
- 巡检 automation prompt 同步：补充「每周一次 schedule 兜底、可能延迟或丢弃」的说明；窗口内 run 数「< 2 报警」的判定不变（多数日子应有 2 条青龙触发，兜底日 3 条）。

### 📝 文档 (Docs)
- README 定时章节改为「两种定时任选其一或同时保留均可」，删除定时延迟的实证时间戳、204 / 200 判定细节、历史迭代注记与重复表述；巡检章节明确「由 WorkBuddy 客户端在本机执行，关机 / 退出不运行」。
- 已知限制由「依赖外部调度、无 GitHub 兜底」改写为「定时兜底有限」；通知额度改为「每次运行都推，兜底那周多一条」。开头段落不再强调单一触发来源。

### ✅ 验证记录
- PyYAML 通过，`on` 含 `workflow_dispatch` / `schedule` / `push`；cron 换算正确；7 个 `run` 块 `bash -n`、`git diff --check`、围栏配对、无明文凭据均通过。
- 未实跑：纯 workflow + 文档改动不触发 CI（`on.push.paths` 只含 `images.txt`），不占微信额度；schedule 下一次触发为下一个 UTC 周三 03:17。

---

## [v2.2.0] - 2026-11-10

> **定时触发从 GitHub `schedule` 迁到 NAS 青龙面板**：仓库移除 `on.schedule`，由面板每天北京时间 06:17 / 18:17 调 REST API 创建 `workflow_dispatch` run。同步逻辑一行未动。
>
> ⚠️ 本版「移除 schedule」的做法已在 **v2.2.1** 调整为「恢复每周一次兜底」；以下记录保留当时的方案与实证，不再代表当前配置。

### 🔧 变更 (Changed)
- 移除 `docker.yaml` 的 `on.schedule`（原 `17 10,22 * * *`），只保留 `workflow_dispatch` 与 `push: images.txt`。保留 schedule 会让 GitHub 延迟派发的事件与面板触发叠加，一天收到 4 条微信通知。
- 直接原因：三次定时运行全部严重延迟且 `created_at == run_started_at` —— 11/09 06:17 → 09:43（+3h26m）、11/09 18:17 → 11/10 04:43（+10h26m）、11/10 06:17 → 12:06（+5h50m）。每次 run 本身仅约 7 分钟、9 步全绿，证明延迟发生在 GitHub **派发事件之前**，与 runner 排队、`concurrency`、脚本耗时无关；v2.1.1 的整点错峰也未改善。
- 巡检 automation「镜像同步失败巡检」第 3 类异常由「窗口内 `event=schedule` 的 run < 2 条」改为「窗口内 `workflow_dispatch` / `schedule` 的 run < 2 条（排除 `push`）」。移除 schedule 后所有 run 都是 `workflow_dispatch`，旧判定会永远误报。

### 🐛 修复 (Fixed)
- **青龙触发脚本误判成功为失败**：脚本原按 `HTTP == 200` 判定，而实测未带 `X-GitHub-Api-Version` 时 GitHub 返回 **204 无响应体**（run 实际已创建）。现改为按 **2xx** 判定、`run_id` 有则显示无则给 Actions 链接，并补上 `-H 'X-GitHub-Api-Version: 2026-03-10'`（该版本返回 200 + `workflow_run_id`）。DSM 备选脚本同步修正。

### 📝 文档 (Docs)
- README：开头说明定时触发改由青龙面板负责；「触发同步」表与「为什么是 `:17`」段重写为「为什么不用 GitHub schedule」；巡检章节补窗口内 run 数判定与其局限；已知限制新增「依赖外部调度」。
- v2.1.1 的错峰说明属当时事实，保留不追改，仅加时间指引（该指引现已更新为 v2.2.1 的「每周一次兜底」，见上方）。

### ✅ 验证记录
- 青龙面板实连：容器时区 `CST +0800`（cron 用 `17 6,18 * * *`）、`api.github.com` 直连 200（无需代理）；脚本上传后回读与本地一致；任务 id 1373 已启用。
- 试跑创建的 run **`42764284202`**（2026-11-10 05:47:10Z = 北京 13:47:10）证明 PAT 有效、dispatch 链路通、run 秒级创建；该 run 日志当时显示 `❌ dispatch 失败 HTTP=204`，即本次修复的直接证据。该 run 最终 **success**（6m45s）：`成功: 2, 跳过: 69, 失败: 0`，`🕐 通知时间（北京时间）: 2026-11-10 13:53:56`、`✅ Server酱 通知发送成功 (第 1 次)`。
- 修正版脚本上传青龙后实跑（北京 13:54:31）：`✅ dispatch 成功 HTTP=200 run_id=42765040955`，任务标绿，GitHub 同步出现 run `42765040955`。加版本头后响应确实由 204 变为 200 并带 `workflow_run_id`，两种响应均被正确判定。
- 两个脚本 `bash -n`、LF 换行、无真实凭据；stub 覆盖 200 / 204 / 401 / 缺 PAT 四种分支，204 与 200 均 `exit 0`，401 与缺 PAT 均 `exit 1`，日志无 PAT 泄漏。

### ⚠️ 已知限制
-（本版当时）仓库已无 GitHub 侧定时兜底：NAS 关机、青龙容器停止、任务停用、面板网络不通或 PAT（建议 90 天）过期都会让定时静默失效，靠巡检的 run 数判定报警。（**自 v2.2.1 起已恢复每周一次 `schedule` 兜底**，见上方。）
- 手动 `Run workflow` 与面板触发的 `event` 同为 `workflow_dispatch`，无法从 API 区分，手动补跑可能掩盖一次定时丢失。
- 触发准点 ≠ 通知准点：同步约 7 分钟，微信约 06:24 / 18:24 到达，但不再有 3~10 小时随机漂移。

---

## [v2.1.2] - 2026-11-08

> **通知正文不再重复标题，时间戳改为北京时间**。只改通知步骤的 Markdown 组装与时间计算，同步逻辑、统计口径与推送时机均未变。含两个提交：`1d22491` + `d41693b`。

### 🔧 变更 (Changed)
- `SC_DESP` 删除开头的 `## $TITLE` 及紧随空行，正文直接从 `$CONTENT` 开始。此前 `title=$SC_TITLE` 已由 Server酱 渲染成微信消息第一行，正文又用同一个 `$TITLE` 渲染一次二级标题，于是每条通知前两行完全相同（例如「⏭️ 同步完成，镜像已是最新」出现两遍，「✅ 阿里云镜像同步成功」也出现两遍）。这不是重复推送，而是单条消息内的重复渲染。
- `TITLE` 仍用于 `title` 参数与四种状态判定；统计行、`SYNC_LIST` 过滤、日志链接、`code == 0` 重试逻辑一行未动。
- **时间改为北京时间**：正文「📅 时间」原为 runner 的 UTC（比手机少 8 小时，如 `08:42` vs `16:42`），现由 `SC_TIME=$(date -u -d '+8 hours' ...)` 生成并标注「（北京时间）」。中国标准时间固定 UTC+8、无夏令时，用 `date -u` 固定偏移，不依赖 `TZ=Asia/Shanghai` / tzdata —— 本机 Git Bash 实测 `TZ=Asia/Shanghai date` 会退回 UTC，线上精简 runner 也可能缺时区库。
- 通知步骤新增一行日志 `🕐 通知时间（北京时间）: ...`，便于在 Action 日志里直接核对推送内容，不打印 SendKey / URL。

### 📝 文档 (Docs)
- README 通知章节补「时间为北京时间」；通知预览说明改写为当前实现（标题只渲染一次、北京时间、全跳过时清单为空），不再把重复标题表述为 Server酱 的正常行为；`doc/serverchan-notify.jpg` 已用 run `42478865178` 的真实通知截图替换（1193×1156，78 KB），并去掉「修正前 / 待替换」措辞。
- 已知限制补注：Server酱 会员不受免费版 5 条/天额度限制。

### ✅ 验证记录
- PyYAML 解析通过，7 个 `run` 块 `bash -n` 通过，`git diff --check` 干净，无明文凭据；正文模板断言首行为 `$CONTENT`、不含 `## $TITLE`、`title` 参数保留。
- 本地实测：UTC `12:08:35` → `date -u -d '+8 hours'` 得 `20:08:35`（正确），而 `TZ=Asia/Shanghai date` 得 `12:08:35`（失效），据此改用固定偏移写法。
- **main 实跑 `42478865178`**（`workflow_dispatch`，6m27s，9 步全绿，`success`）：`成功: 0, 跳过: 71, 失败: 0`；日志「📋 通知明细」输出「（本次无新增或失败项）」、`🕐 通知时间（北京时间）: 2026-11-08 20:15:19`（对应 UTC `12:15:19`）、`✅ Server酱 通知发送成功 (第 1 次)`。微信收到新格式消息：标题只显示一次、时间带「（北京时间）」标注。

---

## [v2.1.1] - 2026-11-08

> **定时错峰**：把两次 schedule 从整点挪到整点后 17 分钟，规避 GitHub Actions 整点高负载导致的定时事件延迟 / 丢弃。同步与通知逻辑一行未动，仅改 cron 表达式与注释。

### 🔧 变更 (Changed)
- `cron` 由 `'0 10,22 * * *'`（北京 18:00 / 06:00）改为 `'17 10,22 * * *'`（北京 **18:17 / 06:17**，UTC 10:17 / 22:17）。
- 依据 GitHub 官方文档：`schedule` 按 UTC 执行，「每小时的开头」属于高负载时段，定时事件可能被延迟、排队过久的作业甚至会被丢弃，建议避开整点。
- 直接诱因：2026-11-08 北京 18:00 那次定时**完全没有触发**（19:17 之前无任何 run 记录），历史定时时间戳也普遍比计划晚 3~7 小时。19:17 手动补跑 `42475086532` 成功（7m21s，9 步全绿，`成功: 2, 跳过: 69, 失败: 0`，`✅ Server酱 通知发送成功 (第 1 次)`），确认是调度问题，不是脚本或微信链路故障。

### 📝 文档 (Docs)
- README 定时触发表、错峰说明、每日失败巡检章节同步为 18:17 / 06:17，并补充延迟时 30 小时窗口的兜底说明。
- 巡检 automation prompt 的时间描述同步更新；30 小时窗口、每天 07:00、成功静默等逻辑不变。
- v2.1.0 / v2.0.0 里的「UTC 10:00 / 22:00」是当时的真实配置，保留历史不追改，仅加指引注记。

### ✅ 验证记录
- PyYAML 解析通过，所有 `run` 块 `bash -n` 通过，`git diff --check` 干净。
- 本次为纯 workflow + 文档提交，`on.push.paths` 只含 `images.txt`，不会触发 CI、不会推送微信。cron 是否按时触发需等下一个 UTC 10:17 / 22:17 窗口观察；手动 `Run workflow` 无法验证 schedule，故未额外占用 Server酱 免费额度。

### ⚠️ 已知限制
- 错峰只降低延迟概率，GitHub 仍不保证精确到分钟，实际开始时间可能略晚。若定时再次不触发，可用 WorkBuddy automation 主动 `gh workflow run --ref main` 兜底（代价是依赖本机在线）。（**自 v2.2.1 起 `schedule` 恢复为每周一次兜底 `17 3 * * 3`，青龙面板为主触发**；本节仅记录当时方案。）

---

## [v2.1.0] - 2026-11-08

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
- 修正 `docker.yaml` 的 cron 注释：实际为 UTC 10:00 / 22:00（北京 18:00 / 06:00），旧注释误写 UTC 00:00 / 北京 08:00。（该配置自 v2.1.1 起错峰为 UTC 10:17 / 22:17；自 v2.2.1 起改为每周一次兜底 `17 3 * * 3`，见上方。）

### 📝 文档 (Docs)
- README：Secrets 表用 `SERVERCHAN_SENDKEY` 替换 `WEBHOOK_URL`；通知章节改写为 Server酱（两种 SendKey、Markdown 正文、5 条/天额度）；移除已修正的「定时注释有误」已知限制。
- README（合并后补充）：明确正文只列新增 / 失败、跳过项仅在统计体现；新增「每日失败巡检」章节，说明自动化巡检如何兜住「假绿」。
- README 通知预览改用**真实 Server酱 截图** `doc/serverchan-notify.jpg`（1190×1105，103 KB），替换重写前的旧 Webhook 通知外链图与纯文字示例；图片随仓库走，不再依赖 GitHub `user-attachments` 附件。（该截图自 v2.1.2 起已换成新格式通知，见上方。）
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
- 更正定时时间：`0 10,22 * * *` 是 UTC 10:00 / 22:00（北京 18:00 / 06:00），旧注释写的「UTC 00:00 / 北京 08:00」有误。（该配置自 v2.1.1 起错峰为 `17 10,22 * * *`；自 v2.2.1 起改为每周一次兜底 `17 3 * * 3`，见上方。）
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
