---
name: cross-platform-guard
description: Use when reviewing C++ code for Windows/Linux portability, compiler differences, paths, encoding, integer sizes, byte order, ABI, sockets, dynamic libraries, or CMake portability.
---

# 跨平台一致性审查 (cross-platform-guard)

"在我的 Linux 上好好的，到 Windows 就崩/乱码/编不过"——跨平台 bug 的根源往往不是大问题，而是一堆**沉默的假设**：假设 `long` 是 64 位、假设路径用 `/`、假设文件是 UTF-8、假设符号默认导出。这些假设在单平台上永远不会暴露。本 skill 的作用就是把这些隐藏假设逐项揪出来。

## 审查心法
对每段平台相关代码问三个问题：
1. **它隐含假设了哪个平台的行为？**（路径、类型大小、编码、API、符号可见性…）
2. **在另一平台上这个假设会怎样破？**（编不过 / 静默错误 / 运行崩溃 / 乱码）
3. **能否用跨平台抽象（std::filesystem / std::chrono / std::thread / 固定宽度类型）消除分支？**

> 好的跨平台代码不是 `#ifdef` 满天飞，而是**把平台差异收敛到少数抽象层**，业务逻辑保持平台无关。

## Checklist

### 1. 条件编译 / 平台抽象
- [ ] `#ifdef _WIN32 / __linux__ / __APPLE__` 是否**蔓延进业务逻辑**？应收敛到平台抽象层(PAL)，业务代码不感知平台。
- [ ] 平台分支是否**成对完整**（加了 Windows 分支但漏了 Linux，或反之）？有没有 `#else #error` 兜底未知平台？
- [ ] 宏判断是否正确？`_WIN32` 在 32/64 位 Windows 都定义；`_WIN64` 仅 64 位；`__linux__`、`__unix__`。

### 2. 文件系统 / 路径 / 编码（高频翻车区）
- [ ] **路径分隔符**：硬编码 `\` 或 `/`？用 `std::filesystem::path`（自动处理）。
- [ ] **大小写敏感**：Linux 文件名**区分大小写**，Windows 默认不区分。`#include "Foo.h"` 实际文件 `foo.h` → Linux 编不过、Windows 过。文件名/路径大小写要一致。
- [ ] **换行**：CRLF(Windows) vs LF(Linux)。文本模式读写、git `core.autocrlf`、解析协议时按字节算行尾。二进制文件务必用 `std::ios::binary`。
- [ ] **文件编码**：源码/数据文件 Windows 易出 GBK/UTF-16/带 BOM，Linux 默认 UTF-8 → 中文乱码。统一 UTF-8（必要时无 BOM），MSVC 用 `/utf-8`。
- [ ] Windows 路径长度 MAX_PATH(260) 限制、保留名(CON/PRN/NUL/COM1)、盘符；临时目录(`std::filesystem::temp_directory_path`)。

### 3. 整型大小 / 字节序 / 对齐（静默错误重灾区）
- [ ] **`long` 的大小不一致！** Windows(LLP64)：`long`=32 位；Linux 64 位(LP64)：`long`=64 位。`int` 两边都是 32 位、指针都是 64 位。需要固定宽度时用 **`<cstdint>` 的 `int32_t/int64_t/intptr_t`**，别用 `long` 传递需要确定宽度的值（尤其序列化/协议）。
- [ ] `size_t`/指针大小依赖位数——别假设 `sizeof(void*)==sizeof(int)`。
- [ ] **字节序**：x86/x64 都是小端，但**网络协议是大端**。跨机/落盘的二进制数据要显式转换(`htonl`/`ntohl` 或自写)，不要直接 `memcpy` 结构体。
- [ ] **结构体对齐 / padding**：直接把 struct `memcpy` 进文件/网络？不同编译器/平台 padding 可能不同。协议结构用 `#pragma pack` 或逐字段序列化，别依赖默认布局。
- [ ] `wchar_t` 宽度：Windows=16 位(UTF-16)，Linux=32 位(UTF-32/UCS4)——**绝不**跨平台传 `wchar_t` 数据。
- [ ] 有无依赖未定义/实现定义行为：有符号右移、`char` 默认有无符号(ARM 上常为 unsigned)、整型溢出。

### 4. 字符串 / 文本编码策略
- [ ] 字符串策略统一吗？推荐 **"UTF-8 everywhere"**：内部 `std::string` 存 UTF-8，仅在调用 Windows W API 边界转 UTF-16。
- [ ] Windows 上是否混用了 `TCHAR`/`A`/`W` API？文件名含非 ASCII 时，Windows 的 `*A` 函数按 ANSI 代码页解释会乱码——用 `*W` + UTF-16 或 UTF-8 转换。
- [ ] `std::filesystem::path` 在 Windows 内部是 `wchar_t`、Linux 是 `char`——用 `u8path`/`.u8string()` 处理含中文路径，别直接 `.string()` 丢编码。

### 5. 平台 API 差异（优先用标准库抹平）
- [ ] **线程**：优先 `std::thread`/`std::mutex`（已跨平台），避免裸 `pthread` / Win32 `CreateThread` 混用。
- [ ] **时间/计时**：用 `std::chrono::steady_clock`（单调，测耗时）而非 `gettimeofday`/`QueryPerformanceCounter` 各写一套。墙钟用 `system_clock`。
- [ ] **睡眠/定时精度**：Windows 默认定时器粒度约 15.6ms，`sleep_for(1ms)` 可能睡更久（需 `timeBeginPeriod` 或 Win10+ 高精度），Linux 通常更细。依赖高精度定时要注意平台差异。
- [ ] **Socket**：Winsock 需 `WSAStartup`/`WSACleanup`；句柄类型 `SOCKET`(非 int)；关闭用 `closesocket` 非 `close`；错误用 `WSAGetLastError` 非 `errno`；`EWOULDBLOCK` vs `WSAEWOULDBLOCK`；无 `MSG_NOSIGNAL`(用 setsockopt)。封装统一抽象层。
- [ ] **动态库**：`LoadLibrary`/`GetProcAddress`/`.dll` vs `dlopen`/`dlsym`/`.so`；搜索路径规则不同。
- [ ] **进程/信号**：`fork`/POSIX 信号在 Windows 无原生对应；用跨平台库或抽象。

### 6. 动态库符号可见性 / ABI
- [ ] **符号默认可见性相反**：Windows 默认**隐藏**，需 `__declspec(dllexport/dllimport)`；Linux/GCC 默认**全导出**，建议 `-fvisibility=hidden` + `__attribute__((visibility("default")))`。统一用一个**导出宏**（`MYLIB_API`）按平台展开，否则 Windows 上链接报未定义符号。
- [ ] MSVC **运行时库**一致性：`/MD`(动态) vs `/MT`(静态)，跨 DLL 边界混用会堆崩溃。所有模块与第三方库要一致。
- [ ] 跨 DLL 边界**传 STL 对象/在一侧 new 另一侧 delete**？ABI/分配器不匹配会崩。接口用 C ABI 或保证同编译器同运行时。
- [ ] name mangling 差异：导出 C 接口用 `extern "C"`。

### 7. 编译器差异 (MSVC / GCC / Clang)
- [ ] 用了编译器特有扩展？`__declspec` vs `__attribute__`、`#pragma` 差异、`__forceinline` vs `__attribute__((always_inline))`、VLA/零长数组(MSVC 不支持 VLA)。
- [ ] 各编译器**警告级别**与默认标准不同；建议统一 `-std=c++XX`/`/std:c++XX`、开 `-Wall -Wextra`/`/W4` 并按平台分别过一遍。
- [ ] 依赖了某编译器对 UB 的"恰好正确"行为？换平台即崩。

### 8. CMake 跨平台配置
- [ ] 是否硬编码路径/编译器/平台标志？用 **generator expressions**(`$<PLATFORM_ID>`、`$<CXX_COMPILER_ID>`) 和 `if(WIN32)/if(UNIX)`。
- [ ] 编译定义/选项用 **`target_compile_definitions`/`target_compile_options`**（带 PUBLIC/PRIVATE 作用域）而非全局 `add_definitions`。
- [ ] MSVC 运行时：用 `CMAKE_MSVC_RUNTIME_LIBRARY` 统一 /MD|/MT；MSVC 加 `/utf-8`、`/permissive-`、`/EHsc`、`NOMINMAX`、`WIN32_LEAN_AND_MEAN`。
- [ ] 依赖查找跨平台：用 `find_package`/vcpkg/Conan，别假设头/库的绝对路径。
- [ ] 运行期库加载：Linux 的 **RPATH/RUNPATH** vs Windows 把 DLL 放可执行同目录或 PATH；安装规则(`install(RUNTIME/LIBRARY ...)`)区分。

### 9. 第三方库
- [ ] 每个第三方库**两个平台都支持**吗？构建方式（vcpkg/Conan 能否同时覆盖）？
- [ ] 是否引入了**平台独占依赖**（仅 Windows 的 COM/DirectX、仅 Linux 的 X11/特定 .so）？是否有跨平台替代或抽象隔离？

## 输出格式（审查报告）

```
# 跨平台一致性审查报告

## 目标平台 / 工具链
- 平台：Linux(gcc/clang) + Windows(MSVC)... 位数：64-bit...

## 🔴 阻断级（另一平台编不过 / 必然崩溃 / 静默数据错误）
1. [文件:行] 问题：...  影响平台：Windows / Linux
   会怎么破：在 <平台> 上 <编译失败/运行崩溃/乱码/数据错位>
   修法：...

## 🟠 严重（偶发 / 编码 / 精度 / 符号可见性 / 运行时不一致）
...

## 🟡 建议（收敛 #ifdef / 用标准库抽象 / CMake 规范化）
...

## 验证建议
- 在两个平台的 CI 上都编+跑（最可靠）；MSVC 开 /W4 /permissive- /utf-8
- 对序列化/协议数据做跨平台往返测试（endianness、对齐、long 宽度）
```

## 重要提醒
- **最强的跨平台保证是双平台 CI**——静态审查能抓很多假设，但无法替代在两个平台真编真跑。务必建议把双平台纳入 CI。
- 涉及线程/时间精度的跨平台差异，与 [[cpp-concurrency-review]] 配合审。
- 不确定某 API/库在目标平台的确切行为时，明确标注"需在该平台实测/查文档"，不要替它担保。
