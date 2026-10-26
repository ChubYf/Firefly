---
title: unidbg的其他问题
published: 2026-10-19
description: unidbg的其他问题
image: ./cover.jpg
tags: [逆向]
category: unidbg
draft: false
---



[unidbg环境冲突](https://www.yuque.com/docs/share/73ee15f5-6ec4-4f68-be14-a05af2568ab5?#%20%E3%80%8A%E7%8E%AF%E5%A2%83%E5%86%B2%E7%AA%81%E3%80%8B  )
[unidbg函数的副作用](https://www.yuque.com/docs/share/64ff2989-5bdb-4c27-abd9-8c741e0f3acc?#%20%E3%80%8A%E5%87%BD%E6%95%B0%E7%9A%84%E5%89%AF%E4%BD%9C%E7%94%A8%E3%80%8B  )
[unidbg函数初始化问题一](https://www.yuque.com/docs/share/1dce3d7c-fb24-4fe4-a3a7-2f9406ab517c?#%20%E3%80%8AUnidbg%20%E5%87%BD%E6%95%B0%E5%88%9D%E5%A7%8B%E5%8C%96%E9%97%AE%E9%A2%98%EF%BC%88%E4%B8%80%EF%BC%89%E3%80%8B  )
[unidbg函数初始化问题二](https://www.yuque.com/docs/share/e7f09b78-fc7d-41ba-be1b-5b05ece99936?#%20%E3%80%8AUnidbg%20%E5%87%BD%E6%95%B0%E5%88%9D%E5%A7%8B%E5%8C%96%E9%97%AE%E9%A2%98%EF%BC%88%E4%BA%8C%EF%BC%89%E3%80%8B  )



# unidbg环境冲突
## 一、引言

环境冲突是 Unidbg 的结构性问题，它几乎无法被解决，在使用 Unidbg 处理复杂样本时会形成很大阻挠。

## 二、概述

**结构性问题**是一个很好的词，指由于事物的底层机理、结构而出现，并且很难避免和解决的问题。Unidbg 最主要的结构性问题是什么？就是环境冲突。

Unidbg 试图模拟一套 Android Native 可控环境，我们简称它为**模拟环境**，但同时，因为 Unidbg 模拟的不够完善以及模拟本身就无法尽善尽美的特点，所以 Unidbg 常常要依赖外部信息——来自 Android 真实环境的信息，以 JNI 补环境和文件访问这两部分最甚。那么，Unidbg 环境实际上是由模拟环境和 Android 环境融合而来的，这两种环境必然会带来摩擦和冲突，我们举个简单的例子。

假设样本在启动时，会将当前进程的 PID，编码加密后放到某一个文件中。在 Native 层，获取 PID 并解码文件，比对进程 ID 是否一致，一致的话认为环境无异常，走正常逻辑，否则函数返回失败或提高风控等级。

详细一下 Unidbg 里如何处理这个问题

Native 中获取 PID，大抵上底层总会依赖于`getpid`系统调用，看看 Unidbg 如何实现它。

```java
// 获取当前JVM进程的PID
String name = ManagementFactory.getRuntimeMXBean().getName();
String pid = name.split("@")[0];
// 解析为数字，取低 15 位
this.pid = Integer.parseInt(pid) & 0x7fff;
```

Unidbg 返回当前 JAVA 进程的 ID，并将获取到的值 和 0x7FFF 做与运算。按照与运算的规则，这相当于舍去高位，只保留较低的 15 比特。这么做的原因在于，Linux 以及基于它的 Android 上，默认 PID 最大只可达 0x8000。读者可以在 adb shell 中验证这一点

```java
polaris:/ $ cat /proc/sys/kernel/pid_max
32768
```

但是 Mac 上进程 ID 默认可达 100000，Windows 上更是不做限制。

因此出于规范的角度，Unidbg 抹去了当前 Pid 的高位，确保其小于 PID_MAX，更符合 Android 规范，避免超标。

这个 PID 的获取，来自于上文说过的 Unidbg 的模拟环境。

当发生文件访问时，我们会从真机 copy 这个文件，但很难意识到它是编码后的 pid，事实上这只是一个简单的假设，实际上可能更复杂。这个文件，来自于真实环境。

那么在样本的处理逻辑里，就会对不上，文件解析的 pid 和 Native 获取的 pid 有差异。

这个问题可以复杂很多倍，不只是 pid，模拟环境和真实环境的每一点差异都可能带来问题，除非 Unidbg 完全依赖 Android 环境或完全依赖模拟，这就是 Unidbg 最主要的一个结构性问题。

## 三、尾声

模拟环境里得到的信息和 JNI、文件访问获得到的真实 Android 环境信息产生了差异，如何处理？事实上，很多时候我们都没办法定位到问题点，因为复杂样本会大量获取信息，难不成对所有 1000 个来自模拟环境的信息，以及另 1000 个来自 Android 真实环境的信息一一比对？这是很困难的事，目前几乎没法克服这个结构性问题。





# unidbg函数副作用
## 一、引言

函数的副作用是 Unidbg 补环境的一个重要困境。

## 二、概述

首先谈一下函数副作用这个概念，**函数副作用指函数在正常工作任务之外对外部环境所施加的影响**。它是计算机科学里的一个常见词，在高级语言的函数、低级语言的汇编指令层级都可以去讨论这个副作用。

这个概念有些抽象，具体点说，副作用就是除了计算和返回函数结果，函数还产生了某些独立于当前函数的、全局性的影响。举个例子，下面是一个 a 函数

```java
public int a(){
    return 1;
}
```

如果我们调用 a 函数

```java
int ret = a();
```

那么它等价于

```java
int ret = 1;
```

编译器都常常这么去做优化，但如果我们改一下 a 函数呢？

```java
public int a(){
    System.out.println("hello world");
    return 1;
}
```

此时`int ret = a();`还能等价为`int ret = 1;`吗？当然是不行的。

修改后的 a 函数存在副作用，它向控制台输出了"hello world"，这是除了函数结果以外的信息，是和系统的交互。

对全局/外部变量的修改、文件操作、数据库操作、打印输出等操作都可以认为是函数副作用的表现。函数副作用无所谓好坏，它仅仅用于表述函数是否对外部环境产生了影响。

听起来它和 Unidbg 没什么关系，其实不然。

## 三、Unidbg 中的函数副作用

在 Unidbg 补环境中，很多函数本应该对 Android 系统或其他的外部环境造成函数副作用，但我们没有甚至是无法去做处理。

举个例子，下面是我摘取的一个 JNI 补环境片段。

```java
@Override
public void callStaticVoidMethod(BaseVM vm, DvmClass dvmClass, String signature, VarArg varArg) {
    switch (signature){
        case "com/alibaba/wireless/security/open/edgecomputing/ECMiscInfo->registerAppLifeCyCleCallBack()V":{
            return;
        }
        case "com/alibaba/one/android/sdk/OneMain->initialize(Landroid/content/Context;)V":{
            return;
        }
    }
    super.callStaticVoidMethod(vm, dvmClass, signature, varArg);
}
```

样本没有返回值，所以我们的处理是直接 return，试想一下，它的实际函数实现里，真的什么就没做吗？没有返回值，就意味着函数没有任何逻辑吗？

这显然是否定的，**它可能修改或者说维护了一个全局变量，或者发送了一个网络请求，或者上传了一条日志等等**。将补 JNI 或补其他的东西，简单理解为扁平的返回函数的值，那么很可能就会因为没有对环境施加同等的副作用，导致逻辑错误，无法和真机上的执行获得同等效果。

如果安全开发人员有意利用函数副作用，那就更糟糕了。

假如关键函数总会通过 JNI 调用 JAVA 层的某些函数，样本可以统计这些函数的调用，作为日志输出或上传，使用 Unidbg 模拟执行并发起访问，云端发现这台设备总是访问结果，但函数计算过程中所涉及的 JAVA 函数一条日志都没上报，这显然是一个奇怪且不可信的设备环境。这种处理方式有时也叫基于日志流的风控。

## 四、尾声

这个问题很难解决，模拟并非万能。




# unidbg初始化问题介绍
## 一、引言

函数初始化问题是 Unidbg 补环境最核心的问题之一，是处理复杂样本、复杂函数时的一大拦路虎，这个系列对它做探讨，本篇介绍这个话题相关的主要概念，之后的文章则是对应的实践。

## 二、相关概念

什么叫函数初始化？在 Unidbg 里为什么会是个大问题？我们可以通过一个简单的样例，以小见大，搞清楚它的实质。样本是[📎com.aikucun.akapp_6.0.6_60006.apk.7z](https://www.yuque.com/attachments/yuque/0/2022/7z/707476/1662381978844-bbea82cf-350c-492a-9b3a-1266d5578a51.7z)。

JADX 反编译 APK，找到`com.mengxiang.arch.security.MXSecurity`类

```java
package com.mengxiang.arch.security;

import android.content.Context;
import com.akc.im.akc.util.HttpUtil;
import kotlin.Metadata;
import kotlin.jvm.JvmStatic;

public final class MXSecurity {
    public static final MXSecurity f44041a = new MXSecurity();

    static {
        System.loadLibrary("mx");
    }

    private MXSecurity() {
        
    }

    @JvmStatic
    @NotNull
    public static final native String getAppId();

    @JvmStatic
    public static final native int init(@NotNull Context context, boolean z2);

    @JvmStatic
    @NotNull
    public static final native String signV1(@NotNull String str, @NotNull String str2, @NotNull String str3);

    @JvmStatic
    @NotNull
    public static final native String signV2(@NotNull String str);

    @JvmStatic
    @NotNull
    public static final native String signV3(@NotNull String str, @NotNull String str2, @NotNull String str3, @NotNull String str4);
}
```

`init`函数是初始化函数，libmx.so 加载后，如果不先调用`init`函数，那么访问`signV1`、`signV2`、`signV3`、`getAppId`这些函数都会失败。

先看它如何实现这种控制效果，全局变量是最常见的方式，以`signV1`为例。

```c
__int64 __fastcall Java_com_mengxiang_arch_security_MXSecurity_signV1(__int64 *a1, __int64 a2, __int64 a3, __int64 a4, __int64 a5)
{
  __int64 v5; // x8
  const char *v9; // x22
  const char *v10; // x21
  __int64 v11; // x23
  __int64 v12; // x24
  __int64 v13; // x25
  __int64 v14; // x26
  __int64 v15; // x27
  __int64 v16; // x28
  __int64 v17; // x20
  __int64 v18; // x20
  _QWORD *v19; // x23
  __int64 v20; // x8
  char *v21; // x0
  char *v22; // x0
  char *v23; // x0
  char *v24; // x0
  char *v25; // x0
  char *v26; // x0
  __int64 v27; // x8
  char *v28; // x0
  char *v29; // x0
  char *v30; // x0
  size_t v31; // w20
  __int64 v32; // x21
  char *v33; // x1
  __int64 (__fastcall *v34)(__int64 *, char *); // x2
  char *src; // [xsp+8h] [xbp-58h]

  v5 = *a1;
  if ( (byte_32051 & 1) != 0 )
  {
    v9 = (const char *)(*(__int64 (__fastcall **)(__int64 *, __int64, _QWORD))(v5 + 1352))(a1, a4, 0LL);
    src = (char *)(*(__int64 (__fastcall **)(__int64 *, __int64, _QWORD))(*a1 + 1352))(a1, a5, 0LL);
    v10 = (const char *)(*(__int64 (__fastcall **)(__int64 *, __int64, _QWORD))(*a1 + 1352))(a1, a3, 0LL);
    v11 = __strlen_chk("appid=", 7u);
    v12 = __strlen_chk("&noncestr=", 0xBu);
    v13 = strlen(v9);
    v14 = __strlen_chk("&timestamp=", 0xCu);
    v15 = strlen(src);
    v16 = __strlen_chk("&secret=", 9u);
    v17 = __strlen_chk("&url=", 6u);
    v18 = v11 + v12 + v13 + v14 + v15 + v16 + v17 + strlen(v10) + 41;
    v19 = (_QWORD *)operator new[](v18);
    memset(v19, 0, v18);
    *v19 = 0LL;
    strcat((char *)v19, "appid=");
    v20 = 32LL;
    if ( isDebug )
      v20 = 16LL;
    v21 = strcat((char *)v19, *(const char **)(lpAppInfo + v20));
    v22 = strcat(v21, "&noncestr=");
    v23 = strcat(v22, v9);
    v24 = strcat(v23, "&timestamp=");
    v25 = strcat(v24, src);
    v26 = strcat(v25, "&secret=");
    v27 = 40LL;
    if ( isDebug )
      v27 = 24LL;
    v28 = strcat(v26, *(const char **)(lpAppInfo + v27));
    v29 = strcat(v28, "&url=");
    v30 = strcat(v29, v10);
    v31 = strlen(v30);
    v32 = (*(__int64 (__fastcall **)(__int64 *, _QWORD))(*a1 + 1408))(a1, v31);
    (*(void (__fastcall **)(__int64 *, __int64, _QWORD, _QWORD, _QWORD *))(*a1 + 1664))(a1, v32, 0LL, v31, v19);
    v33 = (char *)digest(a1, ALGORITHM_SHA1[0], v32);
    v34 = *(__int64 (__fastcall **)(__int64 *, char *))(*a1 + 1336);
  }
  else
  {
    v34 = *(__int64 (__fastcall **)(__int64 *, char *))(v5 + 1336);
    v33 = "";
  }
  return v34(a1, v33);
}
```

它根据 byte_32051 这个全局变量是否为 1 ，确定走正常逻辑还是返回空。

![](https://cdn.nlark.com/yuque/0/2022/png/707476/1668000707168-cc217281-64f3-4f66-9aab-8aea56b46b05.png)

查看对 byte_32051 的交叉引用，发现每个函数都用到了它。区别仅在于`signV1`、`signV2`等函数是用它来判断

```c
__int64 __fastcall Java_com_mengxiang_arch_security_MXSecurity_signV2(__int64 *a1, __int64 a2, __int64 a3)
{
  __int64 v3; // x8
  const char *v5; // x20
  size_t v6; // w21
  __int64 v7; // x22
  char *v8; // x1
  __int64 (__fastcall *v9)(__int64 *, char *); // x2

  v3 = *a1;
  if ( (byte_32051 & 1) != 0 )
  {
    v5 = (const char *)(*(__int64 (__fastcall **)(__int64 *, __int64, _QWORD))(v3 + 1352))(a1, a3, 0LL);
    v6 = strlen(v5);
    v7 = (*(__int64 (__fastcall **)(__int64 *, _QWORD))(*a1 + 1408))(a1, v6);
    (*(void (__fastcall **)(__int64 *, __int64, _QWORD, _QWORD, const char *))(*a1 + 1664))(a1, v7, 0LL, v6, v5);
    v8 = (char *)digest(a1, ALGORITHM_MD5, v7);
    v9 = *(__int64 (__fastcall **)(__int64 *, char *))(*a1 + 1336);
  }
  else
  {
    v9 = *(__int64 (__fastcall **)(__int64 *, char *))(v3 + 1336);
    v8 = "";
  }
  return v9(a1, v8);
}
```

只有`init`函数是对它赋值

![](https://cdn.nlark.com/yuque/0/2022/png/707476/1668000832003-9f359c17-307f-497b-9138-c4da6740cf0e.png)

具体一点说，只有`init`函数中对运行环境做 Apk 签名校验以及包名校验都无误后，才会赋值为 1。

关于这个简单样本，我们可以总结三点。

1. 初始化函数是必须先于目标函数执行，才能让目标函数顺利函数的函数。比如这个样本的`init`函数就是`signV1`等函数的初始化函数。
2. 初始化函数常常通过全局变量干扰或参与后续的函数执行。
3. 此处设置初始化函数的为了安全考虑，具体而言是做环境检测，通过签名校验、包名校验判断 SO 是否运行在自身进程内。

我们对这三点做进一步讨论。

## 2.1 设计意图

首先谈谈初始化函数的设计意图，在大体上它有三方面作用。

一是安全校验，即检测 SO 运行环境是否安全且符合预期，最常见的就是签名校验、进程校验、包名校验外。

二是 SO 初始化，做一些代码和数据的初始化工作，以及缓存高频使用的 JNI 字段、ID，Android 官方就有这一建议，详见[官方教程](https://developer.android.com/training/articles/perf-jni?hl=zh-cn#jclass-jmethodid-and-jfieldid)。

```java
/*
 * We use a class initializer to allow the native code to cache some
 * field offsets. This native function looks up and caches interesting
 * class/field/method IDs. Throws on failure.
 */
private static native void nativeInit();

static {
    nativeInit();
}
```

三是某些函数的初始化，相比较第二类，它的区别主要在于服务或者说作用的范围，第三类主要是为单个函数准备数据和内容。

我们举个例子，区分第二类和第三类。样本是 [📎xhs.7z](https://www.yuque.com/attachments/yuque/0/2022/7z/707476/1668002928881-8fb68e66-f25e-4bbe-9489-4bd6255ac81b.7z)。

JADX 反编译样本，找到 com.xingin.shield.http.XhsHttpInterceptor 类。

```java
package com.xingin.shield.http;

import android.util.Log;
import j.z.a1.a.b;
import java.io.IOException;
import okhttp3.Interceptor;
import okhttp3.Request;
import okhttp3.Response;

public class XhsHttpInterceptor implements Interceptor {
    public static String TAG = "XhsHttpInterceptor";
    public long cPtr;
    public b<Request> predicate;

    static {
        initializeNative();
    }

    public XhsHttpInterceptor(String str, b<Request> bVar) {
        this.cPtr = initialize(str);
        this.predicate = bVar;
    }

    public static native void initializeNative();

    public static XhsHttpInterceptor newInstance(String str) {
        return new XhsHttpInterceptor(str, null);
    }

    public native void destroy(long j2);

    public void finalize() throws Throwable {
        super.finalize();
        destroy(this.cPtr);
    }

    public native long initialize(String str);

    @Override // okhttp3.Interceptor
    public Response intercept(Interceptor.Chain chain) throws IOException {
        Response intercept;
        long currentTimeMillis = ContextHolder.sJavaLogEnabled ? System.currentTimeMillis() : 0L;
        b<Request> bVar = this.predicate;
        if (bVar != null && !bVar.test(chain.request())) {
            intercept = chain.proceed(chain.request());
        } else {
            intercept = intercept(chain, this.cPtr);
        }
        if (ContextHolder.sJavaLogEnabled) {
            String str = TAG;
            StringBuilder sb = new StringBuilder();
            sb.append("cost");
            sb.append(System.currentTimeMillis() - currentTimeMillis);
            Log.i(str, sb.toString());
        }
        return intercept;
    }

    public native Response intercept(Interceptor.Chain chain, long j2) throws IOException;

    public static XhsHttpInterceptor newInstance(String str, b<Request> bVar) {
        return new XhsHttpInterceptor(str, bVar);
    }
}
```

静态代码块里的`initializeNative`函数就是 SO 初始化函数，它缓存了大量的`MethodID`、`FieldID`供后续使用。

![](https://cdn.nlark.com/yuque/0/2022/png/707476/1668003090849-61db6b6a-a3e0-44a8-a141-4b640d3bae0a.png)

`initialize`就是`intercept`的初始化函数，代码逻辑很清晰的反映了这一点。它的返回结果`cPtr`是`intercept`函数的第二个参数。

## 2.2 问题来源

为什么初始化函数在 Unidbg 中里会成为一个问题？当我们想要调用某个函数时，在真实环境里，只需要 Frida 附加应用进程，直接调用目标函数即可。这是因为所需的初始化函数，很可能在早先已经执行过了，进程环境已经是初始化好了的状态。

但在 Unidbg 里，运行环境实际上就是目标 SO 加载到内存，执行完 JNI_OnLoad 后的状态。正常环境里，会在 SO 加载完后，在 JAVA 层做的这些初始化调用和处理逻辑，Unidbg 因为没法执行 DEX，所以没法感知到这些步骤，更别提实际去执行这些初始化步骤了。

因为相比较真实 Android 环境上的函数调用，在 Unidbg 里调用一个函数，需要手动补齐目标 SO 以及目标函数的初始化函数。如果你用 Unidbg 执行目标函数时，直接报错，或返回空内容，或返回相较于真实情况较短长度的结果，那么就要往初始化问题上思考。（但是，初始化问题并不是造成这些现象的唯一原因。）

问题就是这么个问题，如何分析初始化函数？

`MXSecurity`类的`init`函数是初始化函数，这是静态分析 JAVA 代码（它的名字很显眼），以及静态分析 SO 里的代码，最终确认出来的。

`XhsHttpInterceptor`类里的初始化函数，是静态分析 JAVA 代码逻辑得出的。

似乎分析初始化并不是一个复杂的事，但事实上初始化函数并不总是那么容易找。

## 2.3 初始化函数链

有时候找初始化函数确实很简单，比如上面的那些例子，又比如下面这个例子。静态分析函数名和静态代码块的布置，就可以很轻松的分析出来。

![](https://cdn.nlark.com/yuque/0/2022/png/707476/1662395733701-c2b2ca0b-f908-44ad-ae85-0b72852a6583.png)

但是，这件事并不总是这么简单。举几个例子。

doCommandNative 函数是否存在初始化函数，如果有的话具体是哪个或哪些呢？

```java
package com.kuaishou.android.security.internal.dispatch;

public class JNICLibrary {
    public static native void dCaBk(int i8, String str, byte[] bArr, int i9);

    public static native Object doCommandNative(int i8, Object[] objArr);

    public static native long gDBF();

    public static native String[] gDGI();

    public static native long gKSF();
}
```

a 函数是否存在初始化函数，如果有的话具体是哪个或哪些呢？

```java
package ms.bd.c;

public class k {
    public static native Object a(int i, int i2, long j, String str, Object obj);

    static Object b(int i, int i2, long j, String str, Object obj) {
        return b.b(i, i2, j, str, obj);
    }
}
```

main 函数是否存在初始化函数，如果有的话具体是哪个或哪些呢？

```java
private static native Object[] main(int i, Object[] objArr);

public static String main1(int i, Object[] objArr) {
    Object[] objArr2 = {Integer.valueOf(i), objArr};
    ChangeQuickRedirect changeQuickRedirect2 = changeQuickRedirect;
    if (PatchProxy.isSupport(objArr2, null, changeQuickRedirect2, true, "baaae4aeeb6bb4e055f31454690e1c03", 6917529027641081856L)) {
        return (String) PatchProxy.accessDispatch(objArr2, null, changeQuickRedirect2, true, "baaae4aeeb6bb4e055f31454690e1c03");
    }
    try {
        Object[] main = main(i, objArr);
        return (main == null || !(main[0] instanceof String)) ? "" : (String) main[0];
    } catch (Throwable th) {
        MTGuardLog.setErrorLogan(th);
        return "";
    }
}
```

读者恐怕也一是给不出答案，事实上，在分析初始化函数时，需要依赖对样本 JAVA、Native 代码的静态分析，以及通过 Frida、Xposed 等工具去 Hook、Call，观察函数的执行流，即对应的动态分析，以及一些初始化函数分析的经验，主要是下面三条。

1. 初始化函数未必有很好的命名或很清晰的代码逻辑。
2. 初始化函数未必只有一个，可能有两三个甚至更多，甚至是一条调用链。

比较典型的就是`sgmain`，漁滒有一篇很好的[文章](https://www.52pojie.cn/thread-1680816-1-1.html)，对它的初始化调用链画了一个好图。

![](https://cdn.nlark.com/yuque/0/2022/png/707476/1663212545946-940e1d27-1eea-4f58-9006-7d0e6d1765d5.png)

3. 初始化函数未必是目标函数以外的另一个函数，也可能是自身函数，只不过传参不同。

从下篇文章开始，我们通过实例分析来展示其中的方法论和分析思路。

## 2.4 中段执行

函数初始化问题，当然不局限于“如何找到函数的初始化函数，然后执行它们”，很多时候我们希望不执行初始化函数，但又顺利执行目标函数，我们将这称为中段执行。中段执行的好处很多，很多场景里可以使用它。

举一些可使用的场景

- 使用 Unidbg 模拟执行 metasec_ml 时，执行流会在 JNI_OnLoad 里死循环，问题的产生原因是 Unidbg 的多线程逻辑不够完善，抛开如何修复这个问题不谈，我们是否可以不执行 JNIOnLoad 直接跑目标函数？
- 某个函数拥有极其复杂的初始化调用链，其中的一环或多个步骤无法找齐或良好处理，想直接执行这个目标函数。
- 函数不存在初始化函数，它是一个设备指纹生成函数，具体逻辑分为收集设备信息与加密生成唯一 ID 两个步骤。事先通过静态分析、 Frida Hook 等方式找到了它的加密函数位置，我们不想用 Unidbg 跑整个代码执行流，函数的前一个逻辑所涉及的 JNI 补环境工作量太大了，希望直接执行后一部分逻辑，借助 Unidbg 辅助分析加密函数的代码逻辑。
- 分析 VMP 处理逻辑，不想考虑外层环境，只希望填入正确的虚拟机上下文，使得虚拟机正常执行逻辑，并可观测。

![](https://cdn.nlark.com/yuque/0/2022/png/707476/1662886515974-651a9100-d0bd-4b4c-89ce-2541285ada47.png)这些场景里，中段执行都能带来很大的帮助。

## 三、尾声

本篇对函数初始化问题的产生、内涵、外延都做了比较准确但精练的介绍，下篇开始结合实例做分析。



# unidbg初始化问题示例
## 一、引言

本篇讨论的实例是[📎mtold.7z](https://www.yuque.com/attachments/yuque/0/2022/7z/707476/1668301839424-1da9fe74-08b9-411b-99c0-94403ff25e36.7z)。

## 二、问题分析

JADX 反编译目标 APK，找到`com.meituan.android.common.mtguard.NBridge`类，其中的`main`方法就是目标。

```java
public static native Object[] main(int i, Object[] objArr);
```

它的调用模式很常见，参数 1 指代分支，参数 2 是变长数据，指代这个分支所需的数据。

![](https://cdn.nlark.com/yuque/0/2022/png/707476/1668311533060-268f8c1f-33f7-4509-8f83-f63f2a946e22.png)

在众多的交叉引用里，我们关注于 203 的这一款。

首先对这个函数 做 Spawn Hook，使用 JADX 的右键生成 Frida 代码。

```java
let NBridge = Java.use("com.meituan.android.common.mtguard.NBridge");
NBridge["main"].implementation = function (i, objArr) {
  console.log('main is called' + ', ' + 'i: ' + i + ', ' + 'objArr: ' + objArr);
  let ret = this.main(i, objArr);
  console.log('main ret value is ' + ret);
  return ret;
};
```

稍做封装

```java
function hook(){
    Java.perform(function() {
        let NBridge = Java.use("com.meituan.android.common.mtguard.NBridge");
        NBridge["main"].implementation = function (i, objArr) {
            console.log('main is called' + ', ' + 'i: ' + i + ', ' + 'objArr: ' + objArr);
            let ret = this.main(i, objArr);
            console.log('main ret value is ' + ret);
            return ret;
        };
    })
}

hook();
```

Frida Spawn 唤起进程做 Hook，以免错过较早时机的 main 调用。发现似乎存在某些问题，拦截不到较早时机的信息，这是一个常见问题，我们放后面去说。

改为使用 Frida 附加进程，手动调用 hook 函数

```java
main is called, i: 203, objArr: 9b69f861-e054-4bc4-9daf-d36ae205ed3e,[B@1aa7b4c,2
main is called, i: 203, objArr: 9b69f861-e054-4bc4-9daf-d36ae205ed3e,[B@fefd795,2
main is called, i: 203, objArr: 9b69f861-e054-4bc4-9daf-d36ae205ed3e,[B@f0c60aa,2
main ret value is {"a0":"1.5","a1":"9b69f861-e054-4bc4-9daf-d36ae205ed3e","a2":"99c82af6175158244e0accd223bed32dab663365","a3":2,"a4":1668312094,"a5":"MmOEqYGvRGTZKxHlPvsVZGHcTddlfo0drilXh8C0F9Bsop5NRDVu7yqoj
g06v7hcSmmTIAkWvWW0b7wvohciqbVkMSWdJyWc1Uw4XRP/w18V9t/+5d8HK1yW1JruPSYC6Ktk7vdkv89NimGu9ldhrDSthdPzhM4d1svfiyNNs7vR2mJi5KCKsEvTr4YiKtA0y9zKNT8CAstiA3rz5ezJhp0SLWNRrIVtt2M0XBQdNywD7Pt/7tQZf2Jn9dhqamkp+9RGWvKHS
yg6dJyFq+XtMmTsDax0VF13XoI3QEVRqrrAn96wI45M9fP73FTb","a6":0,"d1":"becd3c63ed75fbbbd3493afbcb5a31b1adb6c975"}
main ret value is {"a0":"1.5","a1":"9b69f861-e054-4bc4-9daf-d36ae205ed3e","a2":"cda86097e2ef8e0b07be1eb12aa933bc45740d53","a3":2,"a4":1668312094,"a5":"MmOEqYGvRGTZKxHlPvsVZGHcTddlfo0drilXh8C0F9Bsop5NRDVu7yqoj
g06v7hcSmmTIAkWvWW0b7wvohciqbVkMSWdJyWc1Uw4XRP/w18V9t/+5d8HK1yW1JruPSYC6Ktk7vdkv89NimGu9ldhrDSthdPzhM4d1svfiyNNs7vR2mJi5KCKsEvTr4YiKtA0y9zKNT8CAstiA3rz5ezJhp0SLWNRrIVtt2M0XBQdNywD7Pt/7tQZf2Jn9dhqamkp+9RGWvKHS
yg6dJyFq+XtMmTsDax0VF13XoI3QEVRqrrAn96wI45M9fP73FTb","a6":0,"d1":"4db2576c4f0e55c842396dc04238957c013fe90e"}
main ret value is {"a0":"1.5","a1":"9b69f861-e054-4bc4-9daf-d36ae205ed3e","a2":"0aa6ada0f53dc57419f3e9a18730497168a16937","a3":2,"a4":1668312094,"a5":"MmOEqYGvRGTZKxHlPvsVZGHcTddlfo0drilXh8C0F9Bsop5NRDVu7yqoj
g06v7hcSmmTIAkWvWW0b7wvohciqbVkMSWdJyWc1Uw4XRP/w18V9t/+5d8HK1yW1JruPSYC6Ktk7vdkv89NimGu9ldhrDSthdPzhM4d1svfiyNNs7vR2mJi5KCKsEvTr4YiKtA0y9zKNT8CAstiA3rz5ezJhp0SLWNRrIVtt2M0XBQdNywD7Pt/7tQZf2Jn9dhqamkp+9RGWvKHS
yg6dJyFq+XtMmTsDax0VF13XoI3QEVRqrrAn96wI45M9fP73FTb","a6":0,"d1":"2f49a7bc71ba8053edd78adcd604ab4ddfd20bcb"}
```

参数 1 是 203 的调用是我们的目标，它的 objArr 似乎由字符串序列、字节数组以及 Integer 对象组成，静态分析可以辅助确认这一点。

![](https://cdn.nlark.com/yuque/0/2022/png/707476/1668312335735-682b8ce6-262b-4b7a-90e5-2e365812f8b2.png)

想办法打印一下这个字节数组，先假设它是字符串，修改代码

```java
function hook(){
  Java.perform(function() {
    let NBridge = Java.use("com.meituan.android.common.mtguard.NBridge");
    var ByteString = Java.use("com.android.okhttp.okio.ByteString");
    NBridge["main"].implementation = function (i, objArr) {
      console.log('main is called' + ', ' + 'i: ' + i + ', ' + 'objArr: ' + objArr);
      let ret = this.main(i, objArr);
      if(i === 203){
        var buffer = Java.array('byte', objArr[1]);
        console.log(objArr[1]+":" + ByteString.of(Java.array('byte', buffer)).utf8());
      }
      console.log('main ret value is ' + ret);
      return ret;
    };
  })
}

// hook();
```

看起来运行结果不错

```java
main is called, i: 203, objArr: 9b69f861-e054-4bc4-9daf-d36ae205ed3e,[B@774bdd9,2
[B@774bdd9:POST /mtapi/v8/channel/rcmdboard __reqTraceID=276c9977-5799-4c65-a673-38daa289a6bd&app=0&category_code=910&ci=1&f=android&msid=6ac246a1664949ebb0a4f2714a58851fa1605466697158066131668311899492&navig
ate_type=910&net_stat=0&partner=4&platform=4&poilist_mt_cityid=1&poilist_wm_cityid=110100&preload=0&push_token=dpshebefd64035e6c9fb9ebcee40f06c8383atpu&rank_list_id=2b95d20e8bfa44e2bdf46f00abc97770&req_time=1
668312542236&userid=-1&utm_campaign=AgroupBgroupC0E0Ghomepage_category1_394__a1__c-1024&utm_content=47c9b7a49ea898b1&utm_medium=android&utm_source=xiaomiyuzhuang2&utm_term=1100090405&uuid=00000000000006AC246A
1664949EBB0A4F2714A58851FA160546669715806613&version=11.9.405&version_name=11.9.405&waimai_sign=VbaaJwqV9JEALc9Nv%2FH4egKOC1TvEuUDSZU65rRIJZYlv3FNRvx%2B50qAZN63z9psxr5d%2FUGNhj1S%0AlYkIOBCM1QDGdOXqXa8FPDFAGfc
SjRDlild4pPFGDifbSyQI%2F8k50NM2QAgMK2phcdvA%2B8PCv0n6%0APR5qvRIuaNCALrtie38%3D%0A&wm_actual_latitude=40240957&wm_actual_longitude=116176188&wm_appversion=11.9.405&wm_ctype=mtandroid&wm_did=47c9b7a49ea898b1&wm
_dtype=MIX%202S&wm_dversion=29_10&wm_latitude=40240957&wm_longitude=116176188&wm_mac=02%3A00%3A00%3A00%3A00%3A00&wm_seq=55&wm_uuid=00000000000006AC246A1664949EBB0A4F2714A58851FA160546669715806613&wm_visitid=4
0c7fb83-2377-42fa-833b-e2b2be98e7ce
main ret value is {"a0":"1.5","a1":"9b69f861-e054-4bc4-9daf-d36ae205ed3e","a2":"34a80b171851f2d057764c7e20678502f8a66d50","a3":2,"a4":1668312547,"a5":"W7UP9ExzHugohP881OPwSshxrXW183dVdRlWcj2ZuFVqx+RFJfaYUdSJS
YA2Yy3ldgAEcRayzg/hBmPFhJ15mHnbX0u+DA7gmiJDSK1ycOXdBOBj4d2g4GwFAR+U/07RPftQ0uyy8+5xg/PvPOZW3ElalTAlNSRHaoa6Ntltb5byK9kcZ04Y7AwISs2dEWxd9jm0nC6SrZtkaOlL9THRDqp04fc//ZUgDD0T+K+N22ryQKpHO7yfy1QX19vhFto4saQHGhru1
bAmNk5U3USrzDR3ZvorUhpx3xS0GZBQ3ezsvDN5mIaNycjAlbuL","a6":0,"d1":"7a228c1b14f652e486d89832e285384e118a7698"}
```

基本的介绍结束，接下来分析它是否有初始化函数，你可能会写像下面这样的 Frida Call 代码。

```java
Java.perform(function () {
    let NBridge = Java.use('com.meituan.android.common.mtguard.NBridge');
    var JString = Java.use('java.lang.String');
    var jinteger = Java.use("java.lang.Integer");
    var param_str = JString.$new("POST /mtapi/v8/channel/rcmdboard __reqTraceID=276c9977-5799-4c65-a673-38daa289a6bd&app=0&category_code=910&ci=1&f=android&msid=6ac246a1664949ebb0a4f2714a58851fa1605466697158066131668311899492&navigate_type=910&net_stat=0&partner=4&platform=4&poilist_mt_cityid=1&poilist_wm_cityid=110100&preload=0&push_token=dpshebefd64035e6c9fb9ebcee40f06c8383atpu&rank_list_id=2b95d20e8bfa44e2bdf46f00abc97770&req_time=668312542236&userid=-1&utm_campaign=AgroupBgroupC0E0Ghomepage_category1_394__a1__c-1024&utm_content=47c9b7a49ea898b1&utm_medium=android&utm_source=xiaomiyuzhuang2&utm_term=1100090405&uuid=00000000000006AC246A1664949EBB0A4F2714A58851FA160546669715806613&version=11.9.405&version_name=11.9.405&waimai_sign=VbaaJwqV9JEALc9Nv%2FH4egKOC1TvEuUDSZU65rRIJZYlv3FNRvx%2B50qAZN63z9psxr5d%2FUGNhj1S%0AlYkIOBCM1QDGdOXqXa8FPDFAGfSjRDlild4pPFGDifbSyQI%2F8k50NM2QAgMK2phcdvA%2B8PCv0n6%0APR5qvRIuaNCALrtie38%3D%0A&wm_actual_latitude=40240957&wm_actual_longitude=116176188&wm_appversion=11.9.405&wm_ctype=mtandroid&wm_did=47c9b7a49ea898b1&wm_dtype=MIX%202S&wm_dversion=29_10&wm_latitude=40240957&wm_longitude=116176188&wm_mac=02%3A00%3A00%3A00%3A00%3A00&wm_seq=55&wm_uuid=00000000000006AC246A1664949EBB0A4F2714A58851FA160546669715806613&wm_visitid=0c7fb83-2377-42fa-833b-e2b2be98e7ce")
    var mode = jinteger.valueOf(2);
    var objectArray = Java.array('java.lang.Object',['9b69f861-e054-4bc4-9daf-d36ae205ed3e',param_str.getBytes(),mode]);
    var ret = NBridge.main(203,objectArray)
    console.log("call result:"+ret.toString())
})
```

运行效果符合预期

![](https://cdn.nlark.com/yuque/0/2022/png/707476/1668312975889-a5cdc1f6-3485-4e9a-93a7-fa3e1af62398.png)

在首篇文章里我们说过

在 Unidbg 里，运行环境实际上就是目标 SO 加载进内存，刚执行完 JNI_OnLoad 的状态。

因此在真实 Android 环境里，如果也在目标 SO 执行完 JNI_OnLoad 后调用，可以返回预期结果，就证明不存在常规的初始化函数。

修改代码，来到这个时机

```java
var android_dlopen_ext = Module.findExportByName(null, "android_dlopen_ext");
if (android_dlopen_ext != null) {
  Interceptor.attach(android_dlopen_ext, {
    onEnter: function (args) {
      this.hook = false;
      var soName = args[0].readCString();
      if (soName.indexOf("libmtguard.so") !== -1) {
        this.hook = true;
      }
    },
    onLeave: function (retval) {
      if (this.hook) {
        var jniOnload = Module.findExportByName("libmtguard.so", "JNI_OnLoad");
        Interceptor.attach(jniOnload, {
          onEnter: function (args) {
            console.log("Enter Mtguard JNI OnLoad");
          },
          onLeave: function (retval) {
            console.log("After Mtguard JNI OnLoad");
            call();
          }
        });
      }
    }
  });
}

function call(){
  Java.perform(function () {
    let NBridge = Java.use('com.meituan.android.common.mtguard.NBridge');
    var JString = Java.use('java.lang.String');
    var jinteger = Java.use("java.lang.Integer");
    var param_str = JString.$new("POST /mtapi/v8/channel/rcmdboard __reqTraceID=276c9977-5799-4c65-a673-38daa289a6bd&app=0&category_code=910&ci=1&f=android&msid=6ac246a1664949ebb0a4f2714a58851fa1605466697158066131668311899492&navigate_type=910&net_stat=0&partner=4&platform=4&poilist_mt_cityid=1&poilist_wm_cityid=110100&preload=0&push_token=dpshebefd64035e6c9fb9ebcee40f06c8383atpu&rank_list_id=2b95d20e8bfa44e2bdf46f00abc97770&req_time=668312542236&userid=-1&utm_campaign=AgroupBgroupC0E0Ghomepage_category1_394__a1__c-1024&utm_content=47c9b7a49ea898b1&utm_medium=android&utm_source=xiaomiyuzhuang2&utm_term=1100090405&uuid=00000000000006AC246A1664949EBB0A4F2714A58851FA160546669715806613&version=11.9.405&version_name=11.9.405&waimai_sign=VbaaJwqV9JEALc9Nv%2FH4egKOC1TvEuUDSZU65rRIJZYlv3FNRvx%2B50qAZN63z9psxr5d%2FUGNhj1S%0AlYkIOBCM1QDGdOXqXa8FPDFAGfSjRDlild4pPFGDifbSyQI%2F8k50NM2QAgMK2phcdvA%2B8PCv0n6%0APR5qvRIuaNCALrtie38%3D%0A&wm_actual_latitude=40240957&wm_actual_longitude=116176188&wm_appversion=11.9.405&wm_ctype=mtandroid&wm_did=47c9b7a49ea898b1&wm_dtype=MIX%202S&wm_dversion=29_10&wm_latitude=40240957&wm_longitude=116176188&wm_mac=02%3A00%3A00%3A00%3A00%3A00&wm_seq=55&wm_uuid=00000000000006AC246A1664949EBB0A4F2714A58851FA160546669715806613&wm_visitid=0c7fb83-2377-42fa-833b-e2b2be98e7ce")
    var mode = jinteger.valueOf(2);
    var objectArray = Java.array('java.lang.Object',['9b69f861-e054-4bc4-9daf-d36ae205ed3e',param_str.getBytes(),mode]);
    var ret = NBridge.main(203,objectArray)
    console.log("call result:"+ret.toString())
  })
}
```

采用 Frida Spawn 处理，附加进程显然时机太晚，拦截不到。

![](https://cdn.nlark.com/yuque/0/2022/png/707476/1668313219916-8bab4a89-2b96-445d-80b7-c003274fc9f3.png)

运行发现这个执行结果为空，这说明存在初始化函数。

接下来寻找初始化函数，一般而言，初始化函数都是 SO 的导出函数，否则不方便介入和影响 SO 的内部环境，形成”初始化“的效果。

所以这里我 Hook 目标 SO 的所有导出函数，在每个导出函数执行前执行我们的目标函数，看从哪里开始返回“正常”结果。[Simp1er](https://github.com/Simp1er/AndroidSec/blob/master/hook_RegisterNative.js) 的脚本可以很方便的确认一个 SO 里有哪些动静态绑定函数。
或者使用这个脚本hook打印所以动态注册的方法
```java
function hook_RegisterNatives() {

    var symbols = Module.enumerateSymbolsSync("libart.so");

    var addrRegisterNatives = null;

    for (var i = 0; i < symbols.length; i++) {

        var symbol = symbols[i];

        //_ZN3art3JNI15RegisterNativesEP7_JNIEnvP7_jclassPK15JNINativeMethodi

        if (symbol.name.indexOf("art") >= 0 &&

                symbol.name.indexOf("JNI") >= 0 &&

                symbol.name.indexOf("RegisterNatives") >= 0 &&

                symbol.name.indexOf("CheckJNI") < 0) {

            addrRegisterNatives = symbol.address;

            console.log("RegisterNatives is at ", symbol.address, symbol.name);

        }

    }

    if (addrRegisterNatives != null) {

        Interceptor.attach(addrRegisterNatives, {

            onEnter: function (args) {

                console.log("[RegisterNatives] method_count:", args[3]);

                var env = args[0];

                var java_class = args[1];

                var class_name = Java.vm.tryGetEnv().getClassName(java_class);

                //console.log(class_name);

                var methods_ptr = ptr(args[2]);

                var method_count = parseInt(args[3]);

                for (var i = 0; i < method_count; i++) {

                    var name_ptr = Memory.readPointer(methods_ptr.add(i * Process.pointerSize * 3));

                    var sig_ptr = Memory.readPointer(methods_ptr.add(i * Process.pointerSize * 3 + Process.pointerSize));

                    var fnPtr_ptr = Memory.readPointer(methods_ptr.add(i * Process.pointerSize * 3 + Process.pointerSize * 2));

                    var name = Memory.readCString(name_ptr);

                    var sig = Memory.readCString(sig_ptr);

                    var find_module = Process.findModuleByAddress(fnPtr_ptr);

                    console.log("[RegisterNatives] java_class:", class_name, "name:", name, "sig:", sig, "fnPtr:", fnPtr_ptr, "module_name:", find_module.name, "module_base:", find_module.base, "offset:", ptr(fnPtr_ptr).sub(find_module.base));

                }

            }

        });

    }

}

setImmediate(hook_RegisterNatives);
```

确认后编写如下代码

```java
var android_dlopen_ext = Module.findExportByName(null, "android_dlopen_ext");
if (android_dlopen_ext != null) {
    Interceptor.attach(android_dlopen_ext, {
        onEnter: function (args) {
            this.hook = false;
            var soName = args[0].readCString();
            if (soName.indexOf("libmtguard.so") !== -1) {
                this.hook = true;
            }
        },
        onLeave: function (retval) {
            if (this.hook) {
                var jniOnload = Module.findExportByName("libmtguard.so", "JNI_OnLoad");
                Interceptor.attach(jniOnload, {
                    onEnter: function (args) {
                        console.log("Enter Mtguard JNI OnLoad");
                    },
                    onLeave: function (retval) {
                        console.log("After Mtguard JNI OnLoad");
                        hook_mtso();
                    }
                });
            }
        }
    });
}

function hook_mtso() {
    Java.perform(function () {
        Java.use("com.meituan.android.common.mtguard.NBridge").main.implementation = function(arg1, arg2) {
            console.log("call com/meituan/android/common/mtguard/NBridge, main(I[Ljava/lang/Object;)[Ljava/lang/Object;");
            call_mtgsig();
            return this.main(arg1, arg2);
        }

        Java.use("com.meituan.android.common.mtguard.NBridge$SIUACollector").getHWProperty.implementation = function() {
            console.log("com/meituan/android/common/mtguard/NBridge$SIUACollector, getHWProperty()Ljava/lang/String;");
            call_mtgsig();
            return this.getHWProperty();
        }

        Java.use("com.meituan.android.common.mtguard.NBridge$SIUACollector").getEnvironmentInfo.implementation = function() {
            console.log("com/meituan/android/common/mtguard/NBridge$SIUACollector, getEnvironmentInfo()Ljava/lang/String;");
            call_mtgsig();
            return this.getEnvironmentInfo();
        }

        Java.use("com.meituan.android.common.mtguard.NBridge$SIUACollector").getEnvironmentInfoExtra.implementation = function() {
            console.log("com/meituan/android/common/mtguard/NBridge$SIUACollector, getEnvironmentInfoExtra()Ljava/lang/String;");
            call_mtgsig();
            return this.getEnvironmentInfoExtra();
        }

        Java.use("com.meituan.android.common.mtguard.NBridge$SIUACollector").getHWStatus.implementation = function() {
            console.log("com/meituan/android/common/mtguard/NBridge$SIUACollector, getHWStatus()Ljava/lang/String;");
            call_mtgsig();
            return this.getHWStatus();
        }

        Java.use("com.meituan.android.common.mtguard.NBridge$SIUACollector").getHWEquipmentInfo.implementation = function() {
            console.log("com/meituan/android/common/mtguard/NBridge$SIUACollector, getHWEquipmentInfo()Ljava/lang/String;");
            call_mtgsig();
            return this.getHWEquipmentInfo();
        }

        Java.use("com.meituan.android.common.mtguard.NBridge$SIUACollector").getUserAction.implementation = function() {
            console.log("com/meituan/android/common/mtguard/NBridge$SIUACollector, getUserAction()Ljava/lang/String;");
            call_mtgsig();
            return this.getUserAction();
        }

        Java.use("com.meituan.android.common.mtguard.NBridge$SIUACollector").getPlatformInfo.implementation = function() {
            console.log("com/meituan/android/common/mtguard/NBridge$SIUACollector, getPlatformInfo()Ljava/lang/String;");
            call_mtgsig();
            return this.getPlatformInfo();
        }

        Java.use("com.meituan.android.common.mtguard.NBridge$SIUACollector").getLocationInfo.implementation = function() {
            console.log("com/meituan/android/common/mtguard/NBridge$SIUACollector, getLocationInfo()Ljava/lang/String;");
            call_mtgsig();
            return this.getLocationInfo();
        }

        Java.use("com.meituan.android.common.mtguard.NBridge$SIUACollector").startCollection.implementation = function() {
            console.log("com/meituan/android/common/mtguard/NBridge$SIUACollector, startCollection()Ljava/lang/String;");
            call_mtgsig();
            return this.startCollection();
        }

        Java.use("com.meituan.android.common.mtguard.NBridge$SIUACollector").getExternalEquipmentInfo.implementation = function() {
            console.log("com/meituan/android/common/mtguard/NBridge$SIUACollector, getExternalEquipmentInfo()Ljava/lang/String;");
            call_mtgsig();
            return this.getExternalEquipmentInfo();
        }
    })
}


function call_mtgsig(){
    Java.perform(function () {
        let NBridge = Java.use('com.meituan.android.common.mtguard.NBridge');
        var JString = Java.use('java.lang.String');
        var jinteger = Java.use("java.lang.Integer");
        var param_str = JString.$new("POST /mtapi/v8/channel/rcmdboard __reqTraceID=276c9977-5799-4c65-a673-38daa289a6bd&app=0&category_code=910&ci=1&f=android&msid=6ac246a1664949ebb0a4f2714a58851fa1605466697158066131668311899492&navigate_type=910&net_stat=0&partner=4&platform=4&poilist_mt_cityid=1&poilist_wm_cityid=110100&preload=0&push_token=dpshebefd64035e6c9fb9ebcee40f06c8383atpu&rank_list_id=2b95d20e8bfa44e2bdf46f00abc97770&req_time=668312542236&userid=-1&utm_campaign=AgroupBgroupC0E0Ghomepage_category1_394__a1__c-1024&utm_content=47c9b7a49ea898b1&utm_medium=android&utm_source=xiaomiyuzhuang2&utm_term=1100090405&uuid=00000000000006AC246A1664949EBB0A4F2714A58851FA160546669715806613&version=11.9.405&version_name=11.9.405&waimai_sign=VbaaJwqV9JEALc9Nv%2FH4egKOC1TvEuUDSZU65rRIJZYlv3FNRvx%2B50qAZN63z9psxr5d%2FUGNhj1S%0AlYkIOBCM1QDGdOXqXa8FPDFAGfSjRDlild4pPFGDifbSyQI%2F8k50NM2QAgMK2phcdvA%2B8PCv0n6%0APR5qvRIuaNCALrtie38%3D%0A&wm_actual_latitude=40240957&wm_actual_longitude=116176188&wm_appversion=11.9.405&wm_ctype=mtandroid&wm_did=47c9b7a49ea898b1&wm_dtype=MIX%202S&wm_dversion=29_10&wm_latitude=40240957&wm_longitude=116176188&wm_mac=02%3A00%3A00%3A00%3A00%3A00&wm_seq=55&wm_uuid=00000000000006AC246A1664949EBB0A4F2714A58851FA160546669715806613&wm_visitid=0c7fb83-2377-42fa-833b-e2b2be98e7ce")
        var mode = jinteger.valueOf(2);
        var objectArray = Java.array('java.lang.Object',['9b69f861-e054-4bc4-9daf-d36ae205ed3e',param_str.getBytes(),mode]);
        var ret = NBridge.main(203,objectArray)
        console.log("call result:"+ret.toString())
    })
}
```

再次 Spawn Hook 测试

![](https://cdn.nlark.com/yuque/0/2022/png/707476/1668313605469-b2349632-c9a0-48e2-b9b4-5510c5071bc4.png)

惊喜的发现只在第一次 main 函数调用前无法获得结果，换而言之，`main`203 的初始化函数是另一个`main`函数调用，将对`main`函数的 Hook 打印处理的更详细一些。

```java
Java.use("com.meituan.android.common.mtguard.NBridge").main.implementation = function(arg1, arg2) {
    console.log("call com/meituan/android/common/mtguard/NBridge, main(I[Ljava/lang/Object;)[Ljava/lang/Object;");
    console.log('main is called' + ', ' + 'i: ' + arg1 + ', ' + 'objArr: ' + arg2);
    call_mtgsig();
    return this.main(arg1, arg2);
}
```

发现是 111 这个分支

![](https://cdn.nlark.com/yuque/0/2022/png/707476/1668313753129-3cbdc4b2-cb2e-4b78-bfcd-8f2e3529a80d.png)

其实静态分析可以琢磨出这一点，但当然没有动态分析这么快速和明显。

![](https://cdn.nlark.com/yuque/0/2022/png/707476/1668313795822-572d58a9-c351-48ca-b98a-202df45e0b62.png)

这样我们就找到了目标函数的初始化函数，在 Unidbg 里先执行 main 111，再执行 203 即可。

## 三、总结

回顾一下基于动态分析找寻初始化函数的这个过程，其实思路很清晰。

1. Hook 得到目标函数基本的执行参数和返回值情况。
2. 在附加进程的情况下 Frida Call ,验证返回结果是否无误，这步骤是为了确认 Call 代码是否正常无误。
3. 在 JNI_OnLoad 结束的时机发起 Frida Call ,确认是否正常无误，如果有问题，比如报错、返回空字符串等等说明有初始化函数。
4. 在 JNI_OnLoad 结束的时机 Hook SO 中所有动静态绑定的 Java Native 函数，在其执行前或执行后调用目标函数，确认初始化函数。

有的读者会简化成另一个路子，就是直接 Spawn 下 Hook SO 中所有动静态绑定的 Java Native 函数，然后观察执行流，将我们的目标函数之前的所有函数都调用一遍。

这个方案当然也完全可行，但读者可能要因此补很多无用的 JNI 环境，成本比较高。

除此之外还有一个问题，就是 Frida Spawn Hook 这些函数经常会出错，没法捕捉到最早的那些执行流，比如本篇就是。这个问题有两个简单解法

- 使用 Frida Native Hook，即对目标函数所对应的 Native 实现做 Hook，稳定性比 JAVA Hook 好很多，但缺点也不是没有—— Native Hook 在 Hook 代码编写、参数和返回值打印等方面都更复杂一些些。
- 使用 [Lsposed](https://github.com/LSPosed/LSPosed) 或其他 JAVA Hook 框架，可以稳定准确不遗漏的 Hook 目标函数。r0ysue大佬对此出过一个很好的[教学视频](https://www.bilibili.com/video/BV1Lb4y1o7Y9/?spm_id_from=333.999.0.0)。

## 四、尾声

动态分析看起来很准确高效，但不能因此忽略静态分析，否则会犯低级错误，下篇我们看一个这样的例子。



