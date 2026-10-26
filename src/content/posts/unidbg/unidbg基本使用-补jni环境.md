---
title: unidbg基本使用-补jni环境
published: 2026-10-01
description: unidbg基本使用-补jni环境
image: ./cover.jpg
tags: [逆向]
category: unidbg
draft: false
---

[unidbg基本使用一](https://www.yuque.com/lilac-2hqvv/xdwlsg/gmbn45b5n6g59kks?#%20%E3%80%8AUnidbg%20%E7%9A%84%E5%9F%BA%E6%9C%AC%E4%BD%BF%E7%94%A8%EF%BC%88%E4%B8%80%EF%BC%89%E3%80%8B  ) 
[unidbg基本使用二](https://www.yuque.com/docs/share/6f14c2e7-558c-4f4e-8c59-771c4073b51e?#%20%E3%80%8AUnidbg%20%E7%9A%84%E5%9F%BA%E6%9C%AC%E4%BD%BF%E7%94%A8%EF%BC%88%E4%BA%8C%EF%BC%89%E3%80%8B) 
[unidbg基本使用三](https://www.yuque.com/lilac-2hqvv/xdwlsg/bmf8lm68ffvhrfu0?#%20%E3%80%8AUnidbg%20%E7%9A%84%E5%9F%BA%E6%9C%AC%E4%BD%BF%E7%94%A8%EF%BC%88%E4%B8%89%EF%BC%89%E3%80%8B  ) 
[unidbg基本使用四](https://www.yuque.com/lilac-2hqvv/xdwlsg/uo6guvore38kr29y?#%20%E3%80%8AUnidbg%20%E7%9A%84%E5%9F%BA%E6%9C%AC%E4%BD%BF%E7%94%A8%EF%BC%88%E5%9B%9B%EF%BC%89%E3%80%8B  ) 
[unidbg基本使用五](https://www.yuque.com/lilac-2hqvv/xdwlsg/hiiaukxvr2zuz5mb?#%20%E3%80%8AUnidbg%20%E7%9A%84%E5%9F%BA%E6%9C%AC%E4%BD%BF%E7%94%A8%EF%BC%88%E4%BA%94%EF%BC%89%E3%80%8B  ) 
[unidbg基本使用六](https://www.yuque.com/lilac-2hqvv/xdwlsg/avlc01hrko6yvr26?#%20%E3%80%8AUnidbg%20%E7%9A%84%E5%9F%BA%E6%9C%AC%E4%BD%BF%E7%94%A8%EF%BC%88%E5%85%AD%EF%BC%89%E3%80%8B  ) 
[unidbg基本使用七](https://www.yuque.com/lilac-2hqvv/xdwlsg/ybo3xo4kloaqc1xq?#%20%E3%80%8AUnidbg%20%E7%9A%84%E5%9F%BA%E6%9C%AC%E4%BD%BF%E7%94%A8%EF%BC%88%E4%B8%83%EF%BC%89%E3%80%8B  ) 
[unidbg基本使用八](https://www.yuque.com/lilac-2hqvv/xdwlsg/td2t5cvohdedb878?#%20%E3%80%8AUnidbg%20%E7%9A%84%E5%9F%BA%E6%9C%AC%E4%BD%BF%E7%94%A8%EF%BC%88%E5%85%AB%EF%BC%89%E3%80%8B  ) 
[unidbg基本使用九](https://www.yuque.com/lilac-2hqvv/xdwlsg/nwqez8c4hm5hri8c?#%20%E3%80%8AUnidbg%20%E7%9A%84%E5%9F%BA%E6%9C%AC%E4%BD%BF%E7%94%A8%EF%BC%88%E4%B9%9D%EF%BC%89%E3%80%8B )




# unidbg基本使用(类和方法调用)


## 创建类和实例化
在 unidbg 中，创建类和实例化对象是补全 JNI 环境的基础操作，核心通过 `VM`（Dalvik 虚拟机）的 `resolveClass()` 和 `newObject()` 方法完成。

### 🧱 核心概念：DvmClass 与 DvmObject

在深入代码之前，需要理解 unidbg 中代表 Java 类和对象的两个核心类型：

*   **`DvmClass`**：代表一个 Java 类在虚拟环境中的“代言人”。它相当于 Java 的 `Class` 对象，可以用来调用静态方法、获取方法 ID 等。
*   **`DvmObject`**：代表一个 Java 对象在虚拟环境中的“壳”或句柄。当 SO 层持有 Java 对象引用时，这个引用在 unidbg 内部就表现为 `DvmObject`。它封装了对象的实际值，并提供了 `getValue()` 和 `setValue()` 来读写。

### 🏗️ 创建类：`vm.resolveClass()`

`resolveClass()` 方法让 unidbg 的虚拟机查找并加载指定的 Java 类。它返回一个 `DvmClass` 对象，后续的静态方法调用和实例创建都基于它进行。

**基本用法**：传入类的全限定名，**必须使用 `/` 分隔包名**，而不是 `.`。
```java
// 解析 com/example/SecurityUtils 类
DvmClass securityClass = vm.resolveClass("com/example/SecurityUtils");
```

**带父类的解析**：`resolveClass` 还支持传入第二个参数来指定父类，这在模拟 Android Framework 类的继承链时非常有用。
```java
DvmClass cContext = vm.resolveClass("android/content/Context");
DvmClass cContextWrapper = vm.resolveClass("android/content/ContextWrapper", cContext);
DvmClass cApplication = vm.resolveClass("android/app/Application", cContextWrapper);
```

### 🎯 实例化对象：`DvmClass.newObject()`

在获取 `DvmClass` 后，调用其 `newObject()` 方法即可创建该类的实例，返回一个 `DvmObject`。

**基本用法**：`newObject()` 接收可变参数，对应 Java 构造函数的参数。如果构造函数无参数，传入 `null` 即可。
```java
// 创建 MainActivity 的无参实例
DvmObject<?> activityObj = vm.resolveClass("com/dta/lesson2/MainActivity").newObject(null);
```
> 这里的 `MainActivity` 类名必须与 SO 中 JNI 函数注册的类名一致，否则 unidbg 在拼接 JNI 函数名时（如 `Java_com_dta_lesson2_MainActivity_md52`）会找不到对应符号。

**带参数的构造**：如果类有带参数的构造函数，可以直接传入参数，unidbg 会尝试匹配。
```java
// 假设 UserInfo 有一个接受 String 的构造函数
DvmObject<?> userInfo = vm.resolveClass("com/example/UserInfo").newObject("PREMIUM");
```

### 📦 对象引用管理：`vm.addLocalObject()`

创建 `DvmObject` 后，如果需要将其作为 `jobject` 传递给 SO 中的 JNI 函数，**必须**先通过 `vm.addLocalObject()` 将其注册到虚拟机的本地引用表中。该方法返回一个 **int 型的 hash 值**，这个 hash 值就是 SO 层看到的 `jobject` 引用。

```java
DvmObject<?> context = vm.resolveClass("android/content/Context").newObject(null);
// 将对象加入本地引用表，获取 jobject 引用（int hash）
int jobjectRef = vm.addLocalObject(context);

// 在调用 JNI 函数时，将这个 int 值作为 jobject 参数传递
List<Object> args = new ArrayList<>();
args.add(vm.getJNIEnv());
args.add(jobjectRef); // jobject 参数
```

### 🔄 代理对象：`ProxyDvmObject.createObject()`

对于 **JDK 标准库中的类**（如 `HashMap`、`JSONObject`、`ArrayList` 等），unidbg 提供了 `ProxyDvmObject.createObject()` 方法来快速封装一个真实的 Java 对象，而无需手动去 `resolveClass` 和 `newObject`。它会自动判断类型并做相应转换。

```java
Map<String, String> realMap = new HashMap<>();
realMap.put("key", "value");

// 将真实的 Java HashMap 包装成 DvmObject
DvmObject<?> mapObject = ProxyDvmObject.createObject(vm, realMap);
vm.addLocalObject(mapObject);
```

### 🔗 JNI 回调中的对象创建

当 SO 通过 JNI 回调要求创建 Java 对象时（如调用 `NewObject`），unidbg 会触发 `AbstractJni` 中的 `newObject` 回调。如果默认实现不满足需求，你需要重写该回调来返回正确的 `DvmObject`。例如，当 SO 请求创建 `ContextImpl` 时：
```java
@Override
public DvmObject<?> newObject(BaseVM vm, DvmClass dvmClass, String signature, VarArg varArg) {
    if ("android/app/ContextImpl-><init>()V".equals(signature)) {
        return vm.resolveClass("android/app/ContextImpl").newObject(null);
    }
    return super.newObject(vm, dvmClass, signature, varArg);
}
```

### ⚠️ 继承链预解析（高级技巧）

某些加固或混淆方案（如抖音的 `libmetasec_ml.so`）不会在 SO 内部初始化，而是**依赖 Java 层的类继承关系**来决定执行分支。此时需要在 `loadLibrary` **之前**，通过嵌套的 `resolveClass` 在 unidbg 的类表中预先建立继承链。

```java
// 建立 MS extends a0 extends k 的继承链
vm.resolveClass("com/bytedance/mobsec/metasec/ml/MS",
    vm.resolveClass("ms/bd/c/a0",
        vm.resolveClass("ms/bd/c/k")));
```
如果缺少这一步，SO 内部的 `FindClass` 或 `IsInstanceOf` 调用会失败，导致在业务调用时出现难以排查的错误。

### 💡 注意事项

*   **类名分隔符**：`resolveClass` 的参数**必须使用 `/`**，而不是 Java 源码中的 `.`。
*   **继承链的静默丢弃**：一个类一旦在类表中登记，后续再调用 `resolveClass` 为它补充父类或接口时，**第二次传入的参数会被静默丢弃**，不会有任何日志提示。因此，继承链的 `resolveClass` 调用必须一次写对、顺序正确。
*   **执行时机**：涉及继承链预解析的 `resolveClass` 调用，**必须出现在 `loadLibrary` 之前**。
*   **`addLocalObject` 的必要性**：直接 `newObject` 创建的对象**不能**直接作为 JNI 参数传递，必须经过 `addLocalObject` 注册，获取其 hash 值后才能传入。

总的来说，`resolveClass` 负责“告诉 unidbg 有哪些类”，`newObject` 负责“创建类的实例”，`addLocalObject` 负责“让 SO 能引用这个实例”，三者配合完成从类到可用 JNI 对象的完整流程。




## 静态和非静态方法的调用
在 unidbg 中，静态方法与实例方法的调用，核心区别在于**调用对象**和**传递的 JNI 引用类型**。静态方法通过 `DvmClass` 调用，传递的是 `jclass` 引用；实例方法通过 `DvmObject` 调用，传递的是 `jobject` 引用。

### 📞 静态方法调用：`DvmClass.callStaticJniMethod*`

静态方法属于类本身，因此调用入口是 `DvmClass` 对象。

**调用流程**：
1.  **获取类对象**：使用 `vm.resolveClass("包名/类名")` 获取 `DvmClass`。
2.  **调用方法**：使用 `DvmClass` 提供的 `callStaticJniMethod*` 系列方法。

**关键 API 速查**：
| 方法 | 适用返回类型 |
| :--- | :--- |
| `callStaticJniMethodObject` | 返回对象（如 `StringObject`, `ByteArray`） |
| `callStaticJniMethodInt` | 返回 `int` |
| `callStaticJniMethodBoolean` | 返回 `boolean` |
| `callStaticJniMethodLong` | 返回 `long` |
| `callStaticJniMethod` (通用) | 返回 `Number` |

**代码示例**（调用静态方法 `getSign`）：
```java
DvmClass signManager = vm.resolveClass("cn/thecover/lib/common/manager/SignManager");
String method = "getSign(Ljava/lang/String;Ljava/lang/String;Ljava/lang/String;)Ljava/lang/String;";

// 使用 callStaticJniMethodObject 获取对象返回值
StringObject result = signManager.callStaticJniMethodObject(
    emulator, 
    method, 
    "param1", "param2", "param3"
);
System.out.println("签名结果: " + result.getValue());
```

### 📞 实例方法调用：`DvmObject.callJniMethod*`

实例方法属于具体的对象，因此必须先创建或获取一个 `DvmObject` 实例，再通过该实例调用。

**调用流程**：
1.  **创建实例**：通过 `DvmClass.newObject(参数...)` 创建对象，返回 `DvmObject`。
2.  **调用方法**：使用 `DvmObject` 提供的 `callJniMethod*` 系列方法。

**代码示例**（调用实例方法 `md5`）：
```java
// 1. 获取类并创建实例
DvmClass securityUtils = vm.resolveClass("com/example/SecurityUtils");
DvmObject<?> instance = securityUtils.newObject(null); // 无参构造

// 2. 通过实例调用方法
String method = "md5(Ljava/lang/String;)Ljava/lang/String;";
DvmObject<?> result = instance.callJniMethodObject(emulator, method, "hello");
System.out.println("MD5: " + result.getValue());
```

### 🔑 核心差异：JNI 引用类型

理解静态与实例方法调用的关键，在于 JNI 函数第二个参数的不同。这是手动构造参数调用 `Module.callFunction` 时必须注意的。

| 方法类型 | 调用入口 | JNI 函数第二个参数 |
| :--- | :--- | :--- |
| **静态方法** | `DvmClass` | **`jclass`**（类的引用） |
| **实例方法** | `DvmObject` | **`jobject`**（对象的引用） |

当你使用 `callStaticJniMethod*` 或 `callJniMethod*` 时，unidbg 内部会自动处理这个差异。但如果通过 `module.callFunction` 手动构造参数调用，就需要自己注意：
- **静态方法**：第二个参数应传入 `dvmClass.hashCode()`。
- **实例方法**：第二个参数应传入 `vm.addLocalObject(dvmObject)` 返回的 hash 值。

### 📝 方法签名格式（Smali）

无论是静态还是实例方法，`method` 参数都必须使用 **Smali 语法**，格式为 `方法名(参数类型列表)返回类型`。

| Java 类型 | Smali 表示 |
| :--- | :--- |
| `void` | `V` |
| `boolean` | `Z` |
| `int` | `I` |
| `long` | `J` |
| `String` | `Ljava/lang/String;` |
| `byte[]` | `[B` |

**示例**：`getSign(String, String, String)` 返回 `String`，对应签名为：
`"getSign(Ljava/lang/String;Ljava/lang/String;Ljava/lang/String;)Ljava/lang/String;"`

### 💡 关键注意事项

*   **参数自动包装**：新版 unidbg 在调用 `callStaticJniMethod*` / `callJniMethod*` 时，对基本类型和常见对象类型（如 `String`, `byte[]`）已能自动包装，无需手动构造 `StringObject` 或 `ByteArray`。
*   **实例方法需先有对象**：调用实例方法前，必须通过 `newObject` 创建对象，或从已有对象（如 `JNI_OnLoad` 中回调传入的 `Context`）获取 `DvmObject`。
*   **静态注册 vs 动态注册**：对于动态注册的 JNI 方法，需确保 `JNI_OnLoad` 已被调用，否则 `findNativeFunction` 会找不到对应函数。
*   **调用失败排查**：若调用失败，可开启 `vm.setVerbose(true)`，观察 unidbg 是否打印了 `findNativeFunction` 的查找过程，以及 SO 内部是否回调了未实现的 Java 方法。




## 方法传入参数处理
### 基本类型
在 Unidbg 中，基本类型参数可以直接传递，Unidbg 会自动将其映射为 JNI 方法的入参。其支持的类型包括 `byte`、`short`、`int`、`long`、`float`、`double`、`boolean` 和 `char`

### 基本对象
基本对象也能够直接传递，因为unidbg在DvmObject类callJniMethod方法中已经帮我们处理了。
Boolean, String, byte\[]等等类型已经帮我们处理了
- jobject 对应于 DvmObject
- jclass 对应于 DvmClass
- jstring 对应于 StringObject
- jarray 对应于 BaseArray
- jobjectArray 对应于 ArrayObject
- jbyteArray 对应于 ByteArray
- jshortArray 对应于 ShortArray
- jintArray 对应于 IntArray
```java
if (args != null) {  
    for (Object arg : args) {  
        if (arg instanceof Boolean) {  
            list.add((Boolean) arg ? VM.JNI_TRUE : VM.JNI_FALSE);  
            continue;  
        } else if(arg instanceof Hashable) {  
            list.add(arg.hashCode()); // dvm object  
  
            if(arg instanceof DvmObject) {  
                vm.addLocalObject((DvmObject<?>) arg);  
            }  
            continue;  
        } else if (arg instanceof DvmAwareObject ||  
                arg instanceof String ||  
                arg instanceof byte[] ||  
                arg instanceof short[] ||  
                arg instanceof int[] ||  
                arg instanceof float[] ||  
                arg instanceof double[] ||  
                arg instanceof Enum) {  
            DvmObject<?> obj = ProxyDvmObject.createObject(vm, arg);  
            list.add(obj.hashCode());  
            vm.addLocalObject(obj);  
            continue;  
        }  
  
        list.add(arg);  
    }  
}
```

```java
public static DvmObject<?> createObject(VM vm, Object value) {  
    if (value == null) {  
        return null;  
    }  
    if (value instanceof Class<?>) {  
        return getObjectType(vm, (Class<?>) value);  
    }  
    if (value instanceof DvmObject) {  
        return (DvmObject<?>) value;  
    }  
  
    if (value instanceof byte[]) {  
        return new ByteArray(vm, (byte[]) value);  
    }  
    if (value instanceof short[]) {  
        return new ShortArray(vm, (short[]) value);  
    }  
    if (value instanceof int[]) {  
        return new IntArray(vm, (int[]) value);  
    }  
    if (value instanceof float[]) {  
        return new FloatArray(vm, (float[]) value);  
    }  
    if (value instanceof double[]) {  
        return new DoubleArray(vm, (double[]) value);  
    }  
    if (value instanceof String) {  
        return new StringObject(vm, (String) value);  
    }  
    Class<?> clazz = value.getClass();  
    if (clazz.isArray()) {  
        if (clazz.getComponentType().isPrimitive()) {  
            throw new UnsupportedOperationException(String.valueOf(value));  
        }  
        Object[] array = (Object[]) value;  
        DvmObject<?>[] dvmArray = new DvmObject[array.length];  
        for (int i = 0; i < array.length; i++) {  
            dvmArray[i] = createObject(vm, array[i]);  
        }  
        return new ArrayObject(dvmArray);  
    }  
  
    return new ProxyDvmObject(vm, value);  
}
```

### JDK标准库对象
对于 JDK 标准库对象（如 `HashMap`、`ArrayList`、`JSONObject` 等），在 Unidbg 中统一使用 `ProxyDvmObject.createObject(vm, value)` 进行封装。这个方法会将普通的 Java 对象转换为 Unidbg 可以识别的 `DvmObject`，以便在 JNI 调用中正确传递。

#### 核心方法：`ProxyDvmObject.createObject`

```java
import com.github.unidbg.linux.android.dvm.jni.ProxyDvmObject;

// 1. 创建普通的 Java 对象
Map<String, String> map = new HashMap<>();
map.put("key1", "value1");
map.put("key2", "value2");

// 2. 通过 ProxyDvmObject.createObject 封装为 DvmObject
DvmObject<?> mapObj = ProxyDvmObject.createObject(vm, map);

// 3. 添加到本地引用表（防止被回收）
mapObj = vm.addLocalObject(mapObj);

// 4. 传递给 Native 函数
module.callFunction(emulator, functionAddress, mapObj);
```

#### 常见 JDK 标准库对象示例

**HashMap / TreeMap：**

```java
Map<String, String> params = new TreeMap<>();
params.put("device", "MI6");
params.put("version", "1.0");

DvmObject<?> mapObj = ProxyDvmObject.createObject(vm, params);
mapObj = vm.addLocalObject(mapObj);
module.callFunction(emulator, functionAddress, mapObj);
```

**ArrayList / List：**

```java
List<String> list = new ArrayList<>();
list.add("item1");
list.add("item2");

DvmObject<?> listObj = ProxyDvmObject.createObject(vm, list);
listObj = vm.addLocalObject(listObj);
module.callFunction(emulator, functionAddress, listObj);
```

**JSONObject（如 `org.json.JSONObject`）：**

```java
JSONObject json = new JSONObject();
json.put("name", "test");
json.put("age", 25);

DvmObject<?> jsonObj = ProxyDvmObject.createObject(vm, json);
jsonObj = vm.addLocalObject(jsonObj);
module.callFunction(emulator, functionAddress, jsonObj);
```

### JDK非标准库对象
对于非 JDK 标准库对象（如 Android Framework 组件 `Context`、`SharedPreference`，或样本中自定义的类），处理的核心方法是 **`vm.resolveClass(className).newObject(value)`**。这与上轮讨论的 `ProxyDvmObject` 不同——`ProxyDvmObject` 适用于能在 JVM 中实际运行的 JDK 对象，而非 JDK 标准库对象在 Unidbg 中只是一个“占位符”，Unidbg 并不真正执行其内部逻辑。

#### 🎯 核心方法：`resolveClass` + `newObject`

```java
// 1. 解析类名（注意：类名使用 "/" 分隔，而非 "."）
DvmClass dvmClass = vm.resolveClass("com/example/MyClass");

// 2. 实例化对象
// 参数 value 是一个占位值，通常传 null 即可
DvmObject<?> dvmObject = dvmClass.newObject(null);
```

这一过程是 **"报错驱动"** 的：当 Native 代码调用 `FindClass("com/example/MyClass")` 时，Unidbg 会查找已注册的类；如果找不到，就会抛出 `NoClassDefFoundError`。你需要根据报错信息，在 Java 层预先 `resolveClass` 对应的类。

#### 🔗 处理类继承关系（关键）

当 Native 代码执行 `IsInstanceOf` 或需要访问父类方法时，必须让 Unidbg 知道类的继承链。这通过 `resolveClass` 的**嵌套调用**实现：

```java
// 建立继承链：MS ← a0 ← k
vm.resolveClass("com/bytedance/mobsec/metasec/ml/MS", 
    vm.resolveClass("ms/bd/c/a0", 
        vm.resolveClass("ms/bd/c/k")
    )
);
```

**读取顺序是从内向外**：
- 最内层：`ms/bd/c/k` 存在，无父类（或父类为 Object）
- 中间层：`ms/bd/c/a0` 存在，父类是 `ms/bd/c/k`
- 最外层：`com/.../MS` 存在，父类是 `ms/bd/c/a0`

**如果不建立继承链会怎样？** `loadLibrary` 和 `JNI_OnLoad` 都会成功，但业务调用时会在某个奇怪的地方崩溃——通常是 `FindClass` 失败或 `IsInstanceOf` 返回错误结果。




## 返回值处理
在 unidbg 中处理返回值，关键在于理解 `call*Method` 系列方法的**类型特异性**。你需要根据 JNI 方法签名中声明的返回类型，选择对应的调用方法，并对返回的对象进行正确提取。这与参数传递的逻辑类似，但方向相反。



### 🎯 返回值类型与方法选择

unidbg 为不同的返回类型提供了专门的调用方法，你需要根据方法签名（Smali 语法）中的返回值类型来选择。

| 返回类型 (Smali)     | 调用的方法                        | 返回的 Java 类型               | 说明             |
| :--------------- | :--------------------------- | :------------------------ | :------------- |
| `Z` (boolean)    | `callStaticJniMethodBoolean` | `boolean`                 | 返回布尔值          |
| `I` (int)        | `callStaticJniMethodInt`     | `int`                     | 返回整型           |
| `J` (long)       | `callStaticJniMethodLong`    | `long`                    | 返回长整型          |
| `V` (void)       | `callStaticJniMethod`        | `void`                    | **无返回值**，仅执行方法 |
| `L...;` (Object) | `callStaticJniMethodObject`  | `DvmObject<?>`            | 返回一个虚拟机对象      |
| `[B` (byte[])    | `callStaticJniMethodObject`  | `ByteArray` / `DvmObject` | 返回字节数组对象       |

对于实例方法，只需将 `callStaticJniMethod*` 替换为对应的 `callJniMethod*` 即可。

### 🛠️ 各类返回值的处理详解

#### 基本类型 (int, long, boolean 等)

调用 `callStaticJniMethodInt` 等方法会**直接返回 Java 基本类型的值**，无需额外转换。

```java
DvmClass clazz = vm.resolveClass("com/example/NativeLib");
// 调用返回 int 的方法
int result = clazz.callStaticJniMethodInt(emulator, "getVersion()I");
System.out.println("Version: " + result);
```

#### Void 类型

对于返回 `void` 的方法，使用通用的 `callStaticJniMethod`。它主要用于执行方法的副作用（如初始化），**没有返回值**。

```java
// 调用 void 方法进行初始化
clazz.callStaticJniMethod(emulator, "init(Landroid/content/Context;)V", context);
```

#### 对象类型 (String, byte[] 等)

这是最常用的场景。`callStaticJniMethodObject` 返回一个 **`DvmObject<?>`**，它是一个包装器，需要通过 **`.getValue()`** 方法来提取底层的真实值。

```java
// 调用返回 String 的方法
StringObject stringObj = clazz.callStaticJniMethodObject(emulator, 
    "getSign(Ljava/lang/String;)Ljava/lang/String;", "input");
String sign = stringObj.getValue(); // 提取实际的 String 值
System.out.println("Sign: " + sign);

// 调用返回 byte[] 的方法
DvmObject<?> resultObj = clazz.callStaticJniMethodObject(emulator, 
    "encrypt([B)[B", inputBytes);
byte[] encrypted = (byte[]) resultObj.getValue(); // 强转为 byte[]
```

> **注意**：如果 `callStaticJniMethodObject` 返回 `null`，通常意味着 SO 执行过程中出现了异常或初始化不完整，需要检查环境补全情况。

#### 数组类型 (String[], Object[] 等)

返回对象数组时，返回的 `DvmObject` 实际上是一个 **`ArrayObject`**。你需要将其强转，然后遍历其内部元素。

```java
DvmObject<?> result = clazz.callStaticJniMethodObject(emulator, 
    "getKeys()[Ljava/lang/String;");

if (result instanceof ArrayObject) {
    ArrayObject arrayObj = (ArrayObject) result;
    DvmObject<?>[] elements = arrayObj.getValue(); // 获取元素数组
    String[] keys = new String[elements.length];
    for (int i = 0; i < elements.length; i++) {
        if (elements[i] instanceof StringObject) {
            keys[i] = ((StringObject) elements[i]).getValue();
        }
    }
    // keys 即为 String[] 结果
}
```

#### Number 类型 (从寄存器读取的返回值)

当你通过 `module.callFunction` 或 `emulator.eFunc` **直接调用函数地址**时，返回的是底层寄存器的值，类型为 **`Number`**。你需要根据模拟器的位数（32/64位）来正确转换。

```java
Number ret = module.callFunction(emulator, offset, args);
long retValue;
if (emulator.is64Bit()) {
    retValue = ret.longValue(); // 64位架构，读取完整的长整型
} else {
    retValue = ret.intValue() & 0xffffffffL; // 32位架构，将 int 转为无符号 long
}
```

### 🚨 特殊场景与问题排查

#### 返回值 null 或错误（初始化问题）

**现象**：函数调用没有报错，但返回 `null` 或错误的值。

**原因**：这**几乎必然是初始化问题**。SO 可能依赖 Java 层主动调用的某个 native 初始化函数（如 `Java_com_xxx_Sec_init`）来完成内部状态准备。

**解决方案**：
1.  **Frida 对比**：在真机上用 Frida 先调用一遍初始化函数，再调用目标函数，验证是否正常。
2.  **逐个排查**：在 unidbg 中，按顺序调用 SO 导出的所有函数，找出那个关键的 `init` 函数。
3.  **静态分析**：结合 IDA 分析，确定初始化函数的调用链，在 unidbg 中按序模拟调用。

#### 需要修改返回值

有时需要在函数执行过程中动态修改其返回值，这可以通过在函数返回前的关键指令处设置断点来实现。

```java
emulator.attach().addBreakPoint(module.base + returnAddress, new BreakPointCallback() {
    @Override
    public boolean onHit(Emulator<?> emulator, long address) {
        // 修改返回寄存器 (ARM64: X0, ARM32: R0)
        RegisterContext ctx = emulator.getContext();
        ctx.setLongArg(0, 0xdeadbeef); // 设置返回值
        return true;
    }
});
```

### 💡 最佳实践

*   **方法选择**：**根据签名选方法**，不要试图用 `Object` 方法去接收基本类型返回值。
*   **对象提取**：**始终使用 `.getValue()`** 来提取 `DvmObject` 底层的真实值。
*   **数组处理**：对返回的 `ArrayObject` 要**先判断再强转**，然后遍历内部元素。
*   **底层调用**：直接调用函数地址时，返回的是 `Number`，要**根据位数正确转换**。
*   **异常排查**：遇到 `null` 或异常值，**优先怀疑初始化不完整**，而不是环境补全本身。





## 源码分析加载类并调用方法过程

### resolveClass
首先跟进resolveClass，实现在BaseVM类中
```java
// 来自BaseVM类
public final DvmClass resolveClass(String className, DvmClass... interfaceClasses) {  
    className = className.replace('.', '/');  
    int hash = Objects.hash(className);  
    DvmClass dvmClass = classMap.get(hash);  
    DvmClass superClass = null;  
    if (interfaceClasses != null && interfaceClasses.length > 0) {  
        superClass = interfaceClasses[0];  
        interfaceClasses = Arrays.copyOfRange(interfaceClasses, 1, interfaceClasses.length);  
    }  
    if (dvmClass == null) {  
        if (dvmClassFactory != null) {  
            dvmClass = dvmClassFactory.createClass(this, className, superClass, interfaceClasses);  
        }  
        if (dvmClass == null) {  
            dvmClass = this.createClass(this, className, superClass, interfaceClasses);  
        }  
        classMap.put(hash, dvmClass);  
    }  
    addGlobalObject(dvmClass);  
    return dvmClass;  
}  
  
@Override  
public DvmClass createClass(BaseVM vm, String className, DvmClass superClass, DvmClass[] interfaceClasses) {  
    return new DvmClass(vm, className, superClass, interfaceClasses);  
}
```
- 将传入的类名中的 `.` 替换为 `/`
- 使用 `Objects.hash(className)` 计算哈希值，在 `classMap`（一个 `Map<Integer, DvmClass>`）中查找是否已经解析过该类。**缓存机制避免了重复创建相同类的 `DvmClass` 对象**。
- 如果传入了 `interfaceClasses` 且长度大于 0，则**第一个元素被视为父类**（`superClass`）。
    
- 剩余的接口元素通过 `Arrays.copyOfRange` 截取，形成新的接口数组。

当缓存未命中时，按以下优先级创建 `DvmClass`：
1. **优先使用 `dvmClassFactory`**：如果外部设置了工厂（例如 `ProxyClassFactory`），则委托工厂创建。这允许 unidbg 在集成 APK 或进行 JNI 转发时，用 `ProxyClass` 替代默认的 `DvmClass`，从而支持更复杂的类行为模拟。
    
2. **回退到默认实现**：工厂未设置或返回 `null` 时，调用 `this.createClass(...)` 创建标准的 `DvmClass`。this.createClass()直接new了一个DvmClass对象
    
3. **存入缓存**：创建后的 `DvmClass` 以哈希值为键放入 `classMap`，后续相同类名的请求将直接命中缓存。

每次 `resolveClass` 返回前，都会调用 `addGlobalObject(dvmClass)`，**将 `DvmClass` 对象注册为 VM 的全局对象**，返回其哈希值。这样做是为了确保该 `DvmClass` 在后续 JNI 调用中拥有稳定的引用标识，避免被垃圾回收或重复创建


### callStaticJniMethod\[..]和callJniMethod\[..]
在resolveClass获得类后，通过类对象或者类本身调用非静态和静态方法，我们分别跟进源码中看看。
```java
// 来自DvmClass类
public void callStaticJniMethod(Emulator<?> emulator, String method, Object...args) {  
    try {  
        callJniMethod(emulator, vm, this, this, method, args);  
    } finally {  
        vm.deleteLocalRefs();  
    }  
}  
  
@SuppressWarnings("unused")  
public boolean callStaticJniMethodBoolean(Emulator<?> emulator, String method, Object...args) {  
    return BaseVM.valueOf(callStaticJniMethodInt(emulator, method, args));  
}  
  
@SuppressWarnings("unused")  
public int callStaticJniMethodInt(Emulator<?> emulator, String method, Object...args) {  
    try {  
        return callJniMethod(emulator, vm, this, this, method, args).intValue();  
    } finally {  
        vm.deleteLocalRefs();  
    }  
}  
  
@SuppressWarnings("unused")  
public long callStaticJniMethodLong(Emulator<?> emulator, String method, Object...args) {  
    try {  
        return callJniMethod(emulator, vm, this, this, method, args).longValue();  
    } finally {  
        vm.deleteLocalRefs();  
    }  
}  
  
@SuppressWarnings("unused")  
public <T extends DvmObject<?>> T callStaticJniMethodObject(Emulator<?> emulator, String method, Object...args) {  
    try {  
        Number number = callJniMethod(emulator, vm, this, this, method, args);  
        return vm.getObject(number.intValue());  
    } finally {  
        vm.deleteLocalRefs();  
    }  
}  
  
final boolean isInstance(DvmClass dvmClass) {  
    if (dvmClass == this) {  
        return true;  
    }  
  
    for (DvmClass dc : interfaceClasses) {  
        if (dc == dvmClass) {  
            return true;  
        }  
    }  
    if (superClass != null) {  
        return superClass.isInstance(dvmClass);  
    } else {  
        return false;  
    }  
}
```
这是DvmClass类的关于调用方法相关的函数，根据返回值不同，使用不同的函数后缀名。可以看到所有方法最终都调用 `callJniMethod`。如果是通过对象调用方法(即通过DvmObject调用)，最终同样会调用`callJniMethod`。
```java
// 来自DvmObject类
public void callJniMethod(Emulator<?> emulator, String method, Object...args) {  
    if (objectType == null) {  
        throw new IllegalStateException("objectType is null");  
    }  
    try {  
        callJniMethod(emulator, vm, objectType, this, method, args);  
    } finally {  
        vm.deleteLocalRefs();  
    }  
}  
  
@SuppressWarnings("unused")  
public boolean callJniMethodBoolean(Emulator<?> emulator, String method, Object...args) {  
    return BaseVM.valueOf(callJniMethodInt(emulator, method, args));  
}  
  
@SuppressWarnings("unused")  
public int callJniMethodInt(Emulator<?> emulator, String method, Object...args) {  
    if (objectType == null) {  
        throw new IllegalStateException("objectType is null");  
    }  
    try {  
        return callJniMethod(emulator, vm, objectType, this, method, args).intValue();  
    } finally {  
        vm.deleteLocalRefs();  
    }  
}  
  
@SuppressWarnings("unused")  
public long callJniMethodLong(Emulator<?> emulator, String method, Object...args) {  
    if (objectType == null) {  
        throw new IllegalStateException("objectType is null");  
    }  
    try {  
        return callJniMethod(emulator, vm, objectType, this, method, args).longValue();  
    } finally {  
        vm.deleteLocalRefs();  
    }  
}  
  
@SuppressWarnings("unused")  
public <V extends DvmObject<?>> V callJniMethodObject(Emulator<?> emulator, String method, Object...args) {  
    if (objectType == null) {  
        throw new IllegalStateException("objectType is null");  
    }  
    try {  
        Number number = callJniMethod(emulator, vm, objectType, this, method, args);  
        return objectType.vm.getObject(number.intValue());  
    } finally {  
        vm.deleteLocalRefs();  
    }  
}
```

### callJniMethod
所有方法最终都调用 `callJniMethod`，保证调用逻辑一致。callJniMethod来自DvmObject类
```java
// 来自DvmObject类
protected static Number callJniMethod(Emulator<?> emulator, VM vm, DvmClass objectType, DvmObject<?> thisObj, String method, Object...args) {  
    UnidbgPointer fnPtr = objectType.findNativeFunction(emulator, method);  
    vm.addLocalObject(thisObj);  
    List<Object> list = new ArrayList<>(10);  
    list.add(vm.getJNIEnv());  
    list.add(thisObj.hashCode());  
    if (args != null) {  
        for (Object arg : args) {  
            if (arg instanceof Boolean) {  
                list.add((Boolean) arg ? VM.JNI_TRUE : VM.JNI_FALSE);  
                continue;  
            } else if(arg instanceof Hashable) {  
                list.add(arg.hashCode()); // dvm object  
  
                if(arg instanceof DvmObject) {  
                    vm.addLocalObject((DvmObject<?>) arg);  
                }  
                continue;  
            } else if (arg instanceof DvmAwareObject ||  
                    arg instanceof String ||  
                    arg instanceof byte[] ||  
                    arg instanceof short[] ||  
                    arg instanceof int[] ||  
                    arg instanceof float[] ||  
                    arg instanceof double[] ||  
                    arg instanceof Enum) {  
                DvmObject<?> obj = ProxyDvmObject.createObject(vm, arg);  
                list.add(obj.hashCode());  
                vm.addLocalObject(obj);  
                continue;  
            }  
  
            list.add(arg);  
        }  
    }  
    return Module.emulateFunction(emulator, fnPtr.peer, list.toArray());  
}
```
`callJniMethod` 是 `DvmClass`（以及 `DvmObject`）中**真正执行 JNI 方法调用的底层静态方法**。它负责解析目标函数地址、组装 JNI 调用参数、注册局部引用，最终通过 Unicorn 引擎执行原生函数。下面逐段解析其实现。

---

#### 一、方法签名与定位
```java
protected static Number callJniMethod(Emulator<?> emulator, VM vm,
                                      DvmClass objectType, DvmObject<?> thisObj,
                                      String method, Object... args)
```

| 参数 | 含义 |
|------|------|
| `emulator` | Unicorn 模拟器实例，用于执行 ARM 指令 |
| `vm` | 当前虚拟机，管理 JNI 引用表、对象表 |
| `objectType` | 声明该方法的类（`DvmClass`），用于查找 native 函数地址 |
| `thisObj` | 调用对象（静态方法时等于 `objectType`） |
| `method` | 方法名或签名 |
| `args` | Java 侧传入的可变参数 |

---

#### 二、查找 native 函数地址

```java
UnidbgPointer fnPtr = objectType.findNativeFunction(emulator, method);
```

- 通过 `objectType.findNativeFunction` 解析出该方法对应的 **native 函数在内存中的地址**（`UnidbgPointer`）。
- 这一步会结合类的继承链、JNI 注册表、`RegisterNatives` 动态注册、符号导出等多种方式定位函数。
- 如果找不到，通常会抛出异常或返回 null，调用方需处理。

---

#### 三、注册调用对象的局部引用

```java
vm.addLocalObject(thisObj);
```

- 把被调用对象（`thisObj`）加入 VM 的**局部引用表**，返回其引用 ID（后续通过 `hashCode()` 获取）。
- 这是 JNI 的硬性要求：所有传给 native 方法的 Java 对象必须先创建局部引用（`NewLocalRef` 语义）。

---

#### 四、构造 JNI 参数列表

JNI 函数的 C 签名固定为：

```c
jret JNICALL func(JNIEnv* env, jobject thisObj, jarg1, jarg2, ...);
```

因此参数列表前两项固定是 `JNIEnv*` 和 `thisObj` 的引用 ID：

```java
List<Object> list = new ArrayList<>(10);
list.add(vm.getJNIEnv());      // 第一个参数：JNIEnv 指针
list.add(thisObj.hashCode());  // 第二个参数：this 对象引用 ID
```

随后遍历 `args`，对每个参数按类型分类处理：

##### 1. 布尔类型
```java
if (arg instanceof Boolean) {
    list.add((Boolean) arg ? VM.JNI_TRUE : VM.JNI_FALSE);
    continue;
}
```
- Java 的 `Boolean` 转为 JNI 约定的 `jboolean` 常量（`JNI_TRUE` / `JNI_FALSE`，即 1 / 0）。
- **不能直接传 Java boolean**，因为 Unicorn 只认数字。

##### 2. Hashable 类型（DvmObject 及子类）
```java
else if (arg instanceof Hashable) {
    list.add(arg.hashCode()); // dvm object
    if (arg instanceof DvmObject) {
        vm.addLocalObject((DvmObject<?>) arg);
    }
    continue;
}
```
- `Hashable` 是 unidbg 中所有可寻址对象的顶层接口（`DvmObject`、`DvmClass` 等都实现了它）。
- **`hashCode()` 在这里不是普通哈希，而是对象在 VM 中的引用 ID**。
- 若参数本身是 `DvmObject`，额外调用 `addLocalObject`，确保其局部引用有效。
- 这里不重复创建新对象，因为对象已存在于 VM 中。

##### 3. 需要包装为 DvmObject 的 Java 类型
```java
else if (arg instanceof DvmAwareObject ||
        arg instanceof String ||
        arg instanceof byte[] ||
        arg instanceof short[] ||
        arg instanceof int[] ||
        arg instanceof float[] ||
        arg instanceof double[] ||
        arg instanceof Enum) {
    DvmObject<?> obj = ProxyDvmObject.createObject(vm, arg);
    list.add(obj.hashCode());
    vm.addLocalObject(obj);
    continue;
}
```
- 这些是 Java 原生类型，native 函数期望它们以 **DvmObject 引用**形式传入。
- 通过 `ProxyDvmObject.createObject` 将其包装成 VM 里的 DvmObject：
  - `String` → `StringObject`
  - 各种数组 → `ArrayObject`
  - `Enum` → 枚举对象
  - `DvmAwareObject` → 由对象自身决定如何映射
- 包装后取 `hashCode()`（引用 ID）并注册局部引用。

##### 4. 其余类型（数字、指针等）
```java
list.add(arg);
```
- 包括 `int`、`long`、`float`、`double`、`UnidbgPointer` 等直接可放入寄存器/栈的原始值。
- 不做转换，直接作为参数传入。

---

#### 五、执行 native 函数

```java
return Module.emulateFunction(emulator, fnPtr.peer, list.toArray());
```

- 调用 `Module.emulateFunction`，这是 unidbg 封装的核心：
  - 将参数列表按 **AAPCS/ARM 调用约定**填充到寄存器（r0–r3）和栈上；
  - 用 Unicorn 执行 `fnPtr` 指向的 native 代码；
  - 返回值为 `Number`，可能是 `int`、`long`、`float`、`double` 等具体类型。
- `fnPtr.peer` 是 UnidbgPointer 的底层地址（`long`）。

---

#### 六、整体流程图

```
调用方 callStaticJniMethodXXX
        │
        ▼
   callJniMethod
        │
        ├─ findNativeFunction ──► 定位 fnPtr
        ├─ addLocalObject(thisObj)
        ├─ 构造 list：
        │     [JNIEnv, thisRef,
        │      Boolean → JNI_TRUE/FALSE,
        │      Hashable → hashCode() [+addLocalObject],
        │      String/数组/枚举/DvmAwareObject → ProxyDvmObject + hashCode + addLocalObject,
        │      其他 → 原值]
        │
        └─ Module.emulateFunction(emulator, fnPtr, args)
                │
                └─► Unicorn 执行 native 代码
                        │
                        └─► 返回 Number
```

---

#### 七、设计要点与注意事项

##### 1. JNI 调用约定严格遵循
- 前两个参数必须是 `JNIEnv*` 和 `this` 引用。
- 所有 Java 对象必须先注册局部引用，再以引用 ID 作为参数传递。
- 布尔值使用 JNI 宏定义而非 Java 值。

##### 2. 引用生命周期由外层管理
- `callJniMethod` 内部**只负责 add，不负责 delete**。
- 局部引用清理交给外层 `callStaticJniMethodXXX` 的 `finally { vm.deleteLocalRefs(); }`，保证一次调用结束后统一释放。

##### 3. 三种参数处理路径

| 类别 | 处理方式 |
|------|----------|
| 已在 VM 中的对象（`Hashable`） | 直接用 `hashCode()` 作为引用 ID |
| Java 原生值（String/数组/Enum） | 通过 `ProxyDvmObject` 包装后入 VM |
| 原始数值/指针 | 原样传递 |

##### 4. `hashCode()` 的语义
- 在普通 Java 中 `hashCode()` 用于哈希表；
- **在 unidbg 的 DvmObject/DvmClass 中，`hashCode()` 被重写为返回 VM 中的对象引用 ID**，这是 unidbg 模拟 JNI 引用机制的关键设计。

##### 5. 与静态/实例方法的关系
- `objectType` 决定函数查找范围；
- `thisObj` 决定第 2 个参数：
  - 静态方法：`thisObj == objectType`，传入类对象引用；
  - 实例方法：`thisObj` 是具体实例，传入实例引用。

---

#### 八、总结

`callJniMethod` 是 unidbg 中**JNI 调用参数编码的枢纽**：

1. **解析目标函数地址**（`findNativeFunction`）；
2. **按 JNI 规范组装参数列表**（`JNIEnv + this + 各类型转换`）；
3. **将 Java 类型映射为 VM 内引用 ID**（`Hashable.hashCode()` 或 `ProxyDvmObject`）；
4. **交由 Unicorn 执行 native 代码**（`Module.emulateFunction`）；
5. **返回值以 `Number` 形式回传**，由外层方法转换为具体 Java 类型。

它屏蔽了 ARM 调用约定、JNI 参数编码、对象引用管理的复杂度，是 unidbg 补环境时主动调用 native 方法的核心引擎。





## 学习脚本
```java
package com.myunidbg;  
  
import com.github.unidbg.AndroidEmulator;  
import com.github.unidbg.arm.backend.Unicorn2Factory;  
import com.github.unidbg.linux.android.AndroidEmulatorBuilder;  
import com.github.unidbg.linux.android.AndroidResolver;  
import com.github.unidbg.linux.android.dvm.*;  
import com.github.unidbg.memory.Memory;  
  
import java.io.File;  
  
public class MainActivity extends AbstractJni {  
    private final AndroidEmulator emulator;  
    private final DvmClass NativeApi;  
    private final VM vm;  
  
    public MainActivity(){  
        emulator = AndroidEmulatorBuilder  
                .for32Bit()  
                .addBackendFactory(new Unicorn2Factory(true))  
                .setProcessName("com.sina.oasis")  
                .build();  
        Memory memory = emulator.getMemory();  
        memory.setLibraryResolver(new AndroidResolver(23));  
        vm = emulator.createDalvikVM(new File("D:\\01ClassMP4\\unidbgfile\\q1\\lvzhou.apk"));  
        vm.setJni(this);  
        vm.setVerbose(true);  
        DalvikModule dm = vm.loadLibrary("oasiscore", true);  
        NativeApi = vm.resolveClass("com/weibo/xvideo/NativeApi");  
        dm.callJNI_OnLoad(emulator);  
    }  
  
    public String call(){  
        String arg1 = "aid=01A-khBWIm48A079Pz_DMW6PyZR8uyTumcCNm4e8awxyC2ANU.&cfrom=28B5295010&cuid=5999578300&noncestr=46274W9279Hr1X49A5X058z7ZVz024&platform=ANDROID&timestamp=1621437643609&ua=Xiaomi-MIX2S__oasis__3.5.8__Android__Android10&version=3.5.8&vid=1019013594003&wm=20004_90024";  
        Boolean arg2 = false;  
        StringObject ret = NativeApi.newObject(null).callJniMethodObject(emulator, "s([BZ)Ljava/lang/String;", arg1.getBytes(), arg2);  
        return ret.toString();  
    }  
  
    public static void main(String[] args){  
        MainActivity ma = new MainActivity();  
        String result = ma.call();  
        System.out.println("NativeApi.s result: " + result);  
    }  
}
```






### 📝 Log4j2：框架内部日志的配置中心

unidbg 使用 **Log4j2** 作为其底层日志框架，核心配置文件是 `log4j.properties`（或 `log4j2.xml`），通常位于 `src/test/resources/` 目录下。这个文件决定了 unidbg 框架自身（如系统调用、内存管理、库加载等模块）的日志级别和输出目的地。

通过修改 `log4j.properties`，你可以：

*   **控制日志级别**：将特定包（如 `com.github.unidbg.linux`）的级别设为 `DEBUG` 或 `TRACE`，以观察更详细的内部流程，例如系统调用的分发细节。
*   **追踪系统调用**：有资料提到，通过该配置文件可以对 `open`、`syscall` 这类系统调用进行追踪。
*   **重定向框架日志**：配置 Appender，将框架日志输出到文件、控制台或其他目的地，实现与业务日志的分离。

### 🔊 `vm.setVerbose()`：JNI 调用日志的总开关

这是日常分析中最常用的日志开关，**专门控制 JNI 调用和系统调用的打印**。

```java
VM vm = emulator.createDalvikVM();
vm.setJni(this);
vm.setVerbose(true); // 开启详细日志，打印每一次 JNI 调用和系统调用
```

**核心作用**：
*   **开启时**：unidbg 会打印出 Native 层对 Java 层的每一次 JNI 调用（方法名、参数、返回值），以及每一次系统调用的信息。这是**调试环境补全问题（如 `NoSuchMethodError`）的首要工具**。
*   **关闭时**：JNI 调用日志被静默，性能提升。

**使用建议**：
*   **调试阶段**：**务必开启** `setVerbose(true)`，它能清晰展示 SO 与外界的交互，帮助你快速定位缺失的环境实现。
*   **生产/性能测试阶段**：**强烈建议关闭**。因为高频 JNI 调用会产生海量日志，性能开销巨大，可能使执行速度下降 **10-100 倍**。

### 📊 日志输出的类别与性能影响

当 `debug` 模式或详细日志开启时，unidbg 会输出不同类别的日志，其性能影响差异显著：

| 日志类别 | 描述 | 性能影响 |
| :--- | :--- | :--- |
| **TRACE** | 指令级执行追踪，每条指令一行 | **极高（Very High）** |
| **MEMORY** | 内存分配/访问日志 | **高（High）** |
| **FUNCTION** | 函数调用边界（进入/退出） | **中（Medium）** |
| **ERROR** | 异常和错误条件 | **低（Low）** |

> 这解释了为什么 `traceCode()` 和 `traceRead/Write()` 在开启后性能会急剧下降，因为它们本质上是在生成 TRACE 和 MEMORY 级别的日志。

### 🎯 日志重定向：将输出写入文件

当日志量较大时（如 `traceRead` 的范围超过 `0x10000`），unidbg 会自动将 Trace 输出重定向到文件。对于 JNI 日志或自定义输出，你也可以手动重定向：

**方法一：重定向 `System.out`（全局生效）**
在程序入口处，将标准输出流重定向到文件，可以捕获所有 `System.out.println` 和 unidbg 默认打印到控制台的日志。
```java
PrintStream fileOut = new PrintStream(new File("jni_trace.log"));
System.setOut(fileOut);
```

**方法二：使用 Trace 的 `setRedirect`（针对特定 Trace）**
对于内存级 Trace，可以使用 `setRedirect` 方法，将特定追踪器的输出定向到文件流，不影响其他控制台输出。
```java
try (PrintStream traceStream = new PrintStream(new File("memory_trace.txt"))) {
    emulator.traceRead(0x10000L, 0x10100L).setRedirect(traceStream);
    emulator.traceWrite(0x10000L, 0x10100L).setRedirect(traceStream);
    // ... 执行目标函数
}
```

### 🧐 常见日志输出含义速查

*   **JNI 调用日志**：格式通常为 `[时间] [线程] [类名.方法名] (参数类型)返回值`。用于确认 Java 方法是否被正确调用、参数和返回值是否符合预期。
*   **系统调用警告（WARN）**：当 unidbg 遇到未实现的系统调用（SVC）时，会打印 `WARN` 日志。这是**补环境的重要线索**，提示你需要去实现对应的 SyscallHandler。
*   **内存访问日志**：格式如 `{date} Memory READ/WRITE at 0x{addr}, data size = ..., data value = ..., PC=..., LR=...`。每行都包含 **PC 和 LR**，让你能同时定位到“哪块内存被访问”和“哪条指令访问的、来自哪个调用”。

### ⚡ 性能考量与最佳实践

日志系统是**性能与可观测性之间的权衡**。遵循以下原则可以平衡二者：

1.  **分级开启**：调试时开启 `setVerbose(true)`，分析完成后立即关闭。使用 `log4j.properties` 精细控制框架内部日志级别，避免无关模块的 DEBUG 日志刷屏。
2.  **精确圈定**：**绝对避免全量 Trace**。无论是 `traceCode()` 还是 `traceRead()`，都必须根据静态分析结果精确圈定地址范围，这是控制日志量的第一原则。
3.  **及时重定向**：当日志量可能较大时，提前使用 `setRedirect` 或 `System.setOut` 将输出写入文件，避免控制台因海量日志而卡死。
4.  **生产环境静默**：在部署签名服务或进行性能压测时，确保 `setVerbose(false)` 且不开启任何 Trace，并使用 `Dynarmic` 后端以获得最佳性能。






# unidbg基本使用(加载虚拟机,so和依赖模块)
## 创建虚拟机
在 unidbg 中，创建虚拟机（VM）是加载 SO 和补全 JNI 环境的前提。`VM` 本质上是一个 **Dalvik 虚拟机的模拟实现**，负责管理 Java 类、对象、JNI 引用以及 Java 方法的调用。创建方式主要有三种，核心区别在于**是否传入 APK 文件**。

### 🏗️ 三种创建方式

#### 方式一：不传参数（最基础）

```java
VM vm = emulator.createDalvikVM();
```
创建一个空的 DalvikVM 实例，**不包含任何 APK 信息**。unidbg 无法自动获取包名、版本、签名等信息。适用于：
- 目标 SO 不依赖 APK 的包名或签名。
- 或者你准备**手动补全**所有环境信息。

#### 方式二：传入 APK 文件（推荐）

```java
VM vm = emulator.createDalvikVM(new File("target.apk"));
```
这是**最常用、最推荐**的方式。传入 APK 后，unidbg 会自动解析并填充大量环境信息：

| 自动获取的信息 | 用途 |
| :--- | :--- |
| **包名**（packageName） | SO 内部调用 `getPackageName()` 时返回正确值 |
| **版本号**（versionName） | 应对版本校验逻辑 |
| **版本代码**（versionCode） | 应对版本相关分支 |
| **签名**（signature） | 应对签名校验逻辑，**大幅减少手动补全** |
| **应用名**（applicationName） | 部分 SO 会校验应用名 |

**优势**：很多 SO 会通过 `PackageManager` 获取签名或包名进行校验。传入 APK 后，unidbg 能自动响应这些调用，避免因环境不一致导致的执行失败。

#### 方式三：传入 APK 路径字符串

```java
VM vm = emulator.createDalvikVM("/path/to/target.apk");
```
与方式二等价，内部会转换为 `File` 对象，属于便捷重载。

### 🔗 创建后的必要配置

创建 VM 后，通常需要完成以下配置才能正常执行 SO：

#### 1. 设置 JNI 处理器（核心）

```java
vm.setJni(new MyJniHandler()); // MyJniHandler 继承自 AbstractJni
```
JNI 处理器负责响应 SO 对 Java 层的所有调用。**不设置 JNI 处理器，SO 调用任何 Java 方法都会失败**。

#### 2. 开启详细日志（调试阶段）

```java
vm.setVerbose(true);
```
打印每一次 JNI 调用和系统调用，是**定位环境缺失问题的首要工具**。生产环境务必关闭。

#### 3. 加载 SO 文件

```java
DalvikModule dm = vm.loadLibrary(new File("libtarget.so"), true);
Module module = dm.getModule();
```
第二个参数 `true` 表示自动调用 `JNI_OnLoad`。**对于动态注册 JNI 函数的 SO，这一步至关重要**——它会触发 `RegisterNatives`，把 JNI 函数注册到 VM 中。

### 📊 三种创建方式对比

| 创建方式 | 包名/签名 | 适用场景 | 推荐度 |
| :--- | :--- | :--- | :--- |
| `createDalvikVM()` | ❌ 无 | SO 不依赖 APK 信息，或手动补全 | ★★ |
| `createDalvikVM(File)` | ✅ 自动解析 | **绝大多数场景** | ★★★★★ |
| `createDalvikVM(String)` | ✅ 自动解析 | 同上，语法糖 | ★★★★★ |

### 💡 最佳实践

**1. 优先传入 APK**
```java
// 推荐写法
VM vm = emulator.createDalvikVM(new File("target.apk"));
vm.setJni(new MyJniHandler());
vm.setVerbose(true);
DalvikModule dm = vm.loadLibrary(new File("libtarget.so"), true);
```

**2. APK 与 SO 的匹配**
传入的 APK **应当与目标 SO 来自同一个应用**。如果 APK 和 SO 不匹配，SO 内部的签名校验会失败，导致执行异常。

**3. 多个 SO 共享一个 VM**
同一应用的所有 SO 文件应**共享同一个 VM 实例**。因为 VM 维护了全局的类表、对象引用表，多个 SO 之间可能需要互相引用。

**4. `createDalvikVM` 的调用时机**
`createDalvikVM` 应在**加载任何 SO 之前**调用。因为 `loadLibrary` 依赖 VM 来解析 JNI 环境和注册动态方法。

**5. 签名校验的应对**
如果传入 APK 后签名校验仍失败，说明 SO 可能通过**自定义路径**读取签名（如直接读 APK 文件）。此时需要：
- 检查 `AbstractJni` 中 `getPackageInfo` 相关回调是否正确响应。
- 必要时重写相关回调，返回正确的签名信息。

**6. 版本兼容性**
不同版本的 unidbg 中 `createDalvikVM` 的签名可能略有差异。如果 IDE 提示方法不存在，检查 `AndroidEmulator` 接口中该方法的定义，确认是否需要调整调用方式。

### 💎 总结

创建虚拟机的核心就一句话：**`emulator.createDalvikVM(new File("target.apk"))`**。传入 APK 能自动完成大部分环境初始化，是 unidbg 补环境工作中**性价比最高的一步**。创建后务必设置 `setJni` 和 `setVerbose`，再调用 `loadLibrary` 加载 SO，就完成了 unidbg 的标准启动流程。



## 加载so
在 unidbg 中加载 SO 文件，核心 API 是 `VM.loadLibrary()`。它负责将 ELF 文件映射到模拟内存、解析符号、重定位，并可选地触发 `JNI_OnLoad`。理解加载流程和时机，是 SO 模拟执行的关键一步。

### 🚀 核心 API：`VM.loadLibrary()`

```java
DalvikModule dm = vm.loadLibrary(File file, boolean forceCallInit);
```

| 参数 | 类型 | 含义 |
| :--- | :--- | :--- |
| **`file`** | `File` | SO 文件对象（也支持传入 APK 路径字符串或字节数组） |
| **`forceCallInit`** | `boolean` | 是否强制调用 `JNI_OnLoad` |

**返回值 `DalvikModule`**：代表一个已加载的 SO 模块，通过它可以获取 `Module` 对象、访问导出符号、调用函数。

```java
DalvikModule dm = vm.loadLibrary(new File("libtarget.so"), true);
Module module = dm.getModule(); // 获取底层 Module 对象
```

### ⚙️ 加载流程详解

`loadLibrary` 内部完成的步骤：

1. **解析 ELF 头**：读取 `.so` 文件的 ELF 格式信息，确定架构（ARM32/ARM64）。
2. **内存映射**：将 SO 的各个段（`.text`、`.data`、`.rodata` 等）映射到模拟内存。
3. **依赖解析**：解析 SO 依赖的其他库（如 `libc.so`、`liblog.so`），并通过 `LibraryResolver` 提供系统库支持。
4. **符号解析与重定位**：处理导入符号，将 GOT/PLT 表中的引用指向正确地址。
5. **调用 `JNI_OnLoad`**（如果 `forceCallInit=true`）：触发 SO 的初始化逻辑。

### 🔑 `forceCallInit` 参数的关键作用

这个参数决定了是否自动调用 `JNI_OnLoad`，对动态注册的 SO 至关重要。

| 值 | 行为 | 适用场景 |
| :--- | :--- | :--- |
| **`true`** | 自动调用 `JNI_OnLoad` | **推荐**。SO 使用 `RegisterNatives` 动态注册 JNI 函数时必须为 `true`，否则后续调用找不到函数 |
| **`false`** | 不自动调用，需手动调用 | 需要手动控制初始化时机，或 `JNI_OnLoad` 会因环境不完整而崩溃时 |

**动态注册的典型流程**：
```java
// JNI_OnLoad 内部会调用 RegisterNatives，将函数注册到 VM
// 如果 forceCallInit=false，则必须手动调用
dm.callJNI_OnLoad(emulator);
```

### 📦 系统库支持：`LibraryResolver`

加载 SO 前，必须设置 `LibraryResolver`，否则依赖的系统库（如 `libc.so`）无法解析。

```java
Memory memory = emulator.getMemory();
memory.setLibraryResolver(new AndroidResolver(23)); // 支持 Android 6.0 (SDK 23)
```

**AndroidResolver 的作用**：根据指定的 SDK 版本，提供对应的系统库桩实现。如果不设置，SO 在重定位阶段就会因找不到 `libc.so` 等符号而失败。

### 🔗 加载多个 SO 的顺序

当一个应用包含多个 SO 且相互依赖时，加载顺序很重要：

```java
// 先加载被依赖的库
vm.loadLibrary(new File("libcrypto.so"), true);
// 再加载主库
vm.loadLibrary(new File("libtarget.so"), true);
```

**原则**：**被依赖的库先加载**。如果主库依赖其他自定义 SO，需要先加载被依赖的库，否则主库重定位时会找不到符号。

### 🛠️ 加载后的常用操作

#### 查找符号并调用函数

```java
// 通过符号名查找
Symbol symbol = module.findSymbolByName("Java_com_example_NativeLib_encrypt");
// 调用函数（相对偏移调用）
Number ret = module.callFunction(emulator, symbol.getValue(), args);
```

#### 通过 DvmClass 调用 JNI 方法

```java
DvmClass clazz = vm.resolveClass("com/example/NativeLib");
Number ret = clazz.callStaticJniMethodObject(emulator, 
    "encrypt(Ljava/lang/String;)Ljava/lang/String;", "input");
```

### ⚠️ 常见问题与排查

**1. `loadLibrary` 抛出 `IOException` 或 `IllegalStateException`**

**原因**：SO 依赖的系统库未找到，或架构不匹配。
**排查**：
- 确认 `LibraryResolver` 已设置且 SDK 版本正确。
- 确认模拟器位数与 SO 架构匹配（`for32Bit()` vs `for64Bit()`）。

**2. `JNI_OnLoad` 执行失败**

**原因**：`JNI_OnLoad` 内部调用了未补全的 Java 方法。
**排查**：开启 `vm.setVerbose(true)`，观察崩溃前最后调用的 JNI 方法，在 `AbstractJni` 子类中补全。

**3. 找不到 JNI 函数**

**原因**：动态注册未触发，或 `forceCallInit=false` 且未手动调用。
**排查**：确认 `forceCallInit=true`，或检查 `JNI_OnLoad` 是否被正确调用。

**4. 签名校验失败**

**原因**：`createDalvikVM` 未传入 APK，或 APK 与 SO 不匹配。
**排查**：使用 `vm.createDalvikVM(new File("target.apk"))` 传入正确 APK。

### 💡 最佳实践

1. **始终传入 APK 创建 VM**：`vm.createDalvikVM(new File("target.apk"))`，自动补全包名和签名。
2. **`forceCallInit` 设为 `true`**：除非有特殊原因，否则保持 `true`，确保动态注册的 JNI 函数可用。
3. **加载前设置 `LibraryResolver`**：这是 SO 重定位成功的前提。
4. **调试阶段开启 `setVerbose(true)`**：观察 `JNI_OnLoad` 执行过程中的 JNI 调用，快速定位缺失的环境。
5. **多 SO 按依赖顺序加载**：被依赖的库先加载，避免符号解析失败。
6. **加载后先验证符号**：通过 `findSymbolByName` 确认目标函数存在，再调用，避免无效操作。

### 💎 总结

加载 SO 的标准流程可以概括为四步：

```java
// 1. 创建 VM（传入 APK）
VM vm = emulator.createDalvikVM(new File("target.apk"));
vm.setJni(new MyJniHandler());
vm.setVerbose(true);

// 2. 设置系统库解析器
emulator.getMemory().setLibraryResolver(new AndroidResolver(23));

// 3. 加载 SO（自动调用 JNI_OnLoad）
DalvikModule dm = vm.loadLibrary(new File("libtarget.so"), true);
Module module = dm.getModule();

// 4. 查找符号并调用
Symbol symbol = module.findSymbolByName("target_function");
Number ret = module.callFunction(emulator, symbol.getValue(), args);
```

掌握这个流程，就完成了 unidbg 模拟执行 SO 的基础搭建。



## 加载依赖模块
在 unidbg 中加载依赖模块，是确保目标 SO 能正常重定位和执行的关键。unidbg 的依赖加载机制有其特定的行为，理解它才能避免“函数找不到”或“库加载失败”等问题。

### 🏗️ 依赖加载的核心机制

unidbg 的 ELF 加载器在解析 SO 文件时，会读取其 `.dynamic` 段中的 **`DT_NEEDED`** 条目，这些条目列出了该 SO 依赖的其他共享库名称。

**关键行为**：当调用 `vm.loadLibrary()` 时，**unidbg 默认不会自动递归加载这些 `DT_NEEDED` 指定的依赖库**。这一点与 Android 系统的动态链接器不同，Android 系统会递归解析并加载所有依赖，而 unidbg 需要你**显式地、按顺序**加载它们。

依赖加载流程可以概括为：
1. **解析 `DT_NEEDED`**：读取目标 SO 依赖的库列表。
2. **查找依赖库**：通过 `LibraryResolver` 或手动指定路径定位这些库文件。
3. **按序加载**：被依赖的库必须先于依赖它的库加载。
4. **符号重定位**：加载完成后，unidbg 解析并重定位目标 SO 的导入符号。

### 🎯 系统库依赖：AndroidResolver

对于 `libc.so`、`libm.so`、`liblog.so` 等 Android 系统库，unidbg 通过 **`AndroidResolver`** 来提供桩实现，**无需你手动加载这些库的真实文件**。

**正确配置方式**：
```java
Memory memory = emulator.getMemory();
// 设置 Android SDK 版本，目前支持 19 和 23
memory.setLibraryResolver(new AndroidResolver(23));
```
设置后，当 unidbg 遇到对 `libc.so` 等系统库的依赖时，会自动从 unidbg 内置的资源中加载对应的桩实现，解析所需的符号。

**`AndroidResolver` 的工作方式**：它实现了 `LibraryResolver` 接口，在 `resolveLibrary()` 方法中，根据指定的 SDK 版本（19 或 23），从类路径下的 `/android/sdk{sdk}/lib{32|64}/` 目录中查找对应的系统库文件。

### 📦 自定义依赖库：显式加载

对于应用自身的 SO 依赖（如 `libc++_shared.so`、`libtprt.so`、`libfoundrycomm.so` 等），必须**手动显式加载**。

**加载顺序原则**：**被依赖的库先加载**。

```java
// 假设 libtarget.so 依赖 libc++_shared.so 和 libhelper.so
// 1. 先加载被依赖的基础库
vm.loadLibrary(new File("libs/libc++_shared.so"), false);
vm.loadLibrary(new File("libs/libhelper.so"), false);

// 2. 再加载目标库
DalvikModule dm = vm.loadLibrary(new File("libs/libtarget.so"), true);
```

**`forceCallInit` 参数的策略**：对于**依赖库**，通常建议将 `forceCallInit` 设为 **`false`**，只进行内存映射和符号解析，不触发其 `JNI_OnLoad`。这样可以避免依赖库的初始化逻辑干扰主流程。对于**目标主库**，则设为 `true` 以触发其 `JNI_OnLoad`。

**一个真实案例**：在模拟腾讯手游加速器的 `libnetcomm.so`（TPRT 保护）时，必须**先显式加载 `libtprt.so` 和 `libfoundrycomm.so`**，因为 `vm.loadLibrary` 不会递归加载这些 `NEEDED` 依赖。

### 🔍 路径配置：addLibrarySearchPath

如果依赖库与目标 SO 不在同一目录，或者你希望 unidbg 在指定路径中自动查找依赖，可以使用 **`addLibrarySearchPath`** 方法。

```java
// 添加库搜索路径，unidbg 会在这些路径中查找依赖库
emulator.getMemory().addLibrarySearchPath("/path/to/your/libs");
```

配置搜索路径后，unidbg 在解析 `DT_NEEDED` 时，会尝试在这些路径中定位依赖库文件，简化手动指定每个依赖库路径的繁琐。

### 🛠️ 排查依赖加载问题的实用工具

当遇到“Library not found”或“Symbol not found”错误时，以下工具能帮你快速定位问题：

| 工具/命令 | 用途 |
| :--- | :--- |
| **`readelf -d target.so`** | 查看目标 SO 的 **`DT_NEEDED`** 条目，列出所有直接依赖的库名 |
| **`readelf -d target.so \| grep NEEDED`** | 快速提取依赖库列表 |
| **`nm -D libdependency.so`** | 查看依赖库的**导出符号表**，确认目标符号是否存在 |
| **`strings target.so \| grep "\.so"`** | 提取 SO 中隐含的依赖库字符串，发现 `DT_NEEDED` 未列出的**运行时动态加载**依赖 |

**排查流程**：
1. 用 `readelf -d` 列出目标 SO 的所有 `DT_NEEDED` 依赖。
2. 检查每个依赖是否已通过 `AndroidResolver`（系统库）或手动 `loadLibrary`（自定义库）加载。
3. 如果依赖已加载但符号仍找不到，用 `nm -D` 确认依赖库中确实导出了该符号。
4. 如果符号存在但仍报错，检查**加载顺序**是否正确——被依赖的库是否先于目标库加载。

### ⚠️ 常见陷阱与注意事项

**1. 动态加载（dlopen）的依赖**

有些 SO 会在运行时通过 `dlopen()` 动态加载其他库，而不是在 `DT_NEEDED` 中声明。unidbg 的 `dlopen` 实现会在调用时加载指定的库。如果 `dlopen` 失败，检查目标库文件是否存在、路径是否正确，以及 `addLibrarySearchPath` 是否覆盖了该路径。

**2. 依赖库的 `JNI_OnLoad` 不应轻易触发**

对于依赖库，`forceCallInit=false` 是更安全的选择。如果依赖库的 `JNI_OnLoad` 被意外调用，可能会执行不必要的初始化逻辑，甚至因环境不完整而崩溃。

**3. 符号版本不匹配**

如果依赖库的版本与目标 SO 期望的不一致，可能出现符号签名不匹配的情况。确保使用的依赖库版本与目标 APK 中的原始版本一致。

**4. 系统库的“假”加载**

`AndroidResolver` 提供的系统库是**桩实现**，并非真实的 Android 系统库。对于大多数标准符号（如 `malloc`、`memcpy`、`pthread_create` 等），unidbg 已有实现。但若 SO 调用了 `AndroidResolver` 中未实现的系统符号，仍需手动通过 `AbstractJni` 或自定义 `LibraryResolver` 来补全。

### 💡 最佳实践总结

1. **先设置 `AndroidResolver`**，让 unidbg 自动处理系统库依赖。
2. **用 `readelf -d` 分析目标 SO**，明确所有 `DT_NEEDED` 依赖。
3. **按依赖顺序显式加载自定义库**：被依赖的库先加载，`forceCallInit` 设为 `false`。
4. **最后加载目标主库**，`forceCallInit` 设为 `true`。
5. **必要时配置 `addLibrarySearchPath`**，简化依赖库定位。
6. **遇到符号问题时**，用 `nm -D` 验证符号是否存在，并检查加载顺序。

遵循以上流程，就能系统性地解决 unidbg 中绝大多数依赖模块加载问题。


## 学习脚本
```java
package com.myunidbg;  
  
import com.github.unidbg.AndroidEmulator;  
import com.github.unidbg.Emulator;  
import com.github.unidbg.Symbol;  
import com.github.unidbg.arm.HookStatus;  
import com.github.unidbg.arm.backend.Unicorn2Factory;  
import com.github.unidbg.hook.ReplaceCallback;  
import com.github.unidbg.hook.whale.IWhale;  
import com.github.unidbg.hook.whale.Whale;  
import com.github.unidbg.linux.android.AndroidEmulatorBuilder;  
import com.github.unidbg.linux.android.AndroidResolver;  
import com.github.unidbg.linux.android.dvm.*;  
import com.github.unidbg.memory.Memory;  
  
import java.io.File;  
  
public class WeiBo extends AbstractJni {  
    private final AndroidEmulator emulator;  
    private final VM vm;  
    private final DvmClass WeiboSecurityUtils;  
  
    public WeiBo(){  
        emulator = AndroidEmulatorBuilder  
                .for32Bit()  
                .addBackendFactory(new Unicorn2Factory(true))  
                .setProcessName("com.weibo.international")  
                .build();  
        Memory memory = emulator.getMemory();  
        memory.setLibraryResolver(new AndroidResolver(23));  
        vm = emulator.createDalvikVM(new File("D:\\01ClassMP4\\unidbgfile\\q2\\sinaInternational.apk"));  
        vm.setJni(this);  
        vm.setVerbose(true);  
  
        // hook free  
        IWhale whale = Whale.getInstance(emulator);  
        Symbol freeSym = emulator.getMemory().findModule("libc.so").findSymbolByName("free");  
        whale.inlineHookFunction(freeSym, new ReplaceCallback() {  
            @Override  
            public HookStatus onCall(Emulator<?> emulator, long originFunction) {  
                return HookStatus.LR(emulator, 0);  
            }  
        });  
        //  
  
        DalvikModule dm = vm.loadLibrary("utility", true);  
        WeiboSecurityUtils = vm.resolveClass("com/sina/weibo/security/WeiboSecurityUtils");  
        dm.callJNI_OnLoad(emulator);  
    }  
  
    public String callS(){  
        DvmObject<?> context = vm.resolveClass(  
                "android/app/Application",  
                vm.resolveClass(  
                        "android/content/ContextWrapper",  
                        vm.resolveClass("android/content/Context")  
                )  
        ).newObject(null);  
        String arg1 = "hello world";  
        String arg2 = "123456";  
        return WeiboSecurityUtils.newObject(null).callJniMethodObject(emulator, "calculateS(Landroid/content/Context;Ljava/lang/String;Ljava/lang/String;)Ljava/lang/String;", context, arg1, arg2).getValue().toString();  
    }  
  
    @Override  
    public DvmObject<?> callObjectMethod(BaseVM vm, DvmObject<?> dvmObject, String signature, VarArg varArg) {  
        switch (signature){  
            case "android/content/ContextWrapper->getPackageManager()Landroid/content/pm/PackageManager;":{  
                return vm.resolveClass("android/content/pm/PackageManager").newObject(null);  
            }  
        }  
        return super.callObjectMethod(vm, dvmObject, signature, varArg);  
    }  
  
    public static void main(String[] args){  
        WeiBo weiBo = new WeiBo();  
        String result = weiBo.callS();  
        System.out.println("calculateS result: " + result);  
    }  
}
```




# unidbg基本使用(虚拟模块与svc)
## unidbg的虚拟模块
`VirtualModule`（虚拟模块）是 Unidbg 为解决系统库依赖问题而设计的核心机制。它允许你用 Java 代码来“伪造”一个原本不存在的 SO 库，从而让目标 SO 文件能够正常加载和运行。

### 🎯 为什么需要虚拟模块？

在真实 Android 系统中，一个 SO 文件往往依赖大量的系统库（如 `libandroid.so`、`libmediandk.so`、`libjnigraphics.so` 等）。Unidbg 不可能完整模拟整个 Android 系统环境，因此设计了 `VirtualModule` 机制来“补”上这些缺失的库。

它的工作原理是：Unidbg 将 Java 实现的函数注册为一个“看起来像真 SO 的模块”，当目标 SO 通过 PLT 跳转调用这些函数时，会触发 SVC 指令，Unidbg 接管并执行对应的 Java 代码。

### 📦 Unidbg 自带的虚拟模块

Unidbg 已经内置了一些常用的 Android 虚拟模块，位于 `com.github.unidbg.virtualmodule.android` 包下：

| 虚拟模块类 | 模拟的系统库 | 主要功能 |
| :--- | :--- | :--- |
| `AndroidModule` | `libandroid.so` | 提供 `AAssetManager` 系列函数等 |
| `JniGraphics` | `libjnigraphics.so` | 提供 `Bitmap` 相关的 JNI 图形操作 |
| `MediaNdkModule` | `libmediandk.so` | 提供媒体 NDK 相关函数 |
| `SystemProperties` | — | 拦截 `__system_property_get` 系统属性获取 |

这些模块的使用方式非常简单，只需在加载目标 SO **之前** 创建并注册即可：

```java
// 创建 Dalvik VM
vm = emulator.createDalvikVM(new File("path/to/your.apk"));

// 注册虚拟模块（必须在 loadLibrary 之前）
new AndroidModule(emulator, vm).register(memory);
new JniGraphics(emulator, vm).register(memory);
new MediaNdkModule(emulator, vm).register(memory);
```

### 🛠️ 创建自定义虚拟模块

当目标 SO 依赖的库不在 Unidbg 内置列表中时，就需要自定义虚拟模块。核心是继承 `VirtualModule<T>` 抽象类，并实现 `onInitialize` 方法，在其中定义该模块导出的符号及其 Java 实现。

#### 基本步骤

**1. 继承 `VirtualModule` 类**
```java
public class MyCustomModule extends VirtualModule<VM> {
    // 构造方法中指定模块名称，这个名称就是目标 SO 依赖的库名
    public MyCustomModule(Emulator<?> emulator, VM vm) {
        super(emulator, vm, "libcustom.so");
    }
    
    @Override
    protected void onInitialize(Emulator<?> emulator, VM vm, 
                                 Map<String, UnidbgPointer> symbols) {
        // 在这里注册所有需要导出的函数
    }
}
```

**2. 在 `onInitialize` 中注册导出函数**

`symbols` 这个 `Map` 的 key 是导出函数名，value 是函数的实现指针。Unidbg 提供了 `SvcMemory` 来创建这些实现：

```java
@Override
protected void onInitialize(Emulator<?> emulator, VM vm,
                             Map<String, UnidbgPointer> symbols) {
    boolean is64Bit = emulator.is64Bit();
    SvcMemory svcMemory = emulator.getSvcMemory();
    
    // 为 64 位架构注册一个函数
    symbols.put("my_function", svcMemory.registerSvc(new Arm64Svc() {
        @Override
        public long handle(Emulator<?> emulator) {
            // 从寄存器中读取参数
            RegisterContext context = emulator.getContext();
            int arg0 = context.getIntArg(0);
            Pointer arg1 = context.getPointerArg(1);
            
            // 在这里写你的 Java 实现逻辑
            // ...
            
            return 0; // 返回值
        }
    }));
    
    // 如果需要支持 32 位，同样方式用 ArmSvc 注册
}
```

**3. 注册虚拟模块到内存中**

```java
// 创建并注册你的自定义虚拟模块
new MyCustomModule(emulator, vm).register(memory);

// 然后再加载目标 SO
DalvikModule dm = vm.loadLibrary(new File("libtarget.so"), true);
```

### 💡 完整示例：自定义一个返回固定字符串的函数

假设目标 SO 依赖 `libcustom.so` 中的 `get_secret_key` 函数，我们需要让它返回 `"unidbg_key"`：

```java
public class CustomModule extends VirtualModule<VM> {
    private static final String SECRET = "unidbg_key";
    
    public CustomModule(Emulator<?> emulator, VM vm) {
        super(emulator, vm, "libcustom.so");
    }
    
    @Override
    protected void onInitialize(Emulator<?> emulator, VM vm,
                                 Map<String, UnidbgPointer> symbols) {
        boolean is64Bit = emulator.is64Bit();
        SvcMemory svcMemory = emulator.getSvcMemory();
        
        symbols.put("get_secret_key", svcMemory.registerSvc(
            is64Bit ? new Arm64Svc() {
                @Override
                public long handle(Emulator<?> emulator) {
                    // 在模拟内存中分配字符串
                    return new StringObject(vm, SECRET).getPointer(vm).toIntPeer();
                }
            } : new ArmSvc() {
                @Override
                public long handle(Emulator<?> emulator) {
                    return new StringObject(vm, SECRET).getPointer(vm).toIntPeer();
                }
            }
        ));
    }
}
```

### ⚠️ 关键注意事项

**注册时机至关重要**：虚拟模块必须在 `vm.loadLibrary()` **之前**注册。因为 PLT 表的解析是在加载目标 SO 时完成的，PLT 表项指向的桩代码地址在那一刻就被固定下来了。之后再注册虚拟模块，目标 SO 的 PLT 表中没有对应记录，调用就会失败。

**符号名必须精确匹配**：`symbols.put()` 中的 key 必须与目标 SO 的 `DT_NEEDED` 条目和符号表中引用的名字完全一致，否则解析会失败。

**参数获取方式**：在 `handle` 方法中，通过 `emulator.getContext()` 获取 `RegisterContext`，然后根据函数签名用 `getIntArg(n)`、`getPointerArg(n)` 等方法读取第 n 个参数（从 0 开始）。

**返回值的处理**：返回 `long` 类型的值。如果函数返回指针，需要将 `UnidbgPointer` 转换为 `intPeer()` 或 `toIntPeer()` 后返回。



## svc软中断与jni调用
unidbg 通过巧妙复用 ARM 架构的 **SVC（Supervisor Call，管理调用）** 指令，将所有外部交互——包括 JNI 调用、系统调用和 Hook 回调——统一映射到同一个入口进行处理，从而实现了高效、透明的模拟执行。

### 🧠 设计理念：一条指令统一所有外部交互

在真实的 Android 系统中，SVC 指令是用户态程序请求内核服务的唯一通道，`open`、`read`、`mmap` 等所有系统调用最终都收敛到 `svc #0` 这一条指令。

unidbg 的天才设计在于**复用**了这一机制。它不仅在 SVC 上模拟系统调用，还将 **JNI 函数调用、Hook 回调、虚拟模块的函数实现**统统“伪装”成 SVC。这意味着，当 SO 代码通过函数表调用一个 JNI 函数时，实际上执行到的是一条 SVC 指令，从而触发 unidbg 的中断回调，由 unidbg 在回调中完成实际的 JNI 工作。

这种统一入口的设计带来了几个重要结果：
- **报错栈帧一致**：JNI 报错和 Syscall 报错的栈底都是同一个中断处理器，因此报错栈帧看起来几乎一模一样。
- **后端无关性**：由于 SVC 是 CPU 指令，所有后端（Unicorn、Dynarmic、KVM 等）都能触发并处理它，因此 JNI 调用和系统调用在所有后端上都能正常工作。
- **`intno=2` 的普遍性**：几乎所有 unidbg 报错中都出现的 `intno=2`，正是 SVC 异常对应的中断编号。

### ⚙️ 核心机制：JNI 函数表的 SVC 化注册

unidbg 的 JNI 模拟核心位于 `DalvikVM`（ARM32）和 `DalvikVM64`（ARM64）类中。在 VM 初始化时，它会**为每一个 JNI 函数注册一个 SVC 处理器**，并用这些处理器的地址填充 `JNIEnv` 和 `JavaVM` 的函数表。

**注册过程**由 `svcMemory.registerSvc()` 方法完成。该方法接收一个 `ArmSvc` 或 `Arm64Svc` 的实例，在内存中分配一小段空间，写入一条 SVC 指令，然后返回这段空间的指针。

以 `FindClass` 为例，其注册代码如下所示：

```java
Pointer _FindClass = svcMemory.registerSvc(new ArmSvc() {
    @Override
    public long handle(Emulator<?> emulator) {
        RegisterContext context = emulator.getContext();
        Pointer env = context.getPointerArg(0);      // JNIEnv*
        Pointer className = context.getPointerArg(1); // const char* 类名
        String name = className.getString(0);
        DvmClass dvmClass = resolveClass(name);
        // ... 后续处理
    }
});
```

`DalvikVM64` 中 `registerSvc` 的调用数量极多，涵盖了 JNI 规范定义的约 200 多个函数，包括 `GetVersion`、`FindClass`、`NewObject`、`GetMethodID`、`CallObjectMethod`、`GetFieldID`、`NewGlobalRef` 等所有核心类别。

### 🔗 完整调用链路：从 SO 到 Java

当 SO 中的代码调用一个 JNI 函数时，完整的执行链路如下：

1. **SO 加载函数指针**：Native 代码从 `JNIEnv` 结构中加载目标 JNI 函数的指针。
2. **执行到 SVC 指令**：这个指针指向的并不是真正的 C 函数，而是一段由 unidbg 生成的 **SVC 桩（SVC stub）**——本质上是一条 SVC 指令。
3. **触发中断**：CPU 模拟器执行到这条 SVC 指令，触发软件中断，`intno == ARMEmulator.EXCP_SWI`。
4. **后端分发**：模拟器后端捕获中断，将其分发给 unidbg 的 `SyscallHandler`。
5. **SVC 处理器识别**：`SyscallHandler` 根据触发中断的 **PC 地址**来判断这是哪个 JNI 函数的 SVC 桩，并找到对应的注册处理器。
6. **参数提取与执行**：处理器通过 `RegisterContext` 从 ARM 寄存器中提取参数（如 `JNIEnv*`、`jclass`、方法签名等），然后执行相应的 JNI 操作（如类查找、方法调用）。
7. **结果返回**：处理器的返回值通过 `R0`（ARM32）或 `X0`（ARM64）寄存器返回给 SO 代码，SO 继续执行。

### 📊 与系统调用的区分

unidbg 的 SVC 处理器需要区分两种 SVC 调用：

| SVC 类型 | 特征 | 处理方式 |
| :--- | :--- | :--- |
| **普通系统调用** | `swi == 0`（立即数为 0） | 从 `R7`（ARM32）或 `X8`（ARM64）读取系统调用号，分发到对应的系统调用处理器 |
| **SVC 回调** | `swi != 0` | 执行已注册的 SVC 处理器（即 JNI 函数、Hook 回调等） |

这种区分确保了 unidbg 能够同时处理系统调用和 JNI 调用，而不会相互干扰。

### 💎 总结

unidbg 的 SVC 机制是其**最核心的设计之一**。它将 JNI 调用、系统调用、Hook 回调等所有外部交互统一映射到 SVC 指令上，通过 **`svcMemory.registerSvc()` 注册处理器**、**PC 地址作为分发标识**、**寄存器传递参数和返回值**的方式，构建了一条从 Native 代码到 Java 世界的完整链路。理解了这个机制，就能明白为什么 unidbg 的 JNI 调用在所有后端上都能工作，以及为什么 JNI 报错和系统调用报错的栈帧如此相似。




## svc相关问题
你问到了 unidbg JNI 模拟最核心的两个环节：**JNI 函数指针如何变成 SVC 桩**，以及 **SVC 地址与 Svc 实例的映射关系**。下面分两部分详细拆解。

---

### 一、SO 中的 JNI 调用如何变成 SVC 指令

#### 1.1 JNIEnv 的本质：一张函数指针表

在 Android 的 JNI 规范中，`JNIEnv*` 在 native 层**并不是一个包含函数的对象**，而是一个**指向函数指针表的指针**。这张表就是 `JNINativeInterface` 结构体，它的内存布局是固定的：

```
JNIEnv* ──► ┌─────────────────────┐
            │ reserved0           │  (偏移 0)
            │ reserved1           │  (偏移 8)
            │ reserved2           │  (偏移 16)
            │ reserved3           │  (偏移 24)
            │ GetVersion          │  (偏移 32)
            │ DefineClass         │  (偏移 40)
            │ FindClass           │  (偏移 48)
            │ FromReflectedMethod │  (偏移 56)
            │ ...                 │
            │ GetStringUTFChars   │  (偏移 N)
            │ ...                 │
            └─────────────────────┘
```

当 SO 中写下 `(*env)->FindClass(env, "xxx")` 时，编译器实际生成的是：

```asm
mov x8, [x0, #48]    ; 从 env 指向的表 + 48 字节处取出 FindClass 的函数指针
blr x8               ; 调用这个函数指针
```

**关键点**：SO 本身并不知道 `FindClass` 的实现是什么，它只是从表里取一个指针然后调用。

#### 1.2 unidbg 的“偷梁换柱”

unidbg 在初始化 `DalvikVM` 时，**在模拟内存中构造了一张完整的 `JNINativeInterface` 表**，但表里填充的**不是真实的 JNI 函数地址，而是 SVC 桩的地址**。

具体过程：

1. **分配 JNIEnv 表内存**：在模拟内存中分配一块足够大的区域，用于存放函数指针表。
2. **为每个 JNI 函数注册 SVC**：调用 `svcMemory.registerSvc(new Arm64Svc() {...})`，得到一段 SVC 桩的地址。
3. **填充函数指针表**：将 SVC 桩地址按 JNI 规范规定的偏移，写入表中对应位置。例如 `FindClass` 对应偏移 48，就把 `_FindClass` 这个 SVC 桩地址写进去。
4. **传递 JNIEnv 给 SO**：当调用 `JNI_OnLoad` 或 native 方法时，把这个表的地址作为 `JNIEnv*` 传给 SO。

这样，SO 调用 `FindClass` 时，实际取到的是 SVC 桩的地址，执行到桩里的 `SVC` 指令，触发中断。

#### 1.3 完整调用链

```
SO 代码:  (*env)->FindClass(env, "com/example/Foo")
              │
              ▼
从 JNIEnv 表 + 48 字节处取出函数指针 = SVC 桩地址
              │
              ▼
跳转到 SVC 桩，执行 SVC #0 指令
              │
              ▼
Unicorn 捕获中断，交给 ARM64SyscallHandler
              │
              ▼
根据当前 PC 找到对应的 Svc 实例，调用其 handle()
              │
              ▼
handle() 中执行 Java 层逻辑，返回结果写入 x0
              │
              ▼
返回 SO，SO 认为 FindClass 调用完成
```

---

### 二、SVC 地址从哪来：SvcMemory 与 registerSvc

#### 2.1 SvcMemory：专门存放 SVC 桩的内存区域

unidbg 中有一个专门的类 `SvcMemory`，它继承自 `Memory`，负责在模拟内存中**分配并管理 SVC 桩代码**。

```java
public class SvcMemory extends Memory {
    private final Map<Long, Svc> svcMap = new HashMap<>();
    // ...
}
```

它内部维护了一个 `svcMap`，键是 SVC 桩的地址，值是对应的 `Svc` 实例。**这正是“SVC 地址 → Svc 实例”映射的来源**。

#### 2.2 registerSvc 的注册流程

当你调用 `svcMemory.registerSvc(new Arm64Svc() {...})` 时，内部做了以下几件事：

**第一步：分配内存**

在 `SvcMemory` 管理的地址空间中，分配一小段内存（通常是 4 字节或 8 字节对齐）。这段内存的起始地址就是未来的 SVC 桩地址。

**第二步：生成 SVC 指令机器码**

调用 `Svc.onRegister()` 方法，将 SVC 指令的机器码写入刚分配的内存。

对于 ARM64，`svc #0` 的机器码是 `0xD4000001`；对于 ARM32，`svc #0` 的机器码是 `0xEF000000`。

```java
// Arm64Svc 的 onRegister 大致逻辑
public void onRegister(SvcMemory svcMemory) {
    // 写入 SVC 指令
    byte[] code = { 0x01, 0x00, 0x00, (byte) 0xD4 }; // 小端序
    svcMemory.write(address, code, 0, code.length);
    // 可能还会写入 ret 指令等
}
```

**第三步：记录映射关系**

将分配到的地址和当前的 `Svc` 实例存入 `svcMap`：

```java
svcMap.put(address, this);
```

**第四步：返回地址**

`registerSvc` 返回一个 `Pointer`，指向这段 SVC 桩的起始地址。这个地址就是前面填入 JNIEnv 函数指针表的那个地址。

#### 2.3 SVC 触发后如何找到 Svc 实例

当 Unicorn 执行到 `SVC #0` 时，会触发一个中断异常。unidbg 的 `ARM64SyscallHandler`（继承自 `SyscallHandler`）会捕获这个异常。

处理流程：

1. **获取当前 PC**：从寄存器中读取 PC 值，这个值就是 SVC 桩的地址。
2. **查找 Svc 实例**：用 PC 值作为 key，在 `svcMap` 中查找对应的 `Svc` 实例。
3. **调用 handle**：找到后，调用该 `Svc` 实例的 `handle(emulator)` 方法。
4. **写回返回值**：将 `handle` 返回的值写入返回寄存器（ARM64 的 `x0`）。

```java
// ARM64SyscallHandler 中的大致逻辑
long address = context.getPCPointer().peer;
Svc svc = svcMemory.getSvc(address);
if (svc != null) {
    long result = svc.handle(emulator);
    context.setXLong(0, result);
    // 继续执行
}
```

#### 2.4 为什么用 SVC 而不是直接 Hook

你可能会问：为什么不直接把 JNIEnv 表中的函数指针指向一段 Java 回调，而非要绕一圈用 SVC？

原因在于 **Unicorn 的执行模型**：

- Unicorn 是一个纯 CPU 模拟器，它在执行 native 代码时**不会自动回到 Java 层**。
- 必须有一种机制能让 CPU 在执行过程中“主动暂停”并把控制权交还给 Java。
- **SVC 指令正是 CPU 提供的这种机制**——它触发一个异常，Unicorn 捕获后可以通过 hook 回调回到 Java 层。
- 如果直接放一个 Java 函数指针，CPU 无法识别，会直接崩溃。

所以，**SVC 是 native 代码和 Java 模拟层之间唯一的“合法越界通道”**。

---

### 三、整体图示

```
┌─────────────────────────────────────────────────────────────┐
│  SvcMemory（模拟内存中的一块区域）                            │
│                                                             │
│  地址 0x1000:  [SVC #0]  ← _FindClass 桩                    │
│  地址 0x1004:  [SVC #0]  ← _GetStringUTFChars 桩            │
│  地址 0x1008:  [SVC #0]  ← _CallObjectMethod 桩             │
│  ...                                                        │
│                                                             │
│  svcMap: { 0x1000 → FindClassSvc实例,                        │
│            0x1004 → GetStringUTFCharsSvc实例, ... }          │
└─────────────────────────────────────────────────────────────┘
                          │
                          │ 地址被填入
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  JNINativeInterface 表（模拟内存中的另一块区域）              │
│                                                             │
│  偏移 0:   reserved0                                        │
│  偏移 8:   reserved1                                        │
│  ...                                                        │
│  偏移 48:  0x1000  ← FindClass 的函数指针                    │
│  偏移 N:   0x1004  ← GetStringUTFChars 的函数指针            │
│  ...                                                        │
└─────────────────────────────────────────────────────────────┘
                          │
                          │ JNIEnv* 指向此表
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  SO 代码                                                     │
│                                                             │
│  (*env)->FindClass(env, "xxx")                              │
│      → 从 env+48 取出 0x1000                                │
│      → blr 0x1000                                           │
│      → 执行 SVC #0                                          │
│      → Unicorn 中断 → SyscallHandler                        │
│      → svcMap.get(0x1000) → FindClassSvc实例                │
│      → 调用其 handle() → 返回结果                            │
└─────────────────────────────────────────────────────────────┘
```

---

### 四、总结

| 问题                     | 答案                                                                                                   |
| ---------------------- | ---------------------------------------------------------------------------------------------------- |
| JNI 函数怎么变成 SVC 指令？     | unidbg 在模拟内存中构造 JNIEnv 函数指针表，表中每个槽位填入对应 JNI 函数的 SVC 桩地址。SO 调用 JNI 时取到的是 SVC 桩地址，执行到桩内的 `SVC` 指令触发中断。 |
| SVC 地址从哪来？             | 由 `SvcMemory.registerSvc()` 在模拟内存中分配，并写入 SVC 指令机器码后返回。                                               |
| 如何找到 SVC 地址对应的 Svc 实例？ | `SvcMemory` 内部维护 `svcMap`（地址 → Svc 实例），SVC 中断触发时，SyscallHandler 用当前 PC 值查表得到对应的 Svc 实例。              |
| 为什么必须用 SVC？            | Unicorn 是纯 CPU 模拟器，SVC 是 native 代码主动“暂停”并交还控制权给 Java 层的唯一合法机制。                                       |

这套设计的精妙之处在于：**它没有修改 SO 的任何代码，而是通过替换 JNIEnv 函数指针表的指向，把所有的 JNI 调用统一收敛到了 SVC 这一个入口**。这也是 unidbg 能够灵活模拟各种 JNI 行为、甚至支持自定义 Hook 的根本原因。




## 源码分析DalvikVM注册jni函数过程
emulator.createDalvikVM()创建DalvikVM。
交叉引用createDalvikVM发现方法由AndroidARMEmulator/AndroidARM64Emulator类实现

```java
// 来自 AndroidARMEmulator类
public final VM createDalvikVM(File apkFile) {  
    if (vm != null) {  
        throw new IllegalStateException("vm is already created");  
    }  
    vm = createDalvikVMInternal(apkFile);  
    return vm;  
}
```

```java
// 来自 AndroidARMEmulator类
private VM createDalvikVMInternal(File apkFile) {  
    return new DalvikVM(this, apkFile);  
}
```
继续跟进创建DalvikVM对象的构造函数
构造函数包含这两个操作:
- 创建JNIenv指针，通过svc注册JNI函数并填入函数表
- 创建JavaVM指针，通过svc注册JavaVM函数并填入函数表
先看有关JNIenv部分代码
```java
// 来自DalvikVM类
Pointer _GetVersion = svcMemory.registerSvc(new ArmSvc() {  
    @Override  
    public long handle(Emulator<?> emulator) {  
        return JNI_VERSION_1_6;  
    }  
});  
  
Pointer _DefineClass = svcMemory.registerSvc(new ArmSvc() {  
    @Override  
    public long handle(Emulator<?> emulator) {  
        throw new UnsupportedOperationException();  
    }  
});  
  
Pointer _FindClass = svcMemory.registerSvc(new ArmSvc() {  
    @Override  
    public long handle(Emulator<?> emulator) {  
        RegisterContext context = emulator.getContext();  
        Pointer env = context.getPointerArg(0);  
        Pointer className = context.getPointerArg(1);  
        String name = className.getString(0);  
  
        boolean notFound = notFoundClassSet.contains(name);  
        if (verbose) {  
            if (notFound) {  
                System.out.printf("JNIEnv->FindNoClass(%s) was called from %s%n", name, context.getLRPointer());  
            } else {  
                System.out.printf("JNIEnv->FindClass(%s) was called from %s%n", name, context.getLRPointer());  
            }  
        }  
  
        if (notFound) {  
            throwable = resolveClass("java/lang/NoClassDefFoundError").newObject(name);  
            return 0;  
        }  
  
        DvmClass dvmClass = resolveClass(name);  
        long hash = dvmClass.hashCode() & 0xffffffffL;  
        if (log.isDebugEnabled()) {  
            log.debug("FindClass env={}, className={}, hash=0x{}", env, name, Long.toHexString(hash));  
        }  
        return hash;  
    }  
});
```
通过svcMemory.registerSvc为jni函数注册svc，返回指针
```java
// 来自DalvikVM类
final int last = 0x3a4;  
final UnidbgPointer impl = svcMemory.allocate(last + 4, "JNIEnv.impl");  
for (int i = 0; i <= last; i += 4) {  
    impl.setInt(i, i);  
}  
impl.setPointer(0x10, _GetVersion);  
impl.setPointer(0x14, _DefineClass);  
impl.setPointer(0x18, _FindClass);  
impl.setPointer(0x1c, _FromReflectedMethod);
...
...
...
impl.setPointer(last, _GetModule);  
  
_JNIEnv = svcMemory.allocate(emulator.getPointerSize(), "_JNIEnv");  
_JNIEnv.setPointer(0, impl);
```
然后将jni函数指针填入JNIEnv指向的函数表中.
注册JavaVM函数也是同样的流程。
接下来分析Svc注册过程。
首先是ArmSvc对象的创建
```java
// 来自DalvikVM类
Pointer _GetVersion = svcMemory.registerSvc(new ArmSvc() {  
    @Override  
    public long handle(Emulator<?> emulator) {  
        return JNI_VERSION_1_6;  
    }  
});  
  
Pointer _DefineClass = svcMemory.registerSvc(new ArmSvc() {  
    @Override  
    public long handle(Emulator<?> emulator) {  
        throw new UnsupportedOperationException();  
    }  
});  
  
Pointer _FindClass = svcMemory.registerSvc(new ArmSvc() {  
    @Override  
    public long handle(Emulator<?> emulator) {  
        RegisterContext context = emulator.getContext();  
        Pointer env = context.getPointerArg(0);  
        Pointer className = context.getPointerArg(1);  
        String name = className.getString(0);  
  
        boolean notFound = notFoundClassSet.contains(name);  
        if (verbose) {  
            if (notFound) {  
                System.out.printf("JNIEnv->FindNoClass(%s) was called from %s%n", name, context.getLRPointer());  
            } else {  
                System.out.printf("JNIEnv->FindClass(%s) was called from %s%n", name, context.getLRPointer());  
            }  
        }  
  
        if (notFound) {  
            throwable = resolveClass("java/lang/NoClassDefFoundError").newObject(name);  
            return 0;  
        }  
  
        DvmClass dvmClass = resolveClass(name);  
        long hash = dvmClass.hashCode() & 0xffffffffL;  
        if (log.isDebugEnabled()) {  
            log.debug("FindClass env={}, className={}, hash=0x{}", env, name, Long.toHexString(hash));  
        }  
        return hash;  
    }  
});
```
其中重写的handle方法正是后续unidbg模拟执行so遇到对应jni函数时的处理逻辑。unidbg帮我们实现了一些常用的jni函数处理逻辑，另一些则是抛出异常，需要用户手动补环境。
跟进registerSvc注册过程
```java
// 来自ARMSvcMemory类
@Override  
public UnidbgPointer registerSvc(Svc svc) {  
    final int number;  
    if (svc instanceof ThumbSvc) {  
        if (emulator.is64Bit()) {  
            throw new IllegalStateException("is 64 bit mode");  
        }  
  
        if (++thumbSvcNumber == SyscallHandler.DARWIN_SWI_SYSCALL) {  
            thumbSvcNumber++;  
        }  
        number = thumbSvcNumber;  
    } else if (svc instanceof ArmSvc || svc instanceof Arm64Svc) {  
        if (svc instanceof ArmSvc && emulator.is64Bit()) {  
            throw new IllegalStateException("is 64 bit mode");  
        }  
        if (svc instanceof Arm64Svc && !emulator.is64Bit()) {  
            throw new IllegalStateException("is 32 bit mode");  
        }  
  
        if (++armSvcNumber == SyscallHandler.DARWIN_SWI_SYSCALL) {  
            armSvcNumber++;  
        }  
        number = armSvcNumber;  
    } else {  
        throw new IllegalStateException("svc=" + svc);  
    }  
    if (svcMap.put(number, svc) != null) {  
        throw new IllegalStateException();  
    }  
    return svc.onRegister(this, number);  
}
```
跟进svc.onRegister
```java
// 来自ArmSvc类
public UnidbgPointer onRegister(SvcMemory svcMemory, int svcNumber) {  
    ByteBuffer buffer = ByteBuffer.allocate(8);  
    buffer.order(ByteOrder.LITTLE_ENDIAN);  
    buffer.putInt(assembleSvc(svcNumber)); // svc #svcNumber  
    buffer.putInt(0xe12fff1e); // bx lr  
    byte[] code = buffer.array();  
    String name = getName();  
    UnidbgPointer pointer = svcMemory.allocate(code.length, name == null ? "ArmSvc" : name);  
    pointer.write(code);  
    return pointer;  
}
```
流程如下:
1. 创建 `ArmSvc`并实现handle方法。
    
2. 调用 `registerSvc(svc)`。
    
3. `registerSvc` 分配唯一 SVC 编号(number)，并存入 `svcMap`。
    
4. 调用 `svc.onRegister(this, number)`。
    
5. `onRegister` 在 `SvcMemory` 中生成：
```d
    svc #number
    bx lr
```
    
6. 返回这段代码的地址。
    
7. 目标程序调用这个地址时：
    
    - 执行 `svc #number`，触发模拟器异常。
        
    - Unidbg 捕获异常，根据 `number` 从 `svcMap` 找到对应的 `Svc`。
        
    - 调用该 `Svc` 的处理逻辑(重写的handle方法)。
        
    - 处理完后继续执行 `bx lr`，返回到调用者。




## 源码分析unidbg模拟执行so时调用java层方法
首先我们要知道安卓so层调用java层函数的过程。
1. **获取 JNIEnv 指针**
    
    - 如果当前线程是由 Java 调用的 Native 方法，`JNIEnv*` 会作为参数传入。
        
    - 如果是 Native 自己创建的线程，需要先通过 `JavaVM->AttachCurrentThread` 获取 `JNIEnv`，调用结束后 `DetachCurrentThread`。
        
2. **获取目标 Java 类的 `jclass`**
    
    - 使用 `FindClass("com/example/MyClass")`（注意包名用 `/` 分隔）。
        
    - 或者从已有对象调用 `GetObjectClass`。
        
3. **获取方法 ID**
    
    - 实例方法：`GetMethodID(clazz, "methodName", "签名")`
        
    - 静态方法：`GetStaticMethodID(clazz, "methodName", "签名")`
        
    - 方法签名可通过 `javap -s -p 类名` 获取。
        
4. **调用方法**
    
    - 实例方法：`Call<Type>Method(obj, methodID, 参数...)`
        
    - 静态方法：`CallStatic<Type>Method(clazz, methodID, 参数...)`
        
    - 类型包括 `Void`, `Int`, `Boolean`, `Object` 等。

根据这个过程，我们看看unidbg是如何在模拟执行so时调用java层方法的。
在DalvikVM中注册了很多的`Call<Type>Method`我们取其中的一个进行分析(callObjectMethod)

### CallObjectMethod
```java
// 来自DalvikVM类
Pointer _CallObjectMethod = svcMemory.registerSvc(new ArmSvc() {  
    @Override  
    public long handle(Emulator<?> emulator) {  
        RegisterContext context = emulator.getContext();  
        UnidbgPointer object = context.getPointerArg(1);  
        UnidbgPointer jmethodID = context.getPointerArg(2);  
        if (log.isDebugEnabled()) {  
            log.debug("CallObjectMethod object={}, jmethodID={}", object, jmethodID);  
        }  
        DvmObject<?> dvmObject = getObject(object.toIntPeer());  
        DvmClass dvmClass = dvmObject == null ? null : dvmObject.getObjectType();  
        DvmMethod dvmMethod = dvmClass == null ? null : dvmClass.getMethod(jmethodID.toIntPeer());  
        if (dvmMethod == null) {  
            throw new BackendException();  
        } else {  
            VarArg varArg = ArmVarArg.create(emulator, DalvikVM.this, dvmMethod);  
            DvmObject<?> ret = dvmMethod.callObjectMethod(dvmObject, varArg);  
            if (verbose) {  
                System.out.printf("JNIEnv->CallObjectMethod(%s, %s(%s) => %s) was called from %s%n", dvmObject, dvmMethod.methodName, varArg.formatArgs(), ret, context.getLRPointer());  
            }  
            return addLocalObject(ret);  
        }  
    }  
});
```



#### 1. 注册 SVC 钩子

```java
Pointer _CallObjectMethod = svcMemory.registerSvc(new ArmSvc() { ... });
```

- `svcMemory` 是一块专门用于分发 JNI 调用的内存页。
- `registerSvc` 把一个"桩"写入内存,并在该地址登记一个回调。
- 当 native 代码执行到这个桩地址时,unicorn 通过 SVC 指令触发回调。

被调用的宿主 native 代码通常来自 `JNIEnv` 的函数表(在 unidbg 里由 `DalvikVM` 构造),函数表项的指针就指向这块 svcMemory。

#### 2. 取出参数

```java
RegisterContext context = emulator.getContext();
UnidbgPointer object   = context.getPointerArg(1); // x1: jobject
UnidbgPointer jmethodID = context.getPointerArg(2); // x2: jmethodID
```

这里把 32/64 位寄存器读成 `UnidbgPointer`。注意 `getPointerArg` 内部会做位宽处理,对于 32 位是 int 语义,对 64 位是长整型。

#### 3. 通过 peer 找回 Java 对象与方法

```java
DvmObject<?> dvmObject = getObject(object.toIntPeer());
DvmClass dvmClass = dvmObject == null ? null : dvmObject.getObjectType();
DvmMethod dvmMethod = dvmClass == null ? null : dvmClass.getMethod(jmethodID.toIntPeer());
```

关键点:`toIntPeer()`,**不是** `toInt()`。

- unidbg 内部维护了一张 `Map<Integer, DvmObject<?>>`,俗称 `objectMap`(peer 表)。
- **peer** 是 unidbg 自己分配的、与 native 侧引用一一对应的“句柄”整数。
- native 侧看到的是 `UnidbgPointer`,但真正查表要的是 peer(int),因此要 `toIntPeer()`。

如果 `dvmObject == null`,说明这个 jobject 不是由 unidbg 注册的(例如 `jclass`/`jmethodID` 可能是另一套体系),也可能是传入了一个未跟踪的指针。这时 `dvmClass` 也为 null,`dvmMethod` 也为 null,直接进 `throw new BackendException()`。

同理 `jmethodID.toIntPeer()` 用于定位具体方法。在 unidbg 里,`jmethodID` 本身就是被注册进 DvmClass 的方法句柄对应的 peer。

#### 4. 找不到方法就抛 BackendException

```java
if (dvmMethod == null) {
    throw new BackendException();
}
```

`BackendException` 是 unidbg 的“未实现/错误”异常,会中断模拟。说明调用方(被模拟的 native 代码)使用了 unidbg 未识别的方法 ID,通常意味着:
- 该方法没有被 unidbg 的 DalvikVM 的 `registerMethod`/`DvmClass` 注册过;
- 或方法签名对不上。

#### 5. 组装可变参数

```java
VarArg varArg = ArmVarArg.create(emulator, DalvikVM.this, dvmMethod);
```

- 因为 JNI 的 `CallObjectMethod` 是 C 可变参数,后面几个参数的类型由 Java 方法的签名决定。
- `ArmVarArg` 是平台相关的实现(Arm / Arm64 / X86 各有一套,通过工厂方法 `create` 选择)。
- 它从 `dvmMethod` 里取参数类型列表,然后依次从 x3, x4, x5...(或栈上)读取对应类型参数。
- 支持 int/long/float/double/对象/字符串等。

#### 6. 调用方法

```java
DvmObject<?> ret = dvmMethod.callObjectMethod(dvmObject, varArg);
```
```java
// 来自DvmMethod类
DvmObject<?>  callObjectMethod(DvmObject<?>  dvmObject, VarArg varArg) {  
    BaseVM vm = dvmClass.vm;  
    return checkJni(vm, dvmClass).callObjectMethod(vm, dvmObject, this, varArg);  
}
```
- `callObjectMethod` 是 DvmMethod 的核心入口。
- 它内部会去查找：
  - `AbstractJni` 的实现类(用户实现的 `Jni` 回调);
  - `DvmClass` 上用户 `registerMethod` 注册的 DvmMethod 实现;
  - 或是内置的 `ProxyDvmObject` 等。
- 返回 `DvmObject<?>`,`null` 表示 Java 返回值是 null。

#### 7. 打印 verbose 日志

```java
if (verbose) {
    System.out.printf("JNIEnv->CallObjectMethod(%s, %s(%s) => %s) was called from %s%n",
        dvmObject, dvmMethod.methodName, varArg.formatArgs(), ret, context.getLRPointer());
}
```

- `context.getLRPointer()` 拿到返回地址(x30/LR),即**调用方的 PC**,这对定位 native 中是哪一处 JNI 调用很有用。

#### 8. 注册局部引用并返回

```java
return addLocalObject(ret);
```

- **这是 JNI 语义的关键**。
- JNI 中 `CallObjectMethod` 返回的 `jobject` 是 **局部引用(local reference)** 而不是全局引用。
- `addLocalObject(ret)` 做两件事:
  1. 若 `ret` 非空,把它塞进**当前线程**的局部引用表(由 `DalvikVM` 内部维护,通常是一个栈或列表);
  2. 返回一个 peer(int)的指针给 native 侧当作 `jobject`。
- 之后 native 侧可以用 `DeleteLocalRef` 释放它,或等 native 方法返回时由 unidbg 自动清理。

---

#### 整体数据流

```
native 调用 CallObjectMethod
      │
      ▼
 svcMemory 桩 -> handle()
      │
      ▼
 从 x1/x2 取 object、jmethodID
      │
      ▼
 peer -> objectMap -> DvmObject / DvmClass / DvmMethod
      │
      ▼
 ArmVarArg 解析可变参数(按方法签名)
      │
      ▼
 DvmMethod.callObjectMethod()  ->  用户注册的 Java 侧实现
      │
      ▼
 addLocalObject(ret) -> 返回一个 peer 给 native
```

---

#### 几个容易踩的坑

1. **`toIntPeer()` 不是 `toInt()`**  
   `UnidbgPointer.toIntPeer()` 是 unidbg 的约定:指针里放的不是真实地址,而是它内部的 peer 序号(为了节省空间/避免与真实内存冲突)。别写成 `toInt()` 否则查不到对象。

2. **参数下标从 1 开始**  
   x0 是 JNIEnv,不是 obj。很多人会误以为 `getPointerArg(0)` 就是 obj。

3. **`BackendException` 不是空指针异常**  
   它代表 unidbg 没有登记这个方法,通常需要在 `AbstractJni` 里实现对应方法,或者用 `dvmClass.registerMethod` 手工注册。

4. **局部引用 vs 全局引用**  
   返回前调用 `addLocalObject`,否则 native 侧拿到的 peer 可能不在表里,后面的 `GetObjectClass`、`GetMethodID` 等会崩。

5. **可变参数与签名强相关**  
   `ArmVarArg` 读几个参数完全由 `dvmMethod` 的签名决定。如果签名注册错了,参数就会读串位。

---


### 整体流程
1. DalvikVM注册CallObjectMethod函数，通过svc拦截
```java
// 来自DalvikVM类
Pointer _CallObjectMethod = svcMemory.registerSvc(new ArmSvc() {  
    @Override  
    public long handle(Emulator<?> emulator) {  
        RegisterContext context = emulator.getContext();  
        UnidbgPointer object = context.getPointerArg(1);  
        UnidbgPointer jmethodID = context.getPointerArg(2);  
        if (log.isDebugEnabled()) {  
            log.debug("CallObjectMethod object={}, jmethodID={}", object, jmethodID);  
        }  
        DvmObject<?> dvmObject = getObject(object.toIntPeer());  
        DvmClass dvmClass = dvmObject == null ? null : dvmObject.getObjectType();  
        DvmMethod dvmMethod = dvmClass == null ? null : dvmClass.getMethod(jmethodID.toIntPeer());  
        if (dvmMethod == null) {  
            throw new BackendException();  
        } else {  
            VarArg varArg = ArmVarArg.create(emulator, DalvikVM.this, dvmMethod);  
            DvmObject<?> ret = dvmMethod.callObjectMethod(dvmObject, varArg);  
            if (verbose) {  
                System.out.printf("JNIEnv->CallObjectMethod(%s, %s(%s) => %s) was called from %s%n", dvmObject, dvmMethod.methodName, varArg.formatArgs(), ret, context.getLRPointer());  
            }  
            return addLocalObject(ret);  
        }  
    }  
});
```
2. 拦截后调用DvmMethod类中的CallObjectMethod
```java
// 来自DvmMethod类
DvmObject<?>  callObjectMethod(DvmObject<?>  dvmObject, VarArg varArg) {  
    BaseVM vm = dvmClass.vm;  
    return checkJni(vm, dvmClass).callObjectMethod(vm, dvmObject, this, varArg);  
}
```
3. DvmMethod类中的CallObjectMethod调用AbstractJni中的CallObjectMethod
```java
// 来自AbstractJni类
@Override  
public DvmObject<?> callObjectMethod(BaseVM vm, DvmObject<?> dvmObject, String signature, VarArg varArg) {  
    switch (signature) {  
        case "java/lang/String->getBytes(Ljava/lang/String;)[B": {  
            StringObject string = (StringObject) dvmObject;  
            StringObject encoding = varArg.getObjectArg(0);  
            System.err.println("string=" + string.getValue() + ", encoding=" + encoding.getValue());  
            try {  
                return new ByteArray(vm, string.getValue().getBytes(encoding.value));  
            } catch (UnsupportedEncodingException e) {  
                throw new IllegalStateException(e);  
            }  
        }  
        case "android/content/Context->getPackageManager()Landroid/content/pm/PackageManager;":  
        case "android/app/Activity->getPackageManager()Landroid/content/pm/PackageManager;":  
            return vm.resolveClass("android/content/pm/PackageManager").newObject(null);  
        case "android/content/Context->getApplicationInfo()Landroid/content/pm/ApplicationInfo;":  
        case "android/app/Activity->getApplicationInfo()Landroid/content/pm/ApplicationInfo;":  
            return new ApplicationInfo(vm);  
        case "android/app/Application->getPackageName()Ljava/lang/String;":  
        case "android/content/ContextWrapper->getPackageName()Ljava/lang/String;":  
        case "android/app/Activity->getPackageName()Ljava/lang/String;":  
        case "android/content/Context->getPackageName()Ljava/lang/String;": {  
            String packageName = vm.getPackageName();  
            if (packageName != null) {  
                return new StringObject(vm, packageName);  
            }  
            break;  
        }  
        case "android/content/pm/PackageManager->getPackageInfo(Ljava/lang/String;I)Landroid/content/pm/PackageInfo;": {  
            StringObject packageName = varArg.getObjectArg(0);  
            int flags = varArg.getIntArg(1);  
            if (log.isDebugEnabled()) {  
                log.debug("getPackageInfo packageName={}, flags=0x{}", packageName.getValue(), Integer.toHexString(flags));  
            }  
            return new PackageInfo(vm, packageName.value, flags);  
        }  
        case "android/content/pm/Signature->toByteArray()[B": {  
            if (dvmObject instanceof Signature) {  
                Signature sig = (Signature) dvmObject;  
                return new ByteArray(vm, sig.toByteArray());  
            }  
            break;  
        }  
        case "android/content/pm/Signature->toCharsString()Ljava/lang/String;": {  
            if (dvmObject instanceof Signature) {  
                Signature sig = (Signature) dvmObject;  
                return new StringObject(vm, sig.toCharsString());  
            }  
            break;  
        }  
        case "java/lang/Class->getName()Ljava/lang/String;": {  
            DvmClass clazz = (DvmClass) dvmObject;  
            return new StringObject(vm, clazz.getName());  
        }  
        case "java/lang/String->getClass()Ljava/lang/Class;":  
        case "java/lang/Integer->getClass()Ljava/lang/Class;": {  
            return dvmObject.getObjectType();  
        }  
        case "java/lang/Class->getClassLoader()Ljava/lang/ClassLoader;":  
            return new ClassLoader(vm, signature);  
    }  
  
    throw new UnsupportedOperationException(signature);  
}
```
AbstractJni已经帮我们实现了一些常用java类方法的处理，如果AbstractJni没有处理则需要我们继承AbstractJni添加对应的方法处理。





## 学习脚本
```java
package com.myunidbg;  
  
import com.github.unidbg.AndroidEmulator;  
import com.github.unidbg.arm.backend.Unicorn2Factory;  
import com.github.unidbg.linux.android.AndroidEmulatorBuilder;  
import com.github.unidbg.linux.android.AndroidResolver;  
import com.github.unidbg.linux.android.dvm.*;  
import com.github.unidbg.linux.android.dvm.api.ApplicationInfo;  
import com.github.unidbg.memory.Memory;  
import com.github.unidbg.virtualmodule.android.AndroidModule;  
  
import java.io.File;  
  
public class NetWork extends AbstractJni {  
  
    private final AndroidEmulator emulator;  
    private final VM vm;  
    private final DvmClass NetCrypt;  
  
    public NetWork(){  
        emulator = AndroidEmulatorBuilder  
                .for32Bit()  
                .addBackendFactory(new Unicorn2Factory(true))  
                .setProcessName("cn.xiaochuankeji.tieba")  
                .build();  
        Memory memory = emulator.getMemory();  
        memory.setLibraryResolver(new AndroidResolver(23));  
        vm = emulator.createDalvikVM(new File("D:\\01ClassMP4\\unidbgfile\\q3\\right573.apk"));  
        vm.setJni(this);  
        vm.setVerbose(true);  
        new AndroidModule(emulator, vm).register(memory);  
        DalvikModule dm = vm.loadLibrary("net_crypto", true);  
        dm.callJNI_OnLoad(emulator);  
        NetCrypt = vm.resolveClass("com.izuiyou.network.NetCrypto");  
    }  
  
    public String callSign(){  
        String arg0 = "hello world";  
        byte[] arg1 = "V I 50".getBytes();  
        String ret = NetCrypt.callStaticJniMethodObject(emulator, "sign(Ljava/lang/String;[B)Ljava/lang/String;", arg0, arg1).getValue().toString();  
        return ret;  
    }  
  
    @Override  
    public DvmObject<?> callStaticObjectMethodV(BaseVM vm, DvmClass dvmClass, String signature, VaList vaList){  
        switch (signature){  
            case "com/izuiyou/common/base/BaseApplication->getAppContext()Landroid/content/Context;": {  
                DvmObject<?> context = vm.resolveClass("cn/xiaochuankeji/tieba/AppController").newObject(null);  
                return context;  
            }  
        }  
        return super.callStaticObjectMethodV(vm,  dvmClass, signature,vaList);  
    }  
  
    @Override  
    public DvmObject<?> callObjectMethodV(BaseVM vm, DvmObject<?> dvmObject, String signature, VaList vaList) {  
        switch (signature){  
            case "cn/xiaochuankeji/tieba/AppController->getPackageManager()Landroid/content/pm/PackageManager;": {  
                return vm.resolveClass("android/content/pm/PackageManager").newObject(null);  
            }  
            case "cn/xiaochuankeji/tieba/AppController->getPackageName()Ljava/lang/String;":{  
                String packageName = vm.getPackageName();  
                if (packageName != null) {  
                    return new StringObject(vm, packageName);  
                }  
                break;  
            }  
            case "cn/xiaochuankeji/tieba/AppController->getClass()Ljava/lang/Class;":{  
                return dvmObject.getObjectType();  
            }  
            case "java/lang/Class->getSimpleName()Ljava/lang/String;":{  
                String className = ((DvmClass) dvmObject).getClassName();  
                String[] name = className.split("/");  
                return new StringObject(vm, name[name.length - 1]);  
            }  
            case "cn/xiaochuankeji/tieba/AppController->getFilesDir()Ljava/io/File;":{  
                return vm.resolveClass("java/io/File").newObject("/data/data/cn.xiaochuankeji.tieba/files");  
            }  
            case "java/io/File->getAbsolutePath()Ljava/lang/String;":{  
                return new StringObject(vm, dvmObject.getValue().toString());  
            }  
        }  
        return super.callObjectMethodV(vm, dvmObject, signature, vaList);  
    }  
  
    @Override  
    public boolean callStaticBooleanMethodV(BaseVM vm, DvmClass dvmClass, String signature, VaList vaList) {  
        switch (signature){  
            case "android/os/Debug->isDebuggerConnected()Z":{  
                return false;  
            }  
        }  
        return super.callStaticBooleanMethodV(vm, dvmClass, signature, vaList);  
    }  
  
    @Override  
    public int callStaticIntMethodV(BaseVM vm, DvmClass dvmClass, String signature, VaList vaList) {  
        switch (signature){  
            case "android/os/Process->myPid()I":{  
                return emulator.getPid();  
            }  
        }  
        return super.callStaticIntMethodV(vm, dvmClass, signature, vaList);  
    }  
  
    public static void main(String[] args){  
        NetWork netWork = new NetWork();  
        String result = netWork.callSign();  
        System.out.println("sign result: " + result);  
    }  
}
```




# unidbg基本使用(方法调用参数)
## 方法调用参数处理
在unidbg基本使用一已经写了。这里引用原文章的总结：
- 基本类型直接传递
- 字符串、字节数组等基本对象直接传递，其内部会做封装，也可以自己调用`new StringObject(vm, str)`、`new ByteArray(vm, value)`等。
- JDK 标准库对象，如 HashMap、JSONObject 等，使用`ProxyDvmObject.createObject(vm, value)`处理。
- 非 JDK 标准库对象，如 Android Context、SharedPreference 等，使用`vm.resolveClass(vm，className).newObject(value)`处理。


## 源码分析unidbg jni类型创建
当我们调用callxxxx时最后都会调用DvmObject的callJniMethod方法
```java
protected static Number callJniMethod(Emulator<?> emulator, VM vm, DvmClass objectType, DvmObject<?> thisObj, String method, Object...args) {  
    UnidbgPointer fnPtr = objectType.findNativeFunction(emulator, method);  
    vm.addLocalObject(thisObj);  
    List<Object> list = new ArrayList<>(10);  
    list.add(vm.getJNIEnv());  
    list.add(thisObj.hashCode());  
    if (args != null) {  
        for (Object arg : args) {  
            if (arg instanceof Boolean) {  
                list.add((Boolean) arg ? VM.JNI_TRUE : VM.JNI_FALSE);  
                continue;  
            } else if(arg instanceof Hashable) {  
                list.add(arg.hashCode()); // dvm object  
  
                if(arg instanceof DvmObject) {  
                    vm.addLocalObject((DvmObject<?>) arg);  
                }  
                continue;  
            } else if (arg instanceof DvmAwareObject ||  
                    arg instanceof String ||  
                    arg instanceof byte[] ||  
                    arg instanceof short[] ||  
                    arg instanceof int[] ||  
                    arg instanceof float[] ||  
                    arg instanceof double[] ||  
                    arg instanceof Enum) {  
                DvmObject<?> obj = ProxyDvmObject.createObject(vm, arg);  
                list.add(obj.hashCode());  
                vm.addLocalObject(obj);  
                continue;  
            }  
  
            list.add(arg);  
        }  
    }  
    return Module.emulateFunction(emulator, fnPtr.peer, list.toArray());  
}
```
其中包含 Boolean 转为 01，添加局部引用，判断是否已经是`DvmObject`等一系列逻辑，再下面就是对常见类型的包装，如果用户传入 String、byteArray 等类型，会通过`ProxyDvmObject.createObject`自动装箱为 Unidbg JNI 中对应的类型。
跟进ProxyDvmObject类
```java
public class ProxyDvmObject extends DvmObject<Object> {  
  
    private static DvmClass getObjectType(VM vm, Class<?> clazz) {  
        Class<?> superClass = clazz.getSuperclass();  
        DvmClass[] interfaces = new DvmClass[clazz.getInterfaces().length + (superClass == null ? 0 : 1)];  
        int i = 0;  
        if (superClass != null) {  
            interfaces[i++] = getObjectType(vm, superClass);  
        }  
        for (Class<?> cc : clazz.getInterfaces()) {  
            interfaces[i++] = getObjectType(vm, cc);  
        }  
        return vm.resolveClass(clazz.getName().replace('.', '/'), interfaces);  
    }  
  
    /**  
     * mapping java object to dvm object     */    
    public static DvmObject<?> createObject(VM vm, Object value) {  
        if (value == null) {  
            return null;  
        }  
        if (value instanceof Class<?>) {  
            return getObjectType(vm, (Class<?>) value);  
        }  
        if (value instanceof DvmObject) {  
            return (DvmObject<?>) value;  
        }  
  
        if (value instanceof byte[]) {  
            return new ByteArray(vm, (byte[]) value);  
        }  
        if (value instanceof short[]) {  
            return new ShortArray(vm, (short[]) value);  
        }  
        if (value instanceof int[]) {  
            return new IntArray(vm, (int[]) value);  
        }  
        if (value instanceof float[]) {  
            return new FloatArray(vm, (float[]) value);  
        }  
        if (value instanceof double[]) {  
            return new DoubleArray(vm, (double[]) value);  
        }  
        if (value instanceof String) {  
            return new StringObject(vm, (String) value);  
        }  
        Class<?> clazz = value.getClass();  
        if (clazz.isArray()) {  
            if (clazz.getComponentType().isPrimitive()) {  
                throw new UnsupportedOperationException(String.valueOf(value));  
            }  
            Object[] array = (Object[]) value;  
            DvmObject<?>[] dvmArray = new DvmObject[array.length];  
            for (int i = 0; i < array.length; i++) {  
                dvmArray[i] = createObject(vm, array[i]);  
            }  
            return new ArrayObject(dvmArray);  
        }  
  
        return new ProxyDvmObject(vm, value);  
    }  
  
    private ProxyDvmObject(VM vm, Object value) {  
        super(getObjectType(vm, value.getClass()), value);  
    }  
  
}
```
事实上，`createObject`除了可以处理 string、array 这样的普通对象，也可以处理 map 或其他对象类型。来看`ProxyDvmObject` 对象的构造逻辑，它会对类的父类以及接口类做解析，嵌套式的`getObjectType`，最后`resolveClass`再`newObject`得到最终的`DvmObject`。


## VarArg,VarList,JValueList参数调用方式

### 三种调用方式对比

| 特性              | **`A` 后缀 (Array)**                   | **`V` 后缀 (VaList)**       | **无后缀 (VarArg)**         |
| --------------- | ------------------------------------ | ------------------------- | ------------------------ |
| **参数形式**        | `const jvalue *args` (数组)            | `va_list args` (可变参数列表)   | `...` (C语言可变参数)          |
| **访问方式**        | **随机访问**：通过索引访问数组元素                  | **顺序访问**：通过迭代器依次获取        | **随机访问**：通过索引直接获取        |
| **unidbg 中的容器** | JValueList类，继承于VarList，内部处理与jvalue数组 | **`VarList`** 类           | **`VarArg`** 类           |
| **典型函数名**       | `CallStaticObjectMethodA`            | `CallStaticObjectMethodV` | `CallStaticObjectMethod` |
unidbg 在模拟这些 JNI 函数时，采用了不同的策略：

- **对于 `A` 后缀函数**：unidbg 的 DalvikVM 内部会识别函数类型，并期望 native 代码传入一个指向 `jvalue` 数组的指针。它会从该内存地址开始，按照方法签名依次读取参数值。这与 `V` 后缀函数形成对比，后者在 unidbg 中对应的是 `VarList` 类。
    
- **对于 `V` 后缀函数**：unidbg 使用 **`VarList`** 类来封装 `va_list`，提供了 `getIntArg(int)`、`getObjectArg(int)` 等方法来顺序或按索引提取参数。
    
- **对于无后缀函数**：unidbg 使用 **`VarArg`** 类，它提供了 `getObjectArg(int)` 等方法，可以直接按索引获取参数，这在补环境时非常方便。



## 学习脚本
NativeLibrary类
```java
package com.myunidbg;  
  
import com.github.unidbg.AndroidEmulator;  
import com.github.unidbg.arm.backend.Unicorn2Factory;  
import com.github.unidbg.linux.android.AndroidEmulatorBuilder;  
import com.github.unidbg.linux.android.AndroidResolver;  
import com.github.unidbg.linux.android.dvm.*;  
import com.github.unidbg.linux.android.dvm.jni.ProxyDvmObject;  
import com.github.unidbg.memory.Memory;  
  
import java.io.File;  
import java.util.Map;  
import java.util.TreeMap;  
  
public class NativeLibrary extends AbstractJni {  
    private final AndroidEmulator emulator;  
    private VM vm;  
    private DvmClass LibBili;  
  
    public NativeLibrary(){  
        emulator = AndroidEmulatorBuilder  
                .for32Bit()  
                .addBackendFactory(new Unicorn2Factory(true))  
                .setProcessName("tv.danmaku.bili")  
                .build();  
        Memory memory = emulator.getMemory();  
        memory.setLibraryResolver(new AndroidResolver(23));  
        vm = emulator.createDalvikVM(new File("D:\\01ClassMP4\\unidbgfile\\q5\\bilibili.apk"));  
        vm.setJni(this);  
        vm.setVerbose(true);  
        DalvikModule dm = vm.loadLibrary("bili", true);  
        dm.callJNI_OnLoad(emulator);  
        LibBili = vm.resolveClass("com.bilibili.nativelibrary.LibBili");  
    }  
  
    public String callS(){  
        TreeMap<String, String> map = new TreeMap<>();  
        map.put("build", "6180500");  
        map.put("mobi_app", "android");  
        map.put("channel", "shenma069");  
        map.put("appkey", "1d8b6e7d45233436");  
        map.put("s_locale", "zh_CN");  
        DvmObject<?> mapObject = ProxyDvmObject.createObject(vm, map);  
        String ret = LibBili.callStaticJniMethodObject(emulator, "s(Ljava/util/SortedMap;)Lcom/bilibili/nativelibrary/SignedQuery;", mapObject).getValue().toString();  
        return ret;  
    }  
  
    @Override  
    public boolean callBooleanMethod(BaseVM vm, DvmObject<?> dvmObject, String signature, VarArg varArg) {  
        switch (signature){  
            case "java/util/Map->isEmpty()Z":{  
                Map map = (Map)dvmObject.getValue();  
                return map.isEmpty();  
            }  
  
        }  
        return super.callBooleanMethod(vm, dvmObject, signature, varArg);  
    }  
  
    @Override  
    public DvmObject<?> callObjectMethod(BaseVM vm, DvmObject<?> dvmObject, String signature, VarArg varArg) {  
        switch (signature){  
            case "java/util/Map->get(Ljava/lang/Object;)Ljava/lang/Object;":{  
                Map map = (Map)dvmObject.getValue();  
                Object key = varArg.getObjectArg(0).getValue();  
                return ProxyDvmObject.createObject(vm, map.get(key));  
            }  
            case "java/util/Map->put(Ljava/lang/Object;Ljava/lang/Object;)Ljava/lang/Object;":{  
                Map map = (Map)dvmObject.getValue();  
                Object arg0 = varArg.getObjectArg(0).getValue();  
                Object arg1 = varArg.getObjectArg(1).getValue();  
                return ProxyDvmObject.createObject(vm, map.put(arg0, arg1));  
            }  
        }  
        return super.callObjectMethod(vm, dvmObject, signature, varArg);  
    }  
  
    @Override  
    public DvmObject<?> callStaticObjectMethod(BaseVM vm, DvmClass dvmClass, String signature, VarArg varArg) {  
        switch (signature){  
            case "com/bilibili/nativelibrary/SignedQuery->r(Ljava/util/Map;)Ljava/lang/String;":{  
                Map map = (Map)varArg.getObjectArg(0).getValue();  
                return new StringObject(vm, SignedQuery.r(map));  
            }  
        }  
        return super.callStaticObjectMethod(vm, dvmClass, signature, varArg);  
    }  
  
    @Override  
    public DvmObject<?> newObject(BaseVM vm, DvmClass dvmClass, String signature, VarArg varArg) {  
        switch (signature){  
            case "com/bilibili/nativelibrary/SignedQuery-><init>(Ljava/lang/String;Ljava/lang/String;)V":{  
                String arg1 = (String) varArg.getObjectArg(0).getValue();  
                String arg2 = (String) varArg.getObjectArg(1).getValue();  
                SignedQuery object = new SignedQuery(arg1, arg2);  
                return vm.resolveClass("com/bilibili/nativelibrary/SignedQuery").newObject(object);  
            }  
        }  
        return super.newObject(vm, dvmClass, signature, varArg);  
    }  
  
    public static void main(String[] args){  
        NativeLibrary nativeLibrary = new NativeLibrary();  
        String result = nativeLibrary.callS();  
        System.out.println("result: " + result);  
    }  
}
```
SignedQuery类
```java
package com.myunidbg;  
  
import java.io.UnsupportedEncodingException;  
import java.util.Map;  
import java.util.SortedMap;  
import java.util.TreeMap;  
  
/* JADX INFO: compiled from: BL */  
/* JADX INFO: loaded from: classes.dex */  
public final class SignedQuery {  
  
    /* JADX INFO: renamed from: c, reason: collision with root package name */  
    private static final char[] f14881c = "0123456789ABCDEF".toCharArray();  
    public final String a;  
    public final String b;  
  
    public SignedQuery(String str, String str2) {  
        this.a = str;  
        this.b = str2;  
    }  
  
    private static boolean a(char c2, String str) {  
        return (c2 >= 'A' && c2 <= 'Z') || (c2 >= 'a' && c2 <= 'z') || !((c2 < '0' || c2 > '9') && "-_.~".indexOf(c2) == -1 && (str == null || str.indexOf(c2) == -1));  
    }  
  
    static String b(String str) {  
        return c(str, null);  
    }  
  
    static String c(String str, String str2) {  
        StringBuilder sb = null;  
        if (str == null) {  
            return null;  
        }  
        int length = str.length();  
        int i2 = 0;  
        while (i2 < length) {  
            int i3 = i2;  
            while (i3 < length && a(str.charAt(i3), str2)) {  
                i3++;  
            }  
            if (i3 == length) {  
                if (i2 == 0) {  
                    return str;  
                }  
                sb.append((CharSequence) str, i2, length);  
                return sb.toString();  
            }  
            if (sb == null) {  
                sb = new StringBuilder();  
            }  
            if (i3 > i2) {  
                sb.append((CharSequence) str, i2, i3);  
            }  
            i2 = i3 + 1;  
            while (i2 < length && !a(str.charAt(i2), str2)) {  
                i2++;  
            }  
            try {  
                byte[] bytes = str.substring(i3, i2).getBytes("UTF-8");  
                int length2 = bytes.length;  
                for (int i4 = 0; i4 < length2; i4++) {  
                    sb.append('%');  
                    sb.append(f14881c[(bytes[i4] & 240) >> 4]);  
                    sb.append(f14881c[bytes[i4] & 15]);  
                }  
            } catch (UnsupportedEncodingException e) {  
                throw new AssertionError(e);  
            }  
        }  
        return sb == null ? str : sb.toString();  
    }  
  
    static String r(Map<String, String> map) {  
        if (!(map instanceof SortedMap)) {  
            map = new TreeMap(map);  
        }  
        StringBuilder sb = new StringBuilder(256);  
        for (Map.Entry<String, String> entry : map.entrySet()) {  
            String key = entry.getKey();  
            if (!key.isEmpty()) {  
                sb.append(b(key));  
                sb.append("=");  
                String value = entry.getValue();  
                sb.append(value == null ? "" : b(value));  
                sb.append("&");  
            }  
        }  
        int length = sb.length();  
        if (length > 0) {  
            sb.deleteCharAt(length - 1);  
        }  
        if (length == 0) {  
            return null;  
        }  
        return sb.toString();  
    }  
  
    public String toString() {  
        String str = this.a;  
        if (str == null) {  
            return "";  
        }  
        if (this.b == null) {  
            return str;  
        }  
        return this.a + "&sign=" + this.b;  
    }  
}
```




# unidbg基本使用(数据类型处理)
主要是对数据类型处理做了更详细的说明
[Unidbg 的基本使用（五）](https://www.yuque.com/lilac-2hqvv/xdwlsg/hiiaukxvr2zuz5mb?#%20%E3%80%8AUnidbg%20%E7%9A%84%E5%9F%BA%E6%9C%AC%E4%BD%BF%E7%94%A8%EF%BC%88%E4%BA%94%EF%BC%89%E3%80%8B)


# Unidbg基本使用(实践)
[unidbg基本使用六](https://www.yuque.com/lilac-2hqvv/xdwlsg/avlc01hrko6yvr26?#%20%E3%80%8AUnidbg%20%E7%9A%84%E5%9F%BA%E6%9C%AC%E4%BD%BF%E7%94%A8%EF%BC%88%E5%85%AD%EF%BC%89%E3%80%8B  ) 

## 学习脚本
```java
package com.myunidbg;  
  
import com.github.unidbg.AndroidEmulator;  
import com.github.unidbg.arm.backend.Unicorn2Factory;  
import com.github.unidbg.linux.android.AndroidEmulatorBuilder;  
import com.github.unidbg.linux.android.AndroidResolver;  
import com.github.unidbg.linux.android.dvm.*;  
import com.github.unidbg.linux.android.dvm.api.SystemService;  
import com.github.unidbg.linux.android.dvm.jni.ProxyDvmObject;  
import com.github.unidbg.memory.Memory;  
import com.github.unidbg.virtualmodule.android.AndroidModule;  
import com.github.unidbg.virtualmodule.android.JniGraphics;  
  
import java.io.*;  
import java.text.SimpleDateFormat;  
import java.util.Date;  
import java.util.Locale;  
  
  
public class NBridge extends AbstractJni {  
    private final AndroidEmulator emulator;  
    private final VM vm;  
    private final DvmObject<?> SIUACollector;  
    private FileInputStream fileInputStream;  
    private InputStreamReader inputStreamReader;  
    private BufferedReader bufferedReader;  
    private File file;  
    private SimpleDateFormat simpleDataFormat;  
    private Date date;  
  
    public NBridge(){  
        emulator = AndroidEmulatorBuilder  
                .for32Bit()  
                .addBackendFactory(new Unicorn2Factory(true))  
                .setProcessName("com.dazhongdianping")  
                .build();  
        Memory memory = emulator.getMemory();  
        memory.setLibraryResolver(new AndroidResolver(23));  
  
        vm = emulator.createDalvikVM(new File("D:\\01ClassMP4\\unidbgfile\\dazhongdianping\\dazhongdianping.apk"));  
        vm.setJni(this);  
        vm.setVerbose(true);  
  
        new AndroidModule(emulator, vm).register(memory);  
        new JniGraphics(emulator, vm).register(memory);  
  
        DalvikModule dm = vm.loadLibrary("mtguard", true);  
        dm.callJNI_OnLoad(emulator);  
        SIUACollector = vm.resolveClass("com/meituan/android/common/mtguard/NBridge$SIUACollector").newObject(null);  
    }  
  
    @Override  
    public boolean callStaticBooleanMethodV(BaseVM vm, DvmClass dvmClass, String signature, VaList vaList) {  
        switch (signature){  
            case "android/text/TextUtils->isEmpty(Ljava/lang/CharSequence;)Z":{  
                String str = vaList.getObjectArg(0).getValue().toString();  
                if (str == null || str.length() == 0){  
                    return true;  
                }  
                else{  
                    return false;  
                }  
            }  
        }  
        return super.callStaticBooleanMethodV(vm, dvmClass, signature, vaList);  
    }  
  
  
    @Override  
    public void callVoidMethodV(BaseVM vm, DvmObject<?> dvmObject, String signature, VaList vaList) {  
        switch (signature){  
            case "java/lang/StringBuilder-><init>()V":  
            case "com/meituan/android/common/mtguard/NBridge$SIUACollector->safeClose(Ljava/io/Closeable;)V": {  
                return;  
            }  
            case "java/io/FileInputStream-><init>(Ljava/lang/String;)V":{  
                String str = vaList.getObjectArg(0).getValue().toString();  
                System.out.println("FileInputStream arg: " + str); // /proc/cpuinfo  
                try {  
                    if (str.equals("/proc/cpuinfo")){  
                        str = "D:\\00wangan\\02dump\\cpuinfo";  
                    }  
                    fileInputStream = new FileInputStream(new File(str));  
                } catch (FileNotFoundException e) {  
                    throw new RuntimeException(e);  
                }  
                return;  
            }  
            case "java/io/InputStreamReader-><init>(Ljava/io/InputStream;)V":{  
                inputStreamReader = new InputStreamReader(fileInputStream);  
                return;  
            }  
            case "java/io/BufferedReader-><init>(Ljava/io/Reader;)V":{  
                bufferedReader = new BufferedReader(inputStreamReader);  
                return;  
            }  
            case "java/io/BufferedReader->close()V":{  
                try {  
                    bufferedReader.close();  
                } catch (IOException e) {  
                    throw new RuntimeException(e);  
                }  
                return;  
            }  
            case "java/io/File-><init>(Ljava/lang/String;)V":{  
                String str = vaList.getObjectArg(0).getValue().toString();  
                System.out.println("File(): " + str);  
                if (str.equals("/sys/class/power_supply/battery/voltage_now")){  
                    file = new File("D:\\00wangan\\02dump\\voltage_now.txt");  
                }  
                return;  
            }  
            case "java/text/SimpleDateFormat-><init>(Ljava/lang/String;Ljava/util/Locale;)V":{  
                String arg0 = vaList.getObjectArg(0).getValue().toString();  
                Locale locale = (Locale) vaList.getObjectArg(1).getValue();  
                System.out.println("SimpleDataFormat arg0:" + arg0);  
                System.out.println("SimpleDataFormat arg1:" + locale.toString());  
                if (arg0.equals("yyyy-MM-dd HH:mm:ss:sss")){  
                    simpleDataFormat = new SimpleDateFormat(arg0, locale);  
                }  
                return;  
            }  
            case "java/util/Date-><init>()V":{  
                date = new Date();  
                return;  
            }  
        }  
        super.callVoidMethodV(vm, dvmObject, signature, vaList);  
    }  
  
    @Override  
    public DvmObject<?> callObjectMethodV(BaseVM vm, DvmObject<?> dvmObject, String signature, VaList vaList) {  
        switch (signature){  
            case "com/meituan/android/common/mtguard/NBridge$SIUACollector->getEnvironmentInfo()Ljava/lang/String;":{  
                return new StringObject(vm, "0|0|0|-|0|");  
            }  
            case "java/lang/StringBuilder->append(Ljava/lang/String;)Ljava/lang/StringBuilder;":{  
                StringBuilder stringBuilder = (StringBuilder) dvmObject.getValue();  
                stringBuilder.append(vaList.getObjectArg(0).getValue().toString());  
                return ProxyDvmObject.createObject(vm, stringBuilder);  
            }  
            case "com/meituan/android/common/mtguard/NBridge$SIUACollector->isVPN()Ljava/lang/String;":  
            case "com/meituan/android/common/mtguard/NBridge$SIUACollector->systemVolume(Landroid/content/Context;)Ljava/lang/String;":  
            case "com/meituan/android/common/mtguard/NBridge$SIUACollector->getDataActivity(Landroid/content/Context;)Ljava/lang/String;": {  
                return new StringObject(vm, "0");  
            }  
            case "com/meituan/android/common/mtguard/NBridge$SIUACollector->brightness(Landroid/content/Context;)Ljava/lang/String;":{  
                return new StringObject(vm, "0.8");  
            }  
            case "java/lang/StringBuilder->toString()Ljava/lang/String;":{  
                StringBuilder stringBuilder = (StringBuilder) dvmObject.getValue();  
                String result = stringBuilder.toString();  
                return new StringObject(vm, result);  
            }  
            case "android/content/Context->getApplicationContext()Landroid/content/Context;":{  
                return vm.resolveClass("android/content/Context").newObject(null);  
            }  
            case "android/content/Context->getSystemService(Ljava/lang/String;)Ljava/lang/Object;":{  
                StringObject serviceName = vaList.getObjectArg(0);  
                assert serviceName != null;  
                return new SystemService(vm, serviceName.getValue());  
            }  
            case "com/meituan/android/common/mtguard/NBridge$SIUACollector->checkBuildAttribute(Ljava/lang/String;)Ljava/lang/String;":{  
                String arg0 = vaList.getObjectArg(0).getValue().toString();  
                System.out.println("checkBuildAttribute arg0: " + arg0);  
                return new StringObject(vm, arg0);  
            }  
            case "android/view/WindowManager->getDefaultDisplay()Landroid/view/Display;":{  
                return vm.resolveClass("android/view/Display").newObject(null);  
            }  
            case "java/lang/StringBuilder->append(I)Ljava/lang/StringBuilder;":{  
                return ProxyDvmObject.createObject(vm, ((StringBuilder)dvmObject.getValue()).append(vaList.getIntArg(0)));  
            }  
            case "java/lang/StringBuilder->append(C)Ljava/lang/StringBuilder;":{  
                return ProxyDvmObject.createObject(vm, ((StringBuilder)dvmObject.getValue()).append((char)vaList.getIntArg(0)));  
            }  
            case "com/meituan/android/common/mtguard/NBridge$SIUACollector->getTotalInternalMemorySize()Ljava/lang/String;":  
            case "com/meituan/android/common/mtguard/NBridge$SIUACollector->getTotalExternalMemorySize()Ljava/lang/String;": {  
                return new StringObject(vm, "110GB");  
            }  
            case "android/telephony/TelephonyManager->getSimOperator()Ljava/lang/String;":{  
                return new StringObject(vm, "46001");  
            }  
            case "com/meituan/android/common/mtguard/NBridge$SIUACollector->getAccessSubType()Ljava/lang/String;":{  
                return new StringObject(vm, "4G");  
            }  
            case "com/meituan/android/common/mtguard/NBridge$SIUACollector->getCpuInfoType()Ljava/lang/String;":{  
                return new StringObject(vm, "arm");  
            }  
            case "java/io/BufferedReader->readLine()Ljava/lang/String;":{  
                try {  
                    String result = bufferedReader.readLine();  
                    if (result != null){  
                        return new StringObject(vm, bufferedReader.readLine());  
                    }  
                    else{  
                        return null;  
                    }  
                } catch (IOException e) {  
                    throw new RuntimeException(e);  
                }  
            }  
            case "android/hardware/SensorManager->getDefaultSensor(I)Landroid/hardware/Sensor;":{  
                int type = vaList.getIntArg(0);  
                System.out.println("sensor type: " + type);  
                return vm.resolveClass("android/hardware/Sensor").newObject(type);  
            }  
            case "android/hardware/Sensor->getName()Ljava/lang/String;":{  
                int type = (int)dvmObject.getValue();  
                System.out.println("Sensor.getName arg: " + type);  
                if (type == 1){  
                    return new StringObject(vm, "ICM20690");  
                }  
                else if (type == 9){  
                    return new StringObject(vm , "Non-wakeup");  
                }  
                else{  
                    throw new UnsupportedOperationException(signature);  
                }  
            }  
            case "android/hardware/Sensor->getVendor()Ljava/lang/String;":{  
                int type = (int) dvmObject.getValue();  
                System.out.println("Sensor.getVendor arg: " + type);  
                if (type == 1){  
                    return new StringObject(vm, "qualcomm");  
                }  
                else if (type == 9){  
                    return new StringObject(vm, "qualcomm");  
                }  
                else{  
                    throw new UnsupportedOperationException(signature);  
                }  
            }  
            case "com/meituan/android/common/mtguard/NBridge$SIUACollector->getSysProp(Ljava/lang/String;)Ljava/lang/String;":{  
                String str = vaList.getObjectArg(0).getValue().toString();  
                System.out.println("getSysProp arg: " + str);  
                if (str.equals("ro.product.cpu.abi")){  
                    return new StringObject(vm, "arm64-v8a");  
                }  
                else if (str.equals("ro.product.cpu.abi2")){  
                    return new StringObject(vm, "");  
                }  
                else if (str.equals("ro.build.product")){  
                    return new StringObject(vm, "polaris");  
                }  
                else if (str.equals("ro.build.description")){  
                    return new StringObject(vm, "polaris-user 10 QKQ1.190828.002 V12.0.2.0.QDGCNXM release-keys");  
                }  
                else if(str.equals("ro.secure")){  
                    return new StringObject(vm, "1");  
                }  
                else if(str.equals("ro.debuggable")){  
                    return new StringObject(vm, "0");  
                }  
                else if(str.equals("persist.sys.usb.config")){  
                    return new StringObject(vm, "");  
                }  
                else if(str.equals("sys.usb.config")){  
                    return new StringObject(vm, "");  
                }  
                else if(str.equals("sys.usb.state")){  
                    return new StringObject(vm, "");  
                }  
                else if(str.equals("gsm.version.baseband")){  
                    return new StringObject(vm, "4.0.c2.6-00335-0914_2350_3c8fca6,4.0.c2.6-00335-0914_2350_3c8fca6");  
                }  
                if(str.equals("gsm.version.ril-impl")){  
                    return new StringObject(vm, "Qualcomm RIL 1.0");  
                }  
                if(str.equals("gsm.sim.state")){  
                    return new StringObject(vm, "ABSENT,ABSENT");  
                }  
                if(str.equals("gsm.sim.state.2")){  
                    return new StringObject(vm, "");  
                }  
                if(str.equals("wifi.interface")){  
                    return new StringObject(vm, "wlan0");  
                }  
                else{  
                    throw new UnsupportedOperationException(signature);  
                }  
            }  
            case "android/content/Context->getResources()Landroid/content/res/Resources;":{  
                return vm.resolveClass("android/content/res/Resources").newObject(null);  
            }  
            case "android/content/res/Resources->getConfiguration()Landroid/content/res/Configuration;":{  
                return vm.resolveClass("android/content/res/Configuration").newObject(null);  
            }  
            case "android/net/wifi/WifiManager->getConnectionInfo()Landroid/net/wifi/WifiInfo;":{  
                return vm.resolveClass("android/net/wifi/WifiInfo").newObject(null);  
            }  
            case "android/net/wifi/WifiInfo->getSSID()Ljava/lang/String;":{  
                return new StringObject(vm, "Redmi_7B25");  
            }  
            case "android/telephony/TelephonyManager->getNetworkOperator()Ljava/lang/String;":{  
                return new StringObject(vm, "");  
            }  
            case "java/text/SimpleDateFormat->format(Ljava/util/Date;)Ljava/lang/String;":{  
                return new StringObject(vm, simpleDataFormat.format(date));  
            }  
            case "com/meituan/android/common/mtguard/NBridge$SIUACollector->appCache(Landroid/content/Context;)Ljava/lang/String;":{  
                return new StringObject(vm, "149045277");  
            }  
            case "com/meituan/android/common/mtguard/NBridge$SIUACollector->availableSystem()Ljava/lang/String;":{  
                return new StringObject(vm, "unknown");  
            }  
            case "com/meituan/android/common/mtguard/NBridge$SIUACollector->totalMemory()Ljava/lang/String;":{  
                return new StringObject(vm, "5905514496");  
            }  
            case "com/meituan/android/common/mtguard/NBridge$SIUACollector->getFirstLaunchTime(Landroid/content/Context;)Ljava/lang/String;":{  
                return new StringObject(vm, "1665747253983");  
            }  
        }  
        return super.callObjectMethodV(vm, dvmObject, signature, vaList);  
    }  
  
  
    @Override  
    public boolean callBooleanMethodV(BaseVM vm, DvmObject<?> dvmObject, String signature, VaList vaList) {  
        switch (signature){  
            case "com/meituan/android/common/mtguard/NBridge$SIUACollector->isAccessibilityEnable()Z":{  
                return false;  
            }  
            case "com/meituan/android/common/mtguard/NBridge$SIUACollector->isPermissionGranted(Ljava/lang/String;Landroid/content/Context;)Z":{  
                String permissionName = vaList.getObjectArg(0).getValue().toString();  
                System.out.println("check permission:"+permissionName);  
                if (permissionName.equals("android.permission.READ_PHONE_STATE")) {  
                    return true;  
                }  
                else if (permissionName.equals("android.permission.ACCESS_WIFI_STATE")){  
                    return true;  
                }  
                else{  
                    throw new UnsupportedOperationException(signature);  
                }  
            }  
            case "android/content/pm/PackageManager->hasSystemFeature(Ljava/lang/String;)Z":{  
                System.out.println("systemFeature: " + vaList.getObjectArg(0).getValue().toString());  
                return true;  
            }  
            case "java/io/File->exists()Z":  
            case "com/meituan/android/common/mtguard/NBridge$SIUACollector->getBatteryInfo()Z": {  
                return true;  
            }  
            case "java/lang/String->equalsIgnoreCase(Ljava/lang/String;)Z":{  
                boolean bool = dvmObject.getValue().toString().equalsIgnoreCase(vaList.getObjectArg(0).getValue().toString());  
                return bool;  
            }  
        }  
        return super.callBooleanMethodV(vm, dvmObject, signature, vaList);  
    }  
  
    @Override  
    public int callIntMethodV(BaseVM vm, DvmObject<?> dvmObject, String signature, VaList vaList) {  
        switch (signature){  
            case "com/meituan/android/common/mtguard/NBridge$SIUACollector->uiAutomatorClickCount()I":{  
                return 0;  
            }  
            case "android/view/Display->getHeight()I":{  
                return 2160;  
            }  
            case "android/view/Display->getWidth()I":{  
                return 1080;  
            }  
            case "java/lang/String->compareToIgnoreCase(Ljava/lang/String;)I":{  
                String strObj = dvmObject.getValue().toString();  
                return strObj.compareToIgnoreCase(vaList.getObjectArg(0).getValue().toString());  
            }  
            case "com/meituan/android/common/mtguard/NBridge$SIUACollector->boolean2Integer(Z)I":{  
                return vaList.getIntArg(0);  
            }  
            case "com/meituan/android/common/mtguard/NBridge$SIUACollector->androidAppCnt(Landroid/content/Context;)I":{  
                return 519;  
            }  
        }  
        return super.callIntMethodV(vm, dvmObject, signature, vaList);  
    }  
  
    @Override  
    public DvmObject<?> callStaticObjectMethodV(BaseVM vm, DvmClass dvmClass, String signature, VaList vaList) {  
        switch (signature){  
            case "java/lang/String->valueOf(I)Ljava/lang/String;":{  
                String result = String.valueOf(vaList.getIntArg(0));  
                return new StringObject(vm, result);  
            }  
            case "java/lang/Integer->toString(I)Ljava/lang/String;":{  
                int num = vaList.getIntArg(0);  
                return new StringObject(vm, Integer.toString(num));  
            }  
        }  
        return super.callStaticObjectMethodV(vm, dvmClass, signature, vaList);  
    }  
  
  
    @Override  
    public int getStaticIntField(BaseVM vm, DvmClass dvmClass, String signature) {  
        switch (signature){  
            case "android/os/Build$VERSION->SDK_INT:I":{  
                return 29;  
            }  
        }  
        return super.getStaticIntField(vm, dvmClass, signature);  
    }  
  
    @Override  
    public DvmObject<?> getStaticObjectField(BaseVM vm, DvmClass dvmClass, String signature) {  
        switch (signature){  
            case "android/os/Build->BOARD:Ljava/lang/String;":{  
                return new StringObject(vm, "sdm845");  
            }  
            case "android/os/Build->MANUFACTURER:Ljava/lang/String;":  
            case "android/os/Build->BRAND:Ljava/lang/String;": {  
                return new StringObject(vm, "Xiaomi");  
            }  
            case "android/os/Build->MODEL:Ljava/lang/String;":{  
                return new StringObject(vm, "MIX 2S");  
            }  
            case "android/os/Build->PRODUCT:Ljava/lang/String;":  
            case "android/os/Build->DEVICE:Ljava/lang/String;": {  
                return new StringObject(vm, "polaris");  
            }  
            case "android/os/Build->HARDWARE:Ljava/lang/String;":{  
                return new StringObject(vm, "qcom");  
            }  
            case "android/os/Build->HOST:Ljava/lang/String;":{  
                return new StringObject(vm, "c3-miui-ota-bd134.bj");  
            }  
            case "android/os/Build->ID:Ljava/lang/String;":{  
                return new StringObject(vm, "QKQ1.190828.002");  
            }  
            case "android/os/Build$VERSION->RELEASE:Ljava/lang/String;":{  
                return new StringObject(vm, "10");  
            }  
            case "android/os/Build->TAGS:Ljava/lang/String;":{  
                return new StringObject(vm, "release-keys");  
            }  
            case "android/os/Build->FINGERPRINT:Ljava/lang/String;":{  
                return new StringObject(vm, "Xiaomi/polaris/polaris:10/QKQ1.190828.002/V12.0.2.0.QDGCNXM:user/release-keys");  
            }  
            case "android/os/Build->TYPE:Ljava/lang/String;":{  
                return new StringObject(vm, "user");  
            }  
            case "android/os/Build$VERSION->SDK:Ljava/lang/String;":{  
                return new StringObject(vm, "29");  
            }  
        }  
        return super.getStaticObjectField(vm, dvmClass, signature);  
    }  
  
    @Override  
    public DvmObject<?> getObjectField(BaseVM vm, DvmObject<?> dvmObject, String signature) {  
        switch (signature){  
            case "com/meituan/android/common/mtguard/NBridge$SIUACollector->mContext:Landroid/content/Context;":{  
                return vm.resolveClass("android/content/Context").newObject(null);  
            }  
            case "android/content/res/Configuration->locale:Ljava/util/Locale;":{  
                Locale locale = new Locale("zh_CN");  
                return ProxyDvmObject.createObject(vm, locale);  
            }  
            case "com/meituan/android/common/mtguard/NBridge$SIUACollector->batteryHelper:Lcom/meituan/android/common/dfingerprint/collection/utils/BatteryHelper;":{  
                return vm.resolveClass("com/meituan/android/common/dfingerprint/collection/utils/BatteryHelper").newObject(null);  
            }  
        }  
        return super.getObjectField(vm, dvmObject, signature);  
    }  
  
    @Override  
    public int getIntField(BaseVM vm, DvmObject<?> dvmObject, String signature) {  
        switch (signature){  
            // level 电量剩余多少  
            case "com/meituan/android/common/mtguard/NBridge$SIUACollector->level:I":  
                return 3;  
            // scale 电量精度  
            case "com/meituan/android/common/mtguard/NBridge$SIUACollector->scale:I":  
                return 100;  
            // status 电池状态，未知、充电、放电、未充电、满电;  
            case "com/meituan/android/common/mtguard/NBridge$SIUACollector->status:I":  
                return 0;  
        }  
        return super.getIntField(vm, dvmObject, signature);  
    }  
  
    @Override  
    public boolean getBooleanField(BaseVM vm, DvmObject<?> dvmObject, String signature) {  
        switch (signature){  
            case "com/meituan/android/common/mtguard/NBridge$SIUACollector->plugged:Z":{  
                return false;  
            }  
        }  
        return super.getBooleanField(vm, dvmObject, signature);  
    }  
  
    @Override  
    public DvmObject<?> allocObject(BaseVM vm, DvmClass dvmClass, String signature) {  
        switch (signature){  
            case "java/lang/StringBuilder->allocObject":{  
                return dvmClass.newObject(new StringBuilder());  
            }  
            case "java/io/BufferedReader->allocObject":  
            case "java/io/InputStreamReader->allocObject":  
            case "java/io/FileInputStream->allocObject":  
            case "java/text/SimpleDateFormat->allocObject":  
            case "java/util/Date->allocObject":  
            case "java/io/File->allocObject": {  
                return dvmClass.newObject(null);  
            }  
        }  
        return super.allocObject(vm, dvmClass, signature);  
    }  
  
  
    public String getHWEquipmentInfo(){  
        String result = SIUACollector.callJniMethodObject(emulator, "getHWEquipmentInfo()Ljava/lang/String;").getValue().toString();  
        return result;  
    }  
  
  
    private String getEnvironmentInfo(){  
        String result = SIUACollector.callJniMethodObject(emulator, "getEnvironmentInfo()Ljava/lang/String;").getValue().toString();  
        return result;  
    }  
  
    private String getEnvironmentInfoExtra(){  
        String result = SIUACollector.callJniMethodObject(emulator, "getEnvironmentInfoExtra()Ljava/lang/String;").getValue().toString();  
        return result;  
    }  
  
    private String getExternalEquipmentInfo(){  
        String result = SIUACollector.callJniMethodObject(emulator, "getExternalEquipmentInfo()Ljava/lang/String;").getValue().toString();  
        return result;  
    }  
  
    public String getHWProperty(){  
        String result = SIUACollector.callJniMethodObject(emulator, "getHWProperty()Ljava/lang/String;").getValue().toString();  
        return result;  
    }  
  
    public String getHWState(){  
        String result = SIUACollector.callJniMethodObject(emulator, "getHWStatus()Ljava/lang/String;").getValue().toString();  
        return result;  
    }  
  
    public String getLocationInfo(){  
        String result = SIUACollector.callJniMethodObject(emulator, "getLocationInfo()Ljava/lang/String;").getValue().toString();  
        return result;  
    }  
  
    public String getPlatformInfo(){  
        String result = SIUACollector.callJniMethodObject(emulator,"getPlatformInfo()Ljava/lang/String;").getValue().toString();  
        return result;  
    }  
  
    public String getUserAction(){  
        String result = SIUACollector.callJniMethodObject(emulator, "getUserAction()Ljava/lang/String;").getValue().toString();  
        return result;  
    }  
  
  
    public static void main(String[] args){  
        NBridge nBridge = new NBridge();  
        //System.out.println("getEnvironmentInfo: " + nBridge.getEnvironmentInfo());  
        //System.out.println("getEnvironmentInfoExtra: " + nBridge.getEnvironmentInfoExtra());        //System.out.println("getExternalEquipmentInfo: " + nBridge.getExternalEquipmentInfo());        //System.out.println("getHWEquipmentInfo: " + nBridge.getHWEquipmentInfo());        //System.out.println("getHWProperty(): " + nBridge.getHWProperty());        //System.out.println("getHWState(): " + nBridge.getHWState());        //System.out.println("getLocationInfo: " + nBridge.getLocationInfo());        //System.out.println("getPlatformInfo: " + nBridge.getPlatformInfo());        System.out.println("getUserAction: " + nBridge.getUserAction());  
    }  
  
}
```




# unidbg基本使用(获取数据)
JNI补环境时对数据（字段和属性）的处理
1. 第一个方法就是理解方法、字段的语义，返回合适的值。
2. 第二个方法是`ADB`命令 (比如上篇获取屏幕的宽高)
3. 第三个办法是自写 demo app 做验证
4. 第四个办法是 Frida Hook 和 Call
5. 第五个办法是使用 trace
在大体上，我们就使用这五类办法。在实际处理中，trace 是最快的补环境办法，下文讨论几个常见 trace 工具的使用。
## r0tracer
[r0tracer](https://github.com/r0ysue/r0tracer) 是最稳定和顺手的 Java 层 trace 工具

## jnitrace
[jnitrace](https://github.com/chame1eon/jnitrace) 是一个基于 Frida 的专用于追踪 JNI 函数的工具，在理论上，它是最方便好用，专业对口的工具。
[使用 jnitrace 分析 native 方法调用过程jnitrace jnitrace 可以动态跟踪 Java 层通 - 掘金](https://juejin.cn/post/7430494800236888099)
我们并不喜欢使用 JNITrace，尽管它是目前最为完善的 JNI trace 方面的开源工具，因为它**相当相当不稳定**。
使用 JNITrace 无输出或输出遗漏的现象非常常见和严重，而且这种情况依 Frida 版本和测试机版本、型号关系紧密，总之就是不靠谱。
## jtrace
[jtrace](https://github.com/SeeFlowerX/jtrace) 是 SeeFlowerX 开发的一个 JNI 追踪项目，同样基于 Frida。它比 JNITrace 更稳定一些，但又存在新的问题。



# unidbg基本使用(补jni时不同类库的处理)
JNI补环境时对代码（方法）的处理
## JDK 里包含的类库
对于 JDK 里包含的类库，可以直接复现其方法逻辑。

比如对 SimpleDateFormat 的操作

```java
case "java/text/SimpleDateFormat-><init>(Ljava/lang/String;Ljava/util/Locale;)V":{
    String pattern = vaList.getObjectArg(0).getValue().toString();
    Locale locale = (Locale) vaList.getObjectArg(1).getValue();
    simpleDateFormat = new SimpleDateFormat(pattern, locale);
    return;
}
```

比如对 StringBuilder 的操作

```java
case "java/lang/StringBuilder->append(Ljava/lang/String;)Ljava/lang/StringBuilder;":{
    String str = vaList.getObjectArg(0).getValue().toString();
    return ProxyDvmObject.createObject(vm, ((StringBuilder) dvmObject.getValue()).append(str));
}
```

比如对 String 的操作

```java
case "java/lang/String->equalsIgnoreCase(Ljava/lang/String;)Z":{
    return dvmObject.getValue().toString().equalsIgnoreCase(vaList.getObjectArg(0).getValue().toString());
}
```

比如 okhttp3，可以 maven 引入然后正常使用

```java
case "okhttp3/Headers->name(I)Ljava/lang/String;":{
    Headers headers = (Headers) dvmObject.getValue();
    return new StringObject(vm, headers.name(vaList.getIntArg(0)));
}
case "okhttp3/Headers->value(I)Ljava/lang/String;":{
    Headers headers = (Headers) dvmObject.getValue();
    return new StringObject(vm, headers.value(vaList.getIntArg(0)));
}
case "okio/Buffer->clone()Lokio/Buffer;":{
    Buffer buffer = (Buffer) dvmObject.getValue();
    return vm.resolveClass("okio/Buffer").newObject(buffer.clone());
}
case "okhttp3/Request->newBuilder()Lokhttp3/Request$Builder;": {
    Request request = (Request) dvmObject.getValue();
    return vm.resolveClass("okhttp3/Request$Builder").newObject(request.newBuilder());
}
case "okhttp3/Request$Builder->header(Ljava/lang/String;Ljava/lang/String;)Lokhttp3/Request$Builder;": {
    Request.Builder builder = (Request.Builder) dvmObject.getValue();
    builder.header(vaList.getObjectArg(0).getValue().toString(), vaList.getObjectArg(1).getValue().toString());
    return dvmObject;
}
case "okhttp3/Request$Builder->build()Lokhttp3/Request;": {
    Request.Builder builder = (Request.Builder) dvmObject.getValue();
    request = builder.build();
    return vm.resolveClass("okhttp3/Request").newObject(request);
}
```


## 样本自定义的类库
样本自定义的类库，可以在`JADX`、`GDA`、`JEB`等工具中反编译，然后把代码逻辑拷贝到 Unidbg 环境里，前文我们就这么处理过 ContainerUtils、SignedQuery，尤其是各种工具类和方法容器。

有些 Android FrameWork 层的类库也同样可以拷贝过来

比如 TextUtils 里的方法

```java
case "android/text/TextUtils->isEmpty(Ljava/lang/CharSequence;)Z":{
    String str = vaList.getObjectArg(0).getValue().toString();
    return str == null || str.length() == 0;
}
```

比如 Android 的 Base64 工具类，可以直接抠出来 [📎Base64.java](https://www.yuque.com/attachments/yuque/0/2022/java/707476/1668751084861-b37180a5-ae81-444e-987a-0c6651bc38cc.java)。

```java
case "android/util/Base64->decode(Ljava/lang/String;I)[B":{
    String arg = vaList.getObjectArg(0).getValue().toString();
    int flag = vaList.getIntArg(1);
    return new ByteArray(vm, Base64.decode(arg, flag));
}
```

## Android FrameWork 类库

有些 Android FrameWork 类库，使用频率很高，JDK 里又没有其对应，我们就需要复写其语义，实现其基本功能。

比如 SharedPreferences，我们最好写一个小的 demo，实现 Android 中对 SP 的创建、打开、读取、修改、保存这些基本功能。

```java
import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.JSONObject;
import java.io.*;

public class SharedPreferences {
    private JSONObject jsonObject;
    private final File mFile;

    SharedPreferences(File file) {
        mFile = file;
        jsonObject = null;
        loadFromDisk();
    }
    
    private void loadFromDisk() {
        if (mFile.exists() && !mFile.canRead()) {
            System.out.println("Attempt to read preferences file " + mFile + " without permission");
        }
        BufferedInputStream str;
        try {
            str = new BufferedInputStream(new FileInputStream(mFile), 16 * 1024);
            readSP(str);
        } catch (Exception e) {
            System.out.println("getSharedPreferences:" + e);
        }
    }

    public String getString(String key, String defValue) {
        String v = (String)jsonObject.get(key);
        return v != null ? v : defValue;
    }

    public void putString(String key, String value){
        jsonObject.put(key, value);
    }

    private void readSP(InputStream in){
        char[] buf = new char[16 * 1024];
        InputStreamReader input;
        try {
            input = new InputStreamReader(in);
            int len =input.read(buf);
            String text =new String(buf,0, len);
            jsonObject = JSON.parseObject(text);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

以及根据样本所操作的属性和方法，构建对应的类去描述它

```java
public class AccessibilityServiceInfo {
    // 指定您希望服务处理的无障碍事件所属的应用的软件包名称。如果省略此参数，则无障碍服务会被视为可用于处理任何应用的无障碍事件。
    // 即当前无障碍服务作用于哪些app
    public String[] packageNames;
    // 当前无障碍服务的名称
    public String name;
    // 当前无障碍服务所属App的标签名
    public CharSequence label;
    // 当前无障碍服务所属App的包名
    public String packageName;

    public AccessibilityServiceInfo(String packageName, String name, CharSequence label, String[] packageNames){
        this.packageName = packageName;
        this.name = name;
        this.label = label;
        this.packageNames = packageNames;
    }
}
```

## 占位或返回 null

如果其他方法处理不了或者很麻烦，我们会将方法调用”降级“为简单的数据返回或占位或返回 null，这么做有风险，但总比没法继续执行好。

比如

```java
case "android/bluetooth/BluetoothAdapter->getDefaultAdapter()Landroid/bluetooth/BluetoothAdapter;":{
    return dvmClass.newObject(signature);
}
case "android/telephony/SubscriptionManager->from(Landroid/content/Context;)Landroid/telephony/SubscriptionManager;":{
    return dvmClass.newObject(signature);
}
case "android/net/Uri->parse(Ljava/lang/String;)Landroid/net/Uri;":{
    String key = vaList.getObjectArg(0).getValue().toString();
    if(key.equals("content://telephony/siminfo")){
        return dvmClass.newObject(key);
    }
}
```

比如

```java
// 获取packageName信息，后续对比是否有多开助手类app
// https://blog.csdn.net/qq_32227681/article/details/110563688
case "android/content/pm/PackageManager->getInstalledPackages(I)Ljava/util/List;":{
    return null;
}
```

## 尾声

总体上，补 JNI 环境的代码逻辑方面，就这么几种处理办法。

在这几篇文章里，我们讨论了补 JNI 环境的形式与内容。除了这些确切的知识外，我们还有大量的案例可供参考，当不知道如何补 JNI 环境时可以去检索和参考。

百度网盘链接：[https://pan.baidu.com/s/1G0U8frKK3dotY2cryztFjQ](https://pan.baidu.com/s/1G0U8frKK3dotY2cryztFjQ) 提取码：6666


# unidbg基本使用(Context处理)
## 一、引言

《Unidbg 的基本使用（三）》一文中留了个坑。

## 二、初探 Context

报错如下

```java
[08:21:02 080]  WARN [com.github.unidbg.linux.ARM32SyscallHandler] (ARM32SyscallHandler:532) - handleInterrupt intno=2, NR=-1073744548, svcNumber=0x170, PC=unidbg@0xfffe0794, LR=RX@0x4004db2f[libnet_crypto.so]0x4db2f, syscall=null
java.lang.UnsupportedOperationException: com/izuiyou/common/base/BaseApplication->getAppContext()Landroid/content/Context;
	at com.github.unidbg.linux.android.dvm.AbstractJni.callStaticObjectMethodV(AbstractJni.java:503)
	at com.github.unidbg.linux.android.dvm.AbstractJni.callStaticObjectMethodV(AbstractJni.java:437)
	at com.github.unidbg.linux.android.dvm.DvmMethod.callStaticObjectMethodV(DvmMethod.java:64)
	at com.github.unidbg.linux.android.dvm.DalvikVM$113.handle(DalvikVM.java:1810)
	at com.github.unidbg.linux.ARM32SyscallHandler.hook(ARM32SyscallHandler.java:131)
	at com.github.unidbg.arm.backend.Unicorn2Backend$11.hook(Unicorn2Backend.java:347)
	at com.github.unidbg.arm.backend.unicorn.Unicorn$NewHook.onInterrupt(Unicorn.java:109)
```

按照前面讨论的各类规范，应该占位为 Context 实例。

```java
DvmObject<?> context = vm.resolveClass("android/content/Context").newObject(null);
```

但我们却占位为 AppController 实例。

```java
DvmObject<?> context = vm.resolveClass("cn/xiaochuankeji/tieba/AppController").newObject(null);
```

假设按照规范补，后面会出现什么问题？

```java
package com.izuiyou;

import com.github.unidbg.AndroidEmulator;
import com.github.unidbg.arm.backend.Unicorn2Factory;
import com.github.unidbg.linux.android.AndroidEmulatorBuilder;
import com.github.unidbg.linux.android.AndroidResolver;
import com.github.unidbg.linux.android.dvm.*;
import com.github.unidbg.memory.Memory;
import com.github.unidbg.virtualmodule.android.AndroidModule;

import java.io.File;
import java.nio.charset.StandardCharsets;

public class NetWorkNew extends AbstractJni{
    private final AndroidEmulator emulator;
    private final DvmClass NetCrypto;
    private final VM vm;

    public NetWorkNew() {
        emulator = AndroidEmulatorBuilder
                .for32Bit()
                .addBackendFactory(new Unicorn2Factory(true))
                .setProcessName("cn.xiaochuankeji.tieba")
                .build();

        Memory memory = emulator.getMemory();
        memory.setLibraryResolver(new AndroidResolver(23));
        vm = emulator.createDalvikVM(new File("unidbg-android/src/test/resources/zuiyou/right573.apk"));
        vm.setJni(this);
        vm.setVerbose(true);
        // 使用 libandroid.so 的虚拟模块
        new AndroidModule(emulator, vm).register(memory);
        DalvikModule dm = vm.loadLibrary("net_crypto", true);
        NetCrypto = vm.resolveClass("com.izuiyou.network.NetCrypto");
        dm.callJNI_OnLoad(emulator);
    }

    public String callSign(){
        String arg1 = "hello world";
        byte[] arg2 = "V I 50".getBytes(StandardCharsets.UTF_8);
        String ret = NetCrypto.callStaticJniMethodObject(emulator, "sign(Ljava/lang/String;[B)Ljava/lang/String;", arg1, arg2).getValue().toString();
        return ret;
    }

    @Override
    public DvmObject<?> callStaticObjectMethodV(BaseVM vm, DvmClass dvmClass, String signature, VaList vaList) {
        switch (signature){
            case "com/izuiyou/common/base/BaseApplication->getAppContext()Landroid/content/Context;":{
                return vm.resolveClass("android/content/Context").newObject(null);
            }
        }
        return super.callStaticObjectMethodV(vm, dvmClass, signature, vaList);
    }
    

    public static void main(String[] args) {
        NetWorkNew nw = new NetWorkNew();
        String result = nw.callSign();
        System.out.println("call s result:"+result);
    }
}
```

继续运行，获取对象的类实例

```java
java.lang.UnsupportedOperationException: android/content/Context->getClass()Ljava/lang/Class;
	at com.github.unidbg.linux.android.dvm.AbstractJni.callObjectMethodV(AbstractJni.java:416)
	at com.github.unidbg.linux.android.dvm.AbstractJni.callObjectMethodV(AbstractJni.java:262)
	at com.github.unidbg.linux.android.dvm.DvmMethod.callObjectMethodV(DvmMethod.java:89)
```

`DvmObject`的`getObjectType()`用于实现这个功能。

```java
@Override
public DvmObject<?> callObjectMethodV(BaseVM vm, DvmObject<?> dvmObject, String signature, VaList vaList) {
    switch (signature){
        case "android/content/Context->getClass()Ljava/lang/Class;":{
            return dvmObject.getObjectType();
        }
    }
    return super.callObjectMethodV(vm, dvmObject, signature, vaList);
}
```

继续运行

```java
java.lang.UnsupportedOperationException: java/lang/Class->getSimpleName()Ljava/lang/String;
	at com.github.unidbg.linux.android.dvm.AbstractJni.callObjectMethodV(AbstractJni.java:416)
	at com.izuiyou.NetWorkNew.callObjectMethodV(NetWorkNew.java:62)
```

这是在获取类的简写名，比如`android.content.Context`简写名就是`Context`。

`DvmClass`的`getClassName`可以获取类名，我们要截取末尾的名字。

```java
case "java/lang/Class->getSimpleName()Ljava/lang/String;":{
    String className = ((DvmClass) dvmObject).getClassName();
    String[] name = className.split("/");
    return new StringObject(vm, name[name.length - 1]);
}
```

继续运行

```java
java.lang.UnsupportedOperationException: cn/xiaochuankeji/tieba/common/debug/AppLogReporter->reportAppRuntime(Ljava/lang/String;Ljava/lang/String;)V
	at com.github.unidbg.linux.android.dvm.AbstractJni.callStaticVoidMethodV(AbstractJni.java:697)
	at com.github.unidbg.linux.android.dvm.AbstractJni.callStaticVoidMethodV(AbstractJni.java:692)
```

样本在调用`reportAppRuntime`函数，看这个名字是在上报运行情况，打印它的参数看看。

```java
@Override
public void callStaticVoidMethodV(BaseVM vm, DvmClass dvmClass, String signature, VaList vaList) {
    switch (signature){
        case "cn/xiaochuankeji/tieba/common/debug/AppLogReporter->reportAppRuntime(Ljava/lang/String;Ljava/lang/String;)V":{
            String arg1 = vaList.getObjectArg(0).getValue().toString();
            String arg2 = vaList.getObjectArg(1).getValue().toString();
            System.out.println("arg1:"+arg1);
            System.out.println("arg2:"+arg2);
        }
    }
    super.callStaticVoidMethodV(vm, dvmClass, signature, vaList);
}
```

运行

```java
arg1:runtiem
arg2:invalid application name: Context
```

这个上报信息很微妙，参数 1 看起来是 runtime 运行时的错误拼写，参数 2 则是具体信息—— Application 名字不合规。

为什么会出现这样的问题？让我们回到最初的问题上

```java
@Override
public DvmObject<?> callStaticObjectMethodV(BaseVM vm, DvmClass dvmClass, String signature, VaList vaList) {
    switch (signature){
        case "com/izuiyou/common/base/BaseApplication->getAppContext()Landroid/content/Context;":{
            return vm.resolveClass("android/content/Context").newObject(null);
        }
    }
    return super.callStaticObjectMethodV(vm, dvmClass, signature, vaList);
}
```

问题就出在这里，`getAppContext`的返回值类型是 Context，它是一个抽象类，返回的是抽象类的子类对象。在一般情况里我们不需要讲究，但这里它获取了返回的 Context 的类名，因此不得不确认样本到底返回了 Context 的什么子类对象，否则类名就对不上了。

我这里使用 r0tracer 做确认

```java
function main() {
  Java.perform(function () {
    console.Purple("r0tracer begin ... !")
    hook("com.izuiyou.common.base.BaseApplication", "");
  })
}
// frida -U -f cn.xiaochuankeji.tieba -l r0tracer.js --no-pause
```

运行

```java
*** entered com.izuiyou.common.base.BaseApplication.getAppContext

retval: cn.xiaochuankeji.tieba.AppController@7ff0e => "<instance: android.content.Context, $className: cn.xiaochuankeji.tieba.AppController>"
*** exiting com.izuiyou.common.base.BaseApplication.getAppContext
```

可以发现具体的实现子类是`cn.xiaochuankeji.tieba.AppController`

```java
package cn.xiaochuankeji.tieba;

import android.app.Application;
import android.content.Context;
import android.content.Intent;
import android.os.Build;
import android.os.Bundle;
import com.facebook.common.memory.MemoryTrimType;
import com.izuiyou.common.base.BaseApplication;
import com.meituan.robust.ChangeQuickRedirect;
import com.meituan.robust.PatchProxy;
import com.meituan.robust.PatchProxyResult;
import defpackage.u36;
import java.util.Map;
import okhttp3.OkHttpClient;

/* loaded from: classes.dex */
public class AppController extends BaseApplication {
    public static ChangeQuickRedirect changeQuickRedirect;
    public boolean a = false;
    public boolean b;


}
```

因此修改前面的代码为

```java
@Override
public DvmObject<?> callStaticObjectMethodV(BaseVM vm, DvmClass dvmClass, String signature, VaList vaList) {
    switch (signature){
        case "com/izuiyou/common/base/BaseApplication->getAppContext()Landroid/content/Context;":{
            return vm.resolveClass("cn/xiaochuankeji/tieba/AppController").newObject(null);
        }
    }
    return super.callStaticObjectMethodV(vm, dvmClass, signature, vaList);
}
```

这就是第三篇中的逻辑，当时不去讨论它，1 是刚接触 JNI 补环境，有更多的基础内容需要关注，2 是这种情况出现的并不多。

## 三、尾声

应该说，我们填了一个小坑。