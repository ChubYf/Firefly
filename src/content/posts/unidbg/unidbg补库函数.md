---
title: unidbg补库函数
published: 2026-10-10
description: unidbg补库函数
image: ./cover.jpg
tags: [逆向]
category: unidbg
draft: false
---


[unidbg补库函数一](https://www.yuque.com/docs/share/e5d6dc74-031e-4273-9019-aa2a57ca38c6?#%20%E3%80%8A%E8%A1%A5%E5%BA%93%E5%87%BD%E6%95%B0%EF%BC%88%E4%B8%80%EF%BC%89%E3%80%8B  )
[unidbg补库函数二](https://www.yuque.com/docs/share/93dbe014-5f5b-4a5f-b9ae-76ce3004bbb9?#%20%E3%80%8A%E8%A1%A5%E5%BA%93%E5%87%BD%E6%95%B0%EF%BC%88%E4%BA%8C%EF%BC%89%E3%80%8B  )
[unidbg补库函数三](https://www.yuque.com/docs/share/40f8790b-c3d6-451d-93de-96a61877cbc3?#%20%E3%80%8A%E8%A1%A5%E5%BA%93%E5%87%BD%E6%95%B0%EF%BC%88%E4%B8%89%EF%BC%89%E3%80%8B  )
[unidbg补库函数四](https://www.yuque.com/docs/share/78bff5bb-f30d-41bb-b31a-59d7cbf28980?#%20%E3%80%8A%E8%A1%A5%E5%BA%93%E5%87%BD%E6%95%B0%EF%BC%88%E5%9B%9B%EF%BC%89%E3%80%8B  )
[unidbg补库函数五](https://www.yuque.com/docs/share/4db5d3a2-5270-460d-b39a-b7d0fcc46990?#%20%E3%80%8A%E8%A1%A5%E5%BA%93%E5%87%BD%E6%95%B0%EF%BC%88%E4%BA%94%EF%BC%89%E3%80%8B  )




# unidbg补库函数(种类分析)
## 一、引言

前文讨论了补系统调用，这里讨论补库函数，它其实是挺重要的一部分，虽然我们经常感知不到它。

## 二、库函数种类

既然要讨论补库函数这个话题，那就要先好好聊一下库函数的概念和范畴。

在 Android Native 代码开发中，会使用各种各样的库函数，一部分依赖于底层系统调用，一部分依赖于系统环境，一部分依赖于其他库，另一部分什么都不依赖，只是单纯的工具函数，

### 第一类: 依赖于系统调用
依赖于系统调用
以 openat 为代表的各类文件访问函数都依赖于系统调用
```c
.text:0004147C                 EXPORT __openat
.text:0004147C __openat                                ; CODE XREF: j___openat+8↑j
.text:0004147C                                         ; DATA XREF: LOAD:00002670↑o ...
.text:0004147C                 MOV             R12, R7
.text:00041480                 LDR             R7, =0x142
.text:00041484                 SVC             0
.text:00041488                 MOV             R7, R12
.text:0004148C                 CMN             R0, #0x1000
.text:00041490                 BXLS            LR
.text:00041494                 RSB             R0, R0, #0
.text:00041498                 B               j___set_errno_internal
```

对于这一类库函数，它的执行逻辑是程序调用库函数，库函数调用操作系统的系统调用，进而实现具体功能。

### 第二类：依赖于系统环境和上下文

比如在 Android Native 上高频使用的`__system_property_get`函数，用于获取系统属性，函数原型如下

```c
/* Look up a system property by name, copying its value and a
** \0 terminator to the provided pointer.  The total bytes
** copied will be no greater than PROP_VALUE_MAX.  Returns
** the string length of the value.  A property that is not
** defined is identical to a property with a length 0 value.
*/
int __system_property_get(const char *name, char *value);
```

基本用例如下。

```c
#include <jni.h>
#include <string>
#include <sys/system_properties.h>
#include <stdexcept>
#include <android/log.h>


#define LOGE(...) __android_log_print(ANDROID_LOG_ERROR,"Lilac" ,__VA_ARGS__)
extern "C" JNIEXPORT jstring JNICALL
Java_com_example_getprop_MainActivity_stringFromJNI(
        JNIEnv* env,
        jobject /* this */) {
    char temp[128] = "0";
    __system_property_get("ro.build.id", temp);
    LOGE("id:%s",temp);
    __system_property_get("ro.build.version.sdk", temp);
    LOGE("version:%s",temp);
    __system_property_get("ro.board.platform", temp);
    LOGE("platform:%s",temp);
    __system_property_get("ro.product.cpu.abi", temp);
    LOGE("abi2:%s",temp);
    __system_property_get("ro.product.model", temp);
    LOGE("model:%s",temp);
    __system_property_get("ro.product.brand", temp);
    LOGE("brand:%s",temp);
    __system_property_get("ro.product.manufacturer", temp);
    LOGE("manufacturer:%s",temp);
    return env->NewStringUTF("");
}
```

这些属性信息的获取，在很大程度上来自于对`/dev/__properties__`文件的解析。

比如`getenv`函数，它用于获取环境变量，具体而言，它搜索`name`所指向的环境字符串，并返回相关的值给字符串。

```c
char *getenv(const char *name)
```

它和`setenv`用于对进程的环境变量做读和取的操作，虽然底层不依赖于系统调用，但它同样十分重要。

### 第三类: 依赖其他库函数或底层处理。

比如 libandroid.so 这个库，就依赖于相当多的其他的模块，提供了大量 Android 资源的访问功能。

第四类不必说，几乎不用考虑它，没什么事。

## 三、问题分析

从库函数的定义和外延看，我们补库函数的场景主要有几类

1. 库函数底层依赖于系统调用，那么如果 Unidbg 对某个库函数其依赖的系统调用的模拟存在问题，那么既可以在系统调用的层面做处理，也可以退而求其次，在库函数的层面做处理，有时候后者还可以获得更好的灵活性。
2. 库函数的依赖太复杂，没法在 Unidbg 里加载进来，典型的 SO 就是 libandroid.so，那么其中的库函数如何处理？
3. 库函数依赖于整个环境，我们希望可以更好，更灵活的控制它们。

## 四、尾声

不同类型的问题有不同的解法，下篇开始做详细讨论。




# unidbg补库函数(系统调用类)

## 一、引言

本篇讨论系统调用导致的库函数问题。

## 二、问题分析

其实在补系统调用时，我们使用过这些技巧。

比如补系统调用第二篇所讨论的`getrusage`库函数，在那篇里我们通过对其底层`getrusage`硬编码予以处理。但因为它是通过库函数，而非内联汇编直接做系统调用，所以在库函数的层面做 Hook 也可行。下面的代码里，使用 Unidbg 内置的 xHook 对`getrusage`库函数做 Patch 处理，看起来也不错，而且免于自定义系统调用之苦。

```java
package com.example;

import com.github.unidbg.AndroidEmulator;
import com.github.unidbg.Emulator;
import com.github.unidbg.arm.HookStatus;
import com.github.unidbg.arm.backend.Unicorn2Factory;
import com.github.unidbg.hook.ReplaceCallback;
import com.github.unidbg.hook.xhook.IxHook;
import com.github.unidbg.linux.android.AndroidEmulatorBuilder;
import com.github.unidbg.linux.android.AndroidResolver;
import com.github.unidbg.linux.android.XHookImpl;
import com.github.unidbg.linux.android.dvm.AbstractJni;
import com.github.unidbg.linux.android.dvm.DalvikModule;
import com.github.unidbg.linux.android.dvm.DvmClass;
import com.github.unidbg.linux.android.dvm.VM;
import com.github.unidbg.memory.Memory;
import com.github.unidbg.pointer.UnidbgPointer;
import com.sun.jna.Pointer;
import unicorn.Arm64Const;

import java.io.File;

public class Demo extends AbstractJni {
    private final AndroidEmulator emulator;
    private final DvmClass MainActivity;
    private final VM vm;

    public Demo() {
        // 创建模拟器实例
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

    public void hookGetRusage(){
        IxHook xHook = XHookImpl.getInstance(emulator);
        xHook.register("libdemo.so", "getrusage", new ReplaceCallback() {
            @Override
            public HookStatus onCall(Emulator<?> emulator, long originFunction) {
                Pointer rusage = UnidbgPointer.register(emulator, Arm64Const.UC_ARM64_REG_X1);
                byte[] rusageContent = hexStringToByteArray("00000000000000009f4a0b00000000000000000000000000c5e10100000000009052010000000000000000000000000000000000000000000000000000000000255e00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000d02000000000000d300000000000000");
                for(int i = 0;i< rusageContent.length;i++){
                    rusage.setByte(i, rusageContent[i]);
                }
                return HookStatus.LR(emulator, 0);
            }
        });

        xHook.refresh();
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


    public String call(){
        return MainActivity.newObject(null).callJniMethodObject(emulator, "stringFromJNI").getValue().toString();
    }

    public static void main(String[] args) {
        Demo demo = new Demo();
        demo.hookGetRusage();
        System.out.println("ret:"+demo.call());
    }

}
```

比如补系统调用第三篇，对 ioctl 的处理也可以通过 Hook 搞定，上面使用的是 xHook，这里换一个 HookZz 试试，都是可行的方案。

```java
public void hookioctl(){
    //获取 HookZz 实例
    HookZz hookZz = HookZz.getInstance(emulator);
    //创建 replace 回调
    ReplaceCallback replaceCallback = new ReplaceCallback() {
        @Override
        public HookStatus onCall(Emulator<?> emulator, HookContext context, long originFunction) {
            return HookStatus.LR(emulator, -1);
        }
    };
    hookZz.replace(module.findSymbolByName("ioctl"), replaceCallback);
}
```

第三篇里讨论到的 popen 函数，同样可以转换成补库函数的问题，我们看一下它的函数原型。

```java
FILE *popen(const char *command, const char *type);
```

它返回的是一个 FILE 对象，那么我们直接 Hook popen 函数，返回一个文件内容为 command 应该读取到的 FILE 对象不就行了？

```java
FILE *fopen(const char *filename, const char *mode)
```

可以简单理解为把 popen 重定向到 fopen 处理，这么做确实可行，但逻辑上有些怪，这里不做演示。

除了 ioctl 外，bind 函数也常需要 patch 处理，原因同理。

## 三、尾声

具体方法论如下

1. 发现 Unidbg 报错，且原因是 Unidbg 没有处理好系统调用。
2. 通过栈回溯确认样本是通过库函数访问发起了调用，还是内联汇编调用
3. 如果是内联汇编发起系统调用，建议作为补系统调用去处理
4. 如果是库函数调用进一步引发系统调用问题，那么从库函数还是系统调用的层面做处理，既看个人喜好，也看哪一个层级更方便。




# unidbg补库函数(system_property_get处理)
## 一、引言

本篇讨论非系统调用出错，但我们仍然需要处理库函数的场景。

## 二、问题分析

首篇讨论过的`__system_property_get`是一个主要问题。假设样本通过这个 API 获取`ro.build.id`，如果不做任何处理，那么 Unidbg 会返回`MDA89E`。

这些值位于`src/main/resources/android/sdk23/dev/__properties__`文件内，libc 会解析这个文件，查找用户指定的`name`对应的属性值，将它放置到`value`里。

```java
int __system_property_get(const char *name, char *value);
```

为了躲避风控，我们希望不停的变化 Android 设备的这些属性，或者至少是让这些属性更方便被控制，从不同的测试机上 pull 出`/dev/__properties__`文件放到 Unidbg 的`src/main/resources/android/sdk23/dev`目录下是可行的，但不够灵活，十分笨重。

这时候我们就会想 Hook `__system_property_get`实现更灵活的属性控制，这当然是可行的，但事实上我们有更好的办法。因为它实在太高频出现了，所以 Unidbg 内置了对它的 Hook。

如何使用它呢？我这里写了一个简单的 demo [📎app-debug.7z](https://www.yuque.com/attachments/yuque/0/2022/7z/707476/1668233078919-22ca1362-6df6-4e34-a28d-963fa16ab461.7z)，代码逻辑很简单，就是使用`__system_property_get`获取一些常见的 Android 设备属性。

```java
#include <jni.h>
#include <string>
#include <sys/system_properties.h>
#include <stdexcept>
#include <android/log.h>


#define LOGE(...) __android_log_print(ANDROID_LOG_ERROR,"Lilac" ,__VA_ARGS__)
extern "C" JNIEXPORT jstring JNICALL
Java_com_example_getprop_MainActivity_stringFromJNI(
        JNIEnv* env,
        jobject /* this */) {
    char temp[128] = "0";
    __system_property_get("ro.build.id", temp);
    LOGE("id:%s",temp);
    __system_property_get("ro.build.version.sdk", temp);
    LOGE("version:%s",temp);
    __system_property_get("ro.board.platform", temp);
    LOGE("platform:%s",temp);
    __system_property_get("ro.product.cpu.abi", temp);
    LOGE("abi2:%s",temp);
    __system_property_get("ro.product.model", temp);
    LOGE("model:%s",temp);
    __system_property_get("ro.product.brand", temp);
    LOGE("brand:%s",temp);
    __system_property_get("ro.product.manufacturer", temp);
    LOGE("manufacturer:%s",temp);
    return env->NewStringUTF("");
}
```

使用 Unidbg 模拟执行

```java
package com.prop;


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

public class GetProp extends AbstractJni {
    private final AndroidEmulator emulator;
    private final DvmClass MainActivity;
    private final VM vm;

    public GetProp() {
        emulator = AndroidEmulatorBuilder
                .for64Bit()
                .addBackendFactory(new Unicorn2Factory(true))
                .build();

        Memory memory = emulator.getMemory();
        memory.setLibraryResolver(new AndroidResolver(23));
        vm = emulator.createDalvikVM(new File("unidbg-android/src/test/resources/getprop/app-debug.apk"));
        vm.setJni(this);
        vm.setVerbose(true);
        DalvikModule dm = vm.loadLibrary("getprop", true);
        MainActivity = vm.resolveClass("com.example.getprop.MainActivity");
        dm.callJNI_OnLoad(emulator);
    }

    public String call(){
        return MainActivity.newObject(null).callJniMethodObject(emulator, "stringFromJNI").getValue().toString();
    }

    public static void main(String[] args) {
        GetProp getProp = new GetProp();
        getProp.call();
    }
}
```

输入如下

```java
[main]E/Lilac: id:MDA89E
[main]E/Lilac: version:23
[main]E/Lilac: platform:msm8992
[main]E/Lilac: abi2:arm64-v8a
[main]E/Lilac: model:Nexus 5X
[main]E/Lilac: brand:google
[main]E/Lilac: manufacturer:LGE
```

这就是 Unidbg 默认的 `__properties__` 所解析出来的值。接下来使用 Unidbg 自带的 property Hook 做处理

```java
public GetProp() {
    emulator = AndroidEmulatorBuilder
        .for64Bit()
        .addBackendFactory(new Unicorn2Factory(true))
        .build();

    Memory memory = emulator.getMemory();
    memory.setLibraryResolver(new AndroidResolver(23));
    vm = emulator.createDalvikVM(new File("unidbg-android/src/test/resources/getprop/app-debug.apk"));
    vm.setJni(this);
    vm.setVerbose(true);

    // hook system property get
    SystemPropertyHook systemPropertyHook = new SystemPropertyHook(emulator);
    systemPropertyHook.setPropertyProvider(new SystemPropertyProvider() {
        @Override
        public String getProperty(String key) {
            return null;
        }
    });
    memory.addHookListener(systemPropertyHook);
    // 处理完毕

    DalvikModule dm = vm.loadLibrary("getprop", true);
    MainActivity = vm.resolveClass("com.example.getprop.MainActivity");
    dm.callJNI_OnLoad(emulator);
}
```

接下来对 getProperty 做处理，我这里只处理了两项

```java
SystemPropertyHook systemPropertyHook = new SystemPropertyHook(emulator);
systemPropertyHook.setPropertyProvider(new SystemPropertyProvider() {
    @Override
    public String getProperty(String key) {
        switch (key){
            case "ro.build.id":{
                return "get id";
            }
            case "ro.build.version.sdk":{
                return "get sdk";
            }
        }
        return null;
    }
});
memory.addHookListener(systemPropertyHook);
// 处理完毕
```

重新运行，发现效果很好

```java
[main]E/Lilac: id:get id
[main]E/Lilac: version:get sdk
[main]E/Lilac: platform:msm8992
[main]E/Lilac: abi2:arm64-v8a
[main]E/Lilac: model:Nexus 5X
[main]E/Lilac: brand:google
[main]E/Lilac: manufacturer:LGE
```

在使用上的注意点有两个

1 是很多人会忘记`memory.addHookListener(systemPropertyHook);`这一步，不添加监听器拦截就不会生效，绝不能漏。

2 是放的时机晚于 loadlibrary，比如下面这样，这样同样不会生效。

```java
DalvikModule dm = vm.loadLibrary("getprop", true);

SystemPropertyHook systemPropertyHook = new SystemPropertyHook(emulator);
systemPropertyHook.setPropertyProvider(new SystemPropertyProvider() {
    @Override
    public String getProperty(String key) {
        switch (key){
            case "ro.build.id":{
                return "get id";
            }
            case "ro.build.version.sdk":{
                return "get sdk";
            }
        }
        return null;
    }
});
memory.addHookListener(systemPropertyHook);
// 处理完毕
```

必须要早于 SO 解析和装载逻辑的原因也和简单，Hook 的添加时机在 Unidbg Loader 解析符号的地方。

在学习完它的基本使用后，我们简单谈一下它的原理，读者应该意识到了，它和我们自己写的函数 Hook 不太一样，事实上确实如此，这是 Unidbg 内部所使用的基于 SVC Hook 的函数 Hook 处理办法。

我打开 traceCode，从执行流的角度做简单讨论

```java
DalvikModule dm = vm.loadLibrary("getprop", true);

MainActivity = vm.resolveClass("com.example.getprop.MainActivity");
dm.callJNI_OnLoad(emulator);
// 汇编追踪
emulator.traceCode();
```

把目标 SO 放到 IDA 里

![](https://cdn.nlark.com/yuque/0/2022/png/707476/1668235019924-4790cd0d-6e0a-499b-8760-24a90fea3291.png)

第一个调用发生在 0x7C4，在汇编执行流中找到它，找到 PLT 表里它的代码片段

```java
.plt:0000000000000700 .__system_property_get                  ; CODE XREF: Java_com_example_getprop_MainActivity_stringFromJNI+68↓p
.plt:0000000000000700                                         ; Java_com_example_getprop_MainActivity_stringFromJNI+B0↓p ...
.plt:0000000000000700                 ADRP            X16, #__system_property_get_ptr@PAGE
.plt:0000000000000704                 LDR             X17, [X16,#__system_property_get_ptr@PAGEOFF]
.plt:0000000000000708                 ADD             X16, X16, #__system_property_get_ptr@PAGEOFF
.plt:000000000000070C                 BR              X17
.plt:000000000000070C ; End of function .__system_property_get
```

按照预期，程序应该从 0x70C 跳转到 libc 里这个函数的实现上，执行处理完成后，返回到 0x7C8。

那么 0x70C 到 0x7C8 中间发生了什么？找到这段汇编代码。

```java
[14:35:14 560][libgetprop.so 0x0070c] [20021fd6] 0x4000070c: "br x17" x17=0xfffe17f0
[14:35:14 561][Arm64Hook     0x017f0] [413e00d4] 0xfffe17f0: "svc #0x1f2" => x30=0x400007c8
[14:35:14 562][Arm64Hook     0x017f4] [f18740f8] 0xfffe17f4: "ldr x17, [sp], #8" sp=0xbffff5e8 => x17=0x400007c8 sp=0xbffff5f0
[14:35:14 562][Arm64Hook     0x017f8] [20021fd6] 0xfffe17f8: "br x17" x17=0x400007c8
[14:35:14 563][libgetprop.so 0x007c8] [08000090] 0x400007c8: "adrp x8, #0x40000000" => x8=0x40000000
```

可以简单将它理解为转成`SVC`系统调用，然后在 Undibg 的中断 Hook 里做处理，最后恢复寄存器环境，调回 0x7C8。在 Unidbg 内部，SVC Hook 随处可见，但用户使用 Unidbg 做 Hook 时，还是常规的 Console Debugger 以及 HookZz 那些方式更好用，SVC Hook 主要是方便兼容性和持久化。

除了这个函数，在我的另外一篇[文章](https://blog.csdn.net/qq_38851536/article/details/118334978)里，讨论的 setenv 函数也类似，可以 Hook 处理。

## 三、尾声

`__system_property_get`是一个常见、高频的设备信息获取方案，读者在使用 Unidbg 补环境时，不要忘了处理这个函数，返回恰当的值。





# unidbg补库函数(sensor传感器)
## 一、引言

如果 Unidbg 依赖的 SO 没法顺利加载怎么办？

## 二、问题分析

在基础篇里讨论过的虚拟模块，本质上也是补库函数的问题，只不过要补的有点多，而且都在一个模块里，所以 Unidbg 干脆用 SVC Hook 设计成了虚拟模块这样一个机制。

这里直接出题，这是 demo apk：[📎app-debug.7z](https://www.yuque.com/attachments/yuque/0/2022/7z/707476/1668236478829-013d16f9-130b-4b57-9105-b2e193431756.7z)

JADX 反编译 APK，找到`com.example.getsensorinfo.MainActivity`类，其中的`getsensorinfo`就是目标函数。

```java
package com.example.getsensorinfo;

import android.os.Bundle;
import android.widget.TextView;
import androidx.appcompat.app.AppCompatActivity;
import com.example.getsensorinfo.databinding.ActivityMainBinding;

public class MainActivity extends AppCompatActivity {
    private ActivityMainBinding binding;

    public native void getsensorinfo();

    public native String stringFromJNI();

    static {
        System.loadLibrary("getsensorinfo");
    }


    @Override // androidx.fragment.app.FragmentActivity, androidx.activity.ComponentActivity, androidx.core.app.ComponentActivity, android.app.Activity
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        ActivityMainBinding inflate = ActivityMainBinding.inflate(getLayoutInflater());
        this.binding = inflate;
        setContentView(inflate.mo156getRoot());
        TextView tv = this.binding.sampleText;
        tv.setText(stringFromJNI());
    }
}
```

使用 Unidbg 做初步的模拟执行

```java
package com.prop;

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

public class GetSensor extends AbstractJni {
    private final AndroidEmulator emulator;
    private final DvmClass MainActivity;
    private final VM vm;

    public GetSensor() {
        emulator = AndroidEmulatorBuilder
                .for32Bit()
                .addBackendFactory(new Unicorn2Factory(true))
                .build();

        Memory memory = emulator.getMemory();
        memory.setLibraryResolver(new AndroidResolver(23));
        vm = emulator.createDalvikVM(new File("unidbg-android/src/test/resources/getprop/sensor.apk"));
        vm.setJni(this);
        vm.setVerbose(true);


        DalvikModule dm = vm.loadLibrary("getsensorinfo", true);

        MainActivity = vm.resolveClass("com.example.getsensorinfo.MainActivity");
        dm.callJNI_OnLoad(emulator);
    }

    public void call(){
        MainActivity.newObject(null).callJniMethod(emulator, "getsensorinfo");
    }

    public static void main(String[] args) {
        GetSensor sensor = new GetSensor();
        sensor.call();
    }
}
```

运行测试

```java
[15:05:12 474]  INFO [com.github.unidbg.linux.AndroidElfLoader] (AndroidElfLoader:470) - libgetsensorinfo.so load dependency libandroid.so failed
Find native function Java_com_example_getsensorinfo_MainActivity_getsensorinfo => RX@0x40000949[libgetsensorinfo.so]0x949
[15:05:12 625]  WARN [com.github.unidbg.arm.AbstractARMEmulator] (AbstractARMEmulator$1:64) - Fetch memory failed: address=0x818, size=1, value=0x0, PC=unidbg@0x818, LR=RX@0x40000961[libgetsensorinfo.so]0x961
[15:05:12 626]  WARN [com.github.unidbg.AbstractEmulator] (AbstractEmulator:420) - emulate RX@0x40000949[libgetsensorinfo.so]0x949 exception sp=unidbg@0xbffff6e8, msg=unicorn.UnicornException: Invalid memory fetch (UC_ERR_FETCH_UNMAPPED), offset=1ms
Exception in thread "main" java.lang.NullPointerException: Cannot invoke "com.github.unidbg.linux.android.dvm.DvmObject.getValue()" because the return value of "com.github.unidbg.linux.android.dvm.DvmObject.callJniMethodObject(com.github.unidbg.Emulator, String, Object[])" is null
	at com.prop.GetSensor.call(GetSensor.java:40)
```

加载 libandroid.so 所对应的虚拟模块

```java
// 使用 libandroid.so 的虚拟模块
new AndroidModule(emulator, vm).register(memory);
DalvikModule dm = vm.loadLibrary("getsensorinfo", true);
```

运行发现仍然有地址异常，使用 traceCode 分析发生异常的地址

```java
[15:07:15 045][libgetsensorinfo.so 0x0095d] [fff772ef] 0x4000095c: "blx #0x40000844"
[15:07:15 049][libgetsensorinfo.so 0x00844] [00c68fe2] 0x40000844: "add ip, pc, #0, #12" => ip=0x4000084c
[15:07:15 051][libgetsensorinfo.so 0x00848] [03ca8ce2] 0x40000848: "add ip, ip, #0x3000" ip=0x4000084c => ip=0x4000384c
[15:07:15 053][libgetsensorinfo.so 0x0084c] [74f7bce5] 0x4000084c: "ldr pc, [ip, #0x774]!" ip=0x4000384cException in thread "main" java.lang.NullPointerException: Cannot invoke "com.github.unidbg.linux.android.dvm.DvmObject.getValue()" because the return value of "com.github.unidbg.linux.android.dvm.DvmObject.callJniMethodObject(com.github.unidbg.Emulator, String, Object[])" is null
	at com.prop.GetSensor.call(GetSensor.java:44)
	at com.prop.GetSensor.main(GetSensor.java:49)
```

IDA 静态分析 SO，找到具体位置

```java
.text:00000950                 ADD             R2, PC  ; __stack_chk_guard_ptr
.text:00000952                 LDR             R2, [R2] ; __stack_chk_guard
.text:00000954                 LDR             R2, [R2]
.text:00000956                 STR             R2, [SP,#0x48+var_C]
.text:00000958                 STR             R0, [SP,#0x48+var_14]
.text:0000095A                 STR             R1, [SP,#0x48+var_18]
.text:0000095C                 BLX             ASensorManager_getInstance
.text:00000960                 STR             R0, [SP,#0x48+manager]
.text:00000962                 LDR             R0, [SP,#0x48+manager] ; manager
```

问题出在 ASensorManager_getInstance 调用上，这是一个 libandroid.so 里的函数，但 Unidbg 实现的 AndroidModule 虚拟模块里没有实现它。

![](https://cdn.nlark.com/yuque/0/2022/png/707476/1668236995152-a3b29d8f-d8ff-48f6-b417-6326ffc0c52e.png)

这可如何是好？

## 三、解决办法
### 自建AndroidModule
在原有AndroidModule基础上，对 AndroidModule 做了一定扩充，使得访问传感器列表时，硬编码返回了一个传感器的信息。
```java
package com.prop;

import com.github.unidbg.Emulator;
import com.github.unidbg.arm.Arm64Svc;
import com.github.unidbg.arm.ArmSvc;
import com.github.unidbg.arm.backend.BackendException;
import com.github.unidbg.arm.context.RegisterContext;
import com.github.unidbg.linux.android.dvm.DvmClass;
import com.github.unidbg.linux.android.dvm.DvmObject;
import com.github.unidbg.linux.android.dvm.VM;
import com.github.unidbg.linux.android.dvm.api.Asset;
import com.github.unidbg.memory.MemoryBlock;
import com.github.unidbg.memory.SvcMemory;
import com.github.unidbg.pointer.UnidbgPointer;
import com.github.unidbg.virtualmodule.VirtualModule;
import com.sun.jna.Pointer;
import org.apache.commons.logging.Log;
import org.apache.commons.logging.LogFactory;

import java.nio.ByteBuffer;
import java.nio.ByteOrder;
import java.util.Map;

public class AndroidModule extends VirtualModule<VM> {

    private static final Log log = LogFactory.getLog(AndroidModule.class);

    public AndroidModule(Emulator<?> emulator, VM vm) {
        super(emulator, vm, "libandroid.so");
    }

    @Override
    protected void onInitialize(Emulator<?> emulator, final VM vm, Map<String, UnidbgPointer> symbols) {
        boolean is64Bit = emulator.is64Bit();
        SvcMemory svcMemory = emulator.getSvcMemory();
        symbols.put("AAssetManager_fromJava", svcMemory.registerSvc(is64Bit ? new Arm64Svc() {
            @Override
            public long handle(Emulator<?> emulator) {
                return fromJava(emulator, vm);
            }
        } : new ArmSvc() {
            @Override
            public long handle(Emulator<?> emulator) {
                return fromJava(emulator, vm);
            }
        }));
        symbols.put("AAssetManager_open", svcMemory.registerSvc(is64Bit ? new Arm64Svc() {
            @Override
            public long handle(Emulator<?> emulator) {
                return open(emulator, vm);
            }
        } : new ArmSvc() {
            @Override
            public long handle(Emulator<?> emulator) {
                return open(emulator, vm);
            }
        }));
        symbols.put("AAsset_close", svcMemory.registerSvc(is64Bit ? new Arm64Svc() {
            @Override
            public long handle(Emulator<?> emulator) {
                return close(emulator, vm);
            }
        } : new ArmSvc() {
            @Override
            public long handle(Emulator<?> emulator) {
                return close(emulator, vm);
            }
        }));
        symbols.put("AAsset_getBuffer", svcMemory.registerSvc(is64Bit ? new Arm64Svc() {
            @Override
            public long handle(Emulator<?> emulator) {
                return getBuffer(emulator, vm);
            }
        } : new ArmSvc() {
            @Override
            public long handle(Emulator<?> emulator) {
                return getBuffer(emulator, vm);
            }
        }));
        symbols.put("AAsset_getLength", svcMemory.registerSvc(is64Bit ? new Arm64Svc() {
            @Override
            public long handle(Emulator<?> emulator) {
                return getLength(emulator, vm);
            }
        } : new ArmSvc() {
            @Override
            public long handle(Emulator<?> emulator) {
                return getLength(emulator, vm);
            }
        }));
        symbols.put("AAsset_read", svcMemory.registerSvc(is64Bit ? new Arm64Svc() {
            @Override
            public long handle(Emulator<?> emulator) {
                throw new BackendException();
            }
        } : new ArmSvc() {
            @Override
            public long handle(Emulator<?> emulator) {
                return read(emulator, vm);
            }
        }));
        symbols.put("ASensorManager_getInstance", svcMemory.registerSvc(is64Bit ? new Arm64Svc() {
            @Override
            public long handle(Emulator<?> emulator) {
                throw new BackendException();
            }
        } : new ArmSvc() {
            @Override
            public long handle(Emulator<?> emulator) {
                return ASensorManager_getInstance(emulator, vm);
            }
        }));
        symbols.put("ASensor_getName", svcMemory.registerSvc(is64Bit ? new Arm64Svc() {
            @Override
            public long handle(Emulator<?> emulator) {
                throw new BackendException();
            }
        } : new ArmSvc() {
            @Override
            public long handle(Emulator<?> emulator) {
                return ASensor_getName(emulator, vm);
            }
        }));
        symbols.put("ASensor_getVendor", svcMemory.registerSvc(is64Bit ? new Arm64Svc() {
            @Override
            public long handle(Emulator<?> emulator) {
                throw new BackendException();
            }
        } : new ArmSvc() {
            @Override
            public long handle(Emulator<?> emulator) {
                return ASensor_getVendor(emulator, vm);
            }
        }));
        symbols.put("ASensor_getType", svcMemory.registerSvc(is64Bit ? new Arm64Svc() {
            @Override
            public long handle(Emulator<?> emulator) {
                throw new BackendException();
            }
        } : new ArmSvc() {
            @Override
            public long handle(Emulator<?> emulator) {
                return ASensor_getType(emulator, vm);
            }
        }));
        symbols.put("ASensor_getResolution", svcMemory.registerSvc(is64Bit ? new Arm64Svc() {
            @Override
            public long handle(Emulator<?> emulator) {
                throw new BackendException();
            }
        } : new ArmSvc() {
            @Override
            public long handle(Emulator<?> emulator) {
                return ASensor_getResolution(emulator, vm);
            }
        }));
        symbols.put("ASensor_getMinDelay", svcMemory.registerSvc(is64Bit ? new Arm64Svc() {
            @Override
            public long handle(Emulator<?> emulator) {
                throw new BackendException();
            }
        } : new ArmSvc() {
            @Override
            public long handle(Emulator<?> emulator) {
                return ASensor_getMinDelay(emulator, vm);
            }
        }));
        symbols.put("ASensor_getFifoMaxEventCount", svcMemory.registerSvc(is64Bit ? new Arm64Svc() {
            @Override
            public long handle(Emulator<?> emulator) {
                throw new BackendException();
            }
        } : new ArmSvc() {
            @Override
            public long handle(Emulator<?> emulator) {
                return ASensor_getFifoMaxEventCount(emulator, vm);
            }
        }));
        symbols.put("ASensor_getFifoReservedEventCount", svcMemory.registerSvc(is64Bit ? new Arm64Svc() {
            @Override
            public long handle(Emulator<?> emulator) {
                throw new BackendException();
            }
        } : new ArmSvc() {
            @Override
            public long handle(Emulator<?> emulator) {
                return ASensor_getFifoReservedEventCount(emulator, vm);
            }
        }));
        symbols.put("ASensor_getStringType", svcMemory.registerSvc(is64Bit ? new Arm64Svc() {
            @Override
            public long handle(Emulator<?> emulator) {
                throw new BackendException();
            }
        } : new ArmSvc() {
            @Override
            public long handle(Emulator<?> emulator) {
                return ASensor_getStringType(emulator, vm);
            }
        }));
        symbols.put("ASensor_getReportingMode", svcMemory.registerSvc(is64Bit ? new Arm64Svc() {
            @Override
            public long handle(Emulator<?> emulator) {
                throw new BackendException();
            }
        } : new ArmSvc() {
            @Override
            public long handle(Emulator<?> emulator) {
                return ASensor_getReportingMode(emulator, vm);
            }
        }));

        symbols.put("ASensor_isWakeUpSensor", svcMemory.registerSvc(is64Bit ? new Arm64Svc() {
            @Override
            public long handle(Emulator<?> emulator) {
                throw new BackendException();
            }
        } : new ArmSvc() {
            @Override
            public long handle(Emulator<?> emulator) {
                return ASensor_isWakeUpSensor(emulator, vm);
            }
        }));

        symbols.put("ASensorManager_getSensorList", svcMemory.registerSvc(is64Bit ? new Arm64Svc() {
            @Override
            public long handle(Emulator<?> emulator) {
                throw new BackendException();
            }
        } : new ArmSvc() {
            @Override
            public long handle(Emulator<?> emulator) {
                return ASensorManager_getSensorList(emulator, vm);
            }
        }));

    }

    private static long fromJava(Emulator<?> emulator, VM vm) {
        RegisterContext context = emulator.getContext();
        Pointer env = context.getPointerArg(0);
        UnidbgPointer assetManager = context.getPointerArg(1);
        DvmObject<?> obj = vm.getObject(assetManager.toIntPeer());
        if (log.isDebugEnabled()) {
            log.debug("AAssetManager_fromJava env=" + env + ", assetManager=" + obj.getObjectType() + ", LR=" + context.getLRPointer());
        }
        return assetManager.peer;
    }

    private static long open(Emulator<?> emulator, VM vm) {
        RegisterContext context = emulator.getContext();
        Pointer amgr = context.getPointerArg(0);
        String filename = context.getPointerArg(1).getString(0);
        int mode = context.getIntArg(2);
        if (log.isDebugEnabled()) {
            log.debug("AAssetManager_open amgr=" + amgr + ", filename=" + filename + ", mode=" + mode + ", LR=" + context.getLRPointer());
        }
        final int AASSET_MODE_STREAMING = 2;
        final int AASSET_MODE_BUFFER = 3;
        if (mode == AASSET_MODE_STREAMING || AASSET_MODE_BUFFER == mode) {
            byte[] data = vm.openAsset(filename);
            if (data != null) {
                Asset asset = new Asset(vm, filename);
                asset.open(emulator, data);
                return vm.addLocalObject(asset);
            }
        }
        throw new BackendException("filename=" + filename + ", mode=" + mode);
    }

    private static long close(Emulator<?> emulator, VM vm) {
        RegisterContext context = emulator.getContext();
        UnidbgPointer pointer = context.getPointerArg(0);
        Asset asset = vm.getObject(pointer.toIntPeer());
        asset.close();
        if (log.isDebugEnabled()) {
            log.debug("AAsset_close pointer=" + pointer + ", LR=" + context.getLRPointer());
        }
        return 0;
    }

    private static long getBuffer(Emulator<?> emulator, VM vm) {
        RegisterContext context = emulator.getContext();
        UnidbgPointer pointer = context.getPointerArg(0);
        Asset asset = vm.getObject(pointer.toIntPeer());
        UnidbgPointer buffer = asset.getBuffer();
        if (log.isDebugEnabled()) {
            log.debug("AAsset_getBuffer pointer=" + pointer + ", buffer=" + buffer + ", LR=" + context.getLRPointer());
        }
        return buffer.peer;
    }

    private static long getLength(Emulator<?> emulator, VM vm) {
        RegisterContext context = emulator.getContext();
        UnidbgPointer pointer = context.getPointerArg(0);
        Asset asset = vm.getObject(pointer.toIntPeer());
        int length = asset.getLength();
        if (log.isDebugEnabled()) {
            log.debug("AAsset_getLength pointer=" + pointer + ", length=" + length + ", LR=" + context.getLRPointer());
        }
        return length;
    }

    private static long read(Emulator<?> emulator, VM vm) {
        RegisterContext context = emulator.getContext();
        UnidbgPointer pointer = context.getPointerArg(0);
        Pointer buf = context.getPointerArg(1);
        int count = context.getIntArg(2);
        Asset asset = vm.getObject(pointer.toIntPeer());
        byte[] bytes = asset.read(count);
        if (log.isDebugEnabled()) {
            log.debug("AAsset_read pointer=" + pointer + ", buf=" + buf + ", count=" + count + ", LR=" + context.getLRPointer());
        }
        buf.write(0, bytes, 0, bytes.length);
        return bytes.length;
    }

    private static long ASensorManager_getInstance(Emulator<?> emulator, VM vm) {
        RegisterContext context = emulator.getContext();
        DvmClass SensorManager = vm.resolveClass("android/hardware/SensorManager");
        DvmObject<?> mpSensorManager = SensorManager.newObject(null);
        if (log.isDebugEnabled()) {
            log.debug("ASensorManager_getInstance newInstance=" + mpSensorManager +  ", LR=" + context.getLRPointer());
        }
        return mpSensorManager.hashCode();
    }

    private static long ASensor_getName(Emulator<?> emulator, VM vm) {
        RegisterContext context = emulator.getContext();
        UnidbgPointer sensor = context.getPointerArg(0);
        UnidbgPointer namePtr = sensor.getPointer(0);
        if (log.isDebugEnabled()) {
            log.debug("ASensor_getName sensor Address=" + sensor +  ", name=" + namePtr.getString(0) +", LR=" + context.getLRPointer());
        }
        return namePtr.peer;
    }


    private static long ASensor_getVendor(Emulator<?> emulator, VM vm) {

        RegisterContext context = emulator.getContext();
        UnidbgPointer sensor = context.getPointerArg(0);
        UnidbgPointer VendorPtr = sensor.getPointer(4);
        if (log.isDebugEnabled()) {
            log.debug("ASensor_getVendor sensor Address=" + sensor +  ", Vendor=" + VendorPtr.getString(0) +", LR=" + context.getLRPointer());
        }
        return VendorPtr.peer;
    }


    private static long ASensor_getType(Emulator<?> emulator, VM vm) {
        RegisterContext context = emulator.getContext();
        UnidbgPointer sensor = context.getPointerArg(0);
        System.out.println(sensor);

        int Type = sensor.getInt(8);
        if (log.isDebugEnabled()) {
            log.debug("ASensor_getVendor sensor Address=" + sensor +  ", Type=" + Type +", LR=" + context.getLRPointer());
        }

        return Type;
    }


    private static Long ASensor_getResolution(Emulator<?> emulator, VM vm) {
        RegisterContext context = emulator.getContext();
        UnidbgPointer sensor = context.getPointerArg(0);
        float Resolution = sensor.getFloat(12);
        if (log.isDebugEnabled()) {
            log.debug("ASensor_getVendor sensor Address=" + sensor +  ", Resolution=" + Resolution +", LR=" + context.getLRPointer());
        }
        ByteBuffer buffer = ByteBuffer.allocate(4);
        buffer.order(ByteOrder.LITTLE_ENDIAN);
        buffer.putFloat(Resolution);
        buffer.flip();
        return (buffer.getInt() & 0xffffffffL);
    }

    private static long ASensor_getMinDelay(Emulator<?> emulator, VM vm) {
        RegisterContext context = emulator.getContext();
        UnidbgPointer sensor = context.getPointerArg(0);
        int MinDelay = sensor.getInt(16);
        if (log.isDebugEnabled()) {
            log.debug("ASensor_getVendor sensor Address=" + sensor +  ", MinDelay=" + MinDelay +", LR=" + context.getLRPointer());
        }
        return MinDelay;
    }

    private static long ASensor_getFifoMaxEventCount(Emulator<?> emulator, VM vm) {
        RegisterContext context = emulator.getContext();
        UnidbgPointer sensor = context.getPointerArg(0);
        int FifoMaxEventCount = sensor.getInt(20);
        if (log.isDebugEnabled()) {
            log.debug("ASensor_getVendor sensor Address=" + sensor +  ", FifoMaxEventCount=" + FifoMaxEventCount +", LR=" + context.getLRPointer());
        }
        return FifoMaxEventCount;
    }


    private static long ASensor_getFifoReservedEventCount(Emulator<?> emulator, VM vm) {
        RegisterContext context = emulator.getContext();
        UnidbgPointer sensor = context.getPointerArg(0);
        int FifoReservedEventCount = sensor.getInt(24);
        if (log.isDebugEnabled()) {
            log.debug("ASensor_getVendor sensor Address=" + sensor +  ", FifoReservedEventCount=" + FifoReservedEventCount +", LR=" + context.getLRPointer());
        }
        return FifoReservedEventCount;
    }

    private static long ASensor_getStringType(Emulator<?> emulator, VM vm) {
        RegisterContext context = emulator.getContext();
        UnidbgPointer sensor = context.getPointerArg(0);
        UnidbgPointer StringType = sensor.getPointer(28);
        if (log.isDebugEnabled()) {
            log.debug("ASensor_getVendor sensor Address=" + sensor +  ", StringType=" + StringType.getString(0) +", LR=" + context.getLRPointer());
        }
        return StringType.peer;
    }


    private static long ASensor_getReportingMode(Emulator<?> emulator, VM vm) {
        RegisterContext context = emulator.getContext();
        UnidbgPointer sensor = context.getPointerArg(0);
        int ReportingMode = sensor.getInt(32);
        if (log.isDebugEnabled()) {
            log.debug("ASensor_getVendor sensor Address=" + sensor +  ", ReportingMode=" + ReportingMode +", LR=" + context.getLRPointer());
        }
        return ReportingMode;
    }

    private static long ASensor_isWakeUpSensor(Emulator<?> emulator, VM vm) {
        RegisterContext context = emulator.getContext();
        UnidbgPointer sensor = context.getPointerArg(0);
        int isWakeUpSensor = sensor.getInt(36);
        if (log.isDebugEnabled()) {
            log.debug("ASensor_getVendor sensor Address=" + sensor +  ", isWakeUpSensor=" + isWakeUpSensor +", LR=" + context.getLRPointer());
        }
        return isWakeUpSensor;
    }


    private static long ASensorManager_getSensorList(Emulator<?> emulator, VM vm) {
        RegisterContext context = emulator.getContext();
        ;
        Pointer listptr = context.getPointerArg(1);

        Pointer pointer = allocateMemoryBlock(emulator, 40);

        String name = "BMI160 accelerometer";
        Pointer namePointer = emulator.getMemory().malloc(name.length() + 1, true).getPointer();
        namePointer.setString(0, name);
        pointer.setPointer(0, namePointer);

        String Vendor = "Bosch";
        Pointer VendorPointer = emulator.getMemory().malloc(Vendor.length() + 1, true).getPointer();
        VendorPointer.setString(0, Vendor);
        pointer.setPointer(4, namePointer);

        int type = 1;
        pointer.setInt(8, type);

        float mResolution = 0.0047884034F;
        pointer.setFloat(12, mResolution);

        int MinDelay = 2500;
        pointer.setInt(16, MinDelay);

        int FifoMaxEventCount = 5736;
        pointer.setInt(20, FifoMaxEventCount);

        int FifoReservedEventCount = 3000;
        pointer.setInt(24, FifoReservedEventCount);

        String StringType = "android.sensor.accelerometer";
        Pointer StringTypePointer = emulator.getMemory().malloc(StringType.length() + 1, true).getPointer();
        StringTypePointer.setString(0, StringType);
        pointer.setPointer(28, StringTypePointer);

        int ReportingMode = 0;
        pointer.setInt(32, ReportingMode);

        int isWakeUpSensor = 0;
        pointer.setInt(36, isWakeUpSensor);

        Pointer pointerstub = allocateMemoryBlock(emulator, 40);

        listptr.setPointer(0, pointerstub);
        pointerstub.setPointer(0, pointer);

        return 1;
    }

    protected static UnidbgPointer allocateMemoryBlock(Emulator<?> emulator, int length) {
        MemoryBlock memoryBlock = emulator.getMemory().malloc(length, true);
        return memoryBlock.getPointer();
    }
}
```


### hook库函数实现
利用xhook hook库函数补充逻辑
```java
package com.myunidbg;  
  
import com.github.unidbg.AndroidEmulator;  
import com.github.unidbg.Emulator;  
import com.github.unidbg.arm.HookStatus;  
import com.github.unidbg.arm.backend.Unicorn2Factory;  
import com.github.unidbg.arm.context.RegisterContext;  
import com.github.unidbg.hook.ReplaceCallback;  
import com.github.unidbg.hook.xhook.IxHook;  
import com.github.unidbg.linux.android.AndroidEmulatorBuilder;  
import com.github.unidbg.linux.android.AndroidResolver;  
import com.github.unidbg.linux.android.XHookImpl;  
import com.github.unidbg.linux.android.dvm.*;  
import com.github.unidbg.memory.Memory;  
import com.github.unidbg.pointer.UnidbgPointer;  
import com.github.unidbg.virtualmodule.android.AndroidModule;  
import unicorn.ArmConst;  
  
import java.io.File;  
  
public class GetSensor extends AbstractJni {  
    private final AndroidEmulator emulator;  
    private final VM vm;  
    private final DvmClass dvmClass;  
    private IxHook ixHook;  
  
  
    public GetSensor(){  
        emulator = AndroidEmulatorBuilder  
                .for32Bit()  
                .addBackendFactory(new Unicorn2Factory(true))  
                .setProcessName("com.my.getsensor")  
                .build();  
  
        Memory memory = emulator.getMemory();  
        memory.setLibraryResolver(new AndroidResolver(23));  
  
        vm = emulator.createDalvikVM(new File("D:\\01ClassMP4\\unidbgfile\\kuhanshu4\\app-debug.apk"));  
        vm.setVerbose(true);  
        vm.setJni(this);  
  
        new AndroidModule(emulator, vm).register(memory);  
  
  
        DalvikModule dm = vm.loadLibrary("getsensorinfo", true);  
        dm.callJNI_OnLoad(emulator);  
  
        ixHook = XHookImpl.getInstance(emulator);  
        XhookRegister();  
        ixHook.refresh();  
  
        dvmClass = vm.resolveClass("com.example.getsensorinfo.MainActivity");  
        //emulator.traceCode();  
    }  
  
    public void call(){  
        dvmClass.newObject(null).callJniMethod(emulator, "getsensorinfo");  
    }  
  
    public static void main(String[] args){  
        GetSensor getSensor = new GetSensor();  
        getSensor.call();  
    }  
  
    public void XhookRegister(){  
        ixHook.register("libgetsensorinfo.so", "ASensorManager_getInstance", new ReplaceCallback() {  
            @Override  
            public HookStatus onCall(Emulator<?> emulator, long originFunction) {  
                System.out.println("AndroidManager getInstance called");  
                DvmObject<?> dvmObject = vm.resolveClass("android/hardware/SensorManager").newObject(null);  
                return HookStatus.LR(emulator, dvmObject.hashCode());  
            }  
        });  
  
        ixHook.register("libgetsensorinfo.so", "ASensorManager_getSensorList", new ReplaceCallback() {  
            @Override  
            public HookStatus onCall(Emulator<?> emulator, long originFunction) {  
                System.out.println("AndroidManager getSensorList called");  
  
                RegisterContext context = emulator.getContext();  
                UnidbgPointer sensorListPointer = context.getPointerArg(1);  
                UnidbgPointer sensorList = allocateMemory(emulator, 40);  
                UnidbgPointer sensorFunction = allocateMemory(emulator, 40);  
                sensorListPointer.setPointer(0, sensorList);  
                sensorList.setPointer(0, sensorFunction);  
  
                String name = "BMI160 accelerometer";  
                UnidbgPointer namePointer = allocateMemory(emulator, name.length() + 1);  
                namePointer.setString(0, name);  
                sensorFunction.setPointer(0, namePointer);  
  
                String vendor = "Bosch";  
                UnidbgPointer ventorPointer = allocateMemory(emulator, name.length() + 1);  
                ventorPointer.setString(0, vendor);  
                sensorFunction.setPointer(4, ventorPointer);  
  
                int type = 1;  
                sensorFunction.setInt(8, type);  
  
                float mResolution = 0.0047884034F;  
                sensorFunction.setFloat(12, mResolution);  
  
                int minDelay = 2500;  
                sensorFunction.setInt(16, minDelay);  
  
                int FifoMaxEventCount = 5736;  
                sensorFunction.setInt(20, FifoMaxEventCount);  
  
                int FifoReservedEventCount = 3000;  
                sensorFunction.setInt(24, FifoReservedEventCount);  
  
                String StringType = "android.sensor.accelerometer";  
                UnidbgPointer StringTypePointer = emulator.getMemory().malloc(StringType.length() + 1, true).getPointer();  
                StringTypePointer.setString(0, StringType);  
                sensorFunction.setPointer(28, StringTypePointer);  
  
                int ReportingMode = 0;  
                sensorFunction.setInt(32, ReportingMode);  
  
                int isWakeUpSensor = 0;  
                sensorFunction.setInt(36, isWakeUpSensor);  
  
                return HookStatus.LR(emulator, 1);  
            }  
        });  
  
        ixHook.register("libgetsensorinfo.so", "ASensor_getType", new ReplaceCallback() {  
            @Override  
            public HookStatus onCall(Emulator<?> emulator, long originFunction) {  
                UnidbgPointer sensorPointer = emulator.getContext().getPointerArg(0);  
                int type = sensorPointer.getInt(8);  
                System.out.println("get type result: " + type);  
                return HookStatus.LR(emulator, type);  
            }  
        });  
  
        ixHook.register("libgetsensorinfo.so", "ASensor_getName", new ReplaceCallback() {  
            @Override  
            public HookStatus onCall(Emulator<?> emulator, long originFunction) {  
                UnidbgPointer sensorPointer = emulator.getContext().getPointerArg(0);  
                UnidbgPointer namePointer = sensorPointer.getPointer(0);  
                System.out.println("get name result: " + namePointer.getString(0));  
                return HookStatus.LR(emulator, namePointer.toIntPeer());  
            }  
        });  
  
        ixHook.register("libgetsensorinfo.so", "ASensor_getVendor", new ReplaceCallback() {  
            @Override  
            public HookStatus onCall(Emulator<?> emulator, long originFunction) {  
                UnidbgPointer sensorPointer = emulator.getContext().getPointerArg(0);  
                UnidbgPointer vendorPointer = sensorPointer.getPointer(4);  
                System.out.println("get vendor result: " + vendorPointer.getString(0));  
                return HookStatus.LR(emulator, vendorPointer.toIntPeer());  
            }  
        });  
  
        ixHook.register("libgetsensorinfo.so", "ASensor_getResolution", new ReplaceCallback() {  
            @Override  
            public HookStatus onCall(Emulator<?> emulator, long originFunction) {  
                UnidbgPointer sensorPointer = emulator.getContext().getPointerArg(0);  
                float resolution = sensorPointer.getFloat(12);  
                System.out.println("get resolution result: " + resolution);  
                return HookStatus.LR(emulator, (long)resolution);  
            }  
        });  
  
    }  
  
    public UnidbgPointer allocateMemory(Emulator<?> emulator, int length){  
        return emulator.getMemory().malloc(length, true).getPointer();  
    }  
  
}
```


## 四、尾声
除非万不得已，我们并不愿意为补环境花这么多功夫，但传感器检测是很关键的一项检测，在检测模拟器、设备指纹等场景都会使用到，所以才有了这一篇内容。





# unidbg补库函数(与内存相关)
## 一、引言

本篇记录内存相关的库函数、系统调用处理。

## 二、Free

free 函数是重要的内存释放函数，原型如下。

```java
void free(void *ptr)
```

Unidbg 在内存管理方面存在一些问题，在内存的释放上尤为明显，munmap 以及 free 都有不低的出错概率。先讨论 free，常见报错如下。

```java
Invalid address 0x40175000 passed to free: value not allocated
[crash]A/libc: Invalid address 0x40175000 passed to free: value not allocated
```

遇到这个报错，最简单的处理办法就是 hook free 函数，替换它的实现，让它什么都不做就直接返回。这里我用 Whale 来实现，让它直接返回，且返回值是 0，即释放成功之意。

```java
public void patchFree(){
    IWhale whale = Whale.getInstance(emulator);
    Symbol free = emulator.getMemory().findModule("libc.so").findSymbolByName("free");
    whale.inlineHookFunction(free, new ReplaceCallback() {
        @Override
        public HookStatus onCall(Emulator<?> emulator, long originFunction) {
            System.out.println("WInlineHookFunction free=" + emulator.getContext().getPointerArg(0));
            return HookStatus.LR(emulator, 0);
        }
    });
}
```

你可以使用 xHook、HookZz 或其他感兴趣的 Unidbg 内置 Hook 方案，你可能会觉得这么做有一些浪费，很多应该释放的内存没有得到释放，那么也可以根据报错，对报错的待释放内存做处理，比如只有指针地址是 0x40175000、0x40176000 的两处释放失败时，代码如下。

```java
public void patchFree(){
    IWhale whale = Whale.getInstance(emulator);
    Symbol free = emulator.getMemory().findModule("libc.so").findSymbolByName("free");
    whale.inlineHookFunction(free, new ReplaceCallback() {
        @Override
        public HookStatus onCall(Emulator<?> emulator, long originFunction) {
            System.out.println("WInlineHookFunction free=" + emulator.getContext().getPointerArg(0));
            long addr = emulator.getContext().getPointerArg(0).peer;
            if(addr == 0x40175000 | addr == 0x40176000){
                return HookStatus.LR(emulator, 0);
            }else {
                return HookStatus.RET(emulator, originFunction);
            }
        }
    });
}
```

## 三、munmap

简单谈谈 munmap ，它的实现位于`src/main/java/com/github/unidbg/spi/AbstractLoader.java`。它出现的频率太高了，这里找个样本展示一下。

样本：[📎wanggetong.7z](https://www.yuque.com/attachments/yuque/0/2022/7z/707476/1669621710877-4e73096c-3db9-4281-a176-2b94e3d45d56.7z)，Unidbg 示例代码如下

```java
package com.guli;

import com.github.unidbg.AndroidEmulator;
import com.github.unidbg.arm.backend.Unicorn2Factory;
import com.github.unidbg.linux.android.AndroidEmulatorBuilder;
import com.github.unidbg.linux.android.AndroidResolver;
import com.github.unidbg.linux.android.dvm.AbstractJni;
import com.github.unidbg.linux.android.dvm.DalvikModule;
import com.github.unidbg.linux.android.dvm.DvmObject;
import com.github.unidbg.linux.android.dvm.VM;
import com.github.unidbg.memory.Memory;

import java.io.File;

public class WangGeTong extends AbstractJni {
    private final AndroidEmulator emulator;
    private final DvmObject BlackBox;
    private final VM vm;

    public WangGeTong() {
        emulator = AndroidEmulatorBuilder
                .for32Bit()
                .addBackendFactory(new Unicorn2Factory(true))
                .build();

        Memory memory = emulator.getMemory();
        memory.setLibraryResolver(new AndroidResolver(23));
        vm = emulator.createDalvikVM(new File("unidbg-android/src/test/resources/guli/wanggetong.apk"));
        vm.setJni(this);
        vm.setVerbose(true);
        DalvikModule dm = vm.loadLibrary("msec", true);
        BlackBox = vm.resolveClass("com.autohome.mainlib.common.util.BlackBox").newObject(null);
        dm.callJNI_OnLoad(emulator);
    }

    public static void main(String[] args) {
        WangGeTong wgt = new WangGeTong();
    }
}
```

运行报错

![](https://cdn.nlark.com/yuque/0/2022/png/707476/1669621682387-cd992423-267f-4e18-a704-270cc7a2ac02.png)

很多人会问我，JNI version:0xFFFFFFFF 是什么意思，是不是 Unidbg 不支持样本所采用的 JNI 版本？这其实就是基础知识不扎实的表现。

让我们回忆一下 JNI_OnLoad 的开发习惯，比如像下面这样。

```java
//注册函数映射
JNIEXPORT jint JNICALL JNI_OnLoad(JavaVM* vm, void* reserved) {
    JNIEnv *pEnv = NULL;
    //获取环境
    jint ret = vm->GetEnv((void**) &pEnv, JNI_VERSION_1_6);
    if (ret != JNI_OK) {
        LOGE("jni_replace JVM ERROR:GetEnv");
        return -1;
    }
    //在{}里面进行方法映射编写，第一个是java端方法名，第二个是方法签名，第三个是c语言形式签名（括号内表示方法返回值）
    JNINativeMethod g_Methods[] = {{"jniOnLoadTest", "()V", (void*) onLoadTest},
                                   {"jniOnload1", "(Lzqc/com/example/Person;)Ljava/lang/String;", (jstring*)onloadTest1}
    };
    jclass cls = pEnv->FindClass("zqc/com/example/NativeTest");
    if (cls == NULL) {
        LOGE("FindClass Error");
        return -1;
    }
    //动态注册本地方法
    ret = pEnv->RegisterNatives(cls, g_Methods,sizeof(g_Methods) / sizeof(g_Methods[0]));
    if (ret != JNI_OK) {
        LOGE("Register Error");
        return -1;
    }
    //返回版本
    return JNI_VERSION_1_6;
}
```

JNI_OnLoad 的返回值必须是 JNI 版本信息，目前支持 1.4、1.6、1.8 三个版本，当返回三者之一时表示一切正常，如果想表示 JNI_OnLoad 处理出错的意思，只需要返回其他值即可，按照惯例一般返回 -1，也就是十六进制的 0xFFFFFFFF，比如上面贴的 demo 代码就是这样。

因此这里 Unidbg 的报错，并不是说样本真采用了什么其他 JNI 版本，而只是表明，样本在早先的 JNI_OnLoad 处理里遇到了问题，直接走到了`return -1`。

所以解决它的问题就是日志往上翻，找最早的报错在哪里

![](https://cdn.nlark.com/yuque/0/2022/png/707476/1669622361365-eb2704ac-f4db-4bc1-8378-47f434a65919.png)

可以看到 munmap 报错，点进去看看，发现是下图这样一个位置。

```java
if (segment == null || segment.size < aligned) {
    throw new IllegalStateException("munmap aligned=0x" + Long.toHexString(aligned) + ", start=0x" + Long.toHexString(start));
}
```

表面是 munmap 的处理有问题，本质上依然是 Unidbg 在内存管理、模块加载相关的处理逻辑上不健全，所以这里主动抛出了一个异常。

如果 SO 加固，比如加个 UPX 壳，那么往往就会遇到这个报错，如果你只是想绕过这个问题，那么只需要注释掉这个`throw`即可，如果你想彻底解决 Unidbg 在这个逻辑下的问题，那么需要对比 Unidbg ElfLoader 逻辑和 Linker 的差异，找到这个问题的根源。

我这里简单注释一下

![](https://cdn.nlark.com/yuque/0/2022/png/707476/1669622543541-9d17ec39-4c40-497b-982b-f32adc24da21.png)

然后运行就进入了正常的补 JNI 逻辑。

![](https://cdn.nlark.com/yuque/0/2022/png/707476/1669622562232-64e3b89d-6e03-46f4-b005-4ec223ff9d8b.png)

感兴趣的读者可以自行尝试。

mmap 和 munmap 相关的问题，在 Unidbg 里很容易出现，而且往往都是发生在加固 SO 的解密和内存释放阶段，这里要阐述两点。

一是这说明 Unidbg 在内存加载方面的处理逻辑做有不到位，有漏洞。

二是不能因此认为"Unidbg 不能处理加固 SO"，只不过是 SO 加固和解密时会更多的使用到系统层的各种功能和特性，而 Unidbg 对它们的支持度不那么高，相对来说更容易出问题罢了。




