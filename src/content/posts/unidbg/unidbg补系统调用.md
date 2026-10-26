---
title: unidbg补系统调用
published: 2026-10-06
description: unidbg补系统调用
image: ./cover.jpg
tags: [逆向]
category: unidbg
draft: false
---

[unidbg补系统调用一](https://www.yuque.com/docs/share/449cfa56-be10-47f4-8d32-58770cc22f81?#%20%E3%80%8A%E8%A1%A5%E7%B3%BB%E7%BB%9F%E8%B0%83%E7%94%A8%EF%BC%88%E4%B8%80%EF%BC%89%E3%80%8B  ) 
[unidbg补系统调用二](https://www.yuque.com/docs/share/cbfb3b38-06a6-4d6d-b4df-ecaec219956f?#%20%E3%80%8A%E8%A1%A5%E7%B3%BB%E7%BB%9F%E8%B0%83%E7%94%A8%EF%BC%88%E4%BA%8C%EF%BC%89%E3%80%8B  ) 
[unidbg补系统调用三](https://www.yuque.com/docs/share/cb5f86f3-fea3-4f0c-a567-39cea30b9307?#%20%E3%80%8A%E8%A1%A5%E7%B3%BB%E7%BB%9F%E8%B0%83%E7%94%A8%EF%BC%88%E4%B8%89%EF%BC%89%E3%80%8B  )
[unidbg补系统调用四](https://www.yuque.com/docs/share/c6b0da92-bbff-4395-adf6-d8afbe5ef123?#%20%E3%80%8A%E8%A1%A5%E7%B3%BB%E7%BB%9F%E8%B0%83%E7%94%A8%EF%BC%88%E5%9B%9B%EF%BC%89%E3%80%8B  )
[unidbg补系统调用五](https://www.yuque.com/docs/share/c4721acc-94cc-4ff5-8dc0-a803444377d8?#%20%E3%80%8A%E8%A1%A5%E7%B3%BB%E7%BB%9F%E8%B0%83%E7%94%A8%EF%BC%88%E4%BA%94%EF%BC%89%E3%80%8B  )




# unidbg补系统调用原因
## 一、引言

这个系列讨论 Unidbg 与系统调用相关的内容，在这部分我们要讨论很多内容。核心有三个话题

1. 如何监控系统调用
2. 系统调用和模拟操作系统这两者间的关联
3. 处理 Unidbg 问题的方法论

这里我们先简单讨论第三点，什么叫处理 Unidbg 问题的方法论？我们前面谈过补环境，包括 JNI 以及文件访问，读者应该很能理解这两类补环境需求，单靠 Unidbg 自身环境，没法完善处理这两部分逻辑。但是除了这两部分补环境，其余什么库函数、系统调用、虚拟模块补环境，其实大家在心里都有抵触。

举两个例子

一是有位读者把 SO 放到自写的 Android demo 里加载并执行其中的方法，运行情况非常良好，但是把这个 SO 放到 Unidbg 里执行，就报了很多错，比如 libart 访问失败等等。

二是有位读者使用 Unidbg 调用某个 Native 函数，正常的补 JNI 和文件访问后，依然报错了，而且看不懂为什么报错。

这两个例子里，使用者感到困惑或者不理解，因为一般情况下，当我们使用工具出现报错时，都会下意识认为是我们自己处理出了问题，又或者这个报错是可控的，比如 JNI 和文件访问就是可控的，Unidbg 提供了处理它们的编程范式。

但是，在使用 Unidbg、qiling 等轻量级模拟器时，你必须接受这样一个事实，就是它们自身环境是不完备的，对很多系统调用的处理是模糊的、片面的。因此当处理样本遇到无法理解的错误时，它有 80% 的可能性是你补环境出了问题，20% 的可能性则是工具在这个逻辑处理上不够完善。即你不仅要质疑自己，而且还要质疑工具。

换句话说，在 JNI 调用和文件访问之外遇到的其他补环境需求，本质上都是突如其来的异常，是我们在替 Unidbg 打补丁或做完善。

## 二、回顾历史

在讨论补系统调用之前，不妨先回顾一下历史，如何编写更安全的代码？这是一个环绕始终的核心问题。

在最开始，Android 代码逻辑都写在 JAVA 层，并且使用 JDK 和 Android FrameWork 提供的各种标准 API 完成各类操作。这个时期，在 Android 逆向领域的经典工具是 [Inspeckage](https://github.com/ac-pm/Inspeckage)，它是一个基于 Xposed 的项目，Hook 了一大批 Android FrameWork 以及 JDK 标准 API，包括加解密、哈希、证书、网络通讯、文件读写、数据库等等。分析 Inspeckage 或同类型工具的 Hook 结果，逆向研究人员可以获得大量关键信息。基于 Frida 框架的 [OkHttpLogger](https://github.com/siyujie/OkHttpLogger-Frida)、[house](https://github.com/nccgroup/house) 、[ssl-pinning-bypass](https://codeshare.frida.re/@pcipolloni/universal-android-ssl-pinning-bypass-with-frida/) 也都是同类型工具。

随后，开发人员将关键代码逻辑通过 NDK 下沉到 Native 层，这么做起初很有效，Android 逆向人员对 NDK 和 Native 很陌生，认为它是一个坚固的黑盒。但很快 PC 攻防的相关知识被引入，IDA、Native Hook、汇编等知识被学习和迁移到 Android Native 逆向领域。

逆向人员发现，在实现各种功能时，会十分依赖标准 C 库以及 ART 虚拟机提供的各类函数，就像 JAVA 层依赖于 JDK 以及 Android 框架层的标准 API 那样。因此，可以同理对这些关键函数做监控。对 ART 的 API Hook 带来了 [frida-unpack](https://github.com/dstmath/frida-unpack)、[frida_ssl_logger](https://github.com/BigFaceCat2017/frida_ssl_logger)、[hook_libart](https://github.com/lasting-yang/frida_hook_libart) 、[jnitrace](https://github.com/chame1eon/jnitrace) 等众多项目，涉及脱壳、函数追踪、流量监控等等。对标准库的追踪，比如 libc.so、linker.so、libz.so 等等，则带来了 [file-system-access-hook](https://codeshare.frida.re/@FrenchYeti/android-file-system-access-hook/) 等项目。

以读文件为例，我们可以使用 Frida Hook 它，示例代码如下。

```java
// native hooks    
Interceptor.attach(
  Module.findExportByName("libc.so", "read"), {
    // fd, buff, len
    onEnter: function(args) {
      if (CONFIG.printLibc === true) {
        var bfr = args[1],
          sz = args[2].toInt32();
        var path = (TraceSysFD["fd-" + args[0].toInt32()] != null) ? TraceSysFD["fd-" + args[0].toInt32()] : "[unknow path]";

        prettyLog("[Libc::read] Read FD (" + path + "," + bfr + "," + sz + ")\n" +
                  rawPrint(path, Memory.readByteArray(bfr, sz)));
      }
    },
    onLeave: function(ret) {

    }
  }
);
```

又比如 strstr

```java
Interceptor.attach(Module.findExportByName("libc.so", "strstr"), {
  onEnter: function(args) {
    console.log("strstr, arg0="+Memory.readUtf8String(args[0]) + ", arg1:"+Memory.readUtf8String(args[1]));
  },
});
```

因为样本依赖这些函数去实现功能，所以监控这些函数就掌握了样本的执行动态，在算法分析、检测点分析等场景里十分好用。

举个例子，如果样本做 Frida 检测时，最终通过 strstr 做文本对比，那么下面的代码很可能就可以绕过。

```java
Interceptor.attach(Module.findExportByName(null, "strstr"),{
  onEnter: function(args){
    if(args[0].readCString().indexOf("frida")!==-1
       ||args[1].readCString().indexOf("frida")!==-1 ||
       args[0].readCString().indexOf("gum-js-loop")!==-1||args[1].readCString().indexOf("gum-js-loop")!==-1||
       args[0].readCString().indexOf("gmain")!==-1
       ||args[1].readCString().indexOf("gmain")!==-1 ||
       args[0].readCString().indexOf("tmp")!==-1
       ||args[1].readCString().indexOf("tmp")!==-1 ||
       args[0].readCString().indexOf("linjector")!==-1
       ||args[1].readCString().indexOf("linjector")!==-1){
      this.frida = Boolean(1);
    }
  },
  onLeave: function(retval){
    if (this.frida) {
      retval.replace(ptr("0x0"));
    }
  }
});
```

开发人员意识到函数必须进一步下沉，使得监控的难度更大。对于 strlen、strcmp 这类函数，完全可以做自实现，而不需要依赖标准库，比如下面的自定义 strstr。

```java
#include<stdio.h>
#include<assert.h>

const char* my_strstr(const char *str1, const char *str2)
{
    assert(str1 != NULL);
    assert(str2 != NULL);
    while (str1 != '\0')
    {
        const char *p = str1;
        const char *q = str2;
        const char *res = NULL;
        if (*p == *q)
        {
            res = p;
            while (*p && *q && *p++ == *q++)
            {
                ;
            }
            if (*q == '\0')
                return res;
        }
        str1++;
    }
    return NULL;
}
```

对于 open/read/write 等库函数，它们依赖于底层的系统调用， 甚至一部分只是系统调用的简单封装，比如库函数`openat`就只是对底层同名系统调用`openat`的简单封装，因此可以直接发起系统调用，而不用库函数。

```java
unsigned int __fastcall _openat(void *a1, void *a2, void *a3, void *a4)
{
  void *v4; // r4
  void *v5; // r5
  void *v6; // r6
  unsigned int result; // r0

  result = linux_eabi_syscall(__NR_openat, a1, a2, a3, a4, v4, v5, v6);
  if ( result > 0xFFFFF000 )
    result = j___set_errno_internal(-result);
  return result;
}
```

用内联汇编可以直接发起系统调用，比如下面这样。

```java
#include <jni.h>
#include <string>
#include <sys/syscall.h>
#include <unistd.h>
#include <fcntl.h>
#include <sys/stat.h>

__attribute__((naked))
long raw_syscall(long _number,...){
    __asm__ __volatile__ ("MOV R12,SP\r\n"
    "STMFD SP!,{R4-R7}\r\n"
    "MOV R7,R0\r\n"
    "MOV R0,R1\r\n"
    "MOV R1,R2\r\n"
    "MOV R2,R3\r\n"
    "LDMIA R12,{R3-R6}\r\n"
    "SVC 0\r\n"
    "LDMFD SP!,{R4-R7}\r\n"
    "mov pc,lr");
};

char* test_syscall(const char* file_path){
    char *result = "";
    long fd = raw_syscall(5,file_path, O_RDONLY | O_CREAT, 400);
    if(fd != -1){
        char buffer[0x100] = {0};
        raw_syscall(3, fd, buffer, 0x100);
        result = buffer;
        raw_syscall(6, fd);
    }
    return result;
}
```

到了这一步，基本就进无可进了，如何监控和拦截系统调用？ 成熟的系统调用拦截技术有不少。

比如 ptrace，strace 这个知名工具底层就是依赖的 ptrace

比如 seccomp，[Frida-Seccomp](https://github.com/Abbbbbi/Frida-Seccomp) 就是基于它的一个好工具。

![](https://cdn.nlark.com/yuque/0/2022/png/707476/1668254734893-da343250-da5d-4169-978f-2f9548a73f77.png)

比如 ebpf，它太火了，使用它监控系统调用是最好的方案。

比如修改 Android 内核，对系统调用表作处理，当然也可以实现监控。

比如土办法，在内存或二进制文件里搜索`SVC`指令，[find-svc](https://bbs.pediy.com/thread-269895.htm)、[frida-svc-interceptor](https://bbs.pediy.com/thread-270144.htm) 等项目就是基于这个思路。

比如基于汇编指令追踪搜索`SVC`指令，[interruptor](https://github.com/FrenchYeti/interruptor) 项目就是如此。

这就是基本的历史和发展情况。

## 三、补系统调用概述

什么叫补系统调用？或者不妨回顾一下，补环境到底是怎么一回事。因为 Unidbg 是基于模拟的方案，既然是模拟，那就必然有两个问题。

1. 模拟不可能考虑到每一个函数和逻辑，即有些函数尚未模拟。
2. 对于已模拟的函数也不可能都自洽，都尽善尽美，即有些函数模拟的不好或“模拟”本身就没法做好这事儿。

关于第一条，以 JNI 为例。Android JNI 函数高达两百多个，其中一小半在 Unidbg 中都尚未实现，像下面这样。

```java
Pointer _CallByteMethodA = svcMemory.registerSvc(new ArmSvc() {
    @Override
    public long handle(Emulator<?> emulator) {
        throw new UnsupportedOperationException();
    }
});
```

这还是 Unidbg 持续更新，修修补补的结果，如果读者遇到这些尚未实现的 JNI 函数，就不得不自己补函数实现，我们在之前的文章对这个问题做过讨论。

系统调用的模拟实现也存在同样情况，即部分系统调用尚未实现。这是我们讨论补系统调用的第一块内容。

关于第二条，同样以 JNI 为例。前面我们将 JNI 函数分成两部分，一部分自身逻辑可以在 Unidbg 中良好模拟，不需要用户做任何处理，比如`GetStringUTFLength`、`GetStringUTFChars`等等。另一部分则必须依赖外部环境，比如`GetStaticIntField`、`CallStaticObjectMethod`等各类访问字段、调用方法的 JNI 函数。

这时候我们会重写所依赖的 JNI 函数，对具体的方法调用做处理，像下面这样。

```java
@Override
public boolean callBooleanMethod(BaseVM vm, DvmObject<?> dvmObject, String signature, VarArg varArg) {
    switch (signature){
        case "java/util/Map->isEmpty()Z":{
            Map map = (Map) dvmObject.getValue();
            return map.isEmpty();
        }
    }
    return super.callBooleanMethod(vm, dvmObject, signature, varArg);
}
```

在文件访问部分，这种无法自洽，需要外部信息的情况更明显。当发生文件访问时，我们从真实设备上拷贝出对应文件或文件的内容，通过代码或虚拟目录的方式“放”到 Unidbg 这个环境里来，这就是补文件访问的主要流程。比如下面这样。

```java
@Override
public FileResult resolve(Emulator emulator, String pathname, int oflags) {
    System.out.println("file open:"+pathname);
    if(pathname.equals("/proc/self/cmdline")){
        return FileResult.success(new ByteArrayFileIO(oflags, pathname, "tv.danmaku.bili\0".getBytes(StandardCharsets.UTF_8)));
    }
    return null;
}
```

在系统调用层面，同样存在这种问题，即某些系统调用的模拟不完善或不齐全，需要使用者做干涉和处理。但是相较于 JNI 或文件访问，这种处理其实是不太一样。最大的差异在于，JNI 或文件访问的补环境，主要问题是怎么补好，让它实现和真实情况下的等价逻辑。而补系统调用，主要是想让它继续往下执行，预期会低很多，因为很多系统调用确实很难处理。

## 四、尾声

从下一篇开始具体讨论概述中所说的三个问题。




# unidbg补系统调用原理和实践
### 一、引言

本篇讨论系统调用未实现的情况。本节样例是 [📎demo.7z](https://www.yuque.com/attachments/yuque/0/2022/7z/707476/1668097973681-d6deed98-5393-4b3a-bcfb-19382455b50a.7z)。

## 二、报错分析

JADX 反编译目 APK，找到目标类`com.example.demo.MainActivity`，其中的`stringFromJNI`函数就是我们的目标。

```java
package com.example.demo;

import android.os.Bundle;
import android.view.View;
import android.widget.Button;
import android.widget.TextView;
import androidx.appcompat.app.AppCompatActivity;
import com.example.demo.databinding.ActivityMainBinding;

public class MainActivity extends AppCompatActivity {
    private ActivityMainBinding binding;

    public native String stringFromJNI();

    static {
        System.loadLibrary("demo");
    }

    @Override // androidx.fragment.app.FragmentActivity, androidx.activity.ComponentActivity, androidx.core.app.ComponentActivity, android.app.Activity
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        ActivityMainBinding inflate = ActivityMainBinding.inflate(getLayoutInflater());
        this.binding = inflate;
        setContentView(inflate.mo181getRoot());
        final TextView tv = this.binding.sampleText;
        Button btn = (Button) findViewById(R.id.button);
        btn.setOnClickListener(new View.OnClickListener() { // from class: com.example.demo.MainActivity.1
            @Override // android.view.View.OnClickListener
            public void onClick(View view) {
                tv.setText(MainActivity.this.stringFromJNI());
            }
        });
    }
}
```

接下来用 Unidbg 做写个基本代码

```java
package com.example;

import com.github.unidbg.AndroidEmulator;
import com.github.unidbg.arm.backend.Unicorn2Factory;
import com.github.unidbg.linux.android.AndroidEmulatorBuilder;
import com.github.unidbg.linux.android.AndroidResolver;
import com.github.unidbg.linux.android.dvm.AbstractJni;
import com.github.unidbg.linux.android.dvm.DalvikModule;
import com.github.unidbg.linux.android.dvm.DvmClass;
import com.github.unidbg.linux.android.dvm.VM;
import com.github.unidbg.memory.Memory;

import java.io.File;

public class Demo extends AbstractJni {
    private final AndroidEmulator emulator;
    private final DvmClass MainActivity;
    private final VM vm;

    public Demo() {
        emulator = AndroidEmulatorBuilder
                .for64Bit()
                .addBackendFactory(new Unicorn2Factory(true))
                .build();

        Memory memory = emulator.getMemory();
        memory.setLibraryResolver(new AndroidResolver(23));
        vm = emulator.createDalvikVM(new File("unidbg-android/src/test/resources/demo/demo.apk"));
        vm.setJni(this);
        vm.setVerbose(true);
        DalvikModule dm = vm.loadLibrary("demo", true);
        MainActivity = vm.resolveClass("com.example.demo.MainActivity");
        dm.callJNI_OnLoad(emulator);
    }

    public String call(){
        return MainActivity.newObject(null).callJniMethodObject(emulator, "stringFromJNI").getValue().toString();
    }

    public static void main(String[] args) {
        Demo demo = new Demo();
        System.out.println("ret:"+demo.call());
    }

}
```

运行报错

```java
Find native function Java_com_example_demo_MainActivity_stringFromJNI => RX@0x4000073c[libdemo.so]0x73c
[00:46:49 186]  WARN [com.github.unidbg.linux.ARM64SyscallHandler] (ARM64SyscallHandler:399) - handleInterrupt intno=2, NR=165, svcNumber=0x0, PC=RX@0x401ba3d4[libc.so]0x6a3d4, LR=RX@0x40000770[libdemo.so]0x770, syscall=null
exit with code: 1
FORTIFY: vsprintf: prevented write past end of buffer
[crash]A/libc: FORTIFY: vsprintf: prevented write past end of buffer
Exception in thread "main" java.lang.NullPointerException: Cannot invoke "com.github.unidbg.linux.android.dvm.DvmObject.getValue()" because the return value of "com.github.unidbg.linux.android.dvm.DvmObject.callJniMethodObject(com.github.unidbg.Emulator, String, Object[])" is null
	at com.example.Demo.call(Demo.java:38)
	at com.example.Demo.main(Demo.java:43)
```

首先是怎么看报错

就像我们前面讲的其他类型报错一样，遇到报错必须得从上往下看，因为下面的报错可能是“果”，上面的报错是“因”。因此这里直接看底部的`FORTIFY: vsprintf: prevented write past end of buffer`这个 crash 是不对的。顶部是`WARN`才是最首先的报错。

这虽然是一个很简单的点，但很多人在补环境时会忽视，习惯性的只看底部。

其次是为什么确定是系统调用相关的调用，而非 JNI 或文件访问等等。比如 JNI 调用看起来和这里很类似。

```java
[01:00:43 681]  WARN [com.github.unidbg.linux.ARM64SyscallHandler] (ARM64SyscallHandler:399) - handleInterrupt intno=2, NR=-129152, svcNumber=0x16f, PC=unidbg@0xfffe0784, LR=RX@0x4003b5e0[libtiger_tally.so]0x3b5e0, syscall=null
java.lang.UnsupportedOperationException: com/aliyun/TigerTally/A->ct()Landroid/content/Context;
	at com.github.unidbg.linux.android.dvm.AbstractJni.callStaticObjectMethodV(AbstractJni.java:503)
	at com.aliyun.TigerTally.callStaticObjectMethodV(TigerTally.java:151)
	at com.github.unidbg.linux.android.dvm.AbstractJni.callStaticObjectMethodV(AbstractJni.java:437)
	at com.github.unidbg.linux.android.dvm.DvmMethod.callStaticObjectMethodV(DvmMethod.java:64)
	at com.github.unidbg.linux.android.dvm.DalvikVM64$112.handle(DalvikVM64.java:1748)
	at com.github.unidbg.linux.ARM64SyscallHandler.hook(ARM64SyscallHandler.java:118)
```

从特征上看，系统调用报错和 JNI 报错的“开头”一样，比如 64 位里都是下面这句 log。

```java
log.warn("handleInterrupt intno=" + intno + ", NR=" + NR + ", svcNumber=0x" + Integer.toHexString(swi) + ", PC=" + pc + ", LR=" + UnidbgPointer.register(emulator, Arm64Const.UC_ARM64_REG_LR) + ", syscall=" + syscall, exception);
```

这是为什么，JNI 调用和系统调用，在实际 Android 的代码逻辑里显然八竿子打不着，为什么在 Unidbg 里，两者报错时开头一致？我们要好好讨论一下这个问题，它确实有讨论的意义。

首先我们将它作为一个系统调用的报错来看，如何理解它。就看上面的这个日志，涉及到下面这些字段。

- intno
- NR
- svcNumber
- PC
- LR
- syscall

PC 和 LR 自不必说，即报错所对应的 SO 中的发生地址以及这个地址所处函数的返回地址。接下来讨论其他字段。

`intno`是异常类型，异常有很多种，比如未定义的指令，软中断、软断点等等，Unidbg 或者说 Unicorn 里对它们的定义如下。

```java
int EXCP_UDEF = 1; /* undefined instruction */
int EXCP_SWI = 2; /* software interrupt */
int EXCP_BKPT = 7; /* software breakpoint */
```

`SVC`指令就是软中断，对应于定义里的`EXCP_SWI`，我们熟悉的系统调用就是通过它发起。因此如果`into`的值是 2，就说明这是一个软中断，也就是系统调用。

如果是其他中断类型，即未定义的指令或者软件中断，在 Unidbg 里会直接断下，交由用户处置，但很少碰到这两种情况。

```java
if (intno == ARMEmulator.EXCP_BKPT) { // bkpt
    createBreaker(emulator).brk(pc, bkpt);
    return;
}
if (intno == ARMEmulator.EXCP_UDEF) {
    createBreaker(emulator).debug();
    return;
}

if (intno != ARMEmulator.EXCP_SWI) {
    throw new BackendException("intno=" + intno);
}
```

NR 就是所谓的调用号，32 位存在 R7 寄存器，64 位存在 X8 寄存器。以 64 位为例，Unidbg 代码如下，从 X8 寄存器取出值。

```java
int NR = backend.reg_read(Arm64Const.UC_ARM64_REG_X8).intValue();
```

读者可能希望从调用号直接对应上系统调用的名字，目前 Unidbg 没有处理好这块的体验优化。

比如 JNI 部分，如果某个函数出了问题，那么就算它没实现，也没搞个空函数抛出异常，好让人知道这是哪个函数出了问题。就像下面这样。

```java
Pointer _CallByteMethodA = svcMemory.registerSvc(new ArmSvc() {
    @Override
    public long handle(Emulator<?> emulator) {
        throw new UnsupportedOperationException();
    }
});
```

在 Unidbg 0.9.5 及以下的老版本上没有这么做，函数没实现就单纯报地址异常，新版做了这样的优化罢了，更方便人去做分析。

Unidbg 对系统调用这块没有做这样的优化，或者说还没做全。`syscall`字段就是这个用途，它在语义上就是提供系统调用的函数名，就像下面这样。因为 132 所对应的系统调用尚未模拟，所以给 syscall 赋其名，WARN 报错的时候好歹打印系统调用对应的名字。

```java
case 175:
    backend.reg_write(ArmConst.UC_ARM_REG_R0, sigprocmask(emulator));
    return;
case 132:
    syscall = "getpgid";
    break;
case 136:
    backend.reg_write(ArmConst.UC_ARM_REG_R0, personality(backend));
    return;
```

但只有很少的几个系统调用这么做了

![](https://cdn.nlark.com/yuque/0/2022/png/707476/1668102632415-a21bc7c4-b1b0-4ade-9aa2-ab7183e1f012.png)

因此在更多时候，我们都需要查看系统调用表，以便确定到底是哪个系统调用出了问题。比如这里是 64 位下的 165 调用号。访问[网站](https://chromium.googlesource.com/chromiumos/docs/+/master/constants/syscalls.md#arm64-64_bit)，确认它是 getrusage 这个系统调用。

![](https://cdn.nlark.com/yuque/0/2022/png/707476/1668102766415-32d13670-f19a-45b4-86a3-70d5e9b8040a.png)

还有一个字段是 svcNumber，我们还没有提。它是`SVC`指令后面跟着的数字。这里大家可能有两点关切，1 是 Unidbg 如何解析这个值，2 是这个值有什么用。

先看第一点，首先是 Unidbg 如何获取这条汇编

```java
@Override
public void hook(Unicorn u, int intno, Object user) {
    int swi;
    if (is64Bit) {
        UnidbgPointer pc = UnidbgPointer.register(emulator, Arm64Const.UC_ARM64_REG_PC);
        swi = (pc.getInt(-4) >> 5) & 0xffff;
    } else {
        UnidbgPointer pc = UnidbgPointer.register(emulator, ArmConst.UC_ARM_REG_PC);
        boolean isThumb = ARM.isThumb(UnicornBackend.this);
        if (isThumb) {
            swi = pc.getShort(-2) & 0xff;
        } else {
            swi = pc.getInt(-4) & 0xffffff;
        }
    }
    callback.hook(UnicornBackend.this, intno, swi, user);
}
```

之前学过，这是一个 Unicorn 中断 Hook，按照 Unicorn 中断 Hook 的特性，它会断在`SVC xxx`的下一行代码。举个例子，如果对下面的汇编代码片段做中断 Hook，那么会断在 0x1000C 这条指令执行前。

```java
#0x10000 .text:00040F0C 07 C0 A0 E1                 MOV             R12, R7
#0x10004 .text:00040F10 4E 70 A0 E3                 MOV             R7, #0x4E ; 'N'
#0x10008 .text:00040F14 00 00 00 EF                 SVC             0
#0x1000C .text:00040F18 0C 70 A0 E1                 MOV             R7, R12
#0x10010 .text:00040F1C 01 0A 70 E3                 CMN             R0, #0x1000
```

为了获取 `SVC #imme`这条指令，Unidbg 要顺着 PC 往上解析一定的字节，ARM64 或 ARM32 的 ARM 模式下都是 4 个字节，Thumb 则是 2 字节。这就是`pc.getInt(-4)`以及`getShort(-2)`的作用。

然后是解析这个指令，常规方法肯定是使用 Capstone 这样的反汇编工具去做处理，但因为`SVC xxx`指令的反汇编很简单，以及 Capstone 反汇编的开销较大，能少用尽量少用。所以 Unidbg 采用了自定义反汇编。

我们以 ARM64 下的指令解析为例，展示如何做这条指令的反汇编。

文档：[📎DDI_0596_ARM_a64_instruction_set_architecture.pdf](https://www.yuque.com/attachments/yuque/0/2022/pdf/707476/1668131446242-ad31ba68-c8f5-4d31-b83b-4b9072e4434d.pdf)

从中找到 SVC 指令的位编码图（也叫位图）如下，我们要获取其中的 imm16。

![](https://cdn.nlark.com/yuque/0/2022/png/707476/1668131471561-a3d97259-030d-4c00-8067-83e1db51b10b.png)

首先右移 5 位，让 imm16 成为这个数的低 16 位，然后和 0xFFFF 做与运算，舍去了这个数的高位，只留下低 16 位。这么做就得到了 `imm16`的值。

```java
swi = (pc.getInt(-4) >> 5) & 0xffff;
```

第二个问题是这个值有什么用。

在前文我们说过，`SVC`后面跟着的值，也就是`SWI`，它并无实际意义，而且约定俗成为 0。那么为什么 Unidbg 还要解析出这个值呢？事实上，这是因为 Unidbg 并不仅仅用`SVC`指令去处理系统调用。它还用 SVC 指令去做 JNI 跳板函数以及函数 Hook，其中 JNI 跳板函数这一块用到了`SWI`。我们简单讨论一下 JNI 函数，关于 函数 Hook 放到具体的文章里再讨论。

一个 JNI 跳板函数的实现如下

```java
svc #SWI;
bx lr
```

当陷入中断 Hook 时，解析`SWI`，如果这个立即数不为 0 ，那么就是一处 JNI 跳板函数而非系统调用，然后根据立即数的具体值确定是哪一个 JNI 函数。

换句话说，在 Unidbg 里，JNI 调用被“提升”到了和系统调用相同的级别，这就可以解释为什么 JNI 报错也发生在 syscallHandler 这件事了。

我们可以再回过头看一下 JNI 报错

```java
[01:00:43 681]  WARN [com.github.unidbg.linux.ARM64SyscallHandler] (ARM64SyscallHandler:399) - handleInterrupt intno=2, NR=-129152, svcNumber=0x16f, PC=unidbg@0xfffe0784, LR=RX@0x4003b5e0[libtiger_tally.so]0x3b5e0, syscall=null
```

`intno`是 2，因为 JNI 也是通过`SVC`指令处理。

`NR`在这里是 -129152，这显然不是任何一个合理的调用号，这实际上是因为 JNI 调用并不会对 R7、X8 做赋值，所以它是外部上下文中错乱的寄存器值。

`svcNumber`是 0x16F，而不是 0，所以它是 JNI 调用而非系统调用。

这里我们捋顺了 JNI 报错以及系统调用都报错于`xxxSyscallHandler`的原因，分辨两者也很简单。

1 是`svcNumber`是否为 0，2 是`xxxSyscallHandler`报错后面是否跟着来自`AbstractJni` 的经典异常，像下面这样。

```java
java.lang.UnsupportedOperationException: com/aliyun/TigerTally/A->ct()Landroid/content/Context;
	at com.github.unidbg.linux.android.dvm.AbstractJni.callStaticObjectMethodV(AbstractJni.java:503)
	at com.aliyun.TigerTally.callStaticObjectMethodV(TigerTally.java:151)
	at com.github.unidbg.linux.android.dvm.AbstractJni.callStaticObjectMethodV(AbstractJni.java:437)
	at com.github.unidbg.linux.android.dvm.DvmMethod.callStaticObjectMethodV(DvmMethod.java:64)
	at com.github.unidbg.linux.android.dvm.DalvikVM64$112.handle(DalvikVM64.java:1748)
	at com.github.unidbg.linux.ARM64SyscallHandler.hook(ARM64SyscallHandler.java:118)
```

## 三、样本分析

接下来可以聚焦在样本上了

```java
Find native function Java_com_example_demo_MainActivity_stringFromJNI => RX@0x4000073c[libdemo.so]0x73c
[00:46:49 186]  WARN [com.github.unidbg.linux.ARM64SyscallHandler] (ARM64SyscallHandler:399) - handleInterrupt intno=2, NR=165, svcNumber=0x0, PC=RX@0x401ba3d4[libc.so]0x6a3d4, LR=RX@0x40000770[libdemo.so]0x770, syscall=null
exit with code: 1
FORTIFY: vsprintf: prevented write past end of buffer
[crash]A/libc: FORTIFY: vsprintf: prevented write past end of buffer
Exception in thread "main" java.lang.NullPointerException: Cannot invoke "com.github.unidbg.linux.android.dvm.DvmObject.getValue()" because the return value of "com.github.unidbg.linux.android.dvm.DvmObject.callJniMethodObject(com.github.unidbg.Emulator, String, Object[])" is null
	at com.example.Demo.call(Demo.java:38)
	at com.example.Demo.main(Demo.java:43)
```

`NR=165, svcNumber=0x0`，所以它是系统调用，而且具体的系统调用我们查过，对应于 getrusage。

接下来到 PC 和 LR 分别看看具体什么情况

PC 是 libc 的 0x6a3d4 地址处，将`src/main/resources/android/sdk23/lib64/libc.so`从 Unidbg 里拷贝出来，放到 IDA 里解析（千万不要从手机里 pull 出 libc 然后分析，这是明朝的剑斩清朝的官）。

```java
.text:000000000006A3CC                 EXPORT getrusage
.text:000000000006A3CC getrusage                               ; DATA XREF: LOAD:0000000000004BF8↑o
.text:000000000006A3CC ; __unwind {
.text:000000000006A3CC                 MOV             X8, #0xA5
.text:000000000006A3D0                 SVC             0
.text:000000000006A3D4                 CMN             X0, #1,LSL#12
.text:000000000006A3D8                 CINV            X0, X0, HI
.text:000000000006A3DC                 B.HI            __set_errno_internal
.text:000000000006A3E0                 RET
```

LR 是我们的样例，地址是 0x770。

```java
__int64 __fastcall Java_com_example_demo_MainActivity_stringFromJNI(_JNIEnv *a1)
{
    struct rusage usage; // [xsp+48h] [xbp-A8h] BYREF
    char v4[12]; // [xsp+DCh] [xbp-14h] BYREF
    __int64 v5; // [xsp+E8h] [xbp-8h]

    v5 = *(_QWORD *)(_ReadStatusReg(ARM64_SYSREG(3, 3, 13, 0, 2)) + 40);
    getrusage(RUSAGE_SELF, &usage);
    sub_7EC(v4, 10LL, &unk_920, *(_QWORD *)&usage.ru_utime + usage.ru_maxrss);
    return _JNIEnv::NewStringUTF(a1, v4);
}
```

这件事其实很清楚，涉及到三方。

首先是样本方，它其实就是访问了一个库函数，叫`getrusage`，通过它获取了一些数据。

其次是库函数方，libc 调用`getrusage`，发现它其实只是对同名系统调用`getrusage`的简单包装。

最后是 Unidbg 方，它需要模拟执行 getruage 函数，但是这个函数没有模拟。

![](https://cdn.nlark.com/yuque/0/2022/png/707476/1668134973296-16fe5f52-5287-448e-b2cf-a46487c6f2db.png)

就是这么一件事。

Unidbg 没有模拟这个函数，我们肯定得补上这个模拟。回顾一下 JNI 部分，如果某个 JNI 函数没实现，我们可以重写它，在系统调用部分也同理，我们可以实现自己的系统调用处理器。

```java
package com.example;

import com.github.unidbg.Emulator;
import com.github.unidbg.linux.ARM64SyscallHandler;
import com.github.unidbg.memory.SvcMemory;

public class DemoARM64SyscallHandler extends ARM64SyscallHandler {
    public DemoARM64SyscallHandler(SvcMemory svcMemory) {
        super(svcMemory);
    }

    // 处理尚未模拟实现的系统调用
    @Override
    protected boolean handleUnknownSyscall(Emulator<?> emulator, int NR) {
        if (NR == 165) {
            getrusage(emulator);
            return true;
        }
        return super.handleUnknownSyscall(emulator, NR);
    }

    private void getrusage(Emulator<?> emulator){

    }
}
```

我们的样例是 64 位的，所以这里继承自`ARM64SyscallHandler`，否则应该继承`ARM32SyscallHandler`。

接下来在目标类里做一些修改，使用我们自己的`DemoARM64SyscallHandler`。

```java
package com.example;

import com.github.unidbg.AndroidEmulator;
import com.github.unidbg.arm.backend.Unicorn2Factory;
import com.github.unidbg.file.linux.AndroidFileIO;
import com.github.unidbg.linux.android.AndroidARM64Emulator;
import com.github.unidbg.linux.android.AndroidARMEmulator;
import com.github.unidbg.linux.android.AndroidEmulatorBuilder;
import com.github.unidbg.linux.android.AndroidResolver;
import com.github.unidbg.linux.android.dvm.AbstractJni;
import com.github.unidbg.linux.android.dvm.DalvikModule;
import com.github.unidbg.linux.android.dvm.DvmClass;
import com.github.unidbg.linux.android.dvm.VM;
import com.github.unidbg.memory.Memory;
import com.github.unidbg.memory.SvcMemory;
import com.github.unidbg.unix.UnixSyscallHandler;

import java.io.File;

public class Demo extends AbstractJni {
    private final AndroidEmulator emulator;
    private final DvmClass MainActivity;
    private final VM vm;

    public Demo() {
        // 创建模拟器实例
        AndroidEmulatorBuilder builder = new AndroidEmulatorBuilder(true){
            @Override
            public AndroidEmulator build() {
                return new AndroidARM64Emulator(processName,rootDir,backendFactories) {
                    @Override
                    protected UnixSyscallHandler<AndroidFileIO> createSyscallHandler(SvcMemory svcMemory) {
                        return new DemoARM64SyscallHandler(svcMemory);
                    }
                };
            }
        };
        emulator = builder
                .addBackendFactory(new Unicorn2Factory(true))
                .build();

        Memory memory = emulator.getMemory();
        memory.setLibraryResolver(new AndroidResolver(23));
        vm = emulator.createDalvikVM(new File("unidbg-android/src/test/resources/demo/demo.apk"));
        vm.setJni(this);
        vm.setVerbose(true);
        DalvikModule dm = vm.loadLibrary("demo", true);
        MainActivity = vm.resolveClass("com.example.demo.MainActivity");
        dm.callJNI_OnLoad(emulator);
    }

    public String call(){
        return MainActivity.newObject(null).callJniMethodObject(emulator, "stringFromJNI").getValue().toString();
    }

    public static void main(String[] args) {
        Demo demo = new Demo();
        System.out.println("ret:"+demo.call());
    }

}
```

如果是 ARM32，那么关键代码应该改成下面这样。

```java
AndroidEmulatorBuilder builder = new AndroidEmulatorBuilder(false){
    @Override
    public AndroidEmulator build() {
        return new AndroidARMEmulator(processName,rootDir,backendFactories) {
            @Override
            protected UnixSyscallHandler<AndroidFileIO> createSyscallHandler(SvcMemory svcMemory) {
                return new DemoARM64SyscallHandler(svcMemory);
            }
        };
    }
};
```

再次运行程序

```java
Find native function Java_com_example_demo_MainActivity_stringFromJNI => RX@0x4000073c[libdemo.so]0x73c
调用 getrusage，我还没实现。
exit with code: 1
FORTIFY: vsprintf: prevented write past end of buffer
[crash]A/libc: FORTIFY: vsprintf: prevented write past end of buffer
Exception in thread "main" java.lang.NullPointerException: Cannot invoke "com.github.unidbg.linux.android.dvm.DvmObject.getValue()" because the return value of "com.github.unidbg.linux.android.dvm.DvmObject.callJniMethodObject(com.github.unidbg.Emulator, String, Object[])" is null
	at com.example.Demo.call(Demo.java:53)
	at com.example.Demo.main(Demo.java:58)
```

## 四、具体处理

看起来我们需要理解这个系统调用的语义，以及学习在 Unidbg 里模拟系统调用的一套规则。

### 4.1 getrusage 系统调用

这个系统调用用于查看进程资源消耗情况，具体介绍如下。

每个进程都会消耗诸如内存和 CPU 时间之类的系统资源，getrusage 系统调用允许一个进程监控自己及其子进程已经用掉的资源。

函数原型

```java
int getrusage(int who, struct rusage *usage);
```

参数`who`，它有四个可选值

```java
#define RUSAGE_SELF 0
#define RUSAGE_CHILDREN (- 1)
#define RUSAGE_BOTH (- 2)
#define RUSAGE_THREAD 1
```

0 表示查看当前进程，-1 表示查看所有子进程，-2 表示查看当前进程以及所有子进程 ，1 表示查看当前线程。其中 0 使用的最多。

参数`usage`是一个指向`rusage`结构体的指针，资源消耗情况会返回和填充到这个结构体里。

返回值：执行成功返回 0，发生错误返回 -1。

介绍了它的基本情况，接下来看 rusage 结构体。

```java
struct rusage {
    struct timeval ru_utime; /* user CPU time used */
    struct timeval ru_stime; /* system CPU time used */
    long   ru_maxrss;        /* maximum resident set size */
    long   ru_ixrss;         /* integral shared memory size */
    long   ru_idrss;         /* integral unshared data size */
    long   ru_isrss;         /* integral unshared stack size */
    long   ru_minflt;        /* page reclaims (soft page faults) */
    long   ru_majflt;        /* page faults (hard page faults) */
    long   ru_nswap;         /* swaps */
    long   ru_inblock;       /* block input operations */
    long   ru_oublock;       /* block output operations */
    long   ru_msgsnd;        /* IPC messages sent */
    long   ru_msgrcv;        /* IPC messages received */
    long   ru_nsignals;      /* signals received */
    long   ru_nvcsw;         /* voluntary context switches */
    long   ru_nivcsw;        /* involuntary context switches */
};
```

这个系统调用使用场景不太多，所以这里简单说一下。

首先是 ru_utime，它表示进程在用户模式下的 CPU 使用时长，返回一个 timeval 结构体。

然后是 ru_stime，它表示进程在系统模式下的 CPU 使用时长，同样返回 timeval 结构体。

```java
struct timeval {
  long tv_sec;
  long tv_usec;
};
```

不管是 ru_utime 还是 ru_stime，又或者两者相加，都不等价于这个进程的已打开时间，它仅仅是 CPU 在当前进程上的时间开销的一种度量。

接下来更简略的提一下其他信息

ru_maxrss：返回最大驻留集的大小，单位为kb。不太懂是个啥。

ru_ixrss、ru_idrss、ru_isrss：目前不支持

ru_minflt：缺页中断的次数，且处理这些中断不需要进行I/O

ru_majflt：缺页中断的次数，且处理这些中断需要进行I/O

ru_nswap：目前不支持

ru_inblock：文件系统需要进行输入操作的次数。

ru_oublock：文件系统需要进行输出操作的次数。

ru_msgsnd、ru_msgrcv、ru_nsignals：目前不支持

ru_nvcsw：因进程自愿放弃处理器时间片而导致的上下文切换的次数（通常是为了等待请求的资源）。

ru_nivcsw：因进程时间片使用完毕或被高优先级进程抢断导致的上下文切换的次数。

应该说，它确实返回了很多进程相关的资源消耗信息，但具体如何使用，其实还是不太好用的。

这个函数返回的是资源消耗情况，对于补这一类动态的、实时变化的数据，模拟其实没有什么好办法去处理。随机返回值可以带来动态，但它又会缺少合理性。因此事实上，在实现这类系统调用时，往往会选择“摆烂”，从真机 dump 下来一份数据，然后硬编码返回，或者给与一定的灵活性，由用户自己填充这个结构体。

我们不妨先看一下 Unidbg 以及其他模拟器是怎么处理这一情况以及相关函数。

### 4.2 通常处理

sysinfo、uname 这两个系统调用都是很好的参考范例。

#### sysinfo

它用于获取各种各样的系统统计信息，函数原型如下

```java
int sysinfo(struct sysinfo *info);
```

返回信息填充在 sysinfo 结构体里

```java
struct sysinfo {
  long uptime;             // 启动到现在经过的时间
  unsigned long loads[3];  // I/O 在 1/5/15 分钟内执行的平均时间
  unsigned long totalram;  // 总的可用内存
  unsigned long freeram;   // 还未被使用的内存信息
  unsigned long sharedram; // 共享存储器的大小
  unsigned long bufferram; // 缓冲区大小
  unsigned long totalswap; // 交换区大小
  unsigned long freeswap;  // 可用的交换区大小
  unsigned short procs;    // 当前进程数目
  unsigned short pad;      // 填充
  unsigned long totalhigh; // 高内存大小
  unsigned long freehigh;  // 可用的高内存大小
  unsigned int mem_unit;   // 内存单元大小
  char _f[20 - 2 * sizeof(unsigned long) - sizeof(unsigned int)];
};
```

这些信息很有用的，不管是单纯信息收集还是检测与对抗都能用得到。在实际场景中，往往会使用它的同名库函数，这一点和`getrusage`库函数类似。

```java
#include <stdio.h>
#include <sys/sysinfo.h>

int main(void)
{
    struct sysinfo info;

    sysinfo(&info);

    printf("uptime = %ld\n",info.uptime);
    printf("loads = %ld %ld %ld\n",info.loads[0],info.loads[1],info.loads[2]);
    printf("totalram = %lx\n",info.totalram*info.mem_unit);
    printf("freeram = %lx\n",info.freeram*info.mem_unit);
    printf("sharedram = %lx\n",info.sharedram*info.mem_unit);
    printf("bufferram = %lx\n",info.bufferram*info.mem_unit);
    printf("totalswap = %lx\n",info.totalswap*info.mem_unit);
    printf("freeswap = %lx\n",info.freeswap*info.mem_unit);
    printf("procs = %d\n",info.procs);
    printf("totalhigh = %lx\n",info.totalhigh*info.mem_unit);
    printf("freehigh = %lx\n",info.freehigh*info.mem_unit);
    printf("mem_unit = %d\n",info.mem_unit);

    return 0;
}
```

它的具体使用如下面这样，调用 sysinfo 库函数获取 uptime，即系统启动了多少秒，然后根据当前时间戳计算出设备开机的时间。

```java
int sub_11AB0()
{
  int v0; // r2
  int v1; // r0
  int v3; // [sp+4h] [bp-74h]
  int v4; // [sp+8h] [bp-70h]
  int v5; // [sp+14h] [bp-64h]
  int v6; // [sp+18h] [bp-60h] BYREF
  struct sysinfo v7; // [sp+1Ch] [bp-5Ch] BYREF

  v6 = 0;
  v5 = sysinfo(&v7);
  v4 = 1586062889;
  while ( 1 )
  {
    while ( v4 == 1586062889 )
    {
      v1 = 1529284392;
      if ( !v5 )
        v1 = 1833852255;
      v3 = 0;
      v4 = v1;
    }
    if ( v4 != 1833852255 )
      break;
    time((time_t *)&v6);
    v4 = 1529284392;
    v0 = v6 - v7.uptime;
    if ( v6 <= v7.uptime )
      v0 = v7.uptime - v6;
    v3 = v0;
  }
  return v3;
}
```

如何模拟 sysinfo 系统调用。最简单的办法就是从真机获取一份信息，直接硬编码过来。[ExAndroidNativeEmu](https://github.com/maiyao1988/ExAndroidNativeEmu) 就是这么做的，但对 uptime 这个相对敏感和重要的信息做了动态处理。

```java
def __sysinfo(self, mu, info_ptr):
    '''
        si = {sysinfo} 
        uptime = {__kernel_long_t} 91942
        loads = {__kernel_ulong_t [3]} 
        [0] = {__kernel_ulong_t} 503328
        [1] = {__kernel_ulong_t} 504576
        [2] = {__kernel_ulong_t} 537280
        totalram = {__kernel_ulong_t} 1945137152
        freeram = {__kernel_ulong_t} 47845376
        sharedram = {__kernel_ulong_t} 0
        bufferram = {__kernel_ulong_t} 169373696
        totalswap = {__kernel_ulong_t} 0
        freeswap = {__kernel_ulong_t} 0
        procs = {__u16} 1297
        pad = {__u16} 0
        totalhigh = {__kernel_ulong_t} 1185939456
        freehigh = {__kernel_ulong_t} 1863680
        mem_unit = {__u32} 1
        f = 0 char[8]
        '''
    uptime = int(self._clock_offset + time.time() - self._clock_start)
    if self.__emu.get_arch() == emu_const.ARCH_ARM32:
        mu.mem_write(info_ptr + 0, int(uptime).to_bytes(4, byteorder='little'))
        mu.mem_write(info_ptr + 4, int(503328).to_bytes(4, byteorder='little'))
        mu.mem_write(info_ptr + 8, int(504576).to_bytes(4, byteorder='little'))
        mu.mem_write(info_ptr + 12, int(537280).to_bytes(4, byteorder='little'))
        mu.mem_write(info_ptr + 16, int(1945137152).to_bytes(4, byteorder='little'))
        mu.mem_write(info_ptr + 20, int(47845376).to_bytes(4, byteorder='little'))
        mu.mem_write(info_ptr + 24, int(0).to_bytes(4, byteorder='little'))
        mu.mem_write(info_ptr + 28, int(169373696).to_bytes(4, byteorder='little'))
        mu.mem_write(info_ptr + 32, int(0).to_bytes(4, byteorder='little'))
        mu.mem_write(info_ptr + 36, int(0).to_bytes(4, byteorder='little'))
        mu.mem_write(info_ptr + 40, int(1297).to_bytes(2, byteorder='little'))
        mu.mem_write(info_ptr + 42, int(0).to_bytes(2, byteorder='little'))
        mu.mem_write(info_ptr + 44, int(1185939456).to_bytes(4, byteorder='little'))
        mu.mem_write(info_ptr + 48, int(1863680).to_bytes(4, byteorder='little'))
        mu.mem_write(info_ptr + 52, int(1).to_bytes(4, byteorder='little'))
        mu.mem_write(info_ptr + 56, int(0).to_bytes(8, byteorder='little'))
        #sz 64
    else:
        #arm64
        mu.mem_write(info_ptr + 0, int(uptime).to_bytes(8, byteorder='little'))
        mu.mem_write(info_ptr + 8, int(503328).to_bytes(8, byteorder='little'))
        mu.mem_write(info_ptr + 16, int(504576).to_bytes(8, byteorder='little'))
        mu.mem_write(info_ptr + 24, int(537280).to_bytes(8, byteorder='little'))
        mu.mem_write(info_ptr + 32, int(1945137152).to_bytes(8, byteorder='little'))
        mu.mem_write(info_ptr + 40, int(47845376).to_bytes(8, byteorder='little'))
        mu.mem_write(info_ptr + 48, int(0).to_bytes(8, byteorder='little'))
        mu.mem_write(info_ptr + 56, int(169373696).to_bytes(8, byteorder='little'))
        mu.mem_write(info_ptr + 64, int(0).to_bytes(8, byteorder='little'))
        mu.mem_write(info_ptr + 72, int(0).to_bytes(8, byteorder='little'))
        mu.mem_write(info_ptr + 80, int(1297).to_bytes(2, byteorder='little'))
        mu.mem_write(info_ptr + 82, int(0).to_bytes(6, byteorder='little')) #pading

        mu.mem_write(info_ptr + 88, int(0).to_bytes(8, byteorder='little'))
        mu.mem_write(info_ptr + 96, int(0).to_bytes(8, byteorder='little'))
        mu.mem_write(info_ptr + 104, int(1).to_bytes(4, byteorder='little'))
        mu.mem_write(info_ptr + 108, int(0).to_bytes(4, byteorder='little'))
        #sz 112
        #
        logging.warning("syscall sysinfo buf 0x%08X return fixed value"%(info_ptr))
        return 0
        #
```

[Qiling](https://github.com/qilingframework/qiling) 也是这么做的，而且 uptime 也是硬编码。

```java
def ql_syscall_sysinfo(ql: Qiling, info: int):
    # TODO: the packing method for 'long' fields is set to ql.pack even though
    # it is unclear whether it should match pointersize or be 64 bits anyway

    fields = (
        (0x0000000000001234, ql.pack),   # uptime
        (0x0000000000002000, ql.pack),   # loads (1 min)
        (0x0000000000002000, ql.pack),   # loads (5 min)
        (0x0000000000002000, ql.pack),   # loads (15 min)
        (0x0000000010000000, ql.pack),   # total ram
        (0x0000000010000000, ql.pack),   # free ram
        (0x0000000010000000, ql.pack),   # shared memory
        (0x0000000000000000, ql.pack),   # memory used by buffers
        (0x0000000000000000, ql.pack),   # total swap
        (0x0000000000000000, ql.pack),   # free swap
        (0x0001,             ql.pack16), # nb current processes
        (0x0000000000000000, ql.pack),   # total high mem
        (0x0000000000000000, ql.pack),   # available high mem
        (0x00000000,         ql.pack32)  # memory unit size
    )

    data = b''.join(pmethod(val) for val, pmethod in fields)

    ql.mem.write(info, data.ljust(64, b'\x00'))

    return 0
```

我们当然也可以在 Unidbg 里获取结构体指针，向对应的的地址写入这些值。但 Unidbg 在处理上更好那么一些。首先因为 long 的定义在 32/64 位上大小不同，所以分为了 SysInfo32 和 SysInfo64。

```java
private int sysinfo(Emulator<?> emulator) {
    Arm32RegisterContext context = emulator.getContext();
    Pointer info = context.getR0Pointer();
    if (log.isDebugEnabled()) {
        log.debug("sysinfo info=" + info);
    }
    SysInfo32 sysInfo32 = new SysInfo32(info);
    sysInfo32.pack();
    return 0;
}
```

Sysinfo32 类如下，它就是 sysinfo 结构体，Unidbg 将它封装为对应的类。

```java
package com.github.unidbg.linux.struct;

import com.github.unidbg.pointer.UnidbgStructure;
import com.sun.jna.Pointer;

import java.util.Arrays;
import java.util.List;

public class SysInfo32 extends UnidbgStructure {

    public SysInfo32(Pointer p) {
        super(p);
    }

    public int uptime;
    public int[] loads = new int[3];
    public int totalRam;
    public int freeRam;
    public int sharedRam;
    public int bufferRam;
    public int totalSwap;
    public int freeSwap;
    public short procs;
    public short pad;
    public int totalHigh;
    public int freeHigh;
    public int mem_unit;
    public byte[] _f = new byte[8];

    @Override
    protected List<String> getFieldOrder() {
        return Arrays.asList("uptime", "loads", "totalRam", "freeRam", "sharedRam", "bufferRam", "totalSwap", "freeSwap", "procs", "pad", "totalHigh", "freeHigh", "mem_unit", "_f");
    }
}
```

Unidbg 默认返回空结构体，交由读者对这个结构体做填充，比如像下面这样。

```java
private int sysinfo(Emulator<?> emulator) {
    Arm32RegisterContext context = emulator.getContext();
    Pointer info = context.getR0Pointer();
    if (log.isDebugEnabled()) {
        log.debug("sysinfo info=" + info);
    }
    SysInfo32 sysInfo32 = new SysInfo32(info);
    // 自定义
    sysInfo32.uptime = 1234;
    sysInfo32.bufferRam = 0x125600;
    sysInfo32.pack();
    return 0;
}
```

如果一个东西无法很好的 mock，Unidbg 会交给用户或者干脆不做实现。再看看 uname。

#### uname

它会返回一系列关于主机系统的标识信息，函数原型如下。

```java
int uname(struct utsname *buf);
```

具体看看 utsname 结构体，Android 对它的注释相当详细和好懂。

```java
/** The information returned by uname(). */
struct utsname {
  /** The OS name. "Linux" on Android. */
  char sysname[SYS_NMLN];
  /** The name on the network. Typically "localhost" on Android. */
  char nodename[SYS_NMLN];
  /** The OS release. Typically something like "4.4.115-g442ad7fba0d" on Android. */
  char release[SYS_NMLN];
  /** The OS version. Typically something like "#1 SMP PREEMPT" on Android. */
  char version[SYS_NMLN];
  /** The hardware architecture. Typically "aarch64" on Android. */
  char machine[SYS_NMLN];
  /** The domain name set by setdomainname(). Typically "localdomain" on Android. */
  char domainname[SYS_NMLN];
};
```

我们可以在 adb shell 里通过`uname`命令获取这些信息（毫无疑问，这个命令的底层也依赖于 uname 系统调用），除了不重要的 domainname。

```java
polaris:/ $ uname --help
usage: uname [-asnrvm]

Print system information.

-s      System name
-n      Network (domain) name
-r      Kernel Release number
-v      Kernel Version
-m      Machine (hardware) name
-a      All of the above

polaris:/ $ uname -s
Linux
polaris:/ $ uname -n
localhost
polaris:/ $ uname -r
4.9.186-perf-gd3d6708
polaris:/ $ uname -v
#1 SMP PREEMPT Wed Nov 4 01:05:59 CST 2020
polaris:/ $ uname -m
aarch64
polaris:/ $ uname -a
Linux localhost 4.9.186-perf-gd3d6708 #1 SMP PREEMPT Wed Nov 4 01:05:59 CST 2020 aarch64
```

来看看 ExAndroidNativeEmu 的实现，依然是那个风格，找个真机的 uname 信息一顿填充。

```java
def __uname(self, mu, buf):
    #    uts.sysname = Linux
    #    uts.nodename = localhost
    #    uts.release = 3.10.73-gf97f123
    #    uts.version = #1 SMP PREEMPT Mon Nov 2 20:10:58 UTC 2015
    #    uts.machine = armv8l
    #    uts.domainname = localdomain

    memory_helpers.write_utf8(mu, buf + 0, "Linux")
    memory_helpers.write_utf8(mu, buf + 65, "localhost")
    memory_helpers.write_utf8(mu, buf + 130, "3.10.73-gf97f123")
    memory_helpers.write_utf8(mu, buf + 195, "#1 SMP PREEMPT Mon Nov 2 20:10:58 UTC 2015")
    memory_helpers.write_utf8(mu, buf + 260, "armv8l")
    memory_helpers.write_utf8(mu, buf + 325, "localdomain")
```

再看看 Unidbg 的处理

```java
protected int uname(Emulator<?> emulator) {
    Pointer buf = UnidbgPointer.register(emulator, ArmConst.UC_ARM_REG_R0);
    if (log.isDebugEnabled()) {
        log.debug("uname buf=" + buf);
    }

    final int SYS_NMLN = 65;

    Pointer sysname = buf.share(0);
    sysname.setString(0, "Linux");

    Pointer nodename = sysname.share(SYS_NMLN);
    nodename.setString(0, "localhost");

    Pointer release = nodename.share(SYS_NMLN);
    release.setString(0, "1.0.0-unidbg");

    Pointer version = release.share(SYS_NMLN);
    version.setString(0, "#1 SMP PREEMPT Thu Apr 19 14:36:58 CST 2018");

    Pointer machine = version.share(SYS_NMLN);
    machine.setString(0, "armv7l");

    Pointer domainname = machine.share(SYS_NMLN);
    domainname.setString(0, "localdomain");

    return 0;
}
```

其中发行版 “1.0.0-unidbg”是一个醒目的特征。Unidbg 无意隐藏自身环境，Qiling 也没有做隐藏。

```java
def ql_syscall_uname(ql: Qiling, buf: int):
    UTSLEN = 65

    fields = (
        b'QilingOS',                 # sysname
        b'ql_vm',                    # nodename
        b'99.0-RELEASE',             # release
        b'QilingOS 99.0-RELEASE r1', # version
        b'ql_processor',             # machine
        b''                          # domainname
    )

    for i, f in enumerate(fields):
        ql.mem.write(buf + i * UTSLEN, f.ljust(UTSLEN, b'\x00'))

    return 0
```

### 4.3 补 getrusage

简而言之，要么基于真实环境 hook、dump 出来一份，要么交给用户自己填充。我们两个方案都试一下。

首先看第一个方案，针对我们的 demo 写一下代码。

```java
Interceptor.attach(Module.findExportByName("libc.so", "getrusage"), {
  onEnter: function (args) {
    this.rusage = args[1]
  },
  onLeave: function (retval) {
    console.log(hexdump(this.rusage))
  }
});
```

最好还是确定结构体的长度，方便打印，这里只考虑 64 位。

```java
struct rusage {
    struct timeval ru_utime; /* user CPU time used */
    struct timeval ru_stime; /* system CPU time used */
    long   ru_maxrss;        /* maximum resident set size */
    long   ru_ixrss;         /* integral shared memory size */
    long   ru_idrss;         /* integral unshared data size */
    long   ru_isrss;         /* integral unshared stack size */
    long   ru_minflt;        /* page reclaims (soft page faults) */
    long   ru_majflt;        /* page faults (hard page faults) */
    long   ru_nswap;         /* swaps */
    long   ru_inblock;       /* block input operations */
    long   ru_oublock;       /* block output operations */
    long   ru_msgsnd;        /* IPC messages sent */
    long   ru_msgrcv;        /* IPC messages received */
    long   ru_nsignals;      /* signals received */
    long   ru_nvcsw;         /* voluntary context switches */
    long   ru_nivcsw;        /* involuntary context switches */
};
```

timeval 是两个 long，也就是 16 字节。

```java
struct timeval {
  long tv_sec;
  long tv_usec;
};
```

16 * 2 + 14 * 8 = 144 字节

测试 Frida Hook，需要手动点击 demo 的按钮触发调用逻辑。

```java
PS D:\xxx\files\jscode> frida -U -f com.example.demo -l emo/hookGetrusage.js --no-pause
     ____
    / _  |   Frida 14.2.18 - A world-class dynamic instrumentation toolkit
   | (_| |
    > _  |   Commands:
   /_/ |_|       help      -> Displays the help system
   . . . .       object?   -> Display information about 'object'
   . . . .       exit/quit -> Exit
   . . . .
   . . . .   More info at https://frida.re/docs/home/
Spawned `com.example.demo`. Resuming main thread!
[Pixel 2::com.example.demo]->              0  1  2  3  4  5  6  7  8  9  A  B  C  D  E  F  0123456789ABCDEF
7fe0f75328  00 00 00 00 00 00 00 00 9f 4a 0b 00 00 00 00 00  .........J......
7fe0f75338  00 00 00 00 00 00 00 00 c5 e1 01 00 00 00 00 00  ................
7fe0f75348  90 52 01 00 00 00 00 00 00 00 00 00 00 00 00 00  .R..............
7fe0f75358  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
7fe0f75368  25 5e 00 00 00 00 00 00 00 00 00 00 00 00 00 00  %^..............
7fe0f75378  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
7fe0f75388  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
7fe0f75398  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
7fe0f753a8  0d 02 00 00 00 00 00 00 d3 00 00 00 00 00 00 00  ................
```

接下来赋值，或者说硬编码到 Unidbg getrusage 实现里。首先我用 [Cyberchef](https://gchq.github.io/CyberChef/#recipe=From_Hexdump\(\)To_Hex\('None',0\)&input=N2ZlMGY3NTMyOCAgMDAgMDAgMDAgMDAgMDAgMDAgMDAgMDAgOWYgNGEgMGIgMDAgMDAgMDAgMDAgMDAgIC4uLi4uLi4uLkouLi4uLi4KN2ZlMGY3NTMzOCAgMDAgMDAgMDAgMDAgMDAgMDAgMDAgMDAgYzUgZTEgMDEgMDAgMDAgMDAgMDAgMDAgIC4uLi4uLi4uLi4uLi4uLi4KN2ZlMGY3NTM0OCAgOTAgNTIgMDEgMDAgMDAgMDAgMDAgMDAgMDAgMDAgMDAgMDAgMDAgMDAgMDAgMDAgIC5SLi4uLi4uLi4uLi4uLi4KN2ZlMGY3NTM1OCAgMDAgMDAgMDAgMDAgMDAgMDAgMDAgMDAgMDAgMDAgMDAgMDAgMDAgMDAgMDAgMDAgIC4uLi4uLi4uLi4uLi4uLi4KN2ZlMGY3NTM2OCAgMjUgNWUgMDAgMDAgMDAgMDAgMDAgMDAgMDAgMDAgMDAgMDAgMDAgMDAgMDAgMDAgICVeLi4uLi4uLi4uLi4uLi4KN2ZlMGY3NTM3OCAgMDAgMDAgMDAgMDAgMDAgMDAgMDAgMDAgMDAgMDAgMDAgMDAgMDAgMDAgMDAgMDAgIC4uLi4uLi4uLi4uLi4uLi4KN2ZlMGY3NTM4OCAgMDAgMDAgMDAgMDAgMDAgMDAgMDAgMDAgMDAgMDAgMDAgMDAgMDAgMDAgMDAgMDAgIC4uLi4uLi4uLi4uLi4uLi4KN2ZlMGY3NTM5OCAgMDAgMDAgMDAgMDAgMDAgMDAgMDAgMDAgMDAgMDAgMDAgMDAgMDAgMDAgMDAgMDAgIC4uLi4uLi4uLi4uLi4uLi4KN2ZlMGY3NTNhOCAgMGQgMDIgMDAgMDAgMDAgMDAgMDAgMDAgZDMgMDAgMDAgMDAgMDAgMDAgMDAgMDAgIC4uLi4uLi4uLi4uLi4uLi4) 把 hexdump 出的内容转成十六进制字符串。

![](https://cdn.nlark.com/yuque/0/2022/png/707476/1668147510249-8116fba5-8458-4d5f-9559-2094c2a78900.png)

然后填充到 Unidbg 里，看起来还不错。

```java
package com.example;

import com.github.unidbg.Emulator;
import com.github.unidbg.linux.ARM64SyscallHandler;
import com.github.unidbg.memory.SvcMemory;
import com.github.unidbg.pointer.UnidbgPointer;
import com.sun.jna.Pointer;
import unicorn.Arm64Const;

public class DemoARM64SyscallHandler extends ARM64SyscallHandler {
    public DemoARM64SyscallHandler(SvcMemory svcMemory) {
        super(svcMemory);
    }

    // 处理尚未模拟实现的系统调用
    @Override
    protected boolean handleUnknownSyscall(Emulator<?> emulator, int NR) {
        if (NR == 165) {
            getrusage(emulator);
            return true;
        }
        return super.handleUnknownSyscall(emulator, NR);
    }

    private void getrusage(Emulator<?> emulator){
        Pointer rusage = UnidbgPointer.register(emulator, Arm64Const.UC_ARM64_REG_X1);
        byte[] rusageContent = hexStringToByteArray("00000000000000009f4a0b00000000000000000000000000c5e10100000000009052010000000000000000000000000000000000000000000000000000000000255e00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000d02000000000000d300000000000000");
        for(int i = 0;i< rusageContent.length;i++){
            rusage.setByte(i, rusageContent[i]);
        }
    }

    /* s must be an even-length string. */
    public static byte[] hexStringToByteArray(String s) {
        int len = s.length();
        byte[] data = new byte[len / 2];
        for (int i = 0; i < len; i += 2) {
            data[i / 2] = (byte) ((Character.digit(s.charAt(i), 16) << 4)
                                  + Character.digit(s.charAt(i+1), 16));
        }
        return data;
    }
}
```

如果时间充裕，也可以一个个 long 去填充，看起来更清楚一些。

```java
package com.example;

import com.github.unidbg.Emulator;
import com.github.unidbg.linux.ARM64SyscallHandler;
import com.github.unidbg.memory.SvcMemory;
import com.github.unidbg.pointer.UnidbgPointer;
import com.sun.jna.Pointer;
import unicorn.Arm64Const;

public class DemoARM64SyscallHandler extends ARM64SyscallHandler {
    public DemoARM64SyscallHandler(SvcMemory svcMemory) {
        super(svcMemory);
    }

    // 处理尚未模拟实现的系统调用
    @Override
    protected boolean handleUnknownSyscall(Emulator<?> emulator, int NR) {
        if (NR == 165) {
            getrusage(emulator);
            return true;
        }
        return super.handleUnknownSyscall(emulator, NR);
    }

    /*
    00 00 00 00 00 00 00 00 9f 4a 0b 00 00 00 00 00  .........J......
    00 00 00 00 00 00 00 00 c5 e1 01 00 00 00 00 00  ................
    90 52 01 00 00 00 00 00 00 00 00 00 00 00 00 00  .R..............
    00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
    25 5e 00 00 00 00 00 00 00 00 00 00 00 00 00 00  %^..............
    00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
    00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
    00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
    0d 02 00 00 00 00 00 00 d3 00 00 00 00 00 00 00  ................
     */
    private void getrusage(Emulator<?> emulator){
        Pointer rusage = UnidbgPointer.register(emulator, Arm64Const.UC_ARM64_REG_X1);
        rusage.setLong(0, 0);
        rusage.setLong(8, 0xB4A9FL);
        rusage.setLong(16, 0);
        rusage.setLong(24, 0x1E1C5L);
        rusage.setLong(32, 0x15290L);
        // 继续往下
    }

}
```

接下来试试第二个方案，即交由用户处理。我们要针对 rusage 结构体去写个对应的类。其实这个事有参考代码，Unidbg 在 IOS 的处理上提供了 getrusage 系统调用的模拟实现。

```java
package com.github.unidbg.ios.struct.kernel;

import com.github.unidbg.pointer.UnidbgStructure;
import com.github.unidbg.unix.struct.TimeVal64;
import com.sun.jna.Pointer;

import java.util.Arrays;
import java.util.List;

public class RUsage64 extends UnidbgStructure {

    public TimeVal64 ru_utime; /* user time used */
    public TimeVal64 ru_stime; /* system time used */
    public long ru_maxrss;          /* max resident set size */
    public long ru_ixrss;           /* integral shared text memory size */
    public long ru_idrss;           /* integral unshared data size */
    public long ru_isrss;           /* integral unshared stack size */
    public long ru_minflt;          /* page reclaims */
    public long ru_majflt;          /* page faults */
    public long ru_nswap;           /* swaps */
    public long ru_inblock;         /* block input operations */
    public long ru_oublock;         /* block output operations */
    public long ru_msgsnd;          /* messages sent */
    public long ru_msgrcv;          /* messages received */
    public long ru_nsignals;        /* signals received */
    public long ru_nvcsw;           /* voluntary context switches */
    public long ru_nivcsw;          /* involuntary context switches */

    public void fillDefault() {
        ru_utime.tv_sec = 1;
        ru_utime.tv_usec = System.nanoTime();
        ru_stime.tv_sec = 2;
        ru_stime.tv_usec = System.nanoTime();
        ru_maxrss = 0;
        ru_ixrss = 0;
        ru_idrss = 0;
        ru_isrss = 0;
        ru_minflt = 0;
        ru_majflt = 0;
        ru_nswap = 0;
        ru_inblock = 0;
        ru_oublock = 0;
        ru_msgsnd = 0;
        ru_msgrcv = 0;
        ru_nsignals = 0;
        ru_nvcsw = 0;
        ru_nivcsw = 0;
    }

    public RUsage64(Pointer p) {
        super(p);
    }

    @Override
    protected List<String> getFieldOrder() {
        return Arrays.asList("ru_utime", "ru_stime", "ru_maxrss", "ru_ixrss", "ru_idrss", "ru_isrss",
                "ru_minflt", "ru_majflt", "ru_nswap", "ru_inblock", "ru_oublock",
                "ru_msgsnd", "ru_msgrcv", "ru_nsignals", "ru_nvcsw", "ru_nivcsw");
    }
}
```

IOS 和此处几乎等价，可以参考和使用。

具体模拟逻辑如下

```java
private long getrusage(Emulator<DarwinFileIO> emulator) {
    RegisterContext context = emulator.getContext();
    int who = context.getIntArg(0);
    Pointer r_usage = context.getPointerArg(1);
    RUsage64 usage64 = new RUsage64(r_usage);
    usage64.unpack();
    if (log.isDebugEnabled()) {
        log.debug("getrusage who=" + who + ", r_usage=" + r_usage + ", usage64=" + usage64);
    }
    usage64.fillDefault();
    usage64.pack();
    return 0;
}
```

确实更正规，也更方便“编辑”。

## 五、尾声

本篇对一个简单的样本做了较为详细的讨论，读者遇到 Unidbg 尚未完善的数据获取相关的系统调用，都可以这么处理。



# unidbg补系统调用实践
## 一、引言

本篇继续讨论 Unidbg 中补系统调用相关的处理。样例以及所涉及的文件 [📎dewu.7z](https://www.yuque.com/attachments/yuque/0/2022/7z/707476/1668160141802-da7da032-6a90-4a5f-96c2-683a2b567792.7z)，其中 4.94.apk 是目标 APK。样本有多个系统调用相关的问题，十分适合我们做分析。

本篇基础代码如下。

```java
package com.dewu;

import com.github.unidbg.AndroidEmulator;
import com.github.unidbg.Emulator;
import com.github.unidbg.Module;
import com.github.unidbg.arm.backend.Unicorn2Factory;
import com.github.unidbg.file.FileResult;
import com.github.unidbg.file.IOResolver;
import com.github.unidbg.linux.android.AndroidEmulatorBuilder;
import com.github.unidbg.linux.android.AndroidResolver;
import com.github.unidbg.linux.android.SystemPropertyHook;
import com.github.unidbg.linux.android.SystemPropertyProvider;
import com.github.unidbg.linux.android.dvm.*;
import com.github.unidbg.linux.file.ByteArrayFileIO;
import com.github.unidbg.linux.file.SimpleFileIO;
import com.github.unidbg.memory.Memory;

import java.io.File;
import java.nio.charset.StandardCharsets;
import java.util.UUID;

public class dewuNew extends AbstractJni implements IOResolver {
    @Override
    public FileResult resolve(Emulator emulator, String pathname, int oflags) {
        System.out.println("file open:"+pathname);
        if(pathname.equals("proc/sys/kernel/random/boot")){
            return FileResult.success(new ByteArrayFileIO(oflags, pathname, UUID.randomUUID().toString().getBytes(StandardCharsets.UTF_8)));
        }
        if(pathname.equals("/proc/cpuinfo")){
            return FileResult.success(new SimpleFileIO(oflags, new File("unidbg-android/src/test/resources/dewu/files/cpuinfo"), pathname));
        }
        if(pathname.equals("/proc/version")){
            return FileResult.success(new SimpleFileIO(oflags, new File("unidbg-android/src/test/resources/dewu/files/version"), pathname));
        }
        return null;
    }

    private final AndroidEmulator emulator;
    private final VM vm;
    private final Module module;
    private final DvmClass dvmClass;

    dewuNew() {
        emulator = AndroidEmulatorBuilder
                .for32Bit()
                .setRootDir(new File("unidbg-android/src/test/resources/dewu/rootfs"))
                .addBackendFactory(new Unicorn2Factory(true))
                .build();

        // 模拟器的内存操作接口
        final Memory memory = emulator.getMemory();
        // 设置系统类库解析
        memory.setLibraryResolver(new AndroidResolver(23));
        vm = emulator.createDalvikVM(new File("unidbg-android/src/test/resources/dewu/4.94.apk"));
        vm.setJni(this);
        vm.setVerbose(true);

        emulator.getSyscallHandler().addIOResolver(this);
        emulator.getSyscallHandler().setEnableThreadDispatcher(true);
        emulator.getBackend().registerEmuCountHook(100000); // 设置执行多少条指令切换一次线程

        SystemPropertyHook systemPropertyHook = new SystemPropertyHook(emulator);
        systemPropertyHook.setPropertyProvider(new SystemPropertyProvider() {
            @Override
            public String getProperty(String key) {
                System.out.println("lilac systemkey:" + key);
                switch (key) {
                    case "net.hostname":{
                        return "MIX2S-zhongeryayadeM";
                    }
                    case "ro.serialno":{
                        return "f8a995f5";
                    }
                    case "ro.boot.serialno":{
                        return "f8a995f5";
                    }
                    case "ro.product.brand":{
                        return "Xiaomi";
                    }
                    case "ro.product.manufacturer":{
                        return "Xiaomi";
                    }
                    case "ro.product.model":{
                        return "MIX 2S";
                    }
                    case "ro.product.cpu.abi":{
                        return "arm64-v8a";
                    }
                    case "ro.product.cpu.abilist":{
                        return "arm64-v8a,armeabi-v7a,armeabi";
                    }
                    case "ro.boot.vbmeta.digest":{
                        return null;
                    }
                    case "init.svc.droid4x":{
                        return null;
                    }
                }
                return "";
            }
        });
        memory.addHookListener(systemPropertyHook);

        DalvikModule dm = vm.loadLibrary("szstone", true);

        module = dm.getModule();
        dm.callJNI_OnLoad(emulator);
        dvmClass = vm.resolveClass("com.shizhuang.stone.main.SzSdk");
    }

    public void call_fun() {
        String a1 = "awt0bapt/data/user/0/com.shizhuang.duapp/filesbav4.94.0bavn486bcndewubc";
        int a2 = 1;
        int a3 = 0;
        dvmClass.callStaticJniMethodObject(emulator, "lf(Ljava/lang/String;II)[B", a1, a2, a3);
    }


    public static void main(String[] args) {
        dewuNew dewuNew = new dewuNew();
        dewuNew.call_fun();
    }


    @Override
    public DvmObject<?> callStaticObjectMethod(BaseVM vm, DvmClass dvmClass, String signature, VarArg varArg) {
        switch (signature){
            case "android/app/ActivityThread->currentActivityThread()Landroid/app/ActivityThread;":{
                return dvmClass.newObject(null);
            }
            case "android/provider/Settings$Secure->getString(Landroid/content/ContentResolver;Ljava/lang/String;)Ljava/lang/String;":{
                String name = varArg.getObjectArg(1).getValue().toString();
                switch (name){
                    case "android_id":{
                        return new StringObject(vm, "07916307a127cdf0");
                    }
                }
            }
        }
        return super.callStaticObjectMethod(vm, dvmClass, signature, varArg);
    }

    @Override
    public DvmObject<?> callObjectMethod(BaseVM vm, DvmObject<?> dvmObject, String signature, VarArg varArg) {
        switch (signature){
            case "android/app/ActivityThread->getApplication()Landroid/app/Application;":{
                return vm.resolveClass("android/app/Application").newObject(null);
            }
            case "android/app/Application->getPackageManager()Landroid/content/pm/PackageManager;":{
                return vm.resolveClass("android/content/pm/PackageManager").newObject(null);
            }
            case "android/app/Application->getPackageName()Ljava/lang/String;":{
                return new StringObject(vm, vm.getPackageName());
            }
            case "android/app/Application->getContentResolver()Landroid/content/ContentResolver;":{
                return vm.resolveClass("android/content/ContentResolver").newObject(null);
            }
        }
        return super.callObjectMethod(vm, dvmObject, signature, varArg);
    }


}
```

## 二、clock_gettime

运行代码

```java
[17:55:23 356]  WARN [com.github.unidbg.linux.ARM32SyscallHandler] (ARM32SyscallHandler:532) - handleInterrupt intno=2, NR=263, svcNumber=0x0, PC=RX@0x40117b88[libc.so]0x40b88, LR=RX@0x400f0ef3[libc.so]0x19ef3, syscall=null
java.lang.UnsupportedOperationException: clk_id=2
	at com.github.unidbg.linux.ARM32SyscallHandler.clock_gettime(ARM32SyscallHandler.java:1746)
	at com.github.unidbg.linux.ARM32SyscallHandler.hook(ARM32SyscallHandler.java:409)
	at com.github.unidbg.arm.backend.Unicorn2Backend$11.hook(Unicorn2Backend.java:347)
	at com.github.unidbg.arm.backend.unicorn.Unicorn$NewHook.onInterrupt(Unicorn.java:109)
```

首先对这个报错做解析，它是一个系统调用相关的报错，调用号是 263，根据下面抛出的异常可以看出，这个系统调用是`clock_gettime`。

来到`ARM32SyscallHandler.java:1746`位置看一下

![](https://cdn.nlark.com/yuque/0/2022/png/707476/1668160719601-a87a4a28-0151-4ae9-bf2f-7ad6a5972284.png)

结合报错信息`java.lang.UnsupportedOperationException: clk_id=2`可知，这里就是我们所说的另一种情况，即 Unidbg 对 clock_gettime 做了模拟实现，但是某些参数、某些分支没有做处理。比如这里就是 clk_id 参数为 2 就没有处理。

遇到系统调用相关的报错，第一步肯定是要了解这个系统调用的基本语义。

`clock_gettime`的语义和功能相当丰富，可以获取真实时间，进程时间等多种类型的时间，其函数原型如下。

```java
int clock_gettime(clockid_t __clock, struct timespec* __ts);
```

timespec 结构体

```java
struct timespec {
    long tv_sec; // 秒时间戳
    long tv_nsec; // 余下的纳秒时间戳
};
```

如果想获取毫秒级时间戳，可以像下面这样写。

```java
long long gettime3(){
    struct timespec t = {0, 0};
    clock_gettime(CLOCK_REALTIME, &t);
    return (t.tv_sec*1000) + (t.tv_nsec/1000000);
}
```

参数 1 clockid 即所谓的时钟类型，`CLOCK_REALTIME`是真实时间信息，此时效果和 gettimeofday 一致，但时间精度更高。除了真实时间外，还有许多其他时钟，列举如下。

```java
#define CLOCK_REALTIME 0
#define CLOCK_MONOTONIC 1
#define CLOCK_PROCESS_CPUTIME_ID 2
#define CLOCK_THREAD_CPUTIME_ID 3
#define CLOCK_MONOTONIC_RAW 4
#define CLOCK_REALTIME_COARSE 5
#define CLOCK_MONOTONIC_COARSE 6
#define CLOCK_BOOTTIME 7
#define CLOCK_REALTIME_ALARM 8
#define CLOCK_BOOTTIME_ALARM 9
#define CLOCK_SGI_CYCLE 10
#define CLOCK_TAI 11
```

`CLOCK_MONOTONIC`开头的几个时钟表示从**某个不确定的点**开始计时的时间，这个不确定的点可能是设备启动，也可能是开机后的某个时间，所以叫它不确定的点。

`CLOCK_BOOTTIME`开头的几个时钟表示从设备启动开始计时，即开机时间。

`CLOCK_PROCESS_CPUTIME_ID` 、`CLOCK_THREAD_CPUTIME_ID`表示 CPU 在当前进程/线程上的耗时，相比较进程/线程的真实持续时间，这会是一个很小的数字。

简而言之，时钟可以分为真实时间、CPU 时间、开机时间、从不确定的某个时间点开始计时这四大类。

Unidbg、Qiling、ExAndroidNativeEmu 等模拟器，其实都是对它有所实现，但又没有实现每一个时钟。

`CLOCK_REALTIME`这个时钟，也就是真实时间，来看看各个工具如何实现。

ExAndroidNativeEmu 通过 Python 内置 API 获得了秒级时间戳，然后赋给 tv_sec，tv_nsec 则是置空。

```java
if clk_id == CLOCK_REALTIME:
    # Its time represents seconds and nanoseconds since the Epoch.
    clock_real = calendar.timegm(time.gmtime())

    mu.mem_write(tp_ptr + 0, int(clock_real).to_bytes(self.__ptr_sz, byteorder='little'))
    mu.mem_write(tp_ptr + self.__ptr_sz, int(0).to_bytes(self.__ptr_sz, byteorder='little'))
    return 0
```

这么做不太好，意味着获取返回的真实时间其精度只有秒，和语义有一定偏差。

qiling 则是完全没有管其他类型的时钟，只当成真实时间去处理

```java
def ql_syscall_clock_gettime(ql, clock_gettime_clock_id, clock_gettime_timespec, *args, **kw):
    ql.log.info(f"clock_gettime(clock_id={clock_gettime_clock_id}, tp={hex(clock_gettime_timespec)})")
    now = datetime.now().timestamp()
    tv_sec = floor(now)
    tv_nsec = floor((now - floor(now)) * 1e6)
    tp = timespec(tv_sec= tv_sec, tv_nsec=tv_nsec)
    ql.mem.write(clock_gettime_timespec, bytes(tp))
    ql.log.debug(f"timespec(tv_sec={tv_sec}, tv_nsec={tv_nsec})")
    return 0
```

相比 ExAndroidNativeEmu，它更完善一些，对 tv_nsec 也做了赋值。

Unidbg 在`CLOCK_REALTIME`时钟下，通过 Java 标准库获取了毫秒级时间戳，把秒的部分赋给 tv_sec，再毫秒部分乘 1000000 赋给 tv_nsec。和 Qiling 的逻辑完全一致。

```java
long offset = clk_id == CLOCK_REALTIME ? System.currentTimeMillis() * 1000000L : System.nanoTime() - nanoTime;
```

别忘了还有其他类型的时钟，Qiling 没有做处理，ExAndroidNativeEmu 和 Unidbg 做了部分处理。

第二部分是从不确定的某个点开始计时，包括`CLOCK_MONOTONIC`、`CLOCK_MONOTONIC_RAW`、`CLOCK_MONOTONIC_COARSE` 这几个。

ExAndroidNativeEmu 实际上返回了自身运行的时间，语义上接近“开机时间”。

```java
elif clk_id == CLOCK_MONOTONIC or clk_id == CLOCK_MONOTONIC_COARSE:
    if OVERRIDE_CLOCK:
        mu.mem_write(tp_ptr + 0, int(OVERRIDE_CLOCK_TIME).to_bytes(self.__ptr_sz, byteorder='little'))
        mu.mem_write(tp_ptr + self.__ptr_sz, int(0).to_bytes(self.__ptr_sz, byteorder='little'))
    else:
        clock_add = time.time() - self._clock_start  # Seconds passed since clock_start was set.

        mu.mem_write(tp_ptr + 0, int(self._clock_start + clock_add).to_bytes(self.__ptr_sz, byteorder='little'))
        mu.mem_write(tp_ptr + self.__ptr_sz, int(0).to_bytes(self.__ptr_sz, byteorder='little'))
    return 0
```

Unidbg 处理也类似，返回 Unidbg 自身这个 JAVA 进程所启动的时间。

```java
long offset = clk_id == CLOCK_REALTIME ? System.currentTimeMillis() * 1000000L : System.nanoTime() - nanoTime;
long tv_sec = offset / 1000000000L;
long tv_nsec = offset % 1000000000L;
```

对于开机时间，ExAndroidNativeEmu 没处理，Unidbg 处理上和 CLOCK_MONOTONIC 类似，都表示为Unidbg 的开机时间。

对于`CPU`执行时间，ExAndroidNativeEmu 没有处理，Unidbg 则直接设置为 1 纳秒，也属于摆烂。

```java
case CLOCK_THREAD_CPUTIME_ID:
    tp.setInt(0, 0);
    tp.setInt(4, 1);
    return 0;
```

简而言之，这几款工具都做了一定程度上的模拟，但都不齐全不完善。

我们这里遇到的问题是时钟为 2 时没处理，2 对应于 CLOCK_PROCESS_CPUTIME_ID 即进程的 CPU 时间，这里我们也破罐子破摔，并入 CLOCK_THREAD_CPUTIME_ID。

![](https://cdn.nlark.com/yuque/0/2022/png/707476/1668164742246-42a6a23c-5746-48ca-979d-c74099e35792.png)

你可能会担心这样存在隐患，比如更容易被风控。那么做一定的随机化也是可行的。

读者可能会想到一个问题，这里我们直接在源码里修改，如果下次项目脱离了这个环境，就又得同样修改源码，很不方便，可以像第二篇那样，用自定义的系统调用处理器达到更优雅的效果。

自定义`deWuSyscallHandler`

```java
package com.dewu;

import com.github.unidbg.Emulator;
import com.github.unidbg.arm.backend.Backend;
import com.github.unidbg.linux.ARM32SyscallHandler;
import com.github.unidbg.memory.SvcMemory;
import com.github.unidbg.pointer.UnidbgPointer;
import com.sun.jna.Pointer;
import unicorn.ArmConst;

public class deWuSyscallHandler extends ARM32SyscallHandler {
    public deWuSyscallHandler(SvcMemory svcMemory) {
        super(svcMemory);
    }

    @Override
    protected int clock_gettime(Backend backend, Emulator<?> emulator) {
        int clk_id = backend.reg_read(ArmConst.UC_ARM_REG_R0).intValue();
        Pointer tp = UnidbgPointer.register(emulator, ArmConst.UC_ARM_REG_R1);
        if(clk_id == 2){
            tp.setInt(0, 0);
            tp.setInt(4, 1);
            return 0;
        }
        return super.clock_gettime(backend, emulator);
    }
}
```

在本例中使用它而非默认的 SyscallHandler

```java
package com.dewu;

import com.github.unidbg.AndroidEmulator;
import com.github.unidbg.Emulator;
import com.github.unidbg.Module;
import com.github.unidbg.arm.backend.Unicorn2Factory;
import com.github.unidbg.file.FileResult;
import com.github.unidbg.file.IOResolver;
import com.github.unidbg.file.linux.AndroidFileIO;
import com.github.unidbg.linux.android.*;
import com.github.unidbg.linux.android.dvm.*;
import com.github.unidbg.linux.file.ByteArrayFileIO;
import com.github.unidbg.linux.file.SimpleFileIO;
import com.github.unidbg.memory.Memory;
import com.github.unidbg.memory.SvcMemory;
import com.github.unidbg.unix.UnixSyscallHandler;

import java.io.File;
import java.nio.charset.StandardCharsets;
import java.util.UUID;

public class dewuNew extends AbstractJni implements IOResolver {
    @Override
    public FileResult resolve(Emulator emulator, String pathname, int oflags) {
        System.out.println("file open:"+pathname);
        if(pathname.equals("proc/sys/kernel/random/boot")){
            return FileResult.success(new ByteArrayFileIO(oflags, pathname, UUID.randomUUID().toString().getBytes(StandardCharsets.UTF_8)));
        }
        if(pathname.equals("/proc/cpuinfo")){
            return FileResult.success(new SimpleFileIO(oflags, new File("unidbg-android/src/test/resources/dewu/files/cpuinfo"), pathname));
        }
        if(pathname.equals("/proc/version")){
            return FileResult.success(new SimpleFileIO(oflags, new File("unidbg-android/src/test/resources/dewu/files/version"), pathname));
        }
        return null;
    }

    private final AndroidEmulator emulator;
    private final VM vm;
    private final Module module;
    private final DvmClass dvmClass;

    dewuNew() {
        // 创建模拟器实例
        AndroidEmulatorBuilder builder = new AndroidEmulatorBuilder(false){
            @Override
            public AndroidEmulator build() {
                return new AndroidARMEmulator(processName,rootDir,backendFactories) {
                    @Override
                    protected UnixSyscallHandler<AndroidFileIO> createSyscallHandler(SvcMemory svcMemory) {
                        return new deWuSyscallHandler(svcMemory);
                    }
                };
            }
        };
        emulator = builder
                .setRootDir(new File("unidbg-android/src/test/resources/dewu/rootfs"))
                .addBackendFactory(new Unicorn2Factory(true))
                .build();

        // 模拟器的内存操作接口
        final Memory memory = emulator.getMemory();
        // 设置系统类库解析
        memory.setLibraryResolver(new AndroidResolver(23));
        vm = emulator.createDalvikVM(new File("unidbg-android/src/test/resources/dewu/4.94.apk"));
        vm.setJni(this);
        vm.setVerbose(true);

        emulator.getSyscallHandler().addIOResolver(this);
        emulator.getSyscallHandler().setEnableThreadDispatcher(true);
        emulator.getBackend().registerEmuCountHook(100000); // 设置执行多少条指令切换一次线程

        SystemPropertyHook systemPropertyHook = new SystemPropertyHook(emulator);
        systemPropertyHook.setPropertyProvider(new SystemPropertyProvider() {
            @Override
            public String getProperty(String key) {
                System.out.println("lilac systemkey:" + key);
                switch (key) {
                    case "net.hostname":{
                        return "MIX2S-zhongeryayadeM";
                    }
                    case "ro.serialno":{
                        return "f8a995f5";
                    }
                    case "ro.boot.serialno":{
                        return "f8a995f5";
                    }
                    case "ro.product.brand":{
                        return "Xiaomi";
                    }
                    case "ro.product.manufacturer":{
                        return "Xiaomi";
                    }
                    case "ro.product.model":{
                        return "MIX 2S";
                    }
                    case "ro.product.cpu.abi":{
                        return "arm64-v8a";
                    }
                    case "ro.product.cpu.abilist":{
                        return "arm64-v8a,armeabi-v7a,armeabi";
                    }
                    case "ro.boot.vbmeta.digest":{
                        return null;
                    }
                    case "init.svc.droid4x":{
                        return null;
                    }
                }
                return "";
            }
        });
        memory.addHookListener(systemPropertyHook);

        DalvikModule dm = vm.loadLibrary("szstone", true);

        module = dm.getModule();
        dm.callJNI_OnLoad(emulator);
        dvmClass = vm.resolveClass("com.shizhuang.stone.main.SzSdk");
    }

    public void call_fun() {
        String a1 = "awt0bapt/data/user/0/com.shizhuang.duapp/filesbav4.94.0bavn486bcndewubc";
        int a2 = 1;
        int a3 = 0;
        dvmClass.callStaticJniMethodObject(emulator, "lf(Ljava/lang/String;II)[B", a1, a2, a3);
    }


    public static void main(String[] args) {
        dewuNew dewuNew = new dewuNew();
        dewuNew.call_fun();
    }


    @Override
    public DvmObject<?> callStaticObjectMethod(BaseVM vm, DvmClass dvmClass, String signature, VarArg varArg) {
        switch (signature){
            case "android/app/ActivityThread->currentActivityThread()Landroid/app/ActivityThread;":{
                return dvmClass.newObject(null);
            }
            case "android/provider/Settings$Secure->getString(Landroid/content/ContentResolver;Ljava/lang/String;)Ljava/lang/String;":{
                String name = varArg.getObjectArg(1).getValue().toString();
                switch (name){
                    case "android_id":{
                        return new StringObject(vm, "07916307a127cdf0");
                    }
                }
            }
        }
        return super.callStaticObjectMethod(vm, dvmClass, signature, varArg);
    }

    @Override
    public DvmObject<?> callObjectMethod(BaseVM vm, DvmObject<?> dvmObject, String signature, VarArg varArg) {
        switch (signature){
            case "android/app/ActivityThread->getApplication()Landroid/app/Application;":{
                return vm.resolveClass("android/app/Application").newObject(null);
            }
            case "android/app/Application->getPackageManager()Landroid/content/pm/PackageManager;":{
                return vm.resolveClass("android/content/pm/PackageManager").newObject(null);
            }
            case "android/app/Application->getPackageName()Ljava/lang/String;":{
                return new StringObject(vm, vm.getPackageName());
            }
            case "android/app/Application->getContentResolver()Landroid/content/ContentResolver;":{
                return vm.resolveClass("android/content/ContentResolver").newObject(null);
            }
        }
        return super.callObjectMethod(vm, dvmObject, signature, varArg);
    }


}
```

运行代码会发现新的报错，我们已经越过了这个坑。

这里顺嘴提一句，另一个时间相关的系统调用`gettimeofday`在 Unidbg 里的实现也略有问题。

它的调用原型如下，返回 0 代表调用成功，返回 -1 表示失败。

```java
int gettimeofday(struct timeval *tv, struct timezone *tz);
```

tv 是一个指向 timeval 结构体的指针

```java
struct timeval {
  long tv_sec; 秒时间戳
  long tv_usec; 余下的微秒
};
```

只想获取秒级时间戳时，通常像下面这样写代码。

```java
long gettime(){
    struct timeval t{};
    gettimeofday(&t, nullptr);
    long sec = t.tv_sec;
    return sec;
}
```

想获取毫秒时间戳的时候，则要考虑 tv_usec。

```java
long long gettime2(){
    struct timeval t{};
    gettimeofday(&t, nullptr);
    long sec = t.tv_sec;
    long usec = t.tv_usec;
    return (sec * 1000) + (usec / 1000);
}
```

因为 long 类型在 32/64 位架构上分别占据 4/8 字节，所以 Unidbg 分别实现了 gettimeofday 以及 gettimeofday64，其中的差异仅在于 TimeVal 结构体的表示。

TimeVal32

```java
public class TimeVal32 extends UnidbgStructure {

    public TimeVal32(Pointer p) {
        super(p);
    }

    public int tv_sec;
    public int tv_usec;

    @Override
    protected List<String> getFieldOrder() {
        return Arrays.asList("tv_sec", "tv_usec");
    }

}
```

TimeVal64

```java
public class TimeVal64 extends UnidbgStructure {

    public TimeVal64(Pointer p) {
        super(p);
    }

    public long tv_sec;
    public long tv_usec;

    @Override
    protected List<String> getFieldOrder() {
        return Arrays.asList("tv_sec", "tv_usec");
    }

}
```

再说说 tz ，它是一个指向 timezone 结构体的指针，可用它获取系统的时区信息。Unidbg 在模拟`gettimeofday`时，认为 tv 总是不为空指针，所以只对 tz 做判空处理，具体代码如下。

```java
protected int gettimeofday(Emulator<?> emulator, Pointer tv, Pointer tz) {
    long currentTimeMillis = System.currentTimeMillis();
    long tv_sec = currentTimeMillis / 1000;
    long tv_usec = (currentTimeMillis % 1000) * 1000;
    TimeVal32 timeVal = new TimeVal32(tv);
    timeVal.tv_sec = (int) tv_sec;
    timeVal.tv_usec = (int) tv_usec;
    timeVal.pack();

    if (tz != null) {
        Calendar calendar = Calendar.getInstance();
        int tz_minuteswest = -(calendar.get(Calendar.ZONE_OFFSET) + calendar.get(Calendar.DST_OFFSET)) / (60 * 1000);
        TimeZone timeZone = new TimeZone(tz);
        timeZone.tz_minuteswest = tz_minuteswest;
        timeZone.tz_dsttime = 0;
        timeZone.pack();
    }
    
    return 0;
}
```

这么做是不妥当的，在多个样本上，都可以遇到 tv 为空，专注于使用 tz 获取时区信息的代码实况，如果你也遇到了这种情况，需要对 tv 做一下判空。

```java
@Override
protected int gettimeofday(Emulator<?> emulator, Pointer tv, Pointer tz) {
    long currentTimeMillis = System.currentTimeMillis();
    long tv_sec = currentTimeMillis / 1000;
    long tv_usec = (currentTimeMillis % 1000) * 1000;

    if (tv != null) {
        TimeVal32 timeVal = new TimeVal32(tv);
        timeVal.tv_sec = (int) tv_sec;
        timeVal.tv_usec = (int) tv_usec;
        timeVal.pack();
    }

    if (tz != null) {
        Calendar calendar = Calendar.getInstance();
        int tz_minuteswest = -(calendar.get(Calendar.ZONE_OFFSET) + calendar.get(Calendar.DST_OFFSET)) / (60 * 1000);
        TimeZone timeZone = new TimeZone(tz);
        timeZone.tz_minuteswest = tz_minuteswest;
        timeZone.tz_dsttime = 0;
        timeZone.pack();
    }
    return 0;
}
```

## 三、popen

新的报错看起来相当复杂

```java
[19:11:34 576]  INFO [com.github.unidbg.linux.AndroidSyscallHandler] (AndroidSyscallHandler:480) - Return default pipe pair.
[19:11:34 577]  WARN [com.github.unidbg.linux.ARM32SyscallHandler] (ARM32SyscallHandler:532) - handleInterrupt intno=2, NR=190, svcNumber=0x0, PC=RX@0x40118b5c[libc.so]0x41b5c, LR=RX@0x401065cb[libc.so]0x2f5cb, syscall=null
[19:11:34 578]  WARN [com.github.unidbg.linux.ARM32SyscallHandler] (ARM32SyscallHandler:532) - handleInterrupt intno=2, NR=358, svcNumber=0x0, PC=RX@0x40118db0[libc.so]0x41db0, LR=RX@0x40106635[libc.so]0x2f635, syscall=null
java.lang.AbstractMethodError: com.github.unidbg.linux.file.PipedWriteFileIO
	at com.github.unidbg.file.AbstractFileIO.dup2(AbstractFileIO.java:168)
	at com.github.unidbg.linux.ARM32SyscallHandler.dup3(ARM32SyscallHandler.java:2110)
	at com.github.unidbg.linux.ARM32SyscallHandler.hook(ARM32SyscallHandler.java:499)
```

NR=190 对应于 vfork

![](https://cdn.nlark.com/yuque/0/2022/png/707476/1668165246039-7be448c7-b0fe-43ec-8c27-d0705d2f9951.png)

NR = 358 对应于 dup3

为什么这里有多个系统调用会`WARN`？事实上这是库函数`popen`惹的祸，只要读者看到`com.github.unidbg.linux.file.PipedWriteFileIO`报错，那么就应该直接转入处理`popen`的逻辑。

我们这里要了解三个小知识

1. 基本语义和用法
2. Unidbg 为什么会在这个库函数上出错
3. Unidbg 里如何处理这个问题

事实上，这个问题归到补库函数可能会比较好，但放到补系统调用这里，也并无不妥。

首先，popen 在 Android Native 上是一个高频函数，远非`getrusage`这类函数可比，设备信息收集、环境检测都可以用到它。函数原型如下

```java
FILE *popen(const char *command, const char *type);
```

popen 以创建管道的方式启动一个进程, 并调用 shell，像下面这样使用，比如这里在作用上等价于 adb shell 里输入`getprop ro.build.id`。

```java
std::string cmd = "getprop ro.build.id";
char value[PROP_VALUE_MAX] = {0};
FILE* file = popen(cmd.c_str(), "r");
fread(value, PROP_VALUE_MAX, 1, file);
pclose(file);
```

其次为什么 Unidbg 会在这个库函数上出错，popen 在底层原理上是创建了一个新进程，然后通过管道去做通信，返回数据。但 Unidbg 在多进程相关的系统调用处理上十分不完善，无法支撑`popen`正常的执行它的底层逻辑。

最后是 Unidbg 里如何处理好 popen，作者提供过一种好思路，就是越过创建多线程的相关逻辑，只处理好管道的返回值，来让它的功能基本满足。

下面做演示，这可以当成固定处理模式，遇到了就这么做就行。

第一步是用 Dobby 或任意 Hook 工具，在库函数的层面 Hook 获取到 popen 的 command，放到 emulator 的全局变量里，方便后续形成对应。我这里使用的是 xhook，代码位于 hookPopen 函数内。

```java
package com.dewu;

import com.github.unidbg.AndroidEmulator;
import com.github.unidbg.Emulator;
import com.github.unidbg.Module;
import com.github.unidbg.arm.HookStatus;
import com.github.unidbg.arm.backend.Unicorn2Factory;
import com.github.unidbg.arm.context.RegisterContext;
import com.github.unidbg.file.FileResult;
import com.github.unidbg.file.IOResolver;
import com.github.unidbg.file.linux.AndroidFileIO;
import com.github.unidbg.hook.HookContext;
import com.github.unidbg.hook.ReplaceCallback;
import com.github.unidbg.hook.xhook.IxHook;
import com.github.unidbg.linux.android.*;
import com.github.unidbg.linux.android.dvm.*;
import com.github.unidbg.linux.file.ByteArrayFileIO;
import com.github.unidbg.linux.file.SimpleFileIO;
import com.github.unidbg.memory.Memory;
import com.github.unidbg.memory.SvcMemory;
import com.github.unidbg.unix.UnixSyscallHandler;

import java.io.File;
import java.nio.charset.StandardCharsets;
import java.util.UUID;

public class dewuNew extends AbstractJni implements IOResolver {
    @Override
    public FileResult resolve(Emulator emulator, String pathname, int oflags) {
        System.out.println("file open:"+pathname);
        if(pathname.equals("proc/sys/kernel/random/boot")){
            return FileResult.success(new ByteArrayFileIO(oflags, pathname, UUID.randomUUID().toString().getBytes(StandardCharsets.UTF_8)));
        }
        if(pathname.equals("/proc/cpuinfo")){
            return FileResult.success(new SimpleFileIO(oflags, new File("unidbg-android/src/test/resources/dewu/files/cpuinfo"), pathname));
        }
        if(pathname.equals("/proc/version")){
            return FileResult.success(new SimpleFileIO(oflags, new File("unidbg-android/src/test/resources/dewu/files/version"), pathname));
        }
        return null;
    }

    private final AndroidEmulator emulator;
    private final VM vm;
    private final Module module;
    private final DvmClass dvmClass;

    dewuNew() {
        // 创建模拟器实例
        AndroidEmulatorBuilder builder = new AndroidEmulatorBuilder(false){
            @Override
            public AndroidEmulator build() {
                return new AndroidARMEmulator(processName,rootDir,backendFactories) {
                    @Override
                    protected UnixSyscallHandler<AndroidFileIO> createSyscallHandler(SvcMemory svcMemory) {
                        return new deWuSyscallHandler(svcMemory);
                    }
                };
            }
        };
        emulator = builder
                .setRootDir(new File("unidbg-android/src/test/resources/dewu/rootfs"))
                .addBackendFactory(new Unicorn2Factory(true))
                .build();

        // 模拟器的内存操作接口
        final Memory memory = emulator.getMemory();
        // 设置系统类库解析
        memory.setLibraryResolver(new AndroidResolver(23));
        vm = emulator.createDalvikVM(new File("unidbg-android/src/test/resources/dewu/4.94.apk"));
        vm.setJni(this);
        vm.setVerbose(true);

        emulator.getSyscallHandler().addIOResolver(this);
        emulator.getSyscallHandler().setEnableThreadDispatcher(true);
        emulator.getBackend().registerEmuCountHook(100000); // 设置执行多少条指令切换一次线程

        SystemPropertyHook systemPropertyHook = new SystemPropertyHook(emulator);
        systemPropertyHook.setPropertyProvider(new SystemPropertyProvider() {
            @Override
            public String getProperty(String key) {
                System.out.println("lilac systemkey:" + key);
                switch (key) {
                    case "net.hostname":{
                        return "MIX2S-zhongeryayadeM";
                    }
                    case "ro.serialno":{
                        return "f8a995f5";
                    }
                    case "ro.boot.serialno":{
                        return "f8a995f5";
                    }
                    case "ro.product.brand":{
                        return "Xiaomi";
                    }
                    case "ro.product.manufacturer":{
                        return "Xiaomi";
                    }
                    case "ro.product.model":{
                        return "MIX 2S";
                    }
                    case "ro.product.cpu.abi":{
                        return "arm64-v8a";
                    }
                    case "ro.product.cpu.abilist":{
                        return "arm64-v8a,armeabi-v7a,armeabi";
                    }
                    case "ro.boot.vbmeta.digest":{
                        return null;
                    }
                    case "init.svc.droid4x":{
                        return null;
                    }
                }
                return "";
            }
        });
        memory.addHookListener(systemPropertyHook);

        DalvikModule dm = vm.loadLibrary("szstone", true);

        module = dm.getModule();
        dm.callJNI_OnLoad(emulator);
        dvmClass = vm.resolveClass("com.shizhuang.stone.main.SzSdk");
    }

    public void call_fun() {
        String a1 = "awt0bapt/data/user/0/com.shizhuang.duapp/filesbav4.94.0bavn486bcndewubc";
        int a2 = 1;
        int a3 = 0;
        dvmClass.callStaticJniMethodObject(emulator, "lf(Ljava/lang/String;II)[B", a1, a2, a3);
    }

    public void hookPopen(){
        IxHook xHook = XHookImpl.getInstance(emulator);
        xHook.register("libszstone.so", "popen", new ReplaceCallback() {
            @Override
            public HookStatus onCall(Emulator<?> emulator, HookContext context, long originFunction) {
                RegisterContext registerContext = emulator.getContext();
                String command = registerContext.getPointerArg(0).getString(0);
                emulator.set("command", command);
                System.out.println("command:"+command);
                return HookStatus.RET(emulator, originFunction);
            }
        }, true);
        // 使其生效
        xHook.refresh();
    }


    public static void main(String[] args) {
        dewuNew dewuNew = new dewuNew();
        dewuNew.hookPopen();
        dewuNew.call_fun();
    }


    @Override
    public DvmObject<?> callStaticObjectMethod(BaseVM vm, DvmClass dvmClass, String signature, VarArg varArg) {
        switch (signature){
            case "android/app/ActivityThread->currentActivityThread()Landroid/app/ActivityThread;":{
                return dvmClass.newObject(null);
            }
            case "android/provider/Settings$Secure->getString(Landroid/content/ContentResolver;Ljava/lang/String;)Ljava/lang/String;":{
                String name = varArg.getObjectArg(1).getValue().toString();
                switch (name){
                    case "android_id":{
                        return new StringObject(vm, "07916307a127cdf0");
                    }
                }
            }
        }
        return super.callStaticObjectMethod(vm, dvmClass, signature, varArg);
    }

    @Override
    public DvmObject<?> callObjectMethod(BaseVM vm, DvmObject<?> dvmObject, String signature, VarArg varArg) {
        switch (signature){
            case "android/app/ActivityThread->getApplication()Landroid/app/Application;":{
                return vm.resolveClass("android/app/Application").newObject(null);
            }
            case "android/app/Application->getPackageManager()Landroid/content/pm/PackageManager;":{
                return vm.resolveClass("android/content/pm/PackageManager").newObject(null);
            }
            case "android/app/Application->getPackageName()Ljava/lang/String;":{
                return new StringObject(vm, vm.getPackageName());
            }
            case "android/app/Application->getContentResolver()Landroid/content/ContentResolver;":{
                return vm.resolveClass("android/content/ContentResolver").newObject(null);
            }
        }
        return super.callObjectMethod(vm, dvmObject, signature, varArg);
    }


}
```

运行可以发现 Hook 生效，打印出了具体的命令。

![](https://cdn.nlark.com/yuque/0/2022/png/707476/1668176478389-c5f95873-553b-48e1-b89c-5f316ddf7fdc.png)

可以在 Adb shell 里复现得到结果

```
C:\Users\13352>adb shell
polaris:/ $ stat /data
  File: /data
  Size: 4096     Blocks: 16      IO Blocks: 512 directory
Device: 10305h/66309d    Inode: 2        Links: 53
Access: (0771/drwxrwx--x)       Uid: ( 1000/  system)   Gid: ( 1000/  system)
Access: 2022-04-22 16:08:42.656423789 +0800
Modify: 1970-02-05 00:02:38.459999996 +0800
Change: 1971-12-21 21:33:28.769999994 +0800
```

获取 /data 目录的文件属性信息并不让人感到意外，事实上很多样本都会这么做。文件属性里的 Inode，以及三个时间戳，都是很好的设备标识信息。

如果你不喜欢用 shell 命令，而是希望通过 Hook 获取真实执行流里的“一手数据”，那么可以 Hook popen，[SeeFlowerX](https://gist.github.com/SeeFlowerX) 提供的参考脚本如下。

```java
function hook_libc(){
  let fgets_ptr = Module.findExportByName("libc.so", "fgets");
  let fgets = new NativeFunction(fgets_ptr, "pointer", ["pointer", "int", "pointer"]);
  let popen_addr = Module.findExportByName("libc.so", "popen");
  console.log(`popen_addr => ${popen_addr}`);
  Interceptor.attach(popen_addr, {
    onEnter: function(args){
      let command = args[0].readUtf8String();
      let mode = args[1].readUtf8String();
      console.log(`[popen] [onEnter] command=${command} mode=${mode}`)
    },
    onLeave: function(fp){
      let output = "";
      let buffer = Memory.alloc(1024);
      while (fgets(buffer, 1024, fp) > 0) {
        output += buffer.readUtf8String();
      }
      console.log(`[popen] [onLeave] fp=${fp} output =>${output}<=`);
    }
  })
}
// hook_libc();
```

接下来在系统调用端做处理，重写 pipe2，根据当前的 command 返回从 ADB shell 获取到的值，以及出现 popen 内部会使用的 vfork 和 wait4 系统调用。

```java
package com.dewu;

import com.github.unidbg.Emulator;
import com.github.unidbg.arm.backend.Backend;
import com.github.unidbg.arm.context.EditableArm32RegisterContext;
import com.github.unidbg.linux.ARM32SyscallHandler;
import com.github.unidbg.linux.file.ByteArrayFileIO;
import com.github.unidbg.linux.file.DumpFileIO;
import com.github.unidbg.memory.SvcMemory;
import com.github.unidbg.pointer.UnidbgPointer;
import com.sun.jna.Pointer;
import unicorn.ArmConst;

import java.util.concurrent.ThreadLocalRandom;

public class deWuSyscallHandler extends ARM32SyscallHandler {
    public deWuSyscallHandler(SvcMemory svcMemory) {
        super(svcMemory);
    }

    @Override
    protected boolean handleUnknownSyscall(Emulator<?> emulator, int NR) {
        switch (NR) {
            case 190:
                vfork(emulator);
                return true;
            case 114:
                wait4(emulator);
                return true;
        }
        return super.handleUnknownSyscall(emulator, NR);
    }

    @Override
    protected int clock_gettime(Backend backend, Emulator<?> emulator) {
        int clk_id = backend.reg_read(ArmConst.UC_ARM_REG_R0).intValue();
        Pointer tp = UnidbgPointer.register(emulator, ArmConst.UC_ARM_REG_R1);
        if(clk_id == 2){
            tp.setInt(0, 0);
            tp.setInt(4, 1);
            return 0;
        }
        return super.clock_gettime(backend, emulator);
    }

    private void wait4(Emulator<?> emulator) {
        return;
    }


    private void vfork(Emulator<?> emulator) {
        EditableArm32RegisterContext context = emulator.getContext();
        int childPid = emulator.getPid() +
                ThreadLocalRandom.current().nextInt(256);
        int r0 = childPid;
        System.out.println("vfork pid=" + r0);
        context.setR0(r0);
    }

    @Override
    protected int pipe2(Emulator<?> emulator) {
        EditableArm32RegisterContext context = (EditableArm32RegisterContext)
                emulator.getContext();
        Pointer pipefd = context.getPointerArg(0);
        int write = getMinFd();
        this.fdMap.put(write, new DumpFileIO(write));
        int read = getMinFd();
        String command = emulator.get("command");
        System.out.println("fuck cmd:"+command);
        // stdout中写入popen command 应该返回的结果
        String stdout = "\n";
        if(command.equals("stat /data")){
            stdout = "  File: /data\n" +
                    "  Size: 4096     Blocks: 16      IO Blocks: 512 directory\n" +
                    "Device: 10305h/66309d    Inode: 2        Links: 53\n" +
                    "Access: (0771/drwxrwx--x)       Uid: ( 1000/  system)   Gid: ( 1000/  system)\n" +
                    "Access: 2022-04-22 16:08:42.656423789 +0800\n" +
                    "Modify: 1970-02-05 00:02:38.459999996 +0800\n" +
                    "Change: 1971-07-05 09:54:50.369999990 +0800";
        }
        this.fdMap.put(read, new ByteArrayFileIO(0, "pipe2_read_side", stdout.getBytes()));
        pipefd.setInt(0, read);
        pipefd.setInt(4, write);
        context.setR0(0);
        return 0;
    }
}
```

运行发现一大堆 popen 调用，请读者根据 command 逐一返回合适的值，这里 if 判断改用 switch case 可能更适合。

![](https://cdn.nlark.com/yuque/0/2022/png/707476/1668177170528-ffc57a0e-788e-4062-bd17-e8e5193bca5e.png)

然后继续运行，迎接第三个问题。

## 四、Ioctl

ioctl 也是一个高频问题，但处理它并不麻烦。比如此处运行报错

```java
[22:31:39 557]  WARN [com.github.unidbg.linux.ARM32SyscallHandler] (ARM32SyscallHandler:532) - handleInterrupt intno=2, NR=54, svcNumber=0x0, PC=RX@0x40118aa8[libc.so]0x41aa8, LR=RX@0x4011e7f1[libc.so]0x477f1, syscall=null
java.lang.AbstractMethodError: com.github.unidbg.linux.file.TcpSocket: request=0x8927, argp=0xbffff480
	at com.github.unidbg.file.AbstractFileIO.ioctl(AbstractFileIO.java:64)
	at com.github.unidbg.linux.ARM32SyscallHandler.ioctl(ARM32SyscallHandler.java:2031)
	at com.github.unidbg.linux.ARM32SyscallHandler.hook(ARM32SyscallHandler.java:208)
	at com.github.unidbg.arm.backend.Unicorn2Backend$11.hook(Unicorn2Backend.java:347)
```

不熟悉 ioctl 的读者可以看[这篇](https://segmentfault.com/a/1190000005138358)文章，在 Android Native 里，它常用来获取 MAC 地址。

它的参数 2 `request`用于指令操作类型，对于各种各样的文件和需求，有不同的`request`，Unidbg 首先了其中的一部分，但这里报错的 0x8927 尚未处理。

一个最简单的处理办法就是来到报错点`com.github.unidbg.file.AbstractFileIO.ioctl(AbstractFileIO.java:64)`，将抛出异常改为直接返回 -1，即函数调用失败之意。摆烂可耻但有用。

```java
@Override
public int ioctl(Emulator<?> emulator, long request, long argp) {
    if (log.isTraceEnabled()) {
        emulator.attach().debug();
    }
    return -1;
}
```



# unidbg补系统调用(stat)
## 一、引言

本文继续讨论系统调用的处理情况。本篇样例是 [📎快手_10.3.40.25268.apk.7z](https://www.yuque.com/attachments/yuque/0/2022/7z/707476/1668259027091-e9cb2a07-e478-43b3-bcc2-4c25250f1229.7z)

## 二、问题分析

JADX 反编译目标 APK，找到`com.kuaishou.dfp.envdetect.jni.Watermelon`类，其中的`jniCommand`就是目标函数，它位于 libksse.so。

```java
package com.kuaishou.dfp.envdetect.jni;

import android.content.Context;
import android.util.Base64;
import com.getkeepsafe.relinker.d;
import com.kwai.robust.PatchProxy;
import com.kwai.robust.PatchProxyResult;
import nz.s;
import qz.o;

/* compiled from: kSourceFile */
/* loaded from: classes14.dex */
public class Watermelon {
    public static boolean sLibLoadFail = true;
    public static volatile Watermelon singleton;

    public native Object jniCommand(int i6, Object obj, Object obj2, Object obj3);

    public native byte[] ssec(byte[] bArr, byte[] bArr2, Object obj);

    public Watermelon() {
        loadSoLib();
    }

    public static Watermelon getInstance() {
        Object apply = PatchProxy.apply(null, null, Watermelon.class, "1");
        if (apply != PatchProxyResult.class) {
            return (Watermelon) apply;
        }
        if (singleton == null) {
            synchronized (Watermelon.class) {
                if (singleton == null) {
                    singleton = new Watermelon();
                }
            }
        }
        return singleton;
    }

    public final void loadSoLib() {
        if (PatchProxy.applyVoid(null, this, Watermelon.class, "2")) {
            return;
        }
        try {
            String str = new String(Base64.decode("a3NzZQ==", 0));
            Context n7 = s.e().n();
            if (n7 != null) {
                d.a(n7, str);
            } else {
                System.loadLibrary(str);
            }
            sLibLoadFail = false;
            o.c("so loaded");
        } catch (Throwable unused) {
            o.c("so load failed");
            sLibLoadFail = true;
        }
    }
}
```

它是一个实例方法，我们这里只关注通过它获取 mInodes 的逻辑。

![](https://cdn.nlark.com/yuque/0/2022/png/707476/1668261308348-f86e47b6-64b4-4b20-9bce-517f9b24c735.png)

Frida Call 代码如下

```java
function call(){
    Java.perform(function() {
        let Watermelon = Java.use("com.kuaishou.dfp.envdetect.jni.Watermelon").getInstance();
        let ret = Watermelon["jniCommand"](1114128, null, null, null);
        console.log("result:"+ret)
    })
}
```

运行结果

```java
[MIX 2S::com.smile.gifmaker]-> call()
result:687324639::1394160561|987324656::2294160561|587324657::2394160561|9873246::3294160561|987324656::2294160561
```

接下来在 Unidbg 里复现这一函数的调用，先是初始化代码。

```java
package com.ks;

import com.github.unidbg.AndroidEmulator;
import com.github.unidbg.Emulator;
import com.github.unidbg.arm.backend.Unicorn2Factory;
import com.github.unidbg.file.FileResult;
import com.github.unidbg.file.IOResolver;
import com.github.unidbg.linux.android.AndroidEmulatorBuilder;
import com.github.unidbg.linux.android.AndroidResolver;
import com.github.unidbg.linux.android.dvm.AbstractJni;
import com.github.unidbg.linux.android.dvm.DalvikModule;
import com.github.unidbg.linux.android.dvm.DvmClass;
import com.github.unidbg.linux.android.dvm.VM;
import com.github.unidbg.memory.Memory;

import java.io.File;

public class kuaishou extends AbstractJni implements IOResolver {
    private final AndroidEmulator emulator;
    private final DvmClass Watermelon;
    private final VM vm;

    public kuaishou() {
        emulator = AndroidEmulatorBuilder
            .for32Bit()
            .addBackendFactory(new Unicorn2Factory(true))
            .setProcessName("com.smile.gifmaker")
            .build();

        Memory memory = emulator.getMemory();
        memory.setLibraryResolver(new AndroidResolver(23));
        vm = emulator.createDalvikVM(new File("unidbg-android/src/test/resources/ks/快手_10.3.40.25268.apk"));
        vm.setJni(this);
        vm.setVerbose(true);
        emulator.getSyscallHandler().addIOResolver(this);

        DalvikModule dm = vm.loadLibrary("ksse", true);
        Watermelon = vm.resolveClass("com.kuaishou.dfp.envdetect.jni.Watermelon");
        dm.callJNI_OnLoad(emulator);
    }


    public static void main(String[] args) {
        kuaishou ks = new kuaishou();
    }

    @Override
    public FileResult resolve(Emulator emulator, String pathname, int oflags) {
        System.out.println("file open:"+pathname);
        return null;
    }
}
```

运行效果不错

```java
file open:/dev/__properties__
file open:/proc/stat
JNIEnv->FindClass(com/kuaishou/dfp/envdetect/jni/Watermelon) was called from RX@0x40005a99[libksse.so]0x5a99
JNIEnv->RegisterNatives(com/kuaishou/dfp/envdetect/jni/Watermelon, unidbg@0xbffff628, 2) was called from RX@0x40005c51[libksse.so]0x5c51
RegisterNative(com/kuaishou/dfp/envdetect/jni/Watermelon, jniCommand(ILjava/lang/Object;Ljava/lang/Object;Ljava/lang/Object;)Ljava/lang/Object;, RX@0x40004951[libksse.so]0x4951)
RegisterNative(com/kuaishou/dfp/envdetect/jni/Watermelon, ssec([B[BLjava/lang/Object;)[B, RX@0x40005271[libksse.so]0x5271)
```

接下来调用 jniCommand

```java
package com.ks;

import com.github.unidbg.AndroidEmulator;
import com.github.unidbg.Emulator;
import com.github.unidbg.arm.backend.Unicorn2Factory;
import com.github.unidbg.file.FileResult;
import com.github.unidbg.file.IOResolver;
import com.github.unidbg.linux.android.AndroidEmulatorBuilder;
import com.github.unidbg.linux.android.AndroidResolver;
import com.github.unidbg.linux.android.dvm.AbstractJni;
import com.github.unidbg.linux.android.dvm.DalvikModule;
import com.github.unidbg.linux.android.dvm.DvmClass;
import com.github.unidbg.linux.android.dvm.VM;
import com.github.unidbg.memory.Memory;

import java.io.File;

public class kuaishou extends AbstractJni implements IOResolver {
    private final AndroidEmulator emulator;
    private final DvmClass Watermelon;
    private final VM vm;

    public kuaishou() {
        emulator = AndroidEmulatorBuilder
                .for32Bit()
                .addBackendFactory(new Unicorn2Factory(true))
                .setProcessName("com.smile.gifmaker")
                .build();

        Memory memory = emulator.getMemory();
        memory.setLibraryResolver(new AndroidResolver(23));
        vm = emulator.createDalvikVM(new File("unidbg-android/src/test/resources/ks/快手_10.3.40.25268.apk"));
        vm.setJni(this);
        vm.setVerbose(true);
        emulator.getSyscallHandler().addIOResolver(this);

        DalvikModule dm = vm.loadLibrary("ksse", true);
        Watermelon = vm.resolveClass("com.kuaishou.dfp.envdetect.jni.Watermelon");
        dm.callJNI_OnLoad(emulator);
    }

    public String call(){
        return Watermelon.newObject(null).callJniMethodObject(emulator, "jniCommand(ILjava/lang/Object;Ljava/lang/Object;Ljava/lang/Object;)Ljava/lang/Object;", 1114128, null, null, null).getValue().toString();
    }

    public static void main(String[] args) {
        kuaishou ks = new kuaishou();
        System.out.println("ret:"+ks.call());
    }

    @Override
    public FileResult resolve(Emulator emulator, String pathname, int oflags) {
        System.out.println("file open:"+pathname);
        return null;
    }
}
```

看起来不太妙

```java
Find native function Java_com_kuaishou_dfp_envdetect_jni_Watermelon_jniCommand => RX@0x40004951[libksse.so]0x4951
file open:/data/system
[22:28:42 815]  INFO [com.github.unidbg.linux.ARM32SyscallHandler] (ARM32SyscallHandler:1120) - stat64 pathname=/data/system, LR=RX@0x4001990d[libksse.so]0x1990d
file open:/data/data/
[22:28:42 819]  INFO [com.github.unidbg.linux.ARM32SyscallHandler] (ARM32SyscallHandler:1120) - stat64 pathname=/data/data/, LR=RX@0x4001990d[libksse.so]0x1990d
file open:/data/data/com.android.shell
[22:28:42 823]  INFO [com.github.unidbg.linux.ARM32SyscallHandler] (ARM32SyscallHandler:1120) - stat64 pathname=/data/data/com.android.shell, LR=RX@0x4001990d[libksse.so]0x1990d
file open:/data/system/install_sessions
[22:28:42 823]  INFO [com.github.unidbg.linux.ARM32SyscallHandler] (ARM32SyscallHandler:1120) - stat64 pathname=/data/system/install_sessions, LR=RX@0x4001990d[libksse.so]0x1990d
file open:/data/data/com.google.android.webview
[22:28:42 823]  INFO [com.github.unidbg.linux.ARM32SyscallHandler] (ARM32SyscallHandler:1120) - stat64 pathname=/data/data/com.google.android.webview, LR=RX@0x4001990d[libksse.so]0x1990d
JNIEnv->FindClass(java/lang/String) was called from RX@0x40037b87[libksse.so]0x37b87
JNIEnv->GetMethodID(java/lang/String.<init>([BLjava/lang/String;)V) => 0x782c535e was called from RX@0x40037b9d[libksse.so]0x37b9d
JNIEnv->NewByteArray(19) was called from RX@0x40037bb3[libksse.so]0x37bb3
JNIEnv->SetByteArrayRegion([B@4de5031f, 0, 19, RW@0x40223040) was called from RX@0x40037bc7[libksse.so]0x37bc7
JNIEnv->NewStringUTF("utf-8") was called from RX@0x40037bd5[libksse.so]0x37bd5
JNIEnv->NewObjectV(class java/lang/String, <init>([B@4de5031f, "utf-8") => "nnn|nnn|nnn|nnn|nnn") was called from RX@0x40018283[libksse.so]0x18283
ret:nnn|nnn|nnn|nnn|nnn
```

返回的结果，同样是通过竖杠分割的五部分，但每部分都是`nnn`。首先观察文件访问，存在对五个文件夹的访问，具体库函数是`stat64`。

它是用于获取文件属性的系统调用

```java
int stat(const char *path, struct stat *buf);
```

每个文件都有各种各样的属性，包括下面这些。

```java
struct stat {
    dev_t     st_dev;     /* ID of device containing file */
    ino_t     st_ino;     /* inode number */
    mode_t    st_mode;    /* protection */
    nlink_t   st_nlink;   /* number of hard links */
    uid_t     st_uid;     /* user ID of owner */
    gid_t     st_gid;     /* group ID of owner */
    dev_t     st_rdev;    /* device ID (if special file) */
    off_t     st_size;    /* total size, in bytes */
    blksize_t st_blksize; /* blocksize for file system I/O */
    blkcnt_t  st_blocks;  /* number of 512B blocks allocated */
    time_t    st_atime;   /* time of last access */
    time_t    st_mtime;   /* time of last modification */
    time_t    st_ctime;   /* time of last status change */
};
```

这些文件属性可用作信息收集、设备标识等用途。我这里选择首先补一个虚拟目录。

```java
emulator = AndroidEmulatorBuilder
        .for32Bit()
        .addBackendFactory(new Unicorn2Factory(true))
        .setRootDir(new File("unidbg-android/src/test/resources/ks/rootfs"))
        .setProcessName("com.smile.gifmaker")
        .build();
```

然后根据样本的访问，创建一批空文件夹，看会不会有改善。

![](https://cdn.nlark.com/yuque/0/2022/png/707476/1668263942289-b7f2527e-cde6-4c7f-8c4b-2f72ad69a2b0.png)

运行测试

```java
file open:/data/system
file open:/data/data/
file open:/data/data/com.android.shell
file open:/data/system/install_sessions
file open:/data/data/com.google.android.webview
JNIEnv->FindClass(java/lang/String) was called from RX@0x40037b87[libksse.so]0x37b87
JNIEnv->GetMethodID(java/lang/String.<init>([BLjava/lang/String;)V) => 0x782c535e was called from RX@0x40037b9d[libksse.so]0x37b9d
JNIEnv->NewByteArray(24) was called from RX@0x40037bb3[libksse.so]0x37bb3
JNIEnv->SetByteArrayRegion([B@5d47c63f, 0, 24, RW@0x40223040) was called from RX@0x40037bc7[libksse.so]0x37bc7
JNIEnv->NewStringUTF("utf-8") was called from RX@0x40037bd5[libksse.so]0x37bd5
JNIEnv->NewObjectV(class java/lang/String, <init>([B@5d47c63f, "utf-8") => "0::0|0::0|0::0|0::0|0::0") was called from RX@0x40018283[libksse.so]0x18283
ret:0::0|0::0|0::0|0::0|0::0
```

这里读者应该意识到一些问题。

首先，我们在虚拟目录里处理了 stat64 指向的这些文件，确实有效，结果变成了和真机上更接近的格式，不再是`nnn`。

其次，接近了，但又没完全接近，对照一下看看。

```java
result:687324639::1394160561|987324656::2294160561|587324657::2394160561|9873246::3294160561|987324656::2294160561
0::0|0::0|0::0|0::0|0::0
```

每一个具体的内容上，我们的结果都是 0。

这是不合理的，虽然我们补的是空文件夹，但它同样有自己的文件属性，怎么会返回全 0 呢？

最后是关于这到底是什么属性，其实眼尖的话可以发现端倪。

样本访问了五个文件的属性，函数返回的结果也是五部分，而且格式类似，所以基本可以确定就是从每个文件获取了一个属性。以第一部分为例，将`687324639::1394160561`逆序反转，就是 1650614931936423786，一个纳秒级的时间戳。

![](https://cdn.nlark.com/yuque/0/2022/png/707476/1668267277758-23d8814c-b4e5-428b-a05f-891b84214538.png)

在 ADB 中查看这几个文件夹的属性，可以发现和每一个文件的 Access 对的上，语义上即所谓的“最后一次访问的时间”。

```java
polaris:/ $ stat /data/data/com.google.android.webview
  File: /data/data/com.google.android.webview
  Size: 4096     Blocks: 16      IO Blocks: 512 directory
Device: 10305h/66309d    Inode: 1024007  Links: 7
Access: (0700/drwx------)       Uid: (10176/ u0_a176)   Gid: (10176/ u0_a176)
Access: 2022-04-22 16:08:51.936423786 +0800
Modify: 2022-11-09 10:15:38.954950840 +0800
Change: 2022-11-09 10:15:38.954950840 +0800
```

那么问题来了，既然样本的处理逻辑是获取这五个文件夹的 Access 时间，Unidbg 为什么会返回 0？

事实上，这就是 Unidbg 在文件属性这方面的模拟处理上做的不够好。

对于普通文件，Unidbg 的文件属性编码如下。

```java
@Override
public int fstat(Emulator<?> emulator, StatStructure stat) {
    int st_mode;
    if (IO.STDOUT.equals(file.getName())) {
        st_mode = IO.S_IFCHR | 0x777;
    } else if(Files.isSymbolicLink(file.toPath())) {
        st_mode = IO.S_IFLNK;
    } else {
        st_mode = IO.S_IFREG;
    }
    stat.st_dev = 1;
    stat.st_mode = st_mode;
    stat.st_uid = 0;
    stat.st_gid = 0;
    stat.st_size = file.length();
    stat.st_blksize = emulator.getPageAlign();
    stat.st_ino = 1;
    stat.st_blocks = ((file.length() + emulator.getPageAlign() - 1) / emulator.getPageAlign());
    stat.setLastModification(file.lastModified());
    stat.pack();
    return 0;
}
```

看起来还不错，对时间做了处理，但 inode、dev 这些没法看，比如 inode 竟然硬编码为 1。

对于文件夹或者说目录的处理，可以说几乎没处理，可见`src/main/java/com/github/unidbg/linux/file/DirectoryFileIO.java`。

```java
@Override
public int fstat(Emulator<?> emulator, StatStructure stat) {
    stat.st_mode = IO.S_IFDIR;
    stat.st_dev = 0;
    stat.st_size = 0;
    stat.st_blksize = 0;
    stat.st_ino = 0;
    stat.pack();
    return 0;
}
```

除了文件类型设置为目录外，其余全部置空，这就是返回`0::0|0::0|0::0|0::0|0::0`的原因。

这里可以做验证，胡乱设置一下 Access 时间。

```java
public int fstat(Emulator<?> emulator, StatStructure stat) {
    stat.st_mode = IO.S_IFDIR;
    stat.st_dev = 0;
    stat.st_size = 0;
    stat.st_blksize = 0;
    stat.st_ino = 0;
    // test
    stat.setSt_atim(12345678, 9999);
    stat.pack();
    return 0;
}
```

重新运行

```java
ret:999900876::54321|999900876::54321|999900876::54321|999900876::54321|999900876::54321
```

有读者可能会困惑，为什么 Unidbg 不处理的更好一些？这里大体上有两个思路

1 是我们会将对应文件从真机拷贝到我们的宿主机，也就是电脑环境里，那么可以使用这个文件在当前电脑环境里的文件属性信息，比如 ExAndroidNativeEmu 就这么干，Unidbg 对普通文件的时间属性上也是这么处理的。但总体上这个方法并不特别好，文件在迁移的过程中，属性等信息都变化和失真了。

2 是遇到文件属性处理，直接断点断下来，交给用户处理。使用者可以通过 adb shell 或 Hook 等方式从真机获取这些信息，然后做填充；或者不做填充，单纯置空，反正都交给用户自己处理。这个方案的缺点是麻烦。

因为两个方案都不算完善，更称不上完美，相当无奈。

## 三、尾声

如果在使用 Unidbg 处理样本时，发现样本访问文件属性，自己要长个心眼，分析它的访问意图。




# unidbg补系统调用-getcpu处理问题
## 一、引言

第四篇所讨论的情况，在许多其他系统调用上同样存在，即很多系统调用的模拟不够完善，它不会报错，但可能会带来麻烦。

## 二、问题分析

以 getcpu 这个系统调用为例，它的函数原型如下。

```java
int getcpu(unsigned *cpu, unsigned *node, struct getcpu_cache *tcache);
```

在语义上，它用于获取当前线程或进程正在运行的处理器核心和节点，并将它们写入由 cpu 和 node 参数指向的整数，第三个参数一般不使用，传入 Null 即可。

调用成功时返回 0，出错时返回 -1，并会设置对应的 errno 号。

libc 中基于这个系统调用封装了 sched_getcpu 函数，返回当前是第一个 CPU 核心。

```java
#define _GNU_SOURCE 1
#include <sched.h>

extern "C" int __getcpu(unsigned*, unsigned*, void*);

int sched_getcpu() {
    unsigned cpu;
    int rc = __getcpu(&cpu, nullptr, nullptr);
    if (rc == -1) {
        return -1; // errno is already set.
    }
    return cpu;
}
```

读者可以写一个 demo 试试，隔开一定时间，多次调用 getCPU 函数

```java
void getCPU(){
    LOGI("cpuinfo:%d", sched_getcpu());
}
```

我的测试机是八核处理器，操作系统每次随机分配一个处理器核心来对付我们这个程序，所以会返回 0 - 7 中的一个数字，对应于 8 个处理器单元中的一个 。

运行如下

```java
cpuinfo:7
cpuinfo:4
cpuinfo:5
cpuinfo:2
cpuinfo:2
cpuinfo:1
cpuinfo:5
cpuinfo:0
cpuinfo:2
cpuinfo:2
cpuinfo:0
```

再看看 Unidbg 的实现，它总是返回 0，语义上对应于总是返回第一个 CPU 单元。

```java
private int getcpu(Emulator<?> emulator) {
    Arm32RegisterContext context = emulator.getContext();
    Pointer cpu = context.getR0Pointer();
    Pointer node = context.getR1Pointer();
    Pointer tcache = context.getR2Pointer();
    if (log.isDebugEnabled()) {
        log.debug("getcpu cpu=" + cpu + ", node=" + node + ", tcache=" + tcache);
    }
    if (cpu != null) {
        cpu.setInt(0, 0);
    }
    if (node != null) {
        node.setInt(0, 0);
    }
    return 0;
}
```

这么模拟和真实情况显然有差异，如果样本据此检测设备环境，确实可以形成一定区分。

## 三、尾声

除此之外还有大量的系统调用模拟存在类似情况，我们只能期待样本不在这些地方耍花招，否则可谓是防不胜防。




