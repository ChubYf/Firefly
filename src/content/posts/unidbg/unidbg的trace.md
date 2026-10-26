---
title: unidbg的trace
published: 2026-10-15
description: unidbg的trace
image: ./cover.jpg
tags: [逆向]
category: unidbg
draft: false
---


[Trace in Unidbg (一)](https://www.yuque.com/docs/share/80caf131-9f41-4e0b-b1b6-fa29e7d909eb?#%20%E3%80%8ATrace%20in%20Unidbg%EF%BC%88%E4%B8%80%EF%BC%89%E3%80%8B  )
[Trace in Unidbg (二)](https://www.yuque.com/docs/share/5bb4de33-13cf-4eb0-beba-558af41ebd46?#%20%E3%80%8ATrace%20in%20Unidbg%EF%BC%88%E4%BA%8C%EF%BC%89%E3%80%8B  )






# unidbg的trace介绍和基本使用
## 一、引言

Hook、Debug、Trace 是逆向分析的三大工具。

简单样本用 Hook 基本就够了，但如果样本复杂度相对较高，debug 和 trace 就变得必要。这个系列讨论的是 trace。更准确的说，是以 Unidbg 为主视角去简单讨论 trace，毕竟 trace 可不是一个小话题，讨论它的一角就已经十分不易。

## 二、概述

根据所处的粒度与层级，Trace 有很多类目和表现形式。但最常见的是下面三类

- 函数 trace，包括无差别的函数 trace、导出函数 trace、库函数 trace、系统调用 trace 等。此类 trace 在分析算法执行流、应付 OLLVM 和动态跳转上很好用。
- 基本块 trace，也就是 block trace，常用于控制流的分析与反混淆。
- 汇编 trace，包括 instruction trace 以及仅对某一种、某一类指令的 trace。在分析自定义算法、花指令等场景里都有很大作用。

本篇讨论汇编 trace，它是最重要的 trace 类别，基于汇编 trace 可以实现其他类型的各种 trace。

首先回顾以往工具如何实现汇编 trace。最常见的两类办法是基于调试器和基于动态重编译。

比如 GDB trace、IDA Remote trace、LLDB trace 都是基于调试器的方案，我们对其中的典型 —— IDA Remote tarce 展开讨论。下图红框里的 Instruction tracing 就是汇编追踪。

![](https://cdn.nlark.com/yuque/0/2022/png/707476/1667293759668-8e84571b-9596-45bb-b2ce-46bdab1a531f.png)

它是一个常规且主流的方案，但事实上一直不算好用，最主要的缺陷是慢与不稳定，这些缺陷来自于调试器本身。

首先说**慢**，调试器底层基于异常中断实现指令的单步追踪，异常中断的开销并不小，如果已找到了关键函数，或者能将 trace 的范围限定到足够小，比如几十万行汇编的体量，那么 IDA trace 速度慢并不能慢到哪里去，几个小时内总能完成这项任务。

但如果没找到关键函数，也没约束追踪地址范围，只想做无筛选、无区分、整体性的 trace，汇编执行流辄数千万甚至上亿行，那么速度慢就会成为最大的问题。IDA trace 可能需要十天半个月才能完成，这是几乎无法承受的耗时和代价。

其次是**不稳定**，IDA 容易被检测，大多数样本都会有 IDA 检测或 ptrace 附加检测，这阻止我们使用 IDA 做动态分析，更别提 trace 指令了。如果恰好没有检测或费尽心思绕过了这些检测，也可能因为 IDA 的版本、测试机的型号等玄学问题而闪退崩溃，这种不稳定非常搞人心态，指令追踪了一整天，跑了一半，结果 IDA Debug 突然断了，要重新 trace，这太搞人心态了。

当然，这并不是说以 IDA 为代表的调试器 trace 就没法用，只是说它确实不够好用。

第二类办法是动态重编译，它是目前最流行的汇编 trace 方案，典型代表是 Frida Stalker，原理如下。![](https://cdn.nlark.com/yuque/0/2022/png/707476/1667294245851-4ac53427-57be-497e-b41c-1fc0ffc369f1.png)

简单来说，就是开辟一块新内存，将源程序拷贝过来，在每行代码间留出空间，然后插桩打印执行流和寄存器状态，其中需要针对一些指令做单独处理。

动态重编译是一个很好的方案，trace 效率比调试器快了很多个数量级，可以在很短时间里完成数以亿记的代码踪和记录。尽管动态重编译 trace 的方案里，指令被“膨胀”了很多倍，但处理器执行汇编的速度实在是太快了，所以 trace 也飞快。

如果说它有什么缺陷，感受上主要有两方面。

一是目前大部分的落地方案仅支持 ARM64，ARM32 则不做兼容或支持度不高，比如 Frida Stalker 就是这样，因为 ARM32 过时了。但如果样本仅提供 32 位的 SO，那就相当尴尬。

二是某些指令需要特殊处理，而工具没有对它做特殊处理时就会出 BUG。但这算不上什么麻烦事，只要设置跳过这条指令或针对性做处理即可。

最后是 Unidbg、Qiling 等工具所支持的汇编追踪，它们的底层引擎是 CPU 模拟器，自然可以很方便的去做汇编追踪，我们这里主要指的是 Unicorn。相比较 Stalker，优势是支持 ARM32，除此之外它更稳定，BUG 更少。但缺点也有一些。

首先就是必须得在 Unidbg、QIling 等自身环境里处理好样本，能顺利模拟执行之后，才能使用 它们做完整 trace，而 Stalker 或 Debug Trace 依托于真实 Android 环境，则没有这种担忧。

其次它的速度相比较 Stalker 慢一些，打个比方，如果处理复杂的超大型样本，使用 Stalker 或类似原理的 trace 其耗时需要按小时算，使用 Unidbg trace 就要按天计时，使用 IDA trace 则要按月计时。

这是一个相对夸张的说法，但大体上反映了三者执行速度的差异。

但综合看基于 Unicorn 的 trace 还是不错的，平时也很难遇到超大型的样本，按几亿行的汇编执行流算，在 Unidbg 里一两天也能跑完。

具备 trace 的能力，和实际好不好用是两码事。

从整体上看，评价汇编追踪好不好用有三大因素

- 能力
- 速度
- 展示效果

我们前面 diss 调试器追踪的原因就是能力上不够稳定，速度上极慢。下面再看看展示效果。比如下面是 qiling 的 trace 效果。

```java
[=] 	047c61d4 [libtest.so + 0x00c1d4]  01 30 40 e2   sub  r3, r0, #1
[=] 	r0         : 0x90001ec1
[=] 	r1         : 0x90001c32
[=] 	r2         : 0x2e
[=] 	r3         : 0x90001ec1
[=] 	r4         : 0x90001c38
[=] 	r5         : 0x47ebabc
[=] 	r6         : 0x13
[=] 	r7         : 0x90001eb0
[=] 	r8         : 0x90001c20
[=] 	r9         : 0x12
[=] 	r10        : 0x9
[=] 	r11        : 0x2f
[=] 	r12        : 0x0
[=] 	sp         : 0x7ff3c650
[=] 	lr         : 0x6f732e63
[=] 	pc         : 0x47c61d4
[=] 	cpsr       : 0x200001d3
```

它展示了很多信息

- 当前绝对地址：0x047c61d4
- 当前所处模块：libtest.so
- 相对偏移地址：0x00c1d4
- 机器码：01 30 40 e2
- 汇编代码：sub r3, r0, #1
- 当前所有寄存器的值：从 r0 到 r15，以及 CPSR 的值。

从信息的角度看，该有的都有。但大家应该不会特别喜欢这个展示效果，寄存器的信息冗余太多了，看起来比较吃力。

不论是 IDA trace，还是 Unicorn trace、Stalker，都需要考虑展示的问题，即工具提供了对每条汇编做处理的时机，但究竟如何展示，这是一个问题。

在 IDA trace 的处理上，我们往往会写一个 DBG_Hooks 类型的 IDA 插件，它可以对 trace 做进一步的脚本化处理，比如约束 trace 范围、设计信息展示效果等等。

在 Stalker 上，则出现了很多基于 Stalker 二次封装的工具，它们试图带来更简便和优雅的展示效果。

比如 [sktrace](https://bbs.pediy.com/thread-264680.htm) 就是其中一个，它的效果显然比上面的例子精简很多。

![](https://cdn.nlark.com/yuque/0/2022/png/707476/1667316187775-e69a146a-650f-4505-b1a9-bf9b7770498b.png)

又比如 [frida-trace](https://bbs.pediy.com/thread-273501.htm)，它参考了 sktrace，进一步丰富了展示效果。

![](https://cdn.nlark.com/yuque/0/2022/png/707476/1667316225557-429aa692-15d6-4d78-a449-2eaab86f5c6c.png)

在 0.9.5 以及更早的版本上，Unidbg 并没有很好的处理 code trace，其中最突出的问题是打印内容中不包含寄存器信息，像下面这样。

```java
### Trace Instruction [libjnidispatch.so] [0x03f94] [ 06 00 a0 e1 ] 0x40003f94: mov r0, r6
### Trace Instruction [libjnidispatch.so] [0x03f98] [ 14 d0 4b e2 ] 0x40003f98: sub sp, fp, #0x14
### Trace Instruction [libjnidispatch.so] [0x03f9c] [ f0 88 bd e8 ] 0x40003f9c: pop {r4, r5, r6, r7, fp, pc}
### Trace Instruction [libjnidispatch.so] [0x04048] [ 09 30 a0 e1 ] 0x40004048: mov r3, sb
### Trace Instruction [libjnidispatch.so] [0x0404c] [ 07 10 a0 e1 ] 0x4000404c: mov r1, r7
### Trace Instruction [libjnidispatch.so] [0x04050] [ 06 20 a0 e1 ] 0x40004050: mov r2, r6
### Trace Instruction [libjnidispatch.so] [0x04054] [ 00 00 8d e5 ] 0x40004054: str r0, [sp]
```

那时候我们需要对 Unidbg trace 修修补补，使之具备寄存器展示，或使用热心网友所开发的 [KingTrace](https://github.com/zhkl0228/unidbg/pull/214)。

这种尴尬的状况在 0.9.6 上得到了彻底解决，Unidbg 对 trace 模块做了很多优化，让它和任何 trace 展示相比，效果都毫不逊色。接下来介绍 Unidbg trace 的基本使用。

## 三、Unidbg 指令追踪的使用

本节的样例以及代码如下 [📎com.aikucun.akapp_6.0.6_60006.apk.7z](https://www.yuque.com/attachments/yuque/0/2022/7z/707476/1663845953337-20fca23e-1d0b-4ca4-9fd1-15097fc07984.7z)

```java
package com.aikucun;

import com.github.unidbg.AndroidEmulator;
import com.github.unidbg.Emulator;
import com.github.unidbg.Module;
import com.github.unidbg.arm.backend.Unicorn2Factory;
import com.github.unidbg.debugger.BreakPointCallback;
import com.github.unidbg.linux.android.AndroidEmulatorBuilder;
import com.github.unidbg.linux.android.AndroidResolver;
import com.github.unidbg.linux.android.dvm.*;
import com.github.unidbg.memory.Memory;
import unicorn.Arm64Const;

import java.io.File;
import java.io.FileNotFoundException;
import java.io.FileOutputStream;
import java.io.PrintStream;

public class MXSecurity extends AbstractJni {
    private final AndroidEmulator emulator;
    private final DvmClass security;
    private final VM vm;
    private final Module module;

    public MXSecurity() {
        emulator = AndroidEmulatorBuilder.for64Bit()
                .addBackendFactory(new Unicorn2Factory(false))
                .setProcessName("com.aikucun.akapp")
                .build();
        emulator.getBackend().registerEmuCountHook(10000); // 设置执行多少条指令切换一次线程
        emulator.getSyscallHandler().setVerbose(true);
        emulator.getSyscallHandler().setEnableThreadDispatcher(true);
        Memory memory = emulator.getMemory();
        memory.setLibraryResolver(new AndroidResolver(23));
        vm = emulator.createDalvikVM(new File("unidbg-android/src/test/resources/aikucun/com.aikucun.akapp_6.0.6_60006.apk"));
        vm.setVerbose(true);
        vm.setJni(this);
        DalvikModule dm = vm.loadLibrary("mx", true);
        module = dm.getModule();
        security = vm.resolveClass("com/mengxiang/arch/security/MXSecurity");
        dm.callJNI_OnLoad(emulator);
    }
    
    public void callinit(){
        DvmObject<?> context = vm.resolveClass("com.aikucun.akapp.AppContext").newObject(null);
        int ret = security.callStaticJniMethodInt(emulator, "init", context, false);
        System.out.println("call init:"+ret);
    }

    public void callSignV1(){
        String arg1 = "https://zuul.aikucun.com/aggregation-center-facade/api/app/search/product/image/switch?" +
                "appid=38741001&did=6d2fe7c7702721c6b797cf22ec8f5f58&noncestr=4b373c&timestamp=1662394452&zuul=1";
        String arg2 = "4b373c";
        String arg3 = "1662394452";
        String result = security.callStaticJniMethodObject(emulator, "signV1", arg1, arg2, arg3).getValue().toString();
        System.out.println("call SignV1 result:"+result);
    }

    public static void main(String[] args) {
        MXSecurity mxSecurity = new MXSecurity();
        mxSecurity.callinit();
        mxSecurity.callSignV1();
    }

    @Override
    public DvmObject<?> callObjectMethodV(BaseVM vm, DvmObject<?> dvmObject, String signature, VaList vaList) {
        switch (signature){
            case "com/aikucun/akapp/AppContext->getPackageManager()Landroid/content/pm/PackageManager;":{
                return vm.resolveClass("android/content/pm/PackageManager").newObject(null);
            }
            case "com/aikucun/akapp/AppContext->getPackageName()Ljava/lang/String;":{
                return new StringObject(vm, vm.getPackageName());
            }
        }
        return super.callObjectMethodV(vm, dvmObject, signature, vaList);
    }
}
```

前面我们从一个大的视角讨论了各种 trace 的优劣，这里以 Unidbg trace 为主题，做进一步讨论。首先说结论，和同类型工具相比，Unidbg 的表现属于中上，不是最好用的那种，但绝对够用。

从易用性、效率、稳定性、兼容性、展示效果这五方面去评价它。

在易用性或说使用门槛上，Unidbg traceCode 既高又低。高指的是想使用 Unidbg traceCode，你必须掌握 Unidbg 的使用，只有将目标函数用 Unidbg 完整良好的执行出来，才能得到相应的、完整的 trace；低指的是在 Unidbg 中开启指令追踪十分方便，只需要添加`emulator.traceCode(begin, end)`这样一行代码即可，和它相比 Frida Stalker 以及 IDA Instruction trace 的开启都复杂的多。

在效率或者说速度上，它的速度中等。在较好的测试条件中，每分钟可以 trace 40w 行执行流（2400w/h）。这意味着，简单样本的 trace 耗时在 10 分钟内，中等样本的 trace 耗时在 2 小时内；相当复杂、混淆严重的样本，可能要跑 12 小时甚至更久。这个速度比 IDA Instruction trace 快了很多倍，但比基于动态重编译的 trace 要慢不少，比如 Stalker 等方案在性能优良的测试机 + 适当的优化处理下，可以达到 100w/min 甚至更快。需要注意，上面讨论的是理论速度，在实际情况中，需要做范围过滤等处理，真实 trace 速度可能只有原先 1/4 或更低。

在稳定性上，它的表现最好。IDA trace 要担心突然断掉或应用崩溃，Stalker trace 在某些指令上有 bug，需要做额外处理。Unidbg trace 基于底层的 Unicorn，使用上体验相当丝滑，trace 虽然也有，但同比算是很少。

在兼容性上它的表现也很好，包括 Stalker 在内的大多数指令追踪方案都没兼容或处理好 ARM32，Unidbg 支持 ARM32 + ARM64，但相比 IDA instruction trace 又有些不足，毕竟 IDA 还支持 X86/Mips 等架构的指令追踪。

在展示上，Unidbg trace 的表现很好。在信息维度，包括时间、绝对地址、所处模块、相对偏移、机器码、汇编代码、执行前后的寄存器变化，可以说是一应俱全。在信息处理上，也十分不错，有很多小细节优化。

```java
[20:09:02 665][libmx.so 0x0d960] [41ffff54] 0x4000d960: "b.ne #0x4000d948" nzcv: N=0, Z=1, C=1, V=0
[20:09:02 665][libmx.so 0x0d964] [e00314aa] 0x4000d964: "mov x0, x20" x20=0x403542f0 => x0=0x403542f0
[20:09:02 665][libmx.so 0x0d968] [fd7b43a9] 0x4000d968: "ldp x29, x30, [sp, #0x30]" x29=0xbffff680 x30=0x4000d958 sp=0xbffff650 => x29=0xbffff6f0 x30=0x4000e540
[20:09:02 665][libmx.so 0x0d96c] [f44f42a9] 0x4000d96c: "ldp x20, x19, [sp, #0x20]" x20=0x403542f0 x19=0x40037014 sp=0xbffff650 => x20=0x116 x19=0xfffe1640
[20:09:02 666][libmx.so 0x0d970] [f65741a9] 0x4000d970: "ldp x22, x21, [sp, #0x10]" x22=0x0 x21=0x40354318 sp=0xbffff650 => x22=0x40034000 x21=0x51891008
[20:09:02 666][libmx.so 0x0d974] [f70744f8] 0x4000d974: "ldr x23, [sp], #0x40" sp=0xbffff650 => x23=0x40367d00 sp=0xbffff690
[20:09:02 666][libmx.so 0x0d978] [c0035fd6] 0x4000d978: "ret"
[20:09:02 666][libmx.so 0x0e540] [680240f9] 0x4000e540: "ldr x8, [x19]" x19=0xfffe1640 => x8=0xfffe0ef0
[20:09:02 666][libmx.so 0x0e544] [e10300aa] 0x4000e544: "mov x1, x0" x0=0x403542f0 => x1=0x403542f0
[20:09:02 666][libmx.so 0x0e548] [029d42f9] 0x4000e548: "ldr x2, [x8, #0x538]" x8=0xfffe0ef0 => x2=0xfffe0c00
[20:09:02 666][libmx.so 0x0e54c] [04000014] 0x4000e54c: "b #0x4000e55c"
[20:09:02 667][libmx.so 0x0e55c] [e00313aa] 0x4000e55c: "mov x0, x19" x19=0xfffe1640 => x0=0xfffe1640
[20:09:02 667][libmx.so 0x0e560] [fd7b46a9] 0x4000e560: "ldp x29, x30, [sp, #0x60]" x29=0xbffff6f0 x30=0x4000e540 sp=0xbffff690 => x29=0x0 x30=0x7ffff0000
[20:09:02 667][libmx.so 0x0e564] [f44f45a9] 0x4000e564: "ldp x20, x19, [sp, #0x50]" x20=0x116 x19=0xfffe1640 sp=0xbffff690 => x20=0x0 x19=0x0
[20:09:02 667][libmx.so 0x0e568] [f65744a9] 0x4000e568: "ldp x22, x21, [sp, #0x40]" x22=0x40034000 x21=0x51891008 sp=0xbffff690 => x22=0x0 x21=0x0
[20:09:02 667][libmx.so 0x0e56c] [f85f43a9] 0x4000e56c: "ldp x24, x23, [sp, #0x30]" x24=0x40032048 x23=0x40367d00 sp=0xbffff690 => x24=0x0 x23=0x0
[20:09:02 667][libmx.so 0x0e570] [fa6742a9] 0x4000e570: "ldp x26, x25, [sp, #0x20]" x26=0xb x25=0x6 sp=0xbffff690 => x26=0x0 x25=0x0
[20:09:02 667][libmx.so 0x0e574] [fc6f41a9] 0x4000e574: "ldp x28, x27, [sp, #0x10]" x28=0x8 x27=0xa sp=0xbffff690 => x28=0x0 x27=0x0
[20:09:02 667][libmx.so 0x0e578] [ffc30191] 0x4000e578: "add sp, sp, #0x70" sp=0xbffff690 => sp=0xbffff700
[20:09:02 668][libmx.so 0x0e57c] [40001fd6] 0x4000e57c: "br x2" x2=0xfffe0c00
```

综合而言，Unidbg 指令追踪绝对值得一用！下面开始简单的实践。首先是如何在某个函数执行前这个时机开启指令追踪——调用 `traceCode`即可开启。

```java
public void callSignV1(){
    // 开启指令追踪和记录
    emulator.traceCode();
    String arg1 = "https://zuul.aikucun.com/aggregation-center-facade/api/app/search/product/image/switch?" +
            "appid=38741001&did=6d2fe7c7702721c6b797cf22ec8f5f58&noncestr=4b373c&timestamp=1662394452&zuul=1";
    String arg2 = "4b373c";
    String arg3 = "1662394452";
    String result = security.callStaticJniMethodObject(emulator, "signV1", arg1, arg2, arg3).getValue().toString();
    System.out.println("call SignV1 result:"+result);
}
```

然后是细化的各种需求

### 3.1 约束范围

一般情况下，我们并不关心在目标 SO 地址范围外的执行流，比如 libc 中的函数执行流。原因也很简单，这些函数的作用往往已知，比如 libc 中的 strlen 等等，因此对它们做指令追踪没有必要，还会混淆视听。

`emulator.traceCode()`会进行无差别无限制的指令追踪，像下面这样，所有模块的执行流都会被记录。

![](https://cdn.nlark.com/yuque/0/2022/png/707476/1663852088323-557850bd-4790-45fb-af26-652690627d88.png)

我们可以使用它的重载方法`traceCode(long begin, long end)`，如下所示，只追踪目标 SO 地址范围里的指令。

```java
public void callSignV1(){
    emulator.traceCode(module.base, module.base + module.size);
    String arg1 = "https://zuul.aikucun.com/aggregation-center-facade/api/app/search/product/image/switch?" +
            "appid=38741001&did=6d2fe7c7702721c6b797cf22ec8f5f58&noncestr=4b373c&timestamp=1662394452&zuul=1";
    String arg2 = "4b373c";
    String arg3 = "1662394452";
    String result = security.callStaticJniMethodObject(emulator, "signV1", arg1, arg2, arg3).getValue().toString();
    System.out.println("call SignV1 result:"+result);
}
```

在具体使用上，很多读者会将参数二误以为是长度，写成`emulator.traceCode(module.base, module.size)`，要留心避免这个问题。

在实现逻辑上，它就是简单判断当前所处理的地址是否在这个线性区间里。

```java
private boolean canTrace(long address) {
    return (traceBegin > traceEnd || (address >= traceBegin && address <= traceEnd));
}
```

### 3.2 开启时机

要注意 trace 的时机，如果我们想 trace 从某个函数开始的执行流，那就让 traceCode 早于它执行即可。比如想 trace 从 JNI_OnLoad 开始的目标 SO 执行流，在如下的代码位置添加 trace 即可。

```java
DalvikModule dm = vm.loadLibrary("mx", true);
module = dm.getModule();
security = vm.resolveClass("com/mengxiang/arch/security/MXSecurity");
// trace
emulator.traceCode(module.base, module.base + module.size);
dm.callJNI_OnLoad(emulator);
```

如果想到更早的时机开启追踪，即追踪 init_proc、init_array 这些初始化函数的执行情况，那就需要将 traceCode 放到 loadLibrary 之前调用，代码直接提前到前面是不行的，因为此时还没有取到`module`对象，自然也没有什么`module.base`和`module.size`，这时候最正确的处理办法是使用模块监听器，在模块加载的第一时间开始 trace 。

```java
memory.addModuleListener(new ModuleListener() {
    @Override
    public void onLoaded(Emulator<?> emulator, Module module) {
        if(module.name.equals("libmx.so")){
            emulator.traceCode(module.base, module.base+module.size);
        }
    }
});
DalvikModule dm = vm.loadLibrary("mx", true);
module = dm.getModule();
security = vm.resolveClass("com/mengxiang/arch/security/MXSecurity");
dm.callJNI_OnLoad(emulator);
```

### 3.3 预估耗时

等待一列不知何时到达的火车是痛苦的，所以我们需要一个办法来预估 trace 耗时。一个好的办法是使用“裸”的Unicorn codeHook，因为 trace 的耗时主要来自反汇编、分析和打印寄存器等步骤，所以如果只是 hook 记录指令条数，这会让处理速度大幅提升。因此即使复杂样本，多数也可以在 1h 内统计出指令行数。

我这里选择只记录执行条数，别的都不用，代码逻辑见`codeHook`注释下的代码。100w 以下几分钟的功夫，1000w 以下放一两小时即可，上亿行就需要留一个晚上来跑。

```java
package com.aikucun;

import com.github.unidbg.AndroidEmulator;
import com.github.unidbg.Module;
import com.github.unidbg.arm.backend.Backend;
import com.github.unidbg.arm.backend.CodeHook;
import com.github.unidbg.arm.backend.UnHook;
import com.github.unidbg.arm.backend.Unicorn2Factory;
import com.github.unidbg.linux.android.AndroidEmulatorBuilder;
import com.github.unidbg.linux.android.AndroidResolver;
import com.github.unidbg.linux.android.dvm.*;
import com.github.unidbg.memory.Memory;

import java.io.File;

public class MXSecurity extends AbstractJni {
    private final AndroidEmulator emulator;
    private final DvmClass security;
    private final VM vm;
    private final Module module;
    private long count = 0;

    public MXSecurity() {
        emulator = AndroidEmulatorBuilder.for64Bit()
                .addBackendFactory(new Unicorn2Factory(false))
                .setProcessName("com.aikucun.akapp")
                .build();
        emulator.getBackend().registerEmuCountHook(10000); // 设置执行多少条指令切换一次线程
        emulator.getSyscallHandler().setVerbose(true);
        emulator.getSyscallHandler().setEnableThreadDispatcher(true);
        Memory memory = emulator.getMemory();
        memory.setLibraryResolver(new AndroidResolver(23));
        vm = emulator.createDalvikVM(new File("unidbg-android/src/test/resources/aikucun/com.aikucun.akapp_6.0.6_60006.apk"));
        vm.setVerbose(false);
        vm.setJni(this);
        DalvikModule dm = vm.loadLibrary("mx", true);
        module = dm.getModule();
        security = vm.resolveClass("com/mengxiang/arch/security/MXSecurity");
        dm.callJNI_OnLoad(emulator);

        // codeHook
        emulator.getBackend().hook_add_new(new CodeHook() {
            @Override
            public void hook(Backend backend, long address, int size, Object user) {
                count += 1;
            }

            @Override
            public void onAttach(UnHook unHook) {

            }

            @Override
            public void detach() {

            }
        }, module.base, module.base + module.size, null);


    }

    public void callinit(){
        DvmObject<?> context = vm.resolveClass("com.aikucun.akapp.AppContext").newObject(null);
        int ret = security.callStaticJniMethodInt(emulator, "init", context, false);
        System.out.println("call init:"+ret);
    }

    public void callSignV1(){
        String arg1 = "https://zuul.aikucun.com/aggregation-center-facade/api/app/search/product/image/switch?" +
                "appid=38741001&did=6d2fe7c7702721c6b797cf22ec8f5f58&noncestr=4b373c&timestamp=1662394452&zuul=1";
        String arg2 = "4b373c";
        String arg3 = "1662394452";
        String result = security.callStaticJniMethodObject(emulator, "signV1", arg1, arg2, arg3).getValue().toString();
        System.out.println("call SignV1 result:"+result);
    }

    public static void main(String[] args) {
        MXSecurity mxSecurity = new MXSecurity();
        mxSecurity.callinit();
        mxSecurity.callSignV1();
        System.out.println("count:"+ mxSecurity.count);
    }

    @Override
    public DvmObject<?> callObjectMethodV(BaseVM vm, DvmObject<?> dvmObject, String signature, VaList vaList) {
        switch (signature){
            case "com/aikucun/akapp/AppContext->getPackageManager()Landroid/content/pm/PackageManager;":{
                return vm.resolveClass("android/content/pm/PackageManager").newObject(null);
            }
            case "com/aikucun/akapp/AppContext->getPackageName()Ljava/lang/String;":{
                return new StringObject(vm, vm.getPackageName());
            }
        }
        return super.callObjectMethodV(vm, dvmObject, signature, vaList);
    }
}
```

### 3.4 限定函数

如果想只 trace 某个函数内的汇编，该怎么处理？以本篇为例。

![](https://cdn.nlark.com/yuque/0/2022/png/707476/1663856847582-567ffaf7-e0f6-42e6-a831-3cc6dbb1c865.png)

假如我们只想对上图里的 digest 函数做 traceCode，不想关注除此之外的其他任意部分，该如何实现？这需要结合 Hook，具体场景有下面两种。

一是我们只关注 digest 函数在图中这个地址所发生的调用，而不关注于 digest 在别处的调用。那么首先转到汇编界面，确认调用点是 0xE53C。

```java
.text:000000000000E530                 MOV             X0, X19
.text:000000000000E534                 MOV             X2, X21
.text:000000000000E538                 LDR             X1, [X8] ; "SHA1"
.text:000000000000E53C                 BL              ._Z6digestP7_JNIEnvPKcP11_jbyteArray ; digest(_JNIEnv *,char const*,_jbyteArray *)
.text:000000000000E540                 LDR             X8, [X19]
.text:000000000000E544                 MOV             X1, X0
.text:000000000000E548                 LDR             X2, [X8,#0x538]
.text:000000000000E54C                 B               loc_E55C
```

代码见下面的 traceDigest 函数，在 0xE53C 开始 traceCode，0xE540 处停止 trace。

```java
package com.aikucun;

import com.github.unidbg.AndroidEmulator;
import com.github.unidbg.Emulator;
import com.github.unidbg.Module;
import com.github.unidbg.TraceHook;
import com.github.unidbg.arm.backend.Unicorn2Factory;
import com.github.unidbg.debugger.BreakPointCallback;
import com.github.unidbg.linux.android.AndroidEmulatorBuilder;
import com.github.unidbg.linux.android.AndroidResolver;
import com.github.unidbg.linux.android.dvm.*;
import com.github.unidbg.memory.Memory;

import java.io.File;

public class MXSecurity extends AbstractJni {
    private final AndroidEmulator emulator;
    private final DvmClass security;
    private final VM vm;
    private final Module module;
    private TraceHook traceHook;

    public MXSecurity() {
        emulator = AndroidEmulatorBuilder.for64Bit()
                .addBackendFactory(new Unicorn2Factory(false))
                .setProcessName("com.aikucun.akapp")
                .build();
        emulator.getBackend().registerEmuCountHook(10000); // 设置执行多少条指令切换一次线程
        emulator.getSyscallHandler().setVerbose(true);
        emulator.getSyscallHandler().setEnableThreadDispatcher(true);
        Memory memory = emulator.getMemory();
        memory.setLibraryResolver(new AndroidResolver(23));
        vm = emulator.createDalvikVM(new File("unidbg-android/src/test/resources/aikucun/com.aikucun.akapp_6.0.6_60006.apk"));
        vm.setVerbose(true);
        vm.setJni(this);
        DalvikModule dm = vm.loadLibrary("mx", true);
        module = dm.getModule();
        security = vm.resolveClass("com/mengxiang/arch/security/MXSecurity");
        dm.callJNI_OnLoad(emulator);
        traceDigest();
    }
    
    public void traceDigest(){
        long callAddr = module.base + 0xE53C;

        emulator.attach().addBreakPoint(callAddr, new BreakPointCallback() {\
            @Override
            public boolean onHit(Emulator<?> emulator, long address) {
                traceHook = emulator.traceCode(module.base, module.base+module.size);
                return true;
            }
        });

        emulator.attach().addBreakPoint(callAddr + 4, new BreakPointCallback() {
            @Override
            public boolean onHit(Emulator<?> emulator, long address) {
                traceHook.stopTrace();
                return true;
            }
        });
    }

    public void callinit(){
        DvmObject<?> context = vm.resolveClass("com.aikucun.akapp.AppContext").newObject(null);
        int ret = security.callStaticJniMethodInt(emulator, "init", context, false);
        System.out.println("call init:"+ret);
    }

    public void callSignV1(){
        String arg1 = "https://zuul.aikucun.com/aggregation-center-facade/api/app/search/product/image/switch?" +
                "appid=38741001&did=6d2fe7c7702721c6b797cf22ec8f5f58&noncestr=4b373c&timestamp=1662394452&zuul=1";
        String arg2 = "4b373c";
        String arg3 = "1662394452";
        String result = security.callStaticJniMethodObject(emulator, "signV1", arg1, arg2, arg3).getValue().toString();
        System.out.println("call SignV1 result:"+result);
    }

    public static void main(String[] args) {
        MXSecurity mxSecurity = new MXSecurity();
        mxSecurity.callinit();
        mxSecurity.callSignV1();
    }

    @Override
    public DvmObject<?> callObjectMethodV(BaseVM vm, DvmObject<?> dvmObject, String signature, VaList vaList) {
        switch (signature){
            case "com/aikucun/akapp/AppContext->getPackageManager()Landroid/content/pm/PackageManager;":{
                return vm.resolveClass("android/content/pm/PackageManager").newObject(null);
            }
            case "com/aikucun/akapp/AppContext->getPackageName()Ljava/lang/String;":{
                return new StringObject(vm, vm.getPackageName());
            }
        }
        return super.callObjectMethodV(vm, dvmObject, signature, vaList);
    }
}
```

二是我们关注调用 digest 的所有位置，或者说只要是调用 digest 我们就关心。

![](https://cdn.nlark.com/yuque/0/2022/png/707476/1663857586570-87d9e2c7-5020-4e82-831d-2320002a92a5.png)

代码如下，在进入 digest 函数前开始 traceCode，离开 digest 函数即停止 trace。

```java
public void traceDigest(){
    long callAddr = module.base + 0xd804;

    emulator.attach().addBreakPoint(callAddr, new BreakPointCallback() {
        @Override
        public boolean onHit(Emulator<?> emulator, long address) {
            RegisterContext registerContext = emulator.getContext();
            traceHook = emulator.traceCode(module.base, module.base+module.size);
            emulator.attach().addBreakPoint(registerContext.getLR(), new BreakPointCallback() {
                @Override
                public boolean onHit(Emulator<?> emulator, long address) {
                    traceHook.stopTrace();
                    return true;
                }
            });
            return true;
        }
    });
}
```

看 traceCode 源码可见，如果目标函数被大量调用，就会初始化大量的`AssemblyCodeDumper`，这不是一个单例模式，所以最好我们自己处理一下。

```java
@Override
public TraceHook traceCode(long begin, long end, TraceCodeListener listener) {
    AssemblyCodeDumper hook = new AssemblyCodeDumper(this, begin, end, listener);
    backend.hook_add_new(hook, begin, end, this);
    return hook;
}
```

可以直接封装 AssemblyCodeDumper，那么就只用初始化一次了，一定程度上减小了开销，但代码变麻烦了一些，读者根据自己的喜好选择。

```java
public void traceDigest(){
    final long begin = module.base;
    final long end = module.base+module.size;
    final AssemblyCodeDumper hook = new AssemblyCodeDumper(emulator, begin, end, null);
    long funAddr = module.base + 0xd804;
    emulator.attach().addBreakPoint(funAddr, new BreakPointCallback() {
        @Override
        public boolean onHit(Emulator<?> emulator, long address) {
            // OnEnter
            RegisterContext registerContext = emulator.getContext();
            emulator.getBackend().hook_add_new(hook, begin, end, this);
            emulator.attach().addBreakPoint(registerContext.getLR(), new BreakPointCallback() {
                @Override
                public boolean onHit(Emulator<?> emulator, long address) {
                    hook.stopTrace();
                    return true;
                }
            });
            return true;
        }
    });
}
```

### 3.5 持久化

我们希望将 trace 重定向到文件，以便做进一步分析，而不是在控制台疯狂输出。示例代码如下。

```java
// 导入相关类库
import java.io.FileNotFoundException;
import java.io.FileOutputStream;
import java.io.PrintStream;

// 持久化写入 trace.log        
String traceFile = "yourpath/trace.log";
PrintStream traceStream;
try {
    traceStream = new PrintStream(new FileOutputStream(traceFile), true);
    // 和上面一样，可以自由设置起始和终止地址等等
    emulator.traceCode(module.base, module.base+module.size).setRedirect(traceStream);
} catch (FileNotFoundException e) {
    e.printStackTrace();
}
```

## 四、展示效果的实现

上节讨论了 traceCode 的基本用法，这里我们讨论它如何做到这种展示效果。首先看最朴素的展示效果

```java
[=] 	047c61d4 [libtest.so + 0x00c1d4]  01 30 40 e2   sub  r3, r0, #1
[=] 	r0         : 0x90001ec1
[=] 	r1         : 0x90001c32
[=] 	r2         : 0x2e
[=] 	r3         : 0x90001ec1
[=] 	r4         : 0x90001c38
[=] 	r5         : 0x47ebabc
[=] 	r6         : 0x13
[=] 	r7         : 0x90001eb0
[=] 	r8         : 0x90001c20
[=] 	r9         : 0x12
[=] 	r10        : 0x9
[=] 	r11        : 0x2f
[=] 	r12        : 0x0
[=] 	sp         : 0x7ff3c650
[=] 	lr         : 0x6f732e63
[=] 	pc         : 0x47c61d4
[=] 	cpsr       : 0x200001d3
```

我们称之为基本展示效果，来看看如何实现。

### 4.1 基本展示效果

Unidbg 的 code trace 能力来源于 Unicorn，所以我们直接看 Unicorn 即可。Unicorn 支持多种语言的绑定，我这里选择 Python 进行演示，首先写一个简易 demo。

```java
from unicorn import *

# code
# 200080D2
# 410080D2
# 0000018B
# disasm
# mov x0,1 x0 = 1
# mov x1,2 x1 = 2
# add x0,x0,x1 x0 = x1 + x0 = 3
ARM64_CODE = bytes.fromhex("200080D2410080D20000018B")

# memory address where emulation starts
ADDRESS = 0x10000

# Initialize emulator in ARM mode
mu = Uc(UC_ARCH_ARM64, UC_MODE_ARM)
# map 2MB memory for this emulation
# align 1024
mu.mem_map(ADDRESS, 2 * 1024 * 1024)
mu.mem_write(ADDRESS, ARM64_CODE)

mu.emu_start(ADDRESS, ADDRESS + len(ARM64_CODE))
```

上面是最简单的一个 demo，它执行了三条 ARM64 指令。接下来我添加一个 code hook，对应于代码中的`mu.hook_add`。

```java
from unicorn import *


# callback for tracing instructions
def hook_code(uc, address, size, user_data):
    code = mu.mem_read(address, size)
    print("run Code:"+code.hex())
    print(">>> Tracing instruction at 0x%x, instruction size = 0x%x" %(address, size))


# code
# 200080D2
# 410080D2
# 0000018B
# disasm
# mov x0,1 x0 = 1
# mov x1,2 x1 = 2
# add x0,x0,x1 x0 = x1 + x0 = 3
ARM64_CODE = bytes.fromhex("200080D2410080D20000018B")

# memory address where emulation starts
ADDRESS = 0x10000

# Initialize emulator in ARM mode
mu = Uc(UC_ARCH_ARM64, UC_MODE_ARM)
# map 2MB memory for this emulation
# align 1024
mu.mem_map(ADDRESS, 2 * 1024 * 1024)
mu.mem_write(ADDRESS, ARM64_CODE)

# tracing one instruction with customized callback
mu.hook_add(UC_HOOK_CODE, hook_code, begin=ADDRESS, end=ADDRESS+len(ARM64_CODE))

mu.emu_start(ADDRESS, ADDRESS + len(ARM64_CODE))
```

介绍一下`hook_add`函数

参数 1 是 Hook 类型，Unicorn 支持非常多种 Hook，包括指令 Hook、中断 Hook、基本块 Hook、内存 Hook、内存异常 Hook 等等。`UC_HOOK_CODE`就是本篇的主题指令 Hook。

参数 2 是回调函数，对于`UC_HOOK_CODE`而言，这意味着每行汇编代码执行前，都会进入这个回调函数。同理可得，如果设置为基本块 Hook，那就是每个基本块执行前，都会进入这个回调函数。

参数 3 和 4 是 起始地址和结束地址，即所谓的 Hook 范围。

接下来重点看`hook_code`这个回调函数，它的参数 1 是模拟器实例，参数 2 是当前地址，参数 3 是大小，如果是基本块类型的回调处理函数，那指的就是基本块的大小。如果是指令类型的回调处理函数，指的就是指令大小。因为我们这里是 ARM64 指令，它是定长指令，总是四个字节长度，在 ARM 的 Thumb 模式下或 X86 这样的变长指令架构中，这个参数会很有用，可以帮助我们确定指令长度。最后一个参数是用户数据，它其实是 hook_add 的一个参数，此处不表，用到再说。

```java
# callback for tracing instructions
def hook_code(uc, address, size, user_data):
    code = mu.mem_read(address, size)
    print("run Code:"+code.hex())
    print(">>> Tracing instruction at 0x%x, instruction size = 0x%x" %(address, size))
```

我们的函数很简单，只是打印了当前地址、机器码、指令大小这三个信息，运行效果也很朴素。

```java
run Code:200080d2
>>> Tracing instruction at 0x10000, instruction size = 0x4
run Code:410080d2
>>> Tracing instruction at 0x10004, instruction size = 0x4
run Code:0000018b
>>> Tracing instruction at 0x10008, instruction size = 0x4
```

我们还需要添加反汇编以及寄存器展示，首先处理寄存器。`uc.reg_read`用于读取寄存器的值，写一个`showRegisterStatus`函数，打印 X0-X32 的所有寄存器。

```java
from unicorn import *


def showRegisterStatus(uc):
    out = ""
    out += "X0: %016X\tX1: %016X\n" % (uc.reg_read(
        unicorn.arm64_const.UC_ARM64_REG_X0), uc.reg_read(unicorn.arm64_const.UC_ARM64_REG_X1))
    out += "X2: %016X\tX3: %016X\n" % (uc.reg_read(
        unicorn.arm64_const.UC_ARM64_REG_X2),  uc.reg_read(unicorn.arm64_const.UC_ARM64_REG_X3))
    out += "X4: %016X\tX5: %016X\n" % (uc.reg_read(
        unicorn.arm64_const.UC_ARM64_REG_X4), uc.reg_read(unicorn.arm64_const.UC_ARM64_REG_X5))
    out += "X6: %016X\tX7: %016X\n" % (uc.reg_read(
        unicorn.arm64_const.UC_ARM64_REG_X6), uc.reg_read(unicorn.arm64_const.UC_ARM64_REG_X7))
    out += "X8: %016X\tX9: %016X\n" % (uc.reg_read(
        unicorn.arm64_const.UC_ARM64_REG_X8), uc.reg_read(unicorn.arm64_const.UC_ARM64_REG_X9))
    out += "X10: %016X\tX11: %016X\n" % ( uc.reg_read(
        unicorn.arm64_const.UC_ARM64_REG_X10),  uc.reg_read(unicorn.arm64_const.UC_ARM64_REG_X11))
    out += "X12: %016X\tX13: %016X\n" % ( uc.reg_read(
        unicorn.arm64_const.UC_ARM64_REG_X12),  uc.reg_read(unicorn.arm64_const.UC_ARM64_REG_X13))
    out += "X14: %016X\tX15: %016X\n" % ( uc.reg_read(
        unicorn.arm64_const.UC_ARM64_REG_X14),  uc.reg_read(unicorn.arm64_const.UC_ARM64_REG_X15))
    out += "X16: %016X\tX17: %016X\n" % ( uc.reg_read(
        unicorn.arm64_const.UC_ARM64_REG_X16),  uc.reg_read(unicorn.arm64_const.UC_ARM64_REG_X17))
    out += "X18: %016X\tX19: %016X\n" % ( uc.reg_read(
        unicorn.arm64_const.UC_ARM64_REG_X18),  uc.reg_read(unicorn.arm64_const.UC_ARM64_REG_X19))
    out += "X20: %016X\tX21: %016X\n" % ( uc.reg_read(
        unicorn.arm64_const.UC_ARM64_REG_X20),  uc.reg_read(unicorn.arm64_const.UC_ARM64_REG_X21))
    out += "X22: %016X\tX23: %016X\n" % ( uc.reg_read(
        unicorn.arm64_const.UC_ARM64_REG_X22),  uc.reg_read(unicorn.arm64_const.UC_ARM64_REG_X23))
    out += "X24: %016X\tX25: %016X\n" % ( uc.reg_read(
        unicorn.arm64_const.UC_ARM64_REG_X24),  uc.reg_read(unicorn.arm64_const.UC_ARM64_REG_X25))
    out += "X26: %016X\tX27: %016X\n" % ( uc.reg_read(
        unicorn.arm64_const.UC_ARM64_REG_X26),  uc.reg_read(unicorn.arm64_const.UC_ARM64_REG_X27))
    out += "X28: %016X\tX29: %016X\n" % ( uc.reg_read(
        unicorn.arm64_const.UC_ARM64_REG_X28),  uc.reg_read(unicorn.arm64_const.UC_ARM64_REG_X29))
    out += "X30: %016X\n" % (uc.reg_read(unicorn.arm64_const.UC_ARM64_REG_X30))
    out += "PC: %016X\n" % (uc.reg_read(unicorn.arm64_const.UC_ARM64_REG_PC))
    out += "SP: %016X\n" % (uc.reg_read(unicorn.arm64_const.UC_ARM64_REG_SP))
    return out


# callback for tracing instructions
def hook_code(uc, address, size, user_data):
    code = mu.mem_read(address, size)
    print("run Code:"+code.hex())
    print(">>> Tracing instruction at 0x%x, instruction size = 0x%x" %(address, size))
    print(showRegisterStatus(uc))


# code
# 200080D2
# 410080D2
# 0000018B
# disasm
# mov x0,1 x0 = 1
# mov x1,2 x1 = 2
# add x0,x0,x1 x0 = x1 + x0 = 3
ARM64_CODE = bytes.fromhex("200080D2410080D20000018B")

# memory address where emulation starts
ADDRESS = 0x10000

# Initialize emulator in ARM mode
mu = Uc(UC_ARCH_ARM64, UC_MODE_ARM)
# map 2MB memory for this emulation
# align 1024
mu.mem_map(ADDRESS, 2 * 1024 * 1024)
mu.mem_write(ADDRESS, ARM64_CODE)

# tracing one instruction with customized callback
mu.hook_add(UC_HOOK_CODE, hook_code, begin=ADDRESS, end=ADDRESS+len(ARM64_CODE))

mu.emu_start(ADDRESS, ADDRESS + len(ARM64_CODE))
```

运行效果如下

```java
run Code:0000018b
>>> Tracing instruction at 0x10008, instruction size = 0x4
X0: 0000000000000001	X1: 0000000000000002
X2: 0000000000000000	X3: 0000000000000000
X4: 0000000000000000	X5: 0000000000000000
X6: 0000000000000000	X7: 0000000000000000
X8: 0000000000000000	X9: 0000000000000000
X10: 0000000000000000	X11: 0000000000000000
X12: 0000000000000000	X13: 0000000000000000
X14: 0000000000000000	X15: 0000000000000000
X16: 0000000000000000	X17: 0000000000000000
X18: 0000000000000000	X19: 0000000000000000
X20: 0000000000000000	X21: 0000000000000000
X22: 0000000000000000	X23: 0000000000000000
X24: 0000000000000000	X25: 0000000000000000
X26: 0000000000000000	X27: 0000000000000000
X28: 0000000000000000	X29: 0000000000000000
X30: 0000000000000000
PC: 0000000000010008
SP: 0000000000000000
```

接下来考虑反汇编，我们一般使用 Unicorn 的兄弟项目 Capstone 来完成这件事，请读者自行安装。

```java
from unicorn import *
from capstone import *


def showRegisterStatus(uc):
    out = ""
    out += "X0: %016X\tX1: %016X\n" % (uc.reg_read(
        unicorn.arm64_const.UC_ARM64_REG_X0), uc.reg_read(unicorn.arm64_const.UC_ARM64_REG_X1))
    out += "X2: %016X\tX3: %016X\n" % (uc.reg_read(
        unicorn.arm64_const.UC_ARM64_REG_X2),  uc.reg_read(unicorn.arm64_const.UC_ARM64_REG_X3))
    out += "X4: %016X\tX5: %016X\n" % (uc.reg_read(
        unicorn.arm64_const.UC_ARM64_REG_X4), uc.reg_read(unicorn.arm64_const.UC_ARM64_REG_X5))
    out += "X6: %016X\tX7: %016X\n" % (uc.reg_read(
        unicorn.arm64_const.UC_ARM64_REG_X6), uc.reg_read(unicorn.arm64_const.UC_ARM64_REG_X7))
    out += "X8: %016X\tX9: %016X\n" % (uc.reg_read(
        unicorn.arm64_const.UC_ARM64_REG_X8), uc.reg_read(unicorn.arm64_const.UC_ARM64_REG_X9))
    out += "X10: %016X\tX11: %016X\n" % ( uc.reg_read(
        unicorn.arm64_const.UC_ARM64_REG_X10),  uc.reg_read(unicorn.arm64_const.UC_ARM64_REG_X11))
    out += "X12: %016X\tX13: %016X\n" % ( uc.reg_read(
        unicorn.arm64_const.UC_ARM64_REG_X12),  uc.reg_read(unicorn.arm64_const.UC_ARM64_REG_X13))
    out += "X14: %016X\tX15: %016X\n" % ( uc.reg_read(
        unicorn.arm64_const.UC_ARM64_REG_X14),  uc.reg_read(unicorn.arm64_const.UC_ARM64_REG_X15))
    out += "X16: %016X\tX17: %016X\n" % ( uc.reg_read(
        unicorn.arm64_const.UC_ARM64_REG_X16),  uc.reg_read(unicorn.arm64_const.UC_ARM64_REG_X17))
    out += "X18: %016X\tX19: %016X\n" % ( uc.reg_read(
        unicorn.arm64_const.UC_ARM64_REG_X18),  uc.reg_read(unicorn.arm64_const.UC_ARM64_REG_X19))
    out += "X20: %016X\tX21: %016X\n" % ( uc.reg_read(
        unicorn.arm64_const.UC_ARM64_REG_X20),  uc.reg_read(unicorn.arm64_const.UC_ARM64_REG_X21))
    out += "X22: %016X\tX23: %016X\n" % ( uc.reg_read(
        unicorn.arm64_const.UC_ARM64_REG_X22),  uc.reg_read(unicorn.arm64_const.UC_ARM64_REG_X23))
    out += "X24: %016X\tX25: %016X\n" % ( uc.reg_read(
        unicorn.arm64_const.UC_ARM64_REG_X24),  uc.reg_read(unicorn.arm64_const.UC_ARM64_REG_X25))
    out += "X26: %016X\tX27: %016X\n" % ( uc.reg_read(
        unicorn.arm64_const.UC_ARM64_REG_X26),  uc.reg_read(unicorn.arm64_const.UC_ARM64_REG_X27))
    out += "X28: %016X\tX29: %016X\n" % ( uc.reg_read(
        unicorn.arm64_const.UC_ARM64_REG_X28),  uc.reg_read(unicorn.arm64_const.UC_ARM64_REG_X29))
    out += "X30: %016X\n" % (uc.reg_read(unicorn.arm64_const.UC_ARM64_REG_X30))
    out += "PC: %016X\n" % (uc.reg_read(unicorn.arm64_const.UC_ARM64_REG_PC))
    out += "SP: %016X\n" % (uc.reg_read(unicorn.arm64_const.UC_ARM64_REG_SP))
    return out


def disassemble(code, addr):
    return next(cs.disasm(bytes(code), addr, 1))


# callback for tracing instructions
def hook_code(uc, address, size, user_data):
    code = mu.mem_read(address, size)
    insn = disassemble(code, address)
    print("code:", code.hex())
    print("asm:%s %s" % (insn.mnemonic, insn.op_str))
    print(">>> Tracing instruction at 0x%x, instruction size = 0x%x" %(address, size))
    print(showRegisterStatus(uc))


# code
# 200080D2
# 410080D2
# 0000018B
# disasm
# mov x0,1 x0 = 1
# mov x1,2 x1 = 2
# add x0,x0,x1 x0 = x1 + x0 = 3
ARM64_CODE = bytes.fromhex("200080D2410080D20000018B")

# memory address where emulation starts
ADDRESS = 0x10000

# Initialize emulator in ARM mode
mu = Uc(UC_ARCH_ARM64, UC_MODE_ARM)
cs = Cs(CS_ARCH_ARM64, CS_MODE_ARM)
# map 2MB memory for this emulation
# align 1024
mu.mem_map(ADDRESS, 2 * 1024 * 1024)
mu.mem_write(ADDRESS, ARM64_CODE)

# tracing one instruction with customized callback
mu.hook_add(UC_HOOK_CODE, hook_code, begin=ADDRESS, end=ADDRESS+len(ARM64_CODE))

mu.emu_start(ADDRESS, ADDRESS + len(ARM64_CODE))
```

`cs.disasm`返回一个迭代器，我们使用 next 函数获取返回的指令对象。它有很多属性，我们这里先只关注 mnemonic —— 所谓的指令助记符，或者说操作码，以及 op_str —— 操作数。比如`mov x0, #1`这样一条指令，`mov`就是操作码，`x0, #1`就是操作数。

这么修改后，打印效果如下

```java
code: 200080d2
asm:mov x0, #1
>>> Tracing instruction at 0x10000, instruction size = 0x4
X0: 0000000000000000	X1: 0000000000000000
X2: 0000000000000000	X3: 0000000000000000
X4: 0000000000000000	X5: 0000000000000000
X6: 0000000000000000	X7: 0000000000000000
X8: 0000000000000000	X9: 0000000000000000
X10: 0000000000000000	X11: 0000000000000000
X12: 0000000000000000	X13: 0000000000000000
X14: 0000000000000000	X15: 0000000000000000
X16: 0000000000000000	X17: 0000000000000000
X18: 0000000000000000	X19: 0000000000000000
X20: 0000000000000000	X21: 0000000000000000
X22: 0000000000000000	X23: 0000000000000000
X24: 0000000000000000	X25: 0000000000000000
X26: 0000000000000000	X27: 0000000000000000
X28: 0000000000000000	X29: 0000000000000000
X30: 0000000000000000
PC: 0000000000010000
SP: 0000000000000000
```

相比较参照物，似乎差不多了。其中相对偏移和所处模块这两个信息没有体现出来，因为我们的 demo 仅仅处理简单的汇编指令，不涉及到模块这种更高的概念。

```java
[=] 	047c61d4 [libtest.so + 0x00c1d4]  01 30 40 e2   sub  r3, r0, #1
[=] 	r0         : 0x90001ec1
[=] 	r1         : 0x90001c32
[=] 	r2         : 0x2e
[=] 	r3         : 0x90001ec1
[=] 	r4         : 0x90001c38
[=] 	r5         : 0x47ebabc
[=] 	r6         : 0x13
[=] 	r7         : 0x90001eb0
[=] 	r8         : 0x90001c20
[=] 	r9         : 0x12
[=] 	r10        : 0x9
[=] 	r11        : 0x2f
[=] 	r12        : 0x0
[=] 	sp         : 0x7ff3c650
[=] 	lr         : 0x6f732e63
[=] 	pc         : 0x47c61d4
[=] 	cpsr       : 0x200001d3
```

在这里基本信息的处理上，Unidbg 和我们一样，没什么新奇。比如下面是获取反汇编代码的逻辑，也是依赖 Capstone 去做解析。

```java
@Override
public Instruction[] disassemble(long address, int size, long count) {
    byte[] code = backend.mem_read(address, size);
    return createArm64Disassembler().disasm(code, address, count);
}
```

接下来要考虑如何优化展示效果。

### 4.2 更好的展示效果

以`add x0, x0, x1`这条指令为例，打印所有指令并无意义，**过量的、冗余的信息会反向干扰正常分析**。

```java
code: 0000018b
asm:add x0, x0, x1
>>> Tracing instruction at 0x10008, instruction size = 0x4
X0: 0000000000000001	X1: 0000000000000002
X2: 0000000000000000	X3: 0000000000000000
X4: 0000000000000000	X5: 0000000000000000
X6: 0000000000000000	X7: 0000000000000000
X8: 0000000000000000	X9: 0000000000000000
X10: 0000000000000000	X11: 0000000000000000
X12: 0000000000000000	X13: 0000000000000000
X14: 0000000000000000	X15: 0000000000000000
X16: 0000000000000000	X17: 0000000000000000
X18: 0000000000000000	X19: 0000000000000000
X20: 0000000000000000	X21: 0000000000000000
X22: 0000000000000000	X23: 0000000000000000
X24: 0000000000000000	X25: 0000000000000000
X26: 0000000000000000	X27: 0000000000000000
X28: 0000000000000000	X29: 0000000000000000
X30: 0000000000000000
PC: 0000000000010008
SP: 0000000000000000
```

我们只希望了解**最重要的信息**。什么是最重要的信息？不同的人会给出不同的答案，下面讨论几个”答案“。

#### 4.2.1 方案一

有人认为最重要的信息是目标寄存器。

对于`add x0, x0, x1`，指令的两个操作数分别是 x0、x1，目标寄存器是 x0，新 x0 = x0 + x1，这种观点认为应该展示新 x0，像下面这样。

```java
add x0, x0, x1; x0 = value
```

听起来不错，在实现上要考虑两个问题，1 是确定打印哪些寄存器， 2 是如何打印所需寄存器。

先考虑问题一，目标寄存器或者说我们所关心的寄存器，一般是操作数的第一个数。

比如`sub x8, x29, #0x20`，即 x8 = x29 - 0x20，我们关心 x8。

比如`ldur x9, [x29, #-0x20]`，即读取 x29 - 0x20 地址处的值存储到 x9 寄存器，我们关心 x9。

比如`mov x1, x21`，将 x21 的值赋给 x1，我们关心 x1。

因此可以判断指令是否有操作数，比如`ret`指令就没有操作数；然后判断指令第一个操作数是否是寄存器（而不是立即数或其他），比如`B addr`就可以跟着立即数。接下来用 Capstone 去实践。

首先要开启“详细模式”

```java
cs = Cs(CS_ARCH_ARM64, CS_MODE_ARM)
# 开启细节模式
cs.detail = True
```

打开它可以获得更丰富的反汇编信息，如果不主动打开就只做有限的反汇编和分析，这种设计很合理。用户并不总是需要高级的分析功能，它们会很耗时。

打开前

![](https://cdn.nlark.com/yuque/0/2022/png/707476/1667431931208-870f9d36-6b9d-45d6-989a-0b4a36bdc020.png)

打开后

![](https://cdn.nlark.com/yuque/0/2022/png/707476/1667431789043-c8f26250-d049-4a34-8706-c96b2ee37056.png)

这里我们需要对操作数做分析，以及后续还有一些别的需求，所以打开了详细模式。接下来简单调一下 API，用以取到上文所描述的所需寄存器。

```java
def getTargetRegistrer(insn):
    # 如果指令无操作数
    if len(insn.operands) == 0:
        return ""
    op1 = insn.operands[0]
    # 如果指令第一个操作数不是寄存器
    if op1.type != ARM64_OP_REG:
        return ""
    return insn.reg_name(op1.value.reg)
```

新的完整代码如下

```java
from unicorn import *
from capstone import *
from capstone.arm64 import *


def showRegisterStatus(uc):
    out = ""
    out += "X0: %016X\tX1: %016X\n" % (uc.reg_read(
        unicorn.arm64_const.UC_ARM64_REG_X0), uc.reg_read(unicorn.arm64_const.UC_ARM64_REG_X1))
    out += "X2: %016X\tX3: %016X\n" % (uc.reg_read(
        unicorn.arm64_const.UC_ARM64_REG_X2), uc.reg_read(unicorn.arm64_const.UC_ARM64_REG_X3))
    out += "X4: %016X\tX5: %016X\n" % (uc.reg_read(
        unicorn.arm64_const.UC_ARM64_REG_X4), uc.reg_read(unicorn.arm64_const.UC_ARM64_REG_X5))
    out += "X6: %016X\tX7: %016X\n" % (uc.reg_read(
        unicorn.arm64_const.UC_ARM64_REG_X6), uc.reg_read(unicorn.arm64_const.UC_ARM64_REG_X7))
    out += "X8: %016X\tX9: %016X\n" % (uc.reg_read(
        unicorn.arm64_const.UC_ARM64_REG_X8), uc.reg_read(unicorn.arm64_const.UC_ARM64_REG_X9))
    out += "X10: %016X\tX11: %016X\n" % (uc.reg_read(
        unicorn.arm64_const.UC_ARM64_REG_X10), uc.reg_read(unicorn.arm64_const.UC_ARM64_REG_X11))
    out += "X12: %016X\tX13: %016X\n" % (uc.reg_read(
        unicorn.arm64_const.UC_ARM64_REG_X12), uc.reg_read(unicorn.arm64_const.UC_ARM64_REG_X13))
    out += "X14: %016X\tX15: %016X\n" % (uc.reg_read(
        unicorn.arm64_const.UC_ARM64_REG_X14), uc.reg_read(unicorn.arm64_const.UC_ARM64_REG_X15))
    out += "X16: %016X\tX17: %016X\n" % (uc.reg_read(
        unicorn.arm64_const.UC_ARM64_REG_X16), uc.reg_read(unicorn.arm64_const.UC_ARM64_REG_X17))
    out += "X18: %016X\tX19: %016X\n" % (uc.reg_read(
        unicorn.arm64_const.UC_ARM64_REG_X18), uc.reg_read(unicorn.arm64_const.UC_ARM64_REG_X19))
    out += "X20: %016X\tX21: %016X\n" % (uc.reg_read(
        unicorn.arm64_const.UC_ARM64_REG_X20), uc.reg_read(unicorn.arm64_const.UC_ARM64_REG_X21))
    out += "X22: %016X\tX23: %016X\n" % (uc.reg_read(
        unicorn.arm64_const.UC_ARM64_REG_X22), uc.reg_read(unicorn.arm64_const.UC_ARM64_REG_X23))
    out += "X24: %016X\tX25: %016X\n" % (uc.reg_read(
        unicorn.arm64_const.UC_ARM64_REG_X24), uc.reg_read(unicorn.arm64_const.UC_ARM64_REG_X25))
    out += "X26: %016X\tX27: %016X\n" % (uc.reg_read(
        unicorn.arm64_const.UC_ARM64_REG_X26), uc.reg_read(unicorn.arm64_const.UC_ARM64_REG_X27))
    out += "X28: %016X\tX29: %016X\n" % (uc.reg_read(
        unicorn.arm64_const.UC_ARM64_REG_X28), uc.reg_read(unicorn.arm64_const.UC_ARM64_REG_X29))
    out += "X30: %016X\n" % (uc.reg_read(unicorn.arm64_const.UC_ARM64_REG_X30))
    out += "PC: %016X\n" % (uc.reg_read(unicorn.arm64_const.UC_ARM64_REG_PC))
    out += "SP: %016X\n" % (uc.reg_read(unicorn.arm64_const.UC_ARM64_REG_SP))
    return out


def disassemble(code, addr):
    return next(cs.disasm(bytes(code), addr, 1))


def getTargetRegistrer(insn):
    # 如果指令无操作数
    if len(insn.operands) == 0:
        return ""
    op1 = insn.operands[0]
    # 如果指令第一个操作数不是寄存器
    if op1.type != ARM64_OP_REG:
        return ""
    return insn.reg_name(op1.value.reg)


# callback for tracing instructions
def hook_code(uc, address, size, user_data):
    code = mu.mem_read(address, size)
    insn = disassemble(code, address)
    reg = getTargetRegistrer(insn)
    print("code:", code.hex())
    print("asm:%s %s ;" % (insn.mnemonic, insn.op_str), end="")
    if reg:
        print(reg, "=")
    else:
        print("")
    print(">>> Tracing instruction at 0x%x, instruction size = 0x%x" % (address, size))
    print(showRegisterStatus(uc))


# code
# 200080D2
# 410080D2
# 0000018B
# disasm
# mov x0,1 x0 = 1
# mov x1,2 x1 = 2
# add x0,x0,x1 x0 = x1 + x0 = 3
ARM64_CODE = bytes.fromhex("200080D2410080D20000018B")

# memory address where emulation starts
ADDRESS = 0x10000

# Initialize emulator in ARM mode
mu = Uc(UC_ARCH_ARM64, UC_MODE_ARM)
cs = Cs(CS_ARCH_ARM64, CS_MODE_ARM)
cs.detail = True
# map 2MB memory for this emulation
# align 1024
mu.mem_map(ADDRESS, 2 * 1024 * 1024)
mu.mem_write(ADDRESS, ARM64_CODE)

# tracing one instruction with customized callback
mu.hook_add(UC_HOOK_CODE, hook_code, begin=ADDRESS, end=ADDRESS + len(ARM64_CODE))

mu.emu_start(ADDRESS, ADDRESS + len(ARM64_CODE))
```

运行如下

```java
code: 0000018b
asm:add x0, x0, x1 ;x0 =
>>> Tracing instruction at 0x10008, instruction size = 0x4
X0: 0000000000000001	X1: 0000000000000002
X2: 0000000000000000	X3: 0000000000000000
X4: 0000000000000000	X5: 0000000000000000
X6: 0000000000000000	X7: 0000000000000000
X8: 0000000000000000	X9: 0000000000000000
X10: 0000000000000000	X11: 0000000000000000
X12: 0000000000000000	X13: 0000000000000000
X14: 0000000000000000	X15: 0000000000000000
X16: 0000000000000000	X17: 0000000000000000
X18: 0000000000000000	X19: 0000000000000000
X20: 0000000000000000	X21: 0000000000000000
X22: 0000000000000000	X23: 0000000000000000
X24: 0000000000000000	X25: 0000000000000000
X26: 0000000000000000	X27: 0000000000000000
X28: 0000000000000000	X29: 0000000000000000
X30: 0000000000000000
PC: 0000000000010008
SP: 0000000000000000
```

看起来还不错，我们要改造一下 showRegisterStatus 函数，并不需要读取所有寄存器，只需要处理我们传入的寄存器即可。

```java
def showRegisterStatus(uc, regName):
    regID = getattr(arm64_const, 'UC_ARM64_REG_' + regName.title())
    value = uc.reg_read(regID)
    return "%s = %016X\t" % (regName, value)
```

现在完整代码如下

```java
from unicorn import *
from capstone import *
from capstone.arm64 import *
from unicorn.arm64_const import *


def showRegisterStatus(uc, regName):
    regID = getattr(arm64_const, 'UC_ARM64_REG_' + regName.title())
    value = uc.reg_read(regID)
    return "%s = %016X\t" % (regName, value)


def disassemble(code, addr):
    return next(cs.disasm(bytes(code), addr, 1))


def getTargetRegistrer(insn):
    # 如果指令无操作数
    if len(insn.operands) == 0:
        return ""
    op1 = insn.operands[0]
    # 如果指令第一个操作数不是寄存器
    if op1.type != ARM64_OP_REG:
        return ""
    return insn.reg_name(op1.value.reg)


# callback for tracing instructions
def hook_code(uc, address, size, user_data):
    code = mu.mem_read(address, size)
    insn = disassemble(code, address)
    reg = getTargetRegistrer(insn)
    print("code:",code.hex())
    print("asm:%s %s ;" % (insn.mnemonic, insn.op_str), end="")
    if reg:
        print(showRegisterStatus(uc, reg))
    else:
        print("")
    print(">>> Tracing instruction at 0x%x, instruction size = 0x%x" % (address, size))


# code
# 200080D2
# 410080D2
# 0000018B
# disasm
# mov x0,1 x0 = 1
# mov x1,2 x1 = 2
# add x0,x0,x1 x0 = x1 + x0 = 3
ARM64_CODE = bytes.fromhex("200080D2410080D20000018B")

# memory address where emulation starts
ADDRESS = 0x10000

# Initialize emulator in ARM mode
mu = Uc(UC_ARCH_ARM64, UC_MODE_ARM)
cs = Cs(CS_ARCH_ARM64, CS_MODE_ARM)
cs.detail = True
# map 2MB memory for this emulation
# align 1024
mu.mem_map(ADDRESS, 2 * 1024 * 1024)
mu.mem_write(ADDRESS, ARM64_CODE)

# tracing one instruction with customized callback
mu.hook_add(UC_HOOK_CODE, hook_code, begin=ADDRESS, end=ADDRESS + len(ARM64_CODE))

mu.emu_start(ADDRESS, ADDRESS + len(ARM64_CODE))
```

再优化一下 hook_code 函数的打印排版

```java
# callback for tracing instructions
def hook_code(uc, address, size, user_data):
    code = mu.mem_read(address, size)
    insn = disassemble(code, address)
    reg = getTargetRegistrer(insn)
    line = ""
    line += '%08X  ' % address
    line += '%s  ' % code.hex()
    line += "%s %s  ;" % (insn.mnemonic, insn.op_str)
    if reg:
        line += showRegisterStatus(uc, reg)

    print(line)
```

运行如下，看起来挺像那么回事。

```java
00010000  200080d2  mov x0, #1  ;x0 = 0000000000000000	
00010004  410080d2  mov x1, #2  ;x1 = 0000000000000000	
00010008  0000018b  add x0, x0, x1  ;x0 = 0000000000000001	
```

但我们遇到了问题二——如何打印所需寄存器。读者可以回想一下，我们的 hook_code 位于指令执行前的时机。因此打印的寄存器是指令执行前的状态，而非执行后的状态，上面的打印输出很明显的反映了这一点。

处理上并不麻烦，我们可以在下一条指令的 hook_code 里，处理上一条指令所需的寄存器打印，第二条指令执行前不就是第一条指令执行后吗？让我们再改造一下 hook_code 函数。

```java
regNeed = None
# callback for tracing instructions
def hook_code(uc, address, size, user_data):
    global regNeed
    if regNeed is not None:
        print(showRegisterStatus(uc, regNeed))
    code = mu.mem_read(address, size)
    insn = disassemble(code, address)
    regNeed = getTargetRegistrer(insn)
    line = ""
    line += '%08X  ' % address
    line += '%s  ' % code.hex()
    line += "%s %s  ;" % (insn.mnemonic, insn.op_str)

    print(line, end=" ")
```

通过 regNeed 全局变量存储需要打印的寄存器信息，在下一次 hook_code 中进行打印，运行结果看起来不错。

```java
00010000  200080d2  mov x0, #1  ; x0 = 0000000000000001	
00010004  410080d2  mov x1, #2  ; x1 = 0000000000000002	
00010008  0000018b  add x0, x0, x1  ; 
```

末尾指令会缺少指令，但就这一条指令而已，算不上什么。

#### 4.2.2 方案二

健康的社会不会只有一种声音，也不会只有一个观点，这里讨论第二个方案——认为最重要的信息是“在汇编执行前后，其值发生了变化的寄存器“。

这个方案也很朴素，就是记录每条汇编指令执行前后的寄存器状态，然后寻找差异项，对那些值发生了改变的寄存器做打印和展示。代码实现上，通过全局变量 pre_regs 保存上一次 hook_code 得到的寄存器值，没什么难理解的地方。

```java
from unicorn import *
from capstone import *


def disassemble(code, addr):
    return next(cs.disasm(bytes(code), addr, 1))


def getAllRegistrer(uc):
    values = {}
    for i in regs:
        values[i] = uc.reg_read(regs[i])
    return values


def get_diff_regs(preRegs, nowRegs):
    diffText = ""
    for i in preRegs:
        pre = preRegs[i]
        now = nowRegs[i]
        if pre != now:
            diffText += "%s = %016X\t" %(i, now)
    return diffText


# callback for tracing instructions
def hook_code(uc, address, size, user_data):
    global pre_regs
    if pre_regs is not None:
        now_regs = getAllRegistrer(uc)
        diff = get_diff_regs(pre_regs, now_regs)
        print(diff)
    code = mu.mem_read(address, size)
    insn = disassemble(code, address)
    pre_regs = getAllRegistrer(uc)
    line = ""
    line += '%08X  ' % address
    line += '%s  ' % code.hex()
    line += "%s %s  ;" % (insn.mnemonic, insn.op_str)

    print(line, end=" ")


pre_regs = None
regs = {"x0": unicorn.arm64_const.UC_ARM64_REG_X0, "x1": unicorn.arm64_const.UC_ARM64_REG_X1,
        "x2": unicorn.arm64_const.UC_ARM64_REG_X2, "x3": unicorn.arm64_const.UC_ARM64_REG_X3,
        "x4": unicorn.arm64_const.UC_ARM64_REG_X4, "x5": unicorn.arm64_const.UC_ARM64_REG_X5,
        "x6": unicorn.arm64_const.UC_ARM64_REG_X6, "x7": unicorn.arm64_const.UC_ARM64_REG_X7,
        "x8": unicorn.arm64_const.UC_ARM64_REG_X8, "x9": unicorn.arm64_const.UC_ARM64_REG_X9,
        "x10": unicorn.arm64_const.UC_ARM64_REG_X10, "x11": unicorn.arm64_const.UC_ARM64_REG_X11,
        "x12": unicorn.arm64_const.UC_ARM64_REG_X12, "x13": unicorn.arm64_const.UC_ARM64_REG_X13,
        "x14": unicorn.arm64_const.UC_ARM64_REG_X14, "x15": unicorn.arm64_const.UC_ARM64_REG_X15,
        "x16": unicorn.arm64_const.UC_ARM64_REG_X16, "x17": unicorn.arm64_const.UC_ARM64_REG_X17,
        "x18": unicorn.arm64_const.UC_ARM64_REG_X18, "x19": unicorn.arm64_const.UC_ARM64_REG_X19,
        "x20": unicorn.arm64_const.UC_ARM64_REG_X20, "x21": unicorn.arm64_const.UC_ARM64_REG_X21,
        "x22": unicorn.arm64_const.UC_ARM64_REG_X22, "x23": unicorn.arm64_const.UC_ARM64_REG_X23,
        "x24": unicorn.arm64_const.UC_ARM64_REG_X24, "x25": unicorn.arm64_const.UC_ARM64_REG_X25,
        "x26": unicorn.arm64_const.UC_ARM64_REG_X26, "x27": unicorn.arm64_const.UC_ARM64_REG_X27,
        "x28": unicorn.arm64_const.UC_ARM64_REG_X28, "fp": unicorn.arm64_const.UC_ARM64_REG_FP,
        "lr": unicorn.arm64_const.UC_ARM64_REG_LR, "sp": unicorn.arm64_const.UC_ARM64_REG_SP}

ARM64_CODE = bytes.fromhex("200080D2410080D20000018B")

# memory address where emulation starts
ADDRESS = 0x10000

# Initialize emulator in ARM mode
mu = Uc(UC_ARCH_ARM64, UC_MODE_ARM)
cs = Cs(CS_ARCH_ARM64, CS_MODE_ARM)
cs.detail = True
# map 2MB memory for this emulation
# align 1024
mu.mem_map(ADDRESS, 2 * 1024 * 1024)
mu.mem_write(ADDRESS, ARM64_CODE)

# tracing one instruction with customized callback
mu.hook_add(UC_HOOK_CODE, hook_code, begin=ADDRESS, end=ADDRESS + len(ARM64_CODE))

mu.emu_start(ADDRESS, ADDRESS + len(ARM64_CODE))
```

运行效果如下

```java
00010000  200080d2  mov x0, #1  ; x0 = 0000000000000001	
00010004  410080d2  mov x1, #2  ; x1 = 0000000000000002	
00010008  0000018b  add x0, x0, x1  ; 
```

看起来效果和方案一差不多，但它们的出发点和原理确实不同，在很多指令的展示也有细微差异。

基于 Stalker 的 [sktrace](https://github.com/bmax121/sktrace) 就是基于值比较做的展示，在它的代码里可以看到下面这个函数，它和我们 demo 里的get_diff_regs 作用类似。

```java
def cal_regs_change(self, pre_ctx, ctx):
    print_every_change = True
    eveny_change_str = None
    # ignore pc
    if not pre_ctx:
        for i in range(31):
            self.changed_regs_dict["x"+str(i)] = [ctx.general_regs[i]]
    else:
        if pre_ctx.sp != ctx.sp:
            if "sp" not in self.changed_regs_dict:
                self.changed_regs_dict["sp"] = []
                self.changed_regs_dict["sp"].append(ctx.sp)
                if print_every_change:
                    eveny_change_str = ("{}\t; sp={}->{}".format(self.simple_str(), pre_ctx.sp, ctx.sp))
        for i in range(31):
            if pre_ctx.general_regs[i] != ctx.general_regs[i]:
                if "x"+str(i) not in self.changed_regs_dict:
                    self.changed_regs_dict["x"+str(i)] = []
                self.changed_regs_dict["x"+str(i)].append(ctx.general_regs[i])
                if print_every_change:
                    if eveny_change_str:
                        eveny_change_str += (", x{}={}->{}".format(str(i), pre_ctx.general_regs[i], ctx.general_regs[i]))
                    else:
                        eveny_change_str = ("{}\t; x{}={}->{}".format(self.simple_str(), str(i), pre_ctx.general_regs[i], ctx.general_regs[i]))

    if print_every_change and eveny_change_str:
        print(eveny_change_str)
    pass
```

[frida-trace](https://github.com/IIIImmmyyy/frida-trace) 是基于 stalker 的另一个汇编展示项目，它参考了 sktrace，总体上也是方案二，源码中的 isRegsChange 函数功能类似。

前文提到的，基于 Unidbg 的 [Kingtrace](https://github.com/zhkl0228/unidbg/pull/214) 也是这个思路，其中的 printTrace 函数有注释，写的很清楚

```java
private void printTrace(Backend backend,Capstone.CsInsn[] insns , long address) {
    for (Capstone.CsInsn ins : insns) {
        //查询上否有上一条缓存的指令，有的话，则查询上次的改动寄存器的数值。然后再打印
        if (GlobalData.has_pre && !GlobalData.pre_regname.equals("")){
            Integer regindex = reg_names.get(GlobalData.pre_regname.toUpperCase());
            Number regvalue = backend.reg_read(regindex);
            GlobalData.pre_codestr+=String.format("\t//%s=0x%x" , GlobalData.pre_regname,regvalue);
            printMsg(GlobalData.pre_codestr);
            // 省略
```

#### 4.2.3 Unidbg 的方案

Unidbg 认为，重要信息是当前指令被读取的寄存器，以及被修改的寄存器。应该打印被读取的寄存器在指令执行前的值，以及被修改的寄存器在指令执行前的值。

以`sub x8, x29, #0x48`为例，在方案一里会打印 X8，因为它是目标寄存器；在方案二里会打印 X8，因为指令执行后它发生了改变。在 Unidbg 中 trace 如下。

```java
"add x12, x29, #0x20" x29=0xbfffeea0 => x12=0xbfffeec0
```

它会在指令执行前的时机打印 X29，指令执行后的时机打印 X12。这么做看起来相当清晰，具体怎么实现？

现代反汇编器都会提供充分的指令和寄存器分析，比如当前指令属于哪一类（读写、比较、加减、乘除等等）、是否影响标志位、会读取哪些寄存器、会对哪些寄存器产生修改等等。Capstone 是反汇编器的集大成者，自然也包括这些分析。下面看一下代码实现，`regs_access`会返回两个列表，第一个 list 是当前指令所读取的寄存器，第二个 list 是当前指令所修改的寄存器。

```java
regs_read, regs_written = insn.regs_access()
```

完整代码如下

```java
from unicorn import *
from capstone import *


def disassemble(code, addr):
    return next(cs.disasm(bytes(code), addr, 1))


def getAllRegistrer(uc):
    values = {}
    for i in regs:
        values[i] = uc.reg_read(regs[i])
    return values


def get_diff_regs(preRegs, nowRegs):
    diffText = ""
    for i in preRegs:
        pre = preRegs[i]
        now = nowRegs[i]
        if pre != now:
            diffText += "%s = %016X\t" %(i, now)
    return diffText


# callback for tracing instructions
def hook_code(uc, address, size, user_data):
    global regs_leak
    if regs_leak is not None:
        add = "=>"
        for reg in regs_leak:
            value = uc.reg_read(regs[reg])
            add += " %s = %016X\t" % (reg, value)
        print(add)
    code = mu.mem_read(address, size)
    insn = disassemble(code, address)
    regs_read, regs_written = insn.regs_access()
    line = ""
    line += '%08X  ' % address
    line += '%s  ' % code.hex()
    line += "%s %s; " % (insn.mnemonic, insn.op_str)
    if len(regs_read) > 0:
        for reg in regs_read:
            regName = insn.reg_name(reg)
            value = uc.reg_read(regs[regName])
            line += "%s = %016X\t" % (regName, value)
    if len(regs_written) > 0:
        # 用 map 函数简化映射
        regs_leak = list(map(insn.reg_name, regs_written))
    print(line, end="")


regs_leak = None
regs = {"x0": unicorn.arm64_const.UC_ARM64_REG_X0, "x1": unicorn.arm64_const.UC_ARM64_REG_X1,
        "x2": unicorn.arm64_const.UC_ARM64_REG_X2, "x3": unicorn.arm64_const.UC_ARM64_REG_X3,
        "x4": unicorn.arm64_const.UC_ARM64_REG_X4, "x5": unicorn.arm64_const.UC_ARM64_REG_X5,
        "x6": unicorn.arm64_const.UC_ARM64_REG_X6, "x7": unicorn.arm64_const.UC_ARM64_REG_X7,
        "x8": unicorn.arm64_const.UC_ARM64_REG_X8, "x9": unicorn.arm64_const.UC_ARM64_REG_X9,
        "x10": unicorn.arm64_const.UC_ARM64_REG_X10, "x11": unicorn.arm64_const.UC_ARM64_REG_X11,
        "x12": unicorn.arm64_const.UC_ARM64_REG_X12, "x13": unicorn.arm64_const.UC_ARM64_REG_X13,
        "x14": unicorn.arm64_const.UC_ARM64_REG_X14, "x15": unicorn.arm64_const.UC_ARM64_REG_X15,
        "x16": unicorn.arm64_const.UC_ARM64_REG_X16, "x17": unicorn.arm64_const.UC_ARM64_REG_X17,
        "x18": unicorn.arm64_const.UC_ARM64_REG_X18, "x19": unicorn.arm64_const.UC_ARM64_REG_X19,
        "x20": unicorn.arm64_const.UC_ARM64_REG_X20, "x21": unicorn.arm64_const.UC_ARM64_REG_X21,
        "x22": unicorn.arm64_const.UC_ARM64_REG_X22, "x23": unicorn.arm64_const.UC_ARM64_REG_X23,
        "x24": unicorn.arm64_const.UC_ARM64_REG_X24, "x25": unicorn.arm64_const.UC_ARM64_REG_X25,
        "x26": unicorn.arm64_const.UC_ARM64_REG_X26, "x27": unicorn.arm64_const.UC_ARM64_REG_X27,
        "x28": unicorn.arm64_const.UC_ARM64_REG_X28, "fp": unicorn.arm64_const.UC_ARM64_REG_FP,
        "lr": unicorn.arm64_const.UC_ARM64_REG_LR, "sp": unicorn.arm64_const.UC_ARM64_REG_SP}

ARM64_CODE = bytes.fromhex("200080D2410080D20000018B")

# memory address where emulation starts
ADDRESS = 0x10000

# Initialize emulator in ARM mode
mu = Uc(UC_ARCH_ARM64, UC_MODE_ARM)
cs = Cs(CS_ARCH_ARM64, CS_MODE_ARM)
cs.detail = True
# map 2MB memory for this emulation
# align 1024
mu.mem_map(ADDRESS, 2 * 1024 * 1024)
mu.mem_write(ADDRESS, ARM64_CODE)

# tracing one instruction with customized callback
mu.hook_add(UC_HOOK_CODE, hook_code, begin=ADDRESS, end=ADDRESS + len(ARM64_CODE))

mu.emu_start(ADDRESS, ADDRESS + len(ARM64_CODE))
```

运行效果如下，确实比较好。

```java
00010000  200080d2  mov x0, #1; x0 = 0000000000000000	=> x0 = 0000000000000001	
00010004  410080d2  mov x1, #2; x1 = 0000000000000000	=> x1 = 0000000000000002	
00010008  0000018b  add x0, x0, x1; x0 = 0000000000000001	x1 = 0000000000000002	
```

这个方案不应该说是 Unidbg 想这么做，而是因为有了 Capstone 等类似的现代反汇编器，支持我们这么去做。除了 Unidbg ，ExAndroidNative 的汇编 trace 也是基于此做的信息打印。

```java
# print code and its moudle in a line
DUMP_REG_READ=1
DUMP_REG_WRITE=2
def dump_code(emu, address, size, fd, dump_reg_type=DUMP_REG_READ):  
    mu = emu.mu
    if (emu.get_arch() == emu_const.ARCH_ARM32):
        #判断是否arm，用不同的decoder
        cpsr = mu.reg_read(UC_ARM_REG_CPSR)
        if (cpsr & (1<<5)):
            md = g_md_thumb
        else:
            md = g_md_arm
        #
    else:
        #arm64
        md = g_md_arm64
    #
    instruction = mu.mem_read(address, size)
    codes = md.disasm(instruction, address)
    m = 0
    for i in codes:
        addr = i.address

        name = "unknown"
        module = None
        base = 0
        funName = None
        module = get_module_by_addr(emu, addr)
        if (module != None):
            name = os.path.basename(module.filename)
            base = module.base
            funName = module.is_symbol_addr(addr)
        #

        instruction_str = ''.join('{:02X} '.format(x) for x in i.bytes)
        tid = ""
        if (emu.get_muti_task_support()):
            sch = emu.get_schduler()
            tid = "%d:"%sch.get_current_tid()
        line = "%s(%20s[0x%08X])[%-12s]0x%08X:\t%s\t%s"%(tid, name, base, instruction_str, addr-base, i.mnemonic.upper(), i.op_str.upper())
        if (funName != None):
            line = line + " ; %s"%funName
        #
        regs = i.regs_access()
        if (DUMP_REG_READ == dump_reg_type):
            regs_dump = regs[0]
        elif(DUMP_REG_WRITE == dump_reg_type):
            regs_dump = regs[1]
        #
        regs_io = io.StringIO()
        for rid in regs_dump:
            reg_str = "%s=0x%08X "%(i.reg_name(rid).upper(), mu.reg_read(rid))
            regs_io.write(reg_str)
        #
        regs = regs_io.getvalue()
        if (regs != ""):
            line = "%s\t;(%s)"%(line, regs)
        #
        fd.write(line+"\n")
```

核心代码是调用 Capstone 的`regs_access`API，没错，就是我们刚在 JAVA 里使用过的那个 API。

```java
regs = i.regs_access()
if (DUMP_REG_READ == dump_reg_type):
    regs_dump = regs[0]
elif(DUMP_REG_WRITE == dump_reg_type):
    regs_dump = regs[1]
```

ExAndroidNativeEmu 处理的远不如 Unidbg 精细。

Unidbg 展示的是“被读取的寄存器在指令执行前的状态” 以及“被修改的寄存器在指令执行后的状态”，而 ExAndroidNative 的展示效果则是在“被读取的寄存器在指令执行前的状态”和“被修改的寄存器在**指令执行前**的状态”里二选一，这种展示切换通过`dump_code`函数的`dump_reg_type`参数控制，1 是前者，2 是后者。

```java
DUMP_REG_READ=1
DUMP_REG_WRITE=2
def hook_code(mu, address, size, user_data):
    try:
        emu = user_data
        androidemu.utils.debug_utils.dump_code(emu, address, size, g_cfd, dump_reg_type=2)
    except Exception as e:
        logger.exception("exception in hook_code")
        sys.exit(-1)
```

我们举两个例子

假设 X0 寄存器在指令执行前的值是 0x2D，X1 寄存器在指令执行前的值是 0x3A。

那么在 ExAndroidNativeEmu 里，`DUMP_REG_READ`模式会得到如下打印。

```java
MOV	X0, X1	;(X1=0x0000003A)
```

`DUMP_REG_WRITE`模式会得到如下打印，这个处理很明显不太妙，打印的是被修改寄存器，但值却是修改前的值。

```java
MOV	X0, X1	;(X0=0x0000002D)
```

在 Unidbg 里，则会是下面的展示效果

```java
"mov x0, x1" x1=0x0000003a => x0=0x0000003a
```

在观感上 Unidbg 的展示效果更好。

### 4.3 关于展示的总结

展示的好坏对分析 trace 文件其实有不小影响，在第四部分的第一节里，我们讨论了如何获取基本的信息，第二节里，我们讨论了三类常见的展示优化。在实际方案里，往往会做更多的优化，给予使用者更好的体验，包括但不限于——检测潜在的字符串、打印被修改的内存、体现基本块的切换、特殊指令的处理等等。读者也可以发挥自己的奇思妙想，将指令追踪的展示处理的更漂亮。



# unidbg的trace进阶使用
## 一、引言

本节讨论如何在 Unidbg 中实现函数粒度的 trace。

## 二、介绍

function trace 对搞清执行流、算法分析、以及处理混淆严重的样本有很大帮助。本节介绍 Unidbg 内置函数追踪方案`traceFunctionCall`的基本使用，它是 Unidbg 在 0.9.6 版本上添加的新功能，目前多次修改，已趋于稳定。

首先以上一篇样本为例，进行展示，`traceFunctionCall`在设计上为 Unidbg Debugger 的一部分，在 Unicorn/Unicorn2 Backend 上可以使用。

```java
package com.aikucun;

import com.github.unidbg.AndroidEmulator;
import com.github.unidbg.Emulator;
import com.github.unidbg.Module;
import com.github.unidbg.arm.backend.DynarmicFactory;
import com.github.unidbg.arm.backend.Unicorn2Factory;
import com.github.unidbg.debugger.BreakPointCallback;
import com.github.unidbg.debugger.Debugger;
import com.github.unidbg.debugger.FunctionCallListener;
import com.github.unidbg.linux.android.AndroidEmulatorBuilder;
import com.github.unidbg.linux.android.AndroidResolver;
import com.github.unidbg.linux.android.dvm.*;
import com.github.unidbg.memory.Memory;
import com.github.unidbg.pointer.UnidbgPointer;
import unicorn.Arm64Const;

import java.io.File;
import java.io.FileNotFoundException;
import java.io.FileOutputStream;
import java.io.PrintStream;

public class MXSecurity extends AbstractJni {
    private final AndroidEmulator emulator;
    private final DvmClass security;
    private final VM vm;
    private final Module module;

    public MXSecurity() {
        emulator = AndroidEmulatorBuilder.for64Bit()
                .addBackendFactory(new Unicorn2Factory(false))
                .setProcessName("com.aikucun.akapp")
                .build();
//        emulator.getBackend().registerEmuCountHook(10000); // 设置执行多少条指令切换一次线程
//        emulator.getSyscallHandler().setVerbose(true);
//        emulator.getSyscallHandler().setEnableThreadDispatcher(true);
        Memory memory = emulator.getMemory();
        memory.setLibraryResolver(new AndroidResolver(23));
        vm = emulator.createDalvikVM(new File("unidbg-android/src/test/resources/aikucun/com.aikucun.akapp_6.0.6_60006.apk"));
        vm.setVerbose(true);
        vm.setJni(this);
        DalvikModule dm = vm.loadLibrary("mx", true);
        module = dm.getModule();
        security = vm.resolveClass("com/mengxiang/arch/security/MXSecurity");
        dm.callJNI_OnLoad(emulator);

        // function trace
        Debugger debugger = emulator.attach();
        debugger.traceFunctionCall(module, new FunctionCallListener() {
            @Override
            public void onCall(Emulator<?> emulator, long callerAddress, long functionAddress) {
            }
            @Override
            public void postCall(Emulator<?> emulator, long callerAddress, long functionAddress, Number[] args) {
                System.out.println("onCallFinish caller=" + UnidbgPointer.pointer(emulator, callerAddress) + ", function=" + UnidbgPointer.pointer(emulator, functionAddress));
            }
        });

    }

    public void callinit(){
        DvmObject<?> context = vm.resolveClass("com.aikucun.akapp.AppContext").newObject(null);
        int ret = security.callStaticJniMethodInt(emulator, "init", context, false);
        System.out.println("call init:"+ret);
    }

    public void callSignV1(){
        String arg1 = "https://zuul.aikucun.com/aggregation-center-facade/api/app/search/product/image/switch?" +
                "appid=38741001&did=6d2fe7c7702721c6b797cf22ec8f5f58&noncestr=4b373c&timestamp=1662394452&zuul=1";
        String arg2 = "4b373c";
        String arg3 = "1662394452";
        String result = security.callStaticJniMethodObject(emulator, "signV1", arg1, arg2, arg3).getValue().toString();
        System.out.println("call SignV1 result:"+result);
    }

    public static void main(String[] args) {
        MXSecurity mxSecurity = new MXSecurity();
        mxSecurity.callinit();
        mxSecurity.callSignV1();
    }

    @Override
    public DvmObject<?> callObjectMethodV(BaseVM vm, DvmObject<?> dvmObject, String signature, VaList vaList) {
        switch (signature){
            case "com/aikucun/akapp/AppContext->getPackageManager()Landroid/content/pm/PackageManager;":{
                return vm.resolveClass("android/content/pm/PackageManager").newObject(null);
            }
            case "com/aikucun/akapp/AppContext->getPackageName()Ljava/lang/String;":{
                return new StringObject(vm, vm.getPackageName());
            }
        }
        return super.callObjectMethodV(vm, dvmObject, signature, vaList);
    }
}
```

代码运行如下，展示了调用方地址以及调用的函数。

```java
onCallFinish caller=RX@0x4000d954[libmx.so]0xd954, function=RX@0x4000dab4[libmx.so]0xdab4
onCallFinish caller=RX@0x4000db30[libmx.so]0xdb30, function=RX@0x4000d620[libmx.so]0xd620
onCallFinish caller=RX@0x4000d954[libmx.so]0xd954, function=RX@0x4000dab4[libmx.so]0xdab4
onCallFinish caller=RX@0x4000db30[libmx.so]0xdb30, function=RX@0x4000d620[libmx.so]0xd620
onCallFinish caller=RX@0x4000d954[libmx.so]0xd954, function=RX@0x4000dab4[libmx.so]0xdab4
onCallFinish caller=RX@0x4000db30[libmx.so]0xdb30, function=RX@0x4000d620[libmx.so]0xd620
onCallFinish caller=RX@0x4000d954[libmx.so]0xd954, function=RX@0x4000dab4[libmx.so]0xdab4
onCallFinish caller=RX@0x4000db30[libmx.so]0xdb30, function=RX@0x4000d620[libmx.so]0xd620
onCallFinish caller=RX@0x4000d954[libmx.so]0xd954, function=RX@0x4000dab4[libmx.so]0xdab4
onCallFinish caller=RX@0x4000db30[libmx.so]0xdb30, function=RX@0x4000d620[libmx.so]0xd620
onCallFinish caller=RX@0x4000d954[libmx.so]0xd954, function=RX@0x4000dab4[libmx.so]0xdab4
onCallFinish caller=RX@0x4000db30[libmx.so]0xdb30, function=RX@0x4000d620[libmx.so]0xd620
onCallFinish caller=RX@0x4000d954[libmx.so]0xd954, function=RX@0x4000dab4[libmx.so]0xdab4
onCallFinish caller=RX@0x4000e53c[libmx.so]0xe53c, function=RX@0x4000d330[libmx.so]0xd330
```

以首行为例，它对应于下图这行代码

```java
.text:000000000000D954                 BL              sub_DAB4
```

应该说调用完全符合预期，接下来我们仔细看看`traceFunctionCall`提供了什么，函数原型如下。

```java
/**
 * 目前仅支持 Unicorn/Unicorn2 Backend
 * 追踪module的函数调用，module为null时追踪所有模块
 */
void traceFunctionCall(Module module, FunctionCallListener listener);
```

FunctionCallListener 提供 onCall 和 postCall 两个时机，它对应于 Frida 里的 onEnter 和 onLeave，即函数调用前以及函数执行后。

```java
debugger.traceFunctionCall(module, new FunctionCallListener() {
    @Override
    public void onCall(Emulator<?> emulator, long callerAddress, long functionAddress) {
    }
    @Override
    public void postCall(Emulator<?> emulator, long callerAddress, long functionAddress, Number[] args) {
        
    }
});
```

两个时机都可以访问 callerAddress 和 functionAddress，postCall 时机还可以访问 args，它在语义上和 frida args 类似，args 数组长度不代表函数的入参数目。

在 ARM32 上，Unidbg args 默认 4 个参数

```java
Number[] args = new Number[4];
for (int i = 0; i < args.length; i++) {
    args[i] = context.getIntArg(i);
}
```

在 ARM64上，Unidbg args 默认八个参数

```java
Number[] args = new Number[8];
for (int i = 0; i < args.length; i++) {
    args[i] = context.getLongArg(i);
}
```

因此可以根据地址做判断，对感兴趣的函数做进一步的、参数的展示。

接下来通过调用栈深度，设置函数展示的层级，形成函数调用树。首先需要在 Unidbg 的栈回溯实现中添加一个方法，让它返回栈深度。

位置：src/main/java/com/github/unidbg/unwind/Unwinder.java

```java
package com.github.unidbg.unwind;

import com.github.unidbg.Emulator;
import com.github.unidbg.Module;
import com.github.unidbg.Symbol;
import com.github.unidbg.arm.AbstractARMDebugger;
import com.github.unidbg.memory.MemRegion;
import com.github.unidbg.memory.Memory;
import com.github.unidbg.memory.SvcMemory;
import com.github.unidbg.pointer.UnidbgPointer;
import com.github.zhkl0228.demumble.DemanglerFactory;
import com.github.zhkl0228.demumble.GccDemangler;

public abstract class Unwinder {

    public static final int SYMBOL_SIZE = 0x1000;

    protected final Emulator<?> emulator;

    protected Unwinder(Emulator<?> emulator) {
        this.emulator = emulator;
    }

    public abstract Frame createFrame(UnidbgPointer ip, UnidbgPointer fp);

    protected abstract Frame unw_step(Emulator<?> emulator, Frame frame);

    protected abstract String getBaseFormat();

    public final void unwind() {
        Memory memory = emulator.getMemory();
        String maxLengthSoName = memory.getMaxLengthLibraryName();
        boolean hasTrace = false;

        Frame frame = null;
        while((frame = unw_step(emulator, frame)) != null) {
            if (frame.isFinish()) {
                if (!hasTrace) {
                    System.out.println("Decode backtrace finish");
                }
                return;
            }

            hasTrace = true;
            printFrameElement(maxLengthSoName, memory, frame.ip);
        }

        if (!hasTrace) {
            System.err.println("Decode backtrace failed.");
        }
    }

    private void printFrameElement(String maxLengthSoName, Memory memory, UnidbgPointer ip) {
        final int maxLength = maxLengthSoName.length();
        SvcMemory svcMemory = emulator.getSvcMemory();
        MemRegion region = svcMemory.findRegion(ip.peer);
        Module module = region != null ? null : AbstractARMDebugger.findModuleByAddress(emulator, ip.peer);
        StringBuilder sb = new StringBuilder();
        String format = getBaseFormat();
        if (module != null) {
            sb.append(String.format(format, module.base)).append(String.format(format, ip.peer));
            sb.append(String.format("[%" + maxLength + "s]", module.name));
            sb.append(String.format("[0x%0" + Long.toHexString(memory.getMaxSizeOfLibrary()).length() + "x]", ip.peer - module.base));

            Symbol symbol = module.findClosestSymbolByAddress(ip.peer, false);
            if (symbol != null && ip.peer - symbol.getAddress() <= SYMBOL_SIZE) {
                GccDemangler demangler = DemanglerFactory.createDemangler();
                sb.append(" ").append(demangler.demangle(symbol.getName())).append(" + 0x").append(Long.toHexString(ip.peer - (symbol.getAddress() & ~1)));
            }
        } else {
            sb.append(String.format(format, 0)).append(String.format(format, ip.peer));
            if (region == null) {
                sb.append(String.format("[%" + maxLength + "s]", "0x" + Long.toHexString(ip.peer)));
            } else {
                sb.append(String.format("[%" + maxLength + "s]", region.getName().substring(0, Math.min(maxLength, region.getName().length()))));
            }
            if (ip.peer >= svcMemory.getBase()) {
                sb.append(String.format("[0x%0" + Long.toHexString(memory.getMaxSizeOfLibrary()).length() + "x]", ip.peer - svcMemory.getBase()));
            }
        }
        System.out.println(sb);
    }

    // add
    public final int depth(){
        int count = 0;
        Frame frame = null;
        while((frame = unw_step(emulator, frame)) != null) {
            if(frame.isFinish()){
                return count;
            }
            count++;
        }
        return count;
    }

}
```

代码处理如下

```java
Debugger debugger = emulator.attach();
PrintStream traceStream = null;
try {
    // 保存文件
    String traceFile = "unidbg-android/src/test/resources/aikucun/trace/traceFunctions.txt";
    traceStream = new PrintStream(new FileOutputStream(traceFile), true);
} catch (FileNotFoundException e) {
    e.printStackTrace();
}
final PrintStream finalTraceStream = traceStream;
assert finalTraceStream != null;
debugger.traceFunctionCall(null, new FunctionCallListener() {
    @Override
    public void onCall(Emulator<?> emulator, long callerAddress, long functionAddress) {
        int level = emulator.getUnwinder().depth();
        for(int i = 0 ; i < level ; i++){
            finalTraceStream.print("    |    ");
        }
        finalTraceStream.println("  "+"sub_"+Integer.toHexString((int) (functionAddress-module.base))+"  ");
    }
    @Override
    public void postCall(Emulator<?> emulator, long callerAddress, long functionAddress, Number[] args) {

    }
});
```

因为担心正常的日志输出干扰函数调用树的展示，所以将输出重定向到文件，效果如下。

![](https://cdn.nlark.com/yuque/0/2022/png/707476/1663890975313-ce31cb4c-3425-4ee8-8d6d-944830dcf898.png)

在某些场景里，调用树有很好的作用，但它并不像 frida-trace 生成的调用树那样好用，这可能部分归因于 Unidbg 的栈回溯不够好。

## 三、基本使用

在算法分析过程中，function trace 就是一张索引图，引导我们对函数逐个击破，实现对完整流程的理解。在处理混淆过但模块化做的非常好的样本时，function trace 会非常好用。“模块化做的非常好”指的是，Native 中各种算法和逻辑，由一个个小函数组织和构成。Virjar大佬曾一阵见血的指出

现在ollvm能被手撕的最大原因，就是因为模块化太好，留下了大量可以被hook的sub function。要是整个算法就一个function body，没有其他sub route，我看他怎么trace。

之前的 [情景三](https://t.zsxq.com/06MBmqFmy) 一文里，我仔细阐述过如何利用 function trace 识别加密算法的各个组成部分，读者可以移步观看。

## 四、进阶使用

这里主要谈论两个点，1 是 function trace 如何实现 2 是如何基于 function trace 去设计更有用的东西。

### 4.1 函数追踪的实现

Function trace 的实现分为识别函数和函数 Hook 两部分，第二部分不必多说，主要讨论第一部分，即如何获取模块内包括 sub_xxx 在内的所有导出以及非导出函数的地址、函数名等信息。从一个较高的角度上，函数识别与检测属于反汇编技术的子话题，它有四项基本技术。

1 是基于模式的匹配

IDA、Ghidra、Binary Ninja 等反汇编器都以这项技术为基础，对函数做识别。所谓模式指的是在大多数情况下，函数以及其调用约定所呈现的固定汇编特征——在函数起始处开栈和入栈，在函数结尾处清栈和消栈。我们将它称为函数序言和函数尾声。

反汇编器会收集和维护每种架构、调用约定、大小端序所对应的模式，整合成模式库。在代码中对这些模式做二进制搜索，postpatterns 是对应的函数起始地址，prepatterns 是函数尾声。

比如 [ghidra](https://github.com/NationalSecurityAgency/ghidra/blob/master/Ghidra/Processors/AARCH64/data/patterns/AARCH64_LE_patterns.xml) 对 ARM64 小端序的模式库代码如下

```java
<patternlist>  
  <patternpairs totalbits="32" postbits="16">  <!-- AARCH64 -->
    <prepatterns>
      <data>0xc0 0x03 0x5f 0xd6                  </data> <!-- ret -->
      <data>0xff 0x0f 0x5f 0xd6                  </data> <!-- retab -->
      <data> ........ ........ ........ 000101.. </data> <!-- b <xxx>  shared jump call -->
      <data> 0x20 0x00 0x20 0xd4                 </data> <!-- brk #1 -->
    </prepatterns>
    
    <postpatterns>
      <data> 0xfd 0x7b 0xbf 0xa9 </data> <!--  stp x29, x30, [sp, #-0x10]!  -->
      <data> 0xfe .0001111 0x1. 0xf8 </data> <!--  stp x30, [sp, #-0x..0]!  -->
      <data> 111..... .1....11 10...... 0xa9 </data> <!-- stp x, x, [sp, -0x.0]! -->
      <data> 11101..1 001..011 1011.... 0x6d </data> <!-- stp d, d, [sp, -0x.0]! -->
      <data> 0xff ..000011 000..... 0xd1 </data> <!-- sub sp, sp, #... -->
      <data> 0x7f 0x23 0x03 0xd5 </data> <!--  pacibsp  -->
      <codeboundary/>
      <possiblefuncstart/>
    </postpatterns>
  </patternpairs>
  
  <pattern> <!-- possible function start -->
    <data> 111..... .1....11 10...... 0xa9 </data> <!-- stp x, x, [sp, -0x.0]! -->
    <possiblefuncstart after="defined" isvalid="true"/> <!-- must be something defined right before this -->
  </pattern>
  
  <pattern> <!-- possible function start -->
    <data> 0x........    111..... .1....11 10...... 0xa9 </data> <!-- stp x, x, [sp, -0x.0]! -->
    <possiblefuncstart after="defined" isvalid="true" /> <!-- must be something defined right before this -->
  </pattern>
  
  <pattern> <!-- possible function start -->
    <data> 0xfe .0001111 0x1. 0xf8 </data> <!--  stp x30, [sp, #-0x..0]!  -->
    <possiblefuncstart after="defined" isvalid="true"/> <!-- must be something defined right before this -->
  </pattern>
  
  <pattern> <!-- possible function start -->
    <data> 0x........ 0xfe .0001111 0x1. 0xf8 </data> <!--  stp x30, [sp, #-0x..0]!  -->
    <possiblefuncstart after="defined" isvalid="true"/>  <!-- must be something defined right before this -->
  </pattern>

  <pattern> <!-- solid function start -->
      <data> 0xfd 0x7b 0xbf 0xa9 0xfd 0x03 0x00 0x91  </data> <!-- stp x29, x30, [sp, #-0x10]! and  mov x29,sp -->
      <codeboundary />  <!-- definitely code -->
      <possiblefuncstart /> <!-- must be something defined right before this -->
  </pattern>
  
  <pattern> <!-- solid function start -->
      <data> 0x7f 0x23 0x03 0xd5 0xff ..000011 00000... 0xd1 </data> <!-- pacibsp   sub sp, sp, #... -->
      <codeboundary />  <!-- definitely code -->
      <possiblefuncstart /> <!-- must be something defined right before this -->
  </pattern>
  
  <pattern> <!-- AARCH64 Thunk -->
      <data> ...10000 ........ ........ 1..10000 
             00010001 ........ 01...... 0xf9 
             0x10 ......10 00...... 0x91 
             0x20 0x02 0x1f 0xd6   </data>
      <!-- adrp x16, #...
           ldr  x17, xxxx
           add  x16, x16
           br   x17
       -->
      <funcstart validcode="function" thunk="true"/>
  </pattern>
  
</patternlist>
```

并非所有函数都符合模式特征，所以还有其他方案来做函数识别这件事。

2 是 eh_frame，它是二进制文件中用于异常处理的节区，保存着函数信息，但它和编译器相关联，此处不做讨论。

3 是控制流分析，以 ARM64 为例，我们一般认为 B 指令用于跳转，BL、BLR 这些指令用于子函数调用。因此，可以解析和判定 BL、BLR 所执行的地址就是函数。在静态和动态反汇编中都可以使用这一思路，Unidbg 的 traceFunctionCall 采用的就是这个方案。

```java
@Override
protected void onInstruction(Instruction instruction) {
    String mnemonic = instruction.getMnemonic();
    RegisterContext context = emulator.getContext();
    if ("bl".equals(mnemonic) || "blr".equals(mnemonic)) {
        OpInfo operands = (OpInfo) instruction.getOperands();
        Operand operand = operands.getOperands()[0];
        final long functionAddress;
        switch (operand.getType()) {
            case capstone.Arm64_const.ARM64_OP_IMM:
                functionAddress = operand.getValue().getImm();
                break;
            case capstone.Arm64_const.ARM64_OP_REG:
                functionAddress = context.getLongByReg(instruction.mapToUnicornReg(operand.getValue().getReg()));
                break;
            default:
                throw new UnsupportedOperationException("type=" + operand.getType());
        }
        Number[] args = new Number[8];
        for (int i = 0; i < args.length; i++) {
            args[i] = context.getLongArg(i);
        }
        pushFunction(instruction.getAddress(), functionAddress, instruction.getAddress() + instruction.getSize(), args);
    } else {
        throw new UnsupportedOperationException();
    }
}
```

4 是基于机器学习和深度学习的启发式分析。

这个思路近年来很流行，或者说机器学习和深度学习在所有领域都很流行。函数识别问题可以转化为 NLP 的子问题，对此感兴趣的读者可以看[📎基于机器学习的ARM平台二进制代码函数识别研究.pdf](https://www.yuque.com/attachments/yuque/0/2022/pdf/707476/1663925275873-65b94031-1fbd-4f99-9b07-4bca921019bc.pdf)这篇论文。

需要注意的是，这一思路在本质上隐式的依赖于模式匹配。


## 五、总结

function trace 是一个有用的功能吗？有些人认为，这些功能对于逆向分析来说并不必要，持这种观点的人并不占少数。事实上，除了 traceCode / debugger / traceRead / traceWrite / 污点分析 这些分析技术，其他分析技术都被认为有些鸡肋。不妨这么理解，底层的基础分析技术确实有用，顶部的高级分析技术确实有用，中间的各种分析技术，没那么有用，但学习还需要成本，所以多少显得鸡肋。

![](https://cdn.nlark.com/yuque/0/2022/png/707476/1663896734692-b222cc29-11ed-4cf6-9676-c429856d9835.png)





