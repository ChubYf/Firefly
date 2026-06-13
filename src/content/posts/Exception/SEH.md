---
title: SEH异常处理
published: 2026-06-3
description: x86和x64下的SEH异常处理
image: ./cover.jpg
tags: [逆向]
category: windows异常处理
draft: false
---


[深入解析结构化异常处理(SEH) - by Matt Pietrek-CSDN博客](https://blog.csdn.net/chenlycly/article/details/52575260)
# 前言
windowsSEH异常处理机制在x86和x64下是不同的。想要深入了解异常处理机制需要进入到内核中，这里只在用户态分析SEH异常处理机制。

# SEH异常处理实现
想要了解异常处理机制，得先看看怎么编写异常处理。
基于ai生成:
##  结构化异常处理（SEH）概述

**结构化异常处理（Structured Exception Handling, SEH）** 是微软对 C/C++ 语言的扩展，主要用于处理**硬件/系统级异常**，如**访问违规（Access Violation）**、**除零错误（Division by Zero）**、**空指针解引用（Null Pointer Dereference）** 等。语法上通过 `__try`、`__except`、`__finally` 三个关键字来实现。

> **核心特性**
> 
> - **硬件级异常捕获**：能捕获操作系统抛出的硬件异常（例如除零、内存访问违规）。
>     
> - **资源清理保证**：`__finally` 块可以确保即使在异常发生时，也能执行必要的清理操作（例如释放内存、关闭句柄）。  
>     **注意**：如果 `__try` 块内调用了 `ExitProcess` 或 `ExitThread`，`__finally` 块将**不会**被执行。
>     
> - **异步处理机制**：异常的发生与正常控制流**无关**，可能在**任何时刻**触发。
>



## 核心语法

### `__try`/`__except` 异常捕获与处理

```c
__try {
    // 受保护的代码块（可能触发异常）
}
__except(filter_expression) {
    // 异常处理代码块
}
```

| 返回值宏                           | 数值  | 行为说明                        |
| ------------------------------ | --- | --------------------------- |
| `EXCEPTION_EXECUTE_HANDLER`    | 1   | 识别异常，并执行 `__except` 块       |
| `EXCEPTION_CONTINUE_SEARCH`    | 0   | 不处理，继续向上层搜索异常处理程序           |
| `EXCEPTION_CONTINUE_EXECUTION` | -1  | 消除异常，并从异常发生点**继续执行**（需谨慎使用） |

> **注意**：`EXCEPTION_CONTINUE_EXECUTION` 只有在异常能被安全恢复的情况下才能使用；对于不可继续的异常，若筛选器返回 -1，系统会产生新的异常。



###  `__try` / `__finally`  资源清理与终止处理

```c
__try {
    // 受保护的代码块
}
__finally {
    // 始终执行的清理代码块
}
```
`__finally` 块无论 `__try` 块是以**正常方式**（执行完所有语句）还是**异常方式**（因异常、`return`、`goto` 等）退出，**都会被调用**。

> **使用建议**：`__finally` 内部可通过 `AbnormalTermination()` 函数判断 `__try` 块是否正常终止。若需要从 `__try` 块内"提前退出"且**避免异常终止带来的性能开销**，可以使用 `__leave` 语句

```c
__try {
    // ... 一些代码 ...
    if (某个条件) __leave;   // 直接跳转到 __finally 块
    // ... 后续代码 ...
}
__finally {
    // 始终执行的清理代码
}
```


###  `filter_expression` 筛选器高级用法

### 标准模式：按异常类型分类处理
```c
__try {
    // 可能触发异常的操作
}
__except(GetExceptionCode() == EXCEPTION_INT_DIVIDE_BY_ZERO ? 
         EXCEPTION_EXECUTE_HANDLER : EXCEPTION_CONTINUE_SEARCH) {
    // 仅处理除零异常，其他异常继续向上传递
    printf("除零异常被捕获！\n");
}
```
GetExceptionCode是visualstudio中定义的宏(辅助函数),在visualstudio中可以查看类似定义的辅助函数
```
// SEH intrinsics
#define GetExceptionCode            _exception_code
#define exception_code              _exception_code
#define GetExceptionInformation()   ((struct _EXCEPTION_POINTERS *)_exception_info())
#define exception_info()            ((struct _EXCEPTION_POINTERS *)_exception_info())
#define AbnormalTermination         _abnormal_termination
#define abnormal_termination        _abnormal_termination
```


### 调用外部筛选函数：处理复杂逻辑
```c
int FilterFunction(DWORD dwExceptionCode) {
    if (dwExceptionCode == STATUS_INTEGER_OVERFLOW || 
        dwExceptionCode == STATUS_FLOAT_OVERFLOW) {
        // 执行清理操作
        ResetGlobalVariables();
        return EXCEPTION_CONTINUE_EXECUTION;   // 尝试恢复执行
    }
    return EXCEPTION_CONTINUE_SEARCH;          // 不处理，继续向上
}

__try {
    // 可能发生整数溢出或浮点溢出的代码
}
__except(FilterFunction(GetExceptionCode())) {
    // 当 FilterFunction 返回 EXCEPTION_EXECUTE_HANDLER 时执行
}
```
**关键点**：

- `GetExceptionCode()` **必须**在 `__except` 表达式中**直接调用**，不能通过自定义函数间接调用。
    
- `GetExceptionInformation()` 用于获取更详细的异常上下文信息（如异常发生地址）。
    
- `AbnormalTermination()` 只能在 `__finally` 块内调用，返回 `TRUE` 表示 `__try` 块**非正常终止**。



## 常用辅助函数与 API 一览
| 函数/API                          | 用途                                      | 使用位置                           |
| ------------------------------- | --------------------------------------- | ------------------------------ |
| `GetExceptionCode()`            | 获取当前异常的异常代码（`DWORD` 类型）                 | **必须在 `__except` 筛选器表达式中直接调用** |
| `GetExceptionInformation()`     | 获取包含异常详细信息的结构体指针（`EXCEPTION_POINTERS*`） | **必须在 `__except` 筛选器表达式中直接调用** |
| `AbnormalTermination()`         | 判断 `__try` 块是否以**异常方式**终止               | `__finally` 块内                 |
| `RaiseException()`              | 手动触发一个结构化异常                             | 任何位置                           |
| `SetUnhandledExceptionFilter()` | 设置**未处理异常**的最终捕获函数                      | 进程级全局设置                        |
## `__finally` 块执行保证与限制
**执行情况对比**：

|退出方式|`__finally` 是否执行|是否异常终止|
|---|---|---|
|执行完 `__try` 块最后一条语句|✅ **执行**|❌ 正常终止|
|因结构化异常退出|✅ **执行**|✅ 异常终止|
|通过 `return` 退出|✅ **执行**|✅ 异常终止|
|通过 `goto`/`break`/`continue` 退出|✅ **执行**|✅ 异常终止|
|调用 `longjmp`|✅ **执行**|✅ 异常终止|
|调用 `ExitProcess`/`ExitThread`|❌ **不执行**|-|
## 常见错误代码速查表
| 异常代码宏                          | 数值         | 对应异常场景               |
| ------------------------------ | ---------- | -------------------- |
| `EXCEPTION_ACCESS_VIOLATION`   | 0xC0000005 | 内存访问违规（读/写无效地址）      |
| `EXCEPTION_INT_DIVIDE_BY_ZERO` | 0xC0000094 | 整数除零错误               |
| `EXCEPTION_STACK_OVERFLOW`     | 0xC00000FD | 栈溢出                  |
| `EXCEPTION_BREAKPOINT`         | 0x80000003 | 断点异常（`DebugBreak()`） |
| `EXCEPTION_FLT_DIVIDE_BY_ZERO` | 0xC000008E | 浮点数除零                |
| `STATUS_INTEGER_OVERFLOW`      | 0xC0000095 | 整数溢出                 |
| `STATUS_FLOAT_OVERFLOW`        | 0xC0000091 | 浮点数溢出                |





# x86下的SEH

## 底层实现机制



### 前置知识

#### 异常注册链`EXCEPTION_REGISTRATION_RECORD`
x86 下，每个线程的 **线程信息块（TIB）** 的第一个字段（偏移 `0`）就是**当前异常注册链表头指针**：即 `fs : [0]`
我们可以通过这个网站去查看(这个网站也可以查看一些微软未公开的结构体):[Vergilius Project | Home](https://www.vergiliusproject.com/)
点进去后选择x86 -> windows10 -> 随便选一个版本 -> 输入_teb
![](assets/Pasted%20image%2020260602154639.png)
![](assets/Pasted%20image%2020260602154649.png)
![](assets/Pasted%20image%2020260602154717.png)
可以看到 `fs : [0]` 最终指向的是异常注册帧
```c
typedef struct _EXCEPTION_REGISTRATION_RECORD {
    struct _EXCEPTION_REGISTRATION_RECORD *Next;
    PEXCEPTION_ROUTINE Handler;
} EXCEPTION_REGISTRATION_RECORD;
```
其中Next指向的是下一个注册帧，Handler是函数指针，指向当前异常处理函数.
EXCEPTION_ROUTINE:
```c
typedef
_IRQL_requires_same_
_Function_class_(EXCEPTION_ROUTINE)
EXCEPTION_DISPOSITION
NTAPI
EXCEPTION_ROUTINE (
    _Inout_ struct _EXCEPTION_RECORD *ExceptionRecord,
    _In_ PVOID EstablisherFrame,
    _Inout_ struct _CONTEXT *ContextRecord,
    _In_ PVOID DispatcherContext
    );

typedef EXCEPTION_ROUTINE *PEXCEPTION_ROUTINE;
```
结构体简化为
```c
EXCEPTION_DISPOSITION __cdecl _except_handler(
    EXCEPTION_RECORD         *ExceptionRecord,  // 异常信息
    EXCEPTION_REGISTRATION_RECORD *EstablisherFrame, // 当前帧的基址（通常是EBP或ESP）
    CONTEXT                  *ContextRecord,    // 异常发生时线程上下文
    void                     *DispatcherContext // 分发器上下文（展开时使用）
);
```
返回值EXCEPTION_DISPOSITION
```c
typedef enum _EXCEPTION_DISPOSITION
{
    ExceptionContinueExecution,
    ExceptionContinueSearch,
    ExceptionNestedException,
    ExceptionCollidedUnwind
} EXCEPTION_DISPOSITION;
```

- `FS:[0]` 始终指向**栈上最新的异常注册帧**（即最后压入的 `__try` 块对应的帧）。
    
- 当一个新的 `__try` 块被激活时，编译器会：
    
    1. 在栈上分配一个 `EXCEPTION_REGISTRATION_RECORD`。
        
    2. 将 `FS:[0]` 的旧值存入该帧的 `Next` 字段。
        
    3. 将 `FS:[0]` 更新为当前帧的地址。
        
- 退出 `__try` 时，编译器将 `FS:[0]` 恢复为帧中的 `Next` 值。



#### scopetable作用域表
**scopetable** 是一个**只读常量数组**，位于 `.rdata` 段，每个**包含 SEH 块的函数**都有一张对应的 scopetable。表中每一项描述函数内的一个“保护区域”（即一个 `__try` 块），以及与之关联的筛选器、处理程序或清理函数。

```c
struct _SCOPETABLE_ENTRY {
    DWORD       EnclosingLevel;   // 外层嵌套级别索引，-1 表示顶层（无外层）
    PEXCEPTION_HANDLER FilterFunc;  // 筛选器函数地址（__except 专用）
    PEXCEPTION_HANDLER HandlerFunc; // 处理程序/清理函数地址
};
```
- **`EnclosingLevel`**：  
    指向**外层作用域**在 scopetable 中的索引。如果当前 `__try` 块不被任何其他 `__try` 块包围（即处于最外层），该值为 `-1`（`0xFFFFFFFF`）。
    
- **`FilterFunc`**：  
    对于 `__except` 块，这里存放**筛选器表达式**编译后生成的函数入口地址。  
    对于 `__finally` 块，该字段为 `NULL`。
    
- **`HandlerFunc`**：  
    对于 `__except` 块，指向**异常处理块**（`__except { ... }`）的代码地址。  
    对于 `__finally` 块，指向**终止处理块**（`__finally { ... }`）的代码地址。

**重要**：`FilterFunc` 和 `HandlerFunc` 的签名与标准 `_except_handler` 相同：
```c

EXCEPTION_DISPOSITION _cdecl Func(EXCEPTION_RECORD*, EXCEPTION_REGISTRATION_RECORD*, CONTEXT*, void*);

```
但它们**不是**完整的异常处理程序，而是由编译器生成的“桩函数”，内部包含用户编写的筛选器代码或清理代码。


**scopetable 的构建顺序：**
编译器为函数中的每个 `__try` 块按**词法出现顺序**分配一个索引，但索引的数值并不是直接嵌套深度，而是用于快速定位。为了支持嵌套，`EnclosingLevel` 字段指向**包含当前块的外层块**的索引。

考虑一个嵌套示例：

```cpp

void Func() {
    __try {               // 索引 0
        __try {           // 索引 1
            // ...
        }
        __finally {       // 索引 2
        }
    }
    __except(/* ... */) { // 索引 3
    }
}
```

对应的 scopetable（伪）：

| 索引  | EnclosingLevel | FilterFunc | HandlerFunc     |
| --- | -------------- | ---------- | --------------- |
| 0   | -1             | NULL       | （无，外层__try没有处理） |
| 1   | 0              | NULL       | （内层__try没有独立处理） |
| 2   | 0              | NULL       | `__finally` 块地址 |
| 3   | -1             | 筛选器函数地址    | `__except` 块地址  |

> **注意**：并非每个 `__try` 都直接生成一个 scopetable 条目——实际 MSVC 会为每个保护区域（包括 `__try` 块自身及其内部的 `__except`/`__finally`）分配条目。具体细节依赖于编译器的实现版本，但上述抽象概念足够理解机制。



#### trylevel嵌套层级
**try_level** 是一个**函数局部变量**，存放在当前函数的**栈帧**中（通常紧邻 EBP）。它指示**当前正在执行的语句属于哪一个 scopetable 条目**（即当前活跃的 `__try` 块）。
- **含义**：try_level 的值是一个**整数索引**，指向 scopetable 中的某个条目。该条目描述了最内层（最近进入的）`__try` 块所对应的作用域。
    
- **更新时机**：
    
    - **进入 `__try` 块时**：编译器将 try_level 设置为该 `__try` 块对应的 scopetable 索引。
        
    - **退出 `__try` 块时**：编译器将 try_level 恢复为**外层块**的索引（即该块的 `EnclosingLevel` 值）。
        
    - **如果当前不在任何 `__try` 块内**：try_level 通常为 `-1`。

栈帧布局示例
```
高地址
+-------------------+  
| old EBP(调用者的ebp)  <-  当前EBP指向
| try_level          :   0FFFFFFFFh    |  ← 当前嵌套级别
| scopetable 指针     :   1665F8h       |  ← 指向 .rdata 中的常量表
| Next  (异常注册帧)   :   016328Ch      |  ← 指向链表中上一帧(下一个EXCEPTION_REGISTRATION_RECORD)
| Handler (异常注册帧) :   fs : [0]      |  ← 指向 _except_handler3/4
| ESP值
| ...
| ...
+-------------------+ ← 当前 ESP
低地址
```
- `pScopeTable`：编译时确定的函数 scopetable 基址，**在函数入口压入栈中**，供 `_except_handler4` 使用。
    
- `try_level`：随着 `__try` 块的进入/退出动态变化。




#### exception_handler3
`_except_handler3` 是 MSVC 实现 SEH 机制的**核心枢纽**。它充当了操作系统异常分发器与编译器之间编排的“中间层”，负责解析静态的 `scopetable` 和动态的 `trylevel`，以决定异常的处理方式
```c
int _except_handler3(
    struct _EXCEPTION_RECORD     *ExceptionRecord, // 异常信息
    struct EXCEPTION_REGISTRATION *Registration,   // 指向当前函数的异常帧
    struct _CONTEXT               *Context,        // 异常发生时线程上下文
    void                          *Dispatcher      // 分发器上下文，用于展开阶段
);
```
##### 扩展的异常注册帧

为了让单个函数能管理多个 `__try` 块，编译器扩展了标准的 `EXCEPTION_REGISTRATION_RECORD` 结构
`_except_handler3` 的每次调用分为**筛选阶段**和**展开阶段**，由 `ExceptionRecord->ExceptionFlags` 区分[](https://www.e-com-net.com/article/4788691.htm)[](https://mirror.iscas.ac.cn/riscv-toolchains/git/riscv-collab/riscv-gnu-toolchain/newlib.git/diff/winsup/mingw/samples/seh/eh3.c?h=github/newlib-1_17_0-arc&id=0f285615efa16c783821cd87a1fdabe4913da9a7)。

##### 筛选阶段（第一次调用）

此时 `ExceptionFlags` 未设置展开标志，负责寻找一个愿意处理当前异常的 `__except` 块：

1. **初始化**：在栈上构造 `EXCEPTION_POINTERS` 结构，其地址存入当前函数栈帧固定偏移处，供 `GetExceptionInformation()` 返回[](https://www.e-com-net.com/article/4788691.htm)[](https://blog.51cto.com/u_13927568/5831516)[](https://mirror.iscas.ac.cn/riscv-toolchains/git/riscv-collab/riscv-gnu-toolchain/newlib.git/diff/winsup/mingw/samples/seh/eh3.c?h=github/newlib-1_17_0-arc&id=0f285615efa16c783821cd87a1fdabe4913da9a7)。
    
2. **遍历 `scopetable`**：从当前 `Registration->trylevel` 开始，通过 `previousTryLevel` 向上遍历[](https://blog.csdn.net/2301_80520893/article/details/134925730)[](https://www.e-com-net.com/article/4788691.htm)[](https://mirror.iscas.ac.cn/riscv-toolchains/git/riscv-collab/riscv-gnu-toolchain/newlib.git/diff/winsup/mingw/samples/seh/eh3.c?h=github/newlib-1_17_0-arc&id=0f285615efa16c783821cd87a1fdabe4913da9a7)：
    
    - 若条目有 `lpfnFilter` (即 `__except` 块)，调用该过滤函数。
        
    - 根据返回值决定路径：
        
        - `EXCEPTION_EXECUTE_HANDLER (1)`：**【匹配处理】**，找到处理者，退出搜索。
            
        - `EXCEPTION_CONTINUE_SEARCH (0)`：**【向上传递】**，继续按 `previousTryLevel` 向上层搜索。
            
        - `EXCEPTION_CONTINUE_EXECUTION (-1)`：**【尝试恢复】**，通知系统从异常点**重新执行**指令[](https://blog.csdn.net/2301_80520893/article/details/134925730)。
            
3. **搜索结束**：若遍历完整个 `scopetable` 或 `previousTryLevel = -1` 仍未匹配，返回 `DISPOSITION_CONTINUE_SEARCH`，让操作系统继续向上帧搜索[](https://mirror.iscas.ac.cn/riscv-toolchains/git/riscv-collab/riscv-gnu-toolchain/newlib.git/diff/winsup/mingw/samples/seh/eh3.c?h=github/newlib-1_17_0-arc&id=0f285615efa16c783821cd87a1fdabe4913da9a7)。



 ##### 展开阶段（第二次调用）

筛选阶段匹配到 `EXCEPTION_EXECUTE_HANDLER` 后，系统调用 `RtlUnwind` 重新调用 `_except_handler3`（设置 `EXCEPTION_UNWINDING` 标志），负责资源清理：

1. **识别展开**：检查到展开标志，进入展开分支.
    
2. **执行 `__finally` 块**：从当前 `trylevel` 向上遍历，为每个条目执行 `lpfnHandler`（若为 `__finally` 块）。此过程确保所有嵌套的资源被正确清理，无论是否发生异常。
    
3. **移交控制权**：展开至目标 `__except` 块后，流程直接**跳转至**其 `lpfnHandler` 代码（用户编写的 `__except` 块），**不再返回**原调用者。





### 在汇编代码中理解SEH底层机制
首先在visual studio中进行一些设置让汇编代码更简洁
右键项目 -> 属性 -> c/c++ -> 所有选项 -> 
安全检查 -> 禁用安全检查
基本运行时检查 -> 默认值
支持仅我的代码调试 -> 否

#### 对照组
以空的main函数为对照组
源代码
```c
#include <stdio.h>
#include <Windows.h>

int main()
{
	
	
	return 0;
}
```
汇编代码
```d
#include <stdio.h>
#include <Windows.h>

int main()
{
002B1570 55                   push        ebp  
002B1571 8B EC                mov         ebp,esp  
002B1573 83 EC 40             sub         esp,40h  
002B1576 53                   push        ebx  
002B1577 56                   push        esi  
002B1578 57                   push        edi  
	
	
	return 0;
002B1579 33 C0                xor         eax,eax  
}
002B157B 5F                   pop         edi  
002B157C 5E                   pop         esi  
002B157D 5B                   pop         ebx  
002B157E 8B E5                mov         esp,ebp  
002B1580 5D                   pop         ebp  
002B1581 C3                   ret
```


#### 单个`__try,__except`情况
源代码:
```c
#include <stdio.h>
#include <Windows.h>

int main()
{
	__try
	{
		int a = 1;
	}
	__except(EXCEPTION_EXECUTE_HANDLER)
	{
	}
	
	return 0;
}
```
汇编代码
```c

int main()
{
00161570 55                   push        ebp  
00161571 8B EC                mov         ebp,esp  
00161573 6A FF                push        0FFFFFFFFh  
00161575 68 F8 65 16 00       push        1665F8h  
0016157A 68 8C 32 16 00       push        offset __except_handler3 (016328Ch)  
0016157F 64 A1 00 00 00 00    mov         eax,dword ptr fs:[00000000h]  
00161585 50                   push        eax  
00161586 64 89 25 00 00 00 00 mov         dword ptr fs:[0],esp  
0016158D 83 C4 B4             add         esp,0FFFFFFB4h  
00161590 53                   push        ebx  
00161591 56                   push        esi  
00161592 57                   push        edi  
00161593 89 65 E8             mov         dword ptr [ebp-18h],esp  
	__try
00161596 C7 45 FC 00 00 00 00 mov         dword ptr [ebp-4],0  
	{
		int a = 1;
0016159D C7 45 E4 01 00 00 00 mov         dword ptr [ebp-1Ch],1  
	}
001615A4 C7 45 FC FF FF FF FF mov         dword ptr [ebp-4],0FFFFFFFFh  
001615AB EB 10                jmp         $LN6+0Ah (01615BDh)  
	__except(EXCEPTION_EXECUTE_HANDLER)
001615AD B8 01 00 00 00       mov         eax,1  
001615B2 C3                   ret  
001615B3 8B 65 E8             mov         esp,dword ptr [ebp-18h]  
	}
001615B6 C7 45 FC FF FF FF FF mov         dword ptr [ebp-4],0FFFFFFFFh  
	{
	}
	
	return 0;
001615BD 33 C0                xor         eax,eax  
}
001615BF 8B 4D F0             mov         ecx,dword ptr [ebp-10h]  
001615C2 64 89 0D 00 00 00 00 mov         dword ptr fs:[0],ecx  
001615C9 5F                   pop         edi  
001615CA 5E                   pop         esi  
001615CB 5B                   pop         ebx  
001615CC 8B E5                mov         esp,ebp  
001615CE 5D                   pop         ebp  
001615CF C3                   ret
```

与对照组相比后增加的代码
```c
#include <stdio.h>
#include <Windows.h>

int main()
{
00161573 6A FF                push        0FFFFFFFFh  
00161575 68 F8 65 16 00       push        1665F8h  
0016157A 68 8C 32 16 00       push        offset __except_handler3 (016328Ch)  
0016157F 64 A1 00 00 00 00    mov         eax,dword ptr fs:[00000000h]  
00161585 50                   push        eax  
00161586 64 89 25 00 00 00 00 mov         dword ptr fs:[0],esp  
0016158D 83 C4 B4             add         esp,0FFFFFFB4h   
00161593 89 65 E8             mov         dword ptr [ebp-18h],esp  
	__try
00161596 C7 45 FC 00 00 00 00 mov         dword ptr [ebp-4],0  
	{
		int a = 1;
0016159D C7 45 E4 01 00 00 00 mov         dword ptr [ebp-1Ch],1  
	}
001615A4 C7 45 FC FF FF FF FF mov         dword ptr [ebp-4],0FFFFFFFFh  
001615AB EB 10                jmp         $LN6+0Ah (01615BDh)  
	__except(EXCEPTION_EXECUTE_HANDLER)
001615AD B8 01 00 00 00       mov         eax,1  
001615B2 C3                   ret  
001615B3 8B 65 E8             mov         esp,dword ptr [ebp-18h]  
	}
001615B6 C7 45 FC FF FF FF FF mov         dword ptr [ebp-4],0FFFFFFFFh  
	{
	}
	

}
001615BF 8B 4D F0             mov         ecx,dword ptr [ebp-10h]  
001615C2 64 89 0D 00 00 00 00 mov         dword ptr fs:[0],ecx  

```
我们先忽略掉`__try,__except`块的代码，分析在`__try,except`之外多出来的汇编代码
```d
00161573 6A FF                push        0FFFFFFFFh  
00161575 68 F8 65 16 00       push        1665F8h  
0016157A 68 8C 32 16 00       push        offset __except_handler3 (016328Ch)  
0016157F 64 A1 00 00 00 00    mov         eax,dword ptr fs:[00000000h]  
00161585 50                   push        eax  
00161586 64 89 25 00 00 00 00 mov         dword ptr fs:[0],esp  
0016158D 83 C4 B4             add         esp,0FFFFFFB4h   
00161593 89 65 E8             mov         dword ptr [ebp-18h],esp  

try
except

001615BF 8B 4D F0             mov         ecx,dword ptr [ebp-10h]  
001615C2 64 89 0D 00 00 00 00 mov         dword ptr fs:[0],ecx  
```
其中try块前的代码就是编译器为SEH生成的栈结构
前面四个push依次将try_level, scopetable, Next, Handler压栈。
`00161593 89 65 E8             mov         dword ptr [ebp-18h],esp  `这条指令实际上是把当前ESP的值放在Handler下面(往低地址方向)(`[ebp - 18h]`刚好指向的是Handler下面(往低地址方向)四字节位置)
```
高地址
+-------------------+  
| old EBP(调用者的ebp)  <-  当前EBP指向
| try_level          :   0FFFFFFFFh    |  ← 当前嵌套级别
| scopetable 指针     :   1665F8h       |  ← 指向 .rdata 中的常量表
| Next  (异常注册帧)   :   016328Ch      |  ← 指向链表中上一帧(下一个EXCEPTION_REGISTRATION_RECORD)
| Handler (异常注册帧) :   fs : [0]      |  ← 指向 _except_handler3/4
| ESP值
| ...
| ...
+-------------------+ ← 当前 ESP
低地址
```

在except块后
```c
001615BF 8B 4D F0             mov         ecx,dword ptr [ebp-10h]  
001615C2 64 89 0D 00 00 00 00 mov         dword ptr fs:[0],ecx  
```
这个操作是把异常注册帧还原。使`fs : [0]`指向原本的位置 (`[ebp - 10h]`即Next)，把Next值放回`fs : [0]`




#### 嵌套`__try, __except`情况
同理和上面一样抽取不同的部分，然后发现其实也只有一个异常处理栈帧结构
```c
#include <stdio.h>
#include <Windows.h>

int main()
{
	__try
	{
		__try
		{

		}
		__except (EXCEPTION_EXECUTE_HANDLER)
		{

		}
	}
	__except(EXCEPTION_EXECUTE_HANDLER)
	{

	}
	
	return 0;
}
```

```c
008E1570 55                   push        ebp  
008E1571 8B EC                mov         ebp,esp  
008E1573 6A FF                push        0FFFFFFFFh  
008E1575 68 F8 65 8E 00       push        8E65F8h  
008E157A 68 AC 32 8E 00       push        offset __except_handler3 (08E32ACh)  
008E157F 64 A1 00 00 00 00    mov         eax,dword ptr fs:[00000000h]  
008E1585 50                   push        eax  
008E1586 64 89 25 00 00 00 00 mov         dword ptr fs:[0],esp  
008E158D 83 C4 B8             add         esp,0FFFFFFB8h  
008E1593 89 65 E8             mov         dword ptr [ebp-18h],esp
...
...
...
008E15D8 8B 4D F0             mov         ecx,dword ptr [ebp-10h]  
008E15DB 64 89 0D 00 00 00 00 mov         dword ptr fs:[0],ecx

```





#### 并列多个`__try, __except`情况
和上面情况是一样的，在一个函数内只会注册一个异常处理栈帧结构，然后通过trylevel和scopetable定位到对应的try,except,finally块。
```c
#include <stdio.h>
#include <Windows.h>

int main()
{
	int a = 1;
	int* p = NULL;
	__try
	{
		*p = 2;
		__try
		{
			*p = 3;
		}
		__except(EXCEPTION_EXECUTE_HANDLER)
		{ 

		}
	}
	__except(EXCEPTION_EXECUTE_HANDLER)
	{
		
	}
	
	__try
	{
		*p = 4;
	}
	__except (EXCEPTION_EXECUTE_HANDLER)
	{

	}

	printf("%d", a);
	return 0;
}
```

#### 定位scopetable并分析trylevel变化

```d
int main()
{
00721610  push        ebp  
00721611  mov         ebp,esp  
00721613  push        0FFFFFFFFh  
00721615  push        7265F8h  
0072161A  push        offset __except_handler3 (072340Ch)  
0072161F  mov         eax,dword ptr fs:[00000000h]  
00721625  push        eax  
00721626  mov         dword ptr fs:[0],esp  
0072162D  add         esp,0FFFFFFB0h  
00721630  push        ebx  
00721631  push        esi  
00721632  push        edi  
00721633  mov         dword ptr [ebp-18h],esp  
	int a = 1;
00721636  mov         dword ptr [a],1  
	int* p = NULL;
0072163D  mov         dword ptr [p],0  
	__try
00721644  mov         dword ptr [ebp-4],0  
	{
		*p = 2;
0072164B  mov         eax,dword ptr [p]  
0072164E  mov         dword ptr [eax],2  
		__try
00721654  mov         dword ptr [ebp-4],1  
		{
			*p = 3;
0072165B  mov         eax,dword ptr [p]  
0072165E  mov         dword ptr [eax],3  
		}
00721664  mov         dword ptr [ebp-4],0  
0072166B  jmp         main+6Dh (072167Dh)  
		__except(EXCEPTION_EXECUTE_HANDLER)
0072166D  mov         eax,1  
00721672  ret  
00721673  mov         esp,dword ptr [ebp-18h]  
		}
00721676  mov         dword ptr [ebp-4],0  
		{ 

		}
	}
0072167D  mov         dword ptr [ebp-4],0FFFFFFFFh  
00721684  jmp         main+86h (0721696h)  
	__except(EXCEPTION_EXECUTE_HANDLER)
00721686  mov         eax,1  
0072168B  ret  
0072168C  mov         esp,dword ptr [ebp-18h]  
		{ 

		}
	}
0072168F  mov         dword ptr [ebp-4],0FFFFFFFFh  
	{
		
	}
	
	__try
00721696  mov         dword ptr [ebp-4],2  
	{
		*p = 4;
0072169D  mov         eax,dword ptr [p]  
007216A0  mov         dword ptr [eax],4  
	}
007216A6  mov         dword ptr [ebp-4],0FFFFFFFFh  
007216AD  jmp         $LN16+0Ah (07216BFh)  
	__except (EXCEPTION_EXECUTE_HANDLER)
007216AF  mov         eax,1  
007216B4  ret  
007216B5  mov         esp,dword ptr [ebp-18h]  
	}
007216B8  mov         dword ptr [ebp-4],0FFFFFFFFh  
	{

	}

	printf("%d", a);
007216BF  mov         eax,dword ptr [a]  
007216C2  push        eax  
007216C3  push        offset string "%d" (0725B30h)  
007216C8  call        _printf (07211FEh)  
007216CD  add         esp,8  
	return 0;
007216D0  xor         eax,eax  
}
007216D2  mov         ecx,dword ptr [ebp-10h]  
007216D5  mov         dword ptr fs:[0],ecx  
007216DC  pop         edi  
007216DD  pop         esi  
007216DE  pop         ebx  
007216DF  mov         esp,ebp  
007216E1  pop         ebp  
007216E2  ret
```
##### scopetable分析: 
动态调试定位到scopetable(0x7265f8)
![](assets/Pasted%20image%2020260611213003.png)
三种不同颜色分别代表三个scopetable。
以第一个scopetable为例
ffffffff代表外层trylevel: -1, 0x00721686指向except过滤表达式块, 0x0072168c指向except块内。
其他同理


##### trylevel分析
`[ebp - 4]`指向的是我们的trylevel
可以看到当进入第一个try块时`00721644  mov         dword ptr [ebp-4],0 `将trylevel设为0，此时try,except块匹配scopetable\[0]
进入内嵌第二个try块时`00721654  mov         dword ptr [ebp-4],1  `指向scopetable\[1]。
同理第三个try块时trylevel为2指向scopetable\[2]




## 遍历SEH异常处理表
知道了上面SEH异常处理的底层原理，我们可以通过内联asm汇编代码手动实现异常处理表遍历,注册和撤销

通过判断next指向地址不为0xffffffff遍历当前SEH表
```c
#include <stdio.h>
#include <windows.h>

void TraverseSEHList()
{
	PEXCEPTION_REGISTRATION_RECORD pSEHList = NULL;

	__asm
	{
		push eax
		mov eax, fs:[0]
		mov pSEHList, eax
		pop eax
	}

	printf("SEH LIST: \r\n");
	for (size_t i = 0; pSEHList != (PEXCEPTION_REGISTRATION_RECORD)0xffffffff; i++)
	{
		printf("Num#%d : Next -> 0x%08x  Handler -> 0x%08x\r\n", i, pSEHList->Next, pSEHList->Handler);
		pSEHList = pSEHList->Next;
	}

}

int main()
{
	TraverseSEHList();

	return 0;
}
```


## 手动注册和撤销SEH异常处理
```c
#include <stdio.h>
#include <windows.h>

EXCEPTION_DISPOSITION NTAPI selfSEH(
	EXCEPTION_RECORD* pExceptionRecord,
	PVOID EstablisherFrame,
	CONTEXT* pContextRecord,
	PVOID DispatcherContext)
{
	if (pExceptionRecord->ExceptionCode == EXCEPTION_ACCESS_VIOLATION)
	{
		printf("捕获到地址访问异常\r\n");
		pContextRecord->Eip += 6;
		return ExceptionContinueExecution;
	}

	return ExceptionContinueSearch;
}

void TraverseSEHList()
{
	PEXCEPTION_REGISTRATION_RECORD pSEHList = NULL;

	__asm
	{
		push eax
		mov eax, fs:[0]
		mov pSEHList, eax
		pop eax
	}

	//printf("SEH LIST: \r\n");
	for (size_t i = 0; pSEHList != (PEXCEPTION_REGISTRATION_RECORD)0xffffffff; i++)
	{
		printf("Num#%d : Next -> 0x%08x  Handler -> 0x%08x\r\n", i, pSEHList->Next, pSEHList->Handler);
		pSEHList = pSEHList->Next;
	}

}

int main()
{
	//初始SEH LIST
	printf("初始SEH LIST: \r\n");
	TraverseSEHList();


	//注册SEHLIST
	printf("注册SEHLIST\r\n");
	EXCEPTION_REGISTRATION_RECORD mySEH = { 0 };
	PEXCEPTION_REGISTRATION_RECORD pMySEH = &mySEH;
	mySEH.Handler = selfSEH;
	__asm
	{
		push eax
		push edx

		mov eax, fs : [0]
		mov edx, [pMySEH]
		mov [edx], eax
		
		mov eax, [pMySEH]
		mov fs : [0], eax

		pop edx
		pop eax
	}
	TraverseSEHList();


	//触发异常
	printf("触发异常\r\n");
	int* p = NULL;
	*p = 1;



	//卸载SEHLIST
	printf("卸载SEHLIST\r\n");
	__asm
	{
		push eax
		push edx

		mov eax, [pMySEH]
		mov edx, [eax]
		mov fs : [0], edx

		pop edx
		pop eax
	}
	TraverseSEHList();

	return 0;
}
```






# x64下的SEH

## 前置知识
与 x86 平台基于栈的动态链式模型不同，**x64 的 SEH 是一个基于静态表的模型**。其核心在于，异常处理所需的全部信息（如函数边界、堆栈布局等）都由编译器预先计算好，并以**`.pdata`**和**`.xdata`**节的形式存储在可执行文件的PE结构中。这种设计从根本上改变了异常发生时的查找与处理流程，使其更高效、更安全。


*   **`RUNTIME_FUNCTION` - 函数的定位器**
    该结构定义在 `.pdata` 节中, 是x64下特有的PE数据目录中的异常处理表，为每一个 **非叶函数（指会调用其他函数或分配栈空间的函数）** 提供了一个条目。
    ```cpp
    typedef struct _RUNTIME_FUNCTION {
        ULONG BeginAddress;   // 函数在内存中的起始 RVA（相对虚拟地址）
        ULONG EndAddress;     // 函数在内存中的结束 RVA
        ULONG UnwindData;     // 指向描述该函数堆栈布局的 UNWIND_INFO 结构的 RVA
    } RUNTIME_FUNCTION, *PRUNTIME_FUNCTION;
    ```
    当一个函数触发异常时，系统通过异常发生时的指令指针（RIP），在 `.pdata` 节中进行**二分查找**（通过遍历`RUNTIME_FUNCTION`数组比较当前RIP的地址是否在`RUNTIME_FUNCTION`的`BeginAddress - EndAddress`范围内），从而迅速定位到它所属的 `RUNTIME_FUNCTION` 条目。

*   **`UNWIND_INFO` - 堆栈展开的说明书**
    该结构体是 `.pdata` 条目的目标，包含了如何回溯（展开）函数调用栈的所有指令。
    ```cpp
    typedef struct _UNWIND_INFO {
        UBYTE Version : 3;        // 版本号，当前必须为1
        UBYTE Flags : 5;          // 指示异常处理程序存在的标志位
        UBYTE SizeOfProlog;       // 函数序言（Prolog）的字节数
        UBYTE CountOfCodes;       // 后续展开代码数组的元素个数
        UBYTE FrameRegister : 4;  // 若使用帧指针，指明是哪个寄存器
        UBYTE FrameOffset : 4;    // 帧指针寄存器相对于 RSP 的缩放偏移
        UNWIND_CODE UnwindCode[1];// 一个变长数组，存储具体的堆栈操作指令
        // ... 随后可选地跟着异常处理程序信息或链式展开信息
    } UNWIND_INFO, *PUNWIND_INFO;
    ```
    其中，`Flags` 字段尤为关键，其可取值如下：
    *   **`UNW_FLAG_EHANDLER` (0x01)**：表示该函数有一个**异常处理程序**（`__except` 块），当查找能处理特定异常的函数时必须被调用。
    *   **`UNW_FLAG_UHANDLER` (0x02)**：表示该函数有一个**终止处理程序**（`__finally` 块），在堆栈展开（Unwind）过程中必须被调用，以确保资源被释放。
    *   **`UNW_FLAG_CHAININFO` (0x04)**：表示当前 `UNWIND_INFO` 是一个“链”的一部分，用于描述更复杂的函数（如由 `/Gy` 编译选项产生的函数）的异常信息。
    *   **取值为 0 时表示没有处理程序**


*   **`UNWIND_CODE` - 回滚操作的原子指令**
    `UnwindCode` 数组中的每个元素都编码了函数序言中执行的某一特定操作（如压栈、分配内存、保存寄存器等）。在展开时，系统会逆向执行这些操作，从而精确地恢复调用前的寄存器和栈状态。常见的操作类型包括 `UWOP_PUSH_NONVOL`（压入非易失性寄存器）、`UWOP_ALLOC_LARGE`（分配大块栈空间）等。
```c
typedef union _UNWIND_CODE {
    struct {
        UBYTE CodeOffset;    // 字节1
        UBYTE UnwindOp : 4;  // 字节2的低4位
        UBYTE OpInfo   : 4;  // 字节2的高4位
    };
    USHORT FrameOffset;      // 用于某些操作码的额外信息
} UNWIND_CODE, *PUNWIND_CODE;
```

- `__C_specific_handler`
当函数的 `UNWIND_INFO` 结构设置了 `UNW_FLAG_EHANDLER` 或 `UNW_FLAG_UHANDLER` 标志时，其 `ExceptionHandler` 成员会指向 `__C_specific_handler` 函数的地址。每当发生异常，系统在确定了目标函数后，便会调用此函数。它接收系统传来的标准参数，是衔接系统底层机制与程序员编写逻辑的关键。



- **SCOPE_TABLE**

```c
typedef struct _SCOPE_TABLE_AMD64 {
    DWORD Count;               // 作用域条目的数量
    struct {
        DWORD BeginAddress;    // 保护区域的起始 RVA  即 try块起始地址
        DWORD EndAddress;      // 保护区域的结束 RVA  即 try结束地址
        DWORD HandlerAddress;  // 异常处理程序（__except 或 __finally）地址
        DWORD JumpTarget;      // 异常展开后的目标跳转地址
    } ScopeRecord[1];          // 可变长数组，存放Count个作用域条目
} SCOPE_TABLE_AMD64, *PSCOPE_TABLE_AMD64;
```

该内部结构的各字段含义如下：

- **`Count`**：`ScopeRecord` 数组的元素个数，即函数内 `__try` 块的数量。
    
- **`ScopeRecord`**：每个条目描述一个 `__try` 块的范围以及对应的处理代码：
    
    - `BeginAddress` & `EndAddress`：该 `__try` 块所保护的代码范围（RVA）。
        
    - `HandlerAddress`：处理代码的地址：
        
        - 对于 `__except`：这是过滤（Filter）函数的地址。
            
        - 对于 `__finally`：这是终止（Finally）处理函数的地址。
            
    - `JumpTarget`：
        
        - 对于 `__except`：执行完处理代码后，程序将跳转到此地址继续执行。
            
        - 对于 `__finally`：此字段通常未使用（或为0）。




###  异常分发与展开的完整流程

当一个异常（如除零错误）在某个函数中被触发，整个处理流程大致如下：

#### 1. 异常捕获与初次分发
异常首先由 CPU 捕获，随后操作系统接管。内核态的 `KiDispatchException` 函数在将异常分发给用户态前，会先**给予内核调试器处理该异常的机会**。如果未被调试器处理，控制权会转到用户态的 `RtlDispatchException` 函数，开始遍历查找合适的异常处理器。

#### 2. 查找与处理
`RtlDispatchException` 函数利用当前 RIP，通过**二分查找**在 `.pdata` 节中找到对应的 `RUNTIME_FUNCTION`，进而得到 `UNWIND_INFO`。如果一个函数标识了 `UNW_FLAG_EHANDLER`，则系统会调用其 **“语言特定处理程序”** （如 `__C_specific_handler`），由它来检查当前的异常类型并决定该函数是否能处理这个异常。

#### 3. 堆栈展开 (Unwinding) - 关键步骤
一旦找到**能够处理该异常的**异常处理程序，为了安全地跳转到该程序的入口，系统必须清理（展开）该异常所在函数**及其之间**所有函数在栈上分配的资源和保存的寄存器。`RtlUnwindEx` 是执行展开操作的核心函数。
在 `RtlUnwindEx` 内部，对于每一个需要展开的函数，都会调用 `RtlVirtualUnwind`。该函数会：
*   根据函数的 `UNWIND_INFO` 中的 `UnwindCode` 数组，模拟出回滚该函数栈帧所需的操作。
*   如果函数有 `UNW_FLAG_UHANDLER`（`__finally` 块），则调用它，确保其内部的清理代码被执行。

#### 4. 异常处理的最终执行
经过上述展开过程后，所有应执行的终止处理程序都已运行。此时，系统已经成功清理了栈空间，并将执行环境恢复到了能够处理该异常的正确状态。最后，系统会将控制权转交给最初找到的异常处理程序（`__except` 块）来执行。

###  语言特定处理程序：连接SEH与C++ EH的桥梁

`UNWIND_INFO` 中引用的**语言特定处理程序**（Language-specific Handler）是构建高级语言（如C++）异常处理机制的关键。对于纯SEH（`__try`/`__except`），这个角色由 `__C_specific_handler` 函数充当。
而在MSVC的C++实现中，异常处理是构建在SEH之上的。编译器将C++的 `try`/`catch` 块转换为对名为 **`__CxxFrameHandler3`** 的语言特定处理程序的调用。该处理程序负责：
1.  检查抛出的异常对象类型是否与当前函数的某个 `catch` 块匹配。
2.  如果匹配，则安排展开堆栈并跳转到该 `catch` 块。
3.  如果不匹配，则返回 `ExceptionContinueSearch`，让系统继续向上层调用者寻找合适的 `catch` 块。

### x64 vs. x86：关键差异速览

与 x86 SEH 的核心差异在于实现模型，下面是其主要区别的总结：

| 特性       | x86 SEH                      | x64 SEH                            |
| :------- | :--------------------------- | :--------------------------------- |
| **核心机制** | **基于栈的动态链表**                 | **基于PE文件的静态表**                     |
| **信息存储** | `FS:[0]` 指向的栈上链表，动态注册        | `.pdata` 和 `.xdata` 节，编译时生成        |
| **查找方式** | 遍历栈上链表，可能为 O(n)              | **二分查找** `.pdata` 表，时间复杂度 O(log n) |
| **安全性**  | 较差，异常处理入口位于栈上，易受**栈缓冲区溢出**攻击 | 安全，异常处理信息位于**只读的PE节**中，难以被篡改       |
| **性能**   | 通常较慢，需要维护链表，异常开销较大           | 通常更快，静态表处理高效，对常态执行路径无影响            |
| **兼容性**  | 适用于所有x86 Windows版本           | 所有非x86架构（包括 x64、ARM64等）均采用此模型      |

### 安全性的大幅提升

x64 SEH 基于静态表的设计，从根本上阻断了一种重要的攻击路径——**SEH 覆盖攻击**。在 x86 系统中，由于异常处理链和注册记录位于栈上，精心构造的缓冲区溢出可以覆盖这些指针，从而劫持程序的执行流程。而在 x64 中，所有关键的异常处理信息都存储在只读的 `.pdata` 和 `.xdata` 节中，攻击者无法通过简单的栈溢出修改它们。





## 定位x64SEH处理

x64下编译源代码
```c
#include <stdio.h>
#include <Windows.h>

int Add(int a, int b)
{
	int c = a + b;
	return c;
}

int main()
{
	

	__try
	{
		int a = 1;

		__try
		{
			int* p = NULL;
			*p = 1;
		}
		__except (EXCEPTION_EXECUTE_HANDLER)
		{
			printf("__except\n");
		}
	}
	__finally
	{
		int a = 2;
		printf("__finally\n");
	}


	int c = Add(1, 2);

	return 0;
}
```

使用我们的Die.exe打开该程序 -> 勾选高级选项 -> PE -> 异常 
![](assets/Pasted%20image%2020260612113442.png)
经过调试知道main函数RVA为0x10a0,  在异常RUNTIME_FUNCTION表中定位到该偏移，记录UnWindInfoAddress RVA: 0x000037c0, 通过内存映射定位到UnWindInfo
![](assets/Pasted%20image%2020260612113715.png)

![](assets/Pasted%20image%2020260612181305.png)
红色标记的是UnWindInfo结构体，绿色标记的是UnWindCode, 棕色标记的是__C_specific_handler, 后面跟着的两个字节是为了对齐。然后蓝色标记的是SCOPE_TABLE结构体，起始四字节:0x2表示有两个ScopeRecord数组成员(用蓝色和粉色标记)。通过ScopeRecord数组记录的偏移我们可以定位到对应的try,except块
```d
int main()
{
00000001400010A0  push        rbp  
00000001400010A2  sub         rsp,80h  
00000001400010A9  lea         rbp,[rsp+20h]  
	

	__try
	{
		int a = 1;
00000001400010AE  mov         dword ptr [rbp],1  

		__try
		{
			int* p = NULL;
00000001400010B5  mov         qword ptr [rbp+8],0  
			*p = 1;
00000001400010BD  mov         rax,qword ptr [rbp+8]  
00000001400010C1  mov         dword ptr [rax],1  
		}
00000001400010C7  jmp         $LN9 (01400010D6h)  
		__except (EXCEPTION_EXECUTE_HANDLER)
		{
			printf("__except\n");
00000001400010C9  lea         rcx,[string "__except\n" (01400031D0h)]  
00000001400010D0  call        printf (0140001110h)  
00000001400010D5  nop  
		}
	}
	__finally
	{
		int a = 2;
00000001400010D6  mov         dword ptr [rbp+10h],2  
		printf("__finally\n");
00000001400010DD  lea         rcx,[string "__finally\n" (01400031E0h)]  
00000001400010E4  call        printf (0140001110h)  
00000001400010E9  nop  
	}

	int c = Add(1, 2);
00000001400010EA  mov         edx,2  
00000001400010EF  mov         ecx,1  
00000001400010F4  call        Add (0140001000h)  
00000001400010F9  mov         dword ptr [c],eax  

	return 0;
00000001400010FC  xor         eax,eax  
}
00000001400010FE  lea         rsp,[rbp+60h]  
0000000140001102  pop         rbp  
0000000140001103  ret
```
例如第一个ScopeRecord成员记录的try 范围 `[10b5 - 10c9]`加上基址0x140000000刚好包含try块，`2a00`偏移指向exception过滤表达式的位置
```d
__except (EXCEPTION_EXECUTE_HANDLER)
0000000140002A00  push        rbp  
0000000140002A02  sub         rsp,20h  
0000000140002A06  lea         rbp,[rdx+20h]  
0000000140002A0A  mov         eax,1  
0000000140002A0F  add         rsp,20h  
0000000140002A13  pop         rbp  
0000000140002A14  ret  
0000000140002A15  int         3
```
`10c9`偏移指向except块内容的起始地址。


## 总结

x64下用户写的try,except,finally异常处理在程序运行遇到异常时，处理机制不是像x86一样基于栈的形式，而是通过静态表。这个静态表通过PE文件中的**exception数据目录** -> 多个**RUNTIME_FUNCTION**结构体, 每个结构体记录对应函数的起始和终止地址，并指向一个**UnWIndInfo**结构体, 这个结构体的**flags**字段记录着是否存在**except块，finally块**。紧跟着的是**UnWIndCode**数组。**UnWindCode**数组后面又跟着指向`__C_specific_handler`的偏移地址。`__C_specific_handler`后面可能会有对齐，然后再跟着**SCOPE_TABLE**结构体，该结构体第一个成员Count记录有多少个**ScopeRecord**数组, 每个**ScopeRecord**数组 记录当前函数每个try块的起始和终止地址，except,finally,except过滤表达式的偏移地址。