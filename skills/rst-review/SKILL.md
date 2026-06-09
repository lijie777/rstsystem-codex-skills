---
name: rst-review
description: >
  骨科机器人项目改动聚合审查调度器：自动判断本次代码改动涉及
  哪些领域(渲染/Qt/并发/几何变换/手术安全/数据库/业务日志/跨平台)，调用对应的专项审查 skill，
  在运行时支持子代理时并行审查，把多边发现合并、去重、分级成一份报告，再逐条交用户
  确认、由当前会话实施修复。一条命令覆盖一个改动涉及的全部专项视角，省去手动逐个调用。
  本 skill 为工具中立设计，Claude Code 与 Codex（及其它兼容 Agent Skills 的工具）均可直接使用。
  Use this when the user runs /rst-review (or $rst-review), or asks to 全审 / 聚合审 /
  综合审 / 按相关专项审一遍 / 把相关的 review skill 都跑一下 / comprehensive review of my
  changes, i.e. wants all relevant orthopedic robotics domain-specific review skills applied to the
  current diff (or a whole module) at once, followed by user-confirmed fixes; optionally
  exports the report to a docs/ folder on request.
  触发词：rst-review、聚合审、综合审、全审、全部专项、相关专项都审、一键审查、
  把 review 都跑一遍、综合复核、改动涉及多个领域帮我审、导出报告、导出审查报告、存档报告、--export。
---

# 骨科机器人项目聚合审查调度器 (rst-review)

一个改动常常**同时**踩到多个领域（改个"植入物位姿调整后的 3D 刷新"就同时涉及渲染、Qt 信号槽、几何变换三块）。本 skill 自动识别本次改动涉及哪些领域，把对应的专项审查跑一遍（运行时支持子代理时**并行**，否则主会话**顺序**），再汇总成一份分级报告——你不用记 skill 名、不用逐个手动调用。

> 工具中立：本 skill 只用 Agent Skills 核心格式（name + description + markdown），Claude Code、Codex 等兼容工具直接放入各自的 skills 目录即可用；运行时相关步骤（子代理、谁来修复）在下文写成**自适应**。审查阶段只读，不改文件；修复必须等用户逐条确认。审查报告用简体中文。

## 使用边界

显式触发本 skill 的信号：`rst-review` / `$rst-review` / `/rst-review`、聚合审、综合审、全审、一键审查、把相关 review skill 都跑一下、按骨科机器人项目相关专项审一遍、comprehensive review of my changes。

被显式触发时按本文流程走，不要再叠加其它通用 review 流程。**不要在普通编码任务中自动触发**。用户触发本 skill 本身即表示允许使用子代理做只读审查。

## Step 1 — 确定审查目标与模式

支持两种模式，先判断用户要哪种（不清楚就问一句"审这次改动，还是审整个模块的现有代码？"）。如果请求里同时出现路径和"未提交/改动/diff/提交/分支差异"等词，优先按**路径过滤 diff**处理；只有用户说"整个模块/全量/现有代码/审这个目录本身"时，才进入全量模块审查。

### 模式 A — 改动审查（diff 模式，默认）
审"变化了什么"。按优先级：
1. 用户给了 **git ref / 区间**（如某 commit、`A..B`、`main...HEAD`）→ 审对应 diff：
   - 单个提交：`git show <commit>` 或 `git diff <commit>^..<commit>`
   - 区间/分支差异：`git diff <A>..<B>` / `<A>...<B>` / `<commit>..HEAD`
2. 用户给了 **路径过滤条件**（如"审查 `src/Modules/SurgicalNavigation` 的未提交改动"）→ 在对应 diff 后追加 `-- <path>`：
   - 未提交模块改动：`git diff HEAD -- <path>`
   - 某提交内某模块改动：`git diff <commit>^..<commit> -- <path>`
   - 某区间内某模块改动：`git diff <A>..<B> -- <path>` / `git diff <A>...<B> -- <path>`
3. 无参数 → **未提交改动**：`git diff HEAD`（工作区 + 暂存区 vs HEAD）。
4. 工作区干净 → 回退审**当前分支相对 main 的全部改动**：`git diff main...HEAD`。

以上 git 命令均在**项目仓库根目录**执行（不写死绝对路径，便于他人/其它机器复用）；先用 `git diff --stat` 拿改动文件清单，再做 Step 2 领域路由。

### 模式 B — 全量模块审查（whole-module 模式）
审"一个已存在模块/目录/文件的**现有全部代码**"，与是否改动无关。
- **触发**：用户给的是文件/目录**路径**，或说"审整个 X 模块 / 这个插件 / 这些文件"。
- **取范围**：列出目标下的源码（`*.cpp/*.h/*.cxx/*.ui` 等；排除 `build/`、`ThirdParty/`）。
- **控制成本**：整模块可能很大。按 Step 2 路由命中的专项，让每个子代理**只读与其领域相关的文件**全文审查；文件过多时提示用户范围偏大，建议聚焦关键子目录/核心文件，或分批审。

无论 A/B，都**不要主动构建/测试**（项目 House rule），验证靠源码阅读与 `git diff`。

## Step 2 — 领域路由表（命中即激活对应专项；一个文件可命中多个）

对每个目标文件，按"路径 + 内容关键词"判断激活哪些专项。命中多个就都激活。

| 专项 skill | 路径信号 | 内容关键词信号 |
|---|---|---|
| `render-perf-diagnose` | 影像/分割/渲染/预览模块：`*Image*`、`*Render*`、`*Renderer*`、`*Viewer*`、`*View*`、`*ViewPanel*`、`*VTK*`、`*ITK*`、`*MITK*`、`*Segment*`、`*Seg*`、`*Volume*`、`*Mesh*`、影像/分割/渲染/预览/体绘制 | `vtk`、`itk`、`mitk`、`RenderingManager`、`vtkMapper`、`volume`、`reslice`、`RequestUpdate`、`level window`、`marching`、`actor`、`renderer`、渲染/刷新/掉帧 |
| `qt-review` | 任意 `.cpp/.h/.ui` 含 Qt | `connect(`、`SIGNAL(`/`SLOT(`、`moveToThread`、`QThread`、`connectNotify`/`emitNotify`、`deleteLater`、`QTimer`、`blockSignals`、`QSortFilterProxyModel`、`setModel`、`QDialog::exec`、`QByteArray`/`readyRead`(拆包) |
| `cpp-concurrency-review` | 设备/控制板/通信/线程模块：`*Thread*`、`*Worker*`、`*Serial*`、`*Tcp*`、`*Socket*`、`*Board*`、`*Bd*`、`*Ctrl*`、`*Control*`、`*Controller*`、`*IO*`、`*Device*`、`*Comm*`、`*OTS*`、`*Tracker*`、控制板/控制器/通信/采集/心跳 | `std::thread`、`std::mutex`、`std::atomic`、`QMutex`、`QWaitCondition`、`QSemaphore`、`detach`、`run()` 循环、`volatile`、心跳/worker |
| `geometry-transform-review` | 导航/配准/工具注册/几何算法模块：`*Geometry*`、`*Transform*`、`*Matrix*`、`*Nav*`、`*Navi*`、`*Navigat*`、`*Navigation*`、`*SurgicalNav*`、`*SurgNav*`、`*Reg*`、`*Regis*`、`*Registration*`、`*Register*`、`*ToolReg*`、`*Calib*`、`*Calibration*`、导航/配准/注册/标定/工具 | `Eigen`、`Matrix4`、`Matrix3`、`Quat`、`Quaternion`、`pose`、`Transform`、`inverse()`、`transpose()`、`normalize`、`acos`、`registration`/配准、`Direction`/`spacing`/`origin`、`LPS`/`RAS`、`float[16]`、`vtkMatrix` |
| `surgical-safety-review` | 导航/规划/机械臂/控制板/阶段门禁模块：`*Nav*`、`*Navi*`、`*Navigation*`、`*Plan*`、`*Planning*`、`*Preop*`、`*PreOp*`、`*Config*`、`*Robot*`、`*Robotic*`、`*RobotArm*`、`*Arm*`、`*Manipulator*`、`*Board*`、`*Bd*`、`*Ctrl*`、`*Control*`、`*Controller*`、`*Safety*`、`*Gate*`、`*Stage*`、导航/规划/术前/机械臂/机器人/控制板/门禁 | `OperationState`、阶段切换/门禁、`canEnter`、`m_isMissing`、`heartbeat`/心跳、`timeout`、`ArmMove`/机械臂、使能/enable、`KMessageBox`(确认)、植入物/案例状态、无有效植入物、未选节段、`catch`(吞错)、默认标志位 |
| `database-integrity-review` | `*DataBase*`、`*Database*`、`*DB*`、`*SQLite*`、`*Sql*`、`*Login*`、`*Auth*`、`*User*`、`*Case*`、`*Impl` | `QSqlQuery`、`QSqlDatabase`、`exec(`、`prepare(`、`bindValue`、`QString("...SELECT/INSERT`、`transaction`/`commit`/`rollback`、`lastError`、`loginPass`/password、`hash`、`MAX(` |
| `logging-review` | 日志封装、关键业务流程、设备通信、状态机、病例/规划/导航保存相关改动：`*Log*`、`*Logger*`、`KALog*`、`*Case*`、`*Plan*`、`*Navigat*`、`*Navigation*`、`*Robot*`、`*Arm*`、`*Board*`、`*DB*`、日志/审计/追溯 | `LOG_`、`DB_LOG_`、`LOG_*_CL`、`qDebug`、`qWarning`、`qCritical`、`std::cout`、`std::cerr`、`spdlog`、`SUFFIX`、`KALog`、日志级别、可追溯、审计、保存成功/失败、加载失败、状态切换、心跳、通信回调、错误码、PHI |
| `cross-platform-guard`（默认关） | 仅当出现下列信号才激活 | `#ifdef _WIN32`/`__linux__`、`std::filesystem`、字节序/`<<8`/`htons`、`wchar_t`/`TCHAR`、`dllexport`/visibility、手写协议拆包的字节移位、`fromLocal8Bit`/编码转换、`CMakeLists` 跨平台 |

> 注：`cross-platform-guard` 默认不激活，仅在出现字节序/编码/`#ifdef`/协议拆包/CMake 等真实跨平台信号时才纳入。
> 注：`logging-review` 不是无条件激活；只要 diff 涉及 `LOG_` / `DB_LOG_` / `qDebug` / `std::cout` / `std::cerr` 等日志信号就必须激活。若没有日志改动，但触及设备通信、状态机、病例/规划/导航保存、数据库访问或安全门禁，可激活它检查是否缺少关键日志；若完全无日志风险，应在未激活原因里明确说明。
> 若改动只是注释/文案/格式，告知用户"无需专项审查"并停止。

专项 skill 名 ↔ 中文领域名（填子代理 prompt 用）：`render-perf-diagnose`=渲染性能、`qt-review`=Qt、`cpp-concurrency-review`=并发、`geometry-transform-review`=几何变换、`surgical-safety-review`=手术失效安全、`database-integrity-review`=数据库/数据完整性、`logging-review`=业务日志、`cross-platform-guard`=跨平台。

## Step 3 — 并行审查（每个激活专项一个只读子代理，自适应运行时）

为 Step 2 激活的**每个**专项派一个**只读**子代理，互不依赖、尽量并行。按当前运行时选择子代理机制：

- **Claude Code**：用 Agent 工具，`subagent_type: general-purpose`（只读审查）；模型沿用运行时默认，若设了 `CLAUDE_CODE_SUBAGENT_MODEL` 则用之，**无需**显式传 `model`。同一条消息里一次性发出多个，使其并行。
- **Codex**：用 `multi_agent_v1.spawn_agent`，`agent_type: "explorer"`，`fork_context: false`（只传必要范围/diff 摘要/专项 skill 名，减少上下文污染）；**不设** `model`/`reasoning_effort`/`service_tier`（除非用户指定）。
- **找不到子代理工具时**：先用运行时的工具搜索（如 Codex 的 `tool_search`）查 multi-agent/subagent 工具；仍不可用则**降级为主会话顺序**逐个套各专项 checklist。

通用规则：子代理只读、不改文件、不提交、不运行构建/测试。若激活专项少于 2 个或改动很小，可主会话直接套对应 checklist，避免过度调度；但用户明确要求子代理时必须用子代理。

子代理 prompt 模板（`<领域名>`/`<skill名>` 按 Step 2 末尾对照表替换；`<skill名>` 即 `qt-review` 这类目录名）：

```text
你是骨科机器人项目的 <领域名> 只读代码审查员。

请读取并严格遵守这个专项 skill 作为审查标准：<skill名>
- 若运行时已自动加载该 skill，直接用；
- 否则按 skill 名在已安装位置查找 SKILL.md（不要写死某个用户目录）：
  `.claude/skills/<skill名>/SKILL.md`、`~/.claude/skills/<skill名>/SKILL.md`、
  `.agents/skills/<skill名>/SKILL.md`、`~/.codex/skills/<skill名>/SKILL.md`、
  `$CODEX_HOME/skills/<skill名>/SKILL.md`、`%USERPROFILE%\.codex\skills\<skill名>\SKILL.md`。
- 若无法定位该 skill 文件，明确报告"该专项无法完成"，不要凭空审。

审查范围：
<diff 模式：本次 git diff 摘要/命令；全量模式：该领域相关文件路径列表。
 必要时允许你用只读 git diff / 文本搜索 / 读取文件 查看上下文与被调用方。>

要求：
1. 只审与你专项相关的问题，严格按该 skill 的 checklist 和"本项目易错点速查表"逐条对照。
2. 只报可定位、可解释、可行动的真问题；不要泛泛而谈。
3. 每条发现必须含：文件:行或函数、问题、为什么危险(该领域机理)、修法方向、严重度(阻断/严重/建议)、置信度。
4. 只读审查：不修改文件、不运行构建或测试、不提交。
5. 用中文输出结构化发现清单；没有问题就明确说"未发现该专项问题"，并说明残余风险。
```

等待策略：先完成本地范围识别与非重叠检查，再等待子代理返回；可一次等待多个；不反复忙等；若超时，先汇总已返回结果并说明哪些专项未完成。涉及跨线程的发现，几何/Qt/并发子代理可能各自命中，留待 Step 4 去重。

## Step 4 — 汇总、去重、分级

收齐结果后：
- **去重**：同一文件/函数/根因被多个专项命中的合并为一条，标注"多领域命中"（这类往往更重要）。
- **分级**：🔴 阻断级、🟠 严重、🟡 建议（标准见下），按严重度排序，不要堆砌；🟡 建议类可折叠概述。
- **标注来源**：每条注明来自哪个专项 + 置信度。
- **不臆造**：对不确定结论明确标注"需运行验证 / 需查文档 / 需用户确认"，**不要把推断写成事实**。

分级参考：
- 🔴 阻断级：崩溃、跨线程 UI/对象直接访问、坐标打偏、SQL 注入、明文密码、带错误数据继续、设备失联不 fail-safe、危险物理动作无校验。
- 🟠 严重：偶发错误、状态残留、事务半写、缓存帧无新鲜度、性能路径明显重复刷新、连接/生命周期风险、漏注册元类型/信号静默失效。
- 🟡 建议：风格一致性、局部可维护性、日志级别、验证补强。

## Step 5 — 确认与修复

1. 把分级清单交用户**逐条确认**（确认/跳过/存疑）。无论是否启用导出，都必须先在**当前会话**完整展示检测出来的问题项（🔴/🟠/🟡 + 待确认），报告发出后**停止**，用户确认前**不要改代码**。
2. 用户确认后，由**当前运行时的会话/助手**实施修复（在 Claude Code 即 Claude 修、在 Codex 即 Codex 修），遵守仓库规则：
   - 修改范围只覆盖用户确认的问题。
   - 不主动运行 CMake configure/build/test。
   - 默认中文注释与中文日志；匹配现有结构；新文件按项目编码约定（如要求 UTF-8 BOM）。
   - 修复后用 `git diff`、`git diff --check`、文本搜索与源码检查做轻量验证。
3. 最终用中文总结：改了哪些文件、每处目的、行为影响、已做验证、未验证风险。

## 输出格式

```text
# 骨科机器人项目聚合审查报告

## 审查范围
- 范围（diff 区间 / 模块路径）：
- 改动/目标文件清单：
- 激活专项：
- 未激活专项及原因（含为何未激活 logging-review / cross-platform）：

## 🔴 阻断级
1. [文件:行][来源专项|置信度][多领域命中?]
   问题：
   机理：
   修法：

## 🟠 严重
...

## 🟡 建议（概述）
...

## 待确认
- 建议修复项清单（等用户逐条确认后进入修复）：
- 未完成/超时专项：
- 建议验证：
```

## 可选：导出报告到文件

默认**只在对话里呈现**报告，不写文件。**仅当用户明确要求导出时**（`/rst-review --export`、或请求里说"导出/存档报告"）才落盘：

- **当前会话输出强制保留**：启用导出时，仍必须在当前会话完整显示检测出来的问题项、分级、来源专项、修法和待确认清单。导出文件只是额外存档，**不能**用"已导出到某路径"替代审查报告正文。
- **目录**：当前仓库根下的 `docs/code-reviews/`（相对路径，随项目走、不写绝对路径；目录不存在则先创建）。
- **文件名**：`rst-review-<范围标识>-<YYYYMMDD>.md`
  - `<范围标识>`：未提交=`working-tree`；某次提交=该 commit 短 hash；区间=`A..B`（清洗成文件名安全字符）；路径过滤 diff=`working-tree-<模块名>` 或 `<区间>-<模块名>`；全量模块=模块目录名（如 `SurgicalNavigation`）。
  - `<YYYYMMDD>`：用系统日期命令实际获取，**不要臆造日期**。
  - 若同名已存在，追加 `-2`/`-3` 等，**不覆盖**旧报告。
- **内容**：写入对话中那份完整报告（审查范围 + 🔴/🟠/🟡 + 待确认）；若导出时修复已完成，追加一段"修复总结"（改了哪些文件、每处目的、已做验证、未验证风险）。
- **编码**：UTF-8，遵循项目新文件编码约定。
- 写完在对话里告知导出的**相对路径**。

## 重要提醒
- 这是**调度器**：真正的审查标准在各专项 skill 里，本 skill 只负责"识别领域 + 编排 + 汇总"。专项 skill 更新了，本 skill 自动受益、无需改动。
- `cross-platform-guard` 对本项目默认关闭，只有出现真实跨平台/编码/字节序/CMake/协议信号才激活。
- `logging-review` 按需激活：日志相关 diff 必跑；设备通信、状态机、病例/规划/导航保存、数据库访问、安全门禁等高风险业务路径可跑；完全无日志风险时明确跳过，不要强行输出日志问题。
- `render-perf-diagnose` 若带 `references/`，仅在进入体绘制或 MITK 刷新深水区时再读取对应 reference。
- 改动小（1~2 个文件、单一领域）时，不必动用子代理编排，主会话直接套那一个专项 checklist 即可——避免杀鸡用牛刀。
- 跨线程/几何有效性/设备失联这类问题常横跨多个专项，去重时优先级最高。
- 不确定某领域是否该激活时，宁可多激活一个（成本低于漏审）。
