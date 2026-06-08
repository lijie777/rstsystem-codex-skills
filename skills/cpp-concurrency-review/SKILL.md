---
name: cpp-concurrency-review
description: Use when reviewing C++/Qt threaded, concurrent, multi-process, locking, atomic, QThread, condition-variable, IPC, or intermittent race/deadlock code.
---

# C++ 并发审查 (cpp-concurrency-review)

并发 bug 的可怕之处在于**不可复现**：它依赖时序，在开发机上跑一万次都对，到了用户多核机器、高负载或 Release 优化后才偶发崩溃/数据错乱。所以审查不能靠"读一遍觉得没问题"，必须**系统性地对照每一类已知陷阱逐项排查**，并把"为什么危险"讲清楚——因为很多并发 bug 在单线程直觉下看起来完全正常。

## 审查方法（先建模，再排查）

1. **先画出线程模型**：有几个线程/进程？谁创建谁、生命周期如何？每个线程的职责。
2. **标出共享可变状态**：哪些数据被多个线程访问？其中哪些会被写？**只读共享不需要同步；一旦有写，就必须有同步**。
3. **标出同步原语**：每块共享状态由哪个锁/原子保护？建立"数据 ↔ 保护它的锁"的映射。映射不清晰本身就是高危信号。
4. **逐类对照下面的 checklist**，对每个发现给出：位置、问题、**为什么危险（什么时序下会出错）**、修法方向、严重度。

> 关键心法：不要问"这段代码能工作吗"，要问"**存在某种线程交错(interleaving)使它出错吗**"。只要存在一种坏交错，就是 bug。

## Checklist

### 1. 数据竞争 (data race) —— 最常见、最隐蔽
- [ ] 是否存在**无任何同步**的共享可变变量被多线程读写？（这是 UB，不是"偶尔出错"，编译器可任意优化）
- [ ] 是否误用 `volatile` 当同步手段？`volatile` **不保证原子性也不建立 happens-before**，在 C++ 里它不是线程同步工具（与 Java 不同）。
- [ ] 是否有 **check-then-act / read-modify-write** 被误当原子？如 `if(!ptr) ptr=new...`、`count++`、`if(m.empty()) m.push(...)`——这些都是复合操作，需整体加锁或用原子。
- [ ] 用一个普通 `bool`/`int` 标志位做线程间通知（如 `running=false` 让另一线程退出）？应为 `std::atomic<bool>`，否则可能永远看不到更新。
- [ ] 容器（`std::vector/map`、`QList` 等）被多线程并发读写？标准容器对**同一对象的并发写不安全**（const 并发读 OK）。

### 2. 锁的使用
- [ ] 是否用 **RAII**（`std::lock_guard`/`std::unique_lock`/`std::scoped_lock`/`QMutexLocker`）而非手动 lock/unlock？手动解锁在异常/提前 return 路径上极易漏掉。
- [ ] **持锁期间是否调用了外部代码/回调/虚函数/信号**？这会把未知代码拉进临界区，可能重入加锁→死锁，或长时间持锁。
- [ ] **持锁期间是否做 I/O、阻塞调用、sleep、等待另一个锁或条件**？会放大竞争、诱发死锁。
- [ ] 锁粒度：过粗（一把大锁锁所有东西）→ 伪并发、吞吐崩塌；过细（每个字段一把锁）→ 易死锁、易漏。审查是否匹配实际竞争。
- [ ] **双重检查锁定 (DCLP)** 写法是否正确？经典 `if(!p){lock; if(!p) p=new...}` 在没有正确原子/内存序时是坏的。现代写法用 `std::call_once`+`std::once_flag`，或 `static` 局部变量（C++11 起线程安全初始化）。

### 3. 死锁 / 活锁
- [ ] **多把锁是否始终以固定全局顺序获取**？锁顺序不一致(ABBA)是死锁头号原因。需要同时持有多把锁时，用 `std::scoped_lock(m1,m2)` 或 `std::lock(...)` 一次性按死锁避免算法加锁。
- [ ] 是否存在**嵌套加锁**（持有 A 再去拿 B），尤其经过函数调用链隐藏的？
- [ ] 递归进入同一非递归锁？（同线程二次 lock 普通 `std::mutex` 是 UB/死锁）
- [ ] 是否需要超时获取（`try_lock_for`）以打破潜在死锁？
- [ ] 活锁/饥饿：自旋重试是否可能永远让步？

### 4. std::atomic 与内存序 (memory order)
- [ ] 默认是否用 `memory_order_seq_cst`（最强最稳）？**只有在确有性能证据时才下降到 acquire/release/relaxed**——降级是最容易出错的优化。
- [ ] 用了 `memory_order_relaxed` 做**同步/可见性**？relaxed 只保证单变量原子性，**不建立 happens-before**，常被误用导致另一线程看到半初始化数据。
- [ ] `acquire`/`release` 是否**成对**？release 写必须有对应的 acquire 读才建立同步。
- [ ] `compare_exchange_weak` vs `strong` 用对了吗？weak 可能伪失败，必须放在循环里。
- [ ] **ABA 问题**：无锁结构里指针被改回原值导致 CAS 误判？
- [ ] **伪共享 (false sharing)**：多个线程频繁写的原子/变量是否落在同一 cache line（64B）？考虑 `alignas(64)` 隔离。

### 5. 条件变量 (condition_variable)
- [ ] wait 是否**带谓词**（`cv.wait(lk, []{return ready;})`）或放在 `while(!pred) cv.wait(lk)` 循环里？裸 wait 会被**虚假唤醒 (spurious wakeup)** 坑到。
- [ ] **丢失唤醒 (lost wakeup)**：设置条件与 notify 之间，是否在持有同一把锁的保护下？若 notify 先于 wait 且条件未被检查，会永久阻塞。
- [ ] notify 时机：修改共享条件 → （可在锁内或刚解锁后）notify，确保等待方不会错过。

### 6. 线程生命周期
- [ ] `std::thread` 对象析构前是否 **join 或 detach**？否则析构调用 `std::terminate` 直接崩。注意异常路径——用 RAII 包装(jthread 或自写 guard)。
- [ ] `detach` 后线程是否可能访问**已析构的对象/栈变量**？分离线程的生命周期极难管理，优先 join。
- [ ] 线程函数是否可能**抛出异常逃逸**？逃出线程函数的异常 → `std::terminate`。线程入口要 try/catch。
- [ ] 传给线程的引用/指针/lambda 捕获，其指向对象的生命周期是否覆盖线程运行期？（捕获 `this` 后对象先析构是经典崩因）
- [ ] C++20 可用 `std::jthread` + `stop_token` 做协作式取消，比手写标志位更稳。

### 7. 多进程 / IPC
- [ ] 共享内存中的数据结构是否跨进程正确同步（进程间互斥/命名信号量/原子）？普通 `std::mutex` **不能跨进程**。
- [ ] `fork()` 之后、`exec()` 之前是否只调用 **async-signal-safe** 函数？多线程程序 fork 后子进程只有一个线程，持有的锁可能永远锁住（malloc 死锁是经典）。
- [ ] 信号处理函数里是否只做 async-signal-safe 操作（如只写 `volatile sig_atomic_t`）？
- [ ] 文件锁/记录锁的范围与释放时机是否正确？

### 8. 第三方库的线程安全假设
- [ ] 该库/对象是**线程安全、可重入、还是必须单线程**？查文档，别假设。
- [ ] 库是否有**全局/静态状态**（如某些 C 库的 errno 风格、非重入解析函数）？
- [ ] 库的**回调在哪个线程**触发？（网络/渲染/Qt 库常在内部线程回调，碰你的数据需同步）
- [ ] 是否要求"只能在创建它的线程使用"（很多 GUI/渲染/Qt 对象如此）？

## 输出格式（审查报告）

按严重度分级，让用户先看最致命的。对每条**解释会在什么时序下出错**，而不只是"这里有竞争"。

```
# 并发审查报告

## 线程模型（我的理解）
- 线程/进程清单与职责
- 共享可变状态 → 保护它的同步原语（映射表）

## 🔴 阻断级（数据竞争 / 死锁 / UB，必须修）
1. [文件:行] 问题：...
   为什么危险：在「线程A...线程B...」这种交错下会 <具体后果>
   修法：...

## 🟠 严重（偶发错误 / 资源泄漏 / 误用内存序）
...

## 🟡 建议（锁粒度 / 伪共享 / 可用更稳的写法）
...

## 验证建议
- 用 ThreadSanitizer (clang/gcc `-fsanitize=thread`) 跑现有用例
- Linux: Valgrind Helgrind / DRD；Windows: /analyze、Application Verifier
- 压力测试：多核 + Release + 高并发循环复现
```

## 重要提醒
- **没看到坏交错 ≠ 没有 bug**。如果某块共享状态的同步关系你无法清晰说明，就如实标为"无法确认安全"，不要替它担保。
- 静态审查抓不全时序 bug——**务必建议 ThreadSanitizer 等动态工具**作为补充，二者互补。
- 不确定第三方库的线程语义时，明确说"需查 X 库文档确认"，不要臆断。
