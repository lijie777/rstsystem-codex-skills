# 骨科机器人 Agent Skills

面向骨科机器人研发场景的一组 Codex / Claude Code 审查与诊断 skills。它们来自骨科机器人项目的真实 C++/Qt/VTK/MITK 工作流，重点覆盖手术导航、规划、设备通信、医学影像渲染、数据库、跨平台一致性以及失效安全。

> 注意：`rst-review` 现为工具中立的聚合审查入口，Claude Code 与 Codex 都可以使用。Claude Code 运行时使用 Agent 工具调度只读子代理；Codex 运行时使用 `multi_agent_v1.spawn_agent`，没有子代理能力时会降级为主会话顺序审查。`rst-route-sync` 用于把当前项目模块结构同步进 `rst-review` 的领域路由表，适合换项目或新增模块后先执行一次。其他专项 skill 遵循 Agent Skills 的 `SKILL.md` 基本格式，复制到 `.claude\skills` 或 `.codex\skills` 后即可按名称触发。

## 使用背景：骨科机器人软件研发

骨科机器人软件不是普通业务系统。它同时处理患者影像、术前/术中规划、配准矩阵、导航工具位姿、机械臂状态、植入物数据和多窗口医学影像渲染。很多风险不会表现为编译错误，而是表现为：

- 坐标系方向错、矩阵行列序错，导致导航位置偏移。
- 跟踪丢失、设备断连、数据库失败后系统继续使用旧数据或空数据。
- Qt 信号槽跨线程直连，偶发 UI 崩溃或状态错乱。
- VTK/MITK 过度刷新，大体数据交互掉帧。
- MySQL/SQLite 查询失败静默返回空，病例或植入物配置被写坏。
- Windows/Linux 编译器、编码、路径、ABI 差异导致只在某个平台失败。

这 10 个 skill 的目标是把这些经验固化为可复用的审查清单、路由同步能力和 Agent 工作流，让每次改动都能按领域路由到合适的专项审查。

## 覆盖技术栈

- 语言与构建：C++17、CMake、Visual Studio/MSVC、GCC/Clang 跨平台注意事项。
- GUI 与运行时：Qt Widgets、QObject、信号槽、QThread、QTimer、model/view、事件循环。
- 医学影像与渲染：ITK、VTK、MITK、ITK-SNAP 坐标约定、体绘制、reslice、RenderingManager。
- 几何与导航：Eigen、vtkMatrix4x4、四元数、4x4 齐次矩阵、LPS/RAS/IJK、配准链路、工具/法兰/相机/CT 坐标系。
- 手术安全：阶段门禁、规划保存状态、跟踪缺失、设备心跳、机械臂动作、fail-safe/fail-closed 默认策略。
- 数据与持久化：MySQL、SQLite、QSqlQuery/QSqlDatabase、事务、SQL 注入、cereal 序列化、病例/植入物数据完整性。
- 设备与通信：TCP/串口、控制板、OTS/NDI、心跳、worker 线程、共享状态同步。

## 仓库结构

```text
skills/
  rst-review/
    SKILL.md
  rst-route-sync/
    SKILL.md
  qt-review/
    SKILL.md
  cpp-concurrency-review/
    SKILL.md
  geometry-transform-review/
    SKILL.md
  surgical-safety-review/
    SKILL.md
  database-integrity-review/
    SKILL.md
  logging-review/
    SKILL.md
  render-perf-diagnose/
    SKILL.md
    references/
      mitk-rendering.md
      vtk-volume-rendering.md
  cross-platform-guard/
    SKILL.md
```

## 安装方式

在 Windows 环境中，先克隆仓库：

```powershell
git clone https://github.com/lijie777/rstsystem-codex-skills.git
cd .\rstsystem-codex-skills
```

安装到 Codex：

```powershell
$source = ".\skills\*"
$target = "$env:USERPROFILE\.codex\skills"
New-Item -ItemType Directory -Force -Path $target | Out-Null
Copy-Item -Path $source -Destination $target -Recurse -Force
```

安装到 Claude Code：

```powershell
$source = ".\skills\*"
$target = "$env:USERPROFILE\.claude\skills"
New-Item -ItemType Directory -Force -Path $target | Out-Null
Copy-Item -Path $source -Destination $target -Recurse -Force
```

更新已有安装时，在仓库目录执行 `git pull` 后重新复制一次即可。复制后重新打开 Codex / Claude Code 会话，或刷新 skills 列表。Codex 中通常用 `$skill-name`，Claude Code 中通常用 `/skill-name`，例如：

```text
$rst-review 当前修改未提交的文件
/rst-review 当前修改未提交的文件
$rst-route-sync 同步路由表
/rst-route-sync 同步路由表
$qt-review 帮我审查这个 Qt widget 的信号槽和生命周期
/qt-review 帮我审查这个 Qt widget 的信号槽和生命周期
$geometry-transform-review 看一下这段配准矩阵链路是否方向写反
/geometry-transform-review 看一下这段配准矩阵链路是否方向写反
```

## 推荐使用方式

### 0. 新项目或新增模块：先同步路由表

如果把这套 skills 用到一个新的骨科机器人项目，或当前项目新增了导航、规划、机械臂、控制板、配准、渲染、数据库等模块，先运行：

```text
$rst-route-sync 同步路由表
/rst-route-sync 同步路由表
```

它会扫描当前项目模块结构，判断哪些模块应归入 `rst-review` 的 7 个领域，并把缺失的路径信号追加到 Codex / Claude Code 两份 `rst-review/SKILL.md` 的 Step 2 路由表中。默认模式会先输出拟新增清单，等用户确认后再写；如果你已经确认要自动追加路径信号，可使用：

```text
$rst-route-sync --auto
/rst-route-sync --auto
```

同步完成后，再用 `rst-review` 审查未提交改动、提交区间或指定模块，路由命中会更完整。

### 1. 日常改动：先跑聚合审查

在骨科机器人项目里完成一组未提交改动后，优先使用：

```text
$rst-review 当前修改未提交的文件
/rst-review 当前修改未提交的文件
```

`rst-review` 会先读取当前 diff 范围，再根据文件路径和 diff 内容路由到相关专项。例如：

- `.cpp/.h/.ui` 中出现 Qt 信号槽、`QTimer`、`QThread`：路由到 `qt-review`。
- 导航、规划、机械臂、阶段门禁相关改动：路由到 `surgical-safety-review`。
- 坐标矩阵、配准、Eigen/vtkMatrix、LPS/RAS：路由到 `geometry-transform-review`。
- SQL、数据库、病例/植入物表：路由到 `database-integrity-review`。
- 日志宏、日志级别、日志上下文、`qDebug/std::cout` 混用、高频日志风险：路由到 `logging-review`。
- VTK/MITK/ITK 渲染性能路径：路由到 `render-perf-diagnose`。
- 线程、锁、worker、串口/TCP、心跳：路由到 `cpp-concurrency-review`。
- 编码、路径、字节序、CMake、Windows/Linux 差异：路由到 `cross-platform-guard`。

聚合审查阶段只读，不修改文件。发现问题后应先让用户确认要修哪些项，再进入修复。

### 2. 定向问题：直接使用专项 skill

如果你已经知道问题领域，可以直接调用专项，减少上下文和时间消耗。例如：

```text
$database-integrity-review 审一下病例保存和假体数据库查询
$logging-review 审一下这次改动的日志是否足够且不过量
$render-perf-diagnose 诊断这个 MITK 视图拖动时掉帧的原因
$cpp-concurrency-review 看这个控制板 worker 线程 stop 逻辑是否安全
```

## 每个 skill 的用途与用法

### `rst-review`：骨科机器人项目聚合审查入口

适用场景：

- 当前有一组未提交改动，想一次性审查它是否涉及 Qt、并发、几何变换、手术安全、数据库、渲染性能或跨平台风险。
- 不确定该手动调用哪个专项 skill，希望先让聚合入口根据路径和 diff 内容自动路由。
- 想审某个提交、分支区间或模块目录，而不是只看当前工作区。
- 希望 Claude Code 或 Codex 在具备子代理能力时并行做只读审查，再由主会话统一合并、去重和分级。

在 Codex 中使用：

```text
$rst-review 当前修改未提交的文件
$rst-review 审查提交 a1b2c3d 的改动
$rst-review 审查 a1b2c3d..d4e5f6a 的差异
$rst-review 审查 main...HEAD 的差异
$rst-review 审查 src/Modules/SurgicalNavigation 的未提交改动
$rst-review 审查整个 src/Modules/SurgicalNavigation 模块
$rst-review 当前修改未提交的文件 --export
```

在 Claude Code 中使用：

```text
/rst-review 当前修改未提交的文件
/rst-review 审查提交 a1b2c3d 的改动
/rst-review 审查 a1b2c3d..d4e5f6a 的差异
/rst-review 审查 main...HEAD 的差异
/rst-review 审查 src/Modules/SurgicalNavigation 的未提交改动
/rst-review 审查整个 src/Modules/SurgicalNavigation 模块
/rst-review 当前修改未提交的文件 --export
```

常见场景和实际审查范围：

| 场景 | Codex 写法 | Claude Code 写法 | 实际范围 |
|---|---|---|---|
| 审查当前未提交改动 | `$rst-review 当前修改未提交的文件` | `/rst-review 当前修改未提交的文件` | `git diff HEAD`，覆盖工作区和暂存区相对 HEAD 的变化 |
| 审查某次提交的改动 | `$rst-review 审查提交 a1b2c3d 的改动` | `/rst-review 审查提交 a1b2c3d 的改动` | `git diff a1b2c3d^..a1b2c3d` 或等价 `git show a1b2c3d` |
| 审查两个提交差异 | `$rst-review 审查 a1b2c3d..d4e5f6a 的差异` | `/rst-review 审查 a1b2c3d..d4e5f6a 的差异` | `git diff a1b2c3d..d4e5f6a` |
| 审查分支差异 | `$rst-review 审查 main...HEAD 的差异` | `/rst-review 审查 main...HEAD 的差异` | `git diff main...HEAD`，适合看当前分支相对 main 的改动 |
| 审查某模块的未提交改动 | `$rst-review 审查 src/Modules/SurgicalNavigation 的未提交改动` | `/rst-review 审查 src/Modules/SurgicalNavigation 的未提交改动` | `git diff HEAD -- src/Modules/SurgicalNavigation` |
| 审查某模块在某区间内的改动 | `$rst-review 审查 main...HEAD 中 src/Modules/SurgicalNavigation 的改动` | `/rst-review 审查 main...HEAD 中 src/Modules/SurgicalNavigation 的改动` | `git diff main...HEAD -- src/Modules/SurgicalNavigation` |
| 审查某个模块现有全部代码 | `$rst-review 审查整个 src/Modules/SurgicalNavigation 模块` | `/rst-review 审查整个 src/Modules/SurgicalNavigation 模块` | 全量模块模式，读取目录下源码全文，不依赖 git diff |
| 导出当前未提交改动报告 | `$rst-review 当前修改未提交的文件 --export` | `/rst-review 当前修改未提交的文件 --export` | 同时在对话输出，并写入 `docs/code-reviews/` |

范围写法说明：

- `A..B` 表示从 A 到 B 的直接差异，适合比较两个明确提交。
- `main...HEAD` 表示当前分支相对 main 的合并基差异，适合功能分支自查。
- 命令里包含路径且包含“未提交/改动/diff/提交/分支差异”等词时，按路径过滤 diff 处理。
- 命令里包含路径且明确说“整个模块/全量/现有代码”时，按全量模块审查处理。
- 路径建议写仓库相对路径，例如 `src/Modules/SurgicalNavigation`，不要写本机绝对路径。

导出报告：

```text
$rst-review 当前修改未提交的文件 --export
/rst-review 当前修改未提交的文件 --export
$rst-review 审查 main...HEAD 的差异，导出报告
/rst-review 审查 src/Modules/SurgicalNavigation 的未提交改动，并导出报告
```

导出规则：

- 默认只在对话里输出报告，不写文件。
- 只有命令里明确包含 `--export`、`导出报告`、`存档报告` 这类要求时才落盘。
- 启用导出时，当前会话仍必须完整显示检测出来的问题项、分级、来源专项、修法和待确认清单；导出文件只是额外存档，不能用文件路径替代报告正文。
- 输出目录固定为当前项目仓库内的 `docs/code-reviews/`。
- 文件名格式为 `rst-review-<范围标识>-<YYYYMMDD>.md`。
- 常见文件名示例：`rst-review-working-tree-20260609.md`、`rst-review-main...HEAD-20260609.md`、`rst-review-working-tree-SurgicalNavigation-20260609.md`。
- 如果同名文件已存在，会追加 `-2`、`-3`，不会覆盖旧报告。

内部流程：

1. 先确认审查范围，读取 `git diff --stat` 或目标目录文件清单。
2. 按路径和关键词命中专项 skill，例如 Qt、并发、几何、安全、数据库、渲染、跨平台。
3. 对每个命中的专项启动一个只读审查分支；子代理不可用时，主会话按专项清单顺序审查。
4. 收集各专项发现，按同一根因去重，并标注来源专项、置信度和严重级别。
5. 输出审查报告后停止；只有用户逐条确认修复项后，当前会话才进入修改代码阶段。

执行机理：文档驱动的语义路由

`rst-review` 是一份 `SKILL.md` 文档，不是脚本。它不会由某个传统程序去 `grep` 文件名、做正则匹配、再自动 dispatch 专项审查。真实执行链路是：

1. 主会话先运行只读 git 命令，例如 `git diff`、`git diff --stat`，拿到本次审查范围和改动内容。
2. 主会话逐文件阅读 diff，对照 `rst-review` 的领域路由表，结合路径信号、内容关键词和改动语义判断本次改动涉及哪些领域。
3. 主会话决定是否派发对应专项子代理；没有子代理能力时，由主会话按专项 checklist 顺序审查。
4. 子代理也是 LLM。它会读取对应专项的 `SKILL.md`，把该文档当作审查标准，而不是执行某段专项脚本。

因此，路由表的角色是启发式锚点和 checklist：它帮助模型降低漏判风险，但最终决定是否激活某个专项的是模型对 diff 的语义判断，不是机械路径匹配。

这个差异很重要。如果只是按路径通配机械匹配，部分结果会明显不同：

| 文件/改动形态 | 路由表机械匹配可能会怎样 | `rst-review` 实际应如何判断 | 原因 |
|---|---|---|---|
| `SurgicalNavigatView` | 路径含 `*Navigat*`，可能激活 `geometry-transform-review` | 可以不激活几何变换 | 如果 diff 只是文案或 UI 字符串，没有 Eigen、矩阵、位姿、配准链路等真实几何信号，就不应机械激活 |
| `*ConfigView` / `*NavigatView` | 路径含 `*View*`，可能激活 `render-perf-diagnose` | 可以不激活渲染性能 | 如果内容没有 VTK/MITK/ITK、渲染刷新、mapper、actor、volume 等信号，就不应只因 `View` 触发渲染审查 |
| 头文件或配置中新增中文字符串 | `cross-platform-guard` 默认关，机械路由可能不激活 | 可以主动激活跨平台审查 | 如果看到中文字面量、`std::string`、序列化、编码转换或跨平台持久化链路，就属于真实编码/平台风险信号 |

这也带来一个边界：`rst-review` 是 LLM 判断，不是确定性静态分析器。理论上它可能漏判某个领域，也可能为了降低漏判风险多激活一个专项。为此，报告必须写清楚“激活了哪些专项、为什么未激活某些专项”。如果用户认为漏了某个领域，可以直接要求“也用 geometry 审一遍”或“补跑 cross-platform”，当前会话应追加对应专项审查。

输出重点：

- 审查范围。
- 激活/未激活的专项及原因。
- 阻断级、严重、建议问题。
- 每个问题的文件位置、机理、修法。
- 待确认的修复项和验证建议。

两个运行分支的差异：

| 对比项 | Claude Code 分支 | Codex 分支 |
|---|---|---|
| 常用触发写法 | `/rst-review 当前修改未提交的文件` | `$rst-review 当前修改未提交的文件` |
| 推荐安装目录 | `%USERPROFILE%\.claude\skills\rst-review\SKILL.md` | `%USERPROFILE%\.codex\skills\rst-review\SKILL.md` |
| 子代理机制 | 使用 Claude Code 的 Agent 工具 | 使用 Codex 的 `multi_agent_v1.spawn_agent` |
| 子代理类型 | `subagent_type: general-purpose` | `agent_type: "explorer"` |
| 模型选择 | 沿用 Claude Code 当前默认模型；如果设置了 `CLAUDE_CODE_SUBAGENT_MODEL`，则由运行时自动使用 | 不主动设置 `model`、`reasoning_effort`、`service_tier`，除非用户明确指定 |
| 上下文传递 | 给子代理传审查范围、专项 skill 名和必要 diff 摘要 | `fork_context: false`，只传必要范围和专项信息，减少上下文污染 |
| 专项 skill 查找 | 优先使用已加载 skill；必要时查 `.claude/skills`、`.agents/skills`、`.codex/skills` 等路径 | 优先使用已加载 skill；必要时查 `.codex/skills`、`$CODEX_HOME/skills`、`.agents/skills` 等路径 |
| 子代理不可用时 | 主会话按命中专项顺序执行 checklist | 主会话按命中专项顺序执行 checklist |
| 修复阶段 | 用户确认后由当前 Claude Code 会话修复 | 用户确认后由当前 Codex 会话修复 |

输出报告大致长这样：

```text
# 骨科机器人项目聚合审查报告

## 审查范围
- 范围：未提交改动 / main...HEAD / 指定模块路径
- 改动文件：
- 激活专项：
- 未激活专项及原因：

## 阻断级
1. [文件:行][来源专项|置信度]
   问题：
   机理：
   修法：

## 严重
...

## 建议
...

## 待确认
- 建议修复项：
- 建议验证：
```

使用建议：

- 日常开发先用 `rst-review` 做聚合审查；如果报告里某个领域风险很高，再单独调用对应专项 skill 复核。
- 小改动如果只命中一个领域，`rst-review` 可能会直接套专项 checklist，不一定启动子代理，这是为了减少调度成本。
- 审查阶段默认只读，不自动修代码、不提交、不主动运行 CMake 构建或测试。
- 如果需要把审查报告落盘，可在命令里明确写 `--export` 或“导出报告”，报告会写到项目内的 `docs/code-reviews/`，同时当前会话仍会显示完整问题清单。

重要限制：

- 该 skill 同时面向 Claude Code 与 Codex；子代理接口按运行时自适应，缺少子代理能力时会降级为主会话顺序审查。
- Claude Code 分支使用 Agent 工具和 `subagent_type: general-purpose`；Codex 分支使用 `multi_agent_v1.spawn_agent` 和 `agent_type: "explorer"`。
- 它是审查入口，不直接替代各专项 skill。
- 审查阶段只读，不应自动修改代码或运行 CMake 构建。

### `rst-route-sync`：rst-review 路由表同步器

适用场景：

- 换到新的骨科机器人项目后，模块命名和当前 `rst-review` Step 2 路由表不一致。
- 当前项目新增了导航、规划、机械臂、控制板、配准、渲染、数据库、设备通信等模块，希望 `rst-review` 能正确命中相关专项。
- 想只追加缺失路径信号，保留原有路由表，不重写、不删除、不改变审查逻辑。
- Claude Code 与 Codex 两边都安装了 `rst-review`，需要两份路由表保持一致。

在 Codex 中使用：

```text
$rst-route-sync 同步路由表
$rst-route-sync 更新 rst-review 路由
$rst-route-sync 补全触发词
$rst-route-sync 让 rst-review 认得这个项目
$rst-route-sync --auto
```

在 Claude Code 中使用：

```text
/rst-route-sync 同步路由表
/rst-route-sync 更新 rst-review 路由
/rst-route-sync 补全触发词
/rst-route-sync 让 rst-review 认得这个项目
/rst-route-sync --auto
```

默认工作方式：

1. 定位当前项目根目录，以及 Codex / Claude Code 已安装的 `rst-review/SKILL.md`。
2. 比对两份 `rst-review` 的 Step 2 路由表；如果两份不一致，先停下让用户选择基准。
3. 扫描当前项目模块目录，例如 `src/Plugins/*`、`src/Image/*`、`src/Algorithm/*`、`src/Common/*`、`src/Kernel/*`、`src/Application/*`。
4. 按语义判断每个模块应归入哪些领域：渲染、Qt、并发、几何变换、手术安全、数据库、跨平台。
5. 与现有路径信号做差集，只列出尚未覆盖的模块和领域。
6. 默认先输出拟新增清单，等待用户确认后再写入。
7. 写入时只追加 Step 2 表格里的“路径信号”列，不修改内容关键词、分级规则、审查流程或输出格式。

`--auto` 模式：

- 会跳过确认，直接追加“路径信号”类新增项。
- 仍然不会自动写入跨平台建议，因为 `cross-platform-guard` 在 `rst-review` 中默认保持谨慎触发。
- 仍然不会自动写入内容关键词建议，这类建议只在用户明确点名时才应加入。
- 写完后会重新读取 Codex / Claude Code 两份 `rst-review`，确认新增信号存在、旧信号保留、两份内容一致。

输出重点：

- 当前项目根目录。
- 扫描到的模块数量。
- 已覆盖和待新增的模块。
- 按领域分组的拟新增路径信号。
- 每条新增的理由和置信度。
- 未分类模块清单。
- 仅建议、不默认写入的跨平台信号和内容关键词。
- Codex / Claude Code 两份 `rst-review` 是否一致。

使用建议：

- 第一次把本仓库安装到一个新项目后，先运行一次 `rst-route-sync`，再运行 `rst-review`。
- 大规模新增模块、模块改名、插件目录重组后，再运行一次 `rst-route-sync`。
- 平时小范围改代码，不需要每次都运行；直接用 `rst-review` 审查当前 diff 即可。
- 如果拟新增清单里出现明显误分类，先让用户调整清单，不要直接 `--auto`。

重要限制：

- 它只维护 `rst-review` 的路由命中能力，不做代码审查，也不替代各专项 skill。
- 它只追加路径信号，不删除旧信号，不合并或重写整个表格。
- 它不主动运行 CMake、构建或测试。
- 如果 Codex / Claude Code 只安装了一份 `rst-review`，就只更新存在的那份，并在结果中说明另一份缺失。

### `qt-review`：Qt 运行时与 UI 审查

适用场景：

- Qt Widgets、手写 `setupUi()`、`.ui` 文件、model/view 逻辑。
- `connect`、`SIGNAL/SLOT`、`Qt::DirectConnection`、`QTimer`、`QThread`、`moveToThread`。
- QObject parent/生命周期、`deleteLater`、跨线程 UI 访问。
- 项目自研 `connectNotify/emitNotify` 通知总线。

典型用法：

```text
$qt-review 审查这个 widget 的信号槽和对象生命周期
$qt-review 看看这个 DirectConnection 是否会跨线程访问 UI
```

关注问题：

- 旧式字符串信号槽签名不匹配导致运行时静默失败。
- worker 线程发信号，DirectConnection 让槽在错误线程执行。
- 带 parent 的 QObject 被 moveToThread。
- `blockSignals(true/false)` 裸调用漏恢复。
- 动态重建 layout 时 widget 泄漏或重复删除。

### `cpp-concurrency-review`：C++/Qt 并发审查

适用场景：

- `std::thread`、`std::mutex`、`std::atomic`、条件变量、线程池。
- Qt `QThread`、`QMutex`、`QWaitCondition`、`QSemaphore`。
- 设备 worker、串口/TCP、心跳、停止线程、共享状态。

典型用法：

```text
$cpp-concurrency-review 审查这个采集线程和 stop 逻辑
$cpp-concurrency-review 看看控制板心跳和状态变量有没有数据竞争
```

关注问题：

- 普通 `bool` 被多个线程读写，没有 atomic 或锁。
- `detach` 后捕获 `this`，对象析构后仍访问。
- 条件变量裸 wait，没有谓词。
- 手动 lock/unlock 中途 return 漏解锁。
- 持锁期间发信号、做 I/O 或调用未知回调。

### `geometry-transform-review`：坐标系与导航几何审查

适用场景：

- 配准矩阵、导航工具位姿、机械臂法兰、相机/工具/CT/Spine 坐标系。
- Eigen/vtkMatrix4x4/float[16] 转换。
- DICOM LPS/RAS/IJK，VTK/MITK physical 坐标。
- 四元数、欧拉角、角度/弧度、矩阵有效性。

典型用法：

```text
$geometry-transform-review 检查这条 Tool->Flange->Spine->CT 变换链
$geometry-transform-review 看这个 float[16] 到 Eigen 的转换行列序是否正确
```

关注问题：

- `A2B` 命名方向与矩阵乘法方向不一致。
- 该求逆的矩阵没求逆，或把 4x4 齐次矩阵 transpose 当 inverse。
- 四元数转矩阵前没 normalize。
- float[16] 行主序/列主序串用。
- `acos` 入参不 clamp，零向量归一化导致 NaN。

### `surgical-safety-review`：手术失效安全审查

适用场景：

- 手术阶段门禁、规划保存状态、导航入口校验。
- 跟踪丢失、设备断连、心跳超时、机械臂动作。
- 植入物配置、案例状态、无有效数据时是否 fail-safe。

典型用法：

```text
$surgical-safety-review 审查导航入口门禁是否 fail-closed
$surgical-safety-review 看看跟踪 missing 时是否还会继续用旧位姿
```

关注问题：

- 服务不可用或配置为空时默认放行。
- 修改 pose/transform 后没有回退“已规划”状态。
- 跟踪丢失后继续显示旧位姿。
- 心跳超时只断连接，不报警或不停机。
- 机械臂运动命令缺少范围、速度、奇异性校验。

### `database-integrity-review`：数据库与数据完整性审查

适用场景：

- MySQL/SQLite、`QSqlQuery`、`QSqlDatabase`。
- 病例、用户、植入物、规划配置持久化。
- SQL 拼接、事务、登录密码、字段约束、查询失败处理。

典型用法：

```text
$database-integrity-review 审查病例保存和删除逻辑
$database-integrity-review 看这个假体数据库查询是否会静默失败
```

关注问题：

- 字符串拼接 SQL 导致注入。
- 查询失败返回空集合，上层误当“无数据”。
- 批量/多表写入没有事务。
- 明文密码、默认弱口令、hash 存储和比对不一致。
- `QSqlDatabase` 单例跨线程共享。
- 关键字段无约束或无写前校验。

### `logging-review`：业务日志设计与审查

适用场景：

- 给关键业务类、设备通信、状态机、病例/规划/导航保存流程设计日志点。
- 审查 `LOG_*`、`DB_LOG_*`、`qDebug`、`std::cout/std::cerr` 等日志改动。
- 判断日志是否足够、是否过量、级别是否正确、上下文是否可追溯。

典型用法：

```text
$logging-review 审一下这次改动的日志是否足够且不过量
$logging-review 给 SurgicalPlanWidget 的保存流程设计日志方案
```

关注问题：

- 关键操作、错误路径、外部通信和状态切换缺少可追溯日志。
- 高频刷新、渲染、采样、逐帧位姿路径写入常规 INFO 日志。
- `qDebug/std::cout/std::cerr` 绕过统一日志系统。
- `LOG_*` 使用 `%s/%d` 或字符串拼接，和 spdlog/fmt 风格不一致。
- 日志级别膨胀或不足，安全失效没有使用 `LOG_ERROR` 和可见告警。
- 日志缺少 caseId、路径、状态、错误码等关键上下文，或写入患者 PHI。

### `render-perf-diagnose`：ITK/VTK/MITK 渲染性能诊断

适用场景：

- 首次加载慢、滚切片卡、旋转掉帧、窗宽窗位拖动慢。
- VTK mapper、volume rendering、mesh rendering、MITK `RenderingManager`。
- ITK filter 重算、ITK/VTK 数据转换、体数据/网格规模过大。

典型用法：

```text
$render-perf-diagnose 诊断这个 MITK 视图旋转时掉帧
$render-perf-diagnose 看看体绘制为什么首次加载很慢
```

关注问题：

- 没有计时就下结论。
- ITK filter 每次交互都被 Modified 触发重算。
- VTK pipeline 每帧重执行或重复上传大数据。
- 体绘制用了 CPU mapper，采样步长过小。
- MITK 每次操作 `RequestUpdateAll()`，刷新过多窗口。

附带参考：

- `references/vtk-volume-rendering.md`
- `references/mitk-rendering.md`

这些参考只在进入体绘制或 MITK 刷新深水区时展开。

### `cross-platform-guard`：跨平台一致性审查

适用场景：

- Windows/Linux 路径、编码、换行、大小写、动态库、socket。
- MSVC/GCC/Clang 差异、CMake 平台分支。
- 字节序、结构体对齐、固定宽度整数、`long` 大小。

典型用法：

```text
$cross-platform-guard 审查这个路径和编码处理是否跨平台
$cross-platform-guard 看这个协议包解析在 Windows/Linux 是否一致
```

关注问题：

- 文件名大小写在 Windows 能过、Linux 编不过。
- `std::filesystem::path.string()` 在中文路径下编码不一致。
- `long` 在 Windows 64 位仍是 32 位，Linux 64 位是 64 位。
- 直接 `memcpy` 结构体落盘或走网络，受 padding/字节序影响。
- Windows DLL 符号导出和 Linux visibility 未统一。
- CMake 全局编译选项污染，未按 target 作用域配置。

## 聚合审查建议流程

1. 先用 `git status -sb`、`git diff --stat` 确定范围。
2. 如果是新项目、模块改名或新增大量模块，先用 `$rst-route-sync 同步路由表` 或 `/rst-route-sync 同步路由表` 更新 `rst-review` 路由表。
3. 使用 `$rst-review 当前修改未提交的文件` 或 `/rst-review 当前修改未提交的文件` 做聚合审查。
4. 对高风险结论，再直接调用对应专项复核。
5. 用户确认修复项后再改代码。
6. 修复后做轻量验证：`git diff --check`、目标 `rg`、源码检查。
7. 构建/运行/CMake 验证只在用户明确要求时执行，避免在大型医疗软件仓库里无意触发长时间构建。

## 适用边界

适合：

- 骨科机器人、导航机器人或类似医疗机器人项目。
- C++/Qt/VTK/MITK/ITK/Eigen/CMake 技术栈。
- 医学影像、导航几何、机器人设备通信、数据库持久化相关审查。

不适合直接替代：

- 正式法规验证、IEC 62304 风险管理文档。
- 医学/临床参数最终确认。
- 真实硬件联调、机械臂安全认证。
- 构建系统和 CI 的实际双平台验证。

这些 skill 是工程审查辅助，不是安全认证结论。

## 维护建议

- 当项目新增模块、数据库表、坐标系或设备协议时，同步更新对应专项 skill。
- 对真实线上 bug 进行复盘后，把“形状”和“修法”补进 skill 的本项目易错点。
- `rst-review` 的路由表要保持轻量，只负责分发；专项细则放在各自 skill 内。项目结构变化后优先用 `rst-route-sync` 追加路径信号，而不是手工重写整张路由表。
- 对 Claude Code / Codex 以外的兼容 Agent 使用时，可以复用专项清单和聚合流程；如果该运行时没有等价子代理接口，则按主会话顺序审查执行。
