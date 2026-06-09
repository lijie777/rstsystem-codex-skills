---
name: surgical-safety-review
description: Use when reviewing surgical navigation or robot fail-safe behavior, stage gates, tracking loss, device disconnects, heartbeat timeouts, arm motion limits, invalid data blocking, or unsafe defaults.
---

# 手术失效安全审查 (surgical-safety-review)

这是一台**会移动机械臂、会引导医生在病人身上打钉**的导航机器人。普通代码审查问"功能对不对"；本 skill 问的是更狠的一层：**当某个环节失效时，系统是停下来报警(fail-safe)，还是带着错误数据继续往下走(危险)？** 失效安全 bug 在功能测试里往往看不出来——一切正常时它静默无害，只有设备真断了、跟踪真丢了、数据真空了的那一刻才变成临床事故。审查时要主动设想"如果这一步失败会怎样"，逐项对照下面的规则。

本项目的安全设计**整体是 fail-closed 的好底子**（连接/就绪标志默认 false、阶段门禁、删除二次确认），近期 git 提交（`03aeaee`/`071dca5`/`8877bbd`）一直在补门禁。审查重点是：**老代码里漏掉门禁的地方，和设备物理安全这块明显偏薄的纵深防御。**

## 审查前先确认四件事
1. **这段代码在哪个手术阶段、属于哪个 `OperationState`**？进入它之前必须满足什么前置条件（有 CT、选了节段、配准完成、规划已保存）？
2. **它依赖的设备/数据如果失效会怎样**？跟踪丢帧、控制板失联、配准矩阵无效、植入物为空——每一种失效的代码分支走到哪？
3. **失效后是 fail-safe 还是 fail-open**？默认值、空结果、异常分支落到的是"安全态(拒绝/停机/隐藏)"还是"危险态(放行/继续/用旧数据)"？
4. **这是物理动作还是数据动作**？机械臂使能/运动/OTS 开关属物理危险动作，标准比数据操作更高。

## 0. 本项目专属：OperationState 阶段门禁 — 最高优先级

阶段流转定义在 `src/Common/GlobalData/commonData.h` 的 `enum class OperationState`（案例管理→影像分割→标注→规划→配准→导航→术后验证）。**进入每个阶段前必须校验前置条件**，否则会出现"没 CT 就进分割""没选节段就进规划""规划没保存就进导航""植入物无效却当已规划"这类近期真实 bug。逐条对照：

- [ ] **门禁判定函数遇"服务/配置不可用"必须 `return false`（默认拒绝 = fail-closed）**。正例 `MainView::currentSurgicalConfigHasSelectedLevels()`：`RequestServer` 拿不到服务或配置为空时一律按"未选择"返回 false。反模式信号：`RequestServer<IXxx>()` 后没判空就乐观放行，或 `catch`/异常分支里 `return true`。
- [ ] **门禁要真正挡住切页，并给出中文原因**。正例 `MainView::mainLayoutModuleEntryDeniedReason()` 返回拒绝原因字符串 → `CoreUi::canEnterModuleOrWarn()` 非空则 `KMessageBox::warning` 弹窗 + `return false`、保持当前页。审查：新增一个手术模块入口时，是否接进了这套统一门禁，还是绕过去直接切页。
- [ ] **危险操作后必须主动回退阶段/保存状态**。正例 `SurgicalPlanWidget::downgradeCurrentCasePlanStatusIfPlaned()`：植入物位姿一改，就把案例从"已规划(Planed)"降级回"未规划"，防止旧规划被当成已保存直接进导航。反模式信号：改了 `transform`/`pose`/植入物后，案例状态仍是"已保存/已规划"。
- [ ] **"可保存"的判定要校验数据实质有效，而非仅非空**。正例 `currentScrewHasSavableImplant()`：校验 STL 文件存在 + 变换矩阵齐次行有效，无有效植入物则保存按钮 `setEnabled(false)`。反模式信号：只要指针非空/字符串非空就允许保存。
- [ ] **每个手术模块入口都要有对应门禁**：CT 路径(`currentCaseHasCtPath`)、节段选择(`currentSurgicalConfigHasSelectedLevels`)、规划保存(`currentSurgicalPlanHasAllSavedSegments`)。改 `OperationState` 流转或加新阶段时，全局搜这几个判定函数，确认新路径也被覆盖。

## Checklist

### 1. 跟踪丢失 / 设备失联的 fail-safe
- [ ] **跟踪 `missing` 标记必须阻断计算，禁止用缺失数据继续**。正例：OTS/跟踪服务的 `getNavigatorToolData*` 在 `m_recvData[i].m_isMissing` 为真时 `return false`。反模式信号：拿到 toolInfo 不查 `m_isMissing` 就直接用 `m_tx/m_ty/m_qx...`。
- [ ] **实时位姿无效时隐藏/跳过该工具，绝不回退到上一帧或测试动画**。正例 `SurgicalNavigatWidget.Tracking.cpp`：`!ndiValid` → `setRenderedVisible(false); continue;`，且矩阵先过 `isValidNavMatrix`（齐次行 + 旋转行列式下限）。反模式信号：`if(!valid){ mat = lastMat; }` 或继续渲染旧位姿。
- [ ] **缓存帧要有新鲜度/超时(staleness)检测**。⚠️ 常见薄弱点：OTS/跟踪服务的 `getNavigatorToolData` 只读上一帧缓存，若 NDI/跟踪器 **完全停推帧**(线缆假死)，`m_isMissing` 不会自动翻转，可能持续返回最后一帧。审查：跟踪/设备数据是否带时间戳，超过 N ms 无新帧是否熔断。
- [ ] **设备失联超时阈值是否合理、超时后是否有强制动作**。⚠️ 反模式：术中机械臂控制板心跳监控**失联 15 秒**才反应，且仅断 socket + 发通知、**无强制急停/停机**。审查：心跳超时阈值是否过长、超时后是否只断连不停机。
- [ ] **失效事件的日志级别**：本项目系统性偏低——OTS missing、心跳超时都记 `LOG_INFO`。安全相关失效应 `LOG_ERROR` 并触发可见告警，而非淹没在 INFO 里。

### 2. 运动 / 边界限位（机械臂，纵深防御明显偏薄）
- [ ] **机械臂运动指令在上位机是否有范围/限位/奇异性校验**。⚠️ 反模式：机械臂 `ArmMoveToAppointPostureCmd` 这类运动接口把 x/y/z/rx/ry/rz/speed/accel 直接 set 进 proto 下发，**零校验**，只有一行 qDebug。审查：`set_x/set_y/...` 下发前是否有 `clamp`/工作空间/速度上限检查。
- [ ] **被注释掉的安全功能**。⚠️ 反模式：机械臂边界控制/限位控制函数整段被注释。grep 注释块里的 `Boundary/Safe/Limit/限位/边界`，确认不是把安全功能临时关掉忘了开。
- [ ] **限位完全外包给下位机 = 单点防御**。即便下位机有限位，上位机也应有一层校验，避免越界/奇异姿态指令原样下发。

### 3. 空数据 / 无效输入拦截
- [ ] **取数前判空 + size 检查**。正例：OTS/跟踪数据 `m_recvData.size()==0` → `return false`；标定点数不符不写入并 `qWarning`。反模式信号：`at(0)`/`[0]`/`front()` 前无 size 检查、未配准就拿配准矩阵算。
- [ ] **关键前置数据为空要拦在阶段入口**，而不是放进去后崩在深处（呼应第 0 节门禁）。
- [ ] 区分"固定尺寸 Eigen/数组分量访问(`p[0]`)"与"动态容器越界"——前者无风险，别误报；重点查动态 `std::vector`/`QList` 的越界。

### 4. 错误不可静默吞掉
- [ ] **`catch` 不能只 `cerr`/`qDebug` 就算完**。⚠️ 反模式 `Crypto.cpp`：解密 `catch` 里只 `std::cerr`，函数 `void` 无返回值、调用方无感知，可能用未解密/缺失文件继续。审查：底层工具库的失败是否被吞、调用方能否知道失败。
- [ ] **失败后必须改变控制流**，不能记一行日志继续往下。正例：采点/影像导入模块 DICOM 导入 `catch(...)` → 弹窗 + `return`；手术规划界面相机初始化失败后降级到默认相机。
- [ ] 返回 `bool`/错误码的函数，调用方是否检查了返回值（呼应数据库层 `exec()` 被忽略的同类问题，见 [[database-integrity-review]]）。

### 5. 危险物理操作的二次确认
- [ ] **物理危险动作要有二次确认**。⚠️ 本项目不对称：数据删除有确认（`SpineCaseManager` 删除案例 `KMessageBox::question("此操作不可恢复")`、`SegmentMain` 清除分割结果确认），但**机械臂"运动到指定位姿"/自由拖动使能/OTS 开关均无二次确认**，直接下发命令。审查：使能/运动/上电这类会让机器物理动作的入口是否有确认或安全联锁。

### 6. 默认安全态
- [ ] **状态/使能标志默认应为"安全(false/未连接/禁用)"**。正例：控制板模块的 `m_isSocketConnected=false`、`m_isControlBoardReady=false`；全局设备连接状态 `*_Connect=false`；保存按钮默认 `setEnabled(false)`。反模式信号：连接/就绪/使能/允许 默认 `true`，要靠后续代码去关——一旦那段没执行就处于危险态。

## 本项目易错点速查（识别这些"形状"）

| 形状 | 在哪 | 风险 |
|---|---|---|
| 门禁判定 `RequestServer` 后不判空就放行 / 异常分支 `return true` | 各阶段入口 vs 正例 `MainView::currentSurgicalConfigHasSelectedLevels` | fail-open，无前置条件也能进下一阶段 |
| 改了 pose/transform/植入物，案例仍标"已规划/已保存" | 规划→导航衔接 vs 正例 `downgradeCurrentCasePlanStatusIfPlaned` | 旧规划被当成已保存进导航 |
| 取 toolInfo 不查 `m_isMissing` 就用坐标 | OTS/跟踪采集服务 | 用缺失/无效跟踪数据继续算位姿 |
| `if(!valid) mat=lastMat;` 或继续渲染旧位姿 | 导航实时刷新 vs 正例 `Tracking.cpp` 隐藏工具 | 跟踪丢了却显示旧位置，误导术者 |
| 跟踪缓存帧无时间戳/超时熔断 | OTS/跟踪服务 `getNavigatorToolData` | 设备停推帧(假死)时持续返回最后一帧 |
| 心跳超时阈值过长 + 超时仅断连不停机 | 控制板心跳监控(如 15s 才处理) | 术中失联很久才反应，无急停 |
| 机械臂位姿下发零范围/限位/奇异校验 | 机械臂运动指令下发函数 | 越界/奇异指令原样下发给机器 |
| 安全功能整段被注释 | 机械臂边界/限位控制函数 | 边界控制被关掉 |
| `catch` 只 `cerr`/`qDebug`，函数 void 无返回 | `Crypto.cpp` | 失败静默，调用方拿缺失数据继续 |
| 安全失效记 `LOG_INFO` 而非 `LOG_ERROR`/告警 | OTS missing、心跳超时 | 关键失效淹没在 INFO 里无人看见 |
| 物理动作(使能/运动/上电)无二次确认 | 机械臂使能/运动、OTS/跟踪器开关 | 误触即物理动作 |
| 使能/允许/连接标志默认 `true` | 全局搜默认值 | 初始即危险态 |

> 安全范例（"应该怎么写"的对照）：主界面的统一门禁(fail-closed) + `canEnterModuleOrWarn` 这类入口拦截、危险操作回退规划状态、OTS/跟踪服务的 `m_isMissing` 阻断、跟踪界面隐藏无效工具、删除病例的 `KMessageBox::question` 二次确认。

## 输出格式（审查报告）

```
# 手术失效安全审查报告

## 失效场景梳理（我的理解）
- 这段代码所属手术阶段 / OperationState 与前置条件
- 它依赖的设备/数据，及各自失效时走到的代码分支
- 每个失效分支落在 fail-safe 还是 fail-open

## 🔴 阻断级（带错误数据继续 / 物理动作无校验 / 失效不停机 / 门禁可绕过）
1. [文件:函数] 问题：...
   失效时会发生什么（临床后果）：...
   修法（让它 fail-safe）：...

## 🟠 严重（失效不报警 / 日志级别过低 / 缓存帧无超时 / 危险操作无确认）
...

## 🟡 建议（补纵深防御 / 统一安全常量 / 默认安全态）
...

## 验证建议
- 对每个外部依赖(设备/服务/数据)问一句"它失败时这里怎么走"，确认落在安全态
- 安全相关失效是否 LOG_ERROR + 可见告警（本项目普遍偏 INFO）
- 物理动作入口是否有校验/确认/联锁
```

## 重要提醒
- **安全 bug 的特征是"平时无害"**——功能测试全过不代表 fail-safe，要专门设想失效路径。
- 跟踪/位姿无效的判定常涉及矩阵有效性与坐标变换，配合 [[geometry-transform-review]]（矩阵/四元数是否有效、单位是否一致）一起审。
- 设备线程的失效处理常涉及跨线程与竞态（标志位无同步、deleteLater 竞态），配合 [[qt-review]] 与 [[cpp-concurrency-review]]。
- 不确定某个阈值(超时/限位/精度)的临床合理值时，明确提示这是需要医学/法规(IEC 62304/风险管理)确认的参数，不要臆断数值。
