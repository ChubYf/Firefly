---
title: Capstone,Keystone,Unicorn三剑客
published: 2026-09-21
description: Capstone,Keystone,Unicorn的简单学习
image: ./cover.jpg
tags: [逆向]
category: unidbg
draft: false
---


# 简介
Capstone、Keystone 和 Unicorn 是由同一核心团队（主要来自新加坡南洋理工大学）开发的三大开源二进制分析框架，在逆向工程与安全研究领域常被合称为“逆向三剑客”。它们分工明确，分别解决**反汇编**、**汇编**和**模拟执行**三个核心问题。

### 🧩 Capstone：反汇编框架

Capstone 是一个轻量级、多平台、多架构的**反汇编框架**，目标是为二进制分析与逆向工程领域提供终极的反汇编引擎。

**核心特性**：
- **多架构支持**：覆盖 ARM、ARM64、MIPS、PowerPC、RISC-V、SPARC、SystemZ、X86（16/32/64位）等众多架构。
- **丰富的指令信息**：除了将机器码转换为汇编指令，还能提供指令的语义信息，例如隐式读取和写入的寄存器列表。
- **纯 C 语言实现**：性能高，并提供 Python、Java、Go、Rust 等数十种语言的绑定。
- **线程安全**：设计上支持多线程环境，适合嵌入固件或操作系统内核。
- **开源许可**：采用宽松的 BSD 许可证。

**应用场景**：被 IDA Pro、Ghidra、Radare2 等主流逆向工具广泛采用作为底层反汇编引擎。

### ⚙️ Keystone：汇编框架

Keystone 是一个轻量级、多平台、多架构的**汇编框架**，功能与 Capstone 正好相反——将人类可读的汇编指令**汇编**为机器码。

**核心特性**：
- **多架构支持**：覆盖 Arm、Arm64、Hexagon、Mips、PowerPC、RISC-V、Sparc、SystemZ、X86 等。
- **基于 LLVM**：底层使用 LLVM 的 MC 组件实现，但体量和内存占用比完整 LLVM 小 10 倍以上。
- **多语言绑定**：提供 Java、Python、NodeJS、Ruby、Go、Rust 等绑定。
- **开源**：采用开源许可证发布。

**应用场景**：常用于漏洞利用开发（如生成 shellcode）、ROP gadget 搜索，以及 IDA Pro 的 Keypatch 插件等工具中。

### 🖥️ Unicorn：CPU 模拟器框架

Unicorn 是一个轻量级、多平台、多架构的 **CPU 模拟器框架**，专注于模拟 CPU 指令的执行，而不模拟完整的硬件外设或操作系统环境。

**核心特性**：
- **基于 QEMU**：复用 QEMU 的 CPU 模拟代码，但更加轻量，聚焦于 CPU 指令仿真。
- **多架构支持**：ARM、ARM64、M68K、MIPS、PowerPC、RISC-V、S390x、SPARC、TriCore、X86（16/32/64位）。
- **JIT 编译**：使用即时编译技术获得高性能。
- **细粒度插桩**：支持在指令执行、内存访问等层面进行精细的钩子（hook）操作。
- **多语言绑定**：提供 Python、Java、Go、Rust、.NET 等绑定。
- **开源许可**：采用 GPLv2 许可证。

**应用场景**：广泛用于恶意软件分析、固件仿真、漏洞挖掘中的动态执行，以及构建更高级的仿真框架（如 Qiling）。

### 🔗 三者的协同关系

这三个框架常被组合使用，构成一个完整的二进制分析工作流。一个形象的比喻是：

- **Capstone 是“眼睛”**：负责将机器码翻译成人类可读的汇编代码，帮助分析人员“看懂”二进制程序在做什么。
- **Keystone 是“手”**：负责将汇编代码（或修改后的指令）重新汇编为机器码，用于“修改”或“构造”二进制程序。
- **Unicorn 是“大脑”**：负责实际执行机器码，模拟 CPU 的状态流转和内存交互，用于“运行”和“验证”程序行为。

因此，一个典型的协同场景是：用 **Capstone** 反汇编目标二进制，分析其逻辑；用 **Keystone** 修改或注入新的汇编指令；最后用 **Unicorn** 在受控环境中模拟执行修改后的代码，观察其效果。Radare2 等逆向框架正是通过集成这三者，提供了完整的“反汇编—修改—模拟”能力。




# capstone, keystone基本使用
Capstone 和 Keystone 的基本使用遵循相似的流程：**安装 → 导入模块 → 初始化引擎（指定架构和模式）→ 调用核心方法处理数据**。下面分别给出两者的详细用法和代码示例。

---

## Capstone：反汇编（机器码 → 汇编指令）

### 安装

```bash
pip install capstone
```

### 基本用法

Capstone 的核心流程只有两步：初始化 `Cs` 对象，然后调用 `disasm()` 方法。

```python
from capstone import *

# 待反汇编的机器码
CODE = b"\x55\x48\x8b\x05\xb8\x13\x00\x00"

# 1. 初始化：指定架构（X86）和模式（64位）
md = Cs(CS_ARCH_X86, CS_MODE_64)

# 2. 反汇编：disasm(机器码, 基地址)
for insn in md.disasm(CODE, 0x1000):
    print(f"0x{insn.address:x}:\t{insn.mnemonic}\t{insn.op_str}")
```

输出：
```
0x1000:  push  rbp
0x1001:  mov   rax, qword ptr [rip + 0x13b8]
```

`disasm()` 的第二个参数是第一条指令的**基地址**，只影响输出中显示的地址，不影响反汇编逻辑。返回的每个 `insn` 对象包含以下常用字段：

| 字段 | 说明 |
|------|------|
| `address` | 指令地址 |
| `mnemonic` | 助记符（如 `mov`、`push`） |
| `op_str` | 操作数字符串 |
| `size` | 指令字节长度 |
| `bytes` | 原始机器码 |

### 切换架构和模式

只需修改 `Cs` 构造函数的两个参数即可：

```python
md = Cs(CS_ARCH_ARM, CS_MODE_ARM)          # ARM 模式
md = Cs(CS_ARCH_ARM, CS_MODE_THUMB)        # ARM Thumb 模式
md = Cs(CS_ARCH_MIPS, CS_MODE_MIPS32)      # MIPS32
md = Cs(CS_ARCH_X86, CS_MODE_32)           # X86 32位
md = Cs(CS_ARCH_PPC, CS_MODE_32)           # PowerPC 32位
```

大端序可以通过 `+` 组合额外模式，例如 Mips64 小端序：`CS_ARCH_MIPS + CS_MODE_MIPS64 + CS_MODE_LITTLE_ENDIAN`。

### 开启详细模式

默认情况下，Capstone 只输出助记符和操作数字符串。如果需要获取**隐式读写寄存器、语义分组、操作数类型**等深层信息，需手动开启 `detail` 模式：

```python
md = Cs(CS_ARCH_X86, CS_MODE_64)
md.detail = True

CODE = b"\x48\x8B\x05\xB8\x13\x00\x00"  # mov rax, [rip+0x13b8]
for insn in md.disasm(CODE, 0x1000):
    print(f"{insn.mnemonic} {insn.op_str}")
    for op in insn.operands:
        if op.type == X86_OP_MEM:
            mem = op.value.mem
            print(f"  内存操作数: 基址={md.reg_name(mem.base)} 偏移=0x{mem.disp:x}")
    # 隐式读取的寄存器
    print(f"  隐式读取: {[md.reg_name(r) for r in insn.regs_read]}")
    print(f"  隐式写入: {[md.reg_name(r) for r in insn.regs_write]}")
```

开启 detail 模式会占用更多内存并略微降低速度，仅在需要深层语义信息时使用。

### 其他常用选项

```python
md.syntax = CS_OPT_SYNTAX_ATT      # 切换为 AT&T 语法（默认 Intel）
md.skipdata = True                 # 遇到无法反汇编的字节时跳过，而非停止
md.skipdata_setup = ("db", None, None)  # 跳过的数据以 .byte 伪指令形式输出
```

`disasm_lite()` 是一个更轻量的 API，只返回 `(address, size, mnemonic, op_str)` 元组，速度比 `disasm()` 快约 30%，适合仅需基础信息的场景。

---

## Keystone：汇编（汇编指令 → 机器码）

### 安装

```bash
pip install keystone-engine
```

### 基本用法

Keystone 的流程与 Capstone 对称：初始化 `Ks` 对象，然后调用 `asm()` 方法。

```python
from keystone import *

# 待汇编的汇编代码
CODE = b"INC ECX; ADD EDX, ECX"

try:
    # 1. 初始化：指定架构（X86）和模式（64位）
    ks = Ks(KS_ARCH_X86, KS_MODE_64)

    # 2. 汇编：asm(汇编代码)
    encoding, count = ks.asm(CODE)

    # encoding 是机器码整数的列表，count 是指令数量
    print(f"指令数量: {count}")
    print(f"机器码(Hex): {''.join(f'{x:02x}' for x in encoding)}")
except KsError as e:
    print(f"汇编错误: {e}")
```

输出：
```
指令数量: 2
机器码(Hex): ffc101ca
```

`asm()` 返回一个元组：第一个元素是机器码的**整数列表**，第二个元素是成功汇编的**指令数量**。

### 将汇编结果转为字节串

Keystone 返回的是整数列表，常用于写入内存或生成 shellcode，需要转换为 `bytes`：

```python
encoding, count = ks.asm(CODE)
shellcode = bytes(encoding)
print(shellcode)  # b'\xff\xc1\x01\xca'
```

### 切换架构和模式

与 Capstone 完全一致，只需替换常量前缀 `KS_`：

```python
ks = Ks(KS_ARCH_ARM, KS_MODE_ARM)     # ARM 模式
ks = Ks(KS_ARCH_ARM, KS_MODE_THUMB)   # ARM Thumb 模式
ks = Ks(KS_ARCH_MIPS, KS_MODE_MIPS32) # MIPS32
ks = Ks(KS_ARCH_X86, KS_MODE_32)      # X86 32位
```

### 带地址的汇编

如果汇编代码中包含跳转标签，可以传入基地址让 Keystone 正确计算跳转偏移：

```python
ASM_CODE = """
loop:
    inc eax
    cmp eax, 10
    jl loop
"""
ks = Ks(KS_ARCH_X86, KS_MODE_32)
encoding, count = ks.asm(ASM_CODE, addr=0x1000)  # 指定代码起始地址
```

### 错误处理

Keystone 在遇到非法指令或语法错误时抛出 `KsError`。推荐用 `try/except` 包裹 `asm()` 调用：

```python
try:
    encoding, count = ks.asm(CODE)
except KsError as e:
    print(f"汇编失败: {e}")
```

---

## 两者协同的典型模式

Capstone 和 Keystone 的组合使用非常自然：**Keystone 汇编出机器码，Capstone 反汇编验证结果**，形成一个闭环。

```python
from capstone import *
from keystone import *

# 1. 用 Keystone 汇编
asm_code = b"mov eax, 0x50; push rax"
ks = Ks(KS_ARCH_X86, KS_MODE_64)
machine_code, _ = ks.asm(asm_code)
machine_code = bytes(machine_code)
print(f"汇编结果: {machine_code.hex()}")

# 2. 用 Capstone 反汇编验证
md = Cs(CS_ARCH_X86, CS_MODE_64)
for insn in md.disasm(machine_code, 0x1000):
    print(f"0x{insn.address:x}: {insn.mnemonic} {insn.op_str}")
```

输出：
```c
汇编结果: b85000000050
0x1000: mov eax, 0x50
0x1005: push rax
```

这个模式在漏洞利用开发（生成 shellcode 后验证）、ROP gadget 构造，以及二进制补丁中非常常用。



# unicorn使用
[原创深入浅出 Unicorn 框架学习-Android安全-看雪安全社区｜专业技术交流与安全研究论坛](https://bbs.kanxue.com/thread-289502.htm#msg_header_h3_3)
## unicorn初始化
使用 Python 的 Unicorn 引擎进行初始化，核心是创建一个 `Uc` 类的实例，并指定要模拟的 CPU 架构和模式。下面是完整的初始化流程与代码示例。

### 1. 安装 Unicorn

首先，需要通过 `pip` 安装 Unicorn 的 Python 绑定库。

```bash
pip install unicorn
```

安装完成后，可以在 Python 中运行 `import unicorn` 来验证是否成功。

### 2. 核心初始化步骤

初始化过程主要包含以下几步：

#### 导入必要的模块

通常需要导入核心模块以及特定架构的常量模块。

```python
from unicorn import *
from unicorn.x86_const import *  # 如果模拟x86架构
```

#### 创建模拟器实例 (`Uc` 对象)

这是最关键的一步，通过 `Uc` 类构造函数指定**硬件架构**和**模式**。

```python
# 初始化一个模拟 x86-32 位的引擎
mu = Uc(UC_ARCH_X86, UC_MODE_32)

# 或者，初始化一个模拟 x86-64 位的引擎
# mu = Uc(UC_ARCH_X86, UC_MODE_64)

# 也可以初始化 ARM 或 ARM64 架构
# mu = Uc(UC_ARCH_ARM, UC_MODE_ARM)
# mu = Uc(UC_ARCH_ARM64, UC_MODE_ARM)
```

`Uc` 类接受两个主要参数：**硬件架构**（如 `UC_ARCH_X86`, `UC_ARCH_ARM`, `UC_ARCH_ARM64`, `UC_ARCH_MIPS` 等）和 **CPU 模式**（如 `UC_MODE_32`, `UC_MODE_64`, `UC_MODE_ARM`）。

#### 映射内存

模拟器需要一个可用的内存空间来存放和执行代码。使用 `mem_map` 方法映射一段内存，需要指定起始地址和大小（以字节为单位）。

```python
ADDRESS = 0x1000
SIZE = 2 * 1024 * 1024  # 2MB

# 映射内存，默认权限为可读、可写、可执行
mu.mem_map(ADDRESS, SIZE)

# 也可以显式指定内存权限
# mu.mem_map(ADDRESS, SIZE, UC_MEM_READ | UC_MEM_WRITE | UC_MEM_EXEC)
```

`mem_map(address, size, perm)` 方法中，`perm` 参数是可选的，用于设置内存的读、写、执行权限。

#### 写入待模拟的机器码

将需要模拟执行的二进制机器码写入刚刚映射的内存地址。

```python
# 示例机器码：INC ecx; DEC edx (x86)
X86_CODE32 = b"\x41\x4a"

mu.mem_write(ADDRESS, X86_CODE32)
```

#### 设置寄存器初始值（可选）

在开始执行前，可以通过 `reg_write` 方法设置寄存器的初始值。

```python
mu.reg_write(UC_X86_REG_ECX, 0x1234)
mu.reg_write(UC_X86_REG_EDX, 0x7890)
```

#### 启动模拟执行

使用 `emu_start` 方法开始模拟执行。需要指定代码的起始地址和结束地址。

```python
mu.emu_start(ADDRESS, ADDRESS + len(X86_CODE32))
```

`emu_start(begin, until, timeout=0, count=0)` 方法中，`begin` 是开始执行的地址，`until` 是停止执行的地址。`timeout` 和 `count` 是可选参数，分别用于限制执行时间和指令数量。

#### 读取执行后的状态

执行结束后，可以读取寄存器或内存的值来检查结果。

```python
r_ecx = mu.reg_read(UC_X86_REG_ECX)
r_edx = mu.reg_read(UC_X86_REG_EDX)
print(f"ECX = 0x{r_ecx:x}")
print(f"EDX = 0x{r_edx:x}")
```

### 3. 完整初始化示例

将以上步骤组合起来，就是一个完整的初始化及执行流程。

```python
from unicorn import *
from unicorn.x86_const import *

# 待模拟的机器码: INC ecx; DEC edx
X86_CODE32 = b"\x41\x4a"
ADDRESS = 0x80000

try:
    # 1. 初始化模拟器为 x86-32 位架构
    mu = Uc(UC_ARCH_X86, UC_MODE_32)

    # 2. 在指定地址映射 2MB 内存
    mu.mem_map(ADDRESS, 2 * 1024 * 1024)

    # 3. 将代码写入映射的内存
    mu.mem_write(ADDRESS, X86_CODE32)

    # 4. 设置寄存器初始值
    mu.reg_write(UC_X86_REG_ECX, 0x1234)
    mu.reg_write(UC_X86_REG_EDX, 0x7890)

    # 5. 从 ADDRESS 开始执行，直到代码结束
    mu.emu_start(ADDRESS, ADDRESS + len(X86_CODE32))

    # 6. 读取并打印结果
    print("Emulation done. Below is the CPU context")
    r_ecx = mu.reg_read(UC_X86_REG_ECX)
    r_edx = mu.reg_read(UC_X86_REG_EDX)
    print(f">>> ECX = 0x{r_ecx:x}")
    print(f">>> EDX = 0x{r_edx:x}")

except UcError as e:
    print(f"ERROR: {e}")
```

### 4. 注意事项与扩展

- **错误处理**：Unicorn 的操作可能会抛出 `UcError` 异常。使用 `try...except` 块来捕获和处理这些错误是一个好习惯，例如当访问未映射内存时。
- **上下文保存/恢复**：Unicorn 提供了 `context_save()` 和 `context_restore()` 方法，可以保存和恢复整个模拟器的状态（包括所有寄存器和内存），这在需要反复从某个状态开始执行时非常有用。
- **Hook 机制**：可以通过 `hook_add` 方法添加指令、内存访问等事件的回调函数，用于跟踪执行流程或修改行为。
- **内存映射版本差异**：需要注意的是，不同版本的 Unicorn 在处理高虚拟地址时可能存在差异。例如，在 2.1.0 版本中，访问极高地址可能因 TLB 机制变化而出现问题。

总结来说，初始化 Unicorn 的核心就是**选择架构与模式、映射内存、载入代码**，之后便可以控制 CPU 执行并检查结果。



## unicorn操作内存
Unicorn 的内存操作是模拟执行的核心，主要包括：**映射/取消映射内存、设置权限、读写内存、监控内存访问（Hook）、查询内存区域**。下面按功能分类说明，并给出 Python 示例。

---

### 1. 映射与取消映射内存

#### `mem_map(address, size, perms=UC_PROT_ALL)`
映射一段虚拟内存，供模拟器使用。  
- `address`：起始地址，通常要求 **页对齐**（0x1000 的倍数）。
- `size`：大小，通常也建议页对齐。
- `perms`：权限，默认 `UC_PROT_ALL`（读+写+执行）。可组合：
  - `UC_PROT_READ`
  - `UC_PROT_WRITE`
  - `UC_PROT_EXEC`
  - `UC_PROT_NONE`
  - `UC_PROT_ALL`

```python
from unicorn import *

mu = Uc(UC_ARCH_X86, UC_MODE_32)

ADDR = 0x1000
SIZE = 0x1000  # 4KB
mu.mem_map(ADDR, SIZE, UC_PROT_READ | UC_PROT_WRITE | UC_PROT_EXEC)
```

#### `mem_unmap(address, size)`
取消映射，释放内存。地址和大小同样建议页对齐。

```python
mu.mem_unmap(ADDR, SIZE)
```

---

### 2. 修改内存权限

#### `mem_protect(address, size, perms)`
动态修改已映射内存的权限。例如去掉写权限，模拟只读内存。

```python
mu.mem_protect(ADDR, SIZE, UC_PROT_READ | UC_PROT_EXEC)
```

如果之后模拟代码尝试写入该区域，会触发 `UC_ERR_WRITE_PROT` 异常。

---

### 3. 读写内存

#### `mem_write(address, data)`
向指定地址写入字节串（`bytes`）。可以一次写多个字节。

```python
mu.mem_write(ADDR, b"\x41\x42\x43\x44")  # 写入 'ABCD'
```

#### `mem_read(address, size)`
从指定地址读取指定长度的字节串。

```python
data = mu.mem_read(ADDR, 4)
print(data)  # b'ABCD'
```

**注意**：
- 读写未映射内存会抛出 `UcError`。
- `mem_read` 返回原始字节，多字节整数需根据架构用 `struct` 解析（如小端 `<I`）。
- `mem_write`/`mem_read` 是外部 API，**不会触发内存访问 Hook**（Hook 只针对模拟执行中的内存访问）。

---

### 4. 内存访问 Hook

通过 `hook_add` 可以监控模拟执行过程中的内存读、写、取指，或捕获非法访问。

#### 常用 Hook 类型

| Hook 常量 | 说明 |
|-----------|------|
| `UC_HOOK_MEM_READ` | 合法内存读 |
| `UC_HOOK_MEM_WRITE` | 合法内存写 |
| `UC_HOOK_MEM_FETCH` | 合法取指 |
| `UC_HOOK_MEM_READ_UNMAPPED` | 读未映射内存 |
| `UC_HOOK_MEM_WRITE_UNMAPPED` | 写未映射内存 |
| `UC_HOOK_MEM_FETCH_UNMAPPED` | 取指未映射内存 |
| `UC_HOOK_MEM_READ_PROT` | 读受保护内存 |
| `UC_HOOK_MEM_WRITE_PROT` | 写受保护内存 |
| `UC_HOOK_MEM_FETCH_PROT` | 取指受保护内存 |
| `UC_HOOK_MEM_UNMAPPED` | 所有未映射访问 |
| `UC_HOOK_MEM_PROT` | 所有权限错误 |
| `UC_HOOK_MEM_VALID` | 所有合法访问 |
| `UC_HOOK_MEM_INVALID` | 所有非法访问 |

#### 回调函数签名
```python
def hook_mem(uc, access, address, size, value, user_data):
    # access: UC_MEM_READ / UC_MEM_WRITE / UC_MEM_FETCH
    # address: 访问的地址
    # size: 访问字节数
    # value: 写入的值（读操作时为 0）
    # user_data: 注册时传入的自定义数据
    return True  # 返回 True 继续执行，False 停止模拟
```

#### 示例：监控读写
```python
def hook_mem_access(uc, access, address, size, value, user_data):
    if access == UC_MEM_WRITE:
        print(f"写入: addr=0x{address:x}, size={size}, value=0x{value:x}")
    elif access == UC_MEM_READ:
        print(f"读取: addr=0x{address:x}, size={size}")
    return True

mu.hook_add(UC_HOOK_MEM_READ | UC_HOOK_MEM_WRITE, hook_mem_access)
```

#### 示例：捕获非法访问
```python
def hook_mem_invalid(uc, access, address, size, value, user_data):
    print(f"非法内存访问: access={access}, addr=0x{address:x}, size={size}")
    return False  # 停止模拟

mu.hook_add(UC_HOOK_MEM_INVALID, hook_mem_invalid)
```

---

### 5. 查询内存区域

#### `mem_regions()`
返回当前已映射的所有内存区域列表，每个元素为 `(begin, end, perms)`。  
- `begin`：起始地址  
- `end`：结束地址（**包含最后一个字节**）  
- `perms`：权限

```python
for begin, end, perms in mu.mem_regions():
    print(f"区域: 0x{begin:x} - 0x{end:x}, 权限: {perms}")
```

区域大小 = `end - begin + 1`。

---

### 6. 常见错误与注意事项

- **页对齐**：`mem_map`、`mem_unmap`、`mem_protect` 的地址和大小通常要求 4KB 对齐，否则可能抛出 `UC_ERR_ARG` 或 `UC_ERR_MAP`。
- **异常处理**：所有内存操作都可能抛出 `UcError`，建议用 `try...except` 捕获。
- **权限常量**：`UC_PROT_*` 在 `unicorn` 模块中直接可用。
- **大小端**：`mem_read` 返回原始字节，解析多字节数据时注意目标架构的字节序。
- **Hook 返回值**：返回 `True` 继续执行，`False` 会停止模拟（部分 Hook 可能忽略返回值）。
- **外部 API 与 Hook**：`mem_write`/`mem_read` 不触发 Hook；只有 `emu_start` 执行过程中的内存访问才会触发。

---

### 7. 完整示例

```python
from unicorn import *
from unicorn.x86_const import *

# 初始化 x86-32
mu = Uc(UC_ARCH_X86, UC_MODE_32)

ADDR = 0x1000
SIZE = 0x1000

# 1. 映射内存，可读可写可执行
mu.mem_map(ADDR, SIZE, UC_PROT_READ | UC_PROT_WRITE | UC_PROT_EXEC)

# 2. 写入机器码：mov eax, 0x1234; ret
#    B8 34 12 00 00 C3
code = b"\xB8\x34\x12\x00\x00\xC3"
mu.mem_write(ADDR, code)

# 3. 添加内存写 Hook
def hook_mem_write(uc, access, address, size, value, user_data):
    if access == UC_MEM_WRITE:
        print(f"[Hook] 写内存: addr=0x{address:x}, size={size}, value=0x{value:x}")
    return True

mu.hook_add(UC_HOOK_MEM_WRITE, hook_mem_write)

# 4. 执行
mu.emu_start(ADDR, ADDR + len(code))

# 5. 读取 EAX
eax = mu.reg_read(UC_X86_REG_EAX)
print(f"EAX = 0x{eax:x}")

# 6. 读取内存数据
data = mu.mem_read(ADDR, len(code))
print(f"内存内容: {data.hex()}")

# 7. 查询内存区域
for begin, end, perms in mu.mem_regions():
    print(f"区域: 0x{begin:x} - 0x{end:x}, 权限: {perms}")

# 8. 修改权限为只读
mu.mem_protect(ADDR, SIZE, UC_PROT_READ)

# 9. 取消映射
mu.mem_unmap(ADDR, SIZE)
```




## unicorn寄存器操作
在 Unicorn 中，寄存器操作是模拟执行的核心之一。通过寄存器读写，可以设置初始状态、获取执行结果，并结合 Hook 监控指令执行时的寄存器变化。下面按功能分类说明 Python 中的寄存器操作方法。

---

### 1. 导入寄存器常量

不同架构的寄存器 ID 定义在不同的模块中，必须先导入对应模块。

```python
from unicorn import *
from unicorn.x86_const import *   # x86 / x86-64
# from unicorn.arm_const import *   # ARM
# from unicorn.arm64_const import * # ARM64
# from unicorn.mips_const import *  # MIPS
# from unicorn.sparc_const import * # SPARC
```

寄存器常量命名规则：`UC_<架构>_REG_<寄存器名>`，例如：
- x86：`UC_X86_REG_EAX`、`UC_X86_REG_RAX`、`UC_X86_REG_EFLAGS`
- ARM：`UC_ARM_REG_R0`、`UC_ARM_REG_PC`、`UC_ARM_REG_CPSR`
- ARM64：`UC_ARM64_REG_X0`、`UC_ARM64_REG_PC`
- MIPS：`UC_MIPS_REG_A0`、`UC_MIPS_REG_PC`

---

### 2. 单个寄存器读写

#### `reg_read(reg_id)`
读取寄存器的值，返回整数（Python int，支持大整数）。

```python
eax = mu.reg_read(UC_X86_REG_EAX)
print(f"EAX = 0x{eax:x}")
```

#### `reg_write(reg_id, value)`
向寄存器写入值。

```python
mu.reg_write(UC_X86_REG_EAX, 0x1234)
mu.reg_write(UC_X86_REG_EBX, 0x5678)
```

**注意**：
- 寄存器 ID 必须与当前模拟器架构匹配，否则会抛出 `UcError`。
- 对于 x86-64，写 `EAX` 会自动清零 `RAX` 的高 32 位（符合 x86 语义）。
- 对于 SIMD 寄存器（如 XMM、YMM），`reg_read`/`reg_write` 同样使用整数，Python 支持 128/256 位大整数。
- 写入值超出寄存器宽度时，Unicorn 会截断或报错（取决于版本）。

---

### 3. 批量寄存器读写

Unicorn 1.0+ 提供了批量操作，减少 Python 与 C 之间的调用开销。

#### `reg_read_batch(reg_ids)`
传入寄存器 ID 列表，返回对应值的列表。

```python
regs = [UC_X86_REG_EAX, UC_X86_REG_EBX, UC_X86_REG_ECX]
values = mu.reg_read_batch(regs)
print([hex(v) for v in values])
```

#### `reg_write_batch(reg_ids, values)`
同时写入多个寄存器。

```python
mu.reg_write_batch(
    [UC_X86_REG_EAX, UC_X86_REG_EBX],
    [0x1111, 0x2222]
)
```

**注意**：批量接口要求两个列表长度一致，且寄存器 ID 均有效。

---

### 4. 获取寄存器名称

#### `reg_name(reg_id)`
根据寄存器 ID 返回可读名称。

```python
name = mu.reg_name(UC_X86_REG_EAX)  # 返回 'eax'
print(name)
```

如果传入无效 ID，可能返回 `None` 或抛出异常。

---

### 5. 上下文保存与恢复

Unicorn 提供 `context_save()` 和 `context_restore()` 来保存/恢复整个 CPU 上下文（包括所有寄存器和内部状态）。这比逐个读写寄存器高效得多。

```python
# 保存当前上下文
ctx = mu.context_save()

# ... 执行一些操作，修改寄存器 ...

# 恢复到保存时的状态
mu.context_restore(ctx)
```

**用途**：在需要反复从某个状态开始执行时（如模糊测试、符号执行），可以避免重复初始化。

---

### 6. 在 Hook 中监控寄存器

Unicorn 没有专门的“寄存器读写 Hook”，但可以通过指令 Hook（`UC_HOOK_CODE`）在每条指令执行前读取寄存器，实现监控。

#### 指令 Hook 回调签名
```python
def hook_code(uc, address, size, user_data):
    # address: 当前指令地址
    # size: 指令长度
    eax = uc.reg_read(UC_X86_REG_EAX)
    print(f"执行 0x{address:x}: EAX=0x{eax:x}")
    return True
```

注册：
```python
mu.hook_add(UC_HOOK_CODE, hook_code)
```

#### 只监控特定地址范围
```python
mu.hook_add(UC_HOOK_CODE, hook_code, begin=0x1000, end=0x2000)
```

**注意**：在 Hook 中读写寄存器是安全的，但频繁操作会影响性能。

---

### 7. 完整示例：x86-32 寄存器操作

```python
from unicorn import *
from unicorn.x86_const import *

# 机器码：add eax, ebx; ret
# 01 D8 C3
CODE = b"\x01\xD8\xC3"
ADDR = 0x1000

mu = Uc(UC_ARCH_X86, UC_MODE_32)
mu.mem_map(ADDR, 0x1000)
mu.mem_write(ADDR, CODE)

# 设置初始寄存器值
mu.reg_write(UC_X86_REG_EAX, 0x10)
mu.reg_write(UC_X86_REG_EBX, 0x20)

# 执行
mu.emu_start(ADDR, ADDR + len(CODE))

# 读取结果
eax = mu.reg_read(UC_X86_REG_EAX)
print(f"EAX = 0x{eax:x}")  # 应输出 0x30

# 批量读取
vals = mu.reg_read_batch([UC_X86_REG_EAX, UC_X86_REG_EBX])
print([hex(v) for v in vals])

# 保存/恢复上下文
ctx = mu.context_save()
mu.reg_write(UC_X86_REG_EAX, 0xFFFF)
print(f"修改后 EAX = 0x{mu.reg_read(UC_X86_REG_EAX):x}")
mu.context_restore(ctx)
print(f"恢复后 EAX = 0x{mu.reg_read(UC_X86_REG_EAX):x}")
```

---

### 8. 常见问题与注意事项

- **寄存器 ID 错误**：使用与架构不匹配的寄存器常量会抛出 `UcError: Invalid register ID`。
- **寄存器宽度**：写入值超过寄存器位宽时，Unicorn 可能截断。例如向 32 位 EAX 写 `0x1_0000_0000` 会变成 0。
- **标志寄存器**：如 `UC_X86_REG_EFLAGS`，可以单独读写，但修改标志位可能影响后续条件跳转，需谨慎。
- **浮点/向量寄存器**：XMM 等寄存器通过 `reg_read` 返回整数，操作方式与通用寄存器相同，但语义上需自行解析。
- **性能**：频繁的 `reg_read`/`reg_write` 会降低模拟速度，批量接口和上下文保存/恢复更适合大量操作。
- **版本差异**：批量接口 `reg_read_batch`/`reg_write_batch` 在 Unicorn 1.0 后引入，旧版本可能不支持。

掌握这些寄存器操作后，就能灵活控制模拟器的 CPU 状态，配合内存操作和 Hook 实现复杂的模拟与分析任务。



## unicorn执行控制
在 Unicorn 中，**执行控制**指的是如何启动模拟、暂停/恢复、限制执行范围、响应异常，以及通过 Hook 干预执行流程。下面按功能分类详细说明。

如果说 `emu_start`​ 是让程序跑起来，那么 **Hook** 就是让程序“透明化”。  
Hook 机制是 Unicorn 最强大的特性之一。它允许我们在模拟执行的过程中插入自定义的回调函数（Callback），相当于在 CPU 内部安装了无数个“监控探头”。

通过 Hook，我们可以实现：

- ​**指令追踪**：打印每一条正在执行的汇编指令（Trace）。
- ​**内存监控**：检测程序是否非法读写了内存。
- ​**断点调试**：在特定地址暂停执行。
- ​**补丁/Fuzz**：动态修改寄存器或内存数据。

#### 1. 注册 Hook (安装监控)

- ​**API**​: `uc.hook_add(hook_type, callback, user_data=None, begin=1, end=0)`
    
    - ​`hook_type`: 监控类型（如指令执行、内存读写、中断等）。
    - ​`callback`: 触发时调用的 Python 函数。
    - ​`user_data`: (可选) 传递给回调函数的额外数据。
    - ​`begin`​, `end`: (可选) 仅监控此地址范围内的事件。默认监控全局。
- ​**API**​: `uc.hook_del(hook_handle)`
    
    - 删除已注册的 Hook。
---

### 1. 启动执行：`emu_start`

#### 基本签名
```python
emu_start(begin, until, timeout=0, count=0)
```

| 参数 | 说明 |
|------|------|
| `begin` | 开始执行的地址 |
| `until` | 停止执行的地址（**不包含**该地址，即执行到此地址前停止） |
| `timeout` | 超时时间，单位**微秒**；0 表示不限 |
| `count` | 最大执行指令条数；0 表示不限 |

#### 基本用法
```python
mu.emu_start(ADDR, ADDR + len(CODE))
```

#### 指定结束地址
```python
# 执行到 0x2000 之前停止
mu.emu_start(0x1000, 0x2000)
```

#### 限制指令条数
```python
# 最多执行 100 条指令
mu.emu_start(0x1000, 0x2000, count=100)
```

#### 限制执行时间
```python
# 最多执行 1 秒（1,000,000 微秒）
mu.emu_start(0x1000, 0x2000, timeout=1000000)
```

**注意**：
- `until` 是"运行到此地址**之前**停止"。通常配合 `ret` 或代码末尾地址使用。
- 如果 `until` 地址处的指令永远不会被执行到，模拟会一直运行（除非设置 `count` 或 `timeout`）。
- 若 `until=0`，则相当于不设结束地址（视版本而定，建议显式指定）。

---

### 2. 停止执行：`emu_stop`

在 Hook 回调中调用 `emu_stop()` 可以主动停止模拟。

```python
def hook_code(uc, address, size, user_data):
    if address == 0x1234:
        print("到达目标地址，停止执行")
        uc.emu_stop()
    return True

mu.hook_add(UC_HOOK_CODE, hook_code)
mu.emu_start(0x1000, 0x2000)
```

**特点**：
- `emu_stop()` 可以**在 Hook 中调用**，也可以在另一个线程中调用，实现异步暂停。
- 调用后，`emu_start` 会立即返回（当前指令执行完后）。
- 停止后可以再次调用 `emu_start` 继续执行，或使用 `context_restore` 恢复状态。

#### 从外部线程停止
```python
import threading

def stop_later(uc):
    import time
    time.sleep(2)
    uc.emu_stop()

# 在子线程中启动模拟
t = threading.Thread(target=lambda: mu.emu_start(0x1000, 0x2000))
t.start()

# 主线程 2 秒后停止
stop_later(mu)
t.join()
```

---

### 3. 恢复执行

`emu_start` 返回后，CPU 状态（寄存器、内存）保留在停止时的状态。可以再次调用 `emu_start` 从停止处继续。

```python
# 第一次执行
mu.emu_start(0x1000, 0x1500)

# 读取中间状态
eax = mu.reg_read(UC_X86_REG_EAX)
print(f"中途 EAX = 0x{eax:x}")

# 继续执行
mu.emu_start(mu.reg_read(UC_X86_REG_EIP), 0x2000)
```

**注意**：x86 架构下取当前指令地址用 `EIP`（32 位）或 `RIP`（64 位）。

---

### 4. 指令级 Hook：`UC_HOOK_CODE`

最常用的执行控制手段，每条指令执行前触发。

#### 回调签名
```python
def hook_code(uc, address, size, user_data):
    # address: 当前指令地址
    # size: 指令字节长度
    # user_data: 注册时传入的自定义数据
    return True  # 返回 True 继续，False 停止
```

#### 注册
```python
# 全局 Hook
mu.hook_add(UC_HOOK_CODE, hook_code)

# 限定地址范围 [begin, end]
mu.hook_add(UC_HOOK_CODE, hook_code, begin=0x1000, end=0x2000)
```

#### 带 user_data
```python
mu.hook_add(UC_HOOK_CODE, hook_code, user_data={"count": 0})
```

#### 示例：指令计数与跟踪
```python
def hook_code(uc, address, size, user_data):
    user_data["count"] += 1
    print(f"[{user_data['count']}] 0x{address:x}: {size} bytes")
    return True

ctx = {"count": 0}
mu.hook_add(UC_HOOK_CODE, hook_code, user_data=ctx)
mu.emu_start(0x1000, 0x2000)
print(f"总指令数: {ctx['count']}")
```

#### Hook 返回值语义
- 返回 `True`：继续执行。
- 返回 `False`：**停止模拟**（等价于调用 `emu_stop`）。

---

### 5. 基本块 Hook：`UC_HOOK_BLOCK`

每个基本块（Basic Block）执行前触发一次，比 `UC_HOOK_CODE` 性能更好。

```python
def hook_block(uc, address, size, user_data):
    print(f"基本块: 0x{address:x}, 大小: {size}")
    return True

mu.hook_add(UC_HOOK_BLOCK, hook_block)
```

**特点**：
- 一个基本块内的多条指令只触发一次。
- 适合做粗粒度跟踪，性能优于 `UC_HOOK_CODE`。

---

### 6. 中断 Hook：`UC_HOOK_INTR`

当模拟执行到 `int`/`syscall`/`svc` 等中断指令时触发，用于模拟系统调用。

```python
def hook_intr(uc, intno, user_data):
    print(f"中断号: {intno}")
    # 处理系统调用后，需要手动跳过中断指令
    return True

mu.hook_add(UC_HOOK_INTR, hook_intr)
```

**常见做法**：在中断 Hook 中读取寄存器识别系统调用号，执行对应逻辑，然后修改寄存器（如 `EAX` 为返回值）并继续。

---

### 7. 异常处理：`UcError`

模拟执行中的错误以异常形式抛出，需要捕获处理。

```python
from unicorn import UcError

try:
    mu.emu_start(0x1000, 0x2000)
except UcError as e:
    print(f"模拟错误: {e}")
    print(f"错误码: {mu.errno}")
```

#### 常见错误码

| 错误码 | 含义 |
|--------|------|
| `UC_ERR_OK` | 无错误 |
| `UC_ERR_MAP` | 内存映射错误 |
| `UC_ERR_READ_UNMAPPED` | 读未映射内存 |
| `UC_ERR_WRITE_UNMAPPED` | 写未映射内存 |
| `UC_ERR_FETCH_UNMAPPED` | 取指未映射内存 |
| `UC_ERR_INSN_INVALID` | 非法指令 |
| `UC_ERR_HOOK` | Hook 错误 |
| `UC_ERR_MEM_READ` / `UC_ERR_MEM_WRITE` | Hook 中内存访问错误 |

#### 使用 Hook 捕获非法访问并继续
默认情况下，非法内存访问会停止模拟。但可以通过 `UC_HOOK_MEM_INVALID` Hook 返回 `True` 来**忽略错误并继续**。

```python
def hook_mem_invalid(uc, access, address, size, value, user_data):
    print(f"非法访问: addr=0x{address:x}")
    # 动态映射缺失的内存
    try:
        uc.mem_map(address & ~0xFFF, 0x1000)
        return True  # 继续执行
    except UcError:
        return False  # 停止

mu.hook_add(UC_HOOK_MEM_INVALID, hook_mem_invalid)
```

**注意**：内存 Hook 的返回值语义较特殊：
- `UC_HOOK_MEM_READ_UNMAPPED` 等返回 `True` 表示"已处理，继续"。
- 返回 `False` 表示停止模拟。

---

### 8. 上下文保存与恢复：控制执行流

配合 `context_save`/`context_restore` 可以实现快照式执行控制。

```python
# 执行到某点，保存上下文
mu.emu_start(0x1000, 0x1200)
snapshot = mu.context_save()

# 尝试路径 A
mu.reg_write(UC_X86_REG_EAX, 1)
try:
    mu.emu_start(0x1200, 0x2000)
except UcError:
    pass

# 恢复，尝试路径 B
mu.context_restore(snapshot)
mu.reg_write(UC_X86_REG_EAX, 2)
mu.emu_start(0x1200, 0x2000)
```

**用途**：模糊测试、符号执行、多路径探索。

---

### 9. 完整示例：综合执行控制

```python
from unicorn import *
from unicorn.x86_const import *

# 机器码：
#   mov eax, 1
#   inc eax
#   inc eax
#   ret
CODE = b"\xB8\x01\x00\x00\x00\x40\x40\xC3"
ADDR = 0x1000

mu = Uc(UC_ARCH_X86, UC_MODE_32)
mu.mem_map(ADDR, 0x1000)
mu.mem_write(ADDR, CODE)

# 指令计数
counter = {"n": 0}

def hook_code(uc, address, size, user_data):
    user_data["n"] += 1
    eax = uc.reg_read(UC_X86_REG_EAX)
    print(f"[{user_data['n']}] 0x{address:x} EAX=0x{eax:x}")
    return True

def hook_block(uc, address, size, user_data):
    print(f"--- 基本块 0x{address:x}, 大小={size} ---")
    return True

mu.hook_add(UC_HOOK_CODE, hook_code, user_data=counter)
mu.hook_add(UC_HOOK_BLOCK, hook_block)

try:
    # 最多执行 100 条指令，5 秒超时
    mu.emu_start(ADDR, ADDR + len(CODE), timeout=5_000_000, count=100)
except UcError as e:
    print(f"错误: {e}")

eax = mu.reg_read(UC_X86_REG_EAX)
print(f"结果 EAX = 0x{eax:x}")
print(f"总指令数: {counter['n']}")
```

---

### 10. 注意事项与最佳实践

- **`until` 地址语义**：执行到该地址**之前**停止，不是"执行完该地址"。
- **避免死循环**：始终设置 `count` 或 `timeout`，防止无限循环。
- **Hook 性能**：`UC_HOOK_CODE` 开销大，粗粒度跟踪用 `UC_HOOK_BLOCK`。
- **`emu_stop` 的时机**：在 Hook 中调用后，当前指令会执行完才真正停止。
- **异常 Hook 与 continue**：`UC_HOOK_MEM_INVALID` 返回 `True` 需确保内存问题已解决，否则会反复触发。
- **线程安全**：Unicorn 实例不是线程安全的，`emu_stop` 从其他线程调用是允许的，但其他操作需加锁。
- **`errno` 与异常**：捕获 `UcError` 后可通过 `mu.errno` 获取详细错误码。
- **中断模拟**：`UC_HOOK_INTR` 中通常需要手动修改 PC/寄存器来跳过中断指令，否则会重复触发。










# temp