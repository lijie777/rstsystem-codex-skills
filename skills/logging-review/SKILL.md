---
name: logging-review
description: Use when adding or reviewing business logs in this orthopedic navigation project — designing a logging plan for a class, or auditing whether logs are sufficient / excessive / correct level / well-formatted. 触发词：给某类加日志、补日志、日志够不够、会不会打太多、审一下日志、这条日志该什么级别、关键操作/错误/通信/状态/设备检查日志、可追溯、审计留痕；以及 KALog、LOG_INFO/LOG_WARN/LOG_ERROR/LOG_CRITICAL、LOG_*_CL、DB_LOG_*、SUFFIX、spdlog/fmt 风格日志宏的改动。即使用户只是贴一段代码问“这里日志够不够 / 会不会太多 / 该打什么级别”也应主动使用本 skill。
---

# 业务日志设计与审查 (logging-review)

医疗器械软件的日志目标不是“尽量多记”，而是：**关键操作可追溯，错误路径可定位，持续运行不刷屏**。多了会把日志文件打爆、淹没真信号；少了出了事故无法复盘。本 skill 有两种用法——**设计**（给一个业务类产出日志方案）和**审查**（审现有代码或 diff 的日志是否合规），二者共用同一套标准。

权威源是 Obsidian vault 里的《医疗器械软件关键业务模块日志设计规范》（标记 canonical），本 skill 是它在 RSTSystem 上的**项目化执行版**，已按本项目 KALog 的真实约定做了修正，不复制全文。

## 用前先确认四件事
1. 这段代码是否**关键业务 / 外部通信 / 状态变更**，而不只是纯展示、getter/setter、高频刷新？
2. 它属于哪个 `OperationState` 阶段、依赖哪些设备 / 数据 / 服务？
3. **失败和异常路径**走到哪？是否需要能事后定位的错误日志？
4. 它是否在**高频 / 渲染 / 采样 / 逐帧**路径上？（决定能不能打常规日志）

纯展示、静态配置页、getter/setter、逐帧位姿、持续定位流——不要机械套用本规范。

## 0. 本项目 KALog 硬约定（最容易写错，先记牢）

KALog 基于 spdlog + fmt，宏定义在 `src/Kernel/Log/KALog.h`。

- **默认前缀用 `类名: ` 或 `类名::关键入口: `（冒号风格），中文文案**。这是项目主流，适合普通类、Widget、算法和服务实现。正例：
  - `LOG_WARN("ImageProcessAlgorithm::cropImageByPolyData: input is null")`
  - `LOG_INFO("SpineCaseManager: showEvent刷新案例列表，查询数量={}", records.size())`
  - 中文功能前缀也可：`LOG_WARN("手术导航入口校验失败：未找到规划文件，路径={}", path)`
  - 已有稳定领域前缀可保留，例如 `[DB]`、`[安全拦截]`、`[DicomReceiver]`、`[Import]`、外部进程/通信通道前缀。审查时关注它是否稳定、可检索、与周边模块一致。
  - ❌ 不要临时新造多套 `[模块名]` 方括号风格；❌ 不要新增 `LOG_PREFIX_*` QString 常量来包装前缀，除非周边文件已经采用同一模式。
- **只用 fmt `{}` 占位**。QString 先 `.toStdString()` 再传。禁止：
  - `LOG_ERROR(QString("错误: %1").arg(v))`（宏外 arg 拼接）
  - `LOG_ERROR("错误: %s", v.c_str())`（C 风格 `%s`）
  - `std::cout << ...` / `std::cerr << ...`（绕过日志系统）
  - 正式业务代码里的 `qDebug() << ...`
- **级别 + SUFFIX 自动后缀（最关键的一条）**：`LOG_WARN / LOG_ERROR / LOG_CRITICAL` 会经 `SUFFIX` 宏**自动在末尾追加 `<文件> <函数> <行号>`**。
  - → 警告/错误日志不要为了定位调用栈而重复写函数名：`LOG_ERROR("Exception in loadProsthesisData: {}", e.what())` 里的 `loadProsthesisData` 是多余的（SUFFIX 已带 `__FUNCTION__`），写 `LOG_ERROR("加载假体数据异常: {}, ID: {}", e.what(), id)` 即可。
  - → 允许保留 `类名: 业务事件` 或少量 `类名::关键入口: 业务事件` 作为检索前缀；审查重点是去掉无业务含义的 `Exception in xxx` / `enter xxx` / `called xxx`。
  - → 但 `LOG_INFO / LOG_DEBUG / LOG_TRACE` **不追加任何上下文**，必须自己带全 caseId / 路径 / 关键参数，否则只剩一句没法定位的话。
- **SUFFIX 的两个坑**：
  - `LOG_WARN/ERROR/CRITICAL` 的**第一个参数必须是字符串字面量或 `std::string`**（SUFFIX 对它做 `std::string(msg).append(...)`），不能直接塞 `QString`。
  - 第一个参数里若含**字面 `{` 或 `}`** 又没有对应实参，会被 spdlog 当占位符抛异常——纯文本含花括号要写 `{{` / `}}`，或改成带参数的形式。
- **三套宏怎么选**：`LOG_*` = 写文件（默认用这个）；`LOG_*_CL` = 只输出控制台；`DB_LOG_*` = 文件 + 控制台双写（数据库层等需要同时上屏排查的场景）。
- **运行级别现实**（`KALog.cpp`）：文件 logger = trace（全开），控制台 = INFO，error 立即 flush、每 3 秒 flush 一次。含义：
  - **INFO 及以上才是长期业务追溯主通道；TRACE/DEBUG 仅作临时排查**，不能在正式运行里持续大量输出。
  - **INFO 绝不能进高频 / 逐帧 / 采样路径**——文件级别全开，INFO 刷屏会直接打爆日志文件。

## 1. 应记录的位置（只记关键路径）

| 位置 | 记录内容 | 级别 |
|---|---|---|
| 构造 / 初始化入口 | 初始化开始、依赖状态、完成 | INFO |
| 析构 / 资源释放 | 释放开始、关键资源结果 | INFO |
| 外部服务连接 | 是否可用、连接结果、失败原因 | INFO / ERROR |
| 回调注册 | 注册对象、结果、重复或缺失注册 | INFO / WARN |
| 用户触发关键操作 | 操作名、操作前状态、关键输入 | INFO |
| 前置条件检查 | 设备 / 配置 / 数据 / 状态是否满足 | WARN / ERROR |
| 指令发送 | 指令类型、目标、关键参数、发送结果 | INFO / ERROR |
| 结果回调 | 返回状态、错误码、业务结果 | INFO / ERROR |
| 状态切换 | 切换前后状态、触发来源 | INFO / WARN |
| 异常分支 | 错误原因、影响范围、是否阻断主流程 | ERROR |

## 2. 不应记录的位置
- 高频循环、定时器刷新、持续采样、实时渲染、逐帧位姿 / 坐标变化、普通 UI 刷新。
- 无业务含义的 getter/setter、内部变量赋值。
- 已被上层关键操作日志覆盖的重复中间步骤。
- 仅为“证明函数被调用过”而打的日志。
- 确需观察高频逻辑 → 用限频 / 采样 / DEBUG 开关 / 临时诊断工具，不进长期业务日志。

## 3. 级别边界
| 级别 | 边界 |
|---|---|
| TRACE | 临时、可开关的极细路径，默认不长期输出 |
| DEBUG | 开发调试（参数值、中间结果），默认不进长期业务日志 |
| INFO | 正常关键业务流程和重要状态变化 |
| WARN | 状态异常但暂不阻断主流程（不一致、重复触发、非关键数据缺失） |
| ERROR | 功能无法继续 / 结果失败（设备不可用、连接失败、解析失败） |
| CRITICAL | 安全相关操作失败、不可恢复状态——**不滥用**，仅影响临床安全/系统可用性时 |

## 4. 字段最小集（不要只写“失败了”）
每条关键日志至少 **事件 + 状态 + 关键上下文 + 结果**。错误日志必须带“失败前的设备状态 / 参数 / 返回码”，否则无法定位。

- ❌ `LOG_ERROR("加载失败")`
- ✅ `LOG_ERROR("假体数据加载失败，ID: {}, 错误码: {}", id.toStdString(), code)`
- ❌ `LOG_INFO("保存成功")`（INFO 无 SUFFIX，等于没说）
- ✅ `LOG_INFO("SurgicalPlan: 规划保存成功，caseId={}，节段数={}", caseId.toStdString(), n)`

## 5. 全链路锚点 + 与其它专项审查呼应
跨模块链路（UI → Service/Processor → Render/Device）只在**关键节点**打日志，事后只看日志要能回答“数据在哪一级被拦截、误判、丢失或转换失败”。不是每个函数入口都打。

本项目真实锚点：
- **阶段门禁**：`OperationState` 流转、`MainView::mainLayoutModuleEntryDeniedReason`、`currentCaseHasCtPath` / `currentSurgicalConfigHasSelectedLevels` / `currentSurgicalPlanHasAllSavedSegments`——门禁拒绝要打一条**带中文原因**的日志。
- **设备 / 跟踪**：心跳超时、跟踪 `m_isMissing`、配准矩阵无效。
- **通知总线**：`ServerBase::emitNotify` / `connectNotify` 的关键收发。

⚠️ **呼应 [[surgical-safety-review]]**：安全相关失效（OTS missing、心跳超时、配准无效、机械臂异常）**必须 `LOG_ERROR` 并触发可见告警，不能用 `LOG_INFO` 淹没**——这是本项目系统性偏弱处，审查时重点抓。
**呼应 [[database-integrity-review]]**：病例 / 患者 / 假体等关键数据的访问与修改要留痕；`caseId` 可作为内部追踪字段，患者姓名、身份证号、手机号、完整患者 ID、含患者信息的路径等 PHI 默认避免明文写入日志。若现有需求要求记录，必须说明审计目的、脱敏方式和访问控制。

## 6. 审查模式 Checklist
通用：
- [ ] 这个类是否承载关键业务，而非纯展示？
- [ ] 初始化 / 释放 / 服务连接 / 回调注册是否记录？
- [ ] 关键操作是否记了开始、关键参数、结果？
- [ ] 每个关键操作的前置条件失败是否有日志？
- [ ] 指令发送是否记了目标、参数摘要、发送结果？
- [ ] 回调是否记了返回状态、错误码、业务结果？
- [ ] 状态机切换是否记了前后状态和触发来源？
- [ ] 错误日志上下文是否够定位问题？
- [ ] 高频 / 刷新 / 采样路径是否避免了常规日志？
- [ ] 前缀、术语是否统一、便于检索？

项目专属（最常踩）：
- [ ] 普通类日志优先使用 `类名: ` 冒号前缀；已有 `[DB]` / `[Import]` / 外部通信等领域前缀保持稳定一致？
- [ ] WARN/ERROR/CRITICAL 没有重复手写无业务含义的函数名（SUFFIX 已带）？
- [ ] INFO/DEBUG/TRACE 自带了 caseId/路径等上下文（它们没有 SUFFIX）？
- [ ] 没有 `qDebug`/`cout`/`cerr`/`QString::arg` 拼接？
- [ ] 安全失效是 `LOG_ERROR`（而非 INFO）且有告警？
- [ ] 没有把患者姓名、身份证号、手机号、完整患者 ID、含患者信息的路径等 PHI 明文写进日志？
- [ ] 纯文本里的花括号已转义、WARN/ERROR 第一参数是字面量/std::string？

## 7. 审查输出格式
审查日志时先给结论，再给证据。不要只复述规范；每个问题都要落到文件、函数或 diff 片段。

### 审查范围
说明本次查看的文件、函数、diff 或模块，以及是否只审日志相关问题。

### 总体结论
用一句话判断：`通过`、`有轻微问题`、`有明显风险` 或 `不建议合入`。没有问题时明确说“未发现日志设计问题”，不要强行凑问题。

### 问题列表
按 `高 / 中 / 低` 分级。每条问题包含：
- 位置：文件 + 行号 / 函数 / diff 片段。
- 类型：缺日志、日志过量、级别错误、格式错误、上下文不足、敏感信息、绕过统一日志、不可检索。
- 证据：当前代码表现。
- 风险：为什么影响定位、追溯、审计、运行稳定性或日志容量。
- 建议：加日志、删日志、降级、限频、改格式、补上下文或脱敏。

### 不建议加日志的位置
明确指出高频循环、渲染刷新、持续采样、getter/setter、普通 UI 刷新等不应补常规日志的位置，避免 review 诱导过度打点。

### 建议修改
给出最小修改建议，只覆盖本次 diff 或用户指定范围。不要默认大范围清理历史日志。

### 验证建议
列轻量验证即可：`rg`、`git diff --check`、源码检查、必要的手工触发路径。不要要求 CMake configure/build/test，除非用户明确要求。

## 8. 设计模式工作流（给一个业务类设计日志方案）
1. 先画出该类的**关键业务路径**，再定日志点——不要每个函数入口都打。
2. 按 **生命周期 / 业务流程 / 错误处理 / 通信 / 状态变更 / 设备前置检查** 六类输出。
3. 每个点给：**记 or 不记、级别、前缀（`类名: `）、上下文字段**。
4. 明确列出“**不该记**”的高频 / 渲染 / 采样点。

输出格式：一张 `分类 | 位置 | 级别 | 前缀 | 关键字段` 表 + 一份“不记清单”。

## 9. 轻量检查命令
本项目默认不主动构建。日志审查优先用源码检查和 `rg`：

```powershell
rg -n "qDebug|qWarning|qCritical|std::cout|std::cerr" src include
rg -n "LOG_(INFO|WARN|ERROR|DEBUG|TRACE)\(\"[^\"]*%[a-zA-Z]" src include
rg -n "LOG_(INFO|DEBUG|TRACE)\(.*\+" src include
git diff --check
```

被 `rst-review` 调用时，只有在 diff 涉及 `LOG_` / `DB_LOG_` / `qDebug` / `std::cout` / `std::cerr`，或改动触及设备通信、状态机、病例/规划/导航保存、数据库访问、安全门禁时，才输出日志专项发现。若本次改动与日志风险无关，应明确“日志专项无发现”，避免制造噪声。

## 易错点速查（识别这些“形状”）
| 反模式 | 应改成 |
|---|---|
| 普通类里临时新造 `LOG_INFO("[影像分割] ...")` | 优先 `LOG_INFO("SegmentMain: ...")`；已有稳定领域前缀例外 |
| `LOG_ERROR("Exception in loadX: {}", e.what())` | `LOG_ERROR("加载X异常: {}, ID: {}", e.what(), id)`（去掉函数名，SUFFIX 已带） |
| `LOG_INFO("保存成功")` | `LOG_INFO("类名: 保存成功，caseId={}，path={}", ...)`（INFO 必须自带上下文） |
| 逐帧 / 渲染循环里 `LOG_INFO(...)` | 删除，或降为 `LOG_TRACE` + 开关 / 限频 |
| OTS missing、心跳超时记 `LOG_INFO` | `LOG_ERROR` + 可见告警（见 [[surgical-safety-review]]） |
| `qDebug() << x;` / `std::cout << x;` | `LOG_DEBUG("...: {}", x)` / `LOG_INFO(...)` |
| `LOG_WARN(someQString)` | 第一参数改字面量/std::string：`LOG_WARN("...: {}", someQString.toStdString())` |
| `LOG_INFO("进度 {50%}")` | 转义 `LOG_INFO("进度 {{50%}}")` 或带参数 `LOG_INFO("进度 {}%", 50)` |
| 日志里打患者姓名、身份证号、完整患者 ID 或敏感路径 | 去掉 PHI、脱敏，或只记内部 caseId（见 [[database-integrity-review]]） |

—

权威源：Obsidian vault《医疗器械软件关键业务模块日志设计规范》（canonical）。本 skill 与其保持一致；如规范更新且与本项目 KALog 约定冲突，以本项目实际宏行为（SUFFIX / 级别 / 前缀风格）为准。
