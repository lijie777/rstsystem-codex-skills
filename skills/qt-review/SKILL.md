---
name: qt-review
description: Use when reviewing Qt/C++ code involving signals/slots, QObject ownership, thread affinity, moveToThread, QThread, deleteLater, QTimer, model/view, dialogs, or Qt runtime warnings.
---

# Qt 专项审查 (qt-review)

Qt 在 C++ 之上叠了一套**自己的运行时规则**——对象树所有权、线程亲和性、事件循环驱动的信号槽。很多 bug 在纯 C++ 视角下完全看不出来，却会在 Qt 运行时变成偶发崩溃、界面卡死、信号"明明连了却不触发"、或跨线程访问导致的诡异数据错乱。审查 Qt 代码必须戴上"Qt 运行时"这副眼镜，逐项对照下面的规则。

## 审查前先确认四件事
1. **每个关键 QObject 活在哪个线程**（thread affinity）？GUI 对象(QWidget/QPixmap 等)必须在主线程。
2. **对象树的 parent 关系**：谁拥有谁、谁负责析构、对象建在栈上还是堆上。
3. **哪些 connect 是跨线程的**：跨线程连接的语义和同线程完全不同。
4. **`emitNotify` 是在哪个线程被调用的**（本项目专属）？设备数据多在 TcpServer/串口/QThread 的 worker 线程到达，若该处再 `emitNotify`，而下游用 `Qt::DirectConnection` 接收，则下游槽**直接在 worker 线程执行**——必须把这条调用链先画出来。见下方「服务通知总线」一节。

## 0. 本项目专属：服务通知总线 (connectNotify / emitNotify) — 最高优先级

RSTSystem 的插件之间**不直接 connect**，而是经 `ServerBase` 的自研事件总线解耦：`emitNotify(NotifyStruct{code, paramMap})` 发，`connectNotify(code, receiver, SLOT(...), type)` 收（定义见 `src/Kernel/CoreLib/ServerBase.cpp` + `ServerNotify`）。底层是一个真实 Qt 信号 `notify(unsigned int, const NotifyStruct&)`。审查这套总线时**逐条对照**：

- [ ] **槽签名必须逐字匹配 `slot(unsigned int, const NotifyStruct&)`**。`connectNotify` 内部用**旧式 `SIGNAL()/SLOT()` 字符串宏**连接——参数类型/个数/`const &` 写错，编译器**不报错**，运行时静默失败连不上。接收类必须有 `Q_OBJECT`，槽要声明在 `slots:` 下。
- [ ] **连接类型默认 `Qt::AutoConnection`**：worker 线程 `emitNotify` → GUI 线程 receiver 时自动走 Queued（安全，投递到 GUI 事件循环）。**但本项目大量显式写 `Qt::DirectConnection`**（机械臂 `PluginRoboticArm.cpp`、动力工具 `PluginPowerTool.cpp`、脚踏 `PluginKickstand.cpp`、`UpgradePowerToolWgt.cpp` 等）。DirectConnection 下槽**永远在发射线程执行**——若 `emitNotify` 源头在控制板 TCP/串口 worker 线程，则这些插件的 `getNotifyMessage` 槽就在 worker 线程里跑，槽内任何访问 UI / 本插件 QObject 状态都是**跨线程违规**。每见一个 DirectConnection 的 `connectNotify`，必须回溯 `emitNotify` 的调用线程。
- [ ] **paramMap 里的自定义类型要单独注册**。`NotifyStruct` 本身已在 `ServerBase` 构造里 `qRegisterMetaType<NotifyStruct>("NotifyStruct")` 集中注册；但你塞进 `QMap<QString,QVariant> paramMap` 的自定义结构/枚举（`QVariant::fromValue<T>(...)`）需要自己 `Q_DECLARE_METATYPE(T)`（跨线程 Queued 再加 `qRegisterMetaType<T>()`），否则取出时 QVariant 转换静默失败、拿到空值。
- [ ] **receiver 生命周期 vs 总线生命周期**。`ServerNotify` 活到 server 销毁为止；receiver（常是随手术阶段 `OperationState` 创建/销毁的插件 widget）若先析构——旧式字符串槽在 receiver 析构时 Qt 会自动断开（OK），但**正在 worker 线程经 DirectConnection 投递的那一次**仍可能打到半析构对象。阶段切换/插件 `unInit()` 里应显式 `disconnectNotify`，别只依赖自动断开。
- [ ] **总线是单向广播**：同一 `code` 可被多个 receiver 收。新增 `emitNotify` 时确认没有非预期的二次接收方；改 `code` 的语义/payload 时全局搜 `connectNotify(<code>` 找齐所有下游。
- [ ] **别复用成员 `NotifyStruct` 反复 emit**：典型反例 `PluginControlBoard::sendNotify`（`PluginControlBoard.cpp` 复用成员 `notifyStruct`，就地改 `code/paramMap` 再 emit）。该函数在 socket worker 线程被高频调用，同一对象被并发改写 + 下游 `Qt::DirectConnection` 槽就地读 → 跨线程数据竞争 / 读到半写状态。每次 `emitNotify` 应构造**局部** `NotifyStruct`，不要复用可变成员。

## Checklist

### 1. 信号槽 connect
- [ ] **连接类型对吗**？默认 `AutoConnection`：同线程=Direct(同步直接调用)，跨线程=Queued(经事件循环异步投递)。明确指定 `Qt::DirectConnection` 跨线程使用＝**在发射线程直接执行槽**，等于跨线程访问对象→数据竞争。
- [ ] 用了 `Qt::BlockingQueuedConnection`？**发射线程与接收线程相同时会死锁**。
- [ ] 用**新式函数指针**语法 `connect(a,&A::sig,b,&B::slot)` 吗？旧式 `SIGNAL()/SLOT()` 宏是字符串，拼错只在运行时静默失败、无编译检查。
- [ ] 跨线程信号的**参数类型是否已注册**？自定义类型经 Queued 连接传递需 `qRegisterMetaType<T>()`，否则运行时报 "Cannot queue arguments of type 'T'" 且连接失效。
- [ ] **重复 connect**：同一信号槽连了多次会被调用多次（除非用 `Qt::UniqueConnection`）。
- [ ] connect 到 **lambda** 时，捕获的对象/`this` 生命周期是否覆盖信号存续？对象先析构后信号触发→悬空。用 `connect(sender, &Sender::sig, context, [..]{...})` 的 **context 对象重载**，让连接随 context 析构自动断开。
- [ ] 槽里会不会在错误的线程访问 GUI/对象？
- [ ] 是否检查了 `connect` 返回值（QMetaObject::Connection）？失败的连接静默不工作。
- [ ] **本项目 `Qt::DirectConnection` 极其常见**（TcpServer/串口/心跳线程/通知总线），且**新式函数指针与旧式 `SIGNAL/SLOT` 字符串约各占一半**。把每个 DirectConnection 当成「跨线程嫌疑」逐个回溯发射线程；旧式字符串槽要逐字核对签名（无编译检查）。详见上方「0. 服务通知总线」。

### 2. 对象树与所有权 (parent / 生命周期)
- [ ] **QObject 析构会 delete 所有 children**。栈上对象若设了 parent，会被 parent 二次 delete → double free / 崩溃。规则：**给了 parent 的对象应 new 在堆上**。
- [ ] 析构顺序：手动 `delete` 了一个有 parent 的子对象后，parent 析构时会再删一次吗？（QObject 会自动从 parent 移除，但裸指针别处还在用就悬空。）
- [ ] **`deleteLater()`**：在事件循环中安全删除当前正在处理其信号的对象时使用。但**对象所在线程必须有运行的事件循环**，否则永远不被删→泄漏。
- [ ] QObject **不可拷贝**（拷贝构造/赋值被禁）。按值传 QObject 派生类是设计错误。
- [ ] 用 `QPointer<T>`（对象销毁后自动置空）来持有可能先于自己析构的 QObject 弱引用。
- [ ] 智能指针与 parent 体系**混用**：`unique_ptr<QObject>` 同时又设了 parent → 双重释放。二选一。
- [ ] **本项目几乎不用 `QPointer`/智能指针守护 QObject**，控件一律 `new X(this)` 靠 parent 树回收、设备/socket/线程对象裸 `new`+手动 `deleteLater`。审查重点：①要 `moveToThread` 的 worker/线程/socket 对象**绝不能**挂到 GUI 控件 parent 下（否则不能移动、且会在 GUI 线程被析构）；②通知总线/容器里存的裸 `receiver*` 在手术阶段(`OperationState`)切换销毁后是否仍被 emit 打到——必要时用 `QPointer` 或在 `unInit()` 里 `disconnectNotify`。

### 3. 线程亲和性 (thread affinity) 与 moveToThread
- [ ] `moveToThread()` 前对象**不能有 parent**（带 parent 不能 move，会警告并失败）。
- [ ] move 的是**整棵子树**：children 跟随父对象一起迁移；不能只 move 一个有 children 的对象的一部分。
- [ ] **绝不**在对象所属线程之外**直接调用其成员函数 / 访问其状态**——只能通过 Queued 信号槽或 `QMetaObject::invokeMethod(obj, ..., Qt::QueuedConnection)` 投递。
- [ ] **QThread 用法**：推荐 **worker 对象 + `moveToThread(&thread)`** 模式，而不是子类化 QThread 把逻辑写在 `run()` 里。若子类化 QThread，注意：QThread 对象本身亲和于创建它的线程，只有 `run()` 内代码在新线程；在 `run()` 外（如构造函数）创建的成员对象亲和于**旧**线程。
- [ ] 在 `run()` 或新线程里 `new` 的 QObject，其亲和性属于该线程——别把它当成主线程对象用。
- [ ] **GUI 对象只能在主(GUI)线程**创建和操作。子线程碰 QWidget/QPixmap 等→未定义行为/崩溃。
- [ ] 运行时警告 "QObject: Cannot create children for a parent in a different thread" 就是亲和性违规的信号。

**本项目专属（设备插件普遍如此，重点查）**：
- [ ] **本项目几乎全是「子类化 QThread + 重写 `run()`」**（`mythread`、`FootSwitchMonitor`、`TrackingManager`、`OTSThread` 等），`run()` 内是手写 `while(flag){...}` 轮询循环。两个固定坑：
  - **循环标志位是裸 `bool`**（`exitflag`/`m_monitorFlag`/`m_tracking`），`run()`(worker 线程)读、`stop()`(GUI 线程)写，**无 `std::atomic`/无同步**——worker 可能永远看不到 stop。应为 `std::atomic<bool>`/`QAtomicInt`。深入并发分析转 [[cpp-concurrency-review]]。
  - **`stop()` 里调 `exit()`/`quit()` 对手写 `while` 循环无效**：`exit()/quit()` 只能停 `exec()` 起的事件循环；`run()` 没有事件循环时它们什么都不做，线程只能靠标志位退出。见 `mythread::stop()`（`PluginBaseControlBoard/mythread.cpp`）。
- [ ] **`QTimer::start()` 必须在定时器所属线程调用，且 `moveToThread` 要先于 `start`**。典型反例 `PluginBaseInteractiveBoard::init`（`PluginInteractiveBoard.cpp`）：`m_heartTimer->start(2000)` 在主线程调用后**才** `m_heartTimer->moveToThread(m_thread)`——Qt 会告警 "Timers cannot be started from another thread"，定时器行为不确定；且 `timeout` 用 `Qt::DirectConnection` 连主线程对象的 `heartCheck` 槽，定时器在 worker 线程触发时槽就地在 worker 线程跑、跨线程读写心跳成员。正确顺序：先 `moveToThread` → 再在该线程内（如 worker 的入口槽里）`start`，或干脆把定时器留在 GUI 线程用 `AutoConnection`。
- [ ] **busy-loop 无 `sleep` 满速占核**：`run()` 写成 `while(true){ getXxx(); }` 没有任何 `msleep`/条件等待（典型 `PluginOTS::OTSThread::run()`），连接后满速轮询设备，单核打满 + 与消费侧抢锁。轮询循环必须有节流（`msleep`/`QWaitCondition::wait` 超时）。
- [ ] **`moveToThread` 误用**：典型反例 `SerialPortManager.cpp` —— `m_thread(new QThread(this))` + `m_worker(new SerialPortWorker(this))` 后 `this->moveToThread(m_thread)`。三重问题：① 若 `SerialPortManager` 带 parent，`moveToThread` 会失败（"Cannot move objects with a parent"）；② 把要进入的 QThread 自身设成被移动对象的 child，是自指错误（QThread 控制对象应留在管理线程）；③ worker 是 `this` 的 child，会随 `this` 一起被拖进 worker 线程，原本「worker 在子线程、manager 在 GUI」的意图丢失，manager↔worker 的 connect 静默退化成同线程 Direct。审查 `moveToThread` 时务必核对 parent 关系与「谁该留在哪个线程」。

### 4. 事件循环依赖
- [ ] **主线程事件循环被阻塞**？长耗时操作/忙等/同步 IO 放在主线程会卡死 UI。应移到 worker 线程或拆成异步。
- [ ] 子线程里若**没有调用 `exec()` 起事件循环**，则该线程对象的：Queued 连接、`QTimer`、`deleteLater`、`QEventLoop` **都不工作**。确认 worker 线程是否需要事件循环。
- [ ] `QApplication::processEvents()` 的**重入风险**：会就地分发事件，可能重入当前函数/槽，导致意外递归或状态被打断。慎用。
- [ ] `QTimer` 亲和于其所属线程，回调在该线程事件循环触发——别假设它在主线程。

### 5. 其它高频坑
- [ ] 信号在**对象正在析构期间**仍可能被触发/排队？析构前应 `disconnect` 或用 context 连接。
- [ ] 全局/单例 QObject 的**析构顺序**（尤其依赖 QApplication 生命周期的）。
- [ ] `emit` 一个信号是**同步**调用所有 Direct 槽——会就地深入槽逻辑，注意重入与持锁。
- [ ] 容器里存 `QObject*` 裸指针，对象被 parent 删除后容器里悬空。
- [ ] **`QWaitCondition::wait()` 必须配裸 `QMutex` 手动 lock**，禁止与 `std::unique_lock`/`QMutexLocker` 混用。反例 `PluginOTS.cpp`：`std::unique_lock<QMutex> locker(m); m_waitCondition.wait(&m, 4000)`——`wait` 内部已对锁做 unlock/relock，`unique_lock` 析构又解锁一次 → 二次解锁 UB。要么全程裸 `QMutex` + 手动 lock/unlock，要么用 `QMutexLocker`（但仍要把裸锁地址传给 `wait`，且 `wait` 期间 locker 不能析构）。
- [ ] **`QSemaphore` 当互斥量用时 `acquire`/`release` 必须异常安全配对**。反例 `PluginBaseOTS.cpp`：`m_semaphore(1)` + `acquire(); ...; release();` 包临界区——中途 `return`/抛异常漏掉 `release`，下一帧 `acquire` 永久阻塞采集线程。要么用 `QMutexLocker`（RAII），要么确保所有出口都 `release`。
- [ ] **设备 IO 拆包（`readyRead` + `QByteArray`/`QDataStream`）**：`readyRead` 每次 `readAll()` 拿到的是任意切分的 TCP/串口流，帧头帧尾判定必须基于**跨次累积缓冲**；信任协议里的"长度字段"去 `mid()` 前要校验上界防越界；手写小端移位（`buf[1] << 8`）在 `char`/有符号类型上做会**符号扩展**，应先转 `uchar`/`uint8_t`。反例 `PluginControlBoard.cpp` 的 framing 与 `QByteToShort`、`SerialPortWorker.cpp` 循环内用循环外算出的 `bodyLength` 移除多包。字节序优先用 `QDataStream::setByteOrder` 而非手写移位。深入并发/协议正确性转 [[cpp-concurrency-review]] 与 [[cross-platform-guard]]（字节序/符号扩展）。

## 6. GUI 侧（案例管理 / 手术规划 / 手术配置）非线程类高频坑

这三个模块**纯手写 UI、无 `.ui` 文件**，主线程跑、几乎不碰设备线程，但有一批与上面线程主题正交的 Qt 框架陷阱。审查这类纯 GUI widget 时逐条过：

- [ ] **Model/View：proxy 与 source 的索引必须转换**。`QSortFilterProxyModel` 排序/过滤后行号与底层 `QStandardItemModel` 不同——拿 proxy 行号去访问 source（或反之）会取到**错行**。必须 `mapToSource`/`mapFromSource`（正例 `SpineCaseManager.cpp::restoreSelection`）。另：`setModel()` 之后旧的 `selectionModel()` 指针失效，`selectionChanged` 连接要在 `setModel` 之后建立。
- [ ] **`QDialog::exec()` 起嵌套事件循环 → 重入**。`exec()` 期间界面仍派发其它事件（定时器、通知总线、用户再次点击同一按钮），槽可被重入；`exec()` 返回前外层 `this` 可能已被销毁。案例管理 `onCreateClicked/onEditClicked/onImportClicked` 内均 `dialog.exec()` 且槽内还弹 `KMessageBox`——审查：① 按钮是否防重复点击（重入弹两个框）；② `exec()` 返回后是否假设 `this`/成员仍有效。
- [ ] **`blockSignals(true/false)` 裸调易漏配对 → 信号永久静默**。回填面板防 `setCurrentText`/`setValue` 触发 syncback 回环时（`SurgicalPlanWidget.cpp` 用 `for(w:widgets) w->blockSignals(true) ... false` 包一组 combo），中途 `return`/抛异常会漏掉 `blockSignals(false)`。**优先用栈作用域 `QSignalBlocker`（RAII，离开作用域自动恢复）**——本仓两种写法并存（`QSignalBlocker` 用在配置模块 combo、裸 `blockSignals` 用在规划面板），可作为统一化建议。
- [ ] **`QTimer::singleShot(0, this, lambda)` 把逻辑推到下一轮事件循环 → 捕获的裸下标/指针可能已失效**。带 `this` 接收者时 `this` 销毁会自动断开（安全），但 lambda **额外捕获的裸 `idx`/指针不受保护**，回调执行时数据 vector 可能已增删。正例 `SurgicalPlanWidget.cpp`：回调内 `if (idx >= 0 && idx < screws.size())` 先重校验下标——审查时确认每个延迟回调都做了这种再校验。
- [ ] **动态增删控件：`layout->takeAt()` 只取出 `QLayoutItem`，不删 widget**。节段切换重建按钮组时必须 `delete item->widget(); delete item;`（正例 `PluginSurgicalConfigView.cpp::refreshLevelButtons`）——漏删 widget 即泄漏；反之 widget 已被 parent 树接管又手动 delete 则 double-free。重建后立刻重连 `connect` 是对的，但要确认没有 pending 的 `singleShot`/排队信号仍指向**已 delete 的旧控件**。
- [ ] **区分"手写 `setupUi()`"与"uic 生成 `Ui::X*`"**：本仓 `setupUi()` 是**手写方法名**、无 `ui` 指针、控件直接 `new X(this)` 靠 parent 树回收——**不要**因此误报"缺 `delete ui`"。只有真正 uic 生成、持有 `Ui::X* ui` 成员的才需要配对释放（或用值成员）。
- [ ] **`QString::fromStdString`/`toStdString` 默认按 UTF-8 编解码**。DICOM 患者名、文件路径若对端是本地 ANSI/GBK 编码，中文会乱码；`toStdString()` 直接当 Windows 文件路径喂给非 Qt API（如 `readSTL(path.toStdString())`）在中文路径下可能失败。编码/路径细节转 [[cross-platform-guard]]。
- [ ] **item-based 容器所有权**（本仓三模块**未用**，但其它 widget 可能有，备查）：`QListWidget/QTableWidget` 的 `takeItem` 移交所有权给调用者（须手动 delete），`clear()`/析构则自动删；二者混用易 double-free/泄漏。`setItemWidget` 设的 widget 由 item 接管，重复 set 不删旧 widget。

## 输出格式（审查报告）

```
# Qt 审查报告

## 线程与对象模型（我的理解）
- 各关键 QObject 的线程亲和性
- 对象树 / parent 所有权
- 跨线程的 connect 列表及其连接类型

## 🔴 阻断级（崩溃 / 跨线程直连 / double free / 死锁连接）
1. [文件:行] 问题：...
   Qt 运行时为什么会出错：...
   修法：...

## 🟠 严重（信号静默失效 / 漏注册元类型 / deleteLater 不触发 / UI 卡死）
...

## 🟡 建议（用函数指针语法 / UniqueConnection / QPointer / worker+moveToThread 模式）
...

## 验证建议
- 留意运行时控制台的 QObject 警告（亲和性、无法排队的参数类型、连接失败）
- 用 Qt::DirectConnection 显式标注同线程意图，跨线程一律 Queued
- 检查每个 connect 的返回值与连接类型
- （本项目）对每个 `connectNotify(..., Qt::DirectConnection)`，回溯对应 `emitNotify` 的调用线程，确认槽不会在 worker 线程触碰 GUI/插件状态
```

## 本项目易错点速查（识别这些"形状"）

| 形状 | 在哪 | 风险 |
|---|---|---|
| `connectNotify(code, this, SLOT(...), Qt::DirectConnection)` | `PluginRoboticArm/PowerTool/Kickstand.cpp`、`UpgradePowerToolWgt.cpp` | 槽在 `emitNotify` 发射线程执行；若源头是设备 worker 线程→跨线程访问 |
| `connect(socket, SIGNAL(dataProcess(...)), this, SLOT(...), Qt::DirectConnection)` 后 `socket->moveToThread(t)` | `PluginBaseControlBoard/TcpServer.cpp` | socket 信号在 worker 线程发，DirectConnection 使槽也在 worker 线程跑 |
| `new QThread(this)` + `new Worker(this)` + `this->moveToThread(...)` | `PluginBaseInteractiveBoard/SerialPortManager.cpp` | parent 关系与线程归属错乱，见上「moveToThread 误用」 |
| 子类 QThread，`run(){ while(flag){msleep; emit ...} }`，`flag` 为裸 bool | `mythread`/`FootSwitchMonitor`/`TrackingManager`/`OTSThread` | 标志位无同步；`stop()` 里 `exit()/quit()` 对手写循环无效 |
| `QTimer` 后 `timer->moveToThread(t)`，timeout 用 `Qt::DirectConnection` 连 GUI 对象槽 | `PluginBaseInteractiveBoard/PluginInteractiveBoard.cpp` | 定时器回调在 worker 线程触发，槽却操作 GUI 侧状态 |
| `std::thread([=]{...}).detach()` 捕获 `this`/外部标志 | `PluginBaseControlBoard/MyTimer.h` | 分离线程生命周期失控；回调触碰 QObject 即跨线程 |
| `connect(sender, &Sig, [this,p]{...})` 无 context 接收者 | `PluginSegmentView/AISegmentServer.cpp` | sender 先于 lambda 析构→悬空；改用三参 context 重载 |
| `timer->start(...)` 后才 `timer->moveToThread(t)`，`timeout` 用 DirectConnection | `PluginBaseInteractiveBoard/PluginInteractiveBoard.cpp` | 在非所属线程 start 定时器（告警）；回调在 worker 线程跑却读主线程成员 |
| `this->moveToThread(m_thread)` 把管理对象自身搬走（worker 是 this 的 child） | `PluginBaseInteractiveBoard/SerialPortManager.cpp` | manager 公有方法的 emit 投递线程被改写；`stop()` 里 `emit closePort` 紧跟 `quit()` 不 wait，串口未关线程已退 |
| 复用成员 `NotifyStruct` 就地改 `code/paramMap` 再 `emitNotify` | `PluginBaseControlBoard/PluginControlBoard.cpp::sendNotify` | worker 线程高频复用同一对象 + 下游 Direct 槽就地读 → 数据竞争/读半写态 |
| `std::unique_lock<QMutex>` 配 `QWaitCondition::wait(&m,...)` | `PluginOTS/PluginOTS.cpp` | wait 内部已 relock，unique_lock 析构再解锁一次 → 二次解锁 UB |
| `readyRead`→`readAll()` 后信任长度字段 `mid()`；`buf[1]<<8` 在 char 上移位 | `PluginBaseControlBoard/PluginControlBoard.cpp`、`PluginBaseInteractiveBoard/SerialPortWorker.cpp` | 半包粘包未跨次累积、长度越界、符号扩展 |
| proxy 行号未 `mapToSource` 直接访问 source；`setModel` 后用旧 `selectionModel()` | `PluginSpineCaseManagerView/SpineCaseManager.cpp` | 取错行；旧 selectionModel 指针失效 |
| `layout->takeAt(0)` 后未 `delete item->widget()`（动态重建按钮组） | `PluginSurgicalConfigView/PluginSurgicalConfigView.cpp` | widget 泄漏；或反向 double-free |
| 裸 `blockSignals(true)...false` 包回填，中途可能 return | `PluginSurgicalPlanView/SurgicalPlanWidget.cpp` | 漏 `false` → 信号永久静默；应用 `QSignalBlocker` |
| `QDialog::exec()` 在按钮槽内，槽内再弹 `KMessageBox` | `PluginSpineCaseManagerView/SpineCaseManager.cpp` | 嵌套事件循环重入；exec 返回前 `this` 可能已销毁 |

> 安全范例（可作为「应该怎么写」的对照）：`PluginBaseVoice/WAVPlay.cpp` 的 `Qt::QueuedConnection`、`PluginToolRegistrationView` 的 `QMetaObject::invokeMethod(..., Qt::QueuedConnection)`、`SurgicalNavigatWidget.Tracking.cpp` 把 QTimer 留在 GUI 线程。

## 重要提醒
- Qt 的坑大多是**运行时**而非编译期——审查时要在脑内"跑事件循环"，而不只是读静态调用。
- 跨线程问题与并发数据竞争高度相关：发现跨线程访问时，配合 [[cpp-concurrency-review]] 一起审。
- 不确定某 Qt 类的线程约束（是否 reentrant / 是否仅主线程）时，明确提示需查该类的 Qt 文档 "Thread-Safety" 段落，不要臆断。
