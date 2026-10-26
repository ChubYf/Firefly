---
title: unidbg处理文件访问
published: 2026-10-03
description: unidbg处理文件访问
image: ./cover.jpg
tags: [逆向]
category: unidbg
draft: false
---

[unidbg处理文件访问一](https://www.yuque.com/docs/share/1559bab0-90de-422c-a3fb-eb32e68f44e0?#%20%E3%80%8AUnidbg%20%E4%B8%AD%E5%A4%84%E7%90%86%E6%96%87%E4%BB%B6%E8%AE%BF%E9%97%AE%EF%BC%88%E4%B8%80%EF%BC%89%E3%80%8B  ) 
[unidbg处理文件访问二](https://www.yuque.com/docs/share/c5c3ceab-570c-4e2a-823d-0c0a275cca33?#%20%E3%80%8AUnidbg%20%E4%B8%AD%E5%A4%84%E7%90%86%E6%96%87%E4%BB%B6%E8%AE%BF%E9%97%AE%EF%BC%88%E4%BA%8C%EF%BC%89%E3%80%8B  ) 




# unidbg处理文件访问
## 一、引言

补文件访问在 Unidbg 补环境这项工作里的重要性仅次于补 JNI 环境，它有两个主要特征

一是出现频率高

它的出现频率远高于补系统调用、补库函数、补虚拟模块这些情况，仅次于JNI调用。信息获取和环境检测都会大量使用文件访问。

二是容易被补漏

Unidbg 对文件访问提供了很好的监控和打印，所谓容易补漏，主要是使用者的问题。因为安卓碎片化严重、版本多、权限管理混乱等等原因，因为即使在真实 Android 环境里，访问某个文件没成功也是很常见的事，因此开发上会做大量的兼容处理。换句话说，大部分文件补了最好，不补也不一定有事。因此许多 Unidbg 的使用者走向了极端——完全不关注文件访问，这确实省时省力，而且对 80% 样本来说并影响。但是，在 20% 的样本和场景里会出问题，而且往往越难的样本越会出问题。

因此学好怎么补文件访问还是挺重要的，本篇就是系统的讲这个问题。

## 二、概述

文件访问最主要服务于环境检测和信息收集，举例如下。

- 访问 /proc/self/maps 检测 frida/xposed 等模块
- 访问 /proc/net/tcp 检测 frida/ida server 默认端口
- 访问 /proc/self/cmdline 确认在自身进程内
- 访问 apkfile 做签名校验，解析资源文件等等
- 访问 /xbin/su 检测 root
- 访问 /data/data/package/xxx 读取自身相关文件

Unidbg 通过文件处理器处理文件。默认的文件处理器是 AndroidResolver，我们可以在它的 resolve 方法里处理样本所发起的文件访问。

```java
//source:src/main/java/com/github/unidbg/linux/android/AndroidResolver.java
    @Override
public FileResult<AndroidFileIO> resolve(Emulator<AndroidFileIO> emulator, String path, int oflags) {
    // 相关逻辑，此处省略
    return null;
}
```

在这里写代码可不是一个好方案，出于两方面原因

- 可移植性差：AndroidResolver 属于 Unidbg 源码的一部分，修改 AndroidResolver 不得避免的失去了灵活性。如果想迁移项目，需要对整个 Unidbg 环境打包。
- 环境混乱：我们一般会在 Unidbg 环境里处理多个样本，它们都有文件访问的需求，对于同一个文件，不同样本的预期返回不一定相同，比如 /proc/self/cmdline 查看包名，在 AndroidResolver 中难以做区分。

Unidbg 并不会让我们陷入泥潭，在文件处理上它采用了责任链模式的设计。用户可以自定义一个或多个文件处理器，当文件访问发生时，多个文件处理器都有机会处理该文件访问，直到其中某个对它处理成功，或最终失败返回为止。责任链模式将多个文件处理器串成链，然后让文件访问在链上传递。

![](https://cdn.nlark.com/yuque/0/2022/png/707476/1663979960314-954f455b-b2bc-4d52-8b40-1111cc987bef.png)

先依次进入用户自定义的 IOResolver A 和 B，然后是 AndroidResolver，最后是虚拟文件系统。如果文件访问在较早的某处得到了处理，那就会直接返回。图例所对应的代码如下。

```java
protected final FileResult<T> resolve(Emulator<T> emulator, String pathname, int oflags) {
    FileResult<T> failResult = null;
    // 依次进入每个文件解析器，以期得到处理
    for (IOResolver<T> resolver : resolvers) {
        FileResult<T> result = resolver.resolve(emulator, pathname, oflags);
        if (result != null && result.isSuccess()) {
            emulator.getMemory().setErrno(0);
            return result;
        } else if (result != null) {
            if (failResult == null || !failResult.isFallback()) {
                failResult = result;
            }
        }
    }
    if (failResult != null && !failResult.isFallback()) {
        return failResult;
    }
	// 虚拟文件系统
    FileResult<T> result = emulator.getFileSystem().open(pathname, oflags);
    if (result != null && result.isSuccess()) {
        emulator.getMemory().setErrno(0);
        return result;
    }

   // 一些杂项路径判断和处理，此处省略
    
    return failResult;
}
```

Unidbg 在这里的设计也并非完美，一些特殊以及杂项文件的处理没有设计对应的文件处理器，而是较为零散的放在各处。事实上，将它们重构为一个 procFileIOResolver 会更合理，感兴趣的读者可以去做尝试。

```java
if (("/proc/" + emulator.getPid() + "/fd").equals(pathname) || "/proc/self/fd".equals(pathname)) {
    return createFdDir(oflags, pathname);
}
if (("/proc/" + emulator.getPid() + "/task/").equals(pathname) || "/proc/self/task/".equals(pathname)) {
    return createTaskDir(emulator, oflags, pathname);
}
```

上面我们从一个较高的视角观察了 Unidbg 中文件访问的处理流程，接下来进入细节，可不能眼高手低。首先是定义和添加文件处理器的具体写法，`addIOResolver`用于添加一个文件处理器对象，将如下代码放在早于 loadlibrary 的时机即可生效。

```java
emulator.getSyscallHandler().addIOResolver(new IOResolver<AndroidFileIO>() {
    @Override
    public FileResult<AndroidFileIO> resolve(Emulator<AndroidFileIO> emulator, String pathname, int oflags) {
        System.out.println("file open:"+pathname);
        return null;
    }
});
```

有时候我们会让目标类实现 IOResolver 接口，那么就是`addIOResolver(this)`。

```java
package com.bili;

import com.github.unidbg.AndroidEmulator;
import com.github.unidbg.Emulator;
import com.github.unidbg.arm.backend.Unicorn2Factory;
import com.github.unidbg.file.FileResult;
import com.github.unidbg.file.IOResolver;
import com.github.unidbg.file.linux.AndroidFileIO;
import com.github.unidbg.linux.android.AndroidEmulatorBuilder;
import com.github.unidbg.linux.android.AndroidResolver;
import com.github.unidbg.linux.android.dvm.AbstractJni;
import com.github.unidbg.linux.android.dvm.DalvikModule;
import com.github.unidbg.linux.android.dvm.DvmClass;
import com.github.unidbg.linux.android.dvm.VM;
import com.github.unidbg.memory.Memory;

import java.io.File;

public class Bili extends AbstractJni implements IOResolver<AndroidFileIO>{
    private final AndroidEmulator emulator;
    private final DvmClass LibBili;
    private final VM vm;

    @Override
    public FileResult<AndroidFileIO> resolve(Emulator<AndroidFileIO> emulator, String pathname, int oflags) {
        System.out.println("file open:"+pathname);
        return null;
    }

    public Bili() {
        emulator = AndroidEmulatorBuilder.for32Bit()
                .addBackendFactory(new Unicorn2Factory(false))
                .build();
        emulator.getBackend().registerEmuCountHook(10000); // 设置执行多少条指令切换一次线程
        emulator.getSyscallHandler().setVerbose(true);
        emulator.getSyscallHandler().setEnableThreadDispatcher(true);
        Memory memory = emulator.getMemory();
        memory.setLibraryResolver(new AndroidResolver(23));
        vm = emulator.createDalvikVM(new File("unidbg-android/src/test/resources/bili/bili.apk"));
        vm.setVerbose(true);
        vm.setJni(this);
        
        emulator.getSyscallHandler().addIOResolver(this);

        DalvikModule dm = vm.loadLibrary("bili", true);
        LibBili = vm.resolveClass("com.bilibili.nativelibrary.LibBili");
        dm.callJNI_OnLoad(emulator);
    }

    public void calla(){
        String appkey = LibBili.callStaticJniMethodObject(emulator, "a(Ljava/lang/String;)Ljava/lang/String;", "android").getValue().toString();
        System.out.println("result:"+appkey);
    }


    public static void main(String[] args) {
        Bili bili = new Bili();
        bili.calla();
    }


}
```

在处理很复杂的样本时，我们希望逻辑分离，那么可以把 IOResolver 从主类中摘出去。

```java
package com.bili;

import com.github.unidbg.Emulator;
import com.github.unidbg.file.FileResult;
import com.github.unidbg.file.IOResolver;
import com.github.unidbg.file.linux.AndroidFileIO;

public class BiliIOResolver implements IOResolver<AndroidFileIO> {
    @Override
    public FileResult<AndroidFileIO> resolve(Emulator<AndroidFileIO> emulator, String pathname, int oflags) {
        System.out.println("file open:"+pathname);        
        return null;
    }
}
```

主类代码

```java
package com.bili;

import com.github.unidbg.AndroidEmulator;
import com.github.unidbg.Emulator;
import com.github.unidbg.arm.backend.Unicorn2Factory;
import com.github.unidbg.file.FileResult;
import com.github.unidbg.file.IOResolver;
import com.github.unidbg.file.linux.AndroidFileIO;
import com.github.unidbg.linux.android.AndroidEmulatorBuilder;
import com.github.unidbg.linux.android.AndroidResolver;
import com.github.unidbg.linux.android.dvm.AbstractJni;
import com.github.unidbg.linux.android.dvm.DalvikModule;
import com.github.unidbg.linux.android.dvm.DvmClass;
import com.github.unidbg.linux.android.dvm.VM;
import com.github.unidbg.memory.Memory;

import java.io.File;

public class Bili extends AbstractJni{
    private final AndroidEmulator emulator;
    private final DvmClass LibBili;
    private final VM vm;

    public Bili() {
        emulator = AndroidEmulatorBuilder.for32Bit()
                .addBackendFactory(new Unicorn2Factory(false))
                .build();
        emulator.getBackend().registerEmuCountHook(10000); // 设置执行多少条指令切换一次线程
        emulator.getSyscallHandler().setVerbose(true);
        emulator.getSyscallHandler().setEnableThreadDispatcher(true);
        Memory memory = emulator.getMemory();
        memory.setLibraryResolver(new AndroidResolver(23));
        vm = emulator.createDalvikVM(new File("unidbg-android/src/test/resources/bili/bili.apk"));
        vm.setVerbose(true);
        vm.setJni(this);

        emulator.getSyscallHandler().addIOResolver(new BiliIOResolver());

        DalvikModule dm = vm.loadLibrary("bili", true);
        LibBili = vm.resolveClass("com.bilibili.nativelibrary.LibBili");
        dm.callJNI_OnLoad(emulator);
    }

    public void calla(){
        String appkey = LibBili.callStaticJniMethodObject(emulator, "a(Ljava/lang/String;)Ljava/lang/String;", "android").getValue().toString();
        System.out.println("result:"+appkey);
    }


    public static void main(String[] args) {
        Bili bili = new Bili();
        bili.calla();
    }
}
```

添加多个自定义文件处理器的场景里（比如处理某公司旗下的多个样本，它们在多数文件访问上类似，但少部分需要各自处理。），需要注意后添加的优先级更高。

```java
/**
 * 后面添加的优先级高
 */
void addIOResolver(IOResolver<T> resolver);
```

比如下面的逻辑里，文件访问会先首先经过 B，其次才是 A。

```java
emulator.getSyscallHandler().addIOResolver(new IOResolver<AndroidFileIO>() {
    @Override
    public FileResult<AndroidFileIO> resolve(Emulator<AndroidFileIO> emulator, String pathname, int oflags) {
        System.out.println("Resolver A");
        return null;
    }
});

emulator.getSyscallHandler().addIOResolver(new IOResolver<AndroidFileIO>() {
    @Override
    public FileResult<AndroidFileIO> resolve(Emulator<AndroidFileIO> emulator, String pathname, int oflags) {
        System.out.println("Resolver B");
        return null;
    }
});
```

自定义以及添加文件处理器需要关注的内容就这么多，接下来是具体的文件处理。

## 三、文件访问基本处理

这是本篇的主要内容，也是补环境的实际指导。如果你想学好它，需要意识到它的知识由三部分构成。

![](https://cdn.nlark.com/yuque/0/2022/png/707476/1663983989813-a3022bde-395c-4da8-b671-e1e60033846e.png)

举个例子，我们设置了自定义文件处理器，打印样本所访问的文件，假如日志里出现了下面这么一行

```java
file open:/proc/sys/kernel/random/boot_id
```

Linux/Android 文件系统的知识：boot_id 文件在开机时生成，在设备关机前不会改变内容。我们将这个文件从真机上 push 出来

```java
C:\Users\13352>adb shell
polaris:/ $ cat /proc/sys/kernel/random/boot_id
34aa1c03-992e-4020-b20f-a684e32a5dba
polaris:/ $ cp /proc/sys/kernel/random/boot_id /sdcard
polaris:/ $ exit

C:\Users\13352>adb pull /sdcard/boot_id yourpath
/sdcard/boot_id: 1 file pulled, 0 skipped. 0.0 MB/s (37 bytes in 0.011s)
```

对业务的经验：boot_id 常用于构建设备指纹或单纯的信息采集，需要补它。

Unidbg 对应的实现和映射：像下面这样补它

```java
emulator.getSyscallHandler().addIOResolver(new IOResolver<AndroidFileIO>() {
    @Override
    public FileResult<AndroidFileIO> resolve(Emulator<AndroidFileIO> emulator, String pathname, int oflags) {
        //参数二: 文件路径
        //参数三: 操作文件标志
        switch (pathname){
            case "/proc/sys/kernel/random/boot_id":{
                return FileResult.<AndroidFileIO>success(new SimpleFileIO(oflags, new File("unidbg-android/src/test/resources/dewu/cpu/boot_id"), pathname));
            }
        }
        return null;
    }
});
```

我们主要讲解第一部分，也会适时补充第二、三部分的内容。先看代码，我们用 switch case 对 pathname 做判断，对 boot_id 做处理，return 语句看起来有些复杂，我们拆解一下。

最外层是`FileResult.<AndroidFileIO>success`，我们可以返回任意 AndroidFileIO 类型的文件，这么说有点怪，这是因为 Unidbg 是一个 Android/IOS 双端的 Native 模拟器，除了 AndroidFileIO 还有对应于 IOS 的 DarwinFileIO，所以有此限制。

对文件访问有 success、failed、fallback 三种处理

- success 表示文件顺利访问，参数是 NewFileIO。
- failed 表示文件访问失败，参数是 erron 错误码。
- fallback 表示回退、降级，只有在其他处理器无法处理对该文件的访问时，才由 fallback 指定的文件 IO 来处理

第一种使用最多，只要文件访问的意图正常——信息收集，我们都很乐意正常予以返回。因为我们希望程序收集到足够多且内容合理的设备信息，以顺利执行完函数以及规避简单风控。

SimpleFileIO 是最常使用的 IO 类型，它代表普通文件，上面补 boot_id 就用的它。resolve 所提供的 oflags 和 path 原封不动的传给它，file 是从真机 pull 出来的 boot_id 文件。

```java
public SimpleFileIO(int oflags, File file, String path) {
    super(oflags);
    this.file = file;
    this.path = path;

    if (file.isDirectory()) {
        throw new IllegalArgumentException("file is directory: " + file);
    }
    if (!file.exists()) {
        throw new IllegalArgumentException("file not exists: " + file);
    }
}
```

读者可能会想一个问题，boot_id 在一定程度上用于标识设备，那么在生产环境中，我们需要对它做随机化以躲避风控。使用 SimpleFileIO 就不太好处理这个问题，这时候可以用 ByteArrayFileIO。

```java
emulator.getSyscallHandler().addIOResolver(new IOResolver<AndroidFileIO>() {
    @Override
    public FileResult<AndroidFileIO> resolve(Emulator<AndroidFileIO> emulator, String pathname, int oflags) {
        System.out.println("Resolver A");
        switch (pathname){
            case "/proc/sys/kernel/random/boot_id":{
                return FileResult.<AndroidFileIO>success(new ByteArrayFileIO(oflags,  pathname, UUID.randomUUID().toString().getBytes(StandardCharsets.UTF_8)));
            }
        }
        return null;
    }
});
```

ByteArrayFileIO 在 oflags、pathname 上和 SimpleFileIO 无差别，但它接收一个字节数组而非 File 文件，对于灵活补文件内容有很大的帮助。读者也需知道它的副作用——没办法做文件做写入操作，ByteArrayFileIO 像浮萍漂泊，无根无依。

```java
@Override
public int write(byte[] data) {
    throw new UnsupportedOperationException();
}
```

如果样本访问的是一个目录呢？这时候要用 DirectoryFileIO，File 传入文件夹即可。

```java
case "/data/data/com.sankuai.meituan":{
    return FileResult.<AndroidFileIO>success(new DirectoryFileIO(oflags, pathname, new File("unidbg-android/src/test/resources/meituan/data")));
}
```

但对于文件夹访问而言，使用虚拟文件系统是更好的选择。首先看前文 [《创建模拟器和加载模块》](https://www.yuque.com/docs/share/4b18c830-6763-445a-80a9-6ee3ee822547)的 2-4 小节，设置好虚拟目录的根目录。然后将待处理的文件按照层级放到对应位置。

![](https://cdn.nlark.com/yuque/0/2022/png/707476/1663988014038-670ce7d3-97d6-4f8c-bda7-4728910c0d88.png)

使用虚拟文件系统，我们对文件的掌控程度会下降，在补常规文件时不算好办法，但补文件夹算是一个特殊情况。举个例子，如果样本访问本机的 /system/fonts 目录，对所有字体文件做 md5 然后上传。如果用 DirectoryFileIO 补，那么在补了 fonts 文件夹后，还需要用 SimpleFileIO 处理每个字体文件的访问，工作量相当大，而使用虚拟文件系统，只要把文件夹对应位置放好，就可以顺利处理整个逻辑。

需要注意，补文件访问时不要犯低级错误，比如样本访问 proc/version，尽管和 /proc/version 是一回事，但你可不要多了一个斜杠，pathname 字符串匹配找不到了。

80% 的情况里，你就会遇到这么点文件访问的需求——从真机把对应的文件拖出来，用 SimpleFileIO、ByteArrayFileIO 、DirectoryFileIO 以及虚拟文件系统去处理它。

比如 补 apk，在任何情况下，都不应该对访问 apk 忽略不管，因为它往往用于资源读取或签名校验，不补风险很大。

```java
case "/data/app/com.jingdong.app.mall-jJGgNO9r6uKMLj1ytVwSRw==/base.apk":{
    return FileResult.<AndroidFileIO>success(new SimpleFileIO(oflags, new File("unidbg-android/src/test/resources/jd/jd.apk"), pathname));
}
```

补 CPU 信息

```java
// https://bbs.pediy.com/thread-229579-1.htm
case "/sys/devices/system/cpu/cpu0/cpufreq/cpuinfo_max_freq":{
    return FileResult.<AndroidFileIO>success(new ByteArrayFileIO(oflags, pathname, "1766400".getBytes()));
}
```

补电池信息

```java
case "/sys/class/power_supply/battery/temp":{
    return FileResult.<AndroidFileIO>success(new SimpleFileIO(oflags, new File("unidbg-android/src/test/resources/meituan/files/battery/temp"), pathname));
}

case "/sys/class/power_supply/battery/voltage_now":{
    return FileResult.<AndroidFileIO>success(new SimpleFileIO(oflags, new File("unidbg-android/src/test/resources/meituan/files/battery/voltage_now"), pathname));
}
```

接下来说说 failed，它表示文件访问失败，使用场景其实也不多。因为只要我们什么都不补，AndroidResolver 里没有处理这个文件，虚拟文件系统中也不包含它，那就自动访问失败 + 设置错误码。

但有个情况比较例外——如果文件访问 /data，判断是否有写权限，进而确认设备环境是否 Root 的话，我们需要手动处理。

```java
case "/data":{
    return FileResult.failed(UnixEmulator.EACCES);
}
```

因为虚拟文件系统会自动生成 data 目录，进而访问成功。错误码列表如下，也可以直接使用看 Linux 的错误码文档进行设置，Unidbg 本就是它的简单映射。

```java
package com.github.unidbg.unix;

public interface UnixEmulator {

    int EPERM = 1; /* Operation not permitted */
    int ENOENT = 2; /* No such file or directory */
    int ESRCH = 3; /* No such process */
    int EINTR = 4; /* Interrupted system call */
    int EBADF = 9; /* Bad file descriptor */
    int EAGAIN = 11; /* Resource temporarily unavailable */
    int ENOMEM = 12; /* Cannot allocate memory */
    int EACCES = 13; /* Permission denied */
    int EFAULT = 14; /* Bad address */
    int EEXIST = 17; /* File exists */
    int ENOTDIR = 20; /* Not a directory */
    int EINVAL = 22; /* Invalid argument */
    int ENOTTY = 25; /* Inappropriate ioctl for device */
    int ENOSYS = 38; /* Function not implemented */
    int ENOATTR = 93; /* Attribute not found */
    int EOPNOTSUPP = 95; /* Operation not supported on transport endpoint */
    int EAFNOSUPPORT = 97; /* Address family not supported by protocol family */
    int EADDRINUSE = 98; /* Address already in use */
    int ECONNREFUSED = 111; /* Connection refused */

}
```

## 四、环境检测

当样本做环境检测相关的文件访问时，主要检测这些文件是否存在，以及是否有权限

- Root检测（检测 su、Magisk、Riru，检测市面上的 Root 工具）
- 模拟器检测（检测 Qemu，检测各家模拟器，比如夜神、雷电、Mumu 等模拟器的文件特征、驱动特征等）
- 危险应用检测（各类多开助手、按键精灵、接码平台等）
- 云手机检测 （以各种云手机产品为主）
- Hook框架（以 Xposed、Substrate、Frida 为主）
- 脱壳机（以 Fart、DexHunter、Youpk 三者为主）

我们选择什么都不补就行，因为不管是自定义还是默认的文件处理器，以及虚拟文件系统里，都不会有这样内容，这正合我们的意。就像下样，什么都不补就行了。

![](https://cdn.nlark.com/yuque/0/2022/png/707476/1663992483880-6b2a029e-b619-48e1-9f8c-0b9a690b299d.png)

读者需要自行分辨某个文件访问是否属于此类，如果捉摸不定建议 Google 搜索这个文件的主要用途。




# unidbg处理文件访问(proc伪文件系统)
## 一、引言

本文的主题是 proc 伪文件系统，请读者先 [Google](https://www.google.com.hk/search?q=proc+%E4%BC%AA%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F) 了解它，本篇讨论在 Unidbg 里如何补其中对应的文件。

## 二、概述

proc 是一个由内核动态创建和生成的目录，程序通过访问它，获取各种各样的内核信息。首先的问题必然是为什么要把它单独拎出来讲，它和其他文件访问在补环境上有什么差异？我给出两点理由

- 访问频率高——文件访问中最高频处理的目录
- 容易出错——尽管整体上的处理和普通文件一样，但局部要特殊处理

### 2.1 cmdline

/proc/self(pid)/cmdline 用于查看自身进程名，常用于信息收集或检测运行环境是否有异。比如有些分析人员喜欢将样本 SO 昂到自己的 Android demo 中加载和调试。关于它有几点需知

首先，遇到了它一定要补，它可不是什么无足轻重的文件访问，校验进程名是很重要的代码逻辑。

其次，pid 在 Unidbg 中每次运行都会变化，原因见[《创建模拟器和加载模块》](https://www.yuque.com/docs/share/4b18c830-6763-445a-80a9-6ee3ee822547)的 2-5 小节，因此硬编码无法顺利匹配下一次运行的 pid，请规范处理。

```java
// A
if (("/proc/"+emulator.getPid()+"/cmdline").equals(pathname) || ("/proc/self/cmdline").equals(pathname)) {
    // 具体处理逻辑
}

// B
if (pathname.contains("cmdline")) {
    // 具体处理逻辑
}
```

对于涉及到 pid 的其他访问路径，也建议一律这么做。而且假设 Unidbg 访问 `/proc/1234/maps`或`/proc/self/maps`，你不应该通过 adb 在同样位置去找它，因为 Unidbg 和真实环境的 pid 显然没有对应关系，应该通过 `ps | grep packageName`找到目标 app 在真机上的 pid，然后去做对应的查看。

最后，对于进程名这个需求，没必要从真机拷贝出对应文件，直接 cat 查看，然后用 ByteArrayFileIO 处理就很方便，结尾记得加上`\0`，这是 cmdline 的格式规范，不加的话在解析时存在出错的可能性。

```java
// 查看当前进程名
if (("/proc/"+emulator.getPid()+"/cmdline").equals(pathname) || ("/proc/self/cmdline").equals(pathname)) {
    return FileResult.success(new ByteArrayFileIO(oflags, pathname, "name\0".getBytes()));
}
```

### 2.2 status

/proc/self(pid)/status 最主要用来检测 tracepid 状态，因此偷懒的话可以像下面这样干

```java
if (("proc/" + emulator.getPid() + "/status").equals(pathname)) {
    return FileResult.success(new ByteArrayFileIO(oflags, pathname, ("TracerPid: 0").getBytes()));
}
```

这么做并非毫无风险，样本也可能会校验和关注其他字段，所以像下面这样补更规范一些。

```java
if(pathname.equals("proc/"+emulator.getPid()+"/status")){
    return FileResult.success(new ByteArrayFileIO(oflags, pathname,
            ("Name:   ip.android.view\n" +
            "Umask:  0077\n" +
            "State:  S (sleeping)\n" +
            "Tgid:   "+emulator.getPid()+"\n" +
            "Ngid:   0\n" +
            "Pid:    "+emulator.getPid()+"\n" +
            "PPid:   6119\n" +
            "TracerPid:      0\n" +
            "Uid:    10494   10494   10494   10494\n" +
            "Gid:    10494   10494   10494   10494\n" +
            "FDSize: 512\n" +
            "Groups: 3002 3003 9997 20494 50494 99909997\n" +
            "VmPeak:  2543892 kB\n" +
            "VmSize:  2466524 kB\n" +
            "VmLck:         0 kB\n" +
            "VmPin:         0 kB\n" +
            "VmHWM:    475128 kB\n" +
            "VmRSS:    415548 kB\n" +
            "RssAnon:          144072 kB\n" +
            "RssFile:          267216 kB\n" +
            "RssShmem:           4260 kB\n" +
            "VmData:  1488008 kB\n" +
            "VmStk:      8192 kB\n" +
            "VmExe:        20 kB\n" +
            "VmLib:    239368 kB\n" +
            "VmPTE:      2360 kB\n" +
            "VmPMD:        16 kB\n" +
            "VmSwap:    13708 kB\n" +
            "Threads:        122\n" +
            "SigQ:   0/21555\n" +
            "SigPnd: 0000000000000000\n" +
            "ShdPnd: 0000000000000000\n" +
            "SigBlk: 0000000080001204\n" +
            "SigIgn: 0000000000000001\n" +
            "SigCgt: 0000000e400096fc\n" +
            "CapInh: 0000000000000000\n" +
            "CapPrm: 0000000000000000\n" +
            "CapEff: 0000000000000000\n" +
            "CapBnd: 0000000000000000\n" +
            "CapAmb: 0000000000000000\n" +
            "Seccomp:        2\n" +
            "Speculation_Store_Bypass:       unknown\n" +
            "Cpus_allowed:   07\n" +
            "Cpus_allowed_list:      0-2\n" +
            "Mems_allowed:   1\n" +
            "Mems_allowed_list:      0\n" +
            "voluntary_ctxt_switches:        17290\n" +
            "nonvoluntary_ctxt_switches:     10433").getBytes(StandardCharsets.UTF_8)));
}
```

### 2.3 net

/proc/net 下的文件一律不要补，有两个理由。

一是 Goolg 禁止普通进程访问该目录，这一规定对 Android 10 以及更高的版本均有效，所以我们伪装成高版本，多一事不如少一事，少补几个文件而且无副作用，岂不美哉。

二是这个目录主要用于检测环境，比如 IDA/Frida Server 的端口检测。将真机的对应文件一股脑 拷贝过来，还可能把风险信息和检测点带过来。

简而言之，允许不补，补了还容易错，你说该怎么做？其中最常见的是下面这几个

```java
/proc/net/arp
/proc/net/tcp
/proc/net/unix
```

### 2.4 信息获取

遇到下面这些文件时，建议正常补，并且了解每个文件的用途，在生产环境上部分需要随机化。

```java
/proc/meminfo
/proc/version
/proc/cpuinfo
/proc/stat
/proc/asound/cardX/id
/proc/self/exe
```

### 2.5 maps

/proc/self(pid)/maps 是补 proc 文件访问中最重要的一项，请读者良好处理每一次 maps 访问。因为 maps 访问频率高以及地位特殊，Unidbg 中对它做了专门的处理——如果在自定义 IOResolver 中没有得到处理，会在最后的虚拟文件系统这一块进行如下处理

```java
// src/main/java/com/github/unidbg/file/linux/LinuxFileSystem.java
@Override
public FileResult<AndroidFileIO> open(String pathname, int oflags) {
    if ("/proc/self/maps".equals(pathname) || ("/proc/" + emulator.getPid() + "/maps").equals(pathname)) {
        return FileResult.<AndroidFileIO>success(new MapsFileIO(oflags, pathname, emulator.getMemory().getLoadedModules()));
    }

    return super.open(pathname, oflags);
}
```

MapsFileIO 是 Unidbg 所设计的 fakemaps，它反应了 Unidbg 内存环境中的模块信息，即在语义和格式都等价于真实 maps，如下就是 libmtguard.so 对应的 fakemaps。

```java
40000000-400cd000 r-xp 00000000 b3:19 0          /data/app/com.sankuai.meituan-1/lib/arm/libmtguard.so
400cd000-400ea000 rw-p 000cd890 b3:19 0          /data/app/com.sankuai.meituan-1/lib/arm/libmtguard.so
400ea000-40104000 r-xp 00000000 b3:19 0          /system/lib/libz.so
40104000-40106000 rw-p 0001ace0 b3:19 0          /system/lib/libz.so
40108000-40190000 r-xp 00002000 b3:19 0          /system/lib/libc++.so
40191000-40197000 rw-p 0008b410 b3:19 0          /system/lib/libc++.so
40197000-40199000 r-xp 00000000 b3:19 0          /system/lib/libdl.so
40199000-4019b000 rw-p 00002edc b3:19 0          /system/lib/libdl.so
4019b000-4020e000 r-xp 00000000 b3:19 0          /system/lib/libc.so
4020e000-4021f000 rw-p 00073f60 b3:19 0          /system/lib/libc.so
4021f000-4023e000 r-xp 00000000 b3:19 0          /system/lib/libm.so
4023f000-40241000 rw-p 00020c5c b3:19 0          /system/lib/libm.so
40241000-40249000 r-xp 00000000 b3:19 0          /system/lib/liblog.so
40249000-4024b000 rw-p 00008d08 b3:19 0          /system/lib/liblog.so
4024b000-40250000 r-xp 00000000 b3:19 0          /system/lib/libstdc++.so
40250000-40252000 rw-p 00005dec b3:19 0          /system/lib/libstdc++.so
```

从客观上讲，这个 maps 确实能处理一些问题。比如样本访问 maps，检测其中是否有 frida / xposed / substrate / magisk / riru 等风险模块，真实 maps 会暴露这些信息，而 Unidbg fakeMaps 很干净、简洁，所以不会有任何问题。

既然 Unidbg 对 maps 做了这样周到的处理，为什么还要关注 maps 呢？请读者考虑 fakeMaps 和真实 maps 的关系，fakeMaps 仅包含按目标模块极其依赖的信息，真实 maps 包含整个进程所涉及的所有模块的信息，两者关系如下。

![](https://cdn.nlark.com/yuque/0/2022/png/707476/1664011206414-b2a76a7b-e24d-4c3f-b379-892aec7051ac.png)

因此在内容上，fakeMaps 是 真实 maps 的子集，后者包含了许多前者不具备的信息。如果样本试图获取小框外的信息，就会出问题。

比如真实 maps 包含当前进程所对应的 apk，可以访问和解析它，做签名校验。

![](https://cdn.nlark.com/yuque/0/2022/png/707476/1664011416959-10e9695c-ca45-4791-ab94-cd73b2d21934.png)

比如有些样本会检测安全 SDK 是否已加载，未加载就不提供服务，判断逻辑是 maps 中是否能找到安全 SDK 对应的 SO。

上面两个逻辑，没法依靠 fakemaps 实现处理。 既然 fakeMaps 有这样的问题，使用真实 maps 会好吗？对于上面两个问题来说，使用真实 maps 就解决了问题。但在另外一些场景里，fakeMaps 可行，真实 maps 会出现问题，下面开始举例。

我们知道，maps 反映了内存布局的详细情况，那么如果样本根据 maps 做内存访问呢？比如检查目标函数所属的 libshell-superv.2019.so 是否被 inline hook ，又或者遍历内存寻找 frida 等工具的特征字符串等等。maps 所指引的 755118f000-75511d2000 在真实运行环境上，是目标 SO 起始的一段内存，但在 Undbg 环境里，这段内存甚至没有开辟。

![](https://cdn.nlark.com/yuque/0/2022/png/707476/1664013030637-b8cd799b-fe80-4158-a506-81c75412f9e2.png)

这就好比拿着北京地图在南京开车，怎么能不出错呢？因此不管使用真实 maps 还是 fakeMaps，都可以解决一些问题，但又遭遇另一些问题。

在理论上，我建议遇到对 maps 的访问，就对这部分逻辑做算法分析，完全确定其访问意图，然后反向构造合适的 maps 予以返回。

在实践上，可能没法总这么干，因为太费事了，那么建议优先使用真实 maps，如果出现内存异常（这意味着样本在基于真实 maps 做内存访问），就使用 fakemaps。

## 三、总结

本篇的重要是 maps，请读者务必注意这个问题。之后会补充几个实例分析，增进大家对这个问题的理解。






