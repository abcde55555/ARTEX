# ARTEX 代码库深度分析（安全工作台改造前置调研）

> 对象：本地克隆 `./ARTEX`（`abcde55555/ARTEX`，`main`，HEAD `a9d11cb`，2026-10-10）。只读源码 + `rg`/定向阅读，未运行程序、未联网；每条非显然结论标注 `文件:行`。
> 结论先行：ARTEX 是**单租户、单管理员、面向 Web 资产**的 LLM 多 agent 自主渗透平台。其「双图 + 编排 + 审批门 + 流量留痕」骨架与领域无关，是改造工作台最有价值的资产；但资产/证据/报告模型、提示词、工具集、权限模型都深度绑定 Web 渗透，新增代码审计/逆向/合规域需要动 schema、tool、agent 三层，而不是只加页面。

---

## 1. 系统概览

**定位**：README 自述「AI 自主渗透测试系统（Go 后端 + Next.js 前端）」（`README.md:5`）；架构是 LLM 多 agent 驱动的「双图架构 + worker 过程级信息交换 + planner 多轮共享 todolist」（`README.md:294`）。

| 项 | 事实 | 证据 |
|---|---|---|
| 语言/模块 | Go，module `github.com/Autumn-27/artex`，`go 1.26.3` | `go.mod:1,8` |
| HTTP | 标准库 `net/http` + Go 1.22 method-pattern 路由（`mux.HandleFunc("GET /api/...")`） | `server/server.go:650,655` |
| 数据库 | PostgreSQL（pgx v5 走 `database/sql`）；注释明确「取代旧 graph 单文件 SQLite」；SQLite **仅**用于流量录制旁路索引/FTS | `db/db.go:1-2`、`traffic/traffic.go:37,194` |
| 前端 | Next.js 16 App Router；`NEXT_EXPORT=1` 时 `output: "export"` 静态导出，再 `go:embed all:webui/dist` 内嵌单二进制 | `web/next.config.mjs:6,24-26`、`server/webui_embed.go:18` |
| Agent SDK | 外部 `norma`（agentcore/tool/permission/harness/memory/transcript/skill/mcp…），**已 vendor 到 `./norma`** | `go.mod:3-6`、`norma/README.md` |
| 流量 MITM | `lqqyt2423/go-mitmproxy`；DNS/探测用 `miekg/dns` + `projectdiscovery/dnsx` | `traffic/traffic.go:1-4,204`、`enrich/enrich.go:1-8,55-60` |

**部署形态**（三路并存）

- 多阶段 Docker：node 构建前端 → golang 编译内嵌前端 → python 运行层，`EXPOSE 8787 8788`，`ENTRYPOINT ["/app/start.sh"]`（`Dockerfile:13,24,32,36,60,69-70`）。
- `docker compose`：`artex` + `postgres:16-alpine`，映射 `8787` 与**仅绑定 127.0.0.1 的** `8788`（`docker-compose.yml:37-38`）；`./data`、`./skills` 绑定挂载。
- 单二进制 + 守护脚本：`-addr :8787`（UI/API/SSE 同端口）、`-proxy 127.0.0.1:8788`（空串关闭）（`cmd/artex/main.go:55,57`）。`start.sh` 只做守护循环：退出码 `0` 停止、`75` 重新拉起（页面一键更新/回滚）、其他按 1→2→4…最多 60s 退避（`start.sh:8-12,29`）；换装/校验/回滚刻意放 Go（`selfupdate/selfupdate.go:1-27`）。

**上游与维护状态（重要）**

- 仓库自述是「ARTEX 最后一个版本纯源码备份」，原镜像 `autumn27/artex` 已失效，改由本仓库维护 `mianhuatang11/artex`（`README.md:10`）；CHANGELOG 最新 `0.3.15 - 2026-10-07`（`CHANGELOG.md:7`），HEAD 是 2026-10-10 的构建修复合并。
- 维护性风险：`selfupdate/github.go:15` 把升级源硬编码为上游 `Repo = "Autumn-27/artex"`，README 安装命令指向 `github.com/mhtsec/ARTEX.git`（`README.md:74,89`），实际 `origin` 是 `abcde55555/ARTEX` —— 三处不一致，页面一键更新大概率不可用，且**备份仓库没有上游安全补丁来源**。
- 历史残留：`cmd/artex/main.go:1-4` 注释仍写「dual SQLite graph stores」，实际已是 PostgreSQL，说明重构后未做全量注释清理。

**许可：AGPL-3.0 + README 额外限制**

- `LICENSE` 是 **AGPL-3.0 标准全文 661 行，未附加第 7 条条款**（`LICENSE:1-3`，尾部为 `END OF TERMS AND CONDITIONS`）。
- 额外限制写在 README 而非 LICENSE：「**仅供个人学习、代码研究与本地技术验证使用，不得用于对任何线上系统或网站发起实际测试**」（`README.md:490`）；「严禁……对任何网站、线上服务或联网系统发起扫描、探测、利用或攻击（无论是否获得授权、是否为自有资产）」「严禁用于任何实际的渗透测试、攻防对抗或生产环境」（`README.md:499-500`）。
- 法务张力：AGPL 本身不限制使用领域，README 的「禁止实际测试」是许可外约定；但 AGPL 第 13 条的网络服务源码开放义务是硬的（`README.md:486` 亦重申衍生作品必须 AGPL）。**工作台化必须 AGPL 兼容开源，或另行取得授权。**

---

## 2. 顶层模块清单

| 目录 | 职责 | 关键入口/文件 |
|---|---|---|
| `agent/` | 领域 agent 与工具层：goals/planner/worker/mainagent、ToolSet、提示词目录、压缩、finding 工作流 | `agent/tools.go`(2054 行)、`tools_insert.go`、`planner.go:355`、`worker.go:349`、`mainagent.go:112`、`goals.go:99`、`toolcatalog.go:36`、`promptcatalog.go:110` |
| `cmd/artex/` | 进程入口：flag、日志捕获、selfupdate bootstrap、Manager、HTTP server | `cmd/artex/main.go:55,81,110` |
| `config/` | JSON 配置 + 环境变量：PG DSN、skill_dir、BaseDir | `config/config.go:17-21,43,116,137` |
| `db/` | PostgreSQL 数据层：幂等 schema + 各 store（资产/企业/探索/任务/配置/拦截/通知） | `db/db.go:20`、`db/schema.sql`(1386 行)、`assets.go`、`exploration.go`、`tasks.go`、`config.go`、`tools.go` |
| `enrich/` | 非 AI 异步补全：DNS 解析 + 经录制代理的 HTTP 探测，回写资产图 | `enrich/enrich.go:1-8,44-60` |
| `evidence/` | 漏洞证据独立存储：SHA-256 内容寻址 blob，与可删流量解耦 | `evidence/store.go:1,29-70` |
| `guard/` | 安全边界：PreToolUse 审计 + 拦截规则评估 + 失败归因 | `guard/guard.go:1-8,57,83,148` |
| `intercept/` | 拦截引擎：规则 allow/deny/ask、人工审批 pending、LLM 兜底判定 | `intercept/intercept.go:1-13,429,508,584` |
| `llmpool/` | 多 LLM profile 故障转移链（rank/优先级轮转 + 健康注册表） | `llmpool/pool.go:1-13,44`、`health.go` |
| `llmrec/` | `llm.Provider` 录制装饰器：全量请求/响应落 PG，带 task/worker 归因 | `llmrec/llmrec.go:1-3,14-37` |
| `mcphttp/` | 远程 MCP 客户端：Streamable HTTP + 旧 SSE，工具名 `mcp__server__tool` | `mcphttp/client.go:1-8,33,38-39` |
| `norma/` | vendored Agent Harness SDK（agentcore/tool/permission/harness/compaction/hook/subagent/mcp/plan/memory/skill/coordinator/llm/noa/transcript） | `norma/README.md` |
| `notify/` | 漏洞发现 IM/邮件推送叶子包（钉钉/飞书/企微/Webhook/Telegram/邮件） | `notify/notify.go:1-25` |
| `report/` | 从探索图渲染交付级 Markdown 报告（按严重级排序） | `report/report.go:1,40` |
| `selfupdate/` | 页面一键更新：Release 下载 → SHA256 → 冒烟 → 原子换装/回滚 | `selfupdate/selfupdate.go:1-27`、`github.go`、`stage.go` |
| `server/` | HTTP API、JWT、SSE、Manager/Engine、全部 handler 与接线 | `server/server.go:650`、`engine.go`、`manager.go`、`assembly.go:29`、`orchestration.go:473` |
| `sidequestion/` | `/btw` 旁路提问（快照 + 独立历史，不打断主循环） | `sidequestion/README.md:1-5` |
| `skills/` | 文件系统技能包，每目录一个 `SKILL.md`（api-recon / playwright-cli / scopesentry） | `skills/*/SKILL.md` |
| `traffic/` | 录制型 MITM 代理 + 树状原文 + SQLite 索引/FTS5 + 证据读取 | `traffic/traffic.go:1-4,204,352,2000` |
| `web/` | Next.js 前端源码（27 个 `page.tsx`） | `web/src/app/**/page.tsx`、`web/src/navigation/sidebar/sidebar-items.ts:64` |
| `docs/`、`screenshots/` | 仅 1 篇设计说明（漏洞多流量证据）+ 15 张 README 截图 | `docs/漏洞流量证据.md`、`screenshots/*.png` |

---

## 3. 后端

### 3.1 路由与 API 分组

单一 `Handler()` 装配两个 mux：`/api/*` 走 CORS+JWT，其余交给内嵌前端（`server/server.go:650,938-942`）。`server/server.go` 内共 **243 条** `HandleFunc` 注册（另有 `server/side_questions.go:273-286` 的旁路路由），按前缀分组：

| 分组（条数） | 代表端点 |
|---|---|
| `/api/tasks`(41) | 任务 CRUD、scope、goals、constraints、intents 控制、归档、资产挂载 |
| `/api/exploration`(24) | frontier/graph/nodes/activity、findings + lineage/deepen/retests、tokens |
| `/api/agents`(19) | agent CRUD/config/prompt/wrapup/triggers/prompt-version/variables/visibility |
| `/api/llm`(17) | profiles、retry-policy、pool status/reset、records、models |
| `/api/intercept`(17) | 规则 CRUD/toggle、pending/decide、history/execution、tool-config、judge |
| `/api/skills`+`/api/tools`(19) | 技能列表/上传 zip/文件树/usage；内置工具覆盖、自定义工具 CRUD/test |
| `/api/conversations`+`/api/chat`(13) | 会话、消息、stop、profile；对话/上传/停止 |
| `/api/notify`+`/api/mcp`+`/api/sync`(18) | 渠道 CRUD/test/投递；MCP CRUD/tools/refresh；ScopeSentry 同步 |
| `/api/workspace`+`/api/traffic`+`/api/update`+`/api/logs`(21) | 文件浏览/读写；流量列表/exchange/blob；升级 check/apply/rollback/stream；日志三件套 |
| 资产类（`assets`/`companies`/`task-scope`/`asset-intercept`/`visibility`，14） | 资产图读写、企业范围、任务/资产拦截、资源可见性 |
| 其余（auth/health/stats/audit/gc/tokens/report/active，15） | 认证、健康、审计、统计 |

细节见 `server/server.go:655-935`。注册是集中式字符串表，新增端点只需追加一行并实现 handler；但 handler 分散在 40+ 个文件（`server/assets.go`、`server/intercept.go`、`server/task_control.go` 等），新域建议同步新建 `server/<domain>.go`。

### 3.2 鉴权：单用户 JWT，无角色/RBAC

- HS256，`Subject: "ARTEX"`，TTL 7 天；密钥首次随机生成 32 字节写入 `<baseDir>/jwt.key`（0600），并刻意从可浏览 data 目录迁出（`server/auth.go:20-22,48-84,86-93`）。
- Token 来源：`Authorization: Bearer`、`artex_token` cookie、`?token=`（SSE）（`server/auth.go:106-115`）。`requireAuth` 仅豁免 `/api/auth/*` 与 `/api/health`（`server/auth.go:120-136`）。
- 密码 bcrypt，8 rune ~ 72 byte；`/api/auth/init`「仅首次可设」由主键约束兜底（`server/auth.go:30-44,183-200`）。
- **没有 `users`/`roles`/`tenant_id` 表**（`db/schema.sql` 全文无 users 表），也没有资源所有权过滤——所有登录者都是同一管理员。这是工作台化的最大结构性缺口之一。

### 3.3 SSE 实时

- 进程内 `Broadcaster`：每任务一个订阅集合，`Publish` 非阻塞（缓冲 256，满则丢弃，客户端靠 `Last-Event-ID` 从 DB 补齐）（`server/broadcast.go:1-13,33-62`）。
- `/api/exploration/activity/stream`：先订阅再回放 `activity` backlog（每批 500 直到追平），标准 `id:/data:` 帧，20s 心跳，支持 `?intent=` 只看单条 work（`server/server.go:2979-3075`）。
- 另有 `/api/logs/stream`、`/api/update/stream`、旁路 `/api/side-questions/{id}/events`（`server/server.go:2928`、`server/update.go:334`、`server/side_questions.go:529`）。
- 前端统一 `sseUrl()`：生产同源、dev 直连 `:8787`（Next dev 的 `/api` rewrite 会缓冲 SSE），token 以 query 附加（`web/src/lib/api.ts:166-196`）。

### 3.4 存储层与 PostgreSQL schema

- 启动时 `//go:embed schema.sql` 整篇 `Exec`，靠 `IF NOT EXISTS`/`OR REPLACE`/`DROP TRIGGER IF EXISTS` 幂等，用 `pg_advisory_lock` 串行化多实例迁移（`db/db.go:20-22,83`）。**没有版本化 migration 目录**：schema 即迁移源，新域可直接追加，但旧库兼容要靠 `ALTER TABLE ... IF NOT EXISTS` 手法（`db/schema.sql:516-566`）。
- 约 40 张表（`db/schema.sql`），关键分组：

| 层 | 表（行号） | 说明 |
|---|---|---|
| 资产图（全局共享） | `companies`(:27)、`assets`(:40)、`company_scope`(:115) | 资产类型 `root_domain/ip/subdomain/app/service/endpoint`(:42-44)，部分唯一索引去重(:83-91)、GIN/GIST 索引(:94-101) |
| 探索图（每任务） | `explorations`(:153)、`exploration_nodes`(:170)、`exploration_edges`(:248)、`exploration_anchors`(:276) | 节点 kind `begin/goal/intent/fact/finding/hint/digest`；边 rel `spawns/derived_from/yields/proves/covers`(:248-255) |
| 过程留痕 | `activity`(:298)、`main_sessions`(:346) | 每步工具调用/结果/usage，按 `(exploration_id,id)`、`tool_use_id`、`worker` 建索引 |
| 任务/范围 | `tasks`(:505)、`task_scope`(:738)、`task_constraints`(:287)、`task_archives`(:571)、`task_relations`(:636)、`task_asset_links`(:648)、分类/模板 | scope kind 仅 `company/root_domain/subdomain/ip/cidr/icp/keyword`(:741) |
| Agent/提示词 | `agents`(:766)、`agent_prompts`(:804)、`agent_prompt_vars`(:820)、`agent_triggers`(:993)、`agent_visibility`(:877)、`agent_skill_visibility`(:888) | `agents.role` CHECK 仅 `goals/main/planner/worker/assistant`(:789) |
| 工具/计量 | `tools`(:935)、`tool_usage`(:919)、`skill_usage`(:900) | `tools.kind` builtin/command/script/http/shell，`agents` 为 JSONB 绑定数组 |
| 审批/拦截 | `intercept_rules`(:1033)、`intercept_pending`(:1053)、`asset_intercept_rules`(:1241)、`task_intercept_rules`(:1267) | 规则 action `allow/deny/ask`，超时策略 `timeout_action` |
| 漏洞/证据 | `findings`(:1081)、`finding_retests`(:1112)、`traffic_evidence_snapshots`(:1146)、`finding_traffic_bindings`(:1163) | findings 带 `vulnclass/severity/status/report/asset_ids` |
| 会话/旁路/配置/通知 | `conversations`(:954)、`side_question_*`(:1189,:1207)、`settings`(:356)、`llm_profiles`(:362)、`mcp_servers`(:833)、`server_logs`(:1179)、`notification_*`(:1303,:1341,:1359) | 通知写漏洞事务内盲插事件，投递用租约（`db/schema.sql:1289-1299` 注释） |

**双图是核心知识资产**：资产图是跨任务共享的「真值库」，探索图是每任务的「推进链」，通过 `exploration_anchors(node_id, asset_id)` 连接（`db/schema.sql:276-281`、`README.md:354-390`）。工作台要支持代码审计/逆向，**最小改动是把 `assets.type` 与 `task_scope.kind` 扩展出代码/二进制/制品类型**，或平行引入第二类「工件图」；但锚点、覆盖度（`server/server.go:1741,1769`）与前端覆盖图都假定 Web 资产语义，迁移成本不能低估。

---

## 4. Agent 子系统

### 4.1 角色与编排流

| 角色（agent key） | 职责 | 证据 |
|---|---|---|
| `goals` 目标拆解 | 第 0 轮把任务描述+目标拆成可验证 goal，并抽取操作约束 | `db/db.go:190-193`、`agent/goals.go:99,110`、`agent/promptcatalog.go:115` |
| `planner` 规划 | **唯一意图生成者**；读态势、判目标、按需派 0..N 条 `intent`；持有跨唤醒共享 todolist | `agent/planner.go:355`、`README.md:438-462` |
| `worker` 执行 ×N | 领一条 intent，用真实工具执行，把资产/fact/finding 写回两图后停 | `agent/worker.go:349` |
| `mainagent` 人在环路 | 对话页驱动，只观察 + 注入 hint/高优先级 intent/改目标，不跑自主循环 | `agent/mainagent.go:112` |
| `auto` 平台操作 | 经对话页用工具管理任务/技能/自定义工具/MCP | `db/db.go:210-211`、`server/platform_tools.go:15-35` |
| `pentest` 独立渗透 | 单人从侦察到收尾，自规划/自执行/自验证 | `db/db.go:212-213` |
| `reporter` 报告撰写（运行时 seed 的自定义 agent） | finding 落库后触发，核对证据/绑定流量/写 Markdown 报告 | `server/orchestration.go:745`、`agent/promptcatalog.go:72` |

引擎：每任务 1 个 planner goroutine + `workers` 个 worker（默认 3，`settings.workers` 可调，`server/manager.go:263,319`；`server/engine.go:748-766`）。planner 事件驱动：图变更 debounce 后唤醒一轮，无新方向就派 0 个意图并结束本轮（`server/engine.go:800-863`）；worker 循环 claim 意图执行（`:916-965`）；任务级超时由独立 deadline coordinator 收尾（`:768`）。

### 4.2 ToolSet 与实际暴露给 LLM 的工具

装配顺序：领域工具（`ToolSet`）→ host 工具（traffic/编排/平台）→ SDK 默认工具 → 技能/MCP 增量 → DB `tools` 表按 agent 过滤/覆盖 → finding 工作流补丁 → panic 包裹（`server/assembly.go:39-67`）。

**A. 领域工具**（`agent/tools.go`、`agent/tools_insert.go`）

| 工具 | 作用 | 位置 |
|---|---|---|
| `graph_overview` | 本任务态势（资产计数/frontier/发现/覆盖） | `agent/tools.go:341` |
| `list_findings`/`list_facts`/`node_detail`/`expand_digest` | 读漏洞/事实/节点详情/解压冷摘要 | `:902,940,993` |
| `add_intent`/`add_hint` | 派探索方向 / 注入人类战略提示 | `:1091,1617` |
| `list_goals`/`prove_goal`/`goal_met` | 目标枚举/逐个证明/全局收官（`goal_met` 默认不绑任何 agent） | `:1149,1158,1206`、`agent/toolcatalog.go:56` |
| `record_fact`/`report_finding` | 写事实 / 登记确认漏洞（含 evidence、可选 traffic_refs） | `:1353,1219` |
| `set_goals`/`set_constraints` | 运行时设目标/操作约束（allow/deny） | `:1474,1568` |
| `kill_work`/`steer_work`/`get_worker_output`/`get_worker_trace`/`search_all_worker_traces`/`list_worker_traces` | 跨 work 干预与过程级信息交换 | `:1677,1707,1741,1822,1931,1980` |
| `insert_assets`/`add_company_scope`/`add_task_scope`/`list_assets`/`list_companies`/`list_untested_assets` | 双图写入与范围管理 | `tools_insert.go:121,377,431,530,587,487` |

子集：`PlannerTools()`（读+派意图+目标判定，`agent/tools.go:2033`）、`WorkerTools()`（写回为主，刻意不给 `list_facts`/`list_companies`/`list_worker_traces`，`agent/tools_insert.go:644-662`）、`MainAgentTools()`（人机接口，`:664-686`）。

**B. SDK 默认工具**：Read/Write/Edit/MultiEdit/LS/Glob/Grep/Bash（+Sleep、WebFetch/WebSearch、TodoWrite、AskUserQuestion、后台任务、shell 会话）。worker 刻意剔除 MultiEdit/Glob/Grep（`agent/worker.go:379`）。

**C. 流量工具**：`traffic_search`（host 必填 + URL 子串 + 正文全文，返回轻量索引）、`traffic_get`（取原文）、`traffic_blob`（按 sha256 分段读大 body）（`traffic/traffic.go:2000,2007,2058,2079`）。

**D. 编排/平台工具**：`list_tasks`、`spawn_task`、`pause_task`、`get_task_graph`、`list_task_findings`、`add_task_hint`、`get_task_worker_trace`、`list_task_worker_traces`、`search_task_worker_traces`、`get_task_node_detail`、`update_finding_report`、`list_llm_profiles`（`server/orchestration.go:177-460`）；`create_skill`/`update_skill_file`/`create_custom_tool`/`update_custom_tool`/`create_mcp`/`update_mcp`/`delete_assets_by_host`（`server/platform_tools.go:15-35`）。

**E. finding 工作流**：`get_finding_traffic`、`bind_finding_traffic`、`update_finding_report`，证据版本化、绑定可选（`docs/漏洞流量证据.md`、`agent/finding_workflow.go:25-90`）。

**F. 技能/MCP 增量**：见 4.5/4.6。

### 4.3 预算与限制

- 每 agent：`max_turns`（0=不限）、`run_seconds`（默认 1200s，`pentest` 显式 0=不限）、`interactive_shell`、`web_search`、wrapup/task-timeout wrapup 轮数（`db/schema.sql:766-790`、`db/db.go:190-213`）。
- 任务级：`timeout_seconds`、`plan_heartbeat_seconds`（默认 300s）、并发上限（`task_concurrency_enabled/limit`，默认关/5）（`db/schema.sql:524-525`、`server/server.go:3303-3305`）。
- 单次 run 墙钟：worker `SetRunTimeout`（`server/task_llm.go:570`），到点由 SDK `MaxDuration` 打断在跑工具并进入收尾（`agent/worker.go:424,458,527-529`；settlement 阶段隐藏 Bash 于 `:462`）。
- 上下文/输出：按模型 `ContextWindowK` 算压缩阈值，支持 `noa` 实验压缩（`agent/provider.go:45-47`、`server/manager.go:277`）；`MaxTokens`、`RatePerSecond/RatePerMinute` 共享限速（`agent/provider.go:41-44,62-70`）。
- **没有硬 token 预算/成本熔断**：只有轮次与墙钟限制，多 agent + 重试/压缩/兜底审批/旁路会放大成本。

### 4.4 记忆、transcript、压缩

`transcript.Store` 落盘 raw LLM 对话（`<data>/transcripts`，`server/task_llm.go:562`），支持 session 续跑；压缩为内置 compaction（`agent/compaction.go`）+ 冷节点 digest（`agent/coldgraph.go`、`expand_digest`）；长期记忆用 SDK `memory` 包（`norma/memory/`），ARTEX 侧无 UI；过程记忆即 `activity` 表，worker 可跨 work 检索（`README.md:419-437`），比 transcript 更结构化、更适合审计。

### 4.5 skills 机制

- 文件系统 `skills/<name>/SKILL.md`，YAML frontmatter（`name/description/license/compatibility/allowed-tools/mcps`），由 norma `skill.LoadDir` 解析（`norma/skill/skill.go:207-241`）。
- 根目录优先级：`ARTEX_SKILL_DIR` > `config.json.skill_dir` > 可执行文件同级 `skills/`（`config/config.go:113-127`）；Docker 下绑定挂载持久化。
- 可见性由 `agent_skill_visibility` 控制，组装时只装该 agent 可见的 skill，并生成 Skill 元工具；技能可在 frontmatter 声明 `mcps:`，调用时才解锁对应 MCP 工具（`server/assembly.go:29-77,66-95`）。默认只给 `api-recon` 绑 `auto/pentest/worker`，`playwright-cli`、`scopesentry` 默认不可见（`db/db.go:317-326`）。
- 现成技能即领域扩展范例：`skills/api-recon` 用 Playwright hook/stub 在无凭据下枚举前端 API（含 `scripts/` 下 7 个脚本）；`playwright-cli` 是通用浏览器自动化；`scopesentry` 对接外部扫描平台 MCP。
- 维护瑕疵：`skills/scopesentry/SKILL.md:1-4` frontmatter 畸形（开头 `---` 后空行 + `## name:`，无闭合 `---`），按 `norma/skill/skill.go:259-275` 的宽松解析会退化为「Name 取目录名、Description 为空」，即技能在但对模型不可描述。

### 4.6 MCP / LLM 池 / 旁路 / 通知 / 证据 / 报告

- **MCP**：stdio（SDK `norma/mcp`）+ Streamable HTTP（`2025-06-18`）+ 旧 SSE（`2024-11-05`）（`mcphttp/client.go:1-8,38-39`）；工具统一 `mcp__<server>__<tool>`，按 agent 可见性注入，支持技能门控；清单缓存 `mcp_tools_cache`，UI 可 refresh（`server/server.go:866-870`、`db/schema.sql:864`）。默认 seed 一个 Playwright 浏览器 MCP 且**默认禁用**，流量捕获开关动态注入/剥离录制代理与 CA（`db/db.go:255-270`、`server/manager.go:640`）。
- **LLM 池**：`llm_profiles` 支持 anthropic/openai 双格式、baseURL、代理、限速、上下文窗口、thinking/reasoning、stream/非流式、MaxTokens 字段名（`agent/provider.go:32-78`、`db/schema.sql:362-409`）。`llmpool.Pool` 故障转移：active 置顶、其余按 priority、同 rank 轮转、窗口装不下则跳过、全失败 `ErrExhausted("LLM 轮询：所有配置均不可用")`（`llmpool/pool.go:1-13,30-44,52-56`）；健康状态独立于 provider 缓存（`server/server.go:105-112`）。
- **录制**：`llmrec.Wrap` 录制每次 Stream 的完整请求/累积响应写 PG，带 task/worker 归因，默认关（`settings.llm_record`）（`llmrec/llmrec.go:1-3,14-37`、`server/manager.go:264`、`server/server.go:795-799`）。
- **旁路**：`/btw` 快照主上下文后单次 Provider 调用，不建 session、不写主 transcript/活动流；每父会话 1 并发、进程最多 4、单次 120s；保留历史与用量（`sidequestion/README.md:1-30`、`db/schema.sql:1189-1225`）。
- **通知**：叶子包、渠道实现必须无状态；事件在写漏洞事务内盲插，投递用 `FOR UPDATE SKIP LOCKED` 租约领取，限流不消耗重试预算；六渠道 + 过滤/汇总/重发（`notify/notify.go:1-25`、`db/schema.sql:1289-1299`、`CHANGELOG.md:20-24`、`server/server.go:804-811`）。
- **证据**：正文 SHA-256 内容寻址存 `<data>/evidence/blobs/<前两位>/<hash>.bin`，写入先 staging 再原子可见，读取校验长度+哈希；快照保存原流量 ID/头/正文长度/哈希，与可清理的 `data/traffic` 独立（`evidence/store.go:29-70`、`docs/漏洞流量证据.md`）。
- **报告**：从 findings 渲染 Markdown（标题/目标/资产计数/按严重级排序的漏洞 + PoC/证据）（`report/report.go:1,40`）。**模板是渗透语义**（vulnclass/severity），没有可配置模板/合规章节。

---

## 5. 安全 / 审批 / 留痕控制（未来工作台最重要）

### 5.1 工具审批门（guard + intercept）

- 每次工具调用先过 `PreToolUse` hook：guard 记审计条目（tool/action/reason/command）→ `applyIntercept` 评估（`guard/guard.go:1-8,57-100`）。Bash 与交互式 shell（`shell_open.command`、`shell_send.text`）额外抽取命令文本入审计（`guard/guard.go:64-76`）。
- 规则来自 DB，内存缓存，按 priority 降序；`match_target` 为 `tool_name|tool_input`，`match_type` 为 `string|regex`，`action` 为 `allow|deny|ask`（`intercept/intercept.go:1-13,508-548`、`db/schema.sql:1033-1050`）。
- `ask`：写 `intercept_pending`（规则、会话/任务、agent、工具名、完整 input、审计 payload），向会话或任务活动流写 `intercept_request`，然后**阻塞等待**；超时默认 60s/deny，支持「无超时无限等」，worker 被取消时立即 deny 并标 `not_executed`（`intercept/intercept.go:584-664`）。
- 决策入口 `POST /api/intercept/pending/{id}/decide`，历史/详情/执行结果分端点（`server/server.go:922-928`）。`allow/deny` 与模型判定也以「已决策行」写 pending 表做可观测记录，`decision_source` 区分 rule/model（`intercept/intercept.go:561-570`、`db/schema.sql:1074-1076`）。
- 兜底模型审批：无规则命中时可调 LLM 判 allow/ask/deny，独立 profile、独立 token 计量（lane=`judge`），配置与用量可查（`intercept/intercept.go:429`、`server/intercept.go:21-23,31-55,414-460`）。
- 内置规则首次启动 seed（不是硬编码闸门，用户可删）：rm -rf、关键目录、mkfs、dd、fork bomb、关机、杀进程、shred、清空防火墙、DROP/TRUNCATE、MongoDB drop、Redis FLUSH、curl/wget DELETE、脚本 DELETE、批量删除类路径（`db/db.go:345-521`）；`CHANGELOG.md:13` 补充了「GET/POST 删除类接口」规则。

### 5.2 资产/范围闸门（RoE 的替代物）

- 两层规则：全局 `asset_intercept_rules`（内置 `.gov/.gov.cn/.edu/.edu.cn` 模糊域拦截，`db/db.go:288-320`）+ 任务级 `task_intercept_rules`（`action=block|allow` 白名单）（`db/schema.sql:1241-1280`）。
- 判定：先查 block（全局∪任务 block），命中即拒；若任务存在启用 allow 规则，未命中 allow 也拒；否则放行（`db/asset_intercept_match.go:178-196`）。执行点在 `insert_assets`（`agent/tools_insert.go:232-250`）与资产挂载/列出路径（`db/asset_intercept_match.go:215-250`）。
- 注意：guard 包注释明确「RoE 授权范围机制已被移除，后续可能再加」（`guard/guard.go:5-7`）。当前「授权」= 资产黑/白名单 + 人工审批，没有委托书/时间窗/委托人/签字/过期字段，不可作为正式授权证据。

### 5.3 流量录制代理（MITM 留痕）

- go-mitmproxy 内嵌，监听 `127.0.0.1:8788`，默认 MITM 所有 host；对「证明拦不了」的 host 自动降级透明隧道（fail-open）（`traffic/traffic.go:196-232`）。
- CA 证书 `data/_ca/mitmproxy-ca-cert.pem`，路径注入 worker shell 的 5 个环境变量（`SSL_CERT_FILE`/`CURL_CA_BUNDLE`/`REQUESTS_CA_BUNDLE`/`GIT_SSL_CAINFO`/`NODE_EXTRA_CA_CERTS`）并设 HTTP(S)/ALL_PROXY；注释记录了「python requests 忽略 SSL_CERT_FILE」这类实证坑（`traffic/traffic.go:352-355`、`agent/worker.go:198-221`）。
- 存储：`data/traffic/<host>/...` 原文树 + `_index/index.sqlite`（WAL、`auto_vacuum=incremental`）；FTS5 **trigram**（支持任意子串与 CJK，`ssw0r` 命中 `P@ssw0rd`）；<256KB 内联，超过落内容寻址 blob 并保留 8KB 预览（`traffic/traffic.go:1-4,20-100`）。
- 出口代理：`SetUpstreamProxy` 默认强制直连，避免环境 `HTTP_PROXY` 把目标流量绕到不可达外部代理；配置全局出口后所有请求（含透明隧道）走出口（`traffic/traffic.go:212-231`、`server/server.go:3301`）。
- 清理：删除流量后做 FTS 合并与增量回收；绑定到漏洞的正文另存 evidence，不受清空流量影响（`CHANGELOG.md` 流量段、`docs/漏洞流量证据.md`）。

### 5.4 审计/活动留痕

- `activity` 是全过程流水：每步工具调用、结果、usage、`tool_use_id`、worker、`main_seg`（`db/schema.sql:298-340`），SSE 实时推送且可从 DB 回放，天然适合「操作时间线/取证」。
- `/api/audit` 暴露 guard 审计条目（`server/server.go:784,3864`）；`server_logs` 持久化后端日志（`db/schema.sql:1179`）；`/api/logs`、`/api/logs/history`、`/api/logs/stream` 三件套（`server/server.go:662-664`）；`tool_usage`/`skill_usage` 可按任务/工具聚合（`db/schema.sql:900-930`、`server/server.go:793-794`）。
- 结论：**「谁在何时对什么资产、用什么命令、得到什么结果、是否经人批准」这条链在 ARTEX 里是闭合的**——这是合规检测/留痕能力的最佳基础；缺的是「人」的维度（无用户身份）与「授权」的正式模型。

---

## 6. 前端

- Next.js 16 App Router，路由组：`(auth)` 登录/初始化、`(main)` 主控制台、`(external)` 落地页（`redirect("/function/tasks")`，`web/src/app/(external)/page.tsx:1-4`）。共 27 个 `page.tsx`。
- 页面清单：**功能** `dashboard`、`chat`、`function/tasks`（列表）+ `tasks/detail`（10 个 tab）、`findings` + `findings/detail`、`traffic`、`commands`、`llm-records`、`assets`、`sync`、`workspace`；**系统** `llm`、`agents` + `agents/detail`、`mcp`、`skills`、`tools`、`notify`、`intercept` + `intercept/assets` + `intercept/approvals`、`logs`、`settings`；**认证** `login`、`setup`。
- 任务详情 tab（`web/src/app/(main)/function/tasks/detail/_tabs/`）：`overview`/`sessions`（SSE 执行过程）/`graph`（探索链路）/`coverage-graph`（资产覆盖力导向图）/`assets`/`findings`/`retests`/`report`/`broadcast`/`intercept`。
- 导航集中：`sidebarItems` 两组（功能 10 项、系统 11 项）定义标题/URL/图标/badge（`web/src/navigation/sidebar/sidebar-items.ts:64-98`）。**新增页面必须同步改这里**。
- 图形：`@antv/g6` + `@antv/graphin` + `@xyflow/react` + `recharts`（`web/package.json`）；探索图组件 `web/src/components/exploration-graph.tsx`。
- 状态与 API：无全局数据层框架（zustand 仅 preferences）。统一 `http<T>()`：自动带 `Authorization: Bearer <localStorage artex_token>`，401 清 token 跳登录，错误取 `{error}`（`web/src/lib/api.ts:109-148`）；SSE 用 `sseUrl()`（`:189-196`）；类型集中在 `web/src/lib/types.ts`（1612 行），与后端 DTO 手工对齐。
- Mock：`NEXT_PUBLIC_MOCK=1` 时 `http()` 短路到 `mockHandle()`，本地 fixtures 模拟全部端点、未命中返回安全默认（`web/src/lib/mock/enabled.ts:3`、`handler.ts:1-6`、`data.ts` 4223 行）；mock 模式跳过 `/api` rewrite（`web/next.config.mjs:7,28-33`）。在线 Demo 即「纯前端 + Mock」（`README.md:18`）。
- 构建：`npm run build:static` 产出 `web/out`，Docker COPY 到 `server/webui/dist` 后 `go:embed`（`Dockerfile:28-32`、`server/webui_embed.go:18`）。

---

## 7. 扩展点：新增能力域要动哪里

### 7.1 各层「插槽」

| 层 | 扩展点 | 位置 | 改动方式 |
|---|---|---|---|
| Agent 注册 | 内置 agent seed 数组 | `db/db.go:188-213`（`builtinAgents`）、`:216` seedBuiltins | 追加 `{key,name,role,desc,vars,...}`；**role 受 CHECK 约束**（`db/schema.sql:789`），新角色需改 CHECK 或复用 `assistant` |
| 自定义 agent | 运行时创建 | `POST /api/agents`（`server/server_mgmt.go:246-279`、`db/config.go:555-567`，固定 `role='assistant'`） | UI 创建，无需改代码 |
| 提示词 | 内置默认正文目录 | `agent/promptcatalog.go:110-121`（key=agent key） | 加模板常量 + 注册；`agent_prompts` 版本化且 UI 可编辑（`db/schema.sql:804-818`） |
| 工具（内置领域） | `ToolSet` 构造器 + 各 agent 工具集 | `agent/tools.go:280-320`（`read/write/readExp/writeExpTool` 工厂）、`:2033`、`agent/tools_insert.go:644,664` | 写构造函数并加入对应工具集；可选入目录 `agent/toolcatalog.go:36-70` |
| 工具（跨任务/平台） | host 工具 | `server/orchestration.go:40-84,473-505`、`server/platform_tools.go:15-35` | 实现 `actool.CoreTool` 并在 `hostTools()`/`orchestrationTools()` 注册，再用 `SeedTool` 入库 |
| 工具（免代码） | 自定义工具 kind command/script/http/shell | `server/customtool.go:32-64,144-180`、`db/schema.sql:935-947` | UI「系统 → 工具」创建；kind 白名单在 `server/customtool.go:60` |
| DB schema | 单一幂等 `schema.sql` | `db/schema.sql`（`db/db.go:20` embed，启动整篇 Exec） | 追加 `CREATE TABLE IF NOT EXISTS` + 旧库 `ALTER TABLE ... ADD COLUMN IF NOT EXISTS`；新 store 放 `db/<domain>.go` |
| 服务端路由 | 集中路由表 | `server/server.go:650-942`（注册 655-935，中间件 938） | 加 `mux.HandleFunc("METHOD /api/...", s.handler)` + 实现；横切配置参考 `server/manager.go:252-277` |
| SSE | `Broadcaster` + stream handler | `server/broadcast.go:34-62`、`server/server.go:2979` | 复用 activity 流，或仿 `streamLogs` 新增专用流 |
| 前端页面/API | App Router 目录 + 客户端封装 | `web/src/app/(main)/<group>/<page>/page.tsx`、`web/src/lib/api.ts`、`types.ts` | 新目录即新路由（静态导出生成 `<route>/index.html`）；加 api 方法 + 类型 + mock 分支 |
| 前端导航 | 侧栏配置 | `web/src/navigation/sidebar/sidebar-items.ts:64-98` | 加 item + `NavGroup` |
| 技能 | 文件系统 `SKILL.md` | `skills/<name>/SKILL.md`、`config/config.go:116-127`、`server/assembly.go:36-46` | 新建目录 + frontmatter；可见性写 `agent_skill_visibility`（默认种子 `db/db.go:324`） |
| MCP | 远程/stdio 服务器 | `mcphttp/client.go`、`server/assembly.go:79-140`、`db/schema.sql:833` | UI 配置即可接入，工具自动 `mcp__server__tool` |
| 配置/通知 | JSON+env+DB settings；渠道/事件 | `config/config.go:17-21,129-150`、`server/manager.go:252-277`、`notify/notify.go:18-25`、`db/schema.sql:1303-1385` | 新配置项加 struct+env+`/api/settings`；新事件/渠道按现有接口扩展 |

### 7.2 「新增 agent / 工具 / 页面」清单（从代码直接推导）

**新增内置 agent（例如 `codeaudit`）**
1. 定 role：复用 `assistant` 最省事；新 role 要改 `db/schema.sql:789` 的 CHECK 并在 `db/config.go` 写入路径同步白名单。
2. `db/db.go:188-213` 追加 `builtinAgent{key:"codeaudit", role:"assistant", ...}` 及 prompt 变量目录。
3. `agent/promptcatalog.go:110-121` 增加 `"codeaudit": codeauditDefaultTmpl`。
4. `agent/toolcatalog.go:36-70` 决定默认绑定工具（旧库迁移参考 `server/orchestration.go` 的 `seed*Bindings`）。
5. 执行路径二选一：走编排（引擎 planner/worker 循环，接线点 `server/task_llm.go:550-600` 的 `agentsForTask`）或走对话页（`server/conversations.go:549-563` 的 `ChatAgent` 通路）。
6. 前端「系统 → agents」数据驱动自动列出，无需新页面。

**新增工具（例如 `scan_sast_rule`）**
1. `agent/tools.go` 用 `readTool/writeTool` 或 `t.readExpTool/writeExpTool` 定义 `Name/Description/InputSchema/Run`（工厂签名 `agent/tools.go:280-320`）。
2. 加入 `PlannerTools()`（`:2033`）/`WorkerTools()`（`agent/tools_insert.go:644`）/`MainAgentTools()`（`:664`）；跨任务工具加 `server/orchestration.go:65-84` + `hostTools()`。
3. 要 UI 可管理（改描述/schema/按 agent 勾选）：确保被 `BuiltinToolSeeds()` 收集（`agent/toolcatalog.go:100`）并由 `SeedTool` 写入 `tools` 表；`ToolResolve` 负责运行时过滤/覆盖（`agent/toolcatalog.go:102-116`、`server/assembly.go:60-62`）。
4. 免代码方案：用「系统 → 工具」创建 `kind=command|script|http|shell`（`server/customtool.go`）。
5. 写回要有幂等与审计：参考 `insert_assets` 的 `WriteCounts` 与 activity 落库模式。

**新增页面（例如「代码审计」）**
1. 建 `web/src/app/(main)/function/codeaudit/page.tsx`（客户端组件 + `api.*`，参考 `function/assets/page.tsx`）。
2. 加导航项 `web/src/navigation/sidebar/sidebar-items.ts:64`。
3. `web/src/lib/api.ts` 加方法、`types.ts` 加类型；要 demo 可用则同步 `mock/handler.ts` + `data.ts`。
4. 后端路由加到 `server/server.go:655-935`，handler 建议新文件 `server/codeaudit.go`；表加 `db/schema.sql`。
5. 需实时进度复用 `/api/exploration/activity/stream`（`server/server.go:2979`）或仿 `streamLogs` 新建 SSE。

**关键判断**：ARTEX 的**结构**（agent 注册、工具 seed、技能目录、路由表、导航表）扩展成本很低；真正贵的是**领域模型迁移**。

---

## 8. 现状与差距

### 8.1 已经做得好的

- **多 agent 编排工程完整度高**：事件驱动 planner + N worker + 共享 todolist + 跨 work 过程检索，且有大量「为什么这样设计」的注释（`server/engine.go:800-965`、`README.md:392-462`）。
- **双图知识模型**：资产真值库与探索血缘链分离、锚点连接，覆盖度/覆盖图直接建在图上（`db/schema.sql:170-281`、`README.md:354`）。
- **审批门与留痕闭环**：规则化 allow/deny/ask + 人工 pending + 超时 + LLM judge + 审计条目 + 全过程 activity（第 5 节）。
- **流量证据工程化**：CA 注入、FTS trigram、内容寻址 blob、证据版本化、与可删流量解耦（`traffic/`、`evidence/store.go`）。
- **LLM 运维与工程细节**：多 profile/故障转移/限速/录制/token 多维计量（`llmpool/`、`llmrec/`、`db/llm_usage.go`）；`guardPanic` 防工具 panic 拖垮进程（`agent/assembly.go:70-75`）、workspace 路径逃逸防护（`server/workspace.go:29-38`）、selfupdate 白名单 + SHA256（`selfupdate/github.go:25-31`）、notify SSRF 拨号期防护（`CHANGELOG.md:32`）。

### 8.2 对 代码审计/逆向分析/合规检测 的差距

| 能力域 | 缺什么（证据） |
|---|---|
| **代码审计** | ① 资产模型只有网络资产类型（`db/schema.sql:42-44`），无 repo/commit/文件/符号/依赖/规则命中实体；② 无代码工具（4.2 工具表只有通用 Read/Grep/Bash）；③ 提示词与目标拆解是渗透语义（`agent/promptcatalog.go:115`）；④ 报告模板只认 vulnclass/severity（`report/report.go`）。可行路径：把被审代码挂进任务工作目录 + 通用文件工具 + 新工具，但**必须新增实体表与报告模型**，否则结果只能堆在 fact/finding 自由文本里。 |
| **逆向分析** | ① 无二进制/样本/函数/调用图实体；② 无专用工具（反汇编、strings、符号执行、沙箱）；③ 流量/HTTP 模型无法承载样本行为；④ `max_turns`/`run_seconds` 对长时逆向可能偏紧（`db/schema.sql:776-777`）。可行路径：MCP（IDA/Ghidra/Binary Ninja 生态）+ 自定义工具 + skills；但产物需新图 schema，否则不可检索/复用。 |
| **合规检测** | ① 无「基线/控制项/证据映射/合规结论」模型，`findings.status` 只有处置态（`db/schema.sql:1081,1097`）；② 无报告模板引擎（`report/report.go` 硬编码）；③ 无授权书/RoE 实体（`guard/guard.go:5-7`）；④ 资产拦截只有域名/IP/URL/CIDR（`db/schema.sql:1244-1248`），无法表达合规范围/数据分级。 |
| **跨域共性** | ① **无多租户/多用户**：单管理员 JWT、无 users/roles/tenant 表（`server/auth.go:86-136`）；② 审批记录只有 `agent_name`，无人类身份（`db/schema.sql:1053-1071`）；③ 无数据保留/删除策略（仅流量可清、任务可归档，`db/schema.sql:571`）；④ 无审计事件外发（notify 只推漏洞，`notify/notify.go:1-25`）。 |

### 8.3 风险清单

1. **供应链/维护（高）**：单版本源码备份；`norma` 是上游转闭源后从 module proxy 还原的 vendor（`go.mod:3-6`），无上游安全更新；一键更新仍指向上游（`selfupdate/github.go:15`）。→ 工作台化必须冻结并自维护依赖，评估 fork/替代 SDK。
2. **许可（高）**：AGPL 网络服务源码开放义务（`README.md:486`）+ README「禁止实际测试」（`:490-502`）与 AGPL 授予的使用自由存在张力；闭源分发直接违约，企业/对外服务前需法务确认。
3. **LLM 成本（中高）**：多 agent + 重试/压缩/兜底审批/旁路/录制放大 token；`llmrec`/`judge`/旁路各自计量说明作者已意识到，但仍无**硬预算/配额/成本熔断**（只有 `max_turns`、`run_seconds`）。
4. **单租户/无 RBAC（高，工作台场景）**：任何持 JWT 者可读全部资产/任务/流量/证据，可改 LLM key、删资产、操作工作区文件（`server/server.go:710-716`）。多团队使用前必须补用户/角色/资源级授权。
5. **攻击面放大（中高）**：`/api/workspace/*` 可读写整个数据目录（有逃逸防护但仍是高权限面，`server/workspace.go:29-38`）；自定义工具 `kind=shell/command/script` 等于任意命令执行入口（`server/customtool.go:32-64`），当前只靠 JWT + 审批规则约束；远程 MCP 可配任意 URL/Header（`mcphttp/client.go`）。多用户化后都是越权原语。
6. **可移植性/技术债（中）**：`docs/` 仅 1 篇、无 migration 版本目录、路由与 handler 分散、`cmd/artex/main.go` 注释过期、`skills/scopesentry` frontmatter 畸形、README 三处仓库地址不一致。上手成本主要在读 3900 行 `server/server.go` 与 2054 行 `agent/tools.go`。

### 8.4 机会点

- **把「审批门 + 审计 activity + 证据 blob」产品化为跨域能力**：抽出 `domain` 维度（pentest/codeaudit/reverse/compliance），让 intercept 规则、activity、evidence 都带 domain 标签——这是 ARTEX 最强、最可复用的部分。
- **图模型泛化**：在 `assets.type` 与 `exploration_nodes.kind` 之外引入 `artifact`（代码仓/文件/样本）与 `control`（合规控制项），或新增平行图用 anchors 关联；覆盖度视图可复用。
- **工具层优先 MCP/自定义工具**：Semgrep/SonarQube/IDA/Ghidra 大多可经 MCP 或 `kind=command|script` 接入，无需改 Go 即可先跑通 PoC，稳定后再沉淀为内置 `ToolSet`。
- **技能包是最低成本的域知识载体**：`skills/` 已证明「文件系统 + frontmatter + 按 agent 可见性」可用，可先以 skill 交付审计规则/逆向流程/合规清单，再逐步工具化。
- **报告/合规模块需独立模板引擎**：现 `report/report.go` 只输出漏洞报告；合规/HW 交付物需多模板（等保/OWASP/自定义），建议新增模板表 + 渲染器，而不是继续在 Go 里拼字符串。

---

## 9. 一页速查

- 单二进制 = `artex`（前端 `go:embed`）；端口 **8787**（UI/API/SSE）、**8788**（MITM，仅 loopback）。
- 主库 **PostgreSQL**（幂等 `db/schema.sql`），SQLite 仅流量索引。
- 6 个内置 agent（goals/planner/worker/mainagent/auto/pentest）+ 运行时 reporter；定义在 `db/db.go:188`、提示词 `agent/promptcatalog.go:110`、工具绑定 `agent/toolcatalog.go:36`。
- 243 条 API 路由集中在 `server/server.go:655-935`；JWT 单用户、无 RBAC（`server/auth.go`）。
- 审批 = 规则(allow/deny/ask) + pending 人工 + 超时 + LLM judge（`intercept/`、`guard/`）。
- 留痕三件套 = `activity`（过程）+ `server_logs`（日志）+ `evidence`（SHA-256 blob）。
- 扩展新域最短路径：**skill/MCP 先跑通 → 自定义工具补能力 → 新增 schema 实体与工具 → 新 agent/提示词 → 新页面/导航 → 最后动前端图与报告模板**。
