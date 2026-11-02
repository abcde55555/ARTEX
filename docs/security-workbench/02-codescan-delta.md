# 02 · CodeScan-Artex（SCA Audit）二开差异分析

> **核查口径**（先说清结论的边界）
>
> - 本地仓库：`./ARTEX`（HEAD `a9d11cb`，2026-10-10）与 `./CodeScan-Artex`（HEAD `219775b`，2026-10-19）；只读分析，未修改任一仓库。
> - 差异用两棵工作树对比（排除 `.git`/`node_modules`/`web/out`），并用 `git diff --no-index --numstat --ignore-cr-at-eol` 量化；另把 ARTEX `v0.3.15`（`e6ec569`）导出到 `/tmp` 作为**语义基线**再比一次（原因见 1.3）。
> - 只做静态分析：**没有编译、没有跑测试、没有运行程序**（未安装依赖）。因此下文的「能构建」「行为正确」都是读代码得出的判断，标为推断。
> - 待核实项集中在文末，未把猜测写成结论。

---

## 1. 项目定位

### 1.1 是什么

**SCA Audit**：LLM 多 Agent 驱动的**静态代码审计平台**（Go 后端 + Next.js 前端 + PostgreSQL），定位由 README 首屏和「与扫描器的区别」表直接给出：产出不是规则命中列表，而是「接口清单 + 每条接口所需权限 + 带 `文件:行号` 的缺陷结论 + 依赖可达性判定」（`CodeScan-Artex/README.md:1-27`）。自我约束是「只做静态分析，不发请求、不执行命令、不做运行时验证」，并声称这条边界是**在工具装配层物理删除**而非提示词约束（`README.md:24-75`）。

它不是 ARTEX 的功能追加，而是**把 ARTEX 的渗透语义整体改写成审计语义**：资产从「域名/IP/服务/端点」变为「应用/接口（含权限）」，漏洞证据从「请求响应/命令输出」变为「源码位置 + 代码片段」，流量抓取、Bash、MCP、自定义工具全部从 agent 能力面移除。

### 1.2 仓库与提交

| 项 | 事实 | 证据 |
| --- | --- | --- |
| HEAD | `219775b`「平台权限收敛：管理员专属区守卫 + 发现/任务的部门可见性」，2026-10-19 | `git -C CodeScan-Artex log -1` |
| 提交数 | **8 个**，作者均为 `Ernket`；首个提交 `610e29d`「首个公开版本：SCA Audit 代码审计平台」（2026-10-14）是**整个代码库的单次压缩提交** | `git rev-list --count HEAD` |
| 历史形态 | `610e29d` → `827d10a`（复核 Agent 可修订复测报文 + mock 大改）→ 4 个 README/截图相关提交 → `962e13f` merge → `219775b`（权限收敛，24 文件 +944/−227） | `git log --graph --oneline` |
| 许可证 | AGPL-3.0（`LICENSE:1`；README `625-634` 复述并加用途限制） | `CodeScan-Artex/LICENSE:1` |
| 版本号 | 无 tag；二进制版本靠 `-ldflags -X main.version` 注入，默认 `dev`（`cmd/artex/main.go:25-27`） | — |
| 服务标识 | README/启动横幅/健康检查均已改为「SCA Audit」，但 **Go module 路径未改**，仍是 `github.com/Autumn-27/artex` | `go.mod:1`、`cmd/artex/main.go:39-42`、`server/server.go:995` |

### 1.3 与 ARTEX 的血缘：基线是 **v0.3.15**，不是当前 HEAD

两地仓库**没有共同提交历史**（CodeScan 的 8 个提交与 ARTEX 的提交历史无交集），所以「fork 自哪个版本」只能靠内容比对判定：

- `CodeScan-Artex/CHANGELOG.md` 与 ARTEX `e6ec569`（`docs(changelog): 定版 0.3.15`，2026-10-07）的 CHANGELOG **逐行一致**，CodeScan 仅新增了一段 `[Unreleased]`（`+39` 行）；与 ARTEX 当前 HEAD 的 CHANGELOG 则不同。
- `go.mod` 无 `replace github.com/Autumn-27/norma => ./norma`（ARTEX `a9d11cb` 加了，`ARTEX/go.mod:3-6`），`Dockerfile` 也还是「运行镜像 + 预编译二进制」的旧形态，而不是 ARTEX HEAD 的多阶段源码构建。
- 结论：**CodeScan-Artex ≈ ARTEX v0.3.15 全量源码 + 审计化改造**。ARTEX 在 v0.3.15 之后只有 8 个提交，全部是构建/文档修复（`69ad7b0` norma 回迁、`031dff5/82b6e76/c764649` Docker 多阶段构建、`4e5cd88/912e57d/5c0dbd4/a9d11cb` README 与镜像源），**没有改业务代码**。因此「以 v0.3.15 为基线量出的语义差异」≈「相对当前 ARTEX HEAD 的语义差异」，另外再加两项欠账：**norma 缺位**与**构建脚本落后**。

### 1.4 定位差异一览

| 维度 | ARTEX（渗透） | CodeScan-Artex（代码审计） |
| --- | --- | --- |
| 输入 | 目标任务/域名/资产范围（可选资产同步） | **上传 .zip 源码包**（`web/src/app/(main)/function/tasks/page.tsx:3486-3500`），admin 可后补 |
| agent 工具 | Bash/Kali、WebFetch/WebSearch、流量回放、MCP、自定义工具 | 仅 7 个本地文件工具 + 平台域工具（`agent/codeaudit_tools.go:55-65`） |
| 证据 | 请求/响应、命令输出（流量绑定） | 源码 `文件:行号` + 代码片段 |
| 产出 | 漏洞/PoC/资产图 | 接口清单 + 所需权限 + 缺陷 + 复核三态 + 推演复测报文 |
| 账号 | 单管理员密码（settings 表） | **users/departments 多账号**（管理员 + 部门 owner/visitor） |
| 默认端口 | 8787（`ARTEX/cmd/artex/main.go:55`） | **7878**（`CodeScan-Artex/cmd/artex/main.go:54`） |
| 流量录制代理 | 默认开启 `127.0.0.1:8788` | 代码默认关闭，但 **Dockerfile CMD 仍传 `-proxy :8788`**（`Dockerfile:45`，见 5.3） |

---

## 2. 精确差异

### 2.1 规模

| 口径 | 结果 |
| --- | --- |
| vs ARTEX HEAD `a9d11cb` | 205 行 `diff -rq` 输出；**151 个文件内容不同**、**62 个新增文件**、ARTEX 独有 `norma/`（210 文件）、`docs/`、`web/media/` |
| vs v0.3.15 基线 `e6ec569` | 149 文件不同、**62 文件新增**、2 处删除（`docs/`、`web/media/dashboard.png`）；文本类净变更 **+20,348 / −1,700 行**（Go **+14,061/−746**，TS/TSX **+5,562/−585**） |
| 仓库总量 | Go **106,597** 行（ARTEX 去掉 norma 为 93,282 → **+13,315 / +14.3%**）；web TS/TSX **61,065** 行（56,088 → **+4,977 / +8.9%**）；`page.tsx` 27 → **29** |
| 新增测试 | 新模块内约 **3,609 行**、约 52 个 `Test*` 函数（javadeps 20 个、finding_packet 12 个、task_deps 8 个等） |

> 噪声提示：`skills/` 下 4 个文件在 ARTEX HEAD 里是 CRLF、在 CodeScan 里是 LF，`diff` 会各报 +/−1100 行；`--ignore-cr-at-eol` 后归零。**这不是内容差异**，但反过来说明该仓库是从另一台机器整体上传的，行尾不规范。

### 2.2 树级增删

**新增目录/模块**

- `javadeps/`（12 文件、3,301 行）：本地 Java 依赖读取 + 按需反编译，下文 3.3 详述。
- 账号与权限：`db/users.go`(666)、`server/users_api.go`(365)、`server/authz.go`(310)、`server/policy.go`(119)。
- 代码来源与依赖：`server/task_upload.go`(445)、`server/task_deps.go`(1004)、`server/tool_deps.go`(437)、`db/task_deps.go`(223)、`db/task_uploads.go`(110)。
- 审计工具面：`agent/codeaudit_tools.go`(100)、`server/codeaudit_toolsurface.go`(60)。
- 复核/复测报文：`agent/finding_packet.go`(121)、`db/finding_packet.go`(818)、`server/finding_packet.go`(360)。
- 平台化修补：`evidence/syncdir_other.go` / `syncdir_windows.go`（Windows 上目录 fsync 兼容）、`server/auth_testhelper_test.go` 等测试脚手架。
- 前端：`system/users`、`system/departments` 两个新页面；`admin-only.tsx`、`auth-context.tsx`、`deps-tab.tsx`、`read-only.ts`、`permission-badge.tsx`、`finding-packet-*.tsx`、6 个 `layout.tsx`（给保留下来的旧页面加管理员门禁）。

**改动最大的既有文件**（按实质 diff 行数，`--ignore-cr-at-eol`）

| 文件 | +/− | 改动性质 |
| --- | --- | --- |
| `agent/promptcatalog.go` | 全文改写 | 5 个内置 agent 的提示词从「渗透」整体改写为「审计」 |
| `server/auth.go` | +340/−89 | 单管理员密码 → users 表 + JWT claims + 权限计算 |
| `db/schema.sql` | +144 | 新增 users/departments/user_departments/task_uploads/task_dependencies/finding_packets + 若干列 |
| `server/assembly.go` | +147 | 不再展开 MCP、不再 seed 流量工具、提示词一次性迁移 |
| `server/findings.go`/`db/findings.go` | 大改 | 发现按部门可见性过滤（`FindingScope`，`db/findings.go:168`） |
| `web/src/app/(main)/function/tasks/page.tsx` | +447/−55 | 新建任务必须先上传源码 zip（前端 `3341-3500`，后端 `server/server.go:1582` 强制） |
| `web/src/navigation/sidebar/sidebar-items.ts` | 重写 | 删掉流量/工具执行/MCP/工具/拦截/审批入口，加账号/部门，`adminOnly`/`requires` 过滤 |
| `cmd/artex/main.go` | +11/−10 | 端口 7878、`-proxy` 默认空、品牌与横幅 |

**删除/缺失**

- `norma/`（210 文件、37,226 行 Go）整个消失，`go.mod` 仍 `require github.com/Autumn-27/norma v0.4.3` 但**没有 replace**（`CodeScan-Artex/go.mod:6`；对比 `ARTEX/go.mod:3-6` 的 replace 注释「upstream 已闭源、v0.4.3 从 module proxy 恢复」）。
- `docs/`（ARTEX 仅一个 `漏洞流量证据.md`）与 `web/media/dashboard.png` 被删；但代码里仍大量引用 `docs/java-m2依赖读取与反编译方案.md`、`docs/漏洞复测报文.md` 等设计文档（`server/task_deps.go:26`、`agent/finding_packet.go:18-20`、`server/task_archive_package.go`），**这些文档不在公开仓库里**。

---

## 3. 代码审计能力端到端

### 3.1 一次审计的完整链路

1. **上传源码**：`POST /api/tasks/upload`（`server/server.go:697`）→ `server/task_upload.go:52`。限制：zip ≤ **1 GiB**、解压后 ≤ **4 GiB**、≤ **200,000** 文件（`task_upload.go:31-35`），解压做 zip-slip 防护（`safeZipPath`，`task_upload.go:358`）。上传只落盘到 `data/uploads` staging（`TaskUploadDTO` 不泄露服务器路径），刷新后可继续用（`GET /api/tasks/uploads`）。
2. **建任务**：`POST /api/tasks` → `server/server.go:1516-1660`。校验依次为：建任务权限（管理员或部门 owner）、部门归属（普通用户必须落在自己拥有的部门）、**非 admin 必须带 `upload_id`**（`server.go:1582`）、LLM 链、关联任务/企业、任务级拦截规则；创建成功后 `attachUploadToTask` 把 staging 目录搬进 `data/tasks/<id>/source` 并写 `tasks.source_path`（`server/task_upload.go:220`、`db/schema.sql:1504`）；搬运失败则删任务回滚。
3. **依赖入库（可选）**：任务详情「第三方依赖」页上传内含 jar 的 zip，或按坐标从本机 `.m2` 导入 → 只建「清单 + 类名索引」，**默认一个 jar 都不反编译**（`server/task_deps.go:26-34`、`211`）。
4. **开跑**：web 端默认「先创建、再手动开启」（`server/server.go:1655-1660`）；引擎仍是 ARTEX 的 planner/worker 事件闭环，只是语义变成审计。
5. **执行**：worker 工作目录为 `tasks/<taskID>/i<intentID>`，源码在 `../source`、依赖在 `../deps`（`agent/worker.go:231-240`；`README.md:140`）。提示词明确要求：先 LS/Glob 找入口 → Grep → Read，**不运行、不编译、不联网**（`agent/worker.go:231-236`）。
6. **写回**：资产（接口）→ `insert_assets`；结论 → `record_fact`；缺陷 → `report_finding`（证据必须是 `文件:行号 + 代码片段`，`agent/worker.go:243-245`）。
7. **复核/报告**：独立「漏洞复核」会话三态结论（见 3.4）；`reporter` agent 写 Markdown 报告；发现可导出 Markdown/CSV（`server/server.go:799,2248`；`report/findings.go`），**没有 SARIF**。

### 3.2 「无出网/无执行」是怎么实现的（三层，逐层可验证）

| 层 | 做法 | 证据 |
| --- | --- | --- |
| ① 构造 | 所有 agent 的通用工具集只从 7 个本地文件工具构造：`Read/Write/Edit/MultiEdit/LS/Glob/Grep` | `agent/codeaudit_tools.go:55-65`；`agent/worker.go:165-176`（`defaultToolsExcept` 改用它） |
| ② 装配 | `hostTools()` 只返回编排/复核/平台/依赖工具，不再返回流量工具、自定义工具；`server/assembly.go` **整段删除 MCP 连接逻辑**（保留变量结构） | `server/orchestration.go:43-52`；`server/assembly.go:68-76`；依赖工具按运行任务注入 `server/assembly.go:119` |
| ③ 兜底 | `AugmentTools` 最后按名字黑名单剔除（`bash/shell/webfetch/mcp__*/traffic_*/browser*/nmap*/…`），DB 工具表也绕不过 | `agent/assembly.go:63-66`；`agent/codeaudit_tools.go:39-52,69-100` |
| ④ DB 收敛 | 启动时幂等把 `tools` 表里系统级危险工具 `enabled=false, agents='[]'`，避免管理页显示「仍可用」 | `server/codeaudit_toolsurface.go:23-63` |

附加修补：拦截层新增**总开关**，因为「内置规则是对工具入参做子串/正则匹配」，Write 的文件内容里出现 `DROP TABLE` 之类的字样会被误拦（`intercept/intercept.go:142,303-330`、`guard/guard.go:87-95`、`server/intercept.go:381-409`）。这是审计场景下必须的适配。

> ⚠️ 边界上的两个真实缺口：**(a) 黑名单只能按名字匹配**，用户自建工具若取名 `run_scan` 不在表里——README 自己也承认单靠黑名单无法穷举（`agent/codeaudit_tools.go:56-58`；虽然装配层已不注入自定义工具，但这是「三重防线」里最脆弱的一环）；**(b) 「源码只读」只是提示词约束**——`Write/Edit/MultiEdit` 仍在工具集里（`agent/codeaudit_tools.go:58-60`），未发现文件系统级只读挂载或路径闸门，agent 技术上可以改被审计源码，从而污染「证据必须来自源码」这一前提。

### 3.3 本地 Java 依赖库 + 按需反编译（`javadeps/`）

这是本项目**原创度最高、也最可移植**的部分，核心是把成本拆成两段：

```
data/tasks/<id>/deps/
  INDEX.md                 # agent 先读的依赖清单
  manifest.json            # DB 行的机器可读镜像
  jars/...                 # 用户上传的原始 jar（字节不变）
  index/classes.txt        # 一行一个 FQCN，最便宜的 Grep 目标
  src/<group>/<artifact>/<ver>/  # 仅按需生成（源码包解压 or 反编译产物）
data/cache/deps/<sha256>/  # 全局内容寻址反编译缓存，跨任务复用
```

（布局定义：`javadeps/layout.go:5-80`）

- **Ingest（便宜）**：只读 zip 中央目录，产出坐标、类数、包数、是否 sources、是否 native-heavy、是否损坏；坐标可信度排序 `pom.properties > 仓库路径 > 文件名`（`javadeps/service.go:86-133`、`javadeps/scan.go:15-185`）。不启 JVM、不反编译。
- **Materialize（贵，按需，一次一个 jar）**：优先用真正的 `-sources.jar` 解压（秒级、字节等于原始源码）；否则调反编译器：**Vineflower（需 JRE 17+，优先）/ CFR（JRE 6+，兜底）**，失败自动换另一个并清理半成品（`javadeps/decompile.go:137-157,188-235`）。硬限制：单 jar 超时 **180s**、堆 **2g**（`decompile.go:33-36`）；解压上限 2 GiB / 200,000 文件（`materialize.go:33-36`）。
- **安全姿态**：反编译器只作为**子进程**运行，class 从不在进程内加载；工具发现顺序为「显式配置/环境变量 → `<data>/tools/` → PATH 里的 java」（`decompile.go:310-353`）；产物落在 data 根内；不联网。
- **本地 Maven 仓库（只读取件）**：配置 `java.local_repo` 后可按坐标硬链接进任务 `deps/jars`，**从不遍历、从不写入仓库**；坐标逐段校验并 `ensureInside` 防路径穿越（`javadeps/localrepo.go:30-36,118-210,211-260`；`javadeps/gav.go:73-126`）。未配置时明确报「未配置」，不静默失败。
- **pom 扫描**：勾选后按源码里的 `pom.xml` 补直接依赖，跳过 `test` scope 与 `type=pom`，版本含未解析 `${...}` 时报 `unresolved_version` 而**不猜**（`javadeps/pomxml.go:50-101`；`server/task_deps.go:903-957`）。
- **配置项**：`config.json` 的 `java.{java_path,vineflower_jar,cfr_jar,timeout_seconds,max_heap,local_repo}`，可被 `ARTEX_JAVA_*` 环境变量覆盖（`config/config.go:32-118`）。

### 3.4 产出模型（审计语义）

- **接口 = `assets.type='endpoint'` + `permission`**：六档「公开/注册用户/登录用户/管理员权限/内部接口/其他」（`db/schema.sql:89-114`，`(url,method)` 唯一；`app` 类型表示应用）。
- **缺陷 = `findings`**：`evidence` 文本要求「源码位置 + 代码片段」，`severity`/`status` 白名单校验；发现列表/导出按部门可见性过滤（`db/findings.go:168-200`；状态联动见 `db/finding_retests.go:263-280` 的 `SetFindingStatusTx`）。
- **漏洞复核（三态）**：从漏洞详情发起独立会话，`record_finding_retest_result(verdict, summary, evidence)`，`confirmed/refuted/inconclusive`；会话成功结束且 confirmed → 漏洞状态「已确认」，refuted → 「误报」，inconclusive 不动（提示词 `agent/retester.go:1-40`；工具 `server/finding_retests.go:186-210`；状态回写 `db/finding_retests.go:245-282`）。
- **推演复测报文（非证据）**：`report_finding` 可选 `retest_packets`，按**接口**一份（`asset_id + raw_request` + `grounded_refs` + `assumptions`），单独落 `finding_packets` 表，不进证据版本、不进导出、不参与漏洞成立判定；服务端按第一个空行切头/正文校验方法/路径/Host、凭证必须占位，**校验不过也保存**并标 `valid=false`（`agent/finding_packet.go:17-56`；`db/schema.sql:1190-1235`；`server/finding_packet.go:20-33`）。复核 agent 可用 `save_finding_packet` 修订报文（`server/finding_retests.go:156,315-400`）。
- **报告/导出**：Markdown 汇总、单条、CSV；任务冷归档把 `deps/src`（反编译产物）**整目录排除**——理由是体积、可重建、合规（`server/task_archive_package.go:108-130`）。

### 3.5 UI 变化

- 导航：移除「流量 / 工具执行 / MCP / 工具 / 拦截规则 / 资产拦截 / 审批记录」；新增「账号管理 / 部门管理」；「资产」改名「测试资产」；`adminOnly`、`requires` 两个可见性字段（`web/src/navigation/sidebar/sidebar-items.ts:44-46,63-100`）。
- 新增任务抽屉改为「先传 zip，再填目标」，非管理员不收 zip 不让提交（`web/src/app/(main)/function/tasks/page.tsx:3341-3372,3402-3404`）；目标框预置「先梳理接口 → 复核权限 → 全量审计」模板（`web/src/lib/task-goal-default.ts`）。
- 任务详情新增「依赖」Tab（上传/重扫/按坐标从 .m2 导入/单个反编译/清空，`web/src/app/(main)/function/tasks/detail/_tabs/deps-tab.tsx:23-40`）、「只读」文案助手（`read-only.ts`）、接口权限徽标（`permission-badge.tsx`）、原始数据包/推演报文卡片（`finding-packet-composer.tsx`、`finding-raw-packet-panel.tsx`）。
- 保留但不再有导航入口的旧页面（流量、工具执行、LLM 录制、工作空间、资产同步、MCP）统一用 `AdminOnly` 包一层（`web/src/app/(main)/_components/admin-only.tsx:19-40` + 各 `layout.tsx`），因为静态导出下 middleware 不生效，客户端门禁是唯一可见性控制。

---

## 4. 多账号管理（新增能力）

### 4.1 数据模型

`db/schema.sql:1461-1528`：

- `users(id, username, display_name, password_hash, is_admin, enabled, last_login_at)`，用户名 `lower(username)` 唯一，bcrypt 哈希。
- `departments(id, name, nkey UNIQUE, description)`。
- `user_departments(user_id, department_id, role)`，`role ∈ {owner, visitor}`：**owner = 可对话/建任务/暂停删除任务；visitor = 只读**（注释与 `db/users.go:14-27` 一致）。
- `tasks` 增列 `department_id`、`created_by`、`source_path`；`tasks` 索引 `(department_id, created_at DESC)`。
- 旧库迁移：`settings` 里的单管理员密码**一次性**迁移成 `admin` 账号，密码不变（`server/auth.go:523-548`、`db/users.go:165-189`）；首次初始化 `POST /api/auth/init` 只在 `users` 为空时创建第一个管理员（`server/auth.go:350-408`）；忘记密码用 `reset-password.sh` 直接改库（库内 pgcrypto 生成 bcrypt，脚本不再假设 `settings` 单密码，`reset-password.sh:1-30,144-165`）。

### 4.2 鉴权链

`requireAuth`（JWT，7 天，`server/auth.go:179`）→ `guardTaskAccess`（任务级：任务路径正则匹配，写方法 + 部门访客 → 403；内存里查不到任务时对非管理员默认 404，防止绕过，`server/authz.go:38-95`）→ `guardAdminAreas`（平台级：`adminOnlyAreas` 前缀清单 = 系统配置/更新/日志/LLM/Agent/Skill/工具/MCP/通知/拦截/工作空间/资产同步/流量/`/api/active`；`adminOnlyWrites` = 分类/模板/资产/企业的写操作，`server/policy.go:29-104`）。

权限计算：管理员全开；普通用户 `can_create_task = can_chat = 是否至少是一个部门的 owner`，`IsVisitorOnly` 标记纯访客（`server/auth.go:238-263`）；任务可见性 = 自己创建的 OR 本部门的（同部门 owner 才能写，`server/auth.go:288-324`）；发现（漏洞）用同一套部门模型收口，`selected` 精确导出也带可见性条件，防止猜 id 导出他部门漏洞（`server/authz.go:139-300`、`db/findings.go:168-200,536-560`）。

> 注：最后一笔提交 `219775b` 就是在补这一层——`server/policy.go` 的注释明确写「本项目的部门权限模型原本只覆盖任务这一条线……系统区在 API 层只过了 requireAuth，任何登录账号都能读写」，这次用「路径前缀 → 仅管理员」一次性收口。

### 4.3 UI

`auth-context.tsx` 把「当前账号 + permissions」提升为全局（首挂载拉 `/api/auth/me`，401 统一清 token 跳登录）；登录/初始化页成功后 `adopt()` 立即拿到权限；侧边栏按 `adminOnly`/`requires` 过滤；详情页按 `task.can_write` 禁用控件并给统一只读文案（`web/src/lib/auth-context.tsx:9-80`、`read-only.ts:1-7`、`sidebar-items.ts`）。后端仍独立强制校验（前端只是不摆出无权入口）。

### 4.4 隔离边界（做了什么 / 没做什么）

| 已隔离 | 未隔离 |
| --- | --- |
| 任务（部门 + 创建者）、发现/漏洞列表与导出、对话（访客不能发起）、系统区配置（仅管理员） | **资产图全局共享**（跨任务/跨部门同一份真值库，`db/schema.sql:40-114`），普通账号能看到本部门任务范围之外的资产吗？——按代码只有 `/api/assets` 的写操作限管理员，读仍共享（`server/policy.go:72-84`），需结合产品意图确认 |
| 任务源码目录按任务隔离（`data/tasks/<id>/source`） | 当前模型里没有租户/多组织概念；部门是唯一隔离维度；LLM 配置、技能、通知全为全局管理员配置 |
| 复核会话绑定漏洞 | 任务归档/证据目录是否含其他部门数据？— `task_archive_package` 按任务导出，未见跨部门泄漏；**待核实**（未运行） |

---

## 5. 质量与风险

### 5.1 分叉成本与可合并性（最大风险）

- **无共同历史**：8 个提交、首个提交是全库压缩上传 → 无法 `git merge`/`rebase`，也无法用 `git log -S` 追溯上游改动；两个仓库只能靠**文件级 diff 人工搬运**。
- **提示词是复制粘贴式改写**：5 个内置 agent（`goals/planner/worker/mainagent/auto`）+ `reporter` + `retester` 的正文全部重写（`agent/promptcatalog.go`、`agent/retester.go`），上游提示词的任何改进都无法自动吸收，反之亦然。这是双倍维护成本的主要来源。
- **同 module 路径**：`github.com/Autumn-27/artex` 未改 → 好处是**文件可以直接拷回 ARTEX 编译**，坏处是若两个仓库同时安装/引用会冲突，且品牌与代码身份不一致（module、镜像名 `autumn27/artex`、README 里 `git clone https://github.com/Autumn-27/ARTEX.git` 的安装指引仍是上游仓库，`README.md:334-352`）。
- **保留但不用的子系统**（死代码/误导面）：流量录制 MITM（`server/manager.go:228,384,444,739-772`；`traffic/` 包未删）、自定义工具、MCP 页面与 API（`server/server.go:909-913`，仅管理员可访问但 agent 永远拿不到工具）、渗透技能（`skills/{api-recon,playwright-cli,scopesentry}` 内容与 ARTEX 完全一致，仍在发布包里，但对应 Bash/curl 能力已被移除）；`server/finding_workflow.go:97-105` 里还存在「把 `get_finding_traffic` 重新绑给 worker」的旧迁移，之后靠 `retireCodeAuditDangerousTools` 再收回——顺序敏感、易回归。
- **CHANGELOG 缺口**：新增的多账号/部门权限在 `CHANGELOG.md` 的 `[Unreleased]` 里**完全没有记录**（只记了漏洞展示/依赖/证据存储等），与提交 `219775b` 的重要性不匹配，说明发布说明不可靠。

### 5.2 构建与可复现风险

| 风险 | 证据 | 影响 |
| --- | --- | --- |
| **norma 缺位** | `go.mod:6` require v0.4.3；无 `replace => ./norma`；`go.sum` 有 hash；ARTEX 的注释说明 upstream 已闭源、靠 module proxy 恢复 | 构建必须能访问 Go module proxy 且 proxy 仍留存该版本；离线/内网构建会失败。ARTEX HEAD 正是为解决这个问题把源码 vendor 进仓库 |
| **Dockerfile 落后** | `Dockerfile:1-45` 仍是「运行镜像 + 预编译 `dist/<arch>/artex`」，不是 ARTEX HEAD 的多阶段源码构建；`CMD` 仍带 `-proxy :8788`（第 45 行） | `docker compose build` 不能从纯源码构建（需先手工交叉编译）；镜像默认**开启 MITM 录制代理**，与 README「不做任何对外请求」的表述矛盾（代码默认已关，但镜像入口把它打开了） |
| 端口变化 | 7878（`cmd/artex/main.go:54`、`docker-compose.yml` 映射改为 `7878:7878`） | 从 ARTEX 升级/混部时所有反代、文档、监控端口要改 |
| 未验证构建 | 本次未执行 `go build`/`go test` | 是否可编译、测试是否全绿：**待核实**（未安装依赖、未运行） |

### 5.3 安全与合规

- **AGPL-3.0 义务**：`LICENSE` 保留 AGPL 原文，README 也声明 AGPL（`README.md:625-634`）。但 README **没有任何上游署名/来源说明**（「参考」一节只列了 `oritera/Cairn`，`README.md:617-621`），module 路径与大量文件头注释仍是 `github.com/Autumn-27/artex` / ARTEX 品牌残留（`web/src/lib/api.ts`、`web/src/proxy.ts` 等）。作为衍生作品对外分发/提供网络服务时，**必须保留版权与来源声明并同样以 AGPL 开源**；当前形态存在合规瑕疵（不是「不能做」，而是「说明义务没履行」）。
- **「只读源码」未被技术上强制**（见 3.2 缺口 b）：审计结论的可信度依赖源码未被修改，但 agent 有 Write/Edit。
- **出口面残留**：MITM 流量代理代码仍在二进制内并可被 `-proxy` 打开；MCP/自定义工具的执行器代码也在（仅入口被收口）。声明「物理删除」与实际「不注入 + 黑名单」之间仍有差距，安全审计时应按「能力仍在、默认关闭」评估，而不是按「代码已删除」。
- **反编译第三方代码的合规**：`task_archive_package.go:99-110` 已意识到「把反编译产物打进导出包有合规风险」并在归档中排除，但任务目录里仍是明文源码——对闭源商业 jar 的反编译使用需用户自行负责（README 免责条款有覆盖）。

### 5.4 维护状态

- HEAD `219775b`（2026-10-19）；GitHub 上仓库 last push 同为 **2026-10-19T15:11:23Z**，此后无提交（2026-11-02 查询）。
- 单人维护（8 个提交均同一作者），无 tag、无 release、无 PR/issue 历史（GitHub API：`tags=0`、`releases=0`、`pulls(all)=0`、`open_issues=0`、`stars=0`、`forks=0`），`.github/workflows/release.yml` 与 v0.3.15 一致。
- 代码质量本身**不低**：注释密度高、关键决策有「为什么」、新模块配套测试约 3.6k 行、对成本/安全边界/迁移有显式设计说明（如 `javadeps/service.go:12-30`、`server/task_deps.go:26-34`），明显是熟悉 ARTEX 的人做的深度改造，而不是脚本级 fork。

---

## 6. 合并建议（折进统一工作台）

### 6.1 总体策略

**以 ARTEX 当前 HEAD 为主干**（它才有 norma 源码、可复现构建、后续维护），把 CodeScan-Artex 当作「**一组可移植的能力补丁**」按文件搬运；**不要反向以 CodeScan 为主干，也不要尝试 merge 两个仓库的分支**（无共同祖先，merge 只会产生海量冲突）。同时**不要整包照搬审计化改写**——它把「渗透语义」硬改成了「审计语义」，直接合并会把渗透能力（Bash/流量/MCP）删掉，与「四域统一工作台」目标冲突。

推荐的抽象：引入**任务类型/能力画像（capability profile）**，`pentest` 与 `codeaudit` 各自决定「工具集 + 提示词包 + 输入类型 + 证据类型」，把 CodeScan 的改动从「全局删除」降级为「按任务类型装配」。

### 6.2 可移植清单

| 优先级 | 移植内容 | 理由 / 注意 |
| --- | --- | --- |
| **P0 直接搬** | `db/users.go`、`server/users_api.go`、`server/auth.go` 的多账号改造、`server/authz.go`、`server/policy.go`、schema 中 users/departments/user_departments + tasks 两列、`web` 的 users/departments 页 + `auth-context.tsx` + `admin-only.tsx` + nav 过滤 + `read-only.ts` + `reset-password.sh` | 与业务域无关的通用平台能力；ARTEX 现在只有单管理员，这是工作台化的刚需。移植时把「管理员专属区」清单改成配置化，避免上游新增路由时漏守卫 |
| **P0 直接搬** | `javadeps/` 整个包（12 文件，无业务依赖）+ `config` 的 `JavaDecompiler` 段 | 纯工具库，只依赖 data 目录布局与 `config`；是「逆向分析/代码审计」两域共用的基础能力 |
| **P1 改造后搬** | 源码上传链路：`server/task_upload.go`、`db/task_uploads.go`、`tasks.source_path`、`createTask` 的 upload 绑定 | ARTEX 的任务输入是「目标」，需要扩展为「输入类型：目标 / 源码包 / 二进制」，`attachUploadToTask` 可原样复用 |
| **P1 改造后搬** | 依赖模型与页面：`db/task_deps.go`、`server/task_deps.go`、`server/tool_deps.go`、`deps-tab.tsx` | 强依赖「任务有源码」这一前提；建议把 `list_dependencies`/`ensure_dependency_source` 做成**按任务类型启用**的 host tool，而不是全局只绑 planner+worker |
| **P1 改造后搬** | 审计工具面策略：`agent/codeaudit_tools.go` 的 7 工具集与黑名单 → **改为按画像的能力白名单**（而不是全局黑名单） | 建议方向反过来：默认拒绝，profile 显式声明 `file.read/file.write/net.exec/…`；把黑名单保留为兜底。同时**真正落地「源码只读」**：把审计任务的 `source/` 挂成只读（目录权限或 Write 工具路径闸门），否则证据可信度站不住 |
| **P2 语义合并** | 复核/复测报文：`agent/finding_packet.go`、`db/finding_packet.go`、`server/finding_packet.go`、`report_finding.retest_packets`、`save_finding_packet`、复核三态 | ARTEX 已有「复测（reproduced/fixed）」与流量证据；建议统一成「证据链 = 运行时证据(流量) | 源码证据 | 推演报文(非证据)」三档，`finding_packets` 作为**无流量时的降级展示**，而不是替换流量。三态 verdict 与 ARTEX 的 reproduced/fixed 需做映射迁移 |
| **P2 语义合并** | 审计提示词包（`agent/promptcatalog.go` 的审计正文、`agent/retester.go`、`agent/worker.go`/`planner.go` 的审计段落） | 不要覆盖全局提示词；抽成 `prompt pack`（按任务类型选择），保留 ARTEX 渗透版；注意 CodeScan 用 `setting prompts.codeaudit_v1` 做过一次性强覆盖（`server/assembly.go:159-190`）——合并时**绝不能**照抄这段迁移，否则会把已有用户的渗透提示词全部覆盖 |
| **P2 可选** | 接口权限模型（`assets.permission` + 六档取值 + `permission-badge`）、发现页「所需权限」列 | 是审计域的核心产出，但也可被「合规检测」复用（权限矩阵即合规证据），建议放进统一资产模型 |
| **不要带** | 品牌改写（module 路径/横幅/端口 7878/「SCA Audit」服务名）、`Dockerfile` 旧构建方式、`-proxy :8788` 的 Docker CMD、对 docs 的悬空引用、CHANGELOG 的 `[Unreleased]` 整段、删掉 `web/media`、删掉流量/MCP/自定义工具的做法 | 这些是「把 ARTEX 改造成单一审计平台」的产物，与「四域统一工作台」目标相反；端口与品牌变更会造成部署回归 |

### 6.3 必要重构（搬运时必须做的，否则会欠技术债）

1. **任务类型/能力画像**：`tasks` 增 `kind`（pentest/codeaudit/reverse/compliance）+ 画像配置；`agent.ToolAugment` 按 kind 组装工具集；`server/policy.go` 的路径清单按 kind 与角色求交。
2. **提示词按域分包**：`agent_prompts` 增加 `domain` 维度，默认按任务 kind 选包；禁止「启动时覆盖所有内置 agent 提示词」的迁移模式（CodeScan 的 `prompts.codeaudit_v1` 是反例）。
3. **输入/证据抽象**：`task_inputs(kind, source_zip|target|binary)`、`evidence_refs(kind=source|traffic|packet)`；把 `findings.evidence` 从自由文本升级为结构化引用（文件:行号 / 流量 binding / 报文 id），导出与复核都基于结构化引用。
4. **依赖能力服务化**：`javadeps.Service` 已是无状态服务，接到 server 时只暴露「Ingest / Materialize / ImportFromLocalRepo」三个动作 + 两个 agent 工具，保持「一次一个 jar、上限、缓存」的成本约束。
5. **前端页面合并**：保留 ARTEX 的流量/MCP 页面给渗透域，新增的 users/departments/deps 页面按画像显示；`AdminOnly` 与 `requires` 机制可以整体保留（但后端守卫是唯一真源这一原则要在文档里写死）。

### 6.4 落地顺序与验收

1. **第 1 步**：users/departments/authz/policy + schema 迁移 + 前端账号页 → 验收：老库升级后原单管理员密码可登录、普通账号看不到其他部门任务与漏洞、非管理员打系统 API 全 403。
2. **第 2 步**：javadeps + 依赖 Tab + 依赖工具（按 kind 启用）→ 验收：上传含 jar 的 zip 后 `deps/INDEX.md`、`classes.txt` 生成；未反编译；单个反编译命中缓存；无 java 环境下优雅降级不报任务失败。
3. **第 3 步**：codeaudit 任务类型 + 源码上传 + 审计提示词包 + 7 工具白名单 + 源码只读强制 → 验收：审计任务里 agent 工具列表不含 Bash/MCP/traffic；`source/` 不可写；证据必须是 `文件:行号`。
4. **第 4 步**：复核三态 + 报文 + 报告/导出 → 验收：confirmed/refuted 联动状态且保留原证据；报文校验失败仍保存并标注；导出不含报文。
5. **并行**：把 ARTEX 的 Dockerfile/norma vendoring 保留为主干版本，**不要**把 CodeScan 的旧构建脚本带进来。

---

## 附：未核实 / 需下一步确认

1. **GitHub 元数据**：已用 API 核实（2026-11-02）：last push `2026-10-19T15:11:23Z`，tags/releases/PR 均为 0，stars/forks 为 0；release 二进制不存在。
2. **构建与测试**：`go build ./...`、`go test ./...` 是否通过（缺 norma 源码、未装依赖）——未运行。
3. **运行时行为**：审计任务的真实产出质量（接口权限判定准确率、误报率、单任务 token/时长成本）、反编译实际耗时与失败率——只有 README 与注释中的经验值（如 1.7 MB jar 约 40–60s），未实测。
4. **资产读权限**：`/api/assets` 读对普通用户共享是否会把其他部门的资产暴露到「测试资产」页——需跑接口确认（代码上看只有写受限）。
5. **归档/导出边界**：任务冷归档是否会把其他部门数据带出、证据目录是否按任务隔离——需运行验证。
6. **「只读源码」**：是否存在未读到的文件系统层约束（如容器只读挂载/工作目录权限）——代码层未发现，部署层待核实。
