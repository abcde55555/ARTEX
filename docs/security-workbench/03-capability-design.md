# 03 · 能力层设计：基于 ARTEX 的四域统一安全工作台

> **定位**：能力层（capability layer）设计，不是实现方案。目标是把「渗透测试 / 代码审计 / 逆向分析 / 合规检测」四类工作负载，落到 ARTEX 已有的 agent / tool / skill / 双图 / findings / evidence / guard / report 模型上，给出每域的能力边界、接口、数据模型、UI 与验收标准。
> **代码基线**：本地 clone `./ARTEX`，`main @ a9d11cb`（Go + 内嵌 Next.js + PostgreSQL）。全文 `file:line` 均指该基线，未经运行验证的推断标「待核实」。
> **CodeScan-Artex**：本地 `./CodeScan-Artex`（HEAD `219775b`）已完整取得，其代码审计能力（SCA Audit：只读文件工具面、`javadeps` 本地 Java 依赖反编译、多账号/部门可见性、复核报文）的实测差异见 [02-codescan-delta.md](02-codescan-delta.md)；本文按「接入位」设计，与 02 的实测事实互为补充。
> **边界**：本文只做设计，不含实现；未安装依赖、未运行应用。

---

## 0. 结论摘要

1. **不新增第二套引擎**。ARTEX 的「目标(goal) → 意图(intent) → 事实(fact) → 漏洞(finding) → 证据(evidence) → 报告」闭环本身与域无关（`db/constants.go:6-15`、`db/schema.sql:170-285`）。四域差异应通过**能力画像（capability profile）**注入：工具集、提示词变体、资产类型、证据类型、报告模板、guard 规则集、验收门。
2. **三处必须新建的公共底座**：① `tasks.capability` + `capability_profiles`（当前任务表 `db/schema.sql:505` 无任何能力/域字段，`task_templates` 只是文本预设 `db/schema.sql:613`）；② 统一 `evidence_artifacts`（现有证据存储只服务 HTTP 流量快照 `db/schema.sql:1146-1178` + `evidence/store.go:62`，无法承载代码片段/反编译/配置核查证据）；③ finding 的 CWE/CVE/CVSS/control 映射列（现有 `findings` 仅 `vulnclass/name/severity/summary/evidence` `db/schema.sql:1081-1105`）。
3. **四域落地成本差异极大**：渗透「已具备、补治理」；代码审计「复用度最高、P0 首选」；合规「复用度高、模板与判定规则是主要工作量」；逆向「复用引擎、但要新建沙箱与二进制工具链，成本最高」。
4. **最大风险不是技术，是授权与许可**：上游 README 明确「仅供个人学习、代码研究与本地技术验证，不得用于对任何线上系统发起实际测试」（`README.md:490`），与商业化渗透工作台直接冲突；AGPL-3.0 网络服务条款（`README.md:484-486`）；且 guard 层的 RoE/授权范围机制**已被上游移除**（`guard/guard.go:5-7`）——可信工作台必须自建 scope 强制。

---

## 1. 能力矩阵

### 1.1 ARTEX 现有能力 → 四域映射

状态标签：**已具备** / **部分具备** / **缺失**（差距一句话）。

| # | ARTEX 现有能力（代码位置） | 渗透测试 | 代码审计 | 逆向分析 | 合规检测 | 差距说明 |
|---|---|---|---|---|---|---|
| 1 | 多 agent 引擎：goals/planner/worker/mainagent（`db/db.go:191`、`server/task_llm.go:564,574`） | 已具备 | 部分具备 | 部分具备 | 部分具备 | 角色固定按 `planner/worker/...` 取提示词（`agent/prompt.go:13,38`），无按域切换机制 |
| 2 | 双图：资产图 + 探索图（`db/schema.sql:40,170`） | 已具备 | 部分具备 | 部分具备 | 部分具备 | 资产图只有 6 种网络类型（`db/schema.sql:42-44`），无代码/样本/控制项节点 |
| 3 | 意图/fact/血缘与冷压缩（`db/constants.go:6-15`、`db/schema.sql:248-285`） | 已具备 | 部分具备 | 部分具备 | 部分具备 | fact 语义未按域区分（代码结论/二进制结论/控制项判定），可复用但需 `stage` 标签 |
| 4 | findings 独立表 + 状态机 + 复测（`db/schema.sql:1081,1112`、`db/findings.go:20-63`） | 已具备 | 部分具备 | 部分具备 | 缺失 | 无 CWE/CVE/CVSS/control 映射，无代码/二进制定位字段 |
| 5 | 流量证据快照 + 内容寻址 blob（`db/schema.sql:1146`、`evidence/store.go:33,62`） | 已具备 | 缺失 | 缺失 | 缺失 | 证据模型绑定 HTTP 流量，不能存代码片段/反编译/配置输出 |
| 6 | 报告生成（`report/report.go:49`）+ 多格式导出（`server/server.go:2101` `findingsExport`） | 已具备 | 部分具备 | 部分具备 | 部分具备 | 报告标题/章节固定为「渗透测试报告」，无域模板 |
| 7 | guard + intercept 审批（`guard/guard.go:60,88`、`db/schema.sql:1033,1053`） | 部分具备 | 部分具备 | 部分具备 | 部分具备 | 通用拦截可用；**RoE/授权范围已移除**（`guard/guard.go:5-7`），无域规则集 |
| 8 | 内置破坏性命令 deny 种子（`db/db.go:348-411`） | 已具备 | 部分具备 | 部分具备 | 部分具备 | 只覆盖系统破坏，不覆盖「扫描第三方/执行样本」等域风险 |
| 9 | 工具目录 + 按 agent 绑定 + DB 改描述/schema（`agent/toolcatalog.go:16-31,107`） | 已具备 | 已具备 | 已具备 | 已具备 | 机制域无关，可直接复用（新工具按此注册） |
| 10 | 自定义工具 command/script/http/shell（`server/customtool.go:16-20,68-73`） | 已具备 | 已具备 | 已具备 | 已具备 | 适合包扫描器，但不能实施权限/沙箱控制 |
| 11 | MCP 接入 + 工具缓存（`db/schema.sql:833,864`） | 已具备 | 已具备 | 已具备 | 已具备 | 适合 Ghidra/CodeQL 常驻服务，需按域可见性配置 |
| 12 | Skills（SKILL.md + 可解锁 MCP，`norma/skill`、`skills/`） | 部分具备 | 部分具备 | 缺失 | 缺失 | 只有 api-recon / playwright-cli / scopesentry 三个，无审计/逆向/合规技能包 |
| 13 | 提示词可编辑/版本化（`db/schema.sql:804-831`） | 已具备 | 部分具备 | 部分具备 | 部分具备 | 按 agent key 存储，无 capability 维度 |
| 14 | Agent 触发器 on_finding 等（`db/schema.sql:993`） | 已具备 | 已具备 | 已具备 | 已具备 | reporter 先例可复用于审计复核/合规报告 agent |
| 15 | 工作空间 + 上传（`server/workspace.go:22-23,31`、`server/chatupload.go`） | 部分具备 | 部分具备 | 部分具备 | 部分具备 | 上传上限 512 MiB / 对话 128 MiB；无仓库/样本项目概念，无解压 |
| 16 | 资产同步 ScopeSentry（`server/server.go:872-876`） | 已具备 | 缺失 | 缺失 | 缺失 | 仅网络资产数据源 |
| 17 | 任务范围/覆盖度/覆盖图（`db/schema.sql:738`、`server/server.go:702-707`） | 已具备 | 缺失 | 缺失 | 缺失 | scope 是覆盖度分母，**不是访问控制**（接口 `server/server.go:717-719`、实现 `server/server.go:1825,1845`），代码/控制项无覆盖度 |
| 18 | 漏洞复测 retester（`db/schema.sql:1112`、`server/finding_retests.go`） | 已具备 | 部分具备 | 缺失 | 缺失 | 代码修复验证 / 整改复核需换验证方式 |
| 19 | LLM 多 profile/轮询/限流/token 统计（`db/schema.sql:362`、`server/task_llm.go:500-533`） | 已具备 | 已具备 | 已具备 | 已具备 | 无预算硬门（成本风险见 §5） |
| 20 | 执行隔离/沙箱 | 缺失 | 缺失 | 缺失 | 缺失 | 工具与宿主同容器执行（`docker-compose.yml` 单 artex 服务、`Dockerfile:59-70`），无 nsjail/bwrap/firejail |
| 21 | 任务能力画像/域选择 | 缺失 | 缺失 | 缺失 | 缺失 | `db/schema.sql:505` 无 capability 列；建任务只有描述/目标/企业/模板 |
| 22 | 单域外工具链（nmap/curl/ripgrep 等基础件，`Dockerfile:45`） | 部分具备 | 缺失 | 缺失 | 缺失 | 镜像未装 semgrep/trivy/gitleaks/jadx/rizin/Ghidra/OpenSCAP 等 |

### 1.2 CodeScan-Artex（已核实，详见 02）

CodeScan-Artex（自称 **SCA Audit**）是 ARTEX 的二开分支，把「审计一个代码仓库」拆成可并行的意图，由多 agent 只读源码后写入同构的资产图/探索图；核心增量是**只读工具面 + 本地 Java 依赖反编译 + 复核报文 + 多账号/部门可见性**。

| 能力（实测） | 代码审计 | 其他三域 | 备注 |
|---|---|---|---|
| 只读静态审计（7 个本地文件工具，物理移除 Bash/网络/MCP） | 已具备 | 逆向：思路可复用 | `agent/codeaudit_tools.go`、`server/codeaudit_toolsurface.go` |
| 本地 Java 依赖反编译（`javadeps/`：GAV 解析、pom、本地仓库、反编译、物化） | 已具备 | 逆向：部分复用 | 端到端链路见 02 |
| 独立复核 / 复测报文（`finding_packet`，三态结论） | 已具备 | 不适用 | 可对接 ARTEX 现有 `finding_retests` |
| 多账号 / 部门可见性 / 管理员专属区 | 已具备 | 合规：权限模型可复用 | `db/users.go`、`server/authz.go`、`server/policy.go` |
| Semgrep / CodeQL / Trivy / Gitleaks | 未见内置集成（以 LLM 读源码为主） | 不适用 | **不要**假设已有扫描器编排；接入需新建（见 §2.2） |
| 构建可复现性 | 风险 | — | 未 vendored `norma`（ARTEX 有 `replace ./norma`），见 02 风险节 |

> 建议：CodeScan-Artex 作为**只读审计能力与权限模型的移植来源**，不作为独立引擎接入；其结果统一转成 ARTEX `findings` + `evidence_artifacts`（见 §3），避免双引擎双数据模型。

### 1.3 缺口归纳

- **框架级**：capability profile（域选择 + 工具/提示词/报告/guard 注入）、capability 维度提示词覆盖、按域工作流阶段（stage）。
- **数据级**：代码/二进制/合规资产类型与层级、非流量证据、finding 标准字段、报告制品与模板。
- **工具级**：代码审计工具链、逆向工具链与隔离执行器、合规基线核查工具、域专属 skill 包。
- **UI 级**：能力选择器、代码审计项目视图、样本分析视图、合规控制项/证据矩阵视图、统一证据抽屉。
- **治理级**：scope 强制、域 guard 规则集、授权材料与时效、证据完整性与签名、LLM 数据外发控制。

---

## 2. 四域能力设计

> 统一约定：每域都复用 §3 的**统一资产/证据/报告模型**；「新增工具接口」按 ARTEX 现有注册风格给出（`agent/tools.go:280-320` 的 `readTool/writeTool/readExpTool/writeExpTool` + `agent/toolcatalog.go:16-31` 的 seed + 按 agent 绑定），T1=内置 Go 工具、T2=Bash/skill 封装的 OSS 工具、T3=MCP 服务、T4=自定义工具。

### 2.0 公共框架：能力画像（capability profile）

四域共用一个引擎的前提，是把「域差异」收敛成一张可配置的画像表；否则只能复制四套 agent/工具/报告。

| 画像字段 | 说明 | 注入点（现有机制） |
|---|---|---|
| `key / name / enabled` | `pentest / code_audit / reverse / compliance` | 新建 `capability_profiles` 表；`tasks.capability_key`、`task_templates.capability_key` 外键 |
| `asset_spaces / asset_types` | 该域允许的资产 `space` 与 `type` 白名单 | 资产写入校验与 UI 资产页签（`db/schema.sql:40`） |
| `tool_keys[]` | 该域默认开放的工具集合（内置/MCP/skill/自定义） | 复用 `tools.agents` 绑定矩阵（`db/schema.sql:935-946`）+ `ToolResolve`（`agent/toolcatalog.go:107`），按任务 capability 二次过滤 |
| `prompt_pack` | 该域 planner/worker/goals/mainagent 提示词变体 | `agent/prompt.go:13,38` 的 `PromptOverride` 解析顺序扩为 `capability/role → role → 内置`；新表 `capability_prompts` 或 `agent_prompts` 加 `capability` 列 |
| `workflow_stages[]` | 阶段枚举（recon/analysis/verify/evidence/report 等） | intent payload 增加 `stage`（复用 `exploration_nodes.payload`），任务详情与报告按阶段聚合 |
| `guard_ruleset` | 该域 intercept 规则集 | 复用任务级规则快照（`task_templates.intercept_rules` `db/schema.sql:622`）与 `asset_intercept_rules`（`db/schema.sql:1241`），建任务时按画像灌入 |
| `report_template / evidence_kinds` | 报告模板与证据类型白名单 | 新建 `report_templates`（§3.4）+ `evidence_artifacts.kind` 校验 |
| `acceptance_gate` | 该域验收门（代码审计必须有 file:line、合规每项必须有证据等） | 复用 agent 触发器（`db/schema.sql:993`）与 finding 校验钩子 |

> 这套画像是 P0 第一优先级：它把「四域 = 四套系统」降级为「一套引擎 + 四张配置」，是后续所有域扩展的公共前置。

### 2.1 渗透测试（已具备，补治理与工具链）

- **目标输入**：域名 / IP / CIDR / URL / 企业主体 / ICP / 关键词（`task_scope` 7 类 `db/schema.sql:738`；`company_scope` 5 类 `db/schema.sql:115`）；建任务时从描述解析（`server/server.go:1667-1706`）、手填、ScopeSentry 同步、MCP / 对话上传。
- **工作流阶段**：范围确认 → 侦察（资产/指纹/端口） → 攻击面分析 → 利用与验证 → 证据固化 → 复测 → 报告。现有意图循环已覆盖主体（`agent/planner.go` / `agent/worker.go`），P0 只需给 intent 增加 `stage` 标签并写入模板，使 UI 与报告能按阶段聚合。
- **Agent 角色与提示词要点**：goals 拆目标并抽操作约束（`agent/constraints.go:14`，注入 planner/worker 系统提示 `agent/planner.go:394`、`agent/worker.go:397`）；planner 只派意图、维护共享 todolist；worker 单意图执行；mainagent 人机接口；reporter 由 `on_finding` 触发器写详细报告（先例 `server/orchestration.go:735` 的 `seedReporterAgent`）；retester 复测。提示词要点：授权与范围声明置于最高优先级（复用 constraintBlock 文案）；**只用真实触发结果上报 finding**（现有 pentest 提示词已含该纪律 `agent/promptcatalog.go:31`）；禁止把版本号/CVE 命中当漏洞。
- **工具与集成**：T2 为主——Bash 调 `nmap`（镜像已装 `Dockerfile:45`）、`curl`，经 MITM 记录代理留痕（worker `SetProxy`，`server/task_llm.go:571`）；T3 Playwright MCP（浏览器）、ScopeSentry MCP；skill `api-recon`、`scopesentry`。需新增 `httpx / ffuf / nuclei / sqlmap` 到镜像或 runner（当前镜像**未装**，属缺口）。
- **新增工具接口**（P0/P1）：
  | 工具 | 入参（要点） | 输出 | 说明 |
  |---|---|---|---|
  | `scope_check`（T1，read） | `target`(url/ip/domain/cidr)、`task_id?` | `in_scope`、`matched_rule`、`reason` | 供 agent 自检 + 被 guard 复用为强制门 |
  | `http_request`（T1，write） | `method,url,headers,body,timeout_ms,follow_redirects` | `status,headers,body_preview,traffic_id` | 结构化重放/取证，自动进 evidence；替代裸 curl 便于证据统一 |
  | `scan_recon`（T1，write，异步） | `tool`(httpx/ffuf/nuclei/nmap)、`targets[]`、`options{}`、`severity?` | `job_id`、归一化结果数组（host/url/port/template/cve） | 提交 runner 执行，结果转 fact/asset |
  | `report_scope_attestation`（T1，write） | `task_id`、`authorization_ref`、`expires_at` | `ok` | 记录授权材料引用，任务启动前强校验 |
- **数据模型扩展**：`tasks.capability='pentest'`；finding 增加 `cwe_id/cve_ids/cvss_vector/cvss_score/confidence/location`；新增 `task_authorizations`（授权材料、范围快照、有效期）；`evidence_artifacts` 承载请求/响应与命令输出。
- **工作台 UI 变化**：建任务增加「能力选择」；任务详情 TABS（`web/src/app/(main)/function/tasks/detail/page.tsx:53-64`）在渗透下保持现状，新增「范围与授权」tab；findings 详情增加 CVSS/CWE/CVE 与证据矩阵；报告 tab 增加「范围声明/授权声明/复测结论」。
- **产出物**：渗透测试报告（沿用 `report/report.go:49` 并做模板化）、证据包（现有 md-zip/csv/json 导出，`web/src/app/(main)/function/findings/page.tsx:1236-1245`）、范围与授权声明、复测记录。
- **安全与合规控制**：scope 强制（新增 PreToolUse 校验，位置与 `guard/guard.go:60,88` 同级，且对 Bash/扫描/HTTP 全部生效）；破坏性动作沿用 `ask/deny` 种子（`db/db.go:348-411`）；速率限制用 `llm_profiles.rate_per_second/minute`（`db/schema.sql:371-372`）；审计日志 `guard.Audit`。
- **验收标准**：① 越界目标被拒绝率 100%（含 Bash 直连、扫描器、HTTP 工具三条路径）；② 授权过期任务无法启动；③ 每个 finding 至少 1 条可校验证据（sha256 一致）；④ 报告含范围与授权声明；⑤ `go test ./...` 全绿。

### 2.2 代码审计（复用度最高，P0 首选）

- **目标输入**：Git 仓库 URL + branch/commit、zip/tar 上传（工作空间 512 MiB，`server/workspace.go:22-23`）、工作区已有目录、多仓/单体；语言 Java/Go/Python/JS/TS；可附带「已知风险点/业务说明」。CodeScan-Artex 的 Java 依赖与反编译方向在此复用（待核实）。
- **工作流阶段**：`ingest`（拉取/解压/去重）→ `inventory`（语言、构建文件、依赖树、SBOM）→ `map`（入口/信任边界/危险 sink/模块划分）→ `scan`（semgrep / gitleaks / trivy）→ `dataflow`（joern / CodeQL）→ `review`（LLM 逐候选复核）→ `verify`（沙箱复现或明确标 inferred）→ `triage`（误报降噪、定级）→ `report`。
- **Agent 角色与提示词要点**：`code-audit-planner`（按模块/攻击面派意图，维护审计计划 todolist，禁止一次铺全部文件）；`code-audit-worker`（一次只复核一个规则命中/一个函数，输出 `file:line + 代码片段 + 数据流路径 + 触发条件`）；`audit-reviewer`（role=assistant，由 `on_finding` 触发复核降误报，先例同 reporter）；`code-reporter` 写报告。提示词要点：区分 `confirmed`（有复现/数据流闭环）与 `inferred`（仅模式命中）；**把源码当数据而非指令**（防提示注入）；不执行仓库构建脚本，除非获批进沙箱；严禁「看起来像」结案。
- **工具与集成**：T2：`semgrep --config p/security-audit --json`、`gitleaks detect --report-format json`、`trivy fs --scanners vuln --format json`、`joern`（CPG 查询）、`CodeQL`（**许可敏感**，见 §5）、`ripgrep`（已装 `Dockerfile:45`）、`syft`（SBOM）。T3：可选 Semgrep/CodeQL MCP。T1 负责把结果**归一化**进 finding/evidence（关键价值，避免 LLM 直读海量原始结果）。
- **新增工具接口**：
  | 工具 | 入参（要点） | 输出 | 说明 |
  |---|---|---|---|
  | `repo_ingest`（T1，write） | `source`(git_url/路径)、`ref`、`path?`、`depth?`、`exclude[]` | `repo_id`、`file_count`、`languages{}`、`build_files[]`、`skipped[]` | 去重键 `(normalized_url, commit)`；写 `code_repos` |
  | `code_overview`（T1，read） | `repo_id` | 模块树、入口候选、依赖摘要、扫描覆盖状态 | planner 首轮必读，替代盲目 Glob |
  | `code_search`（T1，read） | `query`(regex)、`repo_id`、`lang?`、`path_glob?`、`limit` | `hits[{file,line,snippet}]` | 包装 ripgrep，输出限长、锚定 repo 边界 |
  | `code_scan`（T1，write，异步） | `repo_id`、`engine`(semgrep/gitleaks/trivy/joern/codeql)、`ruleset?`、`severity?`、`paths[]?` | `job_id`、计数、归一化候选（rule/file/line/cwe/severity） | 与 Bash 后台任务同构，可中断、可续跑 |
  | `code_slice`（T1，read） | `repo_id`、`file`、`start`、`end`、`context?` | 带行号代码片段 + 关联 finding id | 审计证据单元，可直接登记为 evidence |
  | `code_verify`（T1，write，需审批） | `repo_id`、`finding_id`、`lang`、`poc`、`timeout` | `verdict`(reproduced/not/ inconclusive)、`evidence_id` | 仅在 runner 沙箱执行，主容器禁止 |
- **数据模型扩展**：资产新增 `space='code'` 与类型 `repo/file/function/dependency`（层级 repo→file→function，依赖挂 repo；`parent_asset_id`）；新增 `code_repos`（任务内仓库状态、commit、路径、SBOM 引用）；新增 `scan_jobs`（引擎、参数、状态、结果统计、产物路径）；finding 增加 `cwe_id`、`location{repo,file,line,symbol}`、`remediation`、`confidence`；evidence 新增 `kind=code_slice/scan_report/sbom`。
- **工作台 UI 变化**：新增「代码审计」工作台（`/function/code-audit`：项目列表、仓库概览、模块树）；任务详情条件 tab：`代码地图 / 依赖 SBOM / 规则结果 / 数据流`；代码查看器支持 finding 定位与行高亮；findings 列表支持「按 CWE / 按文件」视图；证据抽屉可看代码片段与扫描日志。
- **产出物**：SARIF/JSON 兼容结果、逐 finding 的代码证据与修复建议（diff 形式）、SBOM 与依赖风险清单、审计报告（OWASP Top 10 / CWE 映射、按严重度排序）。
- **安全与合规控制**：仓库内容视为不可信输入（防提示注入、防仓库内恶意脚本）；**默认禁止执行构建脚本**（gradle/npm/go generate 会执行攻击者代码），需审批并进 runner；路径限制在工作区（`wsResolve` 已具备，`server/workspace.go:31`）；上传解压需防 zip bomb（新代码自实现，当前无解压逻辑）；`gitleaks` 结果脱敏；代码外发 LLM 需 DLP/本地模型开关（`llm_profiles.base_url` 可指向本地推理服务，`db/schema.sql:366`）。
- **验收标准**：① 已知漏洞基准（如 OWASP Benchmark/Juliet 子集）召回率与误报率达到约定阈值；② 100% finding 含 `file:line` + 代码片段 + CWE；③ 扫描任务可中断/续跑且结果幂等；④ 10 万文件仓库 ingest 与首轮扫描在约定时限内；⑤ 报告可导出 MD/JSON/SARIF。

### 2.3 逆向分析（引擎可复用，工具链与沙箱为新建）

- **目标输入**：APK/AAB/jar/dex、ELF/so、PE/exe/dll、Mach-O、固件（bin/img/ubi）、内存 dump；上传或工作区路径；样本 sha256 去重；可选关联 CVE/家族/情报。
- **工作流阶段**：`ingest`（哈希、格式识别、熵、壳、证书/签名）→ `triage`（strings、符号、导入表、节区、权限/组件）→ `decompile`（移动端 jadx/apktool；原生 Ghidra/rizin）→ `structure`（函数边界、xref、数据结构恢复）→ `behavior`（网络/文件/加密/权限/持久化）→ `dynamic`（Frida/模拟器，隔离）→ `assess`（恶意性/漏洞/ATT&CK）→ `report`（IOC + 缓解）。
- **Agent 角色与提示词要点**：`reverse-planner`（按样本/架构/组件派意图，维护分析计划）；`reverse-worker`（一次一个函数/一个组件，输出 `地址/偏移 + 反编译片段 + 结论`）；`malware-assessor`（assistant，恶意性与家族判定，保守结论）；`reverse-reporter`（IOC/ATT&CK/缓解）。提示词要点：**静态推测 vs 动态验证必须标注**；结论必须带地址/函数定位；禁止在主容器执行样本；Ghidra/rizin 输出限长分页，避免撑爆上下文。
- **工具与集成**：T2：`file/strings/readelf/objdump/nm`（binutils）、`rizin/radare2`、`Ghidra headless analyzeHeadless`、`jadx`、`apktool`、`binwalk`（固件）、`upx -d`、`capa`（能力识别）、`yara`（规则扫描）、`Frida`（动态，仅 runner）、`angr`（P2 符号执行）。T3：Ghidra MCP（长驻工程、反编译缓存）。**全部为镜像新增**（当前镜像无这些工具）。
- **新增工具接口**：
  | 工具 | 入参（要点） | 输出 | 说明 |
  |---|---|---|---|
  | `binary_ingest`（T1，write） | `path`/上传 id、`format_hint?`、`sha256?` | `sample_id`、`format/arch/endianness`、`entropy`、`packer?`、`hashes{}` | 去重键 sha256；写 `binary_samples` + 资产节点 |
  | `binary_triage`（T1，write，异步） | `sample_id`、`engines[]`(strings/symbols/sections/capa/yara) | 结构化摘要 + 证据引用 | 避免 LLM 直接读海量字符串 |
  | `binary_decompile`（T1，write，异步） | `sample_id`、`target`(函数名/vaddr/offset)、`engine`(ghidra/rizin)、`depth?` | 伪代码/反汇编文本 + 证据 id | 提交 runner；结果缓存到 `binary_functions` |
  | `binary_xrefs`（T1，read） | `sample_id`、`target`、`direction?` | 引用列表（地址/函数/调用点） | 结构恢复用 |
  | `mobile_manifest`（T1，read） | `sample_id` | 组件、导出属性、权限、SDK、证书 | APK/AAB（jadx/apktool 结果归一化） |
  | `dynamic_trace`（T1，write，需审批+隔离） | `sample_id`、`script`、`duration`、`emulator` | 调用/网络/文件事件时间线 + 证据 id | 仅 runner；默认关闭 |
- **数据模型扩展**：资产新增 `space='binary'` 与 `sample/section/function/symbol/string` 类型；新增 `binary_samples`（sha256 唯一、格式、架构、存储路径、隔离级别）；新增 `binary_functions`（地址、名称、伪代码缓存、人工注释）+ `binary_notes`（人工重命名/结论）；finding 增加 `location{offset,symbol,function}`；evidence 新增 `decompiled/cfg/strings/trace`。
- **工作台 UI 变化**：样本库页（哈希/格式/家族/风险）；样本详情：概览、字符串、符号/导入、反编译视图（函数树 + 伪代码 + xref 跳转）、动态时间线、IOC 列表、报告；支持人工标注/重命名（写入 `binary_notes`）；任务详情条件 tab：`样本分析 / 反编译 / 动态行为`。
- **产出物**：逆向分析报告（哈希、格式、行为、IOC、ATT&CK 映射、CVE 指纹、缓解建议）、证据包（反编译片段、字符串导出、trace、哈希清单）、可选 YARA 规则。
- **安全与合规控制**：样本**绝不在主容器执行**；runner 无网络、只读挂载、CPU/内存/时间限额、每次运行留哈希与命令日志；静态分析为默认，动态需人工审批；样本不上传第三方（如 VirusTotal）除非显式授权与告知；样本存储加密、访问审计（恶意样本本身即敏感资产）。
- **验收标准**：① 样本 sha256 去重与哈希记录 100%；② 每条结论含地址/函数定位且可回放证据；③ 隔离验证：runner 内样本运行无外网、无宿主写权限；④ 常见 APK 壳与 ELF 加壳样本可产出 triage 结论；⑤ 报告含 IOC 与 ATT&CK 映射。

### 2.4 合规检测（复用度高，规则与模板是主要工作量）

- **目标输入**：系统描述与边界、定级（等保二级/三级等）、标准选择（等保 2.0 / 密评 / 数据安全法·PIPL / 可选 GDPR·PCI-DSS）、资产清单（可从资产图带入）、配置导出/截图/日志/访谈记录、管理制度文档。
- **工作流阶段**：定级与范围 → 基线控制项生成 → 证据采集（自动核查 + 人工上传 + 访谈）→ 逐项判定 → 差距与整改建议 → 评分 → 报告 → 整改复核。注意：合规的「finding」是**控制项偏差**，不是渗透漏洞。
- **Agent 角色与提示词要点**：`compliance-assessor`（assistant，主导，按控制域派意图）；`control-collector`（worker 变体，一意图一控制域，采集证据并登记）；`compliance-reporter`（生成测评报告）；复用 `on_finding` 触发器做报告生成。提示词要点：**只依据已登记证据判定，不得臆造**；必须写标准版本 + 条款/控制项编号 + 判定 + 理由 + 证据引用；明确自动核查与人工证据的边界；「不适用/无法获取」必须显式说明。
- **工具与集成**：T2：`OpenSCAP`（XCCDF 基线核查，输出 ARF/XML→归一化）、自研/开源等保检查脚本（**许可待核实**）、配置核查 shell/ansible、`openssl`（密码算法套件、证书有效期、密钥长度）、TLS 配置扫描（`testssl.sh` 类）、日志审计项核查。T1 负责把结果映射到控制项与证据。T4 自定义工具适合快速接客户自有检查脚本。
- **新增工具接口**：
  | 工具 | 入参（要点） | 输出 | 说明 |
  |---|---|---|---|
  | `assessment_create`（T1，write） | `standard`(等保2.0/密评/DSL/PIPL/GDPR/PCI)、`level?`、`system_desc`、`boundary`、`asset_ids[]?` | `assessment_id`、控制项总数 | 生成基线控制项快照 |
  | `control_list`（T1，read） | `assessment_id`、`domain?`、`verdict?`、`page` | 控制项树（编号/要求/检查方法/状态） | 供 planner 与 UI 共用 |
  | `evidence_capture`（T1，write） | `assessment_id`、`control_id`、`kind`(cli_output/config_dump/screenshot/doc/interview)、`content`/文件引用、`note` | `evidence_id` + 哈希 | 统一证据登记（复用 `evidence_artifacts`） |
  | `control_assess`（T1，write） | `control_id`、`verdict`(符合/部分符合/不符合/不适用)、`rationale`、`evidence_ids[]`、`severity?` | 更新结果 + 生成控制项 finding | 判定需人工复核标记 |
  | `compliance_report`（T1，write） | `assessment_id`、`template`、`include_evidence?` | `report_artifact_id`、得分摘要 | 固定证据版本，防报告漂移 |
- **数据模型扩展**：新增 `compliance_assessments`（标准/版本/级别/边界/总分/结论）与 `compliance_controls`（`assessment_id`、`standard`、`standard_version`、`level`、`domain`、`item_code`、`requirement`、`check_method`、`verdict`、`weight`、`rationale`、`evidence_ids[]`、`reviewer`）；资产新增 `space='compliance'` 与 `system/control-item` 类型（用于图谱与覆盖度）；finding 增加 `type='control_gap'` + `control_refs`。
- **工作台 UI 变化**：合规项目页 `/function/compliance`（项目/标准/进度）；控制项清单（按「技术/管理」层面 + 条款树，显示判定、证据、责任人、复核状态）；**证据矩阵**（控制项 × 证据，可一键查看原始证据）；得分看板；报告生成与预览；任务详情条件 tab：`控制项 / 证据 / 差距与整改`。
- **产出物**：控制项符合性矩阵、差距分析与整改建议、测评报告（等保 / 密评模板）、证据包（含哈希清单与版本号）、整改复核记录。
- **安全与合规控制**：证据可能含敏感配置/个人信息 → 加密存储、脱敏、最小权限、保留期限（`settings` 配置）；判定必须有人工复核字段与操作留痕；报告固定证据版本（现有 `findings.evidence_version` 思路可复用，`db/schema.sql:1143-1144`）；显式声明「技术辅助，不构成法律意见，也不替代具备资质的测评机构出具的正式结论」。
- **验收标准**：① 所选标准/级别的控制项覆盖 100%；② 每项判定有 ≥1 证据或显式「不适用/无法获取」说明；③ 报告含条款编号、判定、证据引用、得分与整改建议；④ 人工复核与修改留痕可追溯；⑤ 得分计算规则与标准一致并可复算。

---

## 3. 统一资产 / 证据 / 报告模型建议

### 3.1 统一资产模型（在现有 `assets` 上扩展，不推翻）

原则：**资产图仍是唯一图节点表**（`db/schema.sql:40-102`），按 `space` 分区扩展，域内重状态放域扩展表并回挂 `asset_id`；现有 6 种网络类型与唯一索引保持不动（向后兼容）。

| 域 | 资产层级（节点） | 现有/新增 | 关键字段 |
|---|---|---|---|
| 网络 | root_domain → subdomain → ip → service → app → endpoint | 现有 | 沿用现有列与唯一索引 |
| 代码 | repo → file → function；repo → dependency | 新增 type + `space='code'` | `parent_asset_id`、`locator`（path/line/symbol）、`hash`、`meta` |
| 二进制 | sample → section → function → symbol/string | 新增 type + `space='binary'` | `hash`(sha256)、`locator`(offset/vaddr)、`parent_asset_id` |
| 合规 | system → control-domain → control-item | 新增 type + `space='compliance'` | `locator`(standard+item_code)、`meta`(verdict/weight) |

建议新增列：`assets.space`、`assets.parent_asset_id`、`assets.hash`、`assets.locator JSONB`（复用现有 `extra JSONB` 兜底也可，但显式列更利于查询）。唯一索引按 `type` 建部分索引（沿用现有 `uq_av2_*` 风格，新增 `uq_av2_repo`(space,hash)、`uq_av2_sample`(hash)、`uq_av2_control`(system,standard,item_code)）。

### 3.2 统一 finding schema（在现有 `findings` 上扩展）

| 字段 | 类型 | 用途 | 现状 |
|---|---|---|---|
| `capability` | TEXT | pentest/code_audit/reverse/compliance | 新增 |
| `finding_type` | TEXT | vuln/control_gap/config_drift/malware_behavior/ioc | 新增 |
| `cwe_id` | TEXT | CWE 编号 | 新增 |
| `cve_ids` | JSONB | CVE 列表（仅作参考，不等于已验证） | 新增 |
| `cvss_vector` / `cvss_score` | TEXT / NUMERIC | CVSS v3.1/v4 向量与分值 | 新增 |
| `confidence` | TEXT | confirmed / inferred / needs_review | 新增（对齐 pentest 提示词纪律） |
| `location` | JSONB | 端点/仓库+文件+行/样本+偏移+函数/控制项 | 新增 |
| `control_refs` | JSONB | 标准 + 条款/控制项 + 判定（合规） | 新增 |
| `remediation` | TEXT | 修复/整改建议 | 新增（现有 `report` 是长文） |
| `severity` | TEXT | critical/high/medium/low（可扩 info） | 现有 `db/findings.go:57-63` |
| `evidence_version` | BIGINT | 证据版本固定，防报告漂移 | 现有 `db/schema.sql:1143` |

兼容策略：旧字段（`vulnclass/name/summary/evidence/asset_ids/status/worker`）全部保留；新增列可空，旧数据视为 `finding_type='vuln'`、`confidence='confirmed'`。映射规则：等保控制项偏差 → `finding_type='control_gap'` + `control_refs`；代码 → CWE + `location{repo,file,line}`；逆向 → `location{offset,symbol}` + ATT&CK（放 `meta`）。

### 3.3 统一证据模型

现状：`findings.evidence` 是文本；HTTP 证据走 `traffic_evidence_snapshots` + `finding_traffic_bindings`（`db/schema.sql:1146-1178`）；blob 内容寻址 SHA-256（`evidence/store.go:33,62`）。**问题**：代码/反编译/配置核查证据无处可放。

建议新增 `evidence_artifacts`：

| 列 | 说明 |
|---|---|
| `id, task_id, finding_id?, control_id?, assessment_id?` | 多态归属（合规证据挂控制项，代码/逆向证据挂 finding） |
| `kind` | http_traffic / cli_output / code_slice / decompiled / strings_dump / cfg / trace / config_dump / screenshot / doc / interview / hash_manifest |
| `blob_sha256, content_type, size` | 复用内容寻址存储（`evidence/store.go:62` 的落盘+校验逻辑） |
| `source_tool, tool_version, command` | 采集来源与命令，支撑复现 |
| `captured_at, collector` | 时间与采集者（agent/人） |
| `integrity` | `sha256` + `prev_hash`（追加式哈希链） |
| `note, meta JSONB` | 摘要/结构化元数据 |

迁移路径：现有流量证据保留原表，`evidence_artifacts` 以 `kind='http_traffic'` + `snapshot_id` 引用，UI 统一读 `evidence_artifacts` 视图；新证据（代码/二进制/合规）直接入新表。**完整性增强（P2）**：追加式账本 + 可选 RFC 3161 时间戳/签名，导出证据包时附带 manifest（哈希清单 + 采集命令 + 版本）。

### 3.4 统一报告模型

- 新增 `report_templates`（capability、name、format、body 模板）与 `report_artifacts`（`task_id`、`template`、`format`、`content_blob`、`pinned_finding_version`、`pinned_evidence_versions`、`generated_by`、`generated_at`、`sha256`）。
- 渲染器接口化：`report/report.go:49` 的 `Markdown(Input)` 拆为「通用骨架 + 域章节插件」；四域模板共享：范围与授权、方法与环境、资产概览、发现/控制项列表（严重度/条款排序）、证据矩阵、复现/核查步骤、修复/整改建议、复测/复核记录、附录（工具版本、LLM 使用声明）。
- 导出沿用现有 `findingsExport` 的 md-single/md-zip/csv/json 形态（`web/src/app/(main)/function/findings/page.tsx:1236-1245`），新增 SARIF（代码审计）与证据包 zip（含 manifest）。

---

## 4. 合规检测要点

### 4.1 标准映射与检查域

| 标准/法规 | 检查域（层级） | 工作台如何承载 | 自动化程度 |
|---|---|---|---|
| **等保 2.0 GB/T 22239-2019**（网络安全等级保护基本要求） | 技术要求：安全物理环境 / 安全通信网络 / 安全区域边界 / 安全计算环境 / 安全管理中心；管理要求：安全管理制度 / 安全管理机构 / 安全管理人员 / 安全建设管理 / 安全运维管理 | `compliance_controls` 按「层面 → 条款 → 控制项」三级；资产图 `system/control-item` 展示覆盖度 | 技术项可高比例自动核查（配置、日志、加密、访问控制）；管理项需人工证据（制度文档/访谈） |
| 配套：GB/T 25070-2019（设计技术要求）、GB/T 28448-2019（测评要求）、GB/T 22240-2020（定级指南） | 定级、测评方法、结论判定 | 报告模板与判定规则引用；测评结论由资质机构出具 | 辅助生成差距与证据，不替代正式测评 |
| **密评 GB/T 39786-2021**（信息系统密码应用基本要求） | 技术：物理和环境安全 / 网络和通信安全 / 设备和计算安全 / 应用和数据安全；管理：管理制度 / 人员管理 / 建设运行 / 应急处置 | 单独控制项基线；证据含算法套件、密钥长度、证书、TLS 配置、密码模块合规性 | 技术项可自动核查（openssl/TLS 扫描/配置导出）；管理项人工 |
| **《数据安全法》**（2021-09-01 施行） | 数据分类分级、重要数据目录、风险评估、数据出境、安全管理制度 | 数据资产清单 + 分类分级标注 + 处理活动记录；证据为制度文件与评估报告 | 低（以文档与访谈证据为主） |
| **《个人信息保护法》** | 告知同意、最小必要、单独同意、个人信息保护影响评估（PIA）、跨境提供、处理者安全义务 | 个人信息处理活动清单 + 同意机制证据 + PIA 文档 + 安全措施核查 | 中低（技术措施可核查，合法性基础需人工） |
| 可选：GDPR（Art.5/6/30/32/33/35）、PCI-DSS v4.0（12 项要求） | 安全措施、记录、泄露通知、DPIA；支付卡数据环境 | 作为可选「标准包」接入同一控制项模型，字段加 `standard` 即可 | 视控制项而定 |

> 判定分级与结论口径以现行标准/测评准则为准（如 GB/T 28448-2019 的单项符合性判定与综合结论），实现时按所选标准版本冻结规则，**待实现阶段核实具体条文与评分细则**。

### 4.2 证据采集与可评估报告

1. **证据类型**：命令行输出（配置核查）、配置文件快照、日志片段、截图、制度文档、访谈记录、扫描报告。每类统一入 `evidence_artifacts`，带哈希与采集命令。
2. **采集方式**：自动化脚本（OpenSCAP/自研检查脚本/openssl）+ 人工上传（工作台上传）+ agent 引导式问答（worker 生成访谈提纲并登记回答）。
3. **判定流程**：控制项 → 检查方法 → 证据 → 判定（符合/部分符合/不符合/不适用）→ 理由 → 权重 → 得分；判定必须由 `reviewer` 字段记录人工复核人。
4. **可评估报告**：每条结论可回链到条款编号 + 证据 id + 原始证据内容 + 采集时间/工具版本；报告固定证据版本（引用 `evidence_version` 思路）以避免「报告与证据不一致」。
5. **免责声明**：工作台输出为**技术辅助材料**，不构成法律意见；等保测评须由具备资质的测评机构实施，密评须由具备资质的商用密码检测机构实施；最终合规结论以主管部门/测评机构认定为准。

---

## 5. 关键风险

| # | 风险 | 影响 | 依据 | 缓解建议 |
|---|---|---|---|---|
| 1 | **AGPL-3.0 网络服务条款** | 修改后以网络服务方式提供，须向用户提供完整对应源码；衍生作品须 AGPL | `README.md:484-486`、`LICENSE`（AGPL-3.0） | 内部自用也需评估；对外服务则整体开源或取得作者商业授权；扫描器等尽量进程隔离并单独评估「独立作品」边界（需律师意见） |
| 2 | **上游使用限制** | README 明示「仅个人学习/研究/本地隔离验证，不得对任何线上系统发起实际测试」 | `README.md:490-500` | 商用/对外服务前必须联系作者取得书面许可，或对渗透能力层做 clean-room 重写；不可仅以「AGPL 允许使用」抗辩（该限制是作者额外声明，效力待法律确认） |
| 3 | **授权边界（法律）** | 未授权测试可能触犯《网络安全法》《刑法》285/286 条等 | 现有 guard 的 RoE/授权范围机制**已被移除**（`guard/guard.go:5-7`）；`task_scope` 只是覆盖度分母（`server/server.go:717-719,1825`） | P0 自建强制 scope guard（PreToolUse 对所有出网/扫描工具生效）+ 授权材料与有效期 + 双人复核 + 全量审计日志 |
| 4 | **破坏性动作** | 误删/误改靶标或宿主；逆向样本逃逸 | 内置 deny 规则只覆盖系统破坏（`db/db.go:348-411`）；Bash 层仅 `catastrophicBash` 兜底（`norma/tool/bash.go:20,61`）；SDK 权限模式为 `ModeBypass`（`agent/worker.go:436`、`agent/planner.go:412`），实际只靠 intercept/hook | 按 capability 下发规则集；高风险工具（`code_verify`/`dynamic_trace`/主动扫描）强制 ask + 超时 deny；样本执行仅 runner 且无网络 |
| 5 | **LLM 成本/延迟** | 代码审计/逆向上下文巨大，token 成本可达渗透的数倍到数十倍；长任务易超时 | 现有 `llm_profiles` 限流/轮询（`db/schema.sql:362-430`）、任务超时与心跳（`db/schema.sql:524-525`）但无硬预算门 | 分层模型（便宜模型 triage、强模型深挖，`server/task_llm.go:500-533` 已支持按 agent/任务绑 profile）；分块+摘要；每任务 token/时长硬上限；成本看板 |
| 6 | **工具许可** | 分发/商业化时违反许可证 | Ghidra(Apache-2.0)、rizin/radare2(LGPL-3.0)、jadx/apktool(Apache-2.0)、Frida(LGPL)、angr(BSD-2)、semgrep CE(LGPL-2.1)、trivy(Apache-2.0)、gitleaks(MIT)、OpenSCAP(LGPL-2.1)、nuclei(MIT)、sqlmap(GPL-2.0)、nmap(NPSL，非 OSI)、binutils(GPL-3.0)、**CodeQL CLI 为专有许可（仅限特定场景免费）** | 建立许可证清单与「内部分发 vs 对外分发」矩阵；CodeQL 改为可选 BYO 许可；nmap/sqlmap 注意再分发条款 |
| 7 | **证据完整性** | 报告与证据不一致、证据被替换或丢失，影响法律/审计效力 | 现有 SHA-256 内容寻址与版本列（`evidence/store.go:62`、`db/schema.sql:1143`）但无签名/时间戳/链式账本；流量快照可被 GC（`unreferenced_at`） | P1 追加式证据账本 + prev_hash 链；P2 可选 RFC 3161 时间戳/签名；导出 manifest；禁止对已固定证据原地覆盖 |
| 8 | **不可信代码/样本执行** | 仓库构建脚本、样本、PoC 在宿主执行 → 主机失陷；代码注释/字符串中的提示注入劫持 agent | 当前工具与宿主同容器（`docker-compose.yml`），无沙箱 | P0 禁止自动执行；P1 建 `runner` 隔离子服务（无网络/只读/限额）；源码与样本内容按「数据」处理，关闭 web 搜索与出网，worker 低权限 |
| 9 | **数据外发与保密** | 客户源码/样本/证据发送给第三方 LLM，违反保密协议或出境要求 | 现有 LLM 配置支持自定义 `base_url`（`db/schema.sql:366`），可指向本地推理 | 提供本地模型模式；按客户/任务开关「允许外发」；敏感字段脱敏；等保/金融客户默认本地化 |
| 10 | **上游维护状态** | 该仓库自称「最后一个版本纯源码备份」，无后续安全更新 | `README.md` 顶部说明 | Fork 后自维护；建立依赖升级与 CVE 响应机制；关键修复回移 |

---

## 6. 分期落地建议（能力级）

### P0 · 可信最小工作台（约 4–6 周）

目标：让「统一工作台」成立——同一入口可选四域，代码审计跑通端到端，合规能出可评估报告，渗透补上授权治理。

| 能力 | 内容 | 依赖 | 完成判据 |
|---|---|---|---|
| C0-1 能力画像框架 | `capability_profiles` + `tasks.capability` + 提示词 capability 覆盖（`agent/prompt.go:13,38` 解析顺序：capability → agent → 内置）+ 工具按 profile 绑定 + 建任务能力选择器 | 无 | 四域任务可用同一引擎启动，工具/提示词/报告按域生效 |
| C0-2 统一 finding/证据模型 | `findings` 扩列（§3.2）+ `evidence_artifacts` + 统一证据抽屉 | 无（与 C0-1 并行） | 四域 finding 均落在同一列表/详情/导出 |
| C0-3 代码审计 MVP | `repo_ingest`/`code_overview`/`code_search`/`code_scan`(semgrep/gitleaks/trivy)/`code_slice` + 审计提示词 + 代码项目视图 + 报告模板 | C0-1、C0-2 | 单仓（≤5 万文件）端到端出报告；finding 带 file:line |
| C0-4 合规 MVP | `assessment_create`/`control_list`/`evidence_capture`/`control_assess`/`compliance_report` + 等保 2.0 基线模板 + 控制项/证据矩阵视图 | C0-1、C0-2 | 等保三级模板控制项覆盖 100%，报告可导出 |
| C0-5 渗透治理 | scope guard（PreToolUse 强制）+ 授权材料/有效期 + 域规则集 + 报告增加范围声明 | 无 | 越界拒绝率 100%，授权过期不可启动 |

> P0 建议同时交付「代码审计 + 合规」作为对外演示主线：不依赖对线上目标发起测试，法律风险最低、复用度最高；渗透以内部隔离环境演示。

### P1 · 域能力补齐（约 8–12 周）

| 能力 | 内容 | 依赖 |
|---|---|---|
| C1-1 逆向 MVP | `binary_ingest`/`binary_triage`/`binary_decompile`(rizin+objdump)/`mobile_manifest`(jadx/apktool) + 样本库/样本详情 UI + 报告模板 | C0-1、C0-2、**C1-2** |
| C1-2 隔离子服务 runner | 独立 compose 服务承载扫描器/样本/PoC；无网络、只读、限额、异步 job 协议；主容器不执行不可信内容 | 无（安全前置） |
| C1-3 代码审计进阶 | joern/CodeQL 数据流 + SBOM（syft/trivy）+ `code_verify` 沙箱复现 + SARIF 导出 + 误报降噪 reviewer agent | C0-3、C1-2 |
| C1-4 渗透工具链 | httpx/ffuf/nuclei/sqlmap 接入（runner 或镜像）+ `scan_recon`/`http_request` + 主动扫描审批策略 | C0-5、C1-2 |
| C1-5 合规自动化 | OpenSCAP/配置基线/密码套件核查脚本接入 + 密评基线包 + 数据安全法/PIPL 检查清单 + 整改复核流程 | C0-4 |
| C1-6 证据完整性 | 追加式证据账本 + manifest 导出 + 报告固定版本 | C0-2 |

### P2 · 深化与规模化（约 12 周以上）

| 能力 | 内容 | 依赖 |
|---|---|---|
| C2-1 逆向动态与规模化 | Ghidra headless 常驻（MCP）+ Frida 动态跟踪 + angr 符号执行 + 函数级覆盖度视图 | C1-1 |
| C2-2 合规标准扩展 | 密评完整评分、GDPR/PCI-DSS 标准包、跨项目控制项复用 | C1-5 |
| C2-3 证据法律增强 | RFC 3161 时间戳/签名、第三方取证接口、证据包校验工具 | C1-6 |
| C2-4 成本与质量工程 | 分层模型路由、缓存、预算硬门、四域质量基准（召回/误报）回归 | C0、C1 |
| C2-5 多租户/SaaS 合规 | 数据隔离、审计、AGPL 源码提供流程、本地化模型部署方案 | 全部 |

**能力依赖关系（概要）**：C0-1/C0-2 是所有域的前置；C1-2 隔离 runner 是逆向与 PoC 验证的安全前置；C0-5 scope guard 是渗透对外使用（含内部合规演示）的前置；C1-6 证据完整性是合规报告可信的前置。

---

## 7. 待核实清单

1. **CodeScan-Artex 的许可与移植边界**：本地工作树已完整取得（HEAD `219775b`，实测差异见 02）；仍需确认其上游来源许可、与 ARTEX 的上游关系，以及哪些模块可按 AGPL 兼容方式移植。
2. **等保/密评判定细则**：控制项编号、权重、评分与结论口径需按所选标准版本与现行测评准则逐条落实，实现前需专业测评人员确认。
3. **上游 ARTEX 的商用许可**：作者使用限制声明与 AGPL 的叠加效力、可否取得商业授权——需法律意见与作者沟通。
4. **开源合规检查脚本许可**：拟接入的等保检查脚本来源与再分发条款待核实。
5. **镜像工具可用性**：semgrep/trivy/gitleaks/jadx/apktool/rizin/Ghidra/OpenSCAP 等需新增到镜像或 runner，其体积、架构支持（arm64）与离线安装包待核实。
6. **runner 隔离技术选型**：compose 旁车 vs nsjail/bwrap vs Firecracker，需结合部署环境（是否允许嵌套容器/特权）确认。
7. **上传/解压安全**：现有 `server/workspace.go` 无解压逻辑，zip/tar/固件解析需自实现并做 zip bomb/路径穿越防护。
8. **LLM 数据外发策略**：本地模型（vLLM 等）与 `llm_profiles.base_url` 的兼容性与性能待验证。
