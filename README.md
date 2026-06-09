# RSTSystem Agent Skills

面向骨科手术机器人研发场景的一组 Codex / Claude Code 审查与诊断 skills。它们来自 RSTSystem 项目的真实 C++/Qt/VTK/MITK 工作流，重点覆盖手术导航、规划、设备通信、医学影像渲染、数据库、跨平台一致性以及失效安全。

> 注意：`rst-review` 现为工具中立的聚合审查入口，Claude Code 与 Codex 都可以使用。Claude Code 运行时使用 Agent 工具调度只读子代理；Codex 运行时使用 `multi_agent_v1.spawn_agent`，没有子代理能力时会降级为主会话顺序审查。其他专项 skill 遵循 Agent Skills 的 `SKILL.md` 基本格式，复制到 `.claude\skills` 或 `.codex\skills` 后即可按名称触发。

## 使用背景：骨科机器人软件研发

骨科机器人软件不是普通业务系统。它同时处理患者影像、术前/术中规划、配准矩阵、导航工具位姿、机械臂状态、植入物数据和多窗口医学影像渲染。很多风险不会表现为编译错误，而是表现为：

- 坐标系方向错、矩阵行列序错，导致导航位置偏移。
- 跟踪丢失、设备断连、数据库失败后系统继续使用旧数据或空数据。
- Qt 信号槽跨线程直连，偶发 UI 崩溃或状态错乱。
- VTK/MITK 过度刷新，大体数据交互掉帧。
- MySQL/SQLite 查询失败静默返回空，病例或植入物配置被写坏。
- Windows/Linux 编译器、编码、路径、ABI 差异导致只在某个平台失败。

这 8 个 skill 的目标是把这些经验固化为可复用的审查清单和 Agent 工作流，让每次改动都能按领域路由到合适的专项审查。

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
$qt-review 帮我审查这个 Qt widget 的信号槽和生命周期
/qt-review 帮我审查这个 Qt widget 的信号槽和生命周期
$geometry-transform-review 看一下这段配准矩阵链路是否方向写反
/geometry-transform-review 看一下这段配准矩阵链路是否方向写反
```

## 推荐使用方式

### 1. 日常改动：先跑聚合审查

在 RSTSystem 项目里完成一组未提交改动后，优先使用：

```text
$rst-review 当前修改未提交的文件
/rst-review 当前修改未提交的文件
```

`rst-review` 会先读取当前 diff 范围，再根据文件路径和 diff 内容路由到相关专项。例如：

- `.cpp/.h/.ui` 中出现 Qt 信号槽、`QTimer`、`QThread`：路由到 `qt-review`。
- 导航、规划、机械臂、阶段门禁相关改动：路由到 `surgical-safety-review`。
- 坐标矩阵、配准、Eigen/vtkMatrix、LPS/RAS：路由到 `geometry-transform-review`。
- SQL、数据库、病例/植入物表：路由到 `database-integrity-review`。
- VTK/MITK/ITK 渲染性能路径：路由到 `render-perf-diagnose`。
- 线程、锁、worker、串口/TCP、心跳：路由到 `cpp-concurrency-review`。
- 编码、路径、字节序、CMake、Windows/Linux 差异：路由到 `cross-platform-guard`。

聚合审查阶段只读，不修改文件。发现问题后应先让用户确认要修哪些项，再进入修复。

### 2. 定向问题：直接使用专项 skill

如果你已经知道问题领域，可以直接调用专项，减少上下文和时间消耗。例如：

```text
$database-integrity-review 审一下病例保存和假体数据库查询
$render-perf-diagnose 诊断这个 MITK 视图拖动时掉帧的原因
$cpp-concurrency-review 看这个控制板 worker 线程 stop 逻辑是否安全
```

## 每个 skill 的用途与用法

### `rst-review`：RSTSystem 聚合审查入口

适用场景：

- 想对当前 RSTSystem diff 做一次综合审查。
- 不确定改动涉及哪些风险领域，需要自动路由专项。
- 希望当前 Agent 运行时在可用时用只读子代理并行审查 Qt、并发、几何、安全、数据库、渲染、跨平台问题。

典型用法：

```text
$rst-review 当前修改未提交的文件
/rst-review 当前修改未提交的文件
$rst-review 审查 src/Plugins/PluginSurgicalNavigatView 的改动
/rst-review 审查 src/Plugins/PluginSurgicalNavigatView 的改动
$rst-review 对 main...HEAD 做综合审查
/rst-review 对 main...HEAD 做综合审查
```

输出重点：

- 审查范围。
- 激活/未激活的专项及原因。
- 阻断级、严重、建议问题。
- 每个问题的文件位置、机理、修法。
- 待确认的修复项和验证建议。

重要限制：

- 该 skill 同时面向 Claude Code 与 Codex；子代理接口按运行时自适应，缺少子代理能力时会降级为主会话顺序审查。
- Claude Code 分支使用 Agent 工具和 `subagent_type: general-purpose`；Codex 分支使用 `multi_agent_v1.spawn_agent` 和 `agent_type: "explorer"`。
- 它是审查入口，不直接替代各专项 skill。
- 审查阶段只读，不应自动修改代码或运行 CMake 构建。

### `qt-review`：Qt 运行时与 UI 审查

适用场景：

- Qt Widgets、手写 `setupUi()`、`.ui` 文件、model/view 逻辑。
- `connect`、`SIGNAL/SLOT`、`Qt::DirectConnection`、`QTimer`、`QThread`、`moveToThread`。
- QObject parent/生命周期、`deleteLater`、跨线程 UI 访问。
- RSTSystem 自研 `connectNotify/emitNotify` 通知总线。

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
2. 使用 `$rst-review 当前修改未提交的文件` 或 `/rst-review 当前修改未提交的文件` 做聚合审查。
3. 对高风险结论，再直接调用对应专项复核。
4. 用户确认修复项后再改代码。
5. 修复后做轻量验证：`git diff --check`、目标 `rg`、源码检查。
6. 构建/运行/CMake 验证只在用户明确要求时执行，避免在大型医疗软件仓库里无意触发长时间构建。

## 适用边界

适合：

- RSTSystem 或类似骨科机器人、导航机器人项目。
- C++/Qt/VTK/MITK/ITK/Eigen/CMake 技术栈。
- 医学影像、导航几何、机器人设备通信、数据库持久化相关审查。

不适合直接替代：

- 正式法规验证、IEC 62304 风险管理文档。
- 医学/临床参数最终确认。
- 真实硬件联调、机械臂安全认证。
- 构建系统和 CI 的实际双平台验证。

这些 skill 是工程审查辅助，不是安全认证结论。

## 维护建议

- 当 RSTSystem 新增模块、数据库表、坐标系或设备协议时，同步更新对应专项 skill。
- 对真实线上 bug 进行复盘后，把“形状”和“修法”补进 skill 的本项目易错点。
- `rst-review` 的路由表要保持轻量，只负责分发；专项细则放在各自 skill 内。
- 对 Claude Code / Codex 以外的兼容 Agent 使用时，可以复用专项清单和聚合流程；如果该运行时没有等价子代理接口，则按主会话顺序审查执行。
