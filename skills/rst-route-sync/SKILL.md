---
name: rst-route-sync
description: >
  rst-review 领域路由表同步器：根据**当前项目结构**，用语义判定把每个模块归入
  渲染/Qt/并发/几何变换/手术安全/数据库/跨平台 7 个领域，**纯新增**地把缺失的
  路径信号补进 rst-review 的 Step 2 路由表——已有行/信号一律不动、天然幂等。
  解决"换一个骨科机器人项目后，写死的旧项目专属触发词大量 miss、对应专项
  审查被静默漏掉"的问题。同一批新增会**同步写入** Claude 与 Codex 两份 rst-review。
  本 skill 工具中立，Claude Code 与 Codex（及其它兼容 Agent Skills 的工具）均可直接使用。
  Use this when the user runs /rst-route-sync (or $rst-route-sync), or asks to
  同步路由表 / 更新 rst-review 路由 / 补全触发词 / 让 rst-review 认得这个项目 /
  让相关 review 能在这个项目触发 / sync the rst-review routing table to this project,
  i.e. wants rst-review's Step 2 path signals regenerated (additively) from the current
  project's actual module structure so the right domain review skills still fire.
  触发词：rst-route-sync、同步路由表、更新路由表、更新 rst-review 路由、补全触发词、
  补全路由信号、让 rst-review 认得这个项目、换项目触发不了、路由表对不上、--auto。
---

# rst-review 领域路由表同步器 (rst-route-sync)

`rst-review`（聚合审查调度器）被显式触发后，**唯一**靠它 Step 2 的"领域路由表"决定激活哪些专项审查 skill。那张表的路径信号是**手工维护的项目专属名字**（如某项目特有的导航插件/模块目录名）。一旦换到另一个骨科机器人项目，模块命名不同，路径信号大量 miss，对应专项就**永远不会被激活**——审查出现静默盲区。

本 skill 扫描**当前项目**的真实模块结构，用**语义判定**每个模块该归入哪些领域，把缺失的路径信号**纯新增**地补进 rst-review 的 Step 2 表，并**同步写入 Claude / Codex 两份** rst-review。

> 工具中立：本 skill 只用 Agent Skills 核心格式（name + description + markdown）。只读扫描项目、读写两份 rst-review SKILL.md；**只追加、不删改**已有行与信号。报告与注释用简体中文。

## 设计契约（务必遵守，不可偏离）

1. **只改 rst-review 的 Step 2 路由表，不动其它任何部分**（流程/分级/输出格式都不碰）。
2. **纯新增**：已有行、已有路径信号、已有内容关键词**一律保留不删不改**；只往"路径信号"单元末尾追加缺失项。
3. **幂等**：已被现有信号覆盖的 (模块×领域) 一律跳过；重复运行不产生重复信号。
4. **两份同步**：Claude 版 `~/.claude/skills/rst-review/SKILL.md` 与 Codex 版 `~/.codex/skills/rst-review/SKILL.md` 写入**同一批**新增，保持一致。
5. **不主动构建/测试**（遵循项目 House rule）；验证靠改后重新读取文件。
6. **领域集合固定为 7 个**，不新增/不删减领域，只补它们的路径信号。
7. **路径不写死绝对用户名**：用 `$HOME`/`%USERPROFILE%` 等价位置描述，便于他人/它机复用。

## 使用边界

显式触发信号：`/rst-route-sync` / `$rst-route-sync`、同步路由表、更新 rst-review 路由、补全触发词、让 rst-review 认得这个项目、换项目触发不了。**不在普通编码任务中自动触发**。用户触发本 skill 即代表允许只读扫描项目、并读写两份 rst-review。

参数：默认走"出清单→确认→写"；带 `--auto` 时跳过确认、直接写入再回展示改了什么。

## Step 1 — 定位三方文件并校验一致性

1. **当前项目根**：`git rev-parse --show-toplevel`（失败则用当前工作目录）。后续所有项目内命令在此根目录执行。
2. **两份 rst-review**（按已安装位置查找，不写死某用户名）：
   - Claude：`$HOME/.claude/skills/rst-review/SKILL.md`（Windows 等价 `%USERPROFILE%\.claude\...`）
   - Codex：`$HOME/.codex/skills/rst-review/SKILL.md`、或 `$CODEX_HOME/skills/rst-review/SKILL.md`
3. **读取并比对两份的 Step 2 表**：
   - 若**两份不一致**：停下，报告差异，问用户以哪份为基准（不擅自覆盖分歧）。
   - 若**某份缺失**：只更新存在的那份，并明确提示另一份未找到。
   - 若两份一致：继续。

## Step 2 — 枚举当前项目的模块单元

在项目根扫描源码目录，把每个模块作为一个"单元"：
- `src/Plugins/*`：每个 `Plugin*` 目录为一个单元。
- `src/Image/*`、`src/Algorithm/*`、`src/Common/*`、`src/Kernel/*`、`src/ExtendApplication/*`、`src/Application/*`：各子目录为一个单元。
- **排除** `build/`、`ThirdParty/`、以及隐藏目录。
- 若项目结构非 `src/Plugins` 这种布局：退化为按 `src/`（或仓库根）下的实际子树枚举，并提示"结构与默认假设不同，已按实际目录枚举"。

## Step 3 — 解析现有覆盖（建立"已覆盖"基线）

把现有 Step 2 表**每一行**的"路径信号"单元拆成一组路径模式（精确名/子串/`*Xxx*` 通配），建立映射：

```
领域 → { 已覆盖的路径模式集合 }
```

一个模块算"该领域已覆盖"的判据：其目录名/路径**能被该领域行任一现有路径模式匹配**（子串包含或通配命中均算）。

## Step 4 — 逐模块语义判定（LLM，方案 B）

对 Step 2 的**每个**模块：
- 读目录名 + **抽样读少量** `.h/.cpp` 关键内容（头部、关键 include、关键调用/成员），判断它**应归入哪些领域**（可多选）。
- 领域语义锚点（判定参考，不是硬规则）：
  - **渲染性能**：VTK/ITK/MITK、`RenderingManager`、mapper、体绘制、reslice、窗宽窗位、刷新。
  - **Qt**：几乎所有含 Qt 的 GUI/控件/对话框/Model-View 模块；信号槽、`connect`、`moveToThread`、`deleteLater`、`QTimer`、`connectNotify/emitNotify`。
  - **并发**：设备/通信/采集/控制板/跟踪等带线程的模块；`std::thread`/`QThread`/`std::mutex`/`QMutex`、worker、心跳、串口/TCP 收发。
  - **几何变换**：导航/配准/标定/采点/精度验证/几何算法；Eigen、`Matrix4`、四元数、pose、`inverse/transpose`、LPS/RAS、spacing/origin/direction。
  - **手术安全**：阶段门禁、机械臂、控制板、跟踪丢失、心跳超时、规划/配置/导航的 fail-safe；`OperationState`、`m_isMissing`、`heartbeat`、`ArmMove`、使能、空/无效数据拦截。
  - **数据库**：DAO/登录/SQLite/MySQL；`QSqlQuery`、`prepare/bindValue`、事务、`lastError`、密码/hash。
  - **跨平台**（默认关）：字节序、`#ifdef _WIN32/__linux__`、`wchar_t`、`dllexport`、手写协议字节移位、编码转换、CMake 跨平台。
- 输出每个模块的：`应属领域（可多个） + 判定依据 + 置信度`。
- **无法判定领域**的模块：标"未分类"，**不强行归类**，留到 Step 7 交用户定夺。

> 运行时若支持子代理（Claude Code 的 Agent / Codex 的 spawn_agent），可并行分批判定模块以加速；只读、不改文件。不支持则主会话顺序判定。

## Step 5 — 算差集（只补缺口）

对每个 (模块 × 它应属领域)：
- 若该领域行现有路径信号**无一能匹配**此模块路径 → 记为**待新增**。
- 若已被覆盖 → 跳过（幂等来源）。

## Step 6 — 生成新增信号

- **默认精确名**：用模块目录名作为路径信号（如 `PluginBaseTracker`），拟追加进目标领域行"路径信号"单元末尾。
- **同族归并**：若同一命名族有多个模块都要补（如 `PluginBaseTracker`/`PluginTrackerView`/`PluginBaseTrackerView`），改补一条族通配（如 `*Tracker*`），更省且能兜未来同族新增。归并时确认该通配**不会误命中**其它已知不相关模块。
- **只动"路径信号"列**：内容关键词列默认不改。若某模块带来**明显的新关键词族**，作为**单独一类"内容关键词建议"**列出，**默认不写**，等用户点名才加。
- **cross-platform-guard 默认关**：即便扫到字节序/编码/`#ifdef` 信号，也只进"建议"区、**不自动写**，尊重原表"该项默认关"语义。
- 每条新增携带：`模块 → 目标领域行 → 新增信号 → 理由 → 置信度`。

## Step 7 — 确认闸（默认）/ `--auto`（直写）

**默认**：按领域分组输出"拟新增清单"（见输出格式），交用户**逐条/整体确认**；未分类模块、跨平台建议、内容关键词建议各自单列。**用户确认前不写任何文件。**

**`--auto`**：跳过确认，直接写入全部"路径信号"类新增（仍**不**自动写跨平台与内容关键词建议——这两类永远只建议），写完回展示改了什么。

## Step 8 — 写入两份并保持同步

- 用**精确编辑**把确认的信号**追加**进对应领域行"路径信号"单元的末尾：保持表格 markdown 结构完好、分隔符与顿号风格一致、**不动该行其它内容、不动其它行**。
- 同一批改动**同步写入** Claude 与 Codex 两份；某份缺失则只写存在的并提示。

## Step 9 — 校验与中文汇总

- `~/.claude`、`~/.codex` 通常非 git 仓库，无法 `git diff`：写完**重新读取**两份，确认——新增信号已在目标行、旧信号原样保留、两份一致、表格未破坏。
- 抽查一两个已知缺口模块（如当前项目的导航、规划、跟踪或控制板目录）：确认其名字现在能被目标领域行的路径信号命中。
- 中文汇总：
  - 补了哪些领域行、各加了哪些信号（精确名/通配）。
  - 跳过哪些（已覆盖，体现幂等）。
  - 未能分类的模块清单（交用户定夺）。
  - 跨平台建议、内容关键词建议（均未写，列出供用户决定）。
  - 两份是否已同步、有无某份缺失。

## 输出格式（确认清单）

```text
# rst-review 路由表同步 — 拟新增清单

## 范围
- 项目根：
- 扫描到模块数 / 已覆盖 / 待新增：
- 两份 rst-review 状态（一致 / 差异 / 某份缺失）：

## 拟新增路径信号（按领域分组）
### 并发 (cpp-concurrency-review)
- PluginBaseTracker  ← 理由：读到心跳/worker 线程；置信度：高
- *Pedal* (族通配，覆盖 PluginBasePedal)  ← 理由：…；置信度：中
### 手术安全 (surgical-safety-review)
- ...
### 几何变换 (geometry-transform-review)
- ...

## 未分类模块（请定夺）
- PluginXXX：无法判定领域，原因…

## 仅建议、默认不写
- 跨平台信号建议：…（cross-platform 默认关）
- 内容关键词建议：…

## 待确认
- 确认后将同步写入 Claude / Codex 两份 rst-review 的 Step 2 表（仅追加）。
```

## 重要提醒
- 本 skill **只补路径信号、只追加、不删改**；真正的审查标准在各专项 skill 与 rst-review 里，本 skill 不碰审查逻辑。
- 领域集合固定 7 个；`cross-platform-guard` 与内容关键词新增**永远只建议、不自动写**。
- 路径用 `$HOME`/`%USERPROFILE%` 等价描述，**不写死绝对用户名**，保证换人/换机可复用。
- 改动只 1~2 个模块、信号显而易见时，可直接出极简清单，不必动用子代理。
- 写入前后务必确认 Claude / Codex 两份保持一致。
