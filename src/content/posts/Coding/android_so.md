---
title: 安卓开发学习_so层
published: 2026-08-10
description: 安卓开发学习_so层
image: ./cover.jpg
tags: [编程]
category: 编程学习
draft: false
---

# 前言
内容由ai生成

# 初识
1. so开发CMakeLists
2. so的加载过程
3. native函数声明
4. JNI函数的静态注册规则
5. JNIEnv, jobject/jclass
6. NewstringUtf
7. extern "C" JNIEXPORT jstring JNICALL
8. 指定只编译arm64架构


# java, JNI, c/c++数据类型
在 Android NDK 开发（Java 与 C/C++ 交互）中，数据类型的转换是核心基础。Java 是强类型、基于虚拟机的语言，而 C/C++ 是底层系统语言，它们的内存模型和数据结构完全不同。**JNI (Java Native Interface)** 提供了一套中间层类型（如 `jint`, `jobject`）来桥接两者。

下面我将分 **基础数据类型**、**引用数据类型（String/Array）**、**自定义对象** 三个层级，为你详细拆解映射关系和转换代码。

---

### 1. 基础数据类型（直接映射）

对于基本类型，JNI 定义了与 Java 一一对应的别名。它们在 C/C++ 中是简单的值拷贝，**无需转换，直接使用**。

| Java 类型 | JNI 类型 (C/C++) | 字面量描述符 | 转换方式 |
| :--- | :--- | :--- | :--- |
| `boolean` | `jboolean` | `Z` | 直接使用，C++ 中对应 `unsigned char` |
| `byte` | `jbyte` | `B` | 直接使用，对应 `signed char` |
| `char` | `jchar` | `C` | 直接使用，对应 `unsigned short` (Unicode) |
| `short` | `jshort` | `S` | 直接使用 |
| `int` | `jint` | `I` | 直接使用 |
| `long` | `jlong` | `J` | 直接使用（注意：在 64 位系统中仍为 64 位） |
| `float` | `jfloat` | `F` | 直接使用 |
| `double` | `jdouble` | `D` | 直接使用 |
| `void` | `void` | `V` | 无 |

**示例（Native 方法实现）**：
```cpp
// Java 声明: public native int add(int a, int b);
JNIEXPORT jint JNICALL Java_com_example_NativeLib_add
  (JNIEnv *env, jobject thiz, jint a, jint b) {
    // 直接当作 C++ int 使用，无需任何转换
    return a + b;
}
```

---

### 2. 引用数据类型（需要手动转换）

#### A. 字符串 (String) 转换：`jstring` <-> `char*`
Java 字符串是 Unicode 对象，C++ 是 `char*`。**必须通过 JNI 函数获取指针，用完必须释放**，否则内存泄漏。

- **Java -> C++ (获取 UTF-8)**：
  ```cpp
  JNIEXPORT jstring JNICALL Java_NativeLib_processString
    (JNIEnv *env, jobject thiz, jstring jstr) {
      
      // 1. 将 jstring 转换为 C 字符串 (UTF-8)
      const char *c_str = env->GetStringUTFChars(jstr, nullptr);
      if (c_str == nullptr) return nullptr; // 内存分配失败
      
      // 2. 模拟操作：转为大写（这里仅演示打印）
      // 注意：c_str 指向的是 JVM 内部内存的拷贝，不要直接修改它
      LOGD("C++ Received: %s", c_str);
      
      // 3. 释放资源（必须！）
      env->ReleaseStringUTFChars(jstr, c_str);
      
      // 4. 返回新的 jstring（若需返回）
      std::string new_str = "Processed: Hello";
      return env->NewStringUTF(new_str.c_str());
  }
  ```

- **关键函数**：
  - `GetStringUTFChars`：获取 UTF-8 码点数组，用于文本交互。
  - `GetStringChars`：获取 UTF-16 码点数组（`const jchar*`），适合处理 Emoji 或 Unicode 特殊字符。

#### B. 数组 (Array) 转换：`jarray` <-> 指针
数组分为基本类型数组（如 `jintArray`）和对象数组（`jobjectArray`）。基本类型数组支持**临界区 (Critical)** 和 **区域拷贝 (Region)** 两种方式。

- **获取基本类型数组元素（推荐 `GetIntArrayRegion`，安全无内存泄漏）**：
  ```cpp
  JNIEXPORT jint JNICALL Java_NativeLib_sumArray
    (JNIEnv *env, jobject thiz, jintArray jarray) {
      
      jsize len = env->GetArrayLength(jarray);
      
      // 方式 A：使用 Region 拷贝（推荐，无内存泄漏风险，复制开销小）
      jint *buffer = new jint[len];
      env->GetIntArrayRegion(jarray, 0, len, buffer);
      
      jint sum = 0;
      for (int i = 0; i < len; i++) sum += buffer[i];
      delete[] buffer;
      
      // 方式 B：获取直接指针（性能极高，但需手动 Release）
      jint *elems = env->GetIntArrayElements(jarray, nullptr);
      // ... 操作 elems ...
      env->ReleaseIntArrayElements(jarray, elems, JNI_ABORT); 
      // JNI_ABORT: 不将修改同步回 Java（只读），JNI_COMMIT: 同步并释放
      return sum;
  }
  ```

- **操作对象数组（`jobjectArray`）**：
  ```cpp
  JNIEXPORT void JNICALL Java_NativeLib_handleStrings
    (JNIEnv *env, jobject thiz, jobjectArray stringArray) {
      
      jsize len = env->GetArrayLength(stringArray);
      for (int i = 0; i < len; i++) {
          // 获取每个元素（这是一个 jstring）
          jstring j_str = (jstring)env->GetObjectArrayElement(stringArray, i);
          const char *c_str = env->GetStringUTFChars(j_str, nullptr);
          LOGD("Element %d: %s", i, c_str);
          env->ReleaseStringUTFChars(j_str, c_str);
          // 注意：不需要删除 j_str 局部引用，但若循环量巨大，需手动管理局部引用
      }
  }
  ```

---

### 3. 自定义对象（访问字段和方法）

这是最复杂的转换。JNI 通过 **类引用 (jclass)**、**字段ID (jfieldID)** 和 **方法ID (jmethodID)** 来实现读写和调用。

#### A. 映射结构（Java 类 -> C++ 结构）
假设 Java 类：
```java
package com.example;
public class Person {
    public String name;
    private int age;
    public int getAge() { return age; }
}
```

**Native 代码中操作该对象**：

```cpp
JNIEXPORT void JNICALL Java_NativeLib_modifyPerson
  (JNIEnv *env, jobject thiz, jobject personObj) {
      
    // 1. 获取类引用
    jclass personClass = env->FindClass("com/example/Person");
    if (personClass == nullptr) return; // 类未找到
    
    // 2. 获取字段 ID（依据签名）
    jfieldID nameField = env->GetFieldID(personClass, "name", "Ljava/lang/String;");
    jfieldID ageField = env->GetFieldID(personClass, "age", "I");
    
    // 3. 读取字段值
    jstring jName = (jstring)env->GetObjectField(personObj, nameField);
    const char *cName = env->GetStringUTFChars(jName, nullptr);
    jint age = env->GetIntField(personObj, ageField);
    
    // 4. 修改字段值
    jstring newName = env->NewStringUTF("Modified");
    env->SetObjectField(personObj, nameField, newName);
    env->SetIntField(personObj, ageField, age + 10);
    
    // 5. 调用 Java 方法（通过方法 ID）
    jmethodID getAgeMethod = env->GetMethodID(personClass, "getAge", "()I");
    jint returnedAge = env->CallIntMethod(personObj, getAgeMethod);
    
    // 6. 释放资源
    env->ReleaseStringUTFChars(jName, cName);
    // 注：局部引用如 personClass, jName 会在 native 函数返回时自动释放
}
```

**关键要素**：
1.  **`FindClass`**：获取类，路径用 `/` 替代 `.`。
2.  **`GetFieldID` / `GetMethodID`**：第三个参数是**签名 (Signature)**。
    - 对象签名：`L` + 包名路径 + `;`，例如 `"Ljava/lang/String;"`
    - 数组签名：`[I` (int数组), `[Ljava/lang/String;` (字符串数组)
    - 方法签名：`"(参数)返回值"`，例如 `"()I"` 表示无参返回 int，`"(II)V"` 表示两个 int 参数无返回。

---

### 4. 深入理解 JNI 类型签名 (Signature)

在查找字段和方法时，签名是必须的。建议使用 `javap -s -p YourClass.class` 自动生成。

| Java类型 | 签名描述符 |
| :--- | :--- |
| `int` | `I` |
| `long` | `J` |
| `String` | `Ljava/lang/String;` |
| `Object[]` | `[Ljava/lang/Object;` |
| `int[][]` | `[[I` |
| 方法 `void f(int a, boolean b)` | `(IZ)V` |
| 方法 `String f()` | `()Ljava/lang/String;` |

---

### 5. 异常处理与内存管理（必知必会）

1.  **异常检查**：JNI 函数调用失败时（如 `FindClass` 找不到），不一定会崩溃，但会挂起异常。必须检查并清除：
   ```cpp
   if (env->ExceptionCheck()) {
       env->ExceptionDescribe(); // 打印堆栈
       env->ExceptionClear();    // 清除异常，防止后续代码崩溃
       return;
   }
   ```

2.  **内存泄漏（Get/Release 配对）**：
   - 调用 `GetStringUTFChars` / `GetIntArrayElements` 等 **Must** 调用对应的 `Release`。
   - 获取的 `jclass`、`jobject` 是局部引用，Native 函数返回后自动释放。但如果想**在 Native 层保存对象供后续使用**（如后台线程），必须使用 `env->NewGlobalRef(jobj)` 创建全局引用，并在不用时 `env->DeleteGlobalRef()`。

3.  **临界区 Get/Release**：`GetStringCritical` / `GetPrimitiveArrayCritical` 会阻止 GC，效率极高。但在它们和对应的 `Release` 之间，**绝对不能调用任何阻塞操作或 JNI 函数**，否则会导致死锁。

---

### 💎 总结对照表（一图流）

| 场景 | Java 端 | Native 层 (C++) | 转换核心函数 |
| :--- | :--- | :--- | :--- |
| **传值** | `int a` | `jint a` | 无需转换，直接 `a` |
| **传字符串** | `String str` | `jstring jstr` | `GetStringUTFChars` / `NewStringUTF` |
| **传数组 (读取)** | `int[] arr` | `jintArray jarr` | `GetIntArrayElements` / `GetIntArrayRegion` |
| **返回数组** | `int[]` | `jintArray` | `NewIntArray` + `SetIntArrayRegion` |
| **读/写对象属性** | `Person p` | `jobject p` | `GetFieldID` + `GetObjectField` / `Set<Type>Field` |
| **调用对象方法** | `p.getName()` | `jobject p` | `GetMethodID` + `Call<Type>Method` |

**一句话技巧**：
- **基础类型**：直接当成 C++ 变量用。
- **引用类型 (String/Array)**：用 `GetXXX` 获取 C 指针，操作完必须 `Release`。
- **自定义对象**：把 JNI 当作“反射工具”，通过 `ID` 来读写，而不是直接操作内存。





# so常用log输出
包含头文件`#include <android/log.h>`
然后直接使用`__android_log_print()`
```c
__android_log_print(int priority, const char* tag, const char* format, ...)
```
- **`priority`**：日志级别，对应 Java 层的 `Log.v()`、`Log.d()` 等[](https://tsecurity.de/de/2668103/IT+Programmierung/Android+NDK%3A+Implementing+Logging+with+JNI+%26amp%3B+C%2B%2B/de/2/Startseite/de/245520/IT+Programmierung/)。
    
- **`tag`**：日志标签，方便在 Logcat 中过滤[](https://blog.csdn.net/wolf0706/article/details/132319159)。
    
- **`format`** 和 **`...`**：格式化字符串和可变参数，用法和 `printf` 一样
如
```c
// 打印一条 DEBUG 级别的日志
__android_log_print(ANDROID_LOG_DEBUG, "MyNativeTag", "变量 x 的值是: %d", x);
```


# 线程
```c
int pthread_create(pthread_t *thread, 
                   const pthread_attr_t *attr, 
                   void *(*start_routine) (void *), 
                   void *arg);
```
- **`pthread_t *thread`（输出参数）**：
    
    - 这是一个指向 `pthread_t` 类型变量的指针。函数执行成功后，系统会为该线程生成一个唯一的线程 ID，并写入这个指针指向的内存地址。
        
    - 你可以用它来后续操作该线程（比如 `pthread_join` 等待它结束，或 `pthread_cancel` 取消它）。
        
- **`const pthread_attr_t *attr`（输入参数）**：
    
    - 用于设置线程的属性（如栈大小、调度策略、是否分离等）。
        
    - **绝大多数场景直接传 `NULL`**，表示使用默认属性（默认栈大小约 1MB，默认是“可连接”状态）。如果需要自定义，需先调用 `pthread_attr_init` 初始化属性对象。
        
- **`void *(*start_routine) (void *)`（核心入口）**：
    
    - 这是一个**函数指针**，指向新线程要执行的函数。
        
    - 它的签名固定为：接收一个 `void*` 泛型指针作为入参，返回一个 `void*` 泛型指针。
        
    - 这意味着你可以通过 `arg` 传入任意类型的数据，并返回任意类型的结果（通常用于返回状态码或结果对象）。
        
- **`void *arg`（输入参数）**：
    
    - 这是传递给 `start_routine` 函数的参数。你可以传入任意数据的地址（如整型、结构体、类对象指针）。
        
    - **极度危险警告**：千万**不要**传递一个局部变量的地址（如栈上的临时对象）！因为新线程执行时，当前函数可能已经返回，该栈内存已被释放，导致野指针崩溃。必须传递堆上的内存（如 `malloc` 分配）或全局静态变量。


线程创建成功后，必须决定它的资源回收方式，否则会造成**内存泄漏**（线程私有栈和上下文无法释放）。

- **方案 A：`pthread_join`（同步等待）**
    
    - 调用 `pthread_join(thread_id, NULL)` 会**阻塞**当前线程，直到目标线程执行完毕。
        
    - 适用于需要获取线程返回结果，或者保证线程彻底结束再继续执行的场景。
        
    - 注意：一个线程只能被 `join` 一次，且必须未被 `detach`。
        
- **方案 B：`pthread_detach`（分离态）**
    
    - 调用 `pthread_detach(thread_id)` 后，该线程结束时会**自动释放所有资源**，无需也无法调用 `join`。
        
    - 适用于“发射后不管”的后台任务（如日志写入、异步数据上报）。
        
    - 建议在 `pthread_create` 成功后立即调用 `pthread_detach`，以防遗漏。


# JNI_OnLoad

```c
JNIEXPORT jint JNICALL JNI_OnLoad(JavaVM* vm, void* reserved) {
    JNIEnv* env = NULL;
    if (vm->GetEnv((void**)&env, JNI_VERSION_1_6) != JNI_OK) {
        return JNI_ERR;
    }
    // 在这里做动态注册...
    return JNI_VERSION_1_6; // 必须返回版本号
}
```
**触发瞬间**：当你在 Java 层调用 `System.loadLibrary("your_so")` 时，Android 系统会将该 `.so` 文件加载到进程内存中。**在 `loadLibrary` 函数返回之前**，系统会立即查找并调用 `.so` 中的 `JNI_OnLoad` 函数。

**三个关键特征**：

1. **单次执行**：整个进程生命周期内，`JNI_OnLoad` **只会执行一次**（库被加载一次，除非进程重启）。
    
2. **阻塞加载**：如果 `JNI_OnLoad` 中执行耗时操作（如网络请求、大量计算），`System.loadLibrary()` 会一直阻塞，直至 `JNI_OnLoad` 返回。**如果在 UI 线程调用，会导致界面卡顿（ANR）**。
    
3. **已挂载 JVM**：此时 `JNI_OnLoad` 收到的 `JNIEnv*` 指针**已经有效**，可以直接调用 Java 方法，**不需要**调用 `AttachCurrentThread`（因为它运行在调用 `loadLibrary` 的线程，该线程本身已挂载）。
    

**主要用途**：

- 使用 `RegisterNatives` 动态注册 Native 方法（将 C++ 函数指针与 Java native 方法名绑定），避免函数名过长（`Java_包名_类名_方法名`）。
    
- 缓存 `JavaVM*` 全局变量，供后续其他 Native 线程使用。
    
- 检查运行环境版本。



# JavaVM
## 简介

- **它是进程单例**：在 Android 的一个进程中，只有一个 `JavaVM` 对象[](https://blog.csdn.net/qq_45649553/article/details/157473088)。
    
- **它是“函数表”的封装**：无论是 C 的指针还是 C++ 的类，本质都是对 `JNIInvokeInterface` 函数表的操作[](https://cloud.tencent.com.cn/developer/article/1666743?from=15425)。
    
- **它是线程共享的**：`JavaVM` 对象本身可以在进程内的所有线程间自由共享[](http://article.itxueyuan.com/oxRaD)。
    
- **它的核心任务是管理线程和 JNI 环境**：通过它提供的 `AttachCurrentThread`、`DetachCurrentThread` 和 `GetEnv` 等方法，来管理 Native 线程与 JVM 的关系，并获取线程私有的 `JNIEnv` 指针[](https://cloud.tencent.com.cn/developer/article/2246468?policyId=1004)。
    
- **获取它的两种方式**：通常在 `JNI_OnLoad` 函数中获取并保存[](http://article.itxueyuan.com/oxRaD)[](https://blog.csdn.net/qq_45649553/article/details/157473088)，或在任何拥有有效 `JNIEnv` 指针的地方通过 `GetJavaVM` 方法获取[](https://blog.csdn.net/qq_45649553/article/details/157473088)。
    

理解 `JavaVM` 的结构，是安全、高效地进行跨线程 JNI 编程的基石。

## 在C/C++区别
在 C 语言中，`JavaVM` 被定义为一个指向 `JNIInvokeInterface` 结构体的指

```c
// C 语言环境下的简化定义
typedef const struct JNIInvokeInterface* JavaVM;

// JNIInvokeInterface 结构体，包含函数指针
struct JNIInvokeInterface {
    void* reserved0;
    void* reserved1;
    void* reserved2;
    jint (*DestroyJavaVM)(JavaVM*);
    jint (*AttachCurrentThread)(JavaVM*, JNIEnv**, void*);
    jint (*DetachCurrentThread)(JavaVM*);
    jint (*GetEnv)(JavaVM*, void**, jint);
    jint (*AttachCurrentThreadAsDaemon)(JavaVM*, JNIEnv**, void*);
};
```

在 C++ 中，`JavaVM` 被封装成了一个类 `_JavaVM`[](http://article.itxueyuan.com/oxRaD)。它内部持有一个指向 `JNIInvokeInterface` 的函数表指针，并对接口方法进行了面向对象的封装

```c
// C++ 语言环境下的简化定义
struct _JavaVM {
    const struct JNIInvokeInterface* functions; // 核心：指向函数表的指针

    // C++ 封装的方法，内部调用 functions 表中的对应函数
    jint AttachCurrentThread(JNIEnv** p_env, void* thr_args) {
        return functions->AttachCurrentThread(this, p_env, thr_args);
    }
    // ... 其他方法类似
};
```

无论哪种形式，最终调用的都是 `JNIInvokeInterface` 函数表中定义的函数

## JavaVM常用方法

|方法|是否常用|核心用途|Android 禁忌|
|---|---|---|---|
|**GetEnv**|**极高（必用）**|判断当前线程是否有 JNI 能力|无|
|**AttachCurrentThread**|**高（子线程必用）**|给 Native 线程“上户口”|不要在 Java 回调线程里调用|
|**DetachCurrentThread**|**高（成对出现）**|释放线程资源，防止内存泄漏|绝不能分离 Java 主线程或 Binder 线程|
|**AttachCurrentThreadAsDaemon**|**极低**|守护线程挂载|无特殊，但普通 Attach 够用|
|**DestroyJavaVM**|**零（禁用）**|卸载虚拟机|**绝对禁止调用**，会导致崩溃|
### 1.`GetEnv`（最高频：获取 JNI 环境）

**作用**：检查当前线程是否已经挂载（Attach）到 Java 虚拟机，如果已挂载则返回有效的 `JNIEnv*`。

**函数签名（C++ 风格）**：


```cpp
jint GetEnv(void** env, jint version);
```

- **`env`**：输出参数，指向 `JNIEnv*` 指针的地址。
    
- **`version`**：请求的 JNI 版本，通常填 `JNI_VERSION_1_6`。
    

**返回值（极其重要）**：

- **`JNI_OK` (0)**：当前线程已挂载，`env` 有效，可以直接使用。
    
- **`JNI_EDETACHED` (-2)**：当前线程**未挂载**，此时调用 `env` 会崩溃，需要手动 `AttachCurrentThread`。
    
- **`JNI_EVERSION` (-3)**：版本号不匹配（极少出现）。
    

**典型用法（判断分支）**：

```cpp

JNIEnv* env = nullptr;
int status = g_jvm->GetEnv((void**)&env, JNI_VERSION_1_6);
if (status == JNI_OK) {
    // 安全：可以调用 Java 方法
} else if (status == JNI_EDETACHED) {
    // 需要调用 AttachCurrentThread
}
```

### 2. `AttachCurrentThread`（次高频：挂载 Native 线程）

**作用**：将当前**纯 Native 线程**（如 `pthread_create` 创建的）挂载到 Java 虚拟机，使其获得调用 Java 方法的能力。

**函数签名（C++ 风格）**：


```cpp

jint AttachCurrentThread(JNIEnv** p_env, void* thr_args);
```

- **`p_env`**：输出参数，挂载成功后返回当前线程有效的 `JNIEnv*`。
    
- **`thr_args`**：通常传 `NULL`（表示使用默认 Java 线程属性）。
    

**关键坑点**：

- 一个线程可以多次调用 `AttachCurrentThread`，但**必须与 `DetachCurrentThread` 调用次数相等**（引用计数机制）。建议一个线程生命周期内只 Attach 一次。
    
- **Java 层创建的线程（如 `new Thread`）回调 Native 方法时，不需要 Attach！** 只有你自己用 `pthread_create` 开的新线程才需要。
    

---

### 3. `DetachCurrentThread`（必须配对：解除挂载）

**作用**：将当前 Native 线程从 Java 虚拟机分离，释放 JVM 为它分配的资源（如 `JNIEnv` 对应的局部引用表）。

**函数签名（C++ 风格）**：


```cpp

jint DetachCurrentThread();
```

**绝对铁律**：所有调用过 `AttachCurrentThread` 的线程，在**线程退出前必须调用 `DetachCurrentThread`**。如果忘了，会导致线程局部存储（TLS）无法释放，造成**内存泄漏**，甚至在进程退出时触发 JVM 崩溃（Fatal signal 11）。



# JNI Env

## 1.简介
`JNIEnv` 是 JNI 编程中**最核心、最常用**的“工具包”。如果说 `JavaVM` 是“门禁卡”（用来管理线程和获取环境），那么 **`JNIEnv` 就是“万能工具箱”**——Native 代码调用 Java 层的一切操作（创建对象、调用方法、访问字段、抛出异常）都必须通过它来完成。

要彻底理解 `JNIEnv`，你需要抓住它的**三大本质**：**线程私有性**、**函数表结构** 和 **局部引用管理者**。

---

### 1. 本质：它到底是什么？（C vs C++ 视角）

和 `JavaVM` 一样，`JNIEnv` 的实现也因 C/C++ 语言而异，但底层都是**函数指针表**。

-   **在 C 语言中**：`JNIEnv` 是一个指向 `JNINativeInterface` 结构体的**一级指针**。调用函数时必须解引用，并手动传入 `env` 作为第一个参数。
    ```c
    // C 语法
    (*env)->FindClass(env, "java/lang/String");
    ```

-   **在 C++ 语言中**：`JNIEnv` 是一个封装了函数表的类（`_JNIEnv`）。它提供了成员函数，内部自动处理 `this` 指针，写法更简洁。
    ```cpp
    // C++ 语法（最常用）
    env->FindClass("java/lang/String");
    ```
    > **注意**：无论哪种写法，最终调用的底层函数表（`JNINativeInterface`）是完全一致的。

---

### 2. 核心职责：它能干什么？（强大的函数表）

`JNIEnv` 的函数表里包含了 **200 多个** 函数指针，按功能可以分成以下几大类：

| 分类         | 典型函数                                                         | 作用                               |
| :--------- | :----------------------------------------------------------- | :------------------------------- |
| **类与对象操作** | `FindClass`, `NewObject`, `GetObjectClass`                   | 加载 Java 类、创建 Java 对象实例           |
| **方法调用**   | `GetMethodID`, `CallVoidMethod`, `CallIntMethod`             | 获取方法 ID，并调用 Java 层的实例方法          |
| **静态方法调用** | `GetStaticMethodID`, `CallStaticVoidMethod`                  | 调用 Java 层的静态方法                   |
| **字段访问**   | `GetFieldID`, `GetIntField`, `SetIntField`                   | 读取和修改 Java 对象的成员变量               |
| **静态字段访问** | `GetStaticFieldID`, `GetStaticIntField`                      | 读取和修改 Java 类的静态变量                |
| **字符串处理**  | `NewStringUTF`, `GetStringUTFChars`, `ReleaseStringUTFChars` | 在 Native 层创建/解析 Java 字符串（处理编码转换） |
| **数组操作**   | `GetArrayLength`, `GetIntArrayElements`                      | 操作 Java 基本类型数组和对象数组              |
| **异常管理**   | `Throw`, `ThrowNew`, `ExceptionCheck`, `ExceptionClear`      | 抛出 Java 异常，或检查 JNI 调用是否发生异常      |
| **引用管理**   | `NewLocalRef`, `DeleteLocalRef`, `NewGlobalRef`              | 管理 JNI 引用的生命周期（避免内存泄漏）           |
| **类型转换**   | `IsInstanceOf`, `GetDirectBufferAddress`                     | 类型判断和 NIO Buffer 操作              |

> **关键点**：`JNIEnv` 不带“状态”，它只是一个“函数跳转表”的句柄。你调用 `env->FindClass`，它底层会通过 `env` 中绑定的当前线程数据去 JVM 内部查找。

---

### 3. 最重要的特性：线程私有性（黄金法则）

**`JNIEnv` 是线程局部存储（TLS）的，绝对禁止跨线程传递和使用！**

-   如果你在**线程 A** 中拿到了 `JNIEnv*`，把它保存为全局变量，然后在**线程 B** 中去调用 `env->CallVoidMethod`，**程序会立即崩溃**（SIGSEGV）。
-   **原因**：`JNIEnv` 内部维护了当前线程的局部引用表、异常挂起状态等。不同线程的 `JNIEnv` 指向的内存区域完全不同。

**正确的获取方式**：
-   **在 Java 调用的 Native 函数中**：作为第一个参数直接传入，直接使用。
-   **在 `pthread` 或 `std::thread` 创建的纯 Native 线程中**：必须先通过 `JavaVM->AttachCurrentThread()` 挂载线程，挂载成功时返回的 `JNIEnv*` 才是该线程的有效环境。

---

### 4. 引用的“超市管理员”（内存管理核心）

`JNIEnv` 负责管理 JNI 引用，这是导致内存泄漏的高发区。它主要管理三种引用：

-   **局部引用（Local Reference）**：
    -   默认由 `FindClass`、`NewObject`、`GetObjectField` 等返回。
    -   **生命周期**：仅在当前 Native 函数执行期间有效，函数返回后自动释放。
    -   **坑点**：如果在循环中大量创建局部引用（如循环 10000 次创建对象），不及时删除会导致**局部引用表溢出**，崩溃报错 `ReferenceTable overflow`。
    -   **解决**：在循环中手动调用 `env->DeleteLocalRef(local_ref)`。

-   **全局引用（Global Reference）**：
    -   通过 `env->NewGlobalRef(local_ref)` 创建。
    -   **生命周期**：手动管理，直到调用 `env->DeleteGlobalRef()` 才会释放。适用于缓存 `jclass` 或 `jmethodID` 供后续跨线程使用。

-   **弱全局引用（Weak Global Reference）**：
    -   通过 `env->NewWeakGlobalRef` 创建。允许对象被 GC 回收，适合缓存不常变的大对象。

---

### 5. 获取 `JNIEnv*` 的两种标准姿势

**姿势一：在 JNI 函数参数中直接拿（Java 线程回调）**
```cpp
JNIEXPORT void JNICALL Java_com_example_test(JNIEnv* env, jobject thiz) {
    // 直接使用 env，无需任何附加操作
    jclass cls = env->GetObjectClass(thiz);
}
```

**姿势二：在纯 Native 子线程中通过 JavaVM 获取（最安全写法）**
```cpp
JavaVM* g_jvm; // 假设已在 JNI_OnLoad 中全局保存

void native_thread_task() {
    JNIEnv* env = nullptr;
    // 1. 先尝试获取
    int status = g_jvm->GetEnv((void**)&env, JNI_VERSION_1_6);
    
    if (status == JNI_EDETACHED) {
        // 2. 未挂载则挂载
        g_jvm->AttachCurrentThread(&env, nullptr);
    }
    
    // 3. 干活...
    if (env) {
        // 调用 Java 方法...
    }
    
    // 4. 必须分离（仅当是 Native 线程且 Attach 过）
    g_jvm->DetachCurrentThread();
}
```

---

### 6. 极易踩的坑（血泪经验总结）

1.  **不要缓存 `JNIEnv*`**：绝对不要定义一个全局的 `JNIEnv* g_env` 供所有线程用！这样做必崩。
2.  **异常需要手动清除**：在 JNI 中调用 Java 方法后（如 `CallVoidMethod`），如果 Java 层抛出了异常，Native 层不会自动清空。你必须调用 `env->ExceptionCheck()` 检测，并调用 `env->ExceptionClear()` 清除，否则再次调用任何 JNI 函数都会崩溃。
3.  **`GetStringUTFChars` 必须配对 `Release`**：获取 UTF-8 字符串后（尤其获取了 `const char*` 指针），在操作完后必须调用 `ReleaseStringUTFChars`，否则会发生内存泄漏。
4.  **`FindClass` 的坑**：`FindClass` 依赖当前线程的类加载器。在 Java 主线程回调中没问题，但在 Native 子线程中 `AttachCurrentThread` 后，如果类加载器上下文丢失，`FindClass` 可能返回 `null`。此时需要缓存 `jclass` 为全局引用，或通过 `GetObjectClass` 从传入的对象获取。

---

### 一句话总结
> **`JNIEnv` 是绑定当前线程的“JNI 操作手册”，所有的 Java 互调都靠它。它不能跨线程传，创建了引用要记得删，发生了异常要记得清，这就是 JNI 编程的三大生死线。**



# init,init_array函数运行实际
在 Android 系统中，`init` 和 `init_array` 是 SO（共享库）文件**最早**的代码执行入口，其执行时机**早于**我们熟知的 `JNI_OnLoad` 函数[](https://www.secpulse.com/archives/65690.html)。

具体来说，它们的执行发生在 **动态链接器（Linker）** 完成 SO 文件的加载、内存映射和符号重定位之后。

### 执行时机与顺序

当一个 SO 文件被加载时，系统会按照一个严格的顺序来调用其中的初始化代码。整体流程如下：

1. **`System.loadLibrary()` 调用**：一切始于 Java 层的 `System.loadLibrary()` 调用[](https://www.secpulse.com/archives/65690.html)。
    
2. **Linker 加载 SO**：Android 的动态链接器 (`linker`) 负责将 SO 文件加载到进程内存空间，并完成所有符号解析和重定位工作[](https://www.secpulse.com/archives/65690.html)。
    
3. **执行 `DT_INIT` (即 `_init` 函数)**：这是 SO 中最先执行的代码[](https://huaweicloud.csdn.net/65387de091098a10412739cd.html)。它通常由编译器生成，用于执行一些最基础的、不依赖于其他初始化代码的配置。在早期的 ELF 规范中，开发者可以通过定义一个 `_init` 函数来插入自定义的初始化逻辑[](https://www.cnblogs.com/xunbu7/p/6902331.html)。**注意**：`_init` 函数是在 `init_array` **之前**执行的。
    
4. **执行 `DT_INIT_ARRAY` (即 `.init_array` 中的函数)**：在 `_init` 执行完毕后，Linker 会按顺序调用 `.init_array` 段中存储的所有函数指针[](https://blog.csdn.net/justdoyaya/article/details/145798786)。这些函数是 SO 初始化逻辑的主要载体[](https://blog.csdn.net/justdoyaya/article/details/145798786)。
    
5. **执行 `JNI_OnLoad` (如果存在)**：当所有初始化函数执行完毕，Linker 会检查 SO 是否导出了 `JNI_OnLoad` 函数。如果存在，则会在此刻调用它[](https://blog.csdn.net/weixin_30249203/article/details/96674925)。


## 编写
在 Android SO（共享库）开发中，`init` 和 `init_array` 是你**最早可以控制的代码执行入口**。它们的执行时机早于 `JNI_OnLoad`，甚至早于 `main` 函数（对于可执行文件而言）。

既然你已经理解了它们的执行时机（`DT_INIT` -> `DT_INIT_ARRAY` -> `JNI_OnLoad`），接下来重点在于**如何正确编写代码**、**如何控制优先级**，以及**Android 环境下致命的限制规则**。

---

### 1. 如何定义 `init` 和 `init_array` 函数

现代 Android NDK 开发中，**不推荐**直接定义 `_init` 函数（这是旧式 ELF 标准，且与 `-nostdlib` 等编译选项冲突）。**官方推荐且唯一通用的方式**是使用 GCC/Clang 的 **`__attribute__((constructor))`** 属性。

#### 基础用法：定义 `init_array` 中的函数
所有被 `constructor` 修饰的函数，其函数指针都会被编译器自动收集到 `.init_array` 段中。

```cpp
#include <android/log.h>
#include <dlfcn.h>

// 该函数会被放入 .init_array，在 SO 加载时执行
__attribute__((constructor))
void my_early_init() {
    // 注意：此时 Android 的 log 系统已经可用，可以直接打印
    __android_log_print(ANDROID_LOG_DEBUG, "MySO", "init_array 执行，时间最早！");
}
```

#### 控制执行顺序（优先级）
如果你有多个初始化函数，可以用 `(优先级数字)` 控制顺序。**数字越小，执行越早**。

```cpp
// 优先级 101：最先执行
__attribute__((constructor(101)))
void init_logger() {
    // 初始化日志系统
}

// 优先级 102：其次执行
__attribute__((constructor(102)))
void init_crypto() {
    // 解密关键数据
}
```

> **警告**：系统级别的 `constructor` 通常占用 0-100 的优先级，**请使用 >100 的数字**，避免与系统库内部函数冲突。

---

### 2. 高级用法：主动获取 `init` 和 `init_array` 的执行地址

如果你需要更底层的控制（例如手写链接器脚本），可以在 C++ 代码中声明外部符号来观察它们，但**几乎用不到**。日常开发只需 `__attribute__((constructor))`。

---

### 3. 实战：`init_array` 通常用来做什么？（三大场景）

由于它执行时 **`JNI_OnLoad` 尚未被调用**，`JavaVM*` 还是空指针，因此它**不适合**做 JNI 相关的初始化（如缓存 `jclass`）。它最适合做：

1.  **Native 层基础设施初始化**：
    - 初始化 `logcat` 日志过滤系统。
    - 初始化第三方 C/C++ 单例（如 `Protobuf`、`FlatBuffers` 的全局池）。
    - 设置全局的信号量/互斥锁（`pthread_mutex_init`）。
2.  **安全与反调试（早期防护）**：
    - 检测 `ptrace` 或 `TracerPid`，若发现调试器则立即崩溃或退出（反调试的黄金窗口期，因为还未暴露 Java 层符号）。
    - 解密存储在 `.rodata` 中的硬编码字符串。
3.  **Hook 与注入适配**：
    - 在 `init_array` 中替换系统库函数（如 `malloc`、`free`）的符号表（PLT Hook），因为此时动态链接器刚完成重定位，是最佳替换时机。

---

### 4. ⚠️ Android 环境下的“三不”铁律（致命陷阱）

在 `init` / `init_array` 中，你的操作受到极其严格的限制，触犯任何一条都会导致 **`dlopen` 卡死、死锁或直接 SIGSEGV 崩溃**。

#### 铁律一：绝对不能调用 `JNI` 函数
-   **错误示例**：`env->FindClass("com/example/MyClass");`
-   **后果**：`JavaVM` 尚未与当前线程关联（未 Attach），`JNIEnv` 为空，调用必崩。
-   **正确做法**：纯 Native 业务放这里；JNI 相关初始化（如缓存 Class）**必须**放在 `JNI_OnLoad` 中。

#### 铁律二：绝对不能调用 `dlopen` 或 `dlsym`（递归加载锁问题）
-   **错误示例**：在 `init_array` 中调用 `dlopen("libanother.so", RTLD_NOW)`。
-   **后果**：当前线程正持有动态链接器（Linker）的内部锁（`ld.so` 锁），再次调用 `dlopen` 会导致**死锁（Deadlock）**，进程永久卡死。
-   **正确做法**：如果有加载依赖库的需求，放到 `JNI_OnLoad` 中去做，或者交由 Java 层的 `System.loadLibrary` 处理依赖顺序。

#### 铁律三：绝对不能执行耗时操作或创建线程（ANR 风险）
-   **错误示例**：在 `init_array` 中 `sleep(5)` 或 `pthread_create`。
-   **后果**：`System.loadLibrary` 会**同步阻塞**等待 `init_array` 执行完毕。如果耗时超过 5 秒（主线程加载），会触发 ANR（应用无响应）或 Watchdog 超时。
-   **正确做法**：只做**轻量级**初始化（如赋值、简单计算）。如果需要后台线程，请在 `JNI_OnLoad` 中创建。

---

### 5. 如何验证我的 `init_array` 是否执行了？

使用 NDK 自带的 `readelf` 工具查看 SO 的段信息，确认函数是否被正确编译进 `.init_array`：

```bash
# 在终端执行
readelf -d your_lib.so | grep INIT
```
输出中包含 `INIT_ARRAY` 和 `INIT_ARRAYSZ` 即为成功。

---

### 6. 极特殊技巧：绕过 `__attribute__` 手写 `_init`（不推荐）

除非你在写极简的 `crt0` 启动代码，否则不要使用这种方式。它容易与 Android 的 `libc.so` 初始化流程冲突。
```cpp
// 极不推荐，且在某些 NDK 版本下编译警告
extern "C" void _init() {
    // ...
}
```
**建议**：一律使用 `__attribute__((constructor))`。

---

### 总结一句话

> **`init_array`（即 `__attribute__((constructor))`）是 SO 加载阶段最底层的初始化钩子，用于执行不依赖 JVM 的基础 Native 工作。它早于 `JNI_OnLoad`，**严禁**调用 JNI 函数、严禁递归 `dlopen`、严禁耗时阻塞。**将 JNI 相关初始化（类缓存、方法 ID）留在 `JNI_OnLoad` 中，将纯 C/C++ 基础设施放在 `init_array` 中，这是最安全的分工策略。**

如果你需要在 `init_array` 中运行 JNI 相关逻辑（如获取 `JNIEnv`），唯一的方案是保存 `JavaVM*`（但此时无法获取），所以必须放弃。如果非要不可，请封装成延迟初始化，在第一次被 Java 调用时才执行。😊



# so 函数注册
### 1. 静态注册（约定式，基于命名规则）

这是最基础的方式，NDK 早期默认采用。它依赖 **JVM 的命名查找机制**。

- **机制**：当 Java 代码第一次调用 `native` 方法时，JVM 会根据**固定的命名规则**（`Java_包名_类名_方法名`）去 `.so` 库的符号表（Symbol Table）里查找对应的 C/C++ 函数指针。
    
- **写法**：
    
    
    
```cpp
    // Java 层：com.example.MainActivity.nativeInit()
    // C++ 层必须严格命名为：
    JNIEXPORT void JNICALL Java_com_example_MainActivity_nativeInit(JNIEnv* env, jobject thiz) {
        // 实现代码
    }
```
    

| 优点                                          | 缺点                                                           |
| ------------------------------------------- | ------------------------------------------------------------ |
| **编码简单**：无需额外代码，编译器自动处理符号导出。                | **函数名极长**：包名一长串，写起来繁琐，容易拼错。                                  |
| **IDE 支持好**：Android Studio 可以自动生成桩函数（Stub）。 | **符号暴露风险**：所有函数名明文存在于 `.so` 的导出表中，逆向人员一看便知功能，不利于安全防护。        |
| **无需 `JNI_OnLoad`**：即使不写 `JNI_OnLoad` 也能工作。 | **性能稍差**：首次调用时需要按名称在动态库中查找符号（字符串匹配），有微小开销。                   |
|                                             | **混淆兼容差**：Java 层代码如果被 ProGuard 混淆（改名），Native 函数名必须同步修改，否则崩溃。 |

---

### 2. 动态注册（手动映射，基于 `RegisterNatives`）

这是目前**工业级项目（如抖音、微信、游戏引擎）的主流方案**。它在 SO 加载阶段，通过代码手动将 Java 方法名和 C/C++ 函数指针绑定。

- **机制**：在 **`JNI_OnLoad`** 函数中，调用 `env->RegisterNatives()`，向 JVM 注册一个映射关系表。因为 `JNI_OnLoad` 在 SO 被 `loadLibrary` 时立即执行，所以**在 Java 层调用 native 方法之前**，映射就已经建立好了。
    
- **写法**：
    
    1. 定义 C/C++ 实现函数（命名可以随便取，如 `real_init`）。
        
    2. 构造 `JNINativeMethod` 结构体数组。
        
    3. 在 `JNI_OnLoad` 中调用 `RegisterNatives`。
        


```cpp
// 1. C++ 实现函数（名字随意，无需按规则命名）
void native_init_impl(JNIEnv* env, jobject thiz) {
    // 实现代码
}
int native_add_impl(JNIEnv* env, jobject thiz, int a, int b) {
    return a + b;
}
// 2. 映射表：{"Java方法名", "签名", (void*)C++函数指针}
static const JNINativeMethod gMethods[] = {
    {"nativeInit", "()V", (void*)native_init_impl},
    {"nativeAdd", "(II)I", (void*)native_add_impl},
};
// 3. 在 JNI_OnLoad 中动态注册
JNIEXPORT jint JNICALL JNI_OnLoad(JavaVM* vm, void* reserved) {
    JNIEnv* env = nullptr;
    if (vm->GetEnv((void**)&env, JNI_VERSION_1_6) != JNI_OK) {
        return JNI_ERR;
    }
    // 找到 Java 对应的类
    jclass clazz = env->FindClass("com/example/MainActivity");
    if (clazz == nullptr) {
        return JNI_ERR;
    }
    // 注册映射表（数量计算）
    jint result = env->RegisterNatives(clazz, gMethods, sizeof(gMethods) / sizeof(gMethods[0]));
    if (result < 0) {
        return JNI_ERR; // 注册失败，SO 加载失败
    }
    return JNI_VERSION_1_6;
}

```

|优点|缺点|
|---|---|
|**代码简洁**：C++ 函数名可以短小精悍（如 `init`），无需写繁琐的包名前缀。|**代码量略增**：需要维护映射表结构和 `RegisterNatives` 调用逻辑。|
|**效率更高**：映射在加载期完成，调用时直接跳转，无字符串查找开销。|**易写错签名**：方法签名（`(II)I` 等）写错会导致 `NoSuchMethodError`，需用 `javap -s` 仔细核对。|
|**安全性提升**：C++ 函数名可设为 `static` 并隐藏内部符号，不在 `.so` 导出表中暴露明文 Java 方法名。|**强依赖 `JNI_OnLoad`**：如果 `JNI_OnLoad` 执行失败或未执行，所有 Native 方法调用都将崩溃。|
|**混淆友好**：Java 层混淆后，只需同步更新映射表中的字符串名称即可。|**类加载器陷阱**：`FindClass` 在 `JNI_OnLoad` 中可能受类加载器限制，若找不到类需通过 `NewGlobalRef` 缓存 Class。|

---

### 3. 核心区别对比表

|对比维度|静态注册|动态注册|
|---|---|---|
|**映射建立时机**|首次调用 Native 方法时（懒加载）|SO 加载阶段（`JNI_OnLoad` 执行时）|
|**函数命名要求**|必须严格遵守 `Java_包名_类名_方法名`|无要求，可随意命名|
|**符号可见性**|函数为 `JNIEXPORT`，在导出表中可见|函数可设为 `static`，对外隐藏|
|**性能影响**|首次调用有符号查找开销|无运行时查找开销|
|**代码维护成本**|包名变更需改大量函数名|只需修改映射表中的一个字符串|
|**错误检测**|编译时无法检测，运行时 `UnsatisfiedLinkError`|加载时注册失败会直接报错，提前暴露问题|

---

### 4. 实战建议与坑点（经验之谈）

1. **动态注册是工程首选**：绝大多数成熟项目都使用动态注册。它让 C++ 代码更干净，且配合编译器的 `-fvisibility=hidden` 可以极大增加逆向难度。
    
2. **务必校验 `RegisterNatives` 返回值**：动态注册返回小于 `0` 表示失败（如签名写错），此时一定要返回 `JNI_ERR` 让 SO 加载失败，避免带着错误的映射表运行导致隐晦崩溃。
    
3. **`FindClass` 的类加载器上下文**：在 `JNI_OnLoad` 中调用 `FindClass` 时，如果该 SO 是被系统类加载器加载的，只能找到系统类或主 Dex 类。如果类在插件化 Dex 中，可能需要通过 `env->GetObjectClass(thiz)` 利用传入的对象获取，或者把 `jclass` 缓存为全局引用传递进去。
    
4. **静态注册的“残留”风险**：如果混用两种方式，注意静态注册的函数仍会暴露在符号表中，可能被动态注册覆盖，造成调试混乱。建议统一使用动态注册。


# 将多个cpp文件编译成一个so
将多个 C++ 文件编译成一个 `.so` 文件，最主流、最推荐的方式是使用 **CMake**。当然，传统的 **ndk-build** 方式也同样支持。

下面是两种方式的详细配置方法。

### 方法一：使用 CMake（官方推荐）

在 Android Studio 中，CMake 通过 `CMakeLists.txt` 文件来控制编译过程。

**1. 最直接的方式：在 `add_library` 中列出所有文件**

这是最基础的方法，直接在 `add_library` 指令中，把所有的 `.cpp` 文件按顺序列出来。

```cmake
# 指定最低 CMake 版本
cmake_minimum_required(VERSION 3.22.1)

# 设置项目名称
project("my-native-lib")

# 创建共享库 (SHARED)，并列出所有需要编译的源文件
add_library(
        native-lib                             # 生成的库名称，最终为 libnative-lib.so
        SHARED                                 # 库类型：共享库
        src/main/cpp/file1.cpp                 # 源文件1
        src/main/cpp/file2.cpp                 # 源文件2
        src/main/cpp/subdir/file3.cpp          # 源文件3（可以在子目录）
        # ... 在这里继续添加所有 .cpp 文件
)

# 链接系统库，比如 Android 的 log 库
find_library(log-lib log)
target_link_libraries(native-lib ${log-lib})
```

> **注意**：如果工程很大，源文件很多，这种方式会导致 `CMakeLists.txt` 变得冗长且难以维护。

**2. 更优雅的方式：使用变量或 GLOB**

可以使用 `set` 命令将源文件列表存储在一个变量中，或者使用 `file(GLOB)` 命令自动收集指定目录下的所有源文件。

```cmake
cmake_minimum_required(VERSION 3.22.1)
project("my-native-lib")

# 方式 A：使用 set 命令手动将文件路径赋值给一个变量
set(NATIVE_SRC
        src/main/cpp/file1.cpp
        src/main/cpp/file2.cpp
        src/main/cpp/subdir/file3.cpp
)

# 方式 B：使用 GLOB 自动收集 cpp 目录下所有 .cpp 文件（更便捷）
# file(GLOB NATIVE_SRC "src/main/cpp/*.cpp")

add_library(
        native-lib
        SHARED
        ${NATIVE_SRC}   # 直接使用上面定义的变量
)

find_library(log-lib log)
target_link_libraries(native-lib ${log-lib})
```
使用 `GLOB` 时，添加或删除源文件无需修改 `CMakeLists.txt`。但如果新增了文件，有时需要手动触发一次 Gradle Sync 才能被识别。

**3. 管理复杂项目：使用子目录（`add_subdirectory`）**

对于大型项目，可以使用 `add_subdirectory` 来组织不同模块的 CMake 配置，使主配置文件更清晰。

*   **主 `CMakeLists.txt`**：
    ```cmake
    cmake_minimum_required(VERSION 3.22.1)
    project("my-native-lib")
    
    # 添加子目录，CMake 会去执行子目录下的 CMakeLists.txt
    add_subdirectory(src/main/cpp/module_a)
    add_subdirectory(src/main/cpp/module_b)
    
    # ... 其他配置
    ```
    子目录 `module_a` 下的 `CMakeLists.txt` 可以定义自己的库：
    ```cmake
    # src/main/cpp/module_a/CMakeLists.txt
    add_library(module_a_lib SHARED module_a.cpp)
    target_link_libraries(module_a_lib log)
    ```

### 方法二：使用 ndk-build（传统方式）

如果你的项目使用 `ndk-build`，则需要编写 `Android.mk` 文件。

在 `Android.mk` 中，使用 `LOCAL_SRC_FILES` 变量来指定所有需要编译的源文件。

```makefile
LOCAL_PATH := $(call my-dir)

include $(CLEAR_VARS)

# 指定模块名称，最终生成 libnative-lib.so
LOCAL_MODULE := native-lib

# 指定所有源文件，用空格分隔，支持相对路径
LOCAL_SRC_FILES := file1.cpp \
                   file2.cpp \
                   subdir/file3.cpp

# 链接系统库
LOCAL_LDLIBS := -llog

include $(BUILD_SHARED_LIBRARY)
```

### 总结与建议

*   **首选 CMake**：它是 Android Studio 的官方默认构建工具，与 Gradle 集成更好，配置也更灵活强大。
*   **如何选择**：
    *   **文件少**：直接用 `add_library(... file1.cpp file2.cpp ...)`。
    *   **文件多，且会频繁增删**：用 `file(GLOB ...)` 自动收集。
    *   **项目大，模块多**：用 `add_subdirectory` 进行模块化拆分。

无论使用哪种方式，核心都是在构建配置文件（`CMakeLists.txt` 或 `Android.mk`）中，将你所有需要编译的 `.cpp` 文件路径告诉编译系统。


# so路径的获取
在 Android 开发中，动态获取 so 文件的路径，在 Java 层和 Native 层有完全不同的方法。

### Java 层：获取 so 文件路径

#### 1. 获取存放所有 Native 库的目录：`nativeLibraryDir`

最直接、最标准的方法是使用 `Context.getApplicationInfo().nativeLibraryDir`。它返回应用所有 Native 库所在的**目录**路径。

```java
// 在 Activity 或任何有 Context 的地方
String nativeLibDir = getApplicationContext().getApplicationInfo().nativeLibraryDir;
Log.d("Path", "Native library directory: " + nativeLibDir);
// 输出示例: /data/app/~~xxx/com.example.app-xxx/lib/arm64
```

#### 2. 获取特定库的完整路径：`ClassLoader.findLibrary()`

如果你需要特定 so 文件的完整路径，可以使用 `BaseDexClassLoader` 的 `findLibrary` 方法。

```java
// 获取当前类的 ClassLoader
ClassLoader classLoader = getClass().getClassLoader();
// 假设库名为 "native-lib"
String libPath = ((BaseDexClassLoader) classLoader).findLibrary("native-lib");
Log.d("Path", "Full path to libnative-lib.so: " + libPath);
// 输出示例: /data/app/~~xxx/com.example.app-xxx/lib/arm64/libnative-lib.so
```

这个方法更精确，直接返回目标 so 的完整绝对路径。

#### 3. 其他辅助方法

*   **`System.mapLibraryName(String libname)`**：将库名转换为系统对应的文件名，例如 `"native-lib"` 转为 `"libnative-lib.so"`。
*   **`System.loadLibrary(String libname)`**：这是加载库的方法，它内部使用了 `ClassLoader.findLibrary()` 来查找路径。
*   **`System.getProperty("java.library.path")`**：返回系统默认的库搜索路径，一般不包括应用的私有目录。

---

### Native 层：获取当前 so 文件路径

在 C/C++ 代码中，无法直接调用 Java 的 API。常用的方法有：

#### 1. 使用 `dladdr()`（首选方法）

`dladdr()` 是一个标准函数，可以根据一个函数地址获取包含该函数的共享对象（so 文件）的信息。

```cpp
#include <dlfcn.h>
#include <android/log.h>

void print_self_path() {
    Dl_info dl_info;
    // 使用当前函数地址 (&print_self_path) 或任意此 so 中的函数
    if (dladdr((void*)print_self_path, &dl_info)) {
        // dl_info.dli_fname 就是当前 so 文件的完整路径
        __android_log_print(ANDROID_LOG_DEBUG, "NativePath", "Current SO path: %s", dl_info.dli_fname);
    } else {
        __android_log_print(ANDROID_LOG_ERROR, "NativePath", "dladdr failed");
    }
}
```

**优点**：这是获取自身 so 路径最标准、最可靠的方法，不依赖特定 Android 版本。

#### 2. 解析 `/proc/self/maps`（备选方案）

`/proc/self/maps` 文件记录了当前进程的内存映射，包含了所有加载的 so 文件及其加载地址。

```cpp
#include <fstream>
#include <string>
#include <android/log.h>

void find_so_path_from_maps(const std::string& so_name) {
    std::ifstream maps_file("/proc/self/maps");
    std::string line;
    while (std::getline(maps_file, line)) {
        // 查找包含目标 so 名称的行
        if (line.find(so_name) != std::string::npos) {
            // 解析该行以获取路径
            size_t path_pos = line.find('/');
            if (path_pos != std::string::npos) {
                std::string path = line.substr(path_pos);
                __android_log_print(ANDROID_LOG_DEBUG, "NativePath", "Found SO path: %s", path.c_str());
                // 通常只需要找到第一个匹配项即可退出
                break;
            }
        }
    }
}
```

**注意**：`/proc/self/maps` 的输出格式在不同 Android 版本上可能略有差异，解析时需要考虑兼容性。

---

### 关键注意事项

*   **路径不是固定的**：Android 从 5.0 开始，APK 的安装路径是随机的，硬编码路径不可行。
*   **`android:extractNativeLibs` 的影响**：如果清单文件中设置为 `android:extractNativeLibs="false"`，so 文件可能不提取到文件系统，而是直接从 APK 加载。此时 `nativeLibraryDir` 可能返回 APK 内的路径。
*   **`dlopen()` 需要完整路径**：在 Native 层使用 `dlopen()` 加载其他 so 时，应使用完整路径。直接传递文件名（如 `"libfoo.so"`）会因搜索路径不包含应用目录而失败。
*   **线程安全**：在 Native 子线程中调用这些方法时，注意 JNI 环境的正确获取与挂载。



# so之间的相互调用
`dlsym` 是 Android Native 开发中**运行时动态加载**的核心函数。它的作用非常纯粹：**根据一个字符串（符号名），在已经打开的动态库（so 文件）中查找并返回该符号（函数或全局变量）的内存地址**。

简单来说，`dlopen` 负责“开门”（加载 so），`dlsym` 负责“找人”（获取函数指针）。

### 1. 函数原型与头文件

```cpp
#include <dlfcn.h>

void *dlsym(void *handle, const char *symbol);
```

*   **`handle`**：由 `dlopen()` 返回的库句柄。
*   **`symbol`**：你要查找的函数名或全局变量名（**区分大小写**）。
*   **返回值**：成功返回符号的地址（`void*`），失败返回 `NULL`。

> **致命陷阱：`dlsym` 返回 `NULL` 不一定代表失败！** 如果被查找的符号恰好指向地址 `0`，返回值也是 `NULL`。因此，**正确的错误检查必须调用 `dlerror()`**。

---

### 2. 标准安全用法（模板代码）

```cpp
#include <dlfcn.h>
#include <android/log.h>

// 1. 定义函数指针类型（便于转换）
typedef int (*AddFunc)(int, int);

void call_dynamic_function() {
    // 假设已经通过 dlopen 打开了 libplugin.so
    void* handle = dlopen("/data/app/.../libplugin.so", RTLD_NOW);
    if (!handle) {
        LOGE("dlopen failed: %s", dlerror());
        return;
    }

    // 2. 关键：先清空旧的 dlerror() 状态
    dlerror(); 
    
    // 3. 查找符号
    AddFunc add_func = (AddFunc)dlsym(handle, "add");
    
    // 4. 检查错误（必须和 dlerror 配合）
    const char* error = dlerror();
    if (error != NULL) {
        LOGE("dlsym failed: %s", error);
        dlclose(handle);
        return;
    }

    // 5. 安全调用
    if (add_func) {
        int result = add_func(1, 2);
        LOGI("Result: %d", result);
    }
}
```

---

### 3. Android 开发中的三大致命陷阱（必看）

#### 陷阱一：C++ 名称修饰（Name Mangling）
这是导致 `dlsym` 找不到函数的第一大原因。
在 C++ 中，函数重载导致编译器会将函数名改编成类似 `_Z3addii` 的复杂符号。你在代码里写 `"add"`，但 so 里的实际符号是 `_Z3addii`，必然找不到。

**解决方案**：在需要被 `dlsym` 查找的函数前，加上 `extern "C"`，强制使用 C 风格的符号命名（即原名）。
```cpp
// plugin.cpp
extern "C" {
    int add(int a, int b) { return a + b; }
}
// 或者只修饰单个函数
extern "C" int add(int a, int b) { return a + b; }
```

#### 陷阱二：符号默认被隐藏（Visibility）
如果你的 CMake 使用了 `-fvisibility=hidden`（为了减小编译体积和增加逆向难度），编译器会将所有函数视为“私有”，`dlsym` 找不到它们。

**解决方案**：在函数声明前加上可见性属性。
```cpp
// 方法1：在代码中强制导出
__attribute__((visibility("default"))) int add(int a, int b) { return a + b; }

// 方法2：配合 extern "C"
extern "C" __attribute__((visibility("default"))) int add(int a, int b) { return a + b; }
```

#### 陷阱三：找不到时立即报错
`dlsym` 失败后，如果没有及时调用 `dlerror()`，错误信息可能会被后续其他库的调用覆盖，导致你捕获不到真正的错误原因。

**黄金原则**：在调用 `dlsym` 前先调用 `dlerror()` 清空状态；调用后立即检查 `dlerror()` 的返回值。必须使用**上面的标准模板**。

---

### 4. 进阶技巧：查看看 so 里真正的符号名

如果你不确定 so 里的符号名是什么，可以用 NDK 自带的 `readelf` 工具查看：

```bash
# 在 Android 设备或 Linux 终端执行
readelf -s libplugin.so | grep add
```
输出中 `FUNC` 或 `OBJECT` 那一列的名字，就是 `dlsym` 需要传入的准确字符串。如果是 C++ 编译且没有加 `extern "C"`，你会看到一堆 `_Z...` 的乱码。

---

### 5. `dlsym` 与直接链接调用的对比

结合你上轮问的“so间调用”，两种方式的选择如下：

| 调用方式 | 特点 | `dlsym` 适用场景 |
| :--- | :--- | :--- |
| **编译时链接** (`target_link_libraries`) | 编译期确定，直接函数调用，速度快，启动时加载。 | **不适用**。这种情况你直接在代码里 `#include` 头文件并调用即可，不需要 `dlsym`。 |
| **运行时动态加载** (`dlopen` + `dlsym`) | 运行期查找，函数指针调用，按需加载，灵活性高。 | 插件化架构、热修复、条件加载（如仅调试模式下加载测试 so）、解耦核心业务与附加功能。 |

### 总结一句话
**`dlsym` 是“通过名字找地址”的运行时工具。使用它时，务必用 `extern "C"` 防止 C++ 改名，用 `__attribute__((visibility("default")))` 防止符号被隐藏，并严格配合 `dlerror()` 做错误处理。** 

如果你在执行 `dlsym` 时遇到了 `undefined symbol` 的具体错误，可以把 `readelf -s` 的结果发给我，我帮你分析是名称修饰还是可见性问题。😊


# jni创建java对象
在 JNI（Java Native Interface）中，从 Native（C/C++）层创建 Java 对象，核心方法是使用 `JNIEnv` 提供的 **`NewObject`** 系列函数。

这个过程本质上模拟了 Java 的 `new` 关键字：**先找到类，再找到构造函数，最后调用构造函数分配内存并初始化对象**。

以下是标准的三步走流程，以及 Android 开发中必须注意的深坑。

---

### 1. 标准创建流程（三步法）

假设我们要在 Native 层创建一个 Java 的 `Person` 对象：
```java
// Java 层定义
package com.example.app;

public class Person {
    private String name;
    private int age;

    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }
}
```

**Native 层创建代码（C++ 示例）：**

```cpp
JNIEXPORT jobject JNICALL
Java_com_example_app_MainActivity_createPerson(JNIEnv* env, jobject thiz) {

    // 1. 找到 Java 类 (Class)
    // 注意：如果此函数在子线程执行，FindClass 可能返回 null，详见下方"致命陷阱"
    jclass personClass = env->FindClass("com/example/app/Person");
    if (personClass == nullptr) {
        return nullptr; // 表示发生异常
    }

    // 2. 获取构造函数的方法 ID (MethodID)
    // "<init>" 代表构造函数，参数签名："(Ljava/lang/String;I)V" 表示参数为 String 和 int，返回 void
    jmethodID constructorId = env->GetMethodID(personClass, "<init>", "(Ljava/lang/String;I)V");
    if (constructorId == nullptr) {
        return nullptr;
    }

    // 3. 准备构造函数参数并创建对象
    jstring nameStr = env->NewStringUTF("李小明");
    jint age = 18;

    // 调用 NewObject，传入类、构造方法 ID 和参数
    jobject personObj = env->NewObject(personClass, constructorId, nameStr, age);

    // 注意：nameStr 是局部引用，函数返回后自动释放。如果 personObj 要存为全局变量，需 NewGlobalRef
    return personObj;
}
```

---

### 2. 三个关键 API 详解

| API 函数 | 作用 | 关键细节 |
| :--- | :--- | :--- |
| **`FindClass`** | 加载类，返回 `jclass`。 | **类名必须用 `/` 代替 `.`**（如 `"com/example/App"`）。若找不到类，会抛 `ClassNotFoundException`（Native 层需检查是否 `null`）。 |
| **`GetMethodID`** | 获取构造函数或普通方法 ID。 | 构造函数方法名固定为 **`"<init>"`**。**签名（Signature）** 写错会返回 `null`，建议用 `javap -s` 获取。 |
| **`NewObject`** | 分配内存并执行构造函数。 | 变体有 `NewObjectV`（接收 `va_list`）和 `NewObjectA`（接收 `jvalue*` 数组）。 |

---

### 3. 进阶：使用 `AllocObject`（延后初始化）

有时你想先分配对象内存，稍后再调用构造函数（极少使用，除非是做对象池或序列化反序列化）。
```cpp
jobject obj = env->AllocObject(clazz); // 仅分配内存，不调用构造函数
// ... 做一些中间操作 ...
env->CallNonvirtualVoidMethod(obj, clazz, constructorId, args...); // 手动调用构造函数
```
> **警告**：如果在 `AllocObject` 后没有正确调用构造函数，对象将处于未初始化状态，调用其方法会引发崩溃。**通常情况下直接用 `NewObject` 即可**。

---

### 4. Android 开发中的四大致命陷阱（必看）

#### 陷阱一：`FindClass` 在 Native 子线程中返回 `null`
- **现象**：在 `pthread` 或 `std::thread` 中通过 `AttachCurrentThread` 获取 `JNIEnv` 后，调用 `FindClass` 找不到自定义的 Java 类，返回 `null`。
- **原因**：Native 子线程没有 Java 的类加载器（ClassLoader）上下文，无法感知应用的 Dex 路径。
- **解决方案（最佳实践）**：**在 `JNI_OnLoad`（主线程）中提前查找类，并缓存为全局引用。**
  ```cpp
  // 在 JNI_OnLoad 中
  jclass localClass = env->FindClass("com/example/app/Person");
  g_PersonClass = (jclass)env->NewGlobalRef(localClass); // 转为全局引用
  env->DeleteLocalRef(localClass);
  
  // 在子线程中直接用 g_PersonClass，不需要 FindClass
  jobject obj = env->NewObject(g_PersonClass, g_constructorId, ...);
  ```

#### 陷阱二：构造函数的签名写错
- **表现**：`GetMethodID` 返回 `null`，且 `env->ExceptionCheck()` 为真。
- **排查**：编译 Java 代码后，在终端执行 `javap -s -p 你的类全名.class`，复制 `descriptor` 内容填入代码，避免手写拼错。

#### 陷阱三：循环中大量创建对象导致局部引用溢出
- **表现**：在 for 循环中频繁调用 `NewObject`，不删除局部引用，运行一段时间后崩溃，报错 `ReferenceTable overflow`。
- **解决**：在循环体末尾手动调用 `env->DeleteLocalRef(obj)`，释放临时对象占用的局部引用槽位。

#### 陷阱四：构造函数抛出了 Java 异常
- **现象**：`NewObject` 返回 `null`，但程序没有立即崩溃。
- **解决**：`NewObject` 之后，必须调用 `env->ExceptionCheck()` 检查。如果有异常，JNI 层需要清除或返回 Java 层处理，否则后续 JNI 调用会直接崩溃。
  ```cpp
  jobject obj = env->NewObject(clazz, ctor, args);
  if (env->ExceptionCheck()) {
      env->ExceptionDescribe(); // 打印堆栈
      env->ExceptionClear();    // 清除异常，防止崩溃
      return nullptr;
  }
  ```

---

### 5. 全局引用与返回对象

- **默认返回给 Java**：如果你直接在 JNI 函数里 `return personObj;`，它属于**局部引用**，但在返回给 Java 时会被 JVM 自动升级为合适的引用，交给 Java 垃圾回收管理。**不需要**手动转为全局引用。
- **跨线程或长期缓存**：如果你想在 Native 子线程中持续持有这个对象（比如保存在一个全局 C++ 变量中），则必须使用 `env->NewGlobalRef(personObj)`，否则下次 Native 函数返回后，局部引用失效，全局变量变成野指针。

### 总结一句话
**`FindClass` 找类型，`GetMethodID` 拿构造器（`<init>`），`NewObject` 传参造对象。** 记住：自定义类的 `FindClass` 尽量在 `JNI_OnLoad` 中缓存为全局引用，以避开子线程类加载器的坑；创建完成后记得检查异常，这是避免诡异崩溃的核心防线。




# JNI访问对象属性
通过 JNI 在 Native 层访问 Java 层的属性（字段），核心在于两步：**先拿到字段的 ID（`jfieldID`），再通过专门的 Get/Set 函数读写它的值**。

这个过程与调用 Java 方法非常相似，但 API 更简单。JNI 支持访问**实例字段**（属于对象）和**静态字段**（属于类）。

---

### 1. 核心 API 概览

| 操作类型      | 实例字段 (Instance)                  | 静态字段 (Static)                                |
| :-------- | :------------------------------- | :------------------------------------------- |
| **获取 ID** | `GetFieldID`                     | `GetStaticFieldID`                           |
| **读值**    | `GetIntField`、`GetObjectField` 等 | `GetStaticIntField`、`GetStaticObjectField` 等 |
| **写值**    | `SetIntField`、`SetObjectField` 等 | `SetStaticIntField`、`SetStaticObjectField` 等 |

**函数签名**（以 `int` 为例）：
```cpp
// 实例字段
jfieldID fieldId = env->GetFieldID(jclass clazz, const char* name, const char* sig);
jint value = env->GetIntField(jobject obj, jfieldID fieldId);
env->SetIntField(jobject obj, jfieldID fieldId, jint value);

// 静态字段
jfieldID staticFieldId = env->GetStaticFieldID(jclass clazz, const char* name, const char* sig);
jint value = env->GetStaticIntField(jclass clazz, jfieldID staticFieldId);
env->SetStaticIntField(jclass clazz, jfieldID staticFieldId, jint value);
```

---

### 2. 字段签名（类型描述符）

与调用方法不同，字段签名**没有括号和返回值结构**，就是单个类型的编码。

| Java 类型 | 字段签名 |
| :--- | :--- |
| `int` | `I` |
| `boolean` | `Z` |
| `String` | `Ljava/lang/String;` |
| `int[]` | `[I` |
| `MyClass` | `Lcom/example/MyClass;` |

---

### 3. 实战：访问私有实例字段（标准流程）

**假设 Java 类：**
```java
package com.example.app;
public class User {
    private String name;  // 私有实例字段
    public int age;       // 公有实例字段
}
```

**Native 层读写字段（C++）：**

```cpp
// 1. 在 JNI_OnLoad 中缓存字段 ID（最佳实践，避免反复查找）
jfieldID g_name_field_id = nullptr;
jfieldID g_age_field_id = nullptr;

JNIEXPORT jint JNICALL JNI_OnLoad(JavaVM* vm, void* reserved) {
    JNIEnv* env = ...;
    jclass clazz = env->FindClass("com/example/app/User");
    if (clazz == nullptr) return JNI_ERR;

    // 获取字段 ID（即使 private 也能获取！）
    g_name_field_id = env->GetFieldID(clazz, "name", "Ljava/lang/String;");
    g_age_field_id = env->GetFieldID(clazz, "age", "I");

    // 注意：clazz 是局部引用，如果要跨函数使用，必须转为全局引用
    // g_user_class = (jclass)env->NewGlobalRef(clazz);
    env->DeleteLocalRef(clazz);
    return JNI_VERSION_1_6;
}
```

```cpp
// 2. 在 JNI 函数中读写对象属性
JNIEXPORT void JNICALL
Java_com_example_app_MainActivity_modifyUser(JNIEnv* env, jobject thiz, jobject userObj) {

    // ---- 读取 String 字段 ----
    jstring nameStr = (jstring)env->GetObjectField(userObj, g_name_field_id);
    const char* nameCstr = env->GetStringUTFChars(nameStr, nullptr);
    __android_log_print(ANDROID_LOG_DEBUG, "JNI", "Name: %s", nameCstr);
    env->ReleaseStringUTFChars(nameStr, nameCstr); // 释放内存

    // ---- 写入 int 字段 ----
    env->SetIntField(userObj, g_age_field_id, 30);
}
```

> **注意**：`GetObjectField`（以及 `SetObjectField`）返回的 `jstring` 是一个**局部引用**。它会在当前 Native 函数返回后自动释放。如果你想把它存到全局变量，需要用 `NewGlobalRef`。

---

### 4. 访问静态字段（示例）

如果 Java 中有 `public static String TAG = "User";`

**获取 ID 与读写：**
```cpp
// 获取 ID（在 JNI_OnLoad 中）
jfieldID g_tag_field_id = env->GetStaticFieldID(clazz, "TAG", "Ljava/lang/String;");

// 读取
jstring tagStr = (jstring)env->GetStaticObjectField(clazz, g_tag_field_id);

// 写入
jstring newTag = env->NewStringUTF("NewTag");
env->SetStaticObjectField(clazz, g_tag_field_id, newTag);
```

---

### 5. Android 开发中的致命陷阱（必须警惕）

#### 陷阱一：`GetFieldID` 在子线程中返回 `null`
- **现象**：在 `pthread` 子线程中，`FindClass` 返回 `null`，导致 `GetFieldID` 崩溃。
- **原因**：Native 子线程没有正确的 ClassLoader 上下文。
- **解决方案**：**在 `JNI_OnLoad`（主线程）中提前获取所有字段 ID 并缓存为全局变量**。上面的示例正是这样做的。这保证了无论哪个线程调用，都直接使用已缓存的 ID，无需执行 `FindClass`。

#### 陷阱二：签名错误导致 `NoSuchFieldError`
- **表现**：`GetFieldID` 返回 `null`，且抛异常。
- **排查**：编译 Java 后，在 Terminal 执行 `javap -s -p 你的类名.class`，找到字段对应的 `descriptor`（如 `Ljava/lang/String;`），**特别注意引用类型末尾的分号 `;`**。

#### 陷阱三：`GetObjectField` 的局部引用泛滥（循环泄漏）
- 如果在循环中频繁使用 `GetObjectField` 获取 `String` 或数组，并且不处理，会撑爆局部引用表。
- **解决**：如果获取的是临时对象，操作完立刻 `env->DeleteLocalRef(localRef)`。

#### 陷阱四：访问私有字段与混淆（ProGuard/R8）
- JNI **可以绕过 `private` 修饰符**访问字段，编译器不阻止。
- **致命问题**：如果开启代码混淆（Release 包），字段名 `name` 可能被混淆成 `a` 或 `b`。
- **解决方案**：在 Java 层的字段上加上 **`@Keep`** 注解，或在 `proguard-rules.pro` 中添加 `-keepclassmembers class com.example.app.User { <fields>; }`，确保混淆后字段名不变。

#### 陷阱五：对象引用与垃圾回收（GC）
- 当你通过 `SetObjectField` 设置一个新对象（如 `String`）时，JVM 会将该字段指向新对象，并将旧对象的引用计数减一并标记为可回收。这与 Java 层的赋值行为完全一致，JNI 层不需要主动去 `Delete` 旧对象。
- 但如果你从 `GetObjectField` 取出了一个对象，想让它长时间存活，必须用 `NewGlobalRef` 提升为全局引用，否则下一次 JNI 调用返回时，这个局部引用就会失效。

---

### 总结一句话
1. **取 ID**：通过 `GetFieldID`（实例）或 `GetStaticFieldID`（静态）获取字段 ID，务必在 `JNI_OnLoad` 中缓存。
2. **读/写**：使用 `Get<Type>Field` / `Set<Type>Field` 或对应的静态版本。
3. **签名**：用 `javap -s` 严格比对，避免手写错误。
4. **混淆**：Release 包务必 `@Keep` 字段，否则 Native 层会找不到字段。


# JNI访问数组
在 JNI 中访问 Java 数组，跟访问普通对象字段的逻辑完全不同。JNI 专门为**基本类型数组（如 `int[]`）**和**对象数组（如 `String[]`）**提供了两套截然不同的 API，它们的操作效率和内存管理方式天差地别。

---

### 1. 基本类型数组（`int[]`、`byte[]`、`float[]` 等）

这是性能最敏感的区域。JNI 提供了**两种**方式来操作它们，你需要根据数据量大小来选择。

#### 方法一：`Get<Type>ArrayElements` / `Release<Type>ArrayElements`（高性能，零拷贝或指针访问）
这是最常用的方法，返回指向**实际数组内存**的指针（可能是直接指针，也可能是托管内存的拷贝）。**必须配套 `Release` 释放**。

```cpp
// C++ 示例：修改 int[] 数组，将所有元素乘以 2
JNIEXPORT void JNICALL
Java_com_example_NativeLib_doubleArray(JNIEnv* env, jobject thiz, jintArray arr) {
    // 1. 获取数组长度
    jsize len = env->GetArrayLength(arr);

    // 2. 获取指向数组元素的内存指针（jint*）
    // 参数说明：arr 数组对象，JNI_FALSE 表示不拷贝（优先返回直接指针）
    jint* elements = env->GetIntArrayElements(arr, nullptr);
    if (elements == nullptr) {
        return; // 可能 OutOfMemoryError
    }

    // 3. 读写数据（直接在原始内存上操作）
    for (int i = 0; i < len; i++) {
        elements[i] = elements[i] * 2;
    }

    // 4. 必须释放！提交修改并释放指针
    // 参数说明：arr 数组对象，elements 指针，mode 决定同步方式
    // - 0: 将修改同步回 Java 数组，并释放内存
    // - JNI_COMMIT: 同步回 Java 但不释放（极少用）
    // - JNI_ABORT: 放弃修改，直接释放（常用于只读场景）
    env->ReleaseIntArrayElements(arr, elements, 0);
}
```

**对应的 `Get` / `Release` 函数列表**：
| 数组类型 | 获取函数 | 释放函数 |
| :--- | :--- | :--- |
| `boolean[]` | `GetBooleanArrayElements` | `ReleaseBooleanArrayElements` |
| `byte[]` | `GetByteArrayElements` | `ReleaseByteArrayElements` |
| `int[]` | `GetIntArrayElements` | `ReleaseIntArrayElements` |
| `float[]` | `GetFloatArrayElements` | `ReleaseFloatArrayElements` |
| ... | ... | ... |

#### 方法二：`Get<Type>ArrayRegion` / `Set<Type>ArrayRegion`（安全拷贝，适合小数组）
这种方法**不直接操作原始内存**，而是将数据拷贝到 Native 层的缓存数组（C 数组）中，适合处理小数组（如小于 1KB），内存管理更安全简单，无需担心临界区死锁。

```cpp
JNIEXPORT jint JNICALL
Java_com_example_NativeLib_sumArray(JNIEnv* env, jobject thiz, jintArray arr) {
    jsize len = env->GetArrayLength(arr);
    // 1. 在栈上或堆上分配一个临时 C 数组
    int* buffer = new int[len]; // 注意：如果数组太大，栈会溢出，用堆

    // 2. 拷贝 Java 数组内容到 C 缓冲区
    env->GetIntArrayRegion(arr, 0, len, buffer);

    // 3. 计算求和
    int sum = 0;
    for (int i = 0; i < len; i++) { sum += buffer[i]; }

    delete[] buffer; // 释放 C 内存
    return sum;
}
```

---

### 2. 对象数组（`String[]`、`MyClass[]` 等）

对于对象数组，JNI **没有提供**批量获取内存指针的 API。因为数组里存的是**对象引用**（指针），你需要逐个元素操作。

**核心方法**：`GetObjectArrayElement` 和 `SetObjectArrayElement`。

```cpp
// 示例：拼接 String[] 数组中的所有字符串
JNIEXPORT jstring JNICALL
Java_com_example_NativeLib_concatStrings(JNIEnv* env, jobject thiz, jobjectArray stringArray) {
    jsize len = env->GetArrayLength(stringArray);
    std::string result;

    for (int i = 0; i < len; i++) {
        // 取出第 i 个元素（返回的是 jobject，需强转为 jstring）
        jstring str = (jstring)env->GetObjectArrayElement(stringArray, i);
        if (str != nullptr) {
            const char* utf = env->GetStringUTFChars(str, nullptr);
            result += utf;
            env->ReleaseStringUTFChars(str, utf);
        }
        // 注意：str 是一个局部引用，循环结束后自动删除。
        // 如果循环次数极大（10万次），需手动 DeleteLocalRef(str) 防止溢出
        env->DeleteLocalRef(str); // 建议显式删除
    }

    return env->NewStringUTF(result.c_str());
}
```

---

### 3. 创建新数组并返回

如果你需要返回一个新数组给 Java，使用 `New<Type>Array`（基本类型）或 `NewObjectArray`（对象类型）。

```cpp
// 创建 int[10] 并返回
JNIEXPORT jintArray JNICALL
Java_com_example_NativeLib_createArray(JNIEnv* env, jobject thiz) {
    jsize len = 10;
    // 1. 分配数组内存（不初始化）
    jintArray arr = env->NewIntArray(len);
    if (arr == nullptr) return nullptr; // 分配失败

    // 2. 填充数据（填充整个数组）
    int fill_data[10] = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
    env->SetIntArrayRegion(arr, 0, len, fill_data);

    return arr;
}

// 创建 String[5] 并返回
JNIEXPORT jobjectArray JNICALL
Java_com_example_NativeLib_createStringArray(JNIEnv* env, jobject thiz) {
    jclass stringClass = env->FindClass("java/lang/String");
    jobjectArray arr = env->NewObjectArray(5, stringClass, nullptr);
    for (int i = 0; i < 5; i++) {
        jstring str = env->NewStringUTF("Hello");
        env->SetObjectArrayElement(arr, i, str);
        env->DeleteLocalRef(str);
    }
    env->DeleteLocalRef(stringClass); // 注意局部引用清理
    return arr;
}
```

---

### 4. Android 开发中的四大致命陷阱（必看）

#### 陷阱一：`Get<Type>ArrayElements` 后忘了 `Release`
- **后果**：内存泄漏。更严重的是，如果你修改了 `elements` 但没有 `Release`，Java 原始数组**不会被更新**，导致数据错乱。
- **铁律**：`Get...ArrayElements` 和 `Release...ArrayElements` **必须成对出现**，且放在同一个函数内，不要跨函数传递。

#### 陷阱二：`GetPrimitiveArrayCritical` 的滥用（性能陷阱）
JNI 提供了一个极速函数 `GetPrimitiveArrayCritical`，它**直接锁定** JVM 的 GC，返回绝对零拷贝的原始指针。但它的使用极其严苛：
- **禁止** 在 `Critical` 区间调用任何 JNI 函数（不能 `FindClass`、不能 `NewObject`、甚至不能 `ExceptionCheck`）。
- **禁止** 执行任何可能阻塞或睡眠的操作（I/O、锁、`sleep`）。
- **原因**：它会暂停 JVM 的垃圾回收器，如果此时触发任何 JVM 操作，会直接死锁导致进程 ANR 或崩溃。
- **建议**：**新手或普通场景直接用普通的 `Get...ArrayElements` 即可**，其性能差异在 Android 上通常可忽略。

#### 陷阱三：对象数组中的元素引用管理
- `GetObjectArrayElement` 返回的是**局部引用**。如果在循环中获取大量元素且不删除，会撑爆局部引用表（默认 512 个槽位）。
- **解决**：在循环体末尾显式调用 `env->DeleteLocalRef(str)`，如上例所示。

#### 陷阱四：数组的签名（用于动态注册）
当你在 `JNINativeMethod` 表中填写签名时：
- `int[]` → `"[I"`
- `String[]` → `"[Ljava/lang/String;"`
- `byte[][]`（二维数组）→ `"[[B"`
**注意**：对象数组末尾也有分号 `;`，且内部类要用 `$`。

---

### 5. 总结：何时用哪种方法？

| 场景 | 推荐方法 | 理由 |
| :--- | :--- | :--- |
| **大数组（>1KB）且频繁修改** | `GetIntArrayElements` + `Release` | 避免多次拷贝，直接操作内存，性能最优。 |
| **小数组（<1KB）或一次性读取** | `GetIntArrayRegion` / `SetIntArrayRegion` | 代码更安全，无内存泄漏风险，无需配对释放。 |
| **只需要读取，不修改原数组** | `GetIntArrayElements` + `Release` 时传 `JNI_ABORT` | 放弃同步回 Java，减少写回操作的开销。 |
| **对象数组（任何大小）** | `GetObjectArrayElement` / `SetObjectArrayElement` | 唯一的标准方式，记得及时 `DeleteLocalRef`。 |
# JNI访问方法
在 JNI 中访问 Java 层的**函数（方法）**，与访问属性（字段）的逻辑完全对称，但核心 API 是 `Call<Type>Method` 系列。你需要先拿到方法的 ID（`jmethodID`），然后通过 `JNIEnv` 的调用函数去执行它。

JNI 支持调用**实例方法**（属于对象）和**静态方法**（属于类），也支持调用**父类方法**（非虚调用）。

---

### 1. 核心 API 概览

| 操作类型 | 获取 ID | 调用方法 |
| :--- | :--- | :--- |
| **实例方法** | `GetMethodID` | `CallVoidMethod`、`CallIntMethod`、`CallObjectMethod` 等 |
| **静态方法** | `GetStaticMethodID` | `CallStaticVoidMethod`、`CallStaticIntMethod`、`CallStaticObjectMethod` 等 |
| **调用父类方法** | `GetMethodID` | `CallNonvirtualVoidMethod` 等（极少用） |

**函数签名示例（以返回 `int` 为例）：**
```cpp
// 实例方法
jmethodID mid = env->GetMethodID(clazz, "methodName", "(II)I");
jint result = env->CallIntMethod(obj, mid, arg1, arg2);

// 静态方法
jmethodID staticMid = env->GetStaticMethodID(clazz, "staticMethod", "(Ljava/lang/String;)V");
env->CallStaticVoidMethod(clazz, staticMid, jstr);
```

---

### 2. 方法签名（重中之重）

方法签名由 **`(参数类型编码)返回值类型编码`** 组成。这是 JNI 中最容易写错的地方。

| Java 方法 | 签名（Signature） |
| :--- | :--- |
| `void log(String msg)` | `(Ljava/lang/String;)V` |
| `int add(int a, int b)` | `(II)I` |
| `String concat(String a, String b)` | `(Ljava/lang/String;Ljava/lang/String;)Ljava/lang/String;` |
| `void process(byte[] data)` | `([B)V` |
| `int[] getList()` | `()[I` |

> **铁律**：`GetMethodID` 返回 `null` 且抛 `NoSuchMethodError`，99% 是签名写错了。**务必**编译 Java 代码后，用 `javap -s -p 类名.class` 精确获取签名，不要手写。

---

### 3. 实战：在 JNI_OnLoad 中缓存方法 ID（标准写法）

**Java 层代码：**
```java
package com.example.app;

public class Calculator {
    // 实例方法
    public int add(int a, int b) { return a + b; }

    // 静态方法
    public static void logResult(String msg) {
        Log.d("JNI", msg);
    }
}
```

**Native 层（C++）：**
```cpp
// 全局缓存变量
jclass g_calc_class = nullptr;
jmethodID g_add_mid = nullptr;
jmethodID g_log_mid = nullptr;

JNIEXPORT jint JNICALL JNI_OnLoad(JavaVM* vm, void* reserved) {
    JNIEnv* env = ...;
    // 1. 查找类（必须转为全局引用）
    jclass localClazz = env->FindClass("com/example/app/Calculator");
    g_calc_class = (jclass)env->NewGlobalRef(localClazz);
    env->DeleteLocalRef(localClazz);

    // 2. 获取实例方法 ID
    // 参数：类，方法名，签名
    g_add_mid = env->GetMethodID(g_calc_class, "add", "(II)I");

    // 3. 获取静态方法 ID
    g_log_mid = env->GetStaticMethodID(g_calc_class, "logResult", "(Ljava/lang/String;)V");

    return JNI_VERSION_1_6;
}
```

**在 JNI 函数中调用：**
```cpp
JNIEXPORT void JNICALL
Java_com_example_app_MainActivity_testJNI(JNIEnv* env, jobject thiz) {

    // ---- 调用实例方法 ----
    // 假设 Java 层传入了一个 Calculator 对象（通过参数或创建）
    // 此处演示：创建一个新的 Calculator 对象并调用
    jobject calcObj = env->NewObject(g_calc_class, env->GetMethodID(g_calc_class, "<init>", "()V"));
    int result = env->CallIntMethod(calcObj, g_add_mid, 10, 20);
    __android_log_print(ANDROID_LOG_DEBUG, "JNI", "10 + 20 = %d", result);

    // ---- 调用静态方法 ----
    jstring msg = env->NewStringUTF("Calculation done!");
    env->CallStaticVoidMethod(g_calc_class, g_log_mid, msg);
    env->DeleteLocalRef(msg);
    env->DeleteLocalRef(calcObj);
}
```

---

### 4. 类型映射与 Call 函数速查

调用方法时传入的参数必须与签名严格匹配。`Call<Type>Method` 的返回类型对应表：

| Java 返回类型 | JNI 函数 | 参数传入 |
| :--- | :--- | :--- |
| `void` | `CallVoidMethod` | 无返回值，直接调用 |
| `int` | `CallIntMethod` | 传入 `jint` |
| `boolean` | `CallBooleanMethod` | 传入 `jboolean`（C++ 中是 `unsigned char`） |
| `float` | `CallFloatMethod` | 传入 `jfloat` |
| `String` / `Object` | `CallObjectMethod` | 返回 `jobject`（需强转） |
| `long` | `CallLongMethod` | 传入 `jlong`（注意 C++ 字面量加 `LL`，如 `100LL`） |

---

### 5. Android 开发中的五大致命陷阱

#### 陷阱一：在 Native 子线程中 `FindClass` 返回 `null`
- 与访问字段同理，Native 子线程（`pthread`）缺少 ClassLoader 上下文，`FindClass` 找不到自定义类。
- **解决**：**必须在 `JNI_OnLoad`（主线程）中缓存 `jclass` 为全局引用，并在该线程中直接使用缓存的类和方法 ID。**

#### 陷阱二：调用 Java 方法抛出异常，Native 层未处理
- 如果 Java 方法内部抛出了异常（如空指针），Native 层的 `Call...Method` 会立即返回（返回默认值，如 `0` 或 `null`），**但异常状态会挂起**。
- 如果不检查并清除异常，后续任何 JNI 调用都会导致应用崩溃。
- **必须的代码模板**：
```cpp
env->CallVoidMethod(obj, mid, args...);
if (env->ExceptionCheck()) {
    env->ExceptionDescribe(); // 打印堆栈（调试用）
    env->ExceptionClear();    // 清除异常，防止崩溃
    return;                   // 或做其他补救处理
}
```

#### 陷阱三：`CallObjectMethod` 返回的引用管理
- `CallObjectMethod` 返回的对象（如 `String`）是一个**局部引用**。
- 如果返回值需要保存到全局变量或跨函数使用，必须立即用 `env->NewGlobalRef(localRef)` 转为全局引用，否则函数返回后会被回收。

#### 陷阱四：调用父类方法（`CallNonvirtual`）的误用
- `CallNonvirtual<Type>Method` 用于绕过多态，直接调用父类的方法（相当于 Java 的 `super.method()`）。
- **注意**：传入的对象必须是子类实例，但调用的是父类的方法 ID。绝大多数业务场景用不到，新手容易在错误场景下使用导致 `IllegalAccessError`。**正常情况下直接用 `Call...Method` 即可**。

#### 陷阱五：性能问题——频繁跨 JNI 调用
- 每次 `Call...Method` 都有栈切换和类型检查开销。
- **优化建议**：如果在 C++ 循环中需要调用 Java 方法（如 `list.get(i)`），尽量将循环逻辑放到 Native 层处理完后一次性回调 Java，或者使用 `GetArrayElements` 批量处理数据，而不是在循环中反复调用 JNI 函数。

---

### 6. 构造函数（也是方法）

构造函数本质上是一个名为 `<init>` 的实例方法，返回类型是 `void`（签名以 `V` 结尾）。

```cpp
// 获取无参构造函数 ID
jmethodID ctor = env->GetMethodID(clazz, "<init>", "()V");
jobject obj = env->NewObject(clazz, ctor);

// 获取有参构造函数 (参数为 String 和 int)
jmethodID ctor2 = env->GetMethodID(clazz, "<init>", "(Ljava/lang/String;I)V");
jstring name = env->NewStringUTF("Tom");
jobject obj2 = env->NewObject(clazz, ctor2, name, 18);
```

---

### 总结一句话
1. **取 ID**：在 `JNI_OnLoad` 中通过 `GetMethodID`/`GetStaticMethodID` 缓存方法 ID，避免运行时反复查找。
2. **调用**：使用对应的 `Call<Type>Method`/`CallStatic<Type>Method`，参数按顺序传入。
3. **签名**：绝对依赖 `javap -s` 获取，禁止手写（特别是对象和数组的分号）。
4. **安全**：调用后立即用 `ExceptionCheck` 检查异常，否则崩溃风险极高。
5. **性能**：避免在 Native 循环中频繁回调 Java 方法，尽量批量传输数据。


# JNI访问父类

在 JNI 中，“访问父类对象”通常包含三个层面的需求：**获取父类的 Class**、**访问父类定义的字段**、以及**调用父类的方法（而非子类重写的方法）**。

JNI 对继承的处理遵循 Java 的规则，但有一处极易混淆的“语法糖”：**默认的方法调用是虚的（多态），而字段访问是非虚的（看声明类型）**。

以下是针对这三个场景的详细解析和实战代码。

---

### 1. 获取父类的 Class 对象：`GetSuperclass`

如果你手里有一个子类的 `jclass`，想获取它的父类，可以使用 `GetSuperclass`。

```cpp
jclass childClass = env->FindClass("com/example/Dog");
jclass parentClass = env->GetSuperclass(childClass); 
// parentClass 指向 com/example/Animal
// 注意：parentClass 是局部引用，若需跨函数使用，务必转为全局引用
```

---

### 2. 访问父类定义的字段（“字段隐藏”陷阱）

**Java 规则回顾**：Java 的字段（属性）不具备多态性。如果子类定义了与父类同名的字段，则子类字段会“隐藏”父类字段，访问时取决于引用的类型。

**JNI 实战规则**：`GetFieldID` 和 `GetObjectField` 操作的是**对象内存中实际存储的数据**。如果你使用子类的 `jclass` 获取字段 ID，JVM 会返回该子类声明的字段；如果你使用父类的 `jclass` 获取 ID，则返回父类声明的字段。

**示例场景：**
```java
class Animal { public String name = "Animal"; }
class Dog extends Animal { public String name = "Dog"; }
// Dog 对象在内存中同时持有 Animal.name 和 Dog.name 两个值
```

**Native 层访问代码：**
```cpp
void accessFields(JNIEnv* env, jobject dogObj) {
    // 1. 获取子类 Dog 的 Class
    jclass dogClass = env->GetObjectClass(dogObj);
    // 2. 获取父类 Animal 的 Class（通过 GetSuperclass）
    jclass animalClass = env->GetSuperclass(dogClass);

    // ---- 读取子类字段（Dog.name） ----
    jfieldID dogNameId = env->GetFieldID(dogClass, "name", "Ljava/lang/String;");
    jstring dogName = (jstring)env->GetObjectField(dogObj, dogNameId); // 结果是 "Dog"

    // ---- 读取父类字段（Animal.name） ----
    // 关键：必须使用父类的 jclass 来获取 FieldID！
    jfieldID animalNameId = env->GetFieldID(animalClass, "name", "Ljava/lang/String;");
    jstring animalName = (jstring)env->GetObjectField(dogObj, animalNameId); // 结果是 "Animal"

    // 结论：即使对象是 Dog，只要用父类的 FieldID 去读，拿到的就是父类的字段值
}
```

---

### 3. 调用父类的方法（“非虚调用”核心）

这是 JNI 中与 Java 语法差异最大的地方。

- **默认行为 (`Call<Type>Method`)**：**遵循多态**。如果子类重写了方法，调用的一定是子类的实现。
- **调用父类实现 (`CallNonvirtual<Type>Method`)**：**绕过多态**，直接调用指定父类中的方法实现（相当于 Java 中的 `super.method()`）。

**示例场景：**
```java
class Animal { public void speak() { Log.d("JNI", "Animal sound"); } }
class Dog extends Animal { @Override public void speak() { Log.d("JNI", "Woof!"); } }
```

**Native 层调用代码：**
```cpp
void callMethods(JNIEnv* env, jobject dogObj) {
    // 1. 获取 Dog 和 Animal 的 Class
    jclass dogClass = env->GetObjectClass(dogObj);
    jclass animalClass = env->GetSuperclass(dogClass);

    // 2. 获取方法 ID（注意：只需获取一次，且两者方法名和签名相同）
    jmethodID speakMid = env->GetMethodID(animalClass, "speak", "()V"); 
    // 这里用 animalClass 或 dogClass 获取 ID 都行，因为方法名相同

    // 3. 默认调用（多态） -> 输出 "Woof!"
    env->CallVoidMethod(dogObj, speakMid); 

    // 4. 非虚调用（调用父类实现） -> 输出 "Animal sound"
    // 关键：传入父类的 Class 对象，告诉 JVM 从父类开始找实现
    env->CallNonvirtualVoidMethod(dogObj, animalClass, speakMid);
}
```

> **语法说明**：`CallNonvirtual<Type>Method` 的参数比普通调用多一个 `jclass`，这个 `clazz` 参数指定了**从哪个类开始解析方法**（通常是父类）。必须传入正确的父类 `jclass`，否则会抛出 `IllegalAccessError`。

---

### 4. 将子类对象视为父类对象（向上转型）

在 JNI 中，你不需要像 Java 那样写 `(Animal) dogObj` 进行强制转换。因为 `jobject` 本身就是一个泛型句柄，**直接传递即可**。

```cpp
// 假设有一个 JNI 函数，它期望的参数是 jobject（父类类型）
JNIEXPORT void JNICALL Java_com_example_NativeLib_treatAsAnimal(JNIEnv* env, jobject thiz, jobject animalObj) {
    // 即使 Java 层传入的是 Dog 对象，这里也能直接操作
    // 但由于 animalObj 是 jobject，我们需要拿到 Animal 类来调用特定方法
    jclass animalClass = env->FindClass("com/example/Animal");
    jmethodID mid = env->GetMethodID(animalClass, "speak", "()V");
    env->CallVoidMethod(animalObj, mid); // 这里如果 Dog 重写了 speak，依然输出 "Woof!"
}
```

---

### 5. Android 开发中的致命陷阱（必看）

#### 陷阱一：`GetSuperclass` 返回的 `jclass` 是局部引用
- 如果在 `JNI_OnLoad` 之外调用 `GetSuperclass`，返回的 `jclass` 会在函数返回后失效。
- **解决**：如果父类 `jclass` 需要长期使用（如频繁用于 `CallNonvirtual`），必须用 `env->NewGlobalRef` 转为全局引用。

#### 陷阱二：在子线程中调用 `CallNonvirtual` 崩溃
- `CallNonvirtualVoidMethod` 本质上也是 JNI 方法调用。如果在 Native 子线程（`pthread`）中执行，必须先确保该线程通过 `AttachCurrentThread` 挂载，且使用的 `jclass` 和 `jmethodID` 必须是全局有效的。

#### 陷阱三：`CallNonvirtual` 传入错误的 Class 对象
- 如果你传入的是**子类**的 `jclass`，`CallNonvirtual` 的行为等同于普通的 `Call...Method`（仍然会触发多态），不会报错，但逻辑达不到“调用父类”的目的。
- **必须传入直接父类（或更高层级）的 `jclass`，且该 `jclass` 必须持有该方法 ID 对应的正确签名。**

#### 陷阱四：混淆（ProGuard/R8）对方法名的影响
- 如果 `speak()` 被混淆了，`GetMethodID` 会返回 `null`。
- **解决**：在 Java 层的方法上加上 `@Keep` 注解，或在混淆规则中明确保留 `-keepclassmembers class com.example.Animal { void speak(); }`。

---

### 总结一句话

| 场景 | 关键 API / 规则 |
| :--- | :--- |
| 获取父类 Class | `GetSuperclass(childClass)` |
| 读取父类字段 | 使用**父类的 `jclass`** 调用 `GetFieldID`，然后用该 ID 读对象字段 |
| 调用父类方法（非虚） | 使用 **`CallNonvirtual<Type>Method`**，第二个参数传入父类的 `jclass` |
| 向上转型（传递对象） | 直接传递 `jobject` 即可，无需任何强制转换语法 |
| 缓存策略 | 父类的 `jclass` 和父类的方法 ID 最好在 `JNI_OnLoad` 中一次性缓存为全局引用，避免反复 `GetSuperclass` 和 `FindClass` 的性能开销 |


# 内存管理(局部和全局引用)
在 JNI 编程中，**局部引用**和**全局引用**的本质区别在于**生命周期**和**所有权**。理解它们是写出稳定、无内存泄漏 Native 代码的分水岭。

简单来说：**局部引用是“自动管理”的临时工，函数返回即销毁；全局引用是“手动管理”的终身员工，不主动辞退（Delete）就永远存在。**

以下是深度拆解：

---

### 1. 核心本质：句柄（Handle）而非指针
JNI 返回的 `jobject`、`jclass` 等**不是直接的内存地址**，而是 JVM 内部引用表（Reference Table）的**索引（句柄）**。这种间接层允许 JVM 在 GC 时移动对象（压缩堆）而不影响 Native 层的“指针”值。

---

### 2. 局部引用（Local Reference）

- **来源**：`FindClass`、`NewObject`、`GetObjectField`、`CallObjectMethod` 等绝大多数 JNI 函数返回的默认类型。
- **生命周期**：**绑定于当前 Native 线程的当前栈帧**。当 Native 函数执行完毕（返回 Java 层）时，JVM 会**自动删除**该线程创建的所有局部引用。
- **容量限制**：每个线程的局部引用表有大小限制（Android 默认通常为 **512 个**）。超出即崩溃（`ReferenceTable overflow`）。
- **作用域**：**不能跨线程传递**。线程 A 创建的局部引用在线程 B 中无效（会 Crash）。

**实战管理**：
```cpp
// 危险代码：循环中不释放
for (int i = 0; i < 10000; i++) {
    jstring str = env->NewStringUTF("test"); // 第 513 次撑爆表，崩溃！
}

// 安全代码：主动删除
for (int i = 0; i < 10000; i++) {
    jstring str = env->NewStringUTF("test");
    // ... 使用 str ...
    env->DeleteLocalRef(str); // 立刻释放槽位，循环永不过载
}
```

---

### 3. 全局引用（Global Reference）

- **来源**：必须通过 `env->NewGlobalRef(local_ref)` **显式创建**。
- **生命周期**：**手动管理**。直到调用 `env->DeleteGlobalRef(global_ref)` 才会释放。否则即使 Java 层不再使用该对象，GC 也无法回收它。
- **容量限制**：全局引用表也有上限（通常远大于局部表，但总内存受限）。
- **作用域**：**可跨线程传递**。全局引用可在进程内的任何 Native 线程中使用。

**典型用法（结合你之前的 `JNI_OnLoad`）**：
```cpp
jclass localCls = env->FindClass("com/example/MyClass");
// 错误：直接赋值给全局变量，函数返回后 localCls 失效
// g_MyClass = localCls; 

// 正确：必须提升为全局引用
g_MyClass = (jclass)env->NewGlobalRef(localCls); 
env->DeleteLocalRef(localCls); // 局部引用及时释放
```

---

### 4. 弱全局引用（Weak Global Reference）—— 防内存泄漏神器

- **来源**：`env->NewWeakGlobalRef(local_ref)`。
- **特点**：允许 GC 回收底层 Java 对象。常用于缓存 **Activity** 或 **Context**，防止因长期持有导致内存泄漏。
- **使用铁律**：每次使用前**必须判空**，因为对象可能已被回收。

```cpp
// 使用时判空
jobject obj = env->NewLocalRef(g_weak_activity); // 转为局部引用用一下
if (env->IsSameObject(obj, NULL) == JNI_FALSE) {
    // 对象存活，可以安全使用
    env->DeleteLocalRef(obj);
} else {
    // 对象已被 GC 回收，需要重新获取或放弃操作
}
```

---

### 5. 必须澄清的易混淆点（jmethodID / jfieldID）

- **`jmethodID` 和 `jfieldID` 不是引用！** 它们只是 JVM 内部的“偏移量”或“索引”指针。
- **规则**：它们**无需释放**，且**可跨线程**。它们不参与引用计数，不阻止 GC。

---

### 6. 关键对比速查表

| 维度 | 局部引用 (Local) | 全局引用 (Global) | 弱全局引用 (Weak Global) |
| :--- | :--- | :--- | :--- |
| **生命周期** | Native 函数返回时自动释放 | 手动 `Delete` 前始终存活 | 手动 `Delete` 或 GC 回收底层对象 |
| **是否阻止 GC** | **是**（函数执行期间） | **是**（彻底阻止） | **否**（对象可被回收） |
| **跨线程传递** | **禁止** | **允许** | **允许** |
| **容量限制** | 较小（约 512） | 较大（受内存限制） | 较大 |
| **创建方式** | 隐式（默认返回） | `NewGlobalRef` | `NewWeakGlobalRef` |
| **释放方式** | 自动 或 `DeleteLocalRef` | `DeleteGlobalRef` | `DeleteWeakGlobalRef` |
| **典型场景** | 临时字符串、数组、临时对象 | 缓存 `jclass`、服务端回调对象 | 缓存 Context、Activity（防泄漏） |

---

### 7. Android 开发中的“血泪”陷阱

**陷阱一：`JNI_OnLoad` 中未提升为全局引用**
- 这是新手最常见的崩溃：在 `JNI_OnLoad` 中用 `FindClass` 赋给了全局 `jclass` 变量。
- **现象**：子线程（`pthread`）调用该变量时报错 `JNI ERROR (app bug): accessed stale local reference 0x...`。
- **解决**：必须 `NewGlobalRef`。

**陷阱二：将 Activity 保存为强全局引用**
- 如果在某个 JNI 函数中调用 `NewGlobalRef(activity)` 保存了 `MainActivity` 对象，当用户旋转屏幕或退出 Activity 时，该对象无法被回收，导致**严重的内存泄漏**。
- **解决**：使用 **`NewWeakGlobalRef`** 保存 Activity/Context，并在使用时判空。

**陷阱三：`DeleteLocalRef` 过度调用**
- 对于函数返回结果（如 `CallObjectMethod` 返回的字符串），如果立即 `Delete`，后续使用就会崩溃。**Delete 的条件是：确定该引用再也不会被使用**。

**陷阱四：试图 `Delete` `jmethodID`**
- 编译器不会报错，但运行时会忽略或产生未定义行为。**记住：只有 `jobject` 子类（`jclass`、`jstring`、`jarray`）才需要管理，`jmethodID` 和 `jfieldID` 绝对不要释放。**

---

### 总结一句话

> **在 JNI 中，局部引用是“用完即弃”的栈变量（管好循环，防止溢出），全局引用是“永生不死”的堆变量（管好释放，防止泄漏），而弱引用是“可回收”的缓存变量（使用时必须判空）。**


# onCreate native化
在 Android 开发中，将 `onCreate` “Native 化”并不是指在 C++ 里写 `setContentView(R.layout.main)`（这是做不到的），而是指**将原本写在 Java/Kotlin `onCreate` 中的业务初始化逻辑（如加载配置、启动引擎、预计算）剥离出来，放到 Native 层去执行**。

基于你之前对 `JNI_OnLoad`、`init_array` 和线程的深入理解，我们可以把“Native 化”分为三个等级：**预加载级**、**回调级** 和 **全生命周期级**。

---

### 等级一：预加载级（最快，但无界面交互）
**适用场景**：游戏引擎、图像处理算法、第三方 C++ SDK 的全局初始化。

直接在 **`JNI_OnLoad`** 或 **`__attribute__((constructor))`** 中初始化不依赖 `Context`（上下文）的模块。

```cpp
// 在 JNI_OnLoad 或 init_array 中
__attribute__((constructor))
void early_engine_init() {
    // 此时没有 JNIEnv，不能碰 Java 对象
    // 适合：初始化数学库、设置全局 log 回调、预分配内存池
    MyGameEngine::preloadResources();
}
```
> **限制**：拿不到 `Context` 和 `Activity` 实例，无法加载 `res/raw` 或 `assets` 里的资源（除非用 `AAssetManager` 后期传入）。

---

### 等级二：回调级（最常用，Java 主导，Native 干活）
**适用场景**：大多数业务 App。Java 层的 `onCreate` 负责搭 UI 架子（`setContentView`），然后立即回调 Native 层做具体的事。

**1. Java 层（精简到极致）**
```java
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main); // UI 还是得 Java 搭
    
    // 仅保留一行调用，所有逻辑在 Native 处理
    nativeOnCreate(this); 
}

private native void nativeOnCreate(Context context);
```

**2. Native 层（承接全部逻辑）**
```cpp
JNIEXPORT void JNICALL
Java_com_example_MainActivity_nativeOnCreate(JNIEnv* env, jobject thiz, jobject context) {
    // 1. 缓存全局 Context（必须是弱引用 WeakGlobalRef，防止泄漏）
    // g_context = env->NewWeakGlobalRef(context); 
    
    // 2. 获取 AssetManager，用于读取 assets 目录文件
    jclass ctxCls = env->GetObjectClass(context);
    jmethodID getAssets = env->GetMethodID(ctxCls, "getAssets", "()Landroid/content/res/AssetManager;");
    jobject assetManager = env->CallObjectMethod(context, getAssets);
    
    // 3. 初始化 Native 音视频引擎、加载数据库、创建子线程
    native_engine_init(env, assetManager);
    
    // 4. 创建子线程执行耗时任务，避免阻塞 UI 线程
    pthread_t tid;
    pthread_create(&tid, NULL, background_worker, NULL);
    pthread_detach(tid);
}
```

---

### 等级三：全生命周期级（Game/引擎方案）
**适用场景**：使用 `android.app.NativeActivity`，此时 `onCreate` 完全由 Native 接管（其实内部依然有 Java 壳，但对开发者透明）。

当你在 `AndroidManifest.xml` 中声明 `<activity android:name="android.app.NativeActivity">` 时，系统会调用 Native 层的 **`ANativeActivity_onCreate`** 函数。

```cpp
void ANativeActivity_onCreate(ANativeActivity* activity, void* savedState, size_t savedStateSize) {
    // 这就是 Native 版的 "onCreate"
    // 此时的 activity->env 是有效的 JNIEnv
    // activity->clazz 是 NativeActivity 的 jclass
    // activity->assetManager 可以直接操作资源
    
    // 设置生命周期回调
    activity->callbacks->onStart = my_onStart;
    activity->callbacks->onResume = my_onResume;
    activity->callbacks->onPause = my_onPause;
    activity->callbacks->onStop = my_onStop;
    activity->callbacks->onDestroy = my_onDestroy;
}
```
> **优点**：完全摆脱 Java 层 `onCreate` 的束缚。  
> **缺点**：布局渲染必须用 OpenGL 或 Vulkan 直接绘制到 `ANativeWindow`，无法使用 XML 布局文件。

---

### 必须跨越的四个“天堑”（Native 化的限制）

将 `onCreate` Native 化时，以下四个问题极易导致 ANR 或崩溃：

1.  **Context 的跨线程生命周期（致命）**
    - 如果在 `nativeOnCreate` 中将 `jobject context` 保存为**全局引用（`NewGlobalRef`）**，Activity 销毁时无法被 GC 回收，导致内存泄漏。
    - **解**：必须使用 **`NewWeakGlobalRef`**，并在子线程使用时先 `IsSameObject` 判空。

2.  **`FindClass` 在子线程失效**
    - 如果在 `pthread` 子线程中调用 `env->FindClass("com/example/MyClass")`，会因为类加载器缺失返回 `null`。
    - **解**：在 `nativeOnCreate`（主线程）中提前 `FindClass` 并缓存为全局引用，或者通过传入的 `jobject` 的 `GetObjectClass` 来间接获取。

3.  **ANR 风险（UI 线程阻塞）**
    - `nativeOnCreate` 是在 Java 主线程被调用的。如果你在 Native 代码里执行 `sleep(2)` 或复杂解密，界面会直接卡死 2 秒并弹出 ANR。
    - **解**：在 `nativeOnCreate` 中**只做轻量级初始化**（如保存指针、创建信号量）。耗时任务必须丢给 `pthread` 子线程执行。

4.  **资源加载的路径差异**
    - Java 层用 `getResources().openRawResource(R.raw.data)`，Native 层需要用 `AAssetManager_open` 配合 `assets` 目录。**注意**：Native 层无法通过资源 ID（`R.raw.data`）加载资源，只能通过文件名（`data.bin`）加载。

---

### 最佳实践建议（极简模板）

对于 90% 的 Android App，推荐采用 **“等级二：回调级”** 方案，并严格遵守分离原则：

```java
// Java 层只负责 UI 和传递 Context
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);
    MyNativeSdk.init(this); // 传入 Context，Native 层内部用 WeakGlobalRef 持有
}
```

```cpp
// Native 层：负责干重活
JNIEXPORT void JNICALL Java_MyNativeSdk_init(JNIEnv* env, jclass clz, jobject context) {
    // 1. 缓存弱引用
    g_weak_ctx = env->NewWeakGlobalRef(context);
    
    // 2. 获取 AssetManager 并初始化引擎（轻量级）
    // ... 
    
    // 3. 创建子线程执行真正的耗时初始化（如加载大模型）
    pthread_create(&g_init_thread, NULL, heavy_loading_task, NULL);
}
```

### 总结
**Native 化 `onCreate` 的本质是“职责下放”**：
- **UI 构建（XML 渲染）** 和 **系统回调注册** 必须留在 Java。
- **业务算法、数据解析、第三方引擎启动** 全部推到 Native。

切记：不要在 Native 代码中阻塞 `onCreate` 所在的 UI 线程；进入 Native 层后，立即将耗时逻辑异步化，这是保证不 ANR 的底线。😊


# 获取java方法名
在 Android Studio 中获取 Java 方法签名（Method Signature），最标准的方法是使用 JDK 自带的 `javap` 命令。

这里有几种不同的操作方式，你可以根据习惯选择。

### ⚙️ 核心方法：使用 `javap -s` 命令

`javap` 是 Java 类文件反汇编器，可以查看编译后的 `.class` 文件信息。加上 `-s` 参数后，它会打印出类型签名（即 JNI 所需的签名）。

1.  **编译项目**：首先在 Android Studio 中编译你的项目（`Build` -> `Make Project` 或 `Rebuild Project`），这样才能生成最新的 `.class` 文件。
2.  **定位 `.class` 文件**：编译后，`.class` 文件通常位于 `app/build/intermediates/javac/` 目录下。例如，如果你的类在 `com.example.myapp` 包下，路径可能是：
    `app/build/intermediates/javac/debug/classes/com/example/myapp/`

3.  **打开终端并切换目录**：在 Android Studio 底部打开 `Terminal` 窗口，使用 `cd` 命令切换到包含目标 `.class` 文件的目录。
4.  **执行命令**：运行以下命令查看签名：
    ```bash
    javap -s 你的类名.class
    ```
    例如，要查看 `MainActivity` 中所有方法的签名，可以执行：
    ```bash
    javap -s MainActivity.class
    ```
    

### 🚀 进阶技巧

*   **查看所有成员（包括私有）**：添加 `-p` 参数可以显示所有方法和字段，包括私有的。
    ```bash
    javap -s -p 你的类名.class
    ```

*   **查找特定方法**：如果类的方法很多，可以将输出通过管道传递给 `grep` 命令（在 Linux/macOS 下）进行过滤。
    ```bash
    javap -s -p 你的类名.class | grep "方法名"
    ```

### 🛠️ 进阶技巧：配置为 External Tool（外部工具）

为了方便，你可以将上述命令配置为 Android Studio 的一个外部工具，这样在代码编辑器中右键点击类或方法就能直接查看签名。

1.  **打开设置**：`File` -> `Settings`（Windows/Linux）或 `Android Studio` -> `Preferences`（macOS）。
2.  **找到 External Tools**：在设置窗口中，导航到 `Tools` -> `External Tools`。
3.  **添加工具**：点击 `+` 号添加一个新工具。
    *   **Name**：填写工具名称，例如 `Get Java Method Signature`。
    *   **Program**：填写 `$JDKPath$/bin/javap`（或 `javap`，如果已配置环境变量）。
    *   **Arguments**：填写 `-s -p $FileClass$`。
    *   **Working directory**：填写 `$OutputPath$`。
4.  **使用**：配置好后，在编辑器中打开目标 Java 文件，右键点击，选择 `External Tools` -> `Get Java Method Signature`，签名信息就会在底部的 `Run` 窗口中显示出来。

### 📝 其他方法：使用 `javac -h`

如果你需要生成 JNI 头文件（`.h` 文件），也可以使用 `javac -h` 命令，头文件中会包含方法签名。

```bash
javac -h [生成头文件的目录] [你的Java源文件.java]
```


### 💎 总结

*   **最常用**：`javap -s -p 类名.class` 是查看方法签名的标准方式。
*   **关键**：命令执行前，一定要先编译项目，生成最新的 `.class` 文件。
*   **提效**：将命令配置为 `External Tool` 可以极大提升使用效率。

使用 `javap` 时，你通常需要定位到 `build/intermediates/javac/` 目录下。如果你在操作中遇到找不到 `.class` 文件的问题，可以随时再问我。






# temp