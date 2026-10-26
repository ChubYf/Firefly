---
title: unidbg基本介绍
published: 2026-09-27
description: unidbg基本介绍
image: ./cover.jpg
tags: [逆向]
category: unidbg
draft: false
---

参考和学习文章：
[unidbg基本介绍](https://www.yuque.com/lilac-2hqvv/xdwlsg/idgio0?singleDoc#%20%E3%80%8AUnidbg%20%E7%9A%84%E5%9F%BA%E6%9C%AC%E4%BB%8B%E7%BB%8D%E3%80%8B)
[unidbg使用场景](https://www.yuque.com/lilac-2hqvv/xdwlsg/gri3ins5e22b5x7o?#%20%E3%80%8AUnidbg%20%E7%9A%84%E4%BD%BF%E7%94%A8%E5%9C%BA%E6%99%AF%E3%80%8B)
[创建模拟器和加载模块](https://www.yuque.com/lilac-2hqvv/lfssh8/ibfh81?#kGhCS)
[unidbg日志系统](https://www.yuque.com/docs/share/a0335319-e3dd-443c-b00e-b2af39ae7570?#%20%E3%80%8AUnidbg%20%E7%9A%84%E6%97%A5%E5%BF%97%E7%B3%BB%E7%BB%9F%E3%80%8B  )


# 基本框架
unidbg 是一个基于 Java 的开源 **ARM 模拟执行框架**，核心目标是让开发者无需物理设备或完整系统模拟器，就能在桌面环境中加载并执行 Android 或 iOS 的原生库（`.so`/`.dylib`），并进行逆向分析与调试。

### 🏗️ 模块架构：清晰的 Maven 多模块分层

unidbg 采用 Maven 多模块项目结构，通过分层设计实现了核心抽象、平台实现和后端引擎的解耦。

| 模块 | Artifact ID | 核心职责 |
| :--- | :--- | :--- |
| **核心 API** | `unidbg-api` | 定义平台无关的抽象接口，如模拟器基类、内存管理、后端抽象、调试工具等。所有其他模块都依赖于此。 |
| **Android 平台** | `unidbg-android` | 实现 Android 特定模拟，包括 ELF 文件加载、DalvikVM（用于 JNI 模拟）、Android 系统调用处理。 |
| **iOS 平台** | `unidbg-ios` | 实现 iOS/Darwin 模拟，包括 Mach-O 文件加载、Objective-C 与 Swift 运行时模拟、Darwin 系统调用。 |
| **Unicorn2 后端** | `unidbg-unicorn2` | 基于 Unicorn 引擎的**默认后端**，提供全面的调试功能，如 GDB stub、指令追踪、内存读写追踪。 |
| **Dynarmic 后端** | `unidbg-dynarmic` | 基于 Dynarmic 的 **JIT 编译后端**，提供更快的执行速度。 |
| **KVM 后端** | `unidbg-kvm` | Linux 专属，利用 KVM 实现 **ARM64 硬件加速**模拟。 |
| **Hypervisor 后端** | `unidbg-hypervisor` | macOS 专属，利用 Apple Hypervisor.framework 在 Apple Silicon（M1/M2）上实现**硬件加速**。 |

### ⚙️ 核心架构：三层继承与可插拔后端

核心架构围绕**三层继承体系**和**后端抽象层**展开，确保了框架的灵活性与可扩展性。

1.  **抽象模拟器（`AbstractEmulator`）**：位于最顶层，提供平台无关的基础设施。它持有 `Backend` 实例来委托所有 CPU 操作，并负责协调线程调度、内存文件系统、寄存器上下文等组件，同时提供代码、内存读写的追踪钩子。
2.  **抽象 ARM 模拟器（`AbstractARMEmulator` / `AbstractARM64Emulator`）**：位于中间层，提供 ARM 架构特有的功能。例如，管理 Capstone 反汇编器以支持 ARM/Thumb（32位）或 AArch64（64位）指令，并在链接寄存器地址处设置陷阱页以捕获函数返回。
3.  **平台特定模拟器**：位于最底层，如 `AndroidEmulator` 和 `IOSEmulator`，它们将上述抽象层与具体平台的实现（如 ELF/Mach-O 加载器、系统调用处理）相结合。

**后端抽象层**是核心架构的关键，它允许 `Backend` 接口有不同的实现。模拟器的执行、内存访问、寄存器读写等操作都通过该接口委托给具体的后端引擎（如 Unicorn、Dynarmic）完成，从而实现执行引擎的“可插拔”。

### 📱 平台模块的核心职责

*   **Android 模块**：核心是 `AndroidElfLoader`，负责加载 ELF 文件、解析符号、管理依赖。同时，`DalvikVM` 模拟了 JNI 调用环境，使得 `JNI_OnLoad` 等函数可以被调用，并支持通过 `createDalvikVM(apkFile)` 传入 APK 文件以利用部分签名校验信息。
*   **iOS 模块**：核心是 `MachOLoader`，负责加载 Mach-O 二进制文件（包括 FAT 二进制）。它集成了 Objective-C 和 Swift 运行时模拟，需要显式激活才能正确处理 iOS 应用的运行时行为。

### 🔧 调试与 Hook 支持

unidbg 提供了丰富的调试手段。默认的 Unicorn 后端支持控制台调试器、GDB stub 以及指令和内存读写追踪。在 Hook 方面，框架支持多种方式：
*   **Inline Hook**：借助 **Dobby** 库，通过重写指令序列来重定向执行流，可以 Hook 任意函数。
*   **导入 Hook**：在 Android 平台上，通过 **xHook** 修改 ELF 的 GOT/PLT 表来拦截库导入。iOS 平台则支持 **fishhook**、**substrate** 和 **whale**。

### 💻 使用示例：加载并调用 SO 函数

一个典型的使用流程通常包含以下步骤：

1.  **创建模拟器**：选择架构并初始化，例如 `AndroidEmulator emulator = new AndroidARMEmulator();`。
2.  **创建 DalvikVM**：`DalvikVM vm = emulator.createDalvikVM();`。
3.  **加载 SO 文件**：`DalvikModule dm = vm.loadLibrary(new File("libtarget.so"), true);`，第二个参数 `true` 表示自动调用 `JNI_OnLoad`。
4.  **获取模块与函数地址**：通过 `dm.getModule()` 获取模块，再通过符号名或偏移地址找到目标函数。
5.  **调用函数**：使用 `emulator.eFunc` 或 `module.callFunction` 等方式，传入必要参数进行调用。

如果需要处理 JNI 调用，可以通过继承 `AbstractJni` 并设置 `vm.setJni(yourJniImplementation)` 来拦截和补全 SO 对 Java 层的调用。

总的来说，unidbg 的框架设计清晰，通过分层架构、可插拔后端和丰富的平台模拟能力，为移动原生代码的分析提供了一个强大且灵活的基础。



# unidbg中的一些函数
## AndroidEmulator相关函数
`AndroidEmulator` 是 unidbg 框架中用于模拟 Android 运行环境的核心接口。它作为协调枢纽，负责管理内存、虚拟机、后端引擎等核心组件，是进行 SO 文件模拟执行和逆向分析的入口。

### 🏗️ 创建模拟器：AndroidEmulatorBuilder

在 unidbg 中，`AndroidEmulator` 实例通过构建器 `AndroidEmulatorBuilder` 创建，以灵活配置模拟器的各项参数。

| 构建器方法 | 描述 |
| :--- | :--- |
| `for32Bit()` / `for64Bit()` | 指定模拟器的位数，必须与目标 SO 文件的架构匹配。 |
| `setProcessName(String)` | 设置模拟进程名，有助于规避 SO 中针对进程名的反调试检测。 |
| `addBackendFactory(BackendFactory)` | 指定后端引擎，如 `Unicorn2Factory`（默认，调试功能强）或 `DynarmicFactory`（执行速度快）。 |
| `setRootDir(File)` | 设置虚拟根目录，实现文件 I/O 重定向，例如将 `/data/data/` 映射到本地目录。 |
| `build()` | 根据配置创建并返回 `AndroidEmulator` 实例。 |

**创建示例：**
```java
// 创建 64 位模拟器，设置进程名，并指定使用 Dynarmic 后端以加速
AndroidEmulator emulator = AndroidEmulatorBuilder.for64Bit()
        .setProcessName("com.example.app")
        .addBackendFactory(new DynarmicFactory(true))
        .build();
```


### 🧠 核心功能函数解析

`AndroidEmulator` 提供了一系列方法，用于管理模拟器的内存、虚拟机、库加载和调试。

#### 内存与系统库管理
*   **`getMemory()`**：获取内存操作接口 `Memory`，用于内存分配、映射和查询。设置 `LibraryResolver` 以提供系统库支持是常见操作。
    ```java
    Memory memory = emulator.getMemory();
    memory.setLibraryResolver(new AndroidResolver(23)); // 支持 Android SDK 23
    ```
    
*   **`getPid()`**：获取模拟进程的 PID。
*   **`getProcessName()`**：获取模拟进程名。

#### 虚拟机 (DalvikVM) 与 JNI
*   **`createDalvikVM()` / `createDalvikVM(File apkFile)`**：创建 Dalvik 虚拟机实例。传入 APK 文件路径可让 unidbg 自动解析包名、签名等信息，简化 JNI 环境补全。
    ```java
    VM vm = emulator.createDalvikVM(new File("target.apk"));
    vm.setJni(this); // this 需实现 JNI 接口或继承 AbstractJni
    vm.setVerbose(true); // 开启详细日志便于调试
    ```
    
*   **`getDalvikVM()`**：获取已创建的 Dalvik 虚拟机实例。

#### 库加载与函数调用
*   **`loadLibrary(File file, boolean forceCallInit)`**：加载 SO 文件（ELF 格式）。`forceCallInit` 参数决定是否自动调用 `JNI_OnLoad`。
    ```java
    DalvikModule dm = vm.loadLibrary(new File("libtarget.so"), true);
    Module module = dm.getModule(); // 获取 Module 对象用于后续操作
    ```
    
*   **`eFunc`** / **`module.callFunction`**：调用 SO 中导出的函数。`eFunc` 是 `emulator` 上的便捷方法，`module.callFunction` 需要传入模拟器对象、函数偏移地址和参数数组。

#### 调试与追踪
*   **`showRegs()`**：显示当前所有寄存器的状态，便于分析执行现场。
*   **`getBackend()`**：获取底层 CPU 后端对象。通过它可以进行更底层的操作，如注册代码钩子 (`hook_add_new`) 或直接读写寄存器。
*   **`getContext()`**：获取寄存器上下文 `RegisterContext`，用于读写寄存器值。
*   **`traceRead(long begin, long end)`**：追踪指定内存范围的读操作。
*   **`traceWrite(long begin, long end)`**：追踪指定内存范围的写操作。
*   **`traceCode(long begin, long end)`**：追踪指定地址范围的指令执行。

#### 状态查询与内存操作
*   **`isRunning()`**：判断模拟器是否正在运行。
*   **`getStackPoint()`**：获取当前栈指针的值。
*   **`pointer(long address)`**：获取指向指定内存地址的 `UnidbgPointer`，方便进行内存读写。
*   **`getMemoryMap()`**：获取当前内存的映射情况。
*   **`findModule(String moduleName)`**：根据模块名查找已加载的模块。
*   **`findModuleByAddress(long address)`**：根据地址查找所属模块。

### 🔗 相关组件速查

| 组件 | 描述 |
| :--- | :--- |
| **`AndroidResolver`** | 系统库解析器，提供不同 Android SDK 版本（如 19, 23）的系统库支持。 |
| **`DalvikModule`** | 代表一个已加载的 SO 库，通过 `vm.loadLibrary()` 返回。 |
| **`Module`** | 代表一个已加载的模块，提供 `callFunction` 等方法用于调用函数。 |
| **`VM`** | Dalvik 虚拟机接口，负责管理 JNI 环境、对象和类。 |
| **`AbstractJni`** | JNI 回调处理器的基类，通过继承并实现相应方法来补全 Java 环境。 |
| **`Backend`** | CPU 后端抽象接口，所有与 CPU 模拟相关的操作都通过它进行。 |

### 💡 使用建议
*   **架构匹配**：务必根据目标 SO 文件的架构（32/64位）选择对应的构建器方法。
*   **后端选择**：`Unicorn2` 后端调试功能完善，适合分析；`Dynarmic` 后端速度更快，适合批量执行。
*   **APK 文件**：建议传入目标 APK 文件以创建 `DalvikVM`，这能自动补全包名、签名等环境信息，减少手动工作量。
*   **JNI 环境补全**：复杂的 SO 会大量调用 Java 层方法，需继承 `AbstractJni` 并实现对应回调，这是模拟成功的关键。




## Memory相关函数
`Memory` 接口是 unidbg 内存管理的核心，它封装了内存分配、读写、映射查询等底层操作，为模拟执行提供了统一的 API。

### 🏗️ 初始化与配置

在创建模拟器后，通过 `emulator.getMemory()` 获取 `Memory` 实例，并通常需要设置库解析器以支持系统调用。

```java
AndroidEmulator emulator = AndroidEmulatorBuilder.for64Bit().build();
Memory memory = emulator.getMemory();
memory.setLibraryResolver(new AndroidResolver(23)); // 支持 Android 6.0 (SDK 23)
```

### 📦 内存分配与释放

`Memory` 提供了多种分配方式，均返回 `UnidbgPointer` 以便后续操作。

| 方法 | 返回类型 | 描述 |
| :--- | :--- | :--- |
| **`malloc(int size, boolean runtime)`** | `UnidbgPointer` | 分配指定大小的堆内存。`runtime` 参数通常设为 `false`。 |
| **`allocatestack(int size)`** | `UnidbgPointer` | 在栈上分配指定大小的内存空间。 |
| **`mmap(int length, int prot)`** | `UnidbgPointer` | 映射一块具有指定保护属性（读/写/执行）的内存区域。 |
| **`munmap(long address, int size)`** | `void` | 解除指定地址的内存映射。 |

**`malloc` 使用示例：**
```java
// 分配 0x10 字节的内存
MemoryBlock block = memory.malloc(0x10, false);
UnidbgPointer ptr = block.getPointer();
// 使用完毕后需释放
block.free();
```

### ✍️ 内存读写

通过 `UnidbgPointer` 对象可以方便地读写各种类型的数据。

| 方法 | 描述 |
| :--- | :--- |
| **`ptr.getByte()` / `ptr.setByte(byte)`** | 读写 1 字节 |
| **`ptr.getShort()` / `ptr.setShort(short)`** | 读写 2 字节 |
| **`ptr.getInt()` / `ptr.setInt(int)`** | 读写 4 字节 |
| **`ptr.getLong()` / `ptr.setLong(long)`** | 读写 8 字节 |
| **`ptr.getString()` / `ptr.setString(String)`** | 读写字符串 |
| **`ptr.getByteArray(offset, length)`** | 读取字节数组 |
| **`ptr.write(byte[])`** | 写入字节数组 |

**`Memory` 接口本身也提供了一些便捷方法：**
| 方法 | 描述 |
| :--- | :--- |
| **`writestackstring(String value)`** | 将字符串写入栈内存中。 |
| **`writestackBytes(byte[] value)`** | 将字节数组写入栈内存中。 |
| **`read_memory(long addr, int size)`** | 读取原始内存字节。 |
| **`write_memory(long addr, byte[] data)`** | 写入原始内存字节。 |
| **`read_typed(long addr, Class<T> type)`** | 以指定类型（如 int, float）读取内存。 |

### 🔗 指针获取与操作

`UnidbgPointer` 是操作内存的核心工具，通过 `Memory.pointer(long address)` 获取。

```java
UnidbgPointer ptr = memory.pointer(0x12345678);
int value = ptr.getInt(0); // 读取偏移 0 处的 int
ptr.setInt(4, 0xdeadbeef); // 在偏移 4 处写入 int
```

`UnidbgPointer` 还支持指针运算，例如 `ptr.add(offset)` 返回新的指针，方便遍历内存。

### 🗺️ 内存映射查询

可以查询当前模拟器的内存映射情况，用于调试或逆向分析。

| 方法 | 返回类型 | 描述 |
| :--- | :--- | :--- |
| **`getMemoryMap()`** | `Collection<MemoryMap>` | 获取所有内存区域的映射信息。 |
| **`findModule(String moduleName)`** | `Module` | 根据模块名查找已加载的模块。 |
| **`findModuleByAddress(long address)`** | `Module` | 根据地址查找所属的模块。 |
| **`getStackPoint()`** | `long` | 获取当前栈指针的值。 |

`MemoryMap` 对象包含 `base`（起始地址）、`size`（大小）和 `prot`（保护标志）等信息。

### 🔧 其他实用功能

| 方法 | 描述 |
| :--- | :--- |
| **`setLibraryResolver(AndroidResolver resolver)`** | 设置 Android SDK 版本解析器。 |
| **`traceRead(long begin, long end)`** | 追踪指定内存范围的读操作。 |
| **`traceWrite(long begin, long end)`** | 追踪指定内存范围的写操作。 |

### 💡 使用建议

- **及时释放**：通过 `malloc` 分配的内存，在使用完毕后应调用 `free()` 释放，避免内存泄漏。
- **内存对齐**：分配内存时，unidbg 会自动处理对齐，但手动计算偏移时需注意数据类型的大小。
- **调试追踪**：利用 `traceRead`/`traceWrite` 可以快速定位内存访问异常。
- **指针运算**：`UnidbgPointer` 的指针运算功能强大，但需确保偏移量在合法范围内，避免越界访问。




## VM相关函数
`VM`（Dalvik虚拟机）是unidbg中实现JNI环境模拟的核心，它让Native代码（SO库）能够与Java层对象和方法进行交互。其架构主要分为三层：**VM层**（`DalvikVM`/`DalvikVM64`，负责注册JNI函数表）、**对象管理层**（跟踪Java对象的引用和生命周期）和**JNI实现层**（可插拔的`AbstractJni`或`ProxyJni`，提供具体的Java语义）。

### 🏗️ VM的创建与初始化

`VM`实例通过`AndroidEmulator`创建，而非直接实例化。

| 方法 | 描述 |
| :--- | :--- |
| **`createDalvikVM()`** | 创建一个基础的Dalvik虚拟机实例。 |
| **`createDalvikVM(File apkFile)`** | 创建虚拟机并指定APK文件。传入APK可以自动解析包名、版本、签名等信息，**极大地减少了手动补环境的工作量**。 |
| **`getDalvikVM()`** | 获取当前模拟器中已创建的VM实例。 |

**创建示例：**
```java
// 推荐方式：传入APK文件，让unidbg自动完成部分环境初始化
VM vm = emulator.createDalvikVM(new File("target.apk"));

// 或者，不指定APK文件
VM vm = emulator.createDalvikVM();
```

### 🔌 JNI环境设置与调试

创建VM后，需要配置JNI回调处理并可选地开启调试日志。

| 方法 | 描述 |
| :--- | :--- |
| **`setJni(Jni jni)`** | **核心方法**。设置JNI调用处理器。你需要创建一个继承自`AbstractJni`的类，并将其`this`传入。`AbstractJni`已实现了大量常用的JNI方法，对于缺失的方法，你需要重写并在其中补充逻辑。 |
| **`setVerbose(boolean)`** | 开启或关闭详细日志。开启后会打印每一次JNI调用和系统调用，**在调试阶段非常有用**，生产环境建议关闭以提升性能。 |

**配置示例：**
```java
// 假设你的类 `MyJniHandler` 继承了 `AbstractJni`
vm.setJni(new MyJniHandler()); 
vm.setVerbose(true); // 开启详细日志
```

### 🗂️ 类与对象操作

VM提供了丰富的API来在模拟环境中操作Java类和对象。

#### 类解析

| 方法 | 描述 |
| :--- | :--- |
| **`resolveClass(String className)`** | 根据类名（如 `"com/example/MainActivity"`）获取对应的`DvmClass`对象。这是进行任何类级别操作（如调用静态方法、创建实例）的起点。 |

**使用示例：**
```java
DvmClass mainActivityClass = vm.resolveClass("com/example/MainActivity");
```

#### 对象创建与引用管理

| 方法 | 描述 |
| :--- | :--- |
| **`newObject(Object... args)`** | 在`DvmClass`上调用，用于创建该类的实例。返回一个`DvmObject`。 |
| **`addLocalObject(DvmObject<?> obj)`** | 将一个对象添加到本地引用表，并返回其**hash值（int）**。这个hash值在JNI调用中作为`jobject`引用传递，可防止对象被GC。 |
| **`getObject(int hash)`** | 根据hash值从VM中获取对应的`DvmObject`对象。 |
| **`addGlobalObject(DvmObject<?> obj)`** | 向VM添加一个全局对象引用。 |

**使用示例：**
```java
// 创建 MainActivity 类的实例
DvmObject<?> activityObj = mainActivityClass.newObject(null);

// 将其添加到本地引用表，获取 jobject 的 hash 值
int jobjectHash = vm.addLocalObject(activityObj);
```

### 📞 方法调用

调用SO中的JNI方法主要有两种方式：通过`DvmClass`和通过`Module`。

#### 通过 DvmClass 调用（推荐）

这种方式更符合JNI的调用习惯，unidbg会自动处理方法查找和参数封装。

| 方法 | 描述 |
| :--- | :--- |
| **`callStaticJniMethod(Emulator<?> emulator, String methodSignature, Object... args)`** | 调用类的**静态**Native方法。 |
| **`callJniMethod(Emulator<?> emulator, String methodSignature, Object... args)`** | 调用对象的**实例**Native方法。 |

**方法签名**格式为 `"methodName(参数类型)返回类型"`，例如 `"getSign(Ljava/lang/String;)Ljava/lang/String;"`。

**使用示例：**
```java
DvmClass securityClass = vm.resolveClass("com/example/SecurityUtils");

// 调用静态方法，传入一个字符串参数
Number result = securityClass.callStaticJniMethod(emulator, 
    "encrypt(Ljava/lang/String;)Ljava/lang/String;", 
    new StringObject(vm, "hello"));
```

#### 通过 Module 调用（底层方式）

这种方式更底层，需要手动准备所有JNI参数，包括`JNIEnv*`指针和`jobject`引用。

| 方法 | 描述 |
| :--- | :--- |
| **`Module.callFunction(Emulator<?> emulator, long offset, Object... args)`** | 直接调用SO中指定偏移地址的函数。需要手动构造参数列表，通常第一个参数是`vm.getJNIEnv()`。 |

**使用示例：**
```java
// 手动准备JNI参数
List<Object> args = new ArrayList<>();
args.add(vm.getJNIEnv()); // JNIEnv* 指针
args.add(vm.addLocalObject(myObject)); // jobject 引用
args.add(vm.addLocalObject(new StringObject(vm, "input"))); // jstring 参数

Number result = module.callFunction(emulator, 0x12345, args.toArray());
```

### 💡 使用建议

*   **优先使用`DvmClass.callStaticJniMethod`**：相比手动构造参数调用`Module.callFunction`，这种方式更简洁、不易出错。
*   **善用`AbstractJni`**：遇到`UnsupportedOperationException`时，说明JNI调用到了`AbstractJni`未实现的方法。此时需要重写对应方法（如`callVoidMethodV`）并补充逻辑，这是“补环境”的核心工作。
*   **注意参数类型**：传递给JNI方法的对象参数（如String, byte[]）需要用`StringObject`、`ByteArray`等unidbg包装类封装，并通过`vm.addLocalObject()`传递其hash值。
*   **引用管理**：通过`addLocalObject`添加的对象引用，其生命周期由VM管理，无需手动释放，这简化了内存管理。



## 符号调用和地址调用
在 unidbg 中调用 SO 函数主要有两种方式：**符号调用**（通过函数名查找地址）和**地址调用**（直接使用偏移或绝对地址）。理解它们的区别和正确用法，是分析 Native 层逻辑的基础。

---

### 一、核心概念

| 方式 | 依据 | 稳定性 | 适用场景 |
| :--- | :--- | :--- | :--- |
| **符号调用** | 函数导出名（如 `Java_com_example_NativeLib_encrypt`） | 高，不受基址变化影响 | 有导出符号的 JNI 函数、公开 API |
| **地址调用** | 函数在 SO 中的偏移或绝对地址 | 低，SO 更新后偏移可能变化 | 无符号、被混淆、内部私有函数 |

unidbg 中，`Module` 接口同时支持两种方式：
- `Module.callFunction(emulator, offset, args)` → **相对偏移调用**（`offset` 是相对于模块基址的偏移）
- `Emulator.eFunc(address, args)` → **绝对地址调用**（`address` 是运行时内存中的绝对地址）

---

### 二、符号调用

#### 1. 查找符号

加载 SO 后，通过 `Module.findSymbolByName()` 获取 `Symbol` 对象：

```java
DalvikModule dm = vm.loadLibrary(new File("libtarget.so"), true);
Module module = dm.getModule();

Symbol symbol = module.findSymbolByName("Java_com_example_NativeLib_encrypt");
if (symbol == null) {
    throw new IllegalStateException("未找到符号");
}
```

- `findSymbolByName(String name)`：查找导出符号。
- `findSymbolByName(String name, boolean withGlobalSymbol)`：是否包含全局符号（如导入符号）。
- `module.getSymbols()`：获取所有符号集合，可用于遍历。

#### 2. 获取地址与偏移

`Symbol` 提供两个关键方法：

| 方法 | 返回值 | 含义 |
| :--- | :--- | :--- |
| `getAddress()` | 绝对地址 | 运行时内存地址 = 模块基址 + 符号值 |
| `getValue()` | 相对偏移 | 符号在模块内的偏移（`st_value`） |

```java
long absoluteAddr = symbol.getAddress(); // 绝对地址
long offset = symbol.getValue();         // 相对偏移
```

#### 3. 调用示例

**方式一：绝对地址调用（`emulator.eFunc`）**
```java
List<Object> args = new ArrayList<>();
args.add(vm.getJNIEnv());                     // JNIEnv*
args.add(vm.addLocalObject(clazz));           // jclass 或 jobject
args.add(vm.addLocalObject(new StringObject(vm, "input")));

Number ret = emulator.eFunc(symbol.getAddress(), args.toArray());
```

**方式二：相对偏移调用（`module.callFunction`）**
```java
Number ret = module.callFunction(emulator, symbol.getValue(), args.toArray());
```

> **注意**：`module.callFunction` 的第二个参数是**相对偏移**，不要直接传入 `symbol.getAddress()`，否则会重复加基址导致错误。

---

### 三、地址调用

当函数没有导出符号，或你已知函数在 SO 中的偏移时，可直接使用地址调用。

#### 1. 获取函数偏移

- **静态分析**：用 IDA、Ghidra 等工具查看函数地址，减去 SO 的加载基址得到偏移。
- **动态查找**：通过 `module.findClosestSymbolByAddress(address, true)` 找到最近的符号，再计算偏移。
- **已知 JNI 函数**：JNI 函数名通常有规律，但若被混淆则需动态调试。

#### 2. 调用示例

**相对偏移调用（推荐，与模块基址解耦）**
```java
long offset = 0x1234; // 目标函数在 SO 中的偏移
Number ret = module.callFunction(emulator, offset, args.toArray());
```

**绝对地址调用**
```java
long absoluteAddr = module.getBaseAddress() + offset;
Number ret = emulator.eFunc(absoluteAddr, args.toArray());
```

`module.getBaseAddress()` 返回模块加载的基址。

---

### 四、两种方式的对比与选择

| 维度 | 符号调用 | 地址调用 |
| :--- | :--- | :--- |
| **可读性** | 高，直接使用函数名 | 低，需注释偏移含义 |
| **稳定性** | 高，SO 更新后符号名通常不变 | 低，偏移可能因代码变动而改变 |
| **适用性** | 仅限导出符号 | 任意函数，包括内部函数 |
| **查找成本** | 低，`findSymbolByName` 即可 | 高，需逆向分析确定偏移 |
| **推荐场景** | JNI 函数、公开 API | 无符号函数、混淆代码、内部调用 |

**优先使用符号调用**，除非符号不存在或需要调用非导出函数。

---

### 五、JNI 参数构造要点

无论符号调用还是地址调用，调用 JNI 函数时都需要手动构造参数：

```java
List<Object> args = new ArrayList<>();
args.add(vm.getJNIEnv());                          // 固定第一个参数：JNIEnv*
args.add(vm.addLocalObject(jobject));              // 第二个参数：jobject 或 jclass
args.add(vm.addLocalObject(new StringObject(vm, "hello"))); // 后续参数
// 基本类型直接传值，如 int、long、boolean
Number ret = module.callFunction(emulator, offset, args.toArray());
```

- 对象参数必须用 `vm.addLocalObject()` 包装，返回 `int` 型 hash 值作为 `jobject` 引用。
- 字符串用 `StringObject`，字节数组用 `ByteArray` 等。
- 静态 JNI 方法第二个参数是 `jclass`，实例方法是 `jobject`。

---

### 六、高级：通过 VM 的符号调用

对于 JNI 函数，unidbg 提供了更上层的封装，内部自动完成符号查找和参数处理：

```java
DvmClass clazz = vm.resolveClass("com/example/NativeLib");
Number ret = clazz.callStaticJniMethod(emulator, 
    "encrypt(Ljava/lang/String;)Ljava/lang/String;", 
    new StringObject(vm, "input"));
```

这种方式本质仍是符号调用，但隐藏了 `JNIEnv*`、`jclass` 等底层细节，**推荐优先使用**。

---

### 七、注意事项

1. **绝对地址 vs 相对偏移**：`Symbol.getAddress()` 返回绝对地址，`Symbol.getValue()` 返回相对偏移。`module.callFunction` 需要相对偏移，`emulator.eFunc` 需要绝对地址。
2. **符号缺失**：若 `findSymbolByName` 返回 `null`，说明符号未导出或被裁剪，只能通过地址调用。
3. **基址变化**：每次加载 SO，`module.getBaseAddress()` 可能不同，因此**不要硬编码绝对地址**，应使用相对偏移或符号。
4. **参数顺序**：JNI 函数第一个参数永远是 `JNIEnv*`，第二个是 `jclass`/`jobject`，不要遗漏。
5. **返回值**：`callFunction` 和 `eFunc` 返回 `Number`，根据函数返回类型转换（如 `intValue()`、`longValue()`）。若返回对象引用，需通过 `vm.getObject(hash)` 获取。
6. **`eFunc` 与 `callFunction` 的选择**：`eFunc` 直接执行绝对地址，适合已定位到具体地址的场景；`callFunction` 绑定模块，自动处理基址，更安全。

掌握符号调用与地址调用，就能灵活地在 unidbg 中执行 SO 的任意函数，为后续的协议分析、算法还原打下基础。







# unidbg中的hook
## 简介
unidbg 的 Hook 体系是一个分层级、多框架的架构，核心思路是**从高层往低层选**，优先使用能完成任务的最高层级抽象，避免滥用底层的指令级 Hook 导致性能问题。

### 🏗️ Hook 体系架构：六个抽象层级

unidbg 内置了六个 Hook 框架，它们并非并列关系，而是从高到低覆盖了不同的抽象层次：

| 层级 | 框架/机制 | 拦截目标 | 典型场景 |
| :--- | :--- | :--- | :--- |
| **最高层** | `FunctionCallListener` | JNI 函数调用 | 纯观察 JNI 调用，不修改行为 |
| | `SystemPropertyHook` | 系统属性读取 | 拦截 `System.getProperty` 等调用 |
| | **xHook** | **GOT/PLT 表（库函数导入）** | 拦截 SO 对 `malloc`、`gettimeofday` 等外部库函数的调用 |
| | **HookZz** | **函数头（Inline 替换）** | 替换整个函数或包裹函数的 pre/post 逻辑 |
| | **Dobby** | **函数头（Inline 重写）** | 与 HookZz 类似，通用 Inline Hook |
| **最低层** | **CodeHook** | **单条指令** | 指令级追踪、修改 PC 寄存器 |

此外，`emulator.attach().addBreakPoint()` 提供了基于软件断点的回调机制，适合点对点的精确拦截。

### 🔧 各 Hook 框架详解

#### 1. CodeHook（指令级 Hook）

直接基于 Unicorn 后端的 `hook_add_new` 实现，在**每条指令执行前**触发回调。

```java
emulator.getBackend().hook_add_new(new CodeHook() {
    @Override
    public void hook(Backend backend, long address, int size, Object user) {
        if (address == module.base + 0xABE) {
            // 修改 PC 寄存器跳过当前指令
            emulator.getBackend().reg_write(ArmConst.UC_ARM_REG_PC, address + 4 + 1);
        }
    }
}, module.base + 0xABE, module.base + 0xABE, null);
```

**注意事项**：手动修改 PC 时需处理 ARM/Thumb 模式（地址最低位为 1 表示 Thumb）。CodeHook **仅 Unicorn/Unicorn2 后端支持**，Dynarmic 后端不支持。**不要用 CodeHook 监控整个函数的执行轨迹**，性能开销极大。

#### 2. 断点回调（BreakPointCallback）

比 CodeHook 更易用的指令级拦截方式，通过 `emulator.attach()` 注册：

```java
emulator.attach().addBreakPoint(module.base + 0xABE, new BreakPointCallback() {
    @Override
    public boolean onHit(Emulator<?> emulator, long address) {
        emulator.getBackend().reg_write(ArmConst.UC_ARM_REG_PC, address + 4 + 1);
        return true; // 返回 true 表示处理完毕，继续执行
    }
});
```

这种方式**推荐优先于直接使用 CodeHook**，因为 unidbg 会自动处理指令集模式等细节。

#### 3. HookZz（Inline Hook）

HookZz 在**函数头部写入跳转指令**来重定向执行流，支持 `wrap`（包裹）和 `replace`（替换）两种模式。

**包裹模式（wrap）**：在函数执行前后插入逻辑，不改变原函数行为：
```java
IHookZz hookZz = HookZz.getInstance(emulator);
hookZz.wrap(module.base + 0x1234, new WrapCallback<HookZzArm64RegisterContext>() {
    @Override
    public void preCall(Emulator<?> emulator, HookZzArm64RegisterContext ctx, HookEntryInfo info) {
        // 函数执行前：读取参数
        long arg0 = ctx.getXLong(0);
        System.out.println("arg0 = " + arg0);
    }
    @Override
    public void postCall(Emulator<?> emulator, HookZzArm64RegisterContext ctx, HookEntryInfo info) {
        // 函数执行后：读取返回值并修改
        long ret = ctx.getXLong(0);
        ctx.setXLong(0, 0xdeadbeef); // 修改返回值
    }
});
```

**替换模式（replace）**：完全替换原函数的执行逻辑：
```java
hookZz.replace(module.base + 0x1234, new ReplaceCallback() {
    @Override
    public HookStatus onCall(Emulator<?> emulator, long originFunction) {
        // 自定义逻辑，直接返回
        emulator.getBackend().reg_write(ArmConst.UC_ARM_REG_X0, 0x1234);
        return HookStatus.RET(emulator, originFunction); // 直接返回，不执行原函数
    }
});
```

**关键点**：HookZz **必须在 `loadLibrary` 之后使用**，因为需要先通过 `findSymbolByName` 或模块基址定位函数。

#### 4. Dobby（Inline Hook）

与 HookZz 类似，基于 Dobby 库实现 Inline Hook，通过重写指令序列来重定向执行流。适用于需要 Hook **任意函数**（包括非导出函数）的场景。Dobby 与 HookZz 功能重叠，**不建议同时使用两者 Hook 同一函数**。

#### 5. xHook（GOT/PLT Hook）

xHook 通过修改 ELF 的 **GOT（全局偏移表）和 PLT（过程链接表）** 来拦截 SO 对外部库函数的调用。

```java
IxHook xHook = XHookImpl.getInstance(emulator);
xHook.register("libtarget.so", "strlen", new ReplaceCallback() {
    @Override
    public HookStatus onCall(Emulator<?> emulator, long originFunction) {
        // 拦截 strlen 调用，自定义返回值
        emulator.getBackend().reg_write(ArmConst.UC_ARM_REG_X0, 0); // 返回 0
        return HookStatus.RET(emulator, originFunction);
    }
});
xHook.refresh(); // 必须调用 refresh 使 Hook 生效
```

**核心区别**：xHook 拦截的是 **“SO 内部调用外部函数”的入口**（调用方一侧），而 HookZz 修改的是**被调用函数本身的函数头**。

**时机关键**：xHook 应在 `loadLibrary` **之前**注册，因为 `JNI_OnLoad` 经常会调用 libc 函数（如初始化时间戳、读 system property），需要提前拦截。**忘记调用 `xHook.refresh()` 是常见错误**，会导致 Hook 不生效。

#### 6. FunctionCallListener（JNI 调用观察）

纯观察型 Hook，基于 Debugger 内部的 BL/BLR 指令钩子实现，用于监控 JNI 函数的调用时机和参数。

```java
Debugger debugger = emulator.attach();
// 监控所有 JNI 调用，或指定范围
debugger.addFunctionCallListener((emulator, function, args) -> {
    System.out.println("JNI called: " + function);
});
```

#### 7. SystemPropertyHook

专门用于拦截系统属性读取（如 `System.getProperty("ro.build.version.sdk")`），在 `dlsym` 解析符号时把目标函数重定向到 SVC handler，**比 PLT Hook 更早一层**。

### ⚠️ 关键注意事项与最佳实践

**1. 一个函数只用一个 Hook 框架**

同时使用 xHook 和 HookZz Hook 同一函数会导致**不可预测的行为**，因为两者修改的不是同一个位置，对寄存器保存（尤其是 LR）的假设可能错位。如果需要在 Hook 中组合多个逻辑（如同时统计和拦截），应在**同一个 Hook 回调内部**自行组合。

**2. 选型策略：从上往下选**

| 需求                | 推荐框架                 |
| :---------------- | :------------------- |
| 观察 JNI 调用         | FunctionCallListener |
| 拦截系统属性            | SystemPropertyHook   |
| 替换 SO 对外部库函数的调用   | xHook                |
| 替换/包裹任意函数（含非导出函数） | HookZz 或 Dobby       |
| 在单条指令上做手脚         | 断点回调 或 CodeHook      |

**不要直接上 CodeHook**，也不要一上来就写 Inline Hook。

**3. 后端限制**

CodeHook / BlockHook **仅 Unicorn / Unicorn2 后端支持**。如果使用 Dynarmic 后端，需改用其他 Hook 方式。

**4. Hook 与 Patch 的区别**

Hook 是**运行时拦截**，对原始二进制非破坏性；Patch 是**直接修改内存中的机器码**，永久性改变执行逻辑。绕过简单的签名校验时，Patch（如将条件跳转改为无条件跳转）可能比 Hook 更简洁。

**5. 定位 Hook 地址**

无论哪种 Hook，精准定位目标地址都是前提。可通过 `module.findSymbolByName()` 查找导出符号，或结合 IDA/Ghidra 反汇编分析确定偏移，再计算 `module.base + offset`。

**6. Hook 框架选型速查**

unidbg 默认集成的框架包括 Dobby（Inline Hook）、xHook（GOT Hook）、Whale（iOS Inline Hook）、fishhook（iOS 符号重绑定）等。Android 平台最常用的是 **xHook + HookZz/Dobby** 的组合，iOS 平台则使用 Whale 或 fishhook。



## HookZz
HookZz 是 unidbg 中一款功能强大的 Inline Hook 框架，它通过直接修改目标函数的机器码，将执行流重定向到自定义的回调函数，从而实现对 SO 内部任意函数（包括非导出函数）的精细控制。

### ⚙️ 核心原理：指令替换

HookZz 的工作方式是在目标函数的**头部**备份并替换开头的几条指令，将其改写为一条跳转指令（如 `B` 或 `BL`），从而将 CPU 的执行流导向我们预设的回调函数。由于其设计上的优化，HookZz 甚至能够 Hook 仅包含一条指令的“短函数”。

### 🚀 快速上手：基本使用流程

使用 HookZz 的基本步骤如下：

1.  **获取实例**：通过 `HookZz.getInstance(emulator)` 获取 HookZz 实例。
2.  **定位地址**：使用 IDA 等工具找到目标函数在 SO 中的偏移，并计算出其绝对地址（`module.base + offset`）。**注意**：如果目标是 Thumb 指令集（ARM32），地址需要 **+1** 来标记。
3.  **执行 Hook**：调用 `wrap()` 或 `replace()` 方法进行拦截。

### 🎯 两种 Hook 模式

HookZz 提供了两种主要的 Hook 模式：

*   **`wrap()` - 包裹模式**：在不改变原函数逻辑的前提下，在函数**执行前后**插入自定义代码。适合用于记录参数、观察返回值、统计执行次数等**纯观察或轻量干预**场景。
*   **`replace()` - 替换模式**：**完全替换**原函数的执行逻辑。当回调返回 `HookStatus.RET` 时，原函数将**不会执行**。适合用于**模拟函数返回、绕过复杂逻辑、强制修改行为**等场景。

**`wrap` 模式示例（记录参数）**：
```java
IHookZz hookZz = HookZz.getInstance(emulator);
hookZz.wrap(module.base + 0x1234, new WrapCallback<HookZzArm64RegisterContext>() {
    @Override
    public void preCall(Emulator<?> emulator, HookZzArm64RegisterContext ctx, HookEntryInfo info) {
        // 函数执行前：读取第一个参数
        long arg0 = ctx.getXLong(0);
        System.out.println("arg0 = " + arg0);
    }
    @Override
    public void postCall(Emulator<?> emulator, HookZzArm64RegisterContext ctx, HookEntryInfo info) {
        // 函数执行后：修改返回值
        ctx.setXLong(0, 0xdeadbeef);
    }
});
```
> `postCall` 默认不执行，需在 `wrap` 的第三个参数中传入 `true` 来启用。

**`replace` 模式示例（强制返回）**：
```java
hookZz.replace(module.base + 0x1234, new ReplaceCallback() {
    @Override
    public HookStatus onCall(Emulator<?> emulator, long originFunction) {
        // 直接设置返回值，不执行原函数
        emulator.getBackend().reg_write(ArmConst.UC_ARM_REG_X0, 0x1);
        return HookStatus.RET(emulator, originFunction); // 立即返回
    }
});
```

### 📋 寄存器上下文详解

HookZz 的回调方法会传入一个 `RegisterContext` 对象，用于读写寄存器，这是实现精细控制的关键。

| 架构 | 上下文类型 | 常用方法 |
| :--- | :--- | :--- |
| **ARM32** | `HookZzArm32RegisterContext` | `getIntArg(index)` / `setIntArg(index, value)`：获取/设置第 N 个整型参数。 |
| | | `getPointerArg(index)`：获取指针参数，返回 `UnidbgPointer`，可进一步读取内存。 |
| **ARM64** | `HookZzArm64RegisterContext` | `getXLong(index)` / `setXLong(index, value)`：获取/设置第 N 个 64 位参数或返回值。 |
| | | `getXInt(index)` / `setXInt(index, value)`：获取/设置 32 位参数。 |

**通用读写**：你也可以通过 `ctx.getLongArg(index)` 或 `ctx.setLongArg(index, value)` 这类更通用的方法来操作参数。在 `postCall` 中，修改 `X0`（ARM64）或 `R0`（ARM32）通常用于改变函数的返回值。

### ⚠️ 高级技巧与注意事项

*   **地址修正**：在 ARM32 环境下，如果目标函数使用 Thumb 指令集，`module.base + offset + 1` 的地址修正至关重要。
*   **启用 B 分支**：在某些情况下，你可能需要调用 `hookZz.enable_arm_arm64_b_branch()` 来确保 HookZz 正确处理 ARM64 的 B 分支指令。
*   **一个函数只用一个框架**：不要同时使用 HookZz 和 Dobby 等不同框架 Hook 同一个函数，这可能导致不可预测的行为。
*   **架构兼容性**：HookZz 在处理 ARM32 的某些指令（如 `BLX`）时可能存在约 5% 的上下文切换失败风险，且其 ARM64 指令覆盖率（约 78%）低于 Dobby（约 92%），在复杂场景下可能需考虑 Dobby。
*   **性能考量**：Inline Hook 会修改代码，带来额外的性能开销。在高频调用的函数上使用 HookZz 时需留意其对执行速度的影响。

### 💎 总结

HookZz 是 unidbg 中用于拦截和分析 SO 内部函数的**核心工具**。它的 `wrap` 模式适合**观察**，`replace` 模式适合**干预**。通过灵活运用其寄存器上下文，你可以像操作本地变量一样读写模拟 CPU 的寄存器和内存，从而深入洞察和重塑 Native 代码的执行逻辑。





## Dobby
`Dobby` 是 unidbg 中另一款与 `HookZz` 齐名的 Inline Hook 框架。如果说 `HookZz` 以精巧著称，那么 `Dobby` 则以其**更广泛的兼容性**和**对 ARM64 架构更优的支持**见长。它在 unidbg 中常被用于处理 `HookZz` 难以驾驭的复杂场景，尤其是在 64 位环境中。

### ⚙️ 核心原理：函数头重写与执行流重定向

与 `HookZz` 类似，`Dobby` 也是一种 Inline Hook，其核心原理是**直接重写目标函数头部的机器码**，将 CPU 的执行流重定向到用户自定义的回调函数。

它通过在目标函数开头写入一条跳转指令（如 `B` 或 `BL`），将执行权交给 `Dobby` 的代理（trampoline）代码，进而跳转到你的回调。当 `Dobby` 完成 Hook 后，它会提供一个指向原始函数（已处理过指令重定位）的指针，以便在需要时还能调用原函数逻辑。

### 🚀 在 unidbg 中的使用

在 unidbg 中使用 `Dobby` 的流程与 `HookZz` 相似，但 API 略有不同。

1.  **获取实例**：通过 `Dobby.getInstance(emulator)` 获取 `Dobby` 实例。
2.  **定位目标**：通过 `findSymbolByName` 或已知偏移，确定目标函数的地址。
3.  **执行 Hook**：调用 `replace` 方法进行拦截。

**`replace` 模式示例（拦截并修改 `open` 函数）**：
以下示例展示了如何拦截 `libc.so` 中的 `open` 函数，当检测到特定路径时，直接返回 `-1` 以绕过反模拟检查。

```java
// 1. 定位 libc.so 中的 open 函数
Module libc = emulator.getMemory().findModule("libc.so");
Symbol openSymbol = libc.findSymbolByName("open", false);

// 2. 获取 Dobby 实例并执行 Hook
Dobby dobby = Dobby.getInstance(emulator);
dobby.replace(openSymbol, new ReplaceCallback() {
    @Override
    public HookStatus onCall(Emulator<?> emulator, HookContext context, long originFunction) {
        // 获取第一个参数（pathname）
        RegisterContext r = emulator.getContext();
        String pathname = r.getPointerArg(0).getString(0);

        // 3. 条件判断与干预
        if ("/proc/self/anti_syscall_test".equals(pathname)) {
            // 返回 -1，并直接返回到调用者，不执行原 open 函数
            return HookStatus.LR(emulator, -1L);
        }
        // 否则，正常执行原函数
        return HookStatus.RET(emulator, originFunction);
    }
});
```

**关键 API 说明**：
*   **`HookStatus.LR(emulator, returnValue)`**：设置函数返回值（放在返回寄存器中），并**直接返回到调用者**，完全跳过原函数逻辑。用于强制返回的场景。
*   **`HookStatus.RET(emulator, originFunction)`**：跳转到 `originFunction`（原始函数）继续执行。用于观察或修改参数后放行的场景。

### ⚖️ 与 HookZz 的深度对比

虽然两者都是 Inline Hook，但在 unidbg 的模拟环境中，它们的表现有显著差异，这是技术选型的关键。

| 对比维度 | **HookZz (v6.0)** | **Dobby (v2.0)** |
| :--- | :--- | :--- |
| **ARM64 指令覆盖率** | 约 78% | **约 92%**，支持更全面 |
| **ARM32 稳定性** | 处理 `BLX` 指令时约有 **5% 的上下文切换失败率** | 在相同场景下**保持稳定** |
| **寄存器保存** | 存在 `R4-R7` 丢失的风险 | **全寄存器保护**，上下文保存更完整 |
| **跳转指令处理** | 需要**手动修复 PC** 寄存器 | **自动重定位**，简化了处理 |
| **多线程 Hook** | 需要**额外加锁**来保证稳定性 | 使用**原子操作**，并发安全性更好 |
| **ARM32/ARM64 选择** | 在**32位 ARM** 环境下经验更丰富 | 在 **64位 ARM64** 环境下优势明显 |

**核心结论**：对于 **64位 (ARM64)** 的 SO，**优先选择 Dobby**，其更高的指令覆盖率和自动重定位能避免许多难以排查的崩溃问题。对于 **32位 (ARM32)** 的 SO，`HookZz` 依然是可靠的选择，但若遇到稳定性问题，应果断切换到 `Dobby` 测试。

### ⚠️ 注意事项与最佳实践

1.  **地址修正（ARM32）**：与 `HookZz` 相同，如果目标是 Thumb 指令集，传入的地址需要 **`+1`** 来标记 Thumb 模式。
2.  **Hook 时机**：`Dobby` 需要在目标函数被调用**之前**完成 Hook 的注册。通常在 `loadLibrary` 之后、调用目标函数之前进行。
3.  **避免混用**：**绝对不要**用 `Dobby` 和 `HookZz` 同时 Hook 同一个函数，这会导致指令流混乱和不可预知的行为。
4.  **性能考量**：Inline Hook 会带来额外的性能开销，在高频调用或对性能敏感的路径上使用时需谨慎评估。

### 💎 总结

`Dobby` 和 `HookZz` 是 unidbg 中用于精细控制 SO 内部逻辑的两把利刃。`HookZz` 在 ARM32 场景下有其独特的价值，而 **`Dobby` 则是处理 ARM64 复杂 SO 的更稳健、更强大的工具**。当你在 64 位环境中进行 Hook 并遇到 `HookZz` 无法解决的问题时，切换到 `Dobby` 往往是破局的关键。




## xHook
xHook 是 unidbg 中用于**库函数导入拦截**的核心框架。与 HookZz、Dobby 直接改写函数头不同，xHook 的工作层次更高，它修改的是 ELF 文件的**GOT/PLT 表**，从而拦截**调用方**对外部符号的引用。

### ⚙️ 核心原理：修改“引用”而非“函数”

理解 xHook 的关键在于：**它拦截的是“谁在调用”，而不是“谁被调用”**。

当 SO 文件调用一个外部函数（如 `strlen`）时，会通过 PLT（过程链接表）跳转到 GOT（全局偏移表）中记录的地址。xHook 的作用就是**修改 GOT 表中的这个地址**，将其指向我们自定义的回调函数，从而在调用发生前完成拦截。

这种设计带来了两个重要特性：
- **只影响当前模块的调用**：xHook 在注册时可以指定模块名，只会修改目标模块的 GOT/PLT 表，其他模块调用同一函数不受影响。
- **与 Inline Hook 正交**：xHook 修改的是“调用方”的引用表，HookZz/Dobby 修改的是“被调用方”的函数头。两者在原理上可以共存，但**对同一函数同时使用会导致不可预测的行为**。

### 🚀 在 unidbg 中的使用

#### 基本流程

```java
// 1. 获取 xHook 实例
IxHook xHook = XHookImpl.getInstance(emulator);

// 2. 注册 Hook：指定模块、符号名、回调
xHook.register("libtarget.so", "strlen", new ReplaceCallback() {
    @Override
    public HookStatus onCall(Emulator<?> emulator, long originFunction) {
        // 读取第一个参数（字符串指针）
        RegisterContext ctx = emulator.getContext();
        String str = ctx.getPointerArg(0).getString(0);
        System.out.println("strlen called with: " + str);
        
        // 返回自定义值，不执行原函数
        return HookStatus.LR(emulator, (long) str.length());
    }
});

// 3. 必须调用 refresh() 使 Hook 生效
xHook.refresh();
```

#### 关键 API 说明

| API | 作用 |
| :--- | :--- |
| **`register(String module, String symbol, ReplaceCallback callback)`** | 注册 Hook。`module` 是目标模块名（如 `"libtarget.so"`），`symbol` 是外部符号名（如 `"malloc"`）。 |
| **`refresh()`** | **必须调用**，否则注册的 Hook 不会生效。建议使用同步模式 `refresh(false)`，确保 Hook 立即应用。 |
| **`HookStatus.LR(emulator, returnValue)`** | 设置返回值并**直接返回给调用者**，跳过原函数执行。 |
| **`HookStatus.RET(emulator, originFunction)`** | 跳转到原函数继续执行（用于观察或修改参数后放行）。 |

#### 注册时机：越早越好

xHook 的注册时机有一个关键原则：**必须在目标函数被首次调用之前完成**。更准确地说，是“必须在目标函数被首次调用之前”。

实践中，**推荐在 `loadLibrary` 之前注册 xHook 并调用 `refresh()`**，因为 `JNI_OnLoad` 经常会调用 libc 函数（如初始化时间戳、读取 system property），如果这些函数需要被拦截，必须在 `JNI_OnLoad` 执行前就完成 Hook 注册。

```java
// 推荐姿势：先注册 xHook，再加载 SO
IxHook xHook = XHookImpl.getInstance(emulator);
xHook.register("libtarget.so", "gettimeofday", fakeTimeCb);
xHook.refresh(); // 立即生效

// 然后再加载 SO，JNI_OnLoad 中的 libc 调用也会被拦截
DalvikModule dm = vm.loadLibrary(new File("libtarget.so"), true);
```

### ⚖️ 与 HookZz / Dobby 的核心区别

| 维度 | **xHook** | **HookZz / Dobby** |
| :--- | :--- | :--- |
| **拦截层次** | GOT/PLT 表（导入表） | 函数头（Inline 重写） |
| **拦截对象** | **调用方**对某个符号的引用 | **被调用方**的函数本身 |
| **影响范围** | 仅注册时指定的模块 | 全局，所有调用该函数的代码 |
| **适用场景** | 拦截 SO 对 libc 等外部库的调用 | 拦截 SO **内部**的任意函数（含非导出函数） |
| **能否 Hook 内部函数** | **不能**，只能 Hook 导入符号 | **可以**，通过地址即可 |
| **生效时机** | 需在目标调用发生前 `refresh()` | 在目标函数被调用前完成注册即可 |
| **被 Hook 函数地址** | 需要知道模块名和符号名 | 需要知道函数偏移地址 |

**选型原则**：拦截 SO 对外部库函数的调用 → 用 xHook；拦截 SO 内部自有函数 → 用 HookZz/Dobby。

### ⚠️ 注意事项与常见陷阱

**1. 忘记调用 `refresh()`**

这是最常见的错误。注册 Hook 后不调用 `refresh()`，Hook 不会生效，且没有任何报错提示。**封装一个 helper 函数，在注册后自动 refresh** 是推荐的最佳实践。

**2. 与 Inline Hook 混用同一函数**

不要用 xHook 和 HookZz 同时 Hook 同一个函数。两者修改的位置不同（xHook 改调用方引用，HookZz 改被调用方函数头），各自的 trampoline 对寄存器保存和调用链的假设不一致，叠加后行为不可预测。

**3. 多模块注册与正则表达式**

xHook 支持用**正则表达式**匹配模块路径，可以一次注册多个模块的 Hook：

```java
// 匹配所有 /system/ 下的 so 文件
xHook.register("^/system/.*\\.so$", "__android_log_print", myLogPrint);

// 匹配 libtest.so
xHook.register(".*/libtest\\.so$", "malloc", myMalloc);
```

**4. 异步刷新模式**

`refresh(true)` 会创建后台线程异步应用 Hook，适用于 Hook 数量多或需要在运行时动态刷新的场景。但在大多数 unidbg 逆向分析场景中，**同步模式 `refresh(false)` 更简单可靠**。

**5. 不能拦截内联 SVC 调用**

xHook 只能拦截通过 PLT/GOT 的**标准库函数调用**。如果 SO 中使用**内联汇编直接发起 SVC 指令**（不经过 libc 包装），xHook 无法拦截。

**6. 不是所有 libc 函数都能被拦截**

某些函数（如较新 Bionic 中的 `__system_property_get`）可能不经过 PLT 调用，而是直接读取共享内存，xHook 对此无能为力。

### 💎 总结

xHook 是 unidbg 中**补环境**和**拦截库函数调用**的首选工具。它的核心价值在于：**在调用方一侧完成拦截**，信息丰富（能拿到 C 函数参数），侵入性低（不需要修改被调用函数的代码），且只影响注册的模块。使用时记住三个关键点：**在目标调用前注册、必须 refresh、不要与 Inline Hook 混用同一函数**。




## CodeHook
`CodeHook` 是 unidbg 中**最底层、最强大，也最需要谨慎使用**的 Hook 机制。它是唯一能做到**指令粒度**拦截的钩子，但代价是巨大的性能开销。

### ⚙️ 核心原理：逐指令回调

`CodeHook` 直接基于 Unicorn/Unicorn2 后端的 `hook_add_new` 实现。其本质是一个**逐指令回调钩子**——引擎在执行每一条 ARM/ARM64 指令**之前**，都会回调一次 Java 层的 `hook()` 方法。

```java
emulator.getBackend().hook_add_new(new CodeHook() {
    @Override
    public void hook(Backend backend, long address, int size, Object user) {
        // 每条指令执行前触发
        // address: 当前指令地址
        // size: 指令大小
    }
}, begin, end, null); // begin/end 指定生效的地址范围
```

这意味着，如果监控范围包含 N 条指令，回调就会被触发 N 次。对于正常的 SO，一次函数调用可能产生几百到几万条 Trace，原版流程尚可承受；但在 OLLVM 或 VMP 保护下，函数的执行路径会急剧膨胀——VMP 把每条原始指令翻译成数十条 VM Handler 指令逐条解释执行，回调次数可达上亿次。

### 🚀 基本用法

`CodeHook` 最典型的用途是**在特定指令处修改程序执行流**。以下示例在 `module.base + 0xABE` 处插入 Hook，跳过当前指令：

```java
emulator.getBackend().hook_add_new(new CodeHook() {
    @Override
    public void hook(Backend backend, long address, int size, Object user) {
        System.out.println("Executing at: " + Long.toHexString(address));
        // 修改 PC 寄存器：+4 跳过当前 ARM 指令，+1 标记 Thumb 模式
        emulator.getBackend().reg_write(ArmConst.UC_ARM_REG_PC, address + 4 + 1);
    }
}, module.base + 0xABE, module.base + 0xABE, null);
```


> **注意**：`+4` 跳过当前 4 字节 ARM 指令，`+1` 标记 Thumb 指令集（地址最低位为 1 表示 Thumb 模式）。如果目标平台是 ARM64，地址始终 4 字节对齐，不需要 `+1`。

### 🔍 进阶：读取寄存器与内存

`CodeHook` 的回调中可以通过 `backend.reg_read()` 读取任意寄存器，进行指令级的现场观察：

```java
emulator.getBackend().hook_add_new(new CodeHook() {
    @Override
    public void hook(Backend backend, long address, int size, Object user) {
        long x0 = backend.reg_read(Arm64Const.UC_ARM64_REG_X0).longValue();
        System.out.println("Executing at 0x" + Long.toHexString(address) + ", x0 = " + x0);
    }
}, module.base + 0x1B00, module.base + 0x1B00, null);
```


### 🆚 CodeHook 与 Trace：本质相同，用途不同

`CodeHook` 与 unidbg 的 `Trace` 功能存在**本质上的从属关系**：

- **Trace 本身就是用 CodeHook 实现的**。`AssemblyCodeDumper` 类实现了 `CodeHook` 和 `TraceHook` 接口，通过逐指令回调来反汇编并记录执行流。
- **区别在于封装层级**：Trace 是对 CodeHook 的高层封装，内置了反汇编、缓存、输出格式化等优化；而裸 CodeHook 只提供原始的回调，所有处理逻辑都需要自己实现。

**选型原则**：监控整个函数的执行轨迹 → **不要用裸 CodeHook，用 Trace**。Trace 本身是 CodeHook 封装，性能更好、代码更简洁。只有在需要在**特定单条指令**上做精确干预（如修改 PC、读取特定寄存器）时，才使用裸 CodeHook。

### ⚠️ 性能陷阱与使用纪律

`CodeHook` 的性能开销是所有 Hook 框架中最大的，使用时必须严格遵守以下纪律：

**1. 范围尽可能窄**

绝对不要用 CodeHook 监控整个函数的执行。范围大一点性能就会“炸掉”。示例中 Hook 只对 `0x1B00` 这一条指令触发，begin 和 end 设为相同地址。

**2. 回调中的代码尽量短**

不要在回调里打印大量内容或做 IO 操作。回调中的代码越短越好，避免在指令级回调中执行复杂逻辑。

**3. 回调中绝对不能调用 Java 代码**

JNI 回调在每个指令 hook 上触发会导致**数十倍的性能损失**，而且 GC、死锁、JNI 引用管理在 hook 上下文中完全不可预测。如果确实需要停止模拟，设一个 flag 然后调用 `uc_emu_stop()` 即可。

**4. 后端限制**

`CodeHook` **仅在 Unicorn / Unicorn2 后端上可用**。切换到 Dynarmic 后端后，`hook_add_new(CodeHook, ...)` 会**直接抛出 `UnsupportedOperationException`**，不是“偶发不生效”而是完全不支持。要做指令级 Hook，只能使用 Unicorn 系列后端。

**5. 只在“非做不可”时使用**

高层 Hook 全都不适用时，才考虑 CodeHook。

### 📊 选型决策速查

| 我想做什么 | 推荐框架 | 为什么 |
| :--- | :--- | :--- |
| 在 SO 内部某条指令上做点事 | **CodeHook** | 唯一能做指令级拦截的 Hook |
| 监控整个函数的执行轨迹 | **不要用 CodeHook！用 Trace** | Trace 本身就是 CodeHook 封装，性能更好 |
| Hook SO 内部的函数（看参数/返回值） | HookZz (wrap) | 保留原始函数，前后观察 |
| Hook SO 内部的函数（完全替换） | HookZz (replace) | 直接换，不执行原始代码 |
| 拦截 SO 对 libc 函数的调用 | xHook | PLT Hook，粒度就是库函数 |


### 💡 性能优化方向

如果确实需要在 OLLVM/VMP 场景下使用 CodeHook 进行指令级分析，社区已有优化方案。核心思路是**对反汇编结果做 L1/L2 两级缓存**：由于 OLLVM/VMP 的指令膨胀效果，大量重复指令的机器码是相同的，如果每条指令都调用一次 Capstone 的 `disassemble()`，开销非常可观。使用地址取模的数组作为 L1 缓存，可以省去 HashMap 的查找开销，实测在 OLLVM+VMP 场景下可获得**一百多倍**的速度提升。

### 💎 总结

`CodeHook` 是 unidbg 中**精度最高、代价最大**的 Hook 机制。它的定位非常明确：**在单条指令上做精确干预**。使用时牢记三条铁律：**范围要窄、回调要短、绝不能调 Java**。如果需求是监控函数级别的执行轨迹，请直接用 Trace，而不是裸 CodeHook。``







# Console Debugger
`Console Debugger` 是 unidbg 中最直接、最灵活的交互式调试工具。它允许你在模拟执行过程中暂停程序，通过命令行与模拟器实时交互，从而像使用 GDB 一样，对 ARM 代码进行精细的观察和控制。

### 🏗️ 核心架构：如何附着与控制

`Console Debugger` 的起点是 `emulator.attach()` 方法。

*   **创建与附着**：调用 `emulator.attach()` 会创建一个平台相关的调试器实例（ARM32 对应 `SimpleARMDebugger`，ARM64 对应 `SimpleARM64Debugger`），并将其注册到后端作为调试钩子。此后，模拟器在遇到断点或单步执行时，就会将控制权交给调试器。
*   **调试器类型**：`DebuggerType.CONSOLE` 是默认的交互式控制台调试器。此外还有 `GDB_SERVER`（用于连接 GDB 客户端）和 `ANDROID_SERVER_V7`（用于 IDA 远程调试）。
*   **断点管理**：断点信息存储在 `breakMap` 中，支持**地址断点**、**符号断点**（通过模块和符号名）、**模块偏移断点**以及**临时断点**（命中后自动删除）。你还可以为断点注册 `BreakPointCallback` 回调，当命中时如果回调返回 `true`，调试器将不会进入交互模式，从而实现自动化处理。

### 📋 核心命令速查

调试器提供了一套丰富的命令集，按功能分类如下。

| 功能分类 | 命令 | 说明 |
| :--- | :--- | :--- |
| **执行控制** | `c` | **继续执行**，直到下一个断点 |
| | `n` | **单步跳过**，执行下一条指令 |
| | `s` / `si` | **单步步入**，执行单条指令 |
| | `s <N>` | 执行指定数量（N）的指令 |
| | `s <mnemonic>` | 执行直到指定的助记符（如 `s blx`），性能较低 |
| | `nb` | **在下一个基本块中断** |
| | `stop` | **停止模拟** |
| **断点管理** | `b` | 在当前 PC 地址添加断点 |
| | `b 0x<addr>` | 在指定地址或模块偏移添加断点 |
| | `blr` | 在链接寄存器（LR）的返回地址处添加断点 |
| | `r` | 删除当前 PC 处的断点 |
| | `vbs` | **查看所有断点** |
| **寄存器/内存查看** | `m<reg>` | 查看指定寄存器指向的内存（如 `mx0`，`mr0`） |
| | `m0x<addr>` | 查看指定地址的内存 |
| | `m<reg>s` | 以**空终止字符串**形式查看内存（如 `mx0s`） |
| | `m<reg>std` | 以 **C++ std::string** 形式查看内存（如 `mx0std`） |
| **寄存器/内存修改** | `w<reg> <value>` | 修改指定寄存器的值（如 `wx0 0x1234`） |
| | `wb/ws/wi/wl 0x<addr> <value>` | 向指定地址写入**字节/短整型/整型/长整型** |
| | `wx 0x<addr> <hex>` | 向指定地址写入十六进制字节序列 |
| **反汇编与查看** | `d` / `dis` | 显示当前 PC 处的寄存器和反汇编代码 |
| | `d 0x<addr>` | 反汇编指定地址处的代码 |
| | `bt` | **回溯堆栈**（调用栈展开，仅在运行时可用） |
| | `where` | 显示 Java 堆栈跟踪 |
| **追踪** | `trace [begin-end]` | **指令追踪**，输出到控制台或文件 |
| | `traceRead [begin-end]` | **内存读取追踪** |
| | `traceWrite [begin-end]` | **内存写入追踪** |
| **模块与状态** | `vm [filter]` | **查看已加载的模块**（可按名称或地址过滤） |
| | `threads` | 显示线程列表 |
| | `gc` | 运行 `System.gc()` |
| **内存搜索** | `st <hex>` | 在**栈**中搜索字节模式 |
| | `shw/shr/shx <hex>` | 在**可写/可读/可执行堆**中搜索字节模式 |
| **代码修补** | `p <assembly>` | **在 PC 地址修补汇编指令**（运行时打补丁） |
| | `cc <size>` | 将指定大小的汇编代码转换为 C 函数骨架 |

> **地址规则**：使用 `b 0x<addr>` 时，如果地址小于 `Memory.MMAP_BASE`（0x12000000），它会被视为相对于当前模块的偏移量。

### 🔍 核心功能详解

#### 1. 断点与执行控制

这是调试器最基本的功能。你可以在关键函数或指令处下断，然后通过 `c`、`n`、`s` 等命令控制执行流。`s` 命令非常灵活，支持执行指定数量的指令或直到某个助记符出现，这对于快速跳过循环或定位特定指令类型非常有用。

#### 2. 寄存器与内存的实时读写

调试器允许你直接读取和修改模拟 CPU 的状态。通过 `m` 系列命令可以查看寄存器和内存，`w` 系列命令则可以修改它们。例如，`wx0 0xdeadbeef` 可以将 x0 寄存器的值修改为 `0xdeadbeef`。这在进行算法还原或绕过简单校验时非常直接有效。

#### 3. 指令与内存追踪 (Tracing)

`trace` 系列命令让你无需编写 Hook 代码，就能在控制台直接开启追踪。

*   **指令追踪 (`trace`)**：记录指定范围内执行的每一条指令，并反汇编显示。
*   **内存读写追踪 (`traceRead` / `traceWrite`)**：记录对指定内存范围的读取和写入操作。
*   **输出重定向**：当追踪范围超过 `0x10000` 字节时，输出会自动重定向到 `target/` 目录下的文件，避免控制台被海量日志淹没。

#### 4. 运行时汇编修补 (Patching)

`p` 命令允许你在运行时直接修改指令，实现“热补丁”。它使用 Keystone 引擎将你输入的汇编指令转换为机器码，并写入当前 PC 地址，同时会验证新指令与原指令大小一致，防止代码损坏。

### 📊 平台差异：ARM32 vs ARM64

调试器的命令会根据架构有所不同，主要区别在于寄存器命名和部分命令的可用性。

| 特性 | ARM32 (`SimpleARMDebugger`) | ARM64 (`SimpleARM64Debugger`) |
| :--- | :--- | :--- |
| **寄存器名** | `r0-r7`, `fp`, `ip`, `sp`, `lr`, `pc` | `x0-x28`, `fp`, `sp`, `lr`, `pc` |
| **内存查看** | `mr0` - `mr7` | `mx0` - `mx28` |
| **寄存器写入** | `wr0` - `wr7` | `wx0` - `wx28` |
| **写入大小** | 字节、短整型、整型 | 字节、短整型、整型、**长整型** |
| **Thumb 支持** | **支持**（地址最低位为 1 表示 Thumb） | 不适用（始终为 A64） |

### 💡 使用建议与注意事项

*   **按需使用**：`Console Debugger` 适合**交互式探索**和**精确定位问题**。对于大规模的自动化追踪，建议使用 `traceCode` 等编程接口，它们性能更好且易于集成到脚本中。
*   **注意性能**：单步执行（`s`）和按助记符执行（`s <mnemonic>`）会严重拖慢执行速度，应仅在你需要精确观察执行路径时使用。
*   **善用 `vbs`**：在添加多个断点后，使用 `vbs` 命令查看所有断点列表，可以避免遗忘或地址冲突。
*   **与 Hook 框架互补**：`Console Debugger` 是“手动挡”，适合探索；而 `HookZz`、`Dobby` 等是“自动挡”，适合在确定逻辑后进行自动化干预。两者可以根据场景结合使用。





# 打印函数调用栈
hook目标函数，添加如下代码：
```java
emulator.getUnwinder().unwind();
```





# 监控内存读写
在 unidbg 中监控内存读写，主要有**三种递进的方法**：高层的 `traceRead`/`traceWrite` API、底层的 `ReadHook`/`WriteHook` 接口，以及硬件级的 `Watchpoint`。它们的精度、性能开销和适用场景各不相同。

### 🚀 方法一：traceRead / traceWrite（最便捷）

这是 unidbg 提供的最直接的内存监控方式，适合快速定位“谁读/写了这块内存”。

**基本用法**：在目标函数执行前，圈定要监控的地址范围。

```java
// 监控 0x10000 到 0x10100 范围内的所有内存读写
emulator.traceRead(0x10000L, 0x10100L);
emulator.traceWrite(0x10000L, 0x10100L);

// 执行目标函数
module.callFunction(emulator, offset, args);
```

**重定向输出到文件**：当日志量较大时，建议将追踪结果写入文件，避免淹没控制台。

```java
try (PrintStream traceStream = new PrintStream(new File("target/memory_trace.txt"))) {
    emulator.traceRead(0x10000L, 0x10100L).setRedirect(traceStream);
    emulator.traceWrite(0x10000L, 0x10100L).setRedirect(traceStream);
    module.callFunction(emulator, offset, args);
}
```

**输出格式**：每行会记录访问类型、地址、数据大小、数据值，以及关键的 **PC 和 LR 寄存器**，让你既能知道“哪块内存被访问”，也能定位到“哪条指令访问的、来自哪个调用”。

**核心技巧——圈定范围**：**全量 `traceRead()` 会把堆栈上所有局部变量的访问都打出来**，淹没真正关心的目标。因此，务必根据静态分析结果，精确圈定需要监控的地址段。

**典型应用场景**：
- **跟踪密钥从加载到使用的路径**：圈定密钥所在地址段，观察 round 函数依次访问了哪些字节。
- **S-Box 访问检测**：对 S-Box 地址段开启 `traceRead`，所有读取该区域的指令就是算法的“查表层”，对逆向 AES/RC4 等算法至关重要。
- **数据依赖反推**：用不同输入执行两次，对比内存写记录，输出地址相同但写入值不同的位置，就是与输入强相关的中间变量。

### 🔧 方法二：ReadHook / WriteHook（最灵活）

当需要更精细的控制——比如在 Hook 回调中读取寄存器、修改写入值、或根据条件过滤——可以直接实现 `ReadHook` 和 `WriteHook` 接口。

**实现一个内存追踪 Hook**：

```java
public class TraceMemoryHook implements ReadHook, WriteHook {
    private final PrintStream out;
    
    public TraceMemoryHook(PrintStream out) { this.out = out; }
    
    @Override
    public void hook(Backend backend, long address, int size, Object user) {
        // 内存读取回调：可以读取当前寄存器状态
        long pc = backend.reg_read(Arm64Const.UC_ARM64_REG_PC).longValue();
        byte[] data = backend.mem_read(address, size);
        out.printf("READ  at 0x%x, size=%d, PC=0x%x, data=%s%n",
                address, size, pc, bytesToHex(data));
    }
    
    @Override
    public void hook(Backend backend, long address, int size, long value, Object user) {
        // 内存写入回调：value 就是被写入的值
        long pc = backend.reg_read(Arm64Const.UC_ARM64_REG_PC).longValue();
        out.printf("WRITE at 0x%x, size=%d, value=0x%x, PC=0x%x%n",
                address, size, value, pc);
    }
}
```

**注册 Hook**：通过 `Memory` 接口的 `addHookListener` 方法注册。

```java
TraceMemoryHook memoryHook = new TraceMemoryHook(System.out);
emulator.getMemory().addHookListener(memoryHook);
```

**Hook 注册的底层机制**：上述 `addHookListener` 在底层会调用 `backend.hook_add_new()`，分别注册 `ReadHook` 和 `WriteHook` 回调。unidbg 内部会自动判断地址范围，只对命中区间的访问触发回调。

**适用场景**：需要在监控的同时修改寄存器或内存值、或根据 PC/LR 条件过滤的复杂场景。

### 🎯 方法三：Watchpoint（硬件级，最精准）

Watchpoint（观察点）利用 ARM 架构的**调试寄存器（WRPs）** 来监控内存访问，**不修改任何代码，性能开销极低**，且能精确到**字节级**。

**重要限制**：硬件观察点**目前仅在 Hypervisor 后端（macOS M1+）中可用**，KVM 后端和 Unicorn 后端不支持此功能。

**使用方式**：通过后端的 `addWatchpoint` 方法设置。

```java
// 监控 0x12359000 处 4 字节的写入操作
emulator.getBackend().addWatchpoint(0x12359000L, 4, 
    new WatchpointCallback() {
        @Override
        public void onWatch(Emulator<?> emulator, long address, int size, boolean isWrite) {
            System.out.printf("Watchpoint hit: %s at 0x%x, size=%d%n",
                    isWrite ? "WRITE" : "READ", address, size);
            // 可以在这里读取寄存器，或触发断点
        }
    });
```

**适用场景**：需要对特定内存地址进行**长期、低开销**的监控，且使用 Apple Silicon Mac 进行开发时。

### 🖥️ 方法四：控制台调试器命令（交互式）

在 `Console Debugger` 交互模式下，可以直接使用以下命令进行内存监控：

| 命令 | 作用 | 示例 |
| :--- | :--- | :--- |
| `traceRead [begin-end]` | 追踪内存读取 | `traceRead 0x10000 0x10100` |
| `traceWrite [begin-end]` | 追踪内存写入 | `traceWrite 0x12359000 0x12359100` |

这些命令适合在分析过程中**临时、快速地**查看某段内存的访问情况。

### 📊 三种方式对比与选型

| 维度 | **traceRead/traceWrite** | **ReadHook/WriteHook** | **Watchpoint** |
| :--- | :--- | :--- | :--- |
| **易用性** | ★★★★★ 一行代码 | ★★★ 需实现接口 | ★★★★ 需后端支持 |
| **控制粒度** | 地址范围级 | 地址范围级 + 回调逻辑 | 精确到字节 |
| **性能开销** | 中等（每次访问回调） | 中等 | **极低**（硬件辅助） |
| **后端支持** | Unicorn/Unicorn2 | Unicorn/Unicorn2 | **仅 Hypervisor** |
| **可修改访问行为** | 否 | **是**（可在 Hook 中修改） | 否 |
| **适用场景** | 快速定位内存访问 | 需要过滤、修改、读取寄存器 | Apple Silicon 低开销监控 |

### ⚠️ 关键注意事项

1. **后端限制**：`traceRead`/`traceWrite` 和 `ReadHook`/`WriteHook` **仅在 Unicorn/Unicorn2 后端上可用**。Dynarmic 的 JIT 执行方式无法暴露每条 load/store 的事件回调。
2. **范围必须圈定**：**绝对不要全量开启 `traceRead()`**。全量追踪会把堆栈上所有局部变量访问都打出来，产生海量日志，性能也会严重下降。
3. **输出重定向**：当日志量较大时（`end - begin > 0x10000`），建议使用 `setRedirect()` 将输出写入文件，避免控制台被淹没。
4. **Watchpoint 后端限制**：使用前务必确认后端类型，Hypervisor 后端仅在 macOS M1+ 上可用。

**选型建议**：**快速分析用 `traceRead`/`traceWrite`，需要精细控制用 `ReadHook`/`WriteHook`，长期低开销监控（Apple Silicon）用 `Watchpoint`**。






# trace
`trace` 是 unidbg 的核心能力，它让你能“看见”SO 内部究竟发生了什么。与 Frida 等真机 Hook 工具相比，unidbg 的 trace 完全运行在虚拟环境内，**零干扰、更底层、且完全可控**。unidbg 的 trace 体系分为**指令级、内存级、函数级**三个层次，粒度从细到粗，选择哪个层次取决于你想观察的“分辨率”。

### 🏗️ Trace 的三个层次

#### 指令级 Trace：最底层的“逐帧播放”

指令级 trace 是 unidbg 最强大的能力，它记录**每一条被执行的 ARM/ARM64 指令**，相当于对整个执行过程进行“逐帧播放”。

**开启方式**非常简单，在调用目标函数前开启即可：
```java
// 简单粗暴：追踪后续所有指令
emulator.traceCode();

// 推荐：精确圈定范围，只追踪目标 SO 的代码段
Module module = dm.getModule();
emulator.traceCode(module.base, module.base + module.size);
```
> **注意**：`traceCode(begin, end)` 的第二个参数是**结束地址**，不是长度。

**输出格式**是理解 trace 的关键，每一行包含：
```
[14:32:01 256ms] [libxxx.so][0x00001234] 0x40001234: ldr w8, [x0, #0x10] x0=0x70000000 x8=0x12345678
```
| 列 | 含义 |
|:---|:---|
| `[时间戳]` | 事件顺序，排查偶发分支问题时有用 |
| `[模块+偏移]` | **最重要的一列**，可直接在 IDA 中双击跳转对照 |
| `绝对地址` | Unicorn 视角的虚拟地址 |
| `反汇编` | Capstone 反汇编结果，包含助记符和操作数 |
| `寄存器变化` | 只打印**被这条指令读写**的寄存器，保持紧凑 |

**条件标志**：ARM64 的 `nzcv` 标志也会被打印，例如 `nzcv=0x60000000` 中 `6=0110` 表示 `Z=1, C=1`，即“相等且无借位”，这对分析条件跳转至关重要。

#### 内存级 Trace：关注“数据流”

当你只关心**特定内存区域**的读写时，内存级 trace 比指令级 trace 高效得多——**一行一个事件**，没有大量无关计算指令的噪声。

```java
// 圈定要监控的内存地址范围
emulator.traceRead(0x10000L, 0x10100L);
emulator.traceWrite(0x10000L, 0x10100L);

// 日志量大时重定向到文件
try (PrintStream traceStream = new PrintStream(new File("target/memory_trace.txt"))) {
    emulator.traceRead(0x10000L, 0x10100L).setRedirect(traceStream);
    emulator.traceWrite(0x10000L, 0x10100L).setRedirect(traceStream);
    module.callFunction(emulator, offset, args);
}
```
**输出格式**每行都带 **PC 和 LR**，让你既能知道“哪块内存被访问”，也能定位“哪条指令访问的、来自哪个调用”。

**典型应用**：追踪密钥的加载和使用路径、检测 S-Box 查表操作、通过 diff 两次执行的写入记录反推数据依赖。

#### 函数级 Trace：宏观“调用树”

函数级 trace 记录函数的**调用和返回**，粒度最粗，适合快速把握整体逻辑。使用 `traceFunctionCall` 方法：

```java
Debugger debugger = emulator.attach();
debugger.traceFunctionCall(module, new FunctionCallListener() {
    @Override
    public void onCall(Emulator<?> emulator, long address, FunctionCall call) {
        System.out.println("CALL: " + call);
    }
});
```
`traceFunctionCall` 是 unidbg 0.9.6 后新增的功能，通过监听 `BL`/`BLR` 指令来追踪函数调用关系。它还能**持久化日志**，将追踪结果写入文件便于后续分析。

### 📊 三个层次对比与选型

| 层次 | 粒度 | 输出量级（100ms encrypt） | 适用场景 |
|:---|:---|:---|:---|
| **指令级** | 每条指令 | ~几十万行 | 精确分析算法、反混淆、定位关键指令 |
| **内存级** | 每次内存访问 | ~几千到几万行 | 追踪数据流、密钥、S-Box |
| **函数级** | 每次函数调用 | ~几百行 | 宏观调用树、快速定位关键函数 |

**选型原则**：从粗到细。先用**函数级** trace 定位到关键函数，再用**内存级** trace 追踪数据流，最后用**指令级** trace 精读算法细节。

### ⚡ 性能优化：魔改 trace 的 L1/L2 缓存

unidbg 原版 trace 的性能瓶颈在于**每条指令都要调用一次 Capstone 反汇编**。对于 OLLVM/VMP 保护，指令膨胀会让这个问题急剧恶化——VMP 把每条原始指令翻译成数十条 VM Handler 指令，trace 次数可达**上亿级别**。

社区魔改版的核心优化是引入 **L1/L2 两级缓存**：
- **L2 缓存**：`HashMap` 存储“指令编码 → 反汇编结果”
- **L1 缓存**：**数组**，用指令地址取模直接定位，省去 HashMap 的哈希计算开销

实测在 OLLVM+VMP 场景下，速度提升可达**一百多倍**——需要几小时的 trace 可以压缩到几分钟。

### 🔍 如何分析 Trace 数据

**1. 与 IDA CFG 交叉对照**

IDA 的控制流图（CFG）是**所有可能的路径**，而 trace 是**本次执行的实际路径**。从 trace 中提取所有执行过的 basic block 入口地址，用 IDAPython 在 IDA 中给这些 block 标红，未染色的就是本次未走的分支。对 OLLVM 控制流平坦化样本，这几乎是“救命”的——静态看几百个 case 密密麻麻，实际执行只走了 5-10 个。

**2. Diff 两次 Trace**

用不同输入执行两次，对比 trace 差异。输出地址相同但值不同的位置，就是与输入强相关的**关键中间变量**。

**3. 可视化工具**

社区有 **UnidbgTraceViewer** 等工具，支持**值流反向追踪**（追踪寄存器值的来源）、**前向污点分析**（追踪数据传播路径）和**内存写入对比**等高级分析功能。

### ⚠️ 注意事项

- **后端限制**：**所有 Trace 功能都仅在 Unicorn/Unicorn2 后端可用**。Dynarmic 的 JIT 执行方式无法暴露每条指令/内存访问的事件回调。
- **圈定范围**：**全量 `traceCode()` 或 `traceRead()` 会产生海量日志**，性能也会严重下降。务必精确圈定地址范围。
- **输出重定向**：当日志量较大时，使用 `setRedirect()` 将输出写入文件，避免控制台被淹没。

### 💎 总结

unidbg 的 trace 体系是**从“黑盒”到“白盒”的桥梁**。**函数级**让你看到宏观调用关系，**内存级**让你追踪数据流动，**指令级**让你精读每一行汇编。选对层次、圈好范围、善用分析工具，就能从百万行 trace 中高效捞出关键信息。






# 日志系统
## 一、引言

本篇简单讨论 Unidbg 中如何控制日志输出。

## 二、日志系统

日志反映了软件内部的运行情况，对于 Unidbg 这样的模拟器项目，其内部运行情况直接反映了样本和系统环境的交互以及对系统的影响，其重要性不言而喻。Unidbg 中使用 Apache 的开源项目 [log4j](https://mvnrepository.com/artifact/log4j/log4j/1.2.17) 和 [commons-logging](https://mvnrepository.com/artifact/commons-logging/commons-logging/1.2) 处理日志，就像其他很多 JAVA 项目那样。

Unidbg 的绝大多数代码逻辑都提供了信息展示，但只在`DEBUG`日志等级下才做输出打印。

```java
@Override
public long call(Emulator<?> emulator) {
    if (address == 0 || address == -1) {
        return address;
    }

    if (log.isDebugEnabled()) {
        log.debug("[" + libName + "]CallInitFunction: 0x" + Long.toHexString(address));
    }
    long start = System.currentTimeMillis();
    emulator.eFunc(getAddress());
    if (log.isDebugEnabled()) {
        System.err.println("[" + libName + "]CallInitFunction: 0x" + Long.toHexString(address) + ", offset=" + (System.currentTimeMillis() - start) + "ms");
    }
    return address;
}
```

Unidbg 基于模块去管理输出，想了解哪部分日志，就指定具体的类为 DEBUG 等级。

```java
import org.apache.log4j.Level;
import org.apache.log4j.Logger;

public static void main(String[] args) {
    NetWork nw = new NetWork();
    Logger.getLogger(ARM32SyscallHandler.class).setLevel(Level.DEBUG);
    Logger.getLogger(AndroidSyscallHandler.class).setLevel(Level.DEBUG);
    String result = nw.callSign();
    System.out.println("call s result:"+result);
}
```

比如上面的代码，就是对 ARM32 下的系统调用类、系统调用父类做日志输出。

![](https://cdn.nlark.com/yuque/0/2022/png/707476/1665709432850-0ae4c579-112f-4d53-93dd-42e6564b572b.png)

效果很不错，样本在执行流中所触发的系统调用尽收眼底。

如果你希望输出所有模块的日志，可以修改`unidbg-android/src/test/resources/log4j.properties`文件，将 unidbg 的日志等级从 INFO 改为 DEBUG。

```java
# 修改前
log4j.logger.com.github.unidbg=INFO
# 修改后
log4j.logger.com.github.unidbg=DEBUG
```

那么不管是系统调用、指针管理、多线程，还是多线程处理器等等，所有组件的日志都会打印出来。

![](https://cdn.nlark.com/yuque/0/2022/png/707476/1665709171576-1f69fbfe-6318-4034-9483-e821bae6c6ba.png)

但我们一般不会这么做，过多过杂的日志输出对使用者并无益处，还会拖累 Unidbg 执行的速度。因此在更多的情况下，我们只会专项打开。

## 三、虚拟机日志

除了常规日志，Unidbg 还有另一套日志输出，主要打印 JNI 、Syscall 调用相关的内容。它和常规日志的输出有重叠，但内容更详细一些。

以 \_ToReflectedMethod 这个 JNI 函数为例

```java
Pointer _ToReflectedMethod = svcMemory.registerSvc(new ArmSvc() {
    @Override
    public long handle(Emulator<?> emulator) {
        RegisterContext context = emulator.getContext();
        UnidbgPointer clazz = context.getPointerArg(1);
        UnidbgPointer jmethodID = context.getPointerArg(2);
        DvmClass dvmClass = classMap.get(clazz.toIntPeer());
        DvmMethod dvmMethod = null;
        if (dvmClass != null) {
            dvmMethod = dvmClass.getStaticMethod(jmethodID.toIntPeer());
            if (dvmMethod == null) {
                dvmMethod = dvmClass.getMethod(jmethodID.toIntPeer());
            }
        }
        // 常规日志输出
        if (log.isDebugEnabled()) {
            log.debug("ToReflectedMethod clazz=" + dvmClass + ", jmethodID=" + jmethodID + ", lr=" + context.getLRPointer());
        }
        if (dvmMethod == null) {
            throw new BackendException();
        } else {
            // JNI 日志输出
            if (verbose) {
                System.out.printf("JNIEnv->ToReflectedMethod(%s, %s, %s) was called from %s%n", dvmClass.getClassName(), dvmMethod.methodName, dvmMethod.isStatic ? "is static" : "not static", context.getLRPointer());
            }

            return addLocalObject(dvmMethod.toReflectedMethod());
        }
    }
});
```

我们通过 vm.setVerbose 开启或关闭它。

```java
public WeiBo() {
    emulator = AndroidEmulatorBuilder
        .for32Bit()
        .addBackendFactory(new Unicorn2Factory(true))
        .setProcessName("com.weico.international")
        .build();
    Memory memory = emulator.getMemory();
    memory.setLibraryResolver(new AndroidResolver(23));
    vm = emulator.createDalvikVM(new File("unidbg-android/src/test/resources/weibo/sinaInternational.apk"));
    vm.setJni(this);
    // 设置是否打印以 JNI 为主的虚拟机调用细节
    vm.setVerbose(false);
    DalvikModule dm = vm.loadLibrary("utility", true);
    WeiboSecurityUtils = vm.resolveClass("com/sina/weibo/security/WeiboSecurityUtils");
    dm.callJNI_OnLoad(emulator);
}
```

**在一般情况下，我们都会开启这个日志。**

## 四、对比

在整体上，Unidbg 提供了一种按组件和模块控制输出与否的日志系统，使用体验还不错。另一种常见的日志控制方式是按等级输出。比如 qiling。

下面是 qiling 初始化实例的代码，其中的 verbose 字段就是日志等级。

```java
ql = Qiling([r'rootfs/arm_linux/bin/arm_hello'], r'rootfs/arm_linux', verbose=QL_VERBOSE.DUMP)
```

存在下面这些选项

```java
class QL_VERBOSE(IntEnum):
    DISABLED = -1 # 禁止所有输出
    OFF = 0       # 输出警告信息
    DEFAULT = 1   # 输出警告信息和模拟器的执行信息，是默认的输出等级
    DEBUG = 4     # 在上述基础上增加系统调用的执行信息
    DISASM = 10   # 在上述基础上增加汇编执行流信息
    DUMP = 20     # 在上述基础上增加执行流中的寄存器信息
```

DISABLED 和 OFF 下几乎不会有日志输出，DEFAULT 是默认的输出等级。

![](https://cdn.nlark.com/yuque/0/2022/png/707476/1665710476252-db697909-0c7f-420e-98d0-74e4b498c098.png)

DEBUG 等级会带来更详细的输出，以系统调用为例，参数和返回值的展示更详细了。

![](https://cdn.nlark.com/yuque/0/2022/png/707476/1665711561122-fc7ef845-768e-439e-aad7-c86a9b164c6a.png)

除此之外，在运行完毕后，在 Debug 等级下还会有系统调用的整体“总结”。

![](https://cdn.nlark.com/yuque/0/2022/png/707476/1665711651056-0e5965a1-f8eb-4434-a58e-6cabc61bfbff.png)

以及执行流中所涉及到的字符串

![](https://cdn.nlark.com/yuque/0/2022/png/707476/1665711711957-b6d340a3-968d-4636-a5e0-e1c926545857.png)

需要注意，这并不是在内存中启发式的搜索所有可见字符串，而仅仅是记录了系统调用里的使用到的字符串参数。

这两个功能，其实是 Qiling 试图表现出具备沙箱的属性和能力。虽然很简陋，但确实有沙箱汇总和自动化分析的那个味儿。Unidbg 在这方面其实有所欠缺，没有提供一种对自身执行信息统计和自省的机制。

在 DISASM 等级下，会打印动态反汇编，这在 Unidbg 中由 traceCode 模块处理，而非由日志控制。

```java
[=] 	047d2608 [ld-linux.so.3        + 0x018608]  03 30 8f 10          addne                r3, pc, r3
[=] 	047d260c [ld-linux.so.3        + 0x01860c]  e8 21 83 15          strne                r2, [r3, #0x1e8]
[=] 	047d2610 [ld-linux.so.3        + 0x018610]  00 00 51 e3          cmp                  r1, #0
[=] 	047d2614 [ld-linux.so.3        + 0x018614]  b0 31 9f 15          ldrne                r3, [pc, #0x1b0]
[=] 	047d2618 [ld-linux.so.3        + 0x018618]  2c 20 9d 15          ldrne                r2, [sp, #0x2c]
[=] 	047d261c [ld-linux.so.3        + 0x01861c]  03 30 8f 10          addne                r3, pc, r3
[=] 	047d2620 [ld-linux.so.3        + 0x018620]  04 20 83 15          strne                r2, [r3, #4]
[=] 	047d2624 [ld-linux.so.3        + 0x018624]  29 00 00 ea          b                    #0x47d26d0
[=] 	047d26d0 [ld-linux.so.3        + 0x0186d0]  f8 70 9f e5          ldr                  r7, [pc, #0xf8]
```

在 DUMP 模式下，会打印寄存器信息。

```java
[=] 	047bb21c [ld-linux.so.3        + 0x00121c]  03 21 80 97          strls                r2, [r0, r3, lsl #2]
[=] 	r0         : 0x47eb588
[=] 	r1         : 0xa
[=] 	r2         : 0x47eaf94
[=] 	r3         : 0x17
[=] 	r4         : 0x47ba000
[=] 	r5         : 0xc47eb288
[=] 	r6         : 0x47eb650
[=] 	r7         : 0x6ffffdff
[=] 	r8         : 0x6ffffeff
[=] 	r9         : 0x0
[=] 	r10        : 0x30584
[=] 	r11        : 0x7ff3ceec
[=] 	r12        : 0x47eb588
[=] 	sp         : 0x7ff3cea0
[=] 	lr         : 0xc47eae58
[=] 	pc         : 0x47bb21c
[=] 	cpsr       : 0x800001d3
[=] 	c1_c0_2    : 0x0
[=] 	c13_c0_3   : 0x0
[=] 	fpexc      : 0x40000000
[=] 	d0         : 0x0
[=] 	d1         : 0x0
[=] 	d2         : 0x0
[=] 	d3         : 0x0
[=] 	d4         : 0x0
[=] 	d5         : 0x0
[=] 	d6         : 0x0
[=] 	d7         : 0x0
[=] 	d8         : 0x0
[=] 	d9         : 0x0
[=] 	d10        : 0x0
[=] 	d11        : 0x0
[=] 	d12        : 0x0
[=] 	d13        : 0x0
[=] 	d14        : 0x0
[=] 	d15        : 0x0
[=] 	d16        : 0x0
[=] 	d17        : 0x0
[=] 	d18        : 0x0
[=] 	d19        : 0x0
[=] 	d20        : 0x0
[=] 	d21        : 0x0
[=] 	d22        : 0x0
[=] 	d23        : 0x0
[=] 	d24        : 0x0
[=] 	d25        : 0x0
[=] 	d26        : 0x0
[=] 	d27        : 0x0
[=] 	d28        : 0x0
[=] 	d29        : 0x0
[=] 	d30        : 0x0
[=] 	d31        : 0x0
[=] 	fpscr      : 0x0
```

它并非是打印当前汇编代码可能影响的寄存器，而是遍历打印所有寄存器。

除了常规日志打印外，Qiling 还支持对日志的过滤打印。初始化 qiling 实例时给`filter`字段赋值即可，它支持正则表达式。

比如下面的代码，只展示包含`stat64`字眼的日志。

```java
ql = Qiling([r'rootfs/arm_linux/bin/arm_hello'], r'rootfs/arm_linux', verbose=QL_VERBOSE.DEBUG)
ql.filter = r"stat64"
```

Unidbg 和 Qiling 在日志输出上各有优劣。 Qiling 没有提供对各个组件模块的单独控制的日志处理，控制力度不够细。

但它相对灵活简单，提供了很多有趣的小玩意，比如 Sycall Summary、Strings Occurrences 、Log Filter 等。

## 五、总结

他山之石，可以攻玉。Function Summary 移植到 Unidbg 中，会十分有助于比对、查验 JNI 调用、以及监控观察。