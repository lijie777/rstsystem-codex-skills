---
name: rst-review
description: Use when reviewing RSTSystem changes with a comprehensive Codex workflow that routes the current diff to relevant Qt, concurrency, geometry, surgical safety, database, render performance, and cross-platform specialist skills.
---

# RSTSystem Codex 聚合审查

本 skill 是 Codex 版 RSTSystem 聚合审查调度器。它负责识别当前改动涉及的领域，调用对应的专项 skill，并在可用时通过 Codex 子代理并行审查。审查阶段只读，不修改文件；修复必须等用户逐条确认。

一个改动常常**同时**踩到多个领域（改个"植入物位姿调整后的 3D 刷新"就同时涉及渲染、Qt 信号槽、几何变换三块）。本 skill 自动识别本次改动涉及哪些领域，把对应的专项审查**并行**跑一遍，再汇总成一份分级报告——你不用记 skill 名、不用逐个手动调用。

> 优先级说明：本 skill 是 RSTSystem 聚合审查入口。被显式触发（`rst-review`、`$rst-review`、"聚合审/全审"）时，按本文流程走，不要再叠加其它通用 review 流程。审查报告用简体中文。

## 使用边界

使用本 skill 当用户明确说：

- `rst-review`
- 聚合审、综合审、全审、一键审查
- 把相关 review skill 都跑一下
- 按 RSTSystem 相关专项审一遍
- comprehensive review of my changes

不要在普通编码任务中自动触发。用户触发本 skill 本身即表示允许使用子代理做只读审查。

## Step 1 — 确定审查目标与模式

rst-review 支持两种模式，先判断用户要哪种（不清楚就问一句"审这次改动，还是审整个模块的现有代码？"）：

### 模式 A — 改动审查（diff 模式，默认）
审"变化了什么"。按优先级：
1. 用户给了 **git ref / 区间**（如某 commit、`A..B`、`main...HEAD`）→ 审对应 diff：
   - 单个提交：`git show <commit>` 或 `git diff <commit>^..<commit>`
   - 区间/分支差异：`git diff <A>..<B>` / `<A>...<B>` / `<commit>..HEAD`
2. 无参数 → **未提交改动**：`git diff HEAD`（工作区 + 暂存区 vs HEAD）。
3. 工作区干净 → 回退审**当前分支相对 main 的全部改动**：`git diff main...HEAD`。

以上 git 命令均在**项目仓库根目录**执行（不写死绝对路径，便于他人/其它机器复用）；先用 `git diff --stat` 拿改动文件清单，再做 Step 2 领域路由。

### 模式 B — 全量模块审查（whole-module 模式）
审"一个已存在模块/目录/文件的**现有全部代码**"，与是否改动无关。
- **触发**：用户给的是文件/目录**路径**，或说"审整个 X 模块 / 这个插件 / 这些文件"。
- **取范围**：用 Glob 列出目标下的源码（`*.cpp/*.h/*.cxx/*.ui` 等；排除 `build/`、`ThirdParty/`）。
- **控制成本**：整模块可能很大。按 Step 2 路由命中的专项，让每个子代理**只读与其领域相关的文件**全文审查；文件过多时提示用户范围偏大，建议聚焦关键子目录/核心文件，或分批审。

无论 A/B，都**不要主动构建/测试**（项目 House rule），验证靠源码阅读与 `git diff`。

## Step 2 - 领域路由

根据改动文件路径和 diff 内容激活专项。一个文件可命中多个专项。

| 专项 skill | 路径信号 | 内容关键词信号 |
|---|---|---|
| `qt-review` | 任意 `.cpp/.h/.ui` 且含 Qt | `connect(`、`SIGNAL(`、`SLOT(`、`moveToThread`、`QThread`、`connectNotify`、`emitNotify`、`deleteLater`、`QTimer`、`blockSignals`、`QSortFilterProxyModel`、`QDialog::exec`、`QByteArray`、`readyRead` |
| `cpp-concurrency-review` | `PluginBase*Board`、`PluginOTS`、`PluginBaseOTS`、`*Thread*`、`SerialPort*`、`TcpServer` | `std::thread`、`std::mutex`、`std::atomic`、`QMutex`、`QWaitCondition`、`QSemaphore`、`detach`、`run()`、`volatile`、心跳、worker |
| `geometry-transform-review` | `src/Algorithm/GeometryAlgorithm/`、`Transforms3d`、`PluginSurgicalNavigatView`、`PluginRegistrationTransformationView`、`PluginToolRegistrationView`、`commonData.h` | `Eigen`、`Matrix4`、`Matrix3`、`Quat`、`Quaternion`、`pose`、`Transform`、`inverse()`、`transpose()`、`normalize`、`acos`、配准、`spacing`、`origin`、`Direction`、`LPS`、`RAS`、`float[16]`、`vtkMatrix` |
| `surgical-safety-review` | `PluginSurgicalPlanView`、`PluginSurgicalConfigView`、`PluginSurgicalNavigatView`、`PluginBaseRoboticArm`、`PluginBase*Board`、`MainView`、`CoreUi` | `OperationState`、阶段切换、门禁、`canEnter`、`m_isMissing`、`heartbeat`、`timeout`、`ArmMove`、机械臂、使能、`KMessageBox`、植入物、案例状态、无有效植入物、未选节段 |
| `database-integrity-review` | `src/Common/DataBase/`、`src/Common/SQLiteAliasManager/`、`PluginBaseLoginView`、`DB*Impl`、`SQLite*Manager` | `QSqlQuery`、`QSqlDatabase`、`exec(`、`prepare(`、`bindValue`、`SELECT`、`INSERT`、`transaction`、`commit`、`rollback`、`lastError`、`loginPass`、password、hash、`MAX(` |
| `render-perf-diagnose` | `src/Image/VTKRenderViewer/`、`PluginSegmentView`、`PluginMitkCasePreview`、`casepreviewview`、`*View*Wgt`、`ViewPanel*` | `vtk`、`itk`、`mitk`、`RenderingManager`、`vtkMapper`、`volume`、`reslice`、`RequestUpdate`、`level window`、`marching`、`actor`、`renderer`、渲染、刷新、掉帧 |
| `cross-platform-guard` | 默认不激活，仅在出现明确跨平台/协议/编码信号时激活 | `#ifdef _WIN32`、`__linux__`、`std::filesystem`、字节序、`<<8`、`htons`、`wchar_t`、`TCHAR`、`dllexport`、visibility、`fromLocal8Bit`、编码转换、跨平台 CMake |

如果改动只是注释、文案或格式，说明无需专项审查并停止。

## Step 3 - 子代理并行审查

当 `multi_agent_v1.spawn_agent` 可用时，为每个激活专项派一个只读子代理。若当前工具列表没有子代理工具，先用 `tool_search` 搜索 multi-agent/subagent 工具；仍不可用时，降级为主会话顺序审查。

子代理规则：

- 使用 `agent_type: "explorer"`，因为任务是只读审查。
- 不设置 `model`、`reasoning_effort` 或 `service_tier`，除非用户明确指定。
- `fork_context` 通常设为 `false`，只传必要范围、diff 摘要和专项 skill 路径，减少上下文污染。
- 每个子代理只审自己领域，不修改文件，不提交，不运行构建。
- 如果专项少于 2 个或改动很小，也可以主会话直接套对应 checklist 审查，避免过度调度；但用户明确要求子代理时必须使用子代理。

子代理 prompt 模板：

```text
你是 RSTSystem 项目的 <专项名> 只读代码审查员。

请读取并遵守这个专项 skill：
<专项名>

如果当前会话无法自动加载该 skill，可在 Codex skills 根目录下查找：
<Codex skills root>/<专项名>/SKILL.md

常见根目录是 `$CODEX_HOME/skills` 或 `%USERPROFILE%\.codex\skills`。不要写死某个用户目录；如果无法定位 skill 文件，明确报告该专项无法完成。

审查范围：
<文件列表、git diff 摘要、用户指定范围；必要时允许你用只读 git diff/rg/Get-Content 查看上下文>

要求：
1. 只审与你专项相关的问题。
2. 只报可定位、可解释、可行动的问题；不要泛泛而谈。
3. 每条发现必须包含：文件:行或函数、问题、为什么危险、修法方向、严重度、置信度。
4. 只读审查，不修改文件，不运行 CMake 构建或测试，不提交。
5. 用中文输出结构化发现清单；没有问题就明确说没有发现该专项问题，并说明残余风险。
```

等待策略：

- 先完成本地范围识别和非重叠检查，再 `wait_agent`。
- 可一次等待多个 agent。
- 不反复忙等；如果超时，先汇总已返回结果并说明哪些专项未完成。

## Step 4 - 汇总、去重、分级

收齐结果后按以下方式汇总：

- 同一文件/函数/根因被多个专项命中时合并为一条，标注多领域命中。
- 严重度排序：阻断级、严重、建议。
- 每条保留来源专项和置信度。
- 对不确定结论明确标注“需运行验证/需查文档/需用户确认”，不要把推断写成事实。

分级参考：

- 阻断级：崩溃、跨线程 UI/对象访问、坐标打偏、SQL 注入、明文密码、带错误数据继续、设备失联不 fail-safe、危险物理动作无校验。
- 严重：偶发错误、状态残留、事务半写、缓存帧无新鲜度、性能路径明显重复刷新、连接/生命周期风险。
- 建议：风格一致性、局部可维护性、日志级别、验证补强。

## Step 5 - 确认与修复

审查报告发给用户后停止，等待用户确认要修哪些项。用户确认前不要改代码。

用户确认后由 Codex 当前会话修复，遵守仓库规则：

- 修改范围只覆盖用户确认的问题。
- 不主动运行 CMake configure/build/test。
- 默认使用中文注释和中文日志。
- 修复后用 `git diff`、`git diff --check`、`rg` 和源码检查做轻量验证。
- 最终回复说明修改文件、目的、行为影响、已验证内容和未验证风险。

## 输出格式

```text
# RSTSystem 聚合审查报告

## 审查范围
- 范围：
- 改动文件：
- 激活专项：
- 未激活专项及原因：

## 阻断级
1. [文件:行][来源专项|置信度][多领域命中?]
   问题：
   机理：
   修法：

## 严重
...

## 建议
...

## 待确认
- 建议修复项清单：
- 未完成/超时专项：
- 建议验证：
```

## 注意事项

- `cross-platform-guard` 对 RSTSystem 默认关闭，只有出现真实跨平台、编码、字节序、CMake 或协议信号才激活。
- `render-perf-diagnose` 包含 `references/`，只有进入体绘制或 MITK 刷新深水区时再读取对应 reference。
- 聚合审查是审查入口，不替代专项 skill；专项规则更新后，本 skill 只需要继续路由到对应专项。
- 如果当前会话的已注入技能列表还没刷新，仍可通过 Codex skills 根目录下的 `<专项名>/SKILL.md` 读取对应专项规则。
