---
title: TlsCallBack
published: 2026-06-10
description: TlsCallBack
image: ./cover.jpg
tags: [逆向]
category: Windows逆向
draft: false
---




# TLS

Windows TLS（线程局部存储）是一种让多线程进程中的每个线程都能拥有自己专属、独立数据副本的机制。在多线程环境下，传统的全局和静态变量会被所有线程共享，容易引发数据竞争等问题。TLS正是为了解决这一难题而设计的。

## 静态TLS

**静态TLS**的实现主要由编译器在编译和链接阶段完成，并通过PE文件（可执行文件）的`.tls`节和`IMAGE_TLS_DIRECTORY`结构来承载。

- **使用时，语法非常简洁**，只需在全局或静态变量声明前加上 `__declspec(thread)` 关键字即可。

- **在原理上**，编译器会将这些变量统一放入PE文件中名为 `.tls` 的特殊节。当程序启动或创建新线程时，操作系统会自动为每个线程分配一份独立的TLS数据副本，并将变量的访问地址映射到该副本上。

- 主要特性包括

  ：

  - **高性能**：由于无需API调用开销，运行时速度非常快。
  - **数量限制**：每个进程可用的TLS索引数量有限，最小为64个，最大为1088个。
  - **加载DLL限制**：如果一个包含静态TLS变量的DLL是通过`LoadLibrary`在运行时动态加载的，可能导致程序崩溃，因此通常不建议如此使用。

### 静态TLS实现

```c
#include <stdio.h>
#include <Windows.h>

__declspec(thread) DWORD tlsData1 = 0;
__declspec(thread) DWORD tlsData2 = 1;

DWORD WINAPI workThread(LPVOID lp)
{
	tlsData1++;
	tlsData2++;
	printf("Thread[%d] tlsData1 : %d, tlsData2 : %d\r\n", GetCurrentThreadId(), tlsData1, tlsData2);

	return 0;
}

int main()
{
	HANDLE hThread[3] = { 0 };

	for (size_t i = 0; i < sizeof(hThread) / sizeof(hThread[0]); i++)
	{
		hThread[i] = CreateThread(NULL, 0, workThread, NULL, 0, NULL);
	}

	WaitForMultipleObjects(3, hThread, TRUE, INFINITE);

	for (size_t i = 0; i < sizeof(hThread) / sizeof(hThread[0]); i++)
	{
		CloseHandle(hThread[i]);
	}

	return 0;
}
```

## 动态TLS

**动态TLS**的实现主要依赖于程序员显式调用的Win32 API函数

- **使用步骤包括四个核心的API函数：

  1. **`TlsAlloc`**：分配TLS索引。该函数会返回一个可用的TLS索引，供后续操作使用。
  2. **`TlsSetValue`**：存储数据。将指定数据存储到当前线程的TLS槽中。
  3. **`TlsGetValue`**：获取数据。从当前线程的TLS槽中读取指定数据。
  4. **`TlsFree`**：释放索引。不再需要时，释放之前分配的TLS索引。

- **在原理上**，进程维护一个位标志数组来追踪TLS索引的使用情况。每个线程则拥有一个槽（Slot）数组，由`TlsSetValue`和`TlsGetValue`操作。使用`TlsAlloc`分配的索引，是作为访问各个线程独立槽数组的统一“钥匙”。

- 主要特性包括

  ：

  - **灵活性高**：可在运行时随时创建和销毁，尤其适合DLL中使用。
  - **数量更多**：实际可用的索引数量远超`TLS_MINIMUM_AVAILABLE`的限制。
  - **性能开销**：相比静态TLS，每次访问都需API调用，因此性能开销稍大，但在现代硬件上影响通常不大。

### 动态TLS实现

```c
#include <stdio.h>
#include <Windows.h>

DWORD g_TlsIndex = 0;

DWORD WINAPI WorkThread(LPVOID lp)
{
	CHAR* pData = (CHAR*)LocalAlloc(0, 0xff);

	sprintf_s(pData, 0xff, "ThreadId: %d", GetCurrentThreadId());
	
	TlsSetValue(g_TlsIndex, pData);

	CHAR* pGetData = (CHAR*)TlsGetValue(g_TlsIndex);

	printf("%s\r\n", pGetData);
	
	LocalFree(pData);
	return 0;
}

int main()
{
	HANDLE hThread[3] = { 0 };
	g_TlsIndex = TlsAlloc();

	for (size_t i = 0; i < 3; i++)
	{
		hThread[i] = CreateThread(NULL, 0, WorkThread, NULL, 0, NULL);
	}

	WaitForMultipleObjects(3, hThread, TRUE, INFINITE);

	for (size_t i = 0; i < 3; i++)
	{
		CloseHandle(hThread[i]);
	}

	TlsFree(g_TlsIndex);


	return 0;
}
```

| 特性           | 动态 TLS (Dynamic TLS)                  | 静态 TLS (Static TLS)                |
| -------------- | --------------------------------------- | ------------------------------------ |
| **使用方式**   | 手动调用 Win32 API                      | 使用 `__declspec(thread)` 关键字声明 |
| **实现层级**   | 应用层                                  | 编译器 + 链接器                      |
| **性能**       | 稍低（API调用开销）                     | 高（无额外函数调用）                 |
| **灵活性**     | 高，可在运行时动态管理                  | 低，编译时已确定                     |
| **DLL兼容性**  | 完美兼容隐式和显式加载                  | 显式加载可能引发问题                 |
| **代码复杂度** | 较高，需管理索引和指针                  | 极低，与普通变量类似                 |
| **适用场景**   | 需要动态、灵活管理的场景，尤其适用于DLL | 追求极致性能的轻量级、简单场景       |

## TLS实现底层原理

Windows TLS（线程局部存储）的核心是通过**段寄存器**（x86 的 `FS`，x64 的 `GS`）快速访问**线程环境块（TEB）**，再经由 TEB 中的指针数组或编译器生成的 `__tls_index` 取得线程私有数据。

| 层次     | 静态 TLS                                                     | 动态 TLS                                    |
| -------- | ------------------------------------------------------------ | ------------------------------------------- |
| 编译器   | 将变量放入 `.tls` 段，生成全局 `_tls_index` 和偏移           | 无特殊处理，调用 API                        |
| 链接器   | 生成 TLS 目录表，合并所有 TLS 段                             | 无                                          |
| 加载器   | 解析 TLS 目录，为每个模块分配索引，预留空间                  | 初始化 TEB 中的 `TlsSlots` 数组为空         |
| 线程创建 | 为每个线程的静态 TLS 区域分配内存，初始化数据，并将基址填入 `TlsSlots[模块索引]` | 不分配                                      |
| 访问机制 | `TEB->TlsSlots[模块索引] + 偏移`                             | `TEB->TlsSlots[用户索引]`（用户存储的指针） |
| 汇编代码 | 3~4 条内存访问指令                                           | 2~3 条内存访问指令                          |

### 静态tls

静态 TLS 的起点在于 PE 文件中的一个特殊目录——TLS 目录。它像一份蓝图，告诉操作系统该程序需要 TLS 支持。

这个目录是 PE 可选头中数据目录数组的第 10 个成员（索引为 9），它指向一个 `IMAGE_TLS_DIRECTORY` 结构体。该结构体是 TLS 初始化的核心信息源，其关键字段包括：

- **`StartAddressOfRawData` 与 `EndAddressOfRawData`**：这两个字段定义了 TLS 模板（`.tls` 节）在文件中的地址范围。模板中包含了所有 `__declspec(thread)` 变量的初始值。
- **`AddressOfIndex`**：指向一个 `DWORD` 类型的 TLS 索引。这个索引由加载器在程序启动时填充，是访问线程私有数据的“钥匙”。
- **`AddressOfCallBacks`**：指向一个以 `NULL` 结尾的 TLS 回调函数指针数组。这是实现 TLS 回调机制的关键。

[Vergilius Project | TEB](https://www.vergiliusproject.com/kernels/x86/windows-10/22h2/_TEB)`fs : 0x2c`指向**ThreadLocalStoragePointer**，ThreadLocalStoragePointer指向TlsSlots数组地址

依据上面Tls静态实现汇编代码分析

```c
tlsData1++;
; 取值并加1
004115C9  mov         eax,dword ptr [_tls_index (04171CCh)]   ;tls_index默认为0
004115CE  mov         ecx,dword ptr fs:[2Ch]   ;获取ThreadLocalStoragePointer
004115D5  mov         edx,dword ptr [ecx+eax*4]  ;TlsSlots[首地址 + tls_index]
004115D8  mov         eax,dword ptr [edx+108h]  ;0x108是目标数据在tls节区的偏移，在编译阶段确定
004115DE  add         eax,1  
; 赋值操作
004115E1  mov         ecx,dword ptr [_tls_index (04171CCh)]  
004115E7  mov         edx,dword ptr fs:[2Ch]  
004115EE  mov         ecx,dword ptr [edx+ecx*4]  
004115F1  mov         dword ptr [ecx+108h],eax  
;tlsData2++和上面一样
	tlsData2++;
004115F7  mov         eax,dword ptr [_tls_index (04171CCh)]  
004115FC  mov         ecx,dword ptr fs:[2Ch]  
00411603  mov         edx,dword ptr [ecx+eax*4]  
00411606  mov         eax,dword ptr [edx+104h]  ;与上面唯一不同点,在.tls节区偏移不同,指向tlsData2
0041160C  add         eax,1  
0041160F  mov         ecx,dword ptr [_tls_index (04171CCh)]  
00411615  mov         edx,dword ptr fs:[2Ch]  
0041161C  mov         ecx,dword ptr [edx+ecx*4]  
0041161F  mov         dword ptr [ecx+104h],eax
```

### 动态tls

动态 TLS 使用 Win32 API：`TlsAlloc`、`TlsSetValue`、`TlsGetValue`、`TlsFree`。因为每个线程的**ThreadLocalStoragePointer**值是不同的，所以每个线程拥有不同的**TlsSlots**，**TlsAlloc**为每个线程分配统一的**tls索引**(一个变量的tls索引值是唯一的)。**TlsSetValue和TlsGetValue**本质上通过`TlsSlots[TlsAlloc分配的索引]`来获取和写入值。`TlsSlots[index]`的值是一个指针，这个指针指向我们存储的数据。

# TLS CallBack回调

## 需要tlscallback的原因

线程局部存储（TLS）允许每个线程拥有自己的独立数据副本，从而避免了多线程访问全局变量时的同步问题。TLS本身用 `__declspec(thread)` 等机制声明，但系统仅负责分配存储空间，却不提供机制来自动清理线程退出时这些复杂数据对象占用的资源（如动态分配的内存、文件句柄等）。

这就产生了一个关键问题：**如果线程使用了一个自定义类对象作为TLS变量，当线程退出时，这个对象的析构函数如何自动被调用？**

答案就是：系统做不到。这种缺乏自动清理机制的问题，正是 TLS 回调要解决的核心问题

## tlscallback如何工作

为了让资源清理得以实现，开发者可以**注册一个或多个 TLS 回调函数**。这些函数的签名与 `DllMain` 完全相同，由系统在特定时机自动调用。

当程序中注册了 TLS 回调函数后，Windows 加载器（Loader）会在其执行流程中，根据事件的类型（Reason），自动调用这些已注册的回调函数。

| 调用原因 (Reason)        | 原生功能描述                                                 |
| ------------------------ | ------------------------------------------------------------ |
| `DLL_PROCESS_ATTACH` (1) | **主线程TLS数据初始化**：在程序入口点（OEP）执行之前，调用此回调来初始化进程主线程的TLS数据。 |
| `DLL_THREAD_ATTACH` (2)  | **新线程TLS数据初始化**：每当进程创建一个新线程时，在新线程开始执行其代码前被调用，用于初始化该线程独有的TLS数据副本。 |
| `DLL_THREAD_DETACH` (3)  | **线程TLS数据清理**：当一个线程正常退出时被调用，线程可以在这个回调里释放其TLS数据占用的资源（比如释放动态分配的内存）。 |
| `DLL_PROCESS_DETACH` (0) | **进程TLS数据清理**：在进程退出时被调用，用于执行最终的、进程级别的清理工作。 |

尽管TLS回调的设计初衷是管理线程资源，但它凭借**在程序入口点（OEP）之前执行**的特性，成为了一个极为有效的反调试技术。

当使用调试器加载程序时，通常会在程序入口点中断。但如果程序注册了 TLS 回调，恶意代码就能在调试器拦截到控制权之前抢先运行，比如检测调试器环境并执行自毁逻辑，从而增加分析难度

## tlscallback参数reason的深入解析

- **与 `DllMain` 的高度相似性**：TLS 回调函数与 `DllMain` 函数的签名和 `Reason` 参数的宏定义完全相同，但 `DllMain` 是动态链接库（DLL）的入口点，在 DLL 被加载和卸载时执行；而 TLS 回调是**可执行文件（.exe）** 的一部分。
- **触发时机与动态加载/卸载**：一个关键区别在于**动态加载/卸载**的处理方式。当一个 DLL 通过 `LoadLibrary` 加载时，系统只会为其主线程调用 `DLL_THREAD_ATTACH`，而不会为进程中已存在的其他线程调用。相应地，通过 `FreeLibrary` 卸载 DLL 时，也只对当前正在执行的线程调用 `DLL_THREAD_DETACH`。这意味着在这种动态加载/卸载的场景下，`DLL_THREAD_ATTACH` 和 `DLL_THREAD_DETACH` 的触发具有**线程局部性**。因此，在 TLS 回调中，**你不应该假设针对某个 Reason 的调用对进程内的所有线程都会发生**。
- **⭐ 不要依赖 `DLL_PROCESS_DETACH`**：在 TLS 回调（和 `DllMain`）中，需要重点注意：当 `Reason` 为 `DLL_PROCESS_DETACH` 时，能执行的系统函数非常有限。因为此时进程正在销毁，CRT（C运行时库）可能已被清理，许多系统 API 调用会失败。**典型做法是尽量避免在这个阶段执行任何重要的清理工作。**

## tlscallback实现

### 1.告知链接器，在PE文件中包含TLS目录

x64: #pragma comment(linker, "/INCLUDE:_tls_used")
 x86: #pragma comment(linker, "/INCLUDE:__tls_used")
 区别在于tls前面_下划线的数量, x86: 2个下划线, x64: 1个下划线。

```c
#ifdef _WIN64
	#pragma comment(linker, "/INCLUDE:_tls_used")
#else
	#pragma comment(linker, "/INCLUDE:__tls_used")
#endif
```

### 2.实现tlscallback

```c
VOID NTAPI tlsCallBack1(PVOID dllHandle, DWORD reason, PVOID Reserved)
{
	//MessageBox(NULL, L"Hello,World!", L"TlsCallBack", 0);
	if (IsDebuggerPresent())
	{
		MessageBox(NULL, L"debug detect", L"debug detect", 0);
		ExitProcess(0);
	}
	printf("TlsCallbacks触发: ");
	switch (reason) {
	case DLL_PROCESS_ATTACH:
		printf("TLS: 进程启动\r\n");
		break;
	case DLL_THREAD_ATTACH:
		printf("TLS: 线程启动\r\n");
		break;
	case DLL_THREAD_DETACH:
		printf("TLS: 线程退出\r\n");
		break;
	case DLL_PROCESS_DETACH:
		printf("TLS: 进程退出\r\n");
		break;
	}
}
```

### 3.将tlscallback放入节区中

```c
//方法1
//#pragma data_seg(".CRT$XLB")
//	PIMAGE_TLS_CALLBACK pTls_CallBacks[] = { tlsCallBack1, 0 };
//#pragma data_seg()

// 方法2
#pragma section(".CRT$XLB", read)
__declspec(allocate(".CRT$XLB"))
PIMAGE_TLS_CALLBACK pTlsCallBack = tlsCallBack1;
```

我在x86下使用方法1和方法2都可以，但是在x64下只能使用方法2
 tlscallback函数指针存放节区需要在`.CRT$XLA - .CRT$XLZ`中，因为该节区范围是加载器寻找TLS回调数组的核心标准，是PE文件中TLS目录的`AddressOfCallBacks` 指向的地址。如果放在其他节区中，那么加载器在遍历该TLS回调数组时就无法找到我们自己写的Tlscallback函数了。

- **二进制层面的唯一标准**：在任何方法中，最终都需要构造出 **`AddressOfCallBacks`** 字段，该字段指向一个以**0结尾的指针数组**。
- **数组必须以0结尾**：系统加载器会**遍历这个数组**，直到遇到一个**空指针（NULL）** 才停止。因此，在声明`pTLS_CALLBACKs`数组时，**务必显式地在末尾加上`0`或`NULL`**，否则会导致系统访问越界内存，引发崩溃。
- **慎用`.tls`节区**：`.tls`节区是TLS**数据的模板**，存放的是所有线程共有的初始化数据副本，而回调函数指针数组是**指令**。将它们分开存放，能避免权限混淆可能引发的意外。

.CRT节区本身也在.rdata节区当中，如果没有加载符号信息在ida等工具中是不能直接看到.CRT的

### 完整代码和分析

```c
// 1.告诉链接器在PE包含tls目录
// x86下采用双划线, x64下采用单划线
#ifdef _WIN64
	#pragma comment(linker, "/INCLUDE:_tls_used")
#else
	#pragma comment(linker, "/INCLUDE:__tls_used")
#endif

// 2.实现tlscallback
VOID NTAPI tlsCallBack1(PVOID dllHandle, DWORD reason, PVOID Reserved)
{
	//MessageBox(NULL, L"Hello,World!", L"TlsCallBack", 0);
	if (IsDebuggerPresent())
	{
		MessageBox(NULL, L"debug detect", L"debug detect", 0);
		ExitProcess(0);
	}
	printf("TlsCallbacks触发: ");
	switch (reason) {
	case DLL_PROCESS_ATTACH:
		printf("TLS: 进程启动\r\n");
		break;
	case DLL_THREAD_ATTACH:
		printf("TLS: 线程启动\r\n");
		break;
	case DLL_THREAD_DETACH:
		printf("TLS: 线程退出\r\n");
		break;
	case DLL_PROCESS_DETACH:
		printf("TLS: 进程退出\r\n");
		break;
	}
}

// 3.注册tls
//方法1
//#pragma data_seg(".CRT$XLB")
//	PIMAGE_TLS_CALLBACK pTls_CallBacks[] = { tlsCallBack1, 0 };
//#pragma data_seg()

// 方法2
#pragma section(".CRT$XLB", read)
__declspec(allocate(".CRT$XLB"))
PIMAGE_TLS_CALLBACK pTlsCallBack = tlsCallBack1;
	

// 线程函数
DWORD WINAPI ThreadWork(LPVOID lp)
{
	printf("Thread Work\r\n");
	return 0;
}

int main()
{
	printf("Main函数启动\r\n");

	HANDLE hThread = CreateThread(NULL, 0, ThreadWork, NULL, 0, NULL);
	WaitForSingleObject(hThread, INFINITE);
	CloseHandle(hThread);

	system("pause");
	return 0;
}
```

程序输出结果:

```
TlsCallbacks触发: TLS: 进程启动
Main函数启动
TlsCallbacks触发: TLS: 线程启动
Thread Work
TlsCallbacks触发: TLS: 线程退出
```

最后没有输出TLS 进程退出的可能原因在上面tlscallback参数reason深入解析有提到

```
**⭐ 不要依赖 `DLL_PROCESS_DETACH`**：在 TLS 回调（和 `DllMain`）中，需要重点注意：当 `Reason` 为 `DLL_PROCESS_DETACH` 时，能执行的系统函数非常有限。因为此时进程正在销毁，CRT（C运行时库）可能已被清理，许多系统 API 调用会失败。**典型做法是尽量避免在这个阶段执行任何重要的清理工作。**
```