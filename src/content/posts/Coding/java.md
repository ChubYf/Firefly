---
title: java学习
published: 2026-07-01
description: java学习
image: ./cover.jpg
tags: [编程]
category: 编程学习
draft: false
---

# 前言
内容由ai生成

[学习笔记：黑马程序员Java-基础篇（第一部分黑马程序员java基础入门-CSDN博客](https://blog.csdn.net/D_boj/article/details/132192272)


# 继承与接口
继承（Inheritance）和接口（Interface）是Java面向对象编程（OOP）的两大核心机制，它们共同支撑起代码复用、多态和系统扩展性。下面我会从**定义、语法、使用场景、设计原则**以及**在AWT中的实际应用**五个维度进行详细剖析，并结合代码示例帮助理解。

---

## 1. 继承（Inheritance）

### 1.1 定义与作用
继承是一种 **“is-a”** 关系，它允许一个类（子类）继承另一个类（父类）的成员（字段和方法），从而实现代码复用，并可以在此基础上扩展或重写功能。

- **复用父类代码**：子类自动拥有父类的非私有成员。
- **多态支持**：父类引用可以指向子类对象，动态调用重写后的方法。
- **形成层次结构**：如AWT中的 `Component` → `Container` → `Window` → `Frame`。

### 1.2 语法与关键点
- 使用 `extends` 关键字。
- Java只支持**单继承**（一个类只能有一个直接父类）。
- 子类构造器必须调用父类构造器（隐式调用无参构造，或显式 `super(...)` 调用有参构造）。
- 子类可以重写（Override）父类的方法（使用 `@Override` 注解增强可读性）。
- `final` 类不能被继承，`final` 方法不能被重写。

### 1.3 代码示例
```java
// 父类
class Animal {
    protected String name;
    public Animal(String name) {
        this.name = name;
    }
    public void speak() {
        System.out.println(name + "发出声音");
    }
}

// 子类
class Dog extends Animal {
    private String breed;
    public Dog(String name, String breed) {
        super(name);   // 必须显式调用父类构造器
        this.breed = breed;
    }
    @Override
    public void speak() {
        System.out.println(name + "汪汪叫");
    }
    public void wagTail() {
        System.out.println(name + "摇尾巴");
    }
}
```

### 1.4 抽象类（Abstract Class）
- 使用 `abstract` 修饰，不能实例化。
- 可以包含抽象方法（只有声明，没有实现），子类必须实现所有抽象方法（除非子类也是抽象类）。
- 适合作为模板，定义通用行为框架，如AWT中的 `Component` 就是抽象类。

---

## 2. 接口（Interface）

### 2.1 定义与作用
接口是一种 **“can-do”** 契约，它定义了一组方法签名（常量、抽象方法、默认方法、静态方法），类通过实现接口来承诺提供这些行为。

- **解耦**：接口将“做什么”与“怎么做”分离。
- **多实现**：一个类可以实现多个接口，弥补单继承的不足。
- **定义标准**：如事件监听器、集合框架中的 `Comparable` 等。

### 2.2 语法与关键点
- 使用 `interface` 关键字。
- 接口中方法默认是 `public abstract`（Java 8之前），字段默认是 `public static final`。
- 类使用 `implements` 关键字实现接口，必须实现所有抽象方法（除非类本身是抽象类）。
- **Java 8+** 支持 `default` 方法（默认实现）和 `static` 方法。
- **Java 9+** 支持私有方法。
- 接口可以**多继承**（一个接口可以 `extends` 多个父接口）。

### 2.3 代码示例
```java
// 接口定义
interface Flyable {
    void fly();   // 抽象方法
    default void land() {  // 默认方法
        System.out.println("降落中...");
    }
}

interface Swimmable {
    void swim();
}

// 类实现多个接口
class Duck implements Flyable, Swimmable {
    @Override
    public void fly() {
        System.out.println("鸭子飞行");
    }
    @Override
    public void swim() {
        System.out.println("鸭子游泳");
    }
    // land() 可以选择重写或不重写，因为已有默认实现
}
```

### 2.4 标记接口（Marker Interface）
没有任何方法和字段的接口，如 `Serializable`，仅用于给类打上标记，表示其具备某种特殊能力，JVM或框架会进行特殊处理。

---

## 3. 继承 vs 接口：核心区别

| 维度 | 继承（extends） | 接口（implements） |
|------|----------------|-------------------|
| **关系** | `is-a`（是一个） | `can-do`（能做什么） |
| **继承数量** | 单继承（一个父类） | 多实现（多个接口） |
| **成员** | 可包含字段、具体方法、抽象方法 | 抽象方法为主，Java 8后可包含默认/静态方法（字段只能是常量） |
| **构造器** | 有构造器 | 无构造器 |
| **访问控制** | 可包含不同权限修饰符 | 方法默认为 `public`，字段默认为 `public static final` |
| **设计目的** | 代码复用和层次抽象 | 定义行为契约，解耦实现 |
| **可扩展性** | 父类变化会影响所有子类（脆弱基类问题） | 接口变化影响所有实现类（但默认方法可提供向后兼容） |

---

## 4. 在AWT中的实际应用举例

AWT是继承和接口协同工作的绝佳范例。

### 4.1 继承体系
- `Component`（抽象类）→ `Container`（抽象类）→ `Window` → `Frame`（具体类）。  
  - `Component` 定义了所有UI组件的通用属性（位置、大小、颜色）和事件处理基础设施。
  - `Frame` 通过继承获得了这些能力，并扩展了窗口特有的行为（标题、菜单栏等）。

### 4.2 接口用于事件监听
AWT的事件模型是**基于接口的**。例如，要处理按钮点击，需实现 `ActionListener` 接口：
```java
Button btn = new Button("点击");
btn.addActionListener(new ActionListener() {
    @Override
    public void actionPerformed(ActionEvent e) {
        System.out.println("按钮被点击");
    }
});
```
这里 `ActionListener` 是一个接口，它定义了 `actionPerformed` 这个契约。任何类（可能是 `Frame` 子类，也可能不是）都能实现它，从而获得处理动作事件的能力，**无需继承任何特定的父类**，体现了接口的解耦优势。

类似地，`WindowListener`、`MouseListener` 等都是接口，使得事件处理逻辑可以灵活地放在任何类中，而不受继承限制。

---

## 5. 设计原则与最佳实践

1. **优先使用组合而非继承**  
   继承强耦合，子类依赖父类实现细节，破坏封装。若只是为了复用代码，可改用组合（持有另一个类的实例）和委托。

2. **接口是定义混合型能力（Mixin）的首选**  
   例如 `Runnable`、`Serializable`、`Cloneable`，它们可以为不相关类添加通用功能。

3. **抽象类用于提供基础实现**  
   当多个类有大量公共代码时，抽象类能避免重复，并提供模板方法模式。

4. **接口隔离原则（ISP）**  
   不应强迫实现类依赖它们不需要的方法，应将大接口拆分为多个小接口。

5. **默认方法谨慎使用**  
   默认方法是为API演进提供向后兼容，但可能引起“菱形继承”问题（多实现冲突时需显式覆盖）。

---

## 6. 总结

- **继承**是一种垂直的、强关联的复用机制，适合构建类型层次，但需小心维护。
- **接口**是一种水平的、松耦合的契约机制，适合定义行为标准，增强灵活性和多态性。
- 在实际开发中，二者常配合使用：**抽象类定义骨架，接口定义行为**。例如，`ArrayList` 继承 `AbstractList`（抽象类），同时实现 `List`、`RandomAccess` 等接口。

理解这两个概念的关键在于：**继承强调“是什么”，接口强调“能做什么”**。合理地使用它们，能让代码结构清晰、易于扩展和维护。希望以上解释能帮你透彻掌握这些核心机制。如有具体疑问，欢迎继续提问！


# 内部类
太好了！既然你已经掌握了**匿名类**（它是内部类的一种“极简变体”），现在我们**把视野拉开**，系统学习Java中**全部4种内部类**。

我会继续用**0基础都能听懂**的大白话 + 生活比喻 + 完整代码，带你一次性打通！

---

### 1. 先来个生活大比喻（什么是内部类？）

想象你有一栋**大房子（外部类）**，房子里有：

- **客厅**（成员内部类）：房子自带的功能区，必须依赖房子存在（房子拆了客厅就没了）。
- **工具箱**（静态内部类）：放在储藏间，不依赖房子本身，可以独立拿出来用。
- **厨房里的临时小灶**（局部内部类）：只在做饭（方法）的时候临时搭起来用一下。
- **一次性外卖盒**（匿名类）：用完即扔，没有名字（上节课学的）。

**官方定义**：定义在**另一个类内部**的类，就叫内部类。它可以直接访问外部类的私有成员（就像客厅能直接享用房子的水电）。

---

### 2. 第一种：成员内部类（最基础、最常用）

> **特点**：就像房子的“客厅”，**必须**先有房子对象，才能有客厅对象。它能随便使用外部类的所有东西（包括私有的）。

```java
// 外部类（大房子）
public class House {
    private String address = "北京朝阳区"; // 私有属性

    // 成员内部类（客厅）
    class LivingRoom {
        public void showAddress() {
            // 内部类可以直接访问外部类的私有属性！超方便
            System.out.println("客厅所在的地址是：" + address);
        }
    }

    // 外部类的方法
    public void live() {
        LivingRoom room = new LivingRoom(); // 在房子内部建客厅
        room.showAddress();
    }
}

// 测试类
public class Test {
    public static void main(String[] args) {
        // 1. 必须先创建外部类对象（先有房子）
        House myHouse = new House();
        
        // 2. 通过外部类对象，再去 new 内部类（注意语法格式！）
        House.LivingRoom myRoom = myHouse.new LivingRoom();
        myRoom.showAddress(); // 输出：客厅所在的地址是：北京朝阳区
    }
}
```

> **0基础必记语法**：`外部类.内部类 变量名 = 外部类对象.new 内部类();`  
> 中间那个 `.new` 是固定写法，别写反了！

---

### 3. 第二种：静态内部类（加上 `static`）

> **特点**：相当于一个放在房子里的“独立工具箱”。它**不依赖**外部类对象，自己就能创建。但它不能直接访问外部类的非静态属性（因为没有房子，哪来的水电？）。

```java
// 外部类
public class House {
    private static String city = "北京"; // 静态属性（属于类的）
    private String owner = "张三";      // 非静态属性（属于对象的）

    // 静态内部类（工具箱）
    static class ToolBox {
        public void showCity() {
            // 只能访问外部类的静态成员
            System.out.println("城市：" + city); // 没问题
            // System.out.println(owner); // 报错！因为没房子对象，不知道owner是谁
        }
    }
}

// 测试类
public class Test {
    public static void main(String[] args) {
        // 注意：不需要创建外部类对象！直接 new 内部类
        House.ToolBox box = new House.ToolBox();
        box.showCity(); // 输出：城市：北京
    }
}
```

> **0基础必记语法**：`外部类.静态内部类 变量名 = new 外部类.静态内部类();`  
> **和成员内部类的区别**：少了 `外部类对象.` 这一步。

---

### 4. 第三种：局部内部类（定义在方法里）

> **特点**：就像你**临时**在厨房搭的灶台，只在当前这个方法里有效，方法执行完就“消失”了。外面根本访问不到它。

```java
public class House {
    
    public void cook() {
        // 局部内部类（定义在方法内部）
        class Stove {
            public void fire() {
                System.out.println("灶台开火啦！");
            }
        }
        
        // 只能在当前方法内使用这个类
        Stove stove = new Stove();
        stove.fire();
    }
}

// 测试类
public class Test {
    public static void main(String[] args) {
        House h = new House();
        h.cook(); // 输出：灶台开火啦！
        
        // 下面这行会报错！因为 Stove 类在方法外根本看不见
        // House.Stove s = new House.Stove(); 
    }
}
```

> **0基础避坑**：如果局部内部类要使用方法里的**局部变量**，这个变量必须是 `final` 或“事实不变”（跟匿名类的规则一模一样）。

---

### 5. 第四种：匿名内部类（复习 + 巩固）

就是上节课学的，它是**局部内部类**的简化版，连名字都省了。

```java
// 假设有个接口
interface Eatable {
    void eat();
}

public class Test {
    public static void main(String[] args) {
        // 直接 new 接口，当场实现方法，没有类名
        Eatable food = new Eatable() {
            @Override
            public void eat() {
                System.out.println("匿名内部类在做炸鸡");
            }
        };
        food.eat();
    }
}
```

---

### 6. 四大内部类对比总结（超清晰表格）

| 类型 | 有无 `static` | 是否需要外部类对象才能创建？ | 能否访问外部类的私有属性？ | 使用场景 |
| :--- | :--- | :--- | :--- | :--- |
| **成员内部类** | 无 | **需要**（`外部类对象.new`） | 能（包括非静态和静态） | 和外部类紧密绑定，如 `ArrayList` 中的迭代器 |
| **静态内部类** | **有** | **不需要**（直接 `new`） | 只能访问外部类的**静态**属性 | 独立辅助类，如 `HashMap` 中的 `Node` 节点 |
| **局部内部类** | 无 | 只能在定义它的方法里 `new` | 能 | 只在方法内临时用一次，封装性极强 |
| **匿名内部类** | 无 | 定义的同时就 `new` | 能（但外部变量需 `final`） | 事件监听、线程、临时回调（最简洁） |

---

### 7. 面试/笔试最爱考的坑（this 指的是谁？）

在内部类中，如果变量名冲突了，怎么区分“外部类的属性”和“内部类的属性”？

```java
public class Outer {
    private String name = "外部名字";

    class Inner {
        private String name = "内部名字";

        public void show() {
            System.out.println(name);                    // 内部名字
            System.out.println(this.name);               // 内部名字（同上）
            System.out.println(Outer.this.name);         // 外部名字（固定写法）
        }
    }
}
```

> **口诀**：在内部类里想指代**外部类的属性**，就用 `外部类名.this.属性名`。

---

### 8. 终极警告：内存泄漏（新手最容易犯的错）

**成员内部类**会**隐式持有**外部类对象的引用。  
如果你在外部类里创建了一个成员内部类对象，并且这个对象被全局变量长期持有，那么外部类对象**永远无法被垃圾回收（GC）**，导致内存溢出！

> **解决建议**：如果内部类不需要访问外部类的非静态属性，**优先使用 `static` 静态内部类**，这样就不会持有外部类的引用了。

---

### 9. 给你布置一个“动手小练习”

试着用**静态内部类**写一个简单的“学生管理系统”：

```java
public class School {
    private static String schoolName = "第一中学";
    
    // 请你写一个静态内部类 Student，包含 name 和 age 属性
    // 在 Student 里写一个方法，打印出 "XX 是 第一中学 的学生，年龄 XX"
    // 然后在 main 方法里直接 new Student() 并调用
}
```

> **答案思路**：静态内部类里可以直接访问 `schoolName`（因为是静态的），完全不需要创建 `School` 对象。

---

如果上面这个练习写不出来，或者对 `static` 和非 `static` 的区别还迷糊，**随时喊我**，我再用“电风扇”和“遥控器”的比喻给你讲一遍 `static`！ 😄

# 匿名类

欢迎来到Java世界！针对**0基础**的同学，我会用最生活化的比喻和最简单的代码，带你彻底搞懂**匿名类**。

---

### 1. 先来个生活比喻（什么是匿名类？）

想象一下：

- 你平时点外卖，需要先创建一个**店铺类**（比如`兰州拉面`），然后`new`一个对象（拉面师傅）来做面。这叫**有名字的类**。
- 但如果你今天只想吃一顿**一次性**的汉堡，不想专门去注册一个“汉堡店”的营业执照（定义类），你直接**当场**在厨房里吆喝一声：“给我做个汉堡！”，做完吃掉就完事了。

这个**当场创建、用完即走、没有名字**的“临时小作坊”，就是**匿名类**。

**官方定义**：匿名类就是**没有名字**的内部类。它用于**临时**重写（实现）某个类或接口的方法，通常只用一次。

---

### 2. 0基础语法拆解（看骨架）

匿名类必须依托于 **父类** 或 **接口**。它的骨架长这样：

```java
// 假设有一个接口或父类叫 X
X 对象名 = new X() {
    // 在这里重写 X 中的方法
};
```

**注意**：最后那个`;`分号不能丢！因为这是一条赋值语句。

---

### 3. 实战案例一：基于“接口”的匿名类（最常用）

假设我们有一个**接口**叫`Eatable`（可吃的），里面有一个`eat()`方法。

**不用匿名类的传统写法**（你需要先定义一个实现类）：
```java
// 1. 定义接口
interface Eatable {
    void eat();
}

// 2. 专门写一个类去实现它（有名字的类）
class Apple implements Eatable {
    @Override
    public void eat() {
        System.out.println("我在吃苹果");
    }
}

// 3. 在 main 方法中使用
public class Test {
    public static void main(String[] args) {
        Apple a = new Apple(); // 创建有名字的对象
        a.eat();
    }
}
```

**使用匿名类的写法**（省掉第2步，当场实现）：
```java
// 1. 定义接口
interface Eatable {
    void eat();
}

// 2. 直接在 main 方法中用匿名类
public class Test {
    public static void main(String[] args) {
        // 注意看这里：new 的是接口名 Eatable，但后面跟着大括号 {}
        // 这表示：我临时创建了一个“没有名字”的类，当场实现了 eat 方法
        Eatable food = new Eatable() {
            @Override
            public void eat() {
                System.out.println("我在吃炸鸡（匿名类做的）");
            }
        };
        
        food.eat(); // 输出：我在吃炸鸡（匿名类做的）
    }
}
```

> **0基础解读**：`new Eatable() { ... }` 整个这一大坨，就相当于一个“汉堡”。它没有单独的文件名，没有单独的类名，直接赋值给 `food` 变量拿去用了。

---

### 4. 实战案例二：基于“普通父类”的匿名类

如果你的父类是一个普通类（不是接口），同样可以用匿名类重写它的方法。

```java
// 1. 普通的父类（有名字）
class Animal {
    public void shout() {
        System.out.println("动物在叫");
    }
}

// 2. 使用匿名类
public class Test {
    public static void main(String[] args) {
        // 这里 new 的是父类 Animal，但大括号里重写了 shout 方法
        Animal cat = new Animal() {
            @Override
            public void shout() {
                System.out.println("喵喵喵！");
            }
        };
        
        cat.shout(); // 输出：喵喵喵！
        
        // 对比一下普通的 new Animal()
        Animal dog = new Animal();
        dog.shout(); // 输出：动物在叫（没重写，是原版的）
    }
}
```

---

### 5. 匿名类的核心特征（0基础必记）

| 特点 | 解释 |
| :--- | :--- |
| **没有名字** | 你没法在其他地方 `new` 它，只能在这一个地方用。 |
| **必须立即创建对象** | 定义它的同时，就必须 `new` 出来。 |
| **必须继承父类或实现接口** | 它不能凭空出现，必须依附于某个“模板”。 |
| **只能重写方法** | 你不能在里面随意新增自己独有的方法（因为外部变量类型是父类，调用不了新方法）。 |

---

### 6. 最容易踩的坑（变量访问规则）

在匿名类中，如果你想**使用外部的局部变量**（比如方法里的 `int num`），这个变量必须是 **`final` 或“事实不变”** 的。

**错误示例（会报错）**：
```java
public class Test {
    public static void main(String[] args) {
        int count = 10; // 普通变量
        
        Eatable food = new Eatable() {
            @Override
            public void eat() {
                // 这里试图修改外部的 count，Java 不允许！
                // count = 20; 
                System.out.println("count 是：" + count);
            }
        };
    }
}
```

**正确写法**：Java 8 开始，只要你不去修改 `count`，它默认就是 `final` 的，直接用就行。

```java
public class Test {
    public static void main(String[] args) {
        int count = 10; // 只要不修改它，匿名类里就能直接用
        
        Eatable food = new Eatable() {
            @Override
            public void eat() {
                System.out.println("count 是：" + count); // 完全没问题
            }
        };
        food.eat();
    }
}
```

---

### 7. 匿名类 VS Lambda 表达式（简单提一嘴）

如果你学到后面，会发现**如果接口里只有一个方法**（比如这里的 `eat()`），Java 8 之后可以用更短的 `Lambda` 写：

```java
// 匿名类写法（你现在学的）
Eatable food = new Eatable() {
    @Override
    public void eat() {
        System.out.println("吃吃吃");
    }
};

// Lambda 写法（以后学的，更简洁）
Eatable food = () -> System.out.println("吃吃吃");
```

**建议**：0基础先把**匿名类**学好，因为 Lambda 只是它的“语法糖”，底层逻辑是一样的。

---

### 8. 总结：什么时候用匿名类？

- **场景**：按钮点击事件（GUI编程）、线程启动（`new Thread`）、临时排序比较器（`Comparator`）。
- **口诀**：**“一次就好，懒得起名”**。如果你需要一个类只用这一次，就毫不犹豫地用匿名类，省得单独建一个 `.java` 文件。

**最后的建议**：把上面代码复制到你的 IDE（比如 IDEA）中，自己敲一遍，把 `System.out.println` 里的文字改着玩，运行看输出。眼睛看会了，手一定要跟上！如果有哪里卡住了，随时问我。😄


# 字符串和容器
太棒了！学完了“类的结构”，现在我们进入Java**最实用、最频繁**的两大领域：**字符串（String）** 和 **容器（集合框架）**。

在Java里，**字符串**处理文字，**容器**装数据。这俩是实际开发中**天天都要用**的东西，像吃饭的筷子和碗。

我会继续用**0基础生活比喻 + 极简代码**，带你一次性把最核心的用法刻在脑子里！

---

### 1. 先来两个生活比喻

- **字符串（String）**：就像一张**便利贴**。你把内容写在上面，一旦写好了（**不可变**），如果想改内容，只能撕掉旧的重写一张新的。你不能在原来的字上擦除修改。
- **容器（集合）**：就像**乐高收纳盒**。以前学的`数组`是“固定格子”的鸡蛋托（放满了就塞不下），而容器是**魔法书包**——想放多少放多少，随时增删改查。

---

### 第一部分：字符串（String）——处理文字的利器

#### 1.1 两种创建方式（0基础看这里）
```java
// 方式1：字面量（最常用，像写普通文字）
String name = "张三";

// 方式2：new 对象（了解即可，实际少用）
String name2 = new String("张三");
```
> **新手记住**：**永远用方式1**（双引号直接写），简单高效。

#### 1.2 核心特性：不可变性（面试必考）
```java
String str = "Hello";
str = str + " World"; 
System.out.println(str); // 输出 Hello World
```
> **误区**：你以为是在原`Hello`后面加了`World`？  
> **真相**：其实是JVM在内存里**新建**了一张写有`Hello World`的新便利贴，把旧的`Hello`扔掉了。所以频繁修改字符串很耗内存。

#### 1.3 0基础必会的7个方法（背下来，天天用！）

| 方法 | 作用 | 示例 |
| :--- | :--- | :--- |
| `.length()` | 获取长度（**注意括号！**） | `"abc".length()` → 3 |
| `.equals()` | **比较内容是否相同**（千万别用`==`） | `"A".equals("A")` → true |
| `.charAt(0)` | 获取指定位置的字符（从0开始） | `"abc".charAt(1)` → 'b' |
| `.substring(1)` | 截取字符串（从1截到末尾） | `"abc".substring(1)` → "bc" |
| `.substring(0,2)` | 截取 [0, 2) 左闭右开 | `"abc".substring(0,2)` → "ab" |
| `.contains("b")` | 判断是否包含某段文字 | `"abc".contains("b")` → true |
| `.split(",")` | 按指定符号切分成数组 | `"a,b,c".split(",")` → ["a","b","c"] |

#### 1.4 新手最容易犯的致命错误（`==` vs `equals`）
```java
String a = "hello";
String b = "hello";
String c = new String("hello");

System.out.println(a == b);      // true（特殊原因，字面量相同指向同一块内存）
System.out.println(a == c);      // false（==比较的是内存地址！）
System.out.println(a.equals(c)); // true（equals比较的是内容！）
```
> **铁律**：**比较字符串内容，永远用 `.equals()`**，别用 `==`！

#### 1.5 频繁拼接怎么办？（进阶：StringBuilder）
如果要在循环里拼字符串（比如拼10000次），用`+`会很慢。这时用**可变**的`StringBuilder`：
```java
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 10; i++) {
    sb.append(i); // 像在铅笔上不断追加，不换纸
}
System.out.println(sb.toString()); // 0123456789
```

---

### 第二部分：容器（集合）——装数据的魔法书包

为什么不用数组？因为**数组长度固定**。容器可以**动态扩容**，想装多少装多少。

Java最常用的容器就**三大类**：`ArrayList`（排队列表）、`HashMap`（字典）、`HashSet`（不重复集合）。0基础先死磕前两个！

#### 2.1 ArrayList（有序列表，像排队）
> **生活比喻**：就像超市的**排队队列**，有先来后到，可以插队（指定位置加），也可以叫某个人出列。

```java
import java.util.ArrayList; // 必须导包！

public class Test {
    public static void main(String[] args) {
        // 1. 创建容器（尖括号<>里写装什么类型，叫“泛型”）
        ArrayList<String> list = new ArrayList<>(); // 装字符串的列表
        
        // 2. 增（添加元素）
        list.add("苹果");
        list.add("香蕉");
        list.add(0, "草莓"); // 在索引0插入，后面往后挪
        
        // 3. 删
        list.remove("香蕉"); // 删掉香蕉
        // list.remove(0);   // 删掉索引0的元素
        
        // 4. 改
        list.set(0, "西瓜"); // 把索引0的草莓改成西瓜
        
        // 5. 查（获取长度和元素）
        System.out.println("长度：" + list.size()); // 2
        System.out.println("第0个：" + list.get(0)); // 西瓜
        
        // 6. 遍历（最常用方式，增强for循环）
        for (String item : list) {
            System.out.println(item);
        }
    }
}
```

#### 2.2 HashMap（键值对，像查字典）
> **生活比喻**：就像**手机通讯录**。你查“张三”（**键Key**），就能拿到他的电话号码“1380000”（**值Value**）。**键不能重复**，值可以重复。

```java
import java.util.HashMap;

public class Test {
    public static void main(String[] args) {
        // 1. 创建（<键的类型, 值的类型>）
        HashMap<String, Integer> scoreMap = new HashMap<>();
        
        // 2. 存数据（put）
        scoreMap.put("小明", 95);
        scoreMap.put("小红", 100);
        scoreMap.put("小明", 60); // 键相同，新值覆盖旧值（小明变成60分）
        
        // 3. 取数据（get）
        Integer xiaoMingScore = scoreMap.get("小明");
        System.out.println("小明分数：" + xiaoMingScore); // 60
        
        // 4. 判断是否包含某个键
        if (scoreMap.containsKey("小红")) {
            System.out.println("小红在字典里！");
        }
        
        // 5. 遍历（取出所有的键）
        for (String name : scoreMap.keySet()) {
            System.out.println(name + " 考了 " + scoreMap.get(name));
        }
        // 输出：小明 考了 60   小红 考了 100
    }
}
```

#### 2.3 泛型（尖括号 `<>`）到底是什么？
> `ArrayList<String>` 中的 `<String>` 就是“标签”。  
> 它告诉容器：“**只能往里装字符串！**”如果尝试装数字 `list.add(123)`，编译直接报错，防止你手滑放错东西。

> **0基础记住**：**基本数据类型（int, double）不能用泛型**，要用它们的“包装类”（`Integer`, `Double`）。  
> 比如：`ArrayList<Integer>` 存数字，`ArrayList<String>` 存文字。

---

### 3. 字符串与容器的梦幻联动（实际开发每天写）

这是新手必须掌握的**组合技**：把一串文字拆开装进容器，或者把容器里的东西拼成文字。

```java
import java.util.ArrayList;

public class Test {
    public static void main(String[] args) {
        String data = "苹果,香蕉,西瓜,葡萄";
        
        // 1. 字符串 -> 容器（用 split 拆开）
        String[] fruitsArray = data.split(",");
        ArrayList<String> fruitList = new ArrayList<>();
        for (String f : fruitsArray) {
            fruitList.add(f);
        }
        System.out.println(fruitList); // [苹果, 香蕉, 西瓜, 葡萄]
        
        // 2. 容器 -> 字符串（用 StringBuilder 拼回去）
        StringBuilder sb = new StringBuilder();
        for (String f : fruitList) {
            sb.append(f).append("-");
        }
        // 去掉最后多余的 "-"
        String result = sb.substring(0, sb.length() - 1);
        System.out.println(result); // 苹果-香蕉-西瓜-葡萄
    }
}
```

---

### 4. 终极对比表（背下来！）

| 对比项 | **数组 (Array)** | **ArrayList** | **HashMap** |
| :--- | :--- | :--- | :--- |
| **长度** | 固定，不能变 | 可变，自动扩容 | 可变，自动扩容 |
| **存储结构** | 按索引（0,1,2...） | 按索引（0,1,2...） | 键值对（Key->Value） |
| **访问方式** | `arr[0]` | `.get(0)` | `.get("key")` |
| **常用场景** | 长度明确不变时 | 需要**有序**遍历、增删末尾时 | 需要**根据名字查数据**时（如缓存） |

---

### 5. 给你布置一个“生活小练习”（必做）

> **需求**：假设你有一个学生名单字符串 `"张三,李四,王五,张三"`（有重复）。
> 1. 将它拆开放入 `ArrayList`。
> 2. 再将这些学生放入 `HashSet`（自动去重），最后打印出去重后的结果。
> 3. 利用 `HashMap` 统计每个名字出现了几次（提示：`map.put(name, map.getOrDefault(name, 0) + 1)`）。

**如果你能独立写出上面3步**，你就已经掌握了Java日常开发中80%的字符串+容器操作！

---

如果上面的代码有哪一行看不懂（比如 `getOrDefault` 是啥，或者增强for循环头晕），**立刻告诉我**，我用更细的“掰开揉碎”方式给你重讲！加油！🚀




# 字符串拼接问题
这个问题问得**非常专业**！能想到这一层，说明你已经开始用“程序员的大脑”思考性能问题了。

为了让你这个**0基础**的同学彻底搞懂，我先给你一个**扎心的真相**：

> **你以为的 `str += "a"` 是一笔一划在写作业；**
> **实际上是每写一个字母，你就把整本作业本撕掉，重新抄一本新的。**

下面我用 **“抄作业”** + **“内存透视”** 两层视角，把原理掰开揉碎讲给你听。

---

### 1. 生活比喻：为什么慢？（抄作业的故事）

假设老师让你抄写 **“ABC”** 这个句子，后面还要再加 1000 个字母。

- **错误做法（用 String）**：
  你先把 `A` 写在纸上。当要加 `B` 时，你觉得纸不够长，于是**扔掉旧纸**，拿一张新纸，把 `AB` 重新抄一遍。当要加 `C` 时，又扔掉旧纸，把 `ABC` 重新抄一遍......
  抄到第 1000 个字母时，你相当于把前面 1000 个字母**反复抄写了 1000 遍**。手不断重抄，纸不断扔，当然**极慢**！

- **正确做法（用 StringBuilder）**：
  你拿了一本**活页笔记本（可变空间）**，直接在最后一页往后写，写完直接翻页，**永远不需要重抄前面的内容**。快如闪电！

---

### 2. 底层原理：Java 内部到底发生了什么？

我们要看 **编译之后（底层）** 的代码。你写的代码是：
```java
String str = "A";
for (int i = 0; i < 1000; i++) {
    str = str + "a"; // 看起来只是加一个字母
}
```

**但实际上**，Java编译器（javac）看到 `+` 号，会把代码**偷偷改成**这样（相当于编译器帮你写了隐藏代码）：
```java
String str = "A";
for (int i = 0; i < 1000; i++) {
    // 编译器偷偷创建了一个临时的 StringBuilder（变相等于活页本）
    StringBuilder sb = new StringBuilder();
    sb.append(str);   // 把旧的 "A... " 全部拷进临时本
    sb.append("a");   // 把新字母放进去
    str = sb.toString(); // 调用 toString()，重新生成一个【新的String】对象
}
```

#### 关键致命伤（慢的根源）：
1. **每循环一次，就 `new` 一个 `StringBuilder` 对象**（循环1000次，就new 1000个临时对象，造成内存垃圾）。
2. **`append(str)` 会把原字符串的所有字符全部拷贝一遍**。
3. **`toString()` 又会把全部字符拷贝一遍，生成新字符串**。

**数学公式**：
第一次拷贝 1 个字符，第二次拷贝 2 个字符，第三次拷贝 3 个字符......
总拷贝量 = `1 + 2 + 3 + ... + N = N(N+1)/2`。

> **结论**：当 N=10000 时，字符串总长度只有 10000，但底层**偷偷拷贝了 5000万次字符**！这就是**平方级（O(n²)）**的恐怖开销。CPU 全在干重复的“搬砖”活，能不慢吗？

---

### 3. 内存爆炸（GC垃圾回收）的负担

每一次循环，旧的 `String` 对象就被丢弃了（因为 `str` 指向了新对象）。
循环 10000 次，堆内存里就会产生 **10000 个废弃的字符串对象**。Java 的垃圾回收器（GC）就得不停地打扫这些垃圾，打扫的时候会**暂停你的程序**，这又进一步拖慢了速度。

---

### 4. 代码实战：到底慢多少？（让你惊掉下巴）

我们来做个实验，用代码感受一下**肉眼可见**的差距：

```java
public class TestSpeed {
    public static void main(String[] args) {
        int n = 50000; // 拼接5万次
        
        // ----- 方法1：用 String（慢如蜗牛） -----
        long start1 = System.currentTimeMillis();
        String str = "";
        for (int i = 0; i < n; i++) {
            str = str + "a"; 
        }
        long end1 = System.currentTimeMillis();
        System.out.println("String 耗时：" + (end1 - start1) + " 毫秒");
        // 在我的电脑上输出：String 耗时：15291 毫秒（15秒！卡死了）


        // ----- 方法2：用 StringBuilder（快如闪电） -----
        long start2 = System.currentTimeMillis();
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < n; i++) {
            sb.append("a"); // 直接在后面追加，不产生垃圾
        }
        String result = sb.toString();
        long end2 = System.currentTimeMillis();
        System.out.println("StringBuilder 耗时：" + (end2 - start2) + " 毫秒");
        // 我的电脑输出：StringBuilder 耗时：2 毫秒（几乎感觉不到！）
    }
}
```
> **差距接近 1万倍！** 这就是原理导致的性能鸿沟。

---

### 5. 终极总结（背下来，面试必问！）

| 场景 | 该用什么 | 原因 |
| :--- | :--- | :--- |
| **循环拼接**（for里拼字符串） | **必须用 `StringBuilder`** | 防止反复拷贝全量数据 |
| **单次拼接**（`String a = "A"+"B"`） | 直接用 `+` 或 `String` | 编译器会优化成一次 `new`，没性能问题 |
| **多线程环境拼接** | `StringBuffer`（线程安全，但稍慢） | 底层原理和 `StringBuilder` 一样，只是加了锁 |

---

### 6. 给你的“肌肉记忆”口诀

以后写代码，**只要看到 `for` 循环里面有 `字符串 +=`**，立刻条件反射：

> **“删除它！改写成 `StringBuilder`！”**

**标准写法模板**（直接背）：
```java
// 自动挡写法（固定套路）
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 100; i++) {
    sb.append(i).append(","); // 随便拼
}
String finalStr = sb.toString(); // 最后一次性生成
```

现在你彻底明白了吧？**String 不可变**是它的核心特性（安全、线程安全），但这个优点在循环里就成了**性能杀手**。所以，**把灵活性交给 `StringBuilder`，把最终结果交给 `String`**！

如果对 `StringBuilder` 和 `StringBuffer` 的区别（线程安全）还想深挖，或者想看看内存里的具体图解，随时喊我！😄


# 公共异常和自定义异常

### 1. 先来个生活大比喻（生病与医生）

- **异常（Exception）**：就是程序运行时遇到的“突发状况”。比如身体突然发烧（数组越界）、突然没吃饭（空指针）。
- **公共异常（内置异常）**：就像**感冒、发烧**——这是全人类都会得的常见病。Java已经帮你写好了现成的“病历本”（类），你直接用就行。
- **自定义异常**：就像**“张三的专属恐高症”**——这不是每个人都有的，是你业务逻辑里特有的。Java不知道你的业务规则，所以你得**自己写一个“病历本”**，专门描述这种病。

---

### 2. 先看家族谱系（Java异常体系，简化版）

0基础只需要知道这两大类：

```text
Throwable (最大的祖宗)
   ├── Error (天灾：内存爆了、系统崩了。程序员救不了，忽略不管)
   └── Exception (人祸：程序员的逻辑瑕疵。我们要处理的)
          ├── 运行时异常 (RuntimeException)：编译时不报错，运行才崩。比如 1/0
          └── 受检异常 (非RuntimeException)：编译时就强迫你处理。比如 文件找不到
```

> **0基础口诀**：**我们只关心 Exception 这一支，特别是 RuntimeException（运行时异常）。**

---

### 3. 第一部分：公共异常（Java自带的“常用病历本”）

这些都是Java官方写好的，你**不需要自己定义**，直接用。它们是开发中最常见的“病”。

| 异常名字 (病历本) | 什么时候会得这个病？ (触发场景) | 代码演示 |
| :--- | :--- | :--- |
| **`NullPointerException`** <br>（空指针，最出名的老大） | 你想指挥一个“不存在的人”干活。 | `String s = null; s.length();` <br>（你要测一个空人的身高，直接晕倒） |
| **`ArrayIndexOutOfBoundsException`** <br>（数组越界） | 你有个10个座位（0~9），非要去坐第11个。 | `int[] arr = new int[3]; arr[3] = 1;` |
| **`ArithmeticException`** <br>（算术异常） | 把数字除以 0。 | `int a = 10 / 0;` |
| **`ClassCastException`** <br>（类型转换异常） | 硬说“苹果是香蕉”，强行转换。 | 把猫强转成狗 |
| **`NumberFormatException`** <br>（数字格式异常） | 把字母 “ABC” 转换成数字。 | `Integer.parseInt("abc");` |

> **怎么处理它们？** 这些是运行时异常，一般靠**规范代码**避免（比如加 `if (s != null)` 判断）。Java不强迫你捕获它们。

---

### 4. 第二部分：自定义异常（自己写的“专属病历本”）

#### 为什么要有自定义异常？
因为业务规则千奇百怪。比如银行系统：
- 用户取钱，余额不够了。Java内置异常里有“除以零”，但没有 **“余额不足异常”**。
- 用户注册，年龄填了负数。Java内置异常里没有 **“年龄非法异常”**。

这时候，**你必须自己写一个类**去继承 `Exception` 或 `RuntimeException`。

#### 手把手教：写一个自定义异常（只需2步）

**第1步：写一个类，继承 `Exception`（或 `RuntimeException`）**

```java
// 自定义一个“余额不足”的专属异常
// 继承 RuntimeException 的好处是：不用在方法上强制写 throws（更灵活）
class InsufficientFundsException extends RuntimeException {
    
    // 1. 无参构造（默认）
    public InsufficientFundsException() {
        super(); // 调用父类的构造
    }
    
    // 2. 带错误信息的构造（强烈推荐写这个！）
    public InsufficientFundsException(String message) {
        super(message); // 把异常描述传给父类，方便打印
    }
}
```

**第2步：在业务代码里“抛出”这个异常（使用 `throw` 关键字）**

```java
public class BankAccount {
    private double balance = 100.0; // 余额100块

    public void withdraw(double amount) {
        // 业务判断：如果取的钱 > 余额
        if (amount > balance) {
            // 抛出（throw）我刚刚写的自定义异常！
            // 注意：这里用的是 throw（动作），而不是 throws（声明）
            throw new InsufficientFundsException("余额不足！当前余额：" + balance + "，想取：" + amount);
        }
        // 如果够钱，就减掉
        balance -= amount;
        System.out.println("取款成功，剩余：" + balance);
    }

    public static void main(String[] args) {
        BankAccount account = new BankAccount();
        account.withdraw(200); // 运行到这里，程序直接崩掉，并打印你自定义的错误信息！
        // 控制台输出：Exception in thread "main" InsufficientFundsException: 余额不足！当前余额：100.0，想取：200.0
    }
}
```

> **0基础必看语法区分**：
> - **`throw`** （动词）：**扔**出去。放在方法体里面，后面跟**异常对象**。
> - **`throws`** （名词复数标记）：**声明**我会扔。放在方法签名括号后面，后面跟**异常类名**。

---

### 5. 为什么我的自定义异常要继承 `RuntimeException` vs `Exception`？

这是面试高频题，决策树如下：

- **继承 `RuntimeException`（非受检）**：
  - 调用你方法的人，**可以不写** `try-catch`，程序会直接崩。
  - **适用场景**：**代码逻辑错误，理论上不应该发生**（比如传了个负数年龄）。你希望调用者传参时注意，而不是包一层 `try`。
- **继承 `Exception`（受检）**：
  - 调用你方法的人，**必须**写 `try-catch` 或在方法上写 `throws`，不然编译报错。
  - **适用场景**：**外部环境导致的问题，调用者必须处理**（比如网络断了、文件找不到）。你强迫调用者去“兜底”。

> **业界大牛建议（阿里开发手册）**：**绝大多数业务场景，推荐继承 `RuntimeException`**。这样写代码更干净，不用到处 `try-catch` 污染业务逻辑。

---

### 6. 捕获 vs 抛出（处理异常的两种姿势）

有了异常之后，我们有两条路走：

| 姿势 | 关键词 | 比喻 | 使用场景 |
| :--- | :--- | :--- | :--- |
| **捕获（自己治）** | `try-catch` | 自己感冒了，去药店买药吃，程序继续运行下去。 | 出错了还能补救（比如取钱失败，提示用户“余额不足”，界面不崩）。 |
| **抛出（向上甩锅）** | `throws` / `throw` | 自己治不了，把病历单甩给上级（调用者）处理。 | 底层发生致命问题，让顶层统一处理（比如让Spring全局异常拦截）。 |

**捕获的模板写法（加固一下）**：
```java
try {
    account.withdraw(200); // 这里会抛出异常
} catch (InsufficientFundsException e) {
    // 走到这里表示异常被抓住了，程序不会崩
    System.out.println("捕获到了：" + e.getMessage()); 
    // 在这里可以做善后：比如弹窗提示用户“钱不够”
}
```

---

### 7. 终极总结（0基础大脑存档区）

1. **公共异常**：Java自带，比如空指针、越界，直接认脸就行。
2. **自定义异常**：**必须自己写**，继承 `RuntimeException`，写上带 `message` 的构造方法。
3. **核心动作**：
   - 创建：`new 自定义异常("错误描述")`
   - 扔出去：`throw new 自定义异常("错误描述")`
4. **铁律**：**异常是善意的提醒**。千万不要捕获异常后 `e.printStackTrace()` 打印完就什么事都不做，那是掩耳盗铃！一定要给用户明确的反馈。

---

# 文件数据流



### 2. 两大分类：字节流 vs 字符流（选哪个？）

| 分类 | 管道的“粗细” | 处理什么文件？ | 核心代表类 |
| :--- | :--- | :--- | :--- |
| **字节流** | 最底层，按 **8位二进制** 传输 | **万能**：图片、视频、音频、Word、Excel、文本（任何文件） | `InputStream` / `OutputStream` |
| **字符流** | 按 **单个字符（如中文）** 传输 | **专门**：纯文本文件（`.txt`，`.csv`，`.json`）| `Reader` / `Writer` |

> **0基础选择指南**：**读/写 文本文件（比如记事本）用字符流；复制图片/视频用字节流。**

---

### 3. 终极偷懒大法：JDK 1.7+ 最现代写法（`Files` 工具类）

> **警告**：很多老教程会让你写几十行 `FileInputStream` + `BufferedReader` + 循环 + `finally` 关流。**太复杂了！**  
> 从 Java 7 开始，官方提供了超级简洁的工具类 `java.nio.file.Files`。**0基础请先从这里入手！**

#### 场景一：一口气把整个文本文件读进内存（最常用）
假设桌面上有个 `test.txt`，内容是一首诗。
```java
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.List;

public class Test {
    public static void main(String[] args) throws Exception {
        // 1. 读取所有行，放进 List<String> 容器里（一行就是一个字符串）
        List<String> lines = Files.readAllLines(Paths.get("C:/Users/你的用户名/Desktop/test.txt"));
        
        // 2. 遍历打印
        for (String line : lines) {
            System.out.println(line);
        }
    }
}
```
> **解读**：`Paths.get("文件路径")` 就是告诉水管工去哪抽水。`readAllLines` 一下子把所有水都抽到内存的 `List` 里。**超简单，一行搞定！**

#### 场景二：把字符串直接写入文件
```java
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.Arrays;
import java.util.List;

public class Test {
    public static void main(String[] args) throws Exception {
        List<String> content = Arrays.asList("第一行", "第二行", "第三行");
        
        // 直接写进去！如果文件不存在，会自动创建
        Files.write(Paths.get("C:/Users/你的用户名/Desktop/output.txt"), content);
        System.out.println("写入成功！");
    }
}
```
> **注意**：`Files.write` 默认会**覆盖**旧内容。如果想**追加（续写）**，需要加参数，我们暂不深入。

---

### 4. 传统但高级的写法：`BufferedReader` / `BufferedWriter`（处理超大文件）

如果你要读一个**10GB**的日志文件，用上面的 `readAllLines` 会把内存撑爆（因为一下子全装进来）。这时候要用水管**一点点流**进来——这就是 `BufferedReader`（带缓冲的水管，效率高）。

#### 一边读一边处理（不占内存）
```java
import java.io.BufferedReader;
import java.io.FileReader;
import java.io.IOException;

public class Test {
    public static void main(String[] args) {
        // 使用 try-with-resources（后面讲，保证自动关水管）
        try (BufferedReader br = new BufferedReader(new FileReader("test.txt"))) {
            String line;
            // 一次读一行，读到末尾返回 null
            while ((line = br.readLine()) != null) {
                System.out.println("当前行内容：" + line);
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

#### 一边写一边生成（不占内存）
```java
import java.io.BufferedWriter;
import java.io.FileWriter;
import java.io.IOException;

public class Test {
    public static void main(String[] args) {
        // 第二个参数 true 表示【追加】模式，不传或 false 表示覆盖
        try (BufferedWriter bw = new BufferedWriter(new FileWriter("test.txt", true))) {
            bw.write("这是新增的一行");
            bw.newLine(); // 换行
            bw.write("这是新增的第二行");
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

---

### 5. 必学避坑：复制图片/视频（字节流实战）

如果复制图片，千万不能用 `Reader`（字符流），否则图片会损坏变花屏。必须用**字节流**。

```java
import java.nio.file.Files;
import java.nio.file.Paths;
import java.io.IOException;

public class Test {
    public static void main(String[] args) throws IOException {
        // 读取图片的所有字节 -> 写到新文件
        byte[] data = Files.readAllBytes(Paths.get("C:/photo.jpg"));
        Files.write(Paths.get("C:/copy_photo.jpg"), data);
        System.out.println("图片复制成功！");
    }
}
```
> 看！哪怕是复制图片，用 `Files` 工具类依然**一行读，一行写**，极其简单。

---

### 6. 最重要的保命原则：必须关水管！（`try-with-resources`）

**为什么必须关？** 
水管（流）打开后，会一直占用系统资源。如果不关，**文件会被锁定**，你想用电脑删除这个文件都删不掉，程序运行久了内存会泄露导致卡死。

**老式的写法**（极其繁琐，淘汰）：
```java
BufferedReader br = null;
try {
    br = new BufferedReader(...);
    // ...
} finally {
    if (br != null) br.close(); // 还要套一层 try-catch，恶心至极
}
```

**现代的写法（0基础必须背下来）**：
```java
// 把 创建流的代码 放在 try 后面的小括号 () 里！
// Java 会自动帮你在 try 代码块结束后调用 .close() 关闭水管
try (BufferedReader br = new BufferedReader(new FileReader("test.txt"))) {
    // 你的业务代码
} catch (IOException e) {
    // 处理异常
}
// 执行到这里，br 已经被自动关闭了，不需要写 finally！
```
> **铁律**：凡是实现了 `AutoCloseable` 接口的资源（流、数据库连接），**一律**用 `try-with-resources` 包裹。

---

### 7. 路径的坑（0基础必看）

- **相对路径**：`"test.txt"` 指的是你的**项目根目录**下的文件（如果你在用 IDEA，是项目文件夹下，不是 `src` 里面！）。
- **绝对路径**：`"C:/data/test.txt"`（Windows）或 `"/home/user/test.txt"`（Linux）。

> **新手救急**：如果搞不清相对路径，**先用绝对路径**（比如 `D:/my.txt`），确保程序先跑通，后期再学相对路径。

---

### 8. 终极总结（大脑内存卡）

| 你的需求 | 首选代码（一句话方案） | 注意点 |
| :--- | :--- | :--- |
| **读小文本**（配置文件） | `List<String> lines = Files.readAllLines(Paths.get("路径"));` | 适合几百KB的文件 |
| **写小文本** | `Files.write(Paths.get("路径"), list);` | 默认覆盖，注意备份 |
| **读大文本**（逐行处理） | `try (BufferedReader br = new BufferedReader(new FileReader("路径")))` | 省内存，速度还快 |
| **写大文本**（逐行追加） | `try (BufferedWriter bw = new BufferedWriter(new FileWriter("路径", true)))` | `true` 代表追加模式 |
| **复制图片/视频** | `byte[] d = Files.readAllBytes(path); Files.write(newPath, d);` | 记得用字节数组 |

---
# 字节流和字符流



### 2. 一句话核心区别（背下来！）

| 对比维度 | **字节流（Byte Stream）** | **字符流（Character Stream）** |
| :--- | :--- | :--- |
| **操作单位** | 以 **字节（8位二进制）** 为单位 | 以 **字符（如 'A' 或 '中'）** 为单位 |
| **核心父类** | `InputStream`（输入） / `OutputStream`（输出） | `Reader`（输入） / `Writer`（输出） |
| **是否有编码表** | **没有**！纯粹搬运0和1 | **有**！依赖字符集（如 UTF-8, GBK）解码 |
| **适用场景** | **万能**：图片、音频、视频、压缩包、任何文件 | **专用**：仅限 `.txt`, `.java`, `.html`, `.json` 等文本文件 |
| **是否缓冲** | 一般不带（除非包装） | 强烈建议配合 `BufferedReader` / `BufferedWriter` 使用（提高效率） |

---

### 3. 解剖底层原理（为什么字符流能看懂中文？）

这就涉及到 **“编码/解码”** 的奥秘：

- **硬盘里存的只有 0 和 1**。比如中文 `中` 在 UTF-8 编码下存的是 `11100100 10111000 10101101`（3个字节）。
- **字节流**：读进来就是 `11100100 10111000 10101101`，直接丢给你，不管它是什么意思。
- **字符流**：读进来后，会拿着这张**编码表（如 UTF-8 对照表）**去查表，发现这串二进制对应的是 `中`，于是把 `中` 这个字符放进内存。

> **致命误区**：如果你用 **字节流** 去读文本文件，然后用 `System.out.println` 打印，大概率会输出乱码（因为你没查表）；如果你用 **字符流** 去复制图片，图片会直接损坏变花屏（因为字符流强行把二进制当文字解码，破坏了原始数据）。

---

### 4. 代码实战（用最直观的代码演示区别）

#### 场景一：用字节流复制图片（正确用法）
任何文件（包括图片）只能用字节流，因为不需要看懂内容。

```java
import java.nio.file.Files;
import java.nio.file.Paths;

public class TestByte {
    public static void main(String[] args) throws Exception {
        // 字节流操作：直接 copy 二进制数据
        Files.write(Paths.get("copy.jpg"), Files.readAllBytes(Paths.get("photo.jpg")));
        System.out.println("图片复制成功，无损！");
    }
}
```

#### 场景二：用字符流读取文本（正确用法）
读文本必须用字符流，直接拿到人类语言。

```java
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.List;

public class TestChar {
    public static void main(String[] args) throws Exception {
        // 字符流操作（Files 底层默认是字符流 + UTF-8 解码）
        List<String> lines = Files.readAllLines(Paths.get("test.txt"));
        for (String line : lines) {
            System.out.println("读到了文字：" + line); // 正常显示中文
        }
    }
}
```

#### 场景三：犯傻演示（用字节流读中文会怎样？）
如果用最基本的 `FileInputStream`（字节流）去读中文文件：

```java
import java.io.FileInputStream;

public class TestWrong {
    public static void main(String[] args) throws Exception {
        FileInputStream fis = new FileInputStream("test.txt"); // 假设里面有 "你好"
        int data;
        while ((data = fis.read()) != -1) {
            System.out.print(data + " "); 
            // 输出一堆数字：228 189 160 229 165 189 （这是二进制的十进制表现，看着像天书）
        }
        fis.close();
    }
}
```
> 这就是字节流的本质——它眼里只有 **228, 189, 160** 这堆数字，不认识“你好”。

---

### 5. 至关重要的“桥接”知识（转换流）

**实际开发中，我们有时候会陷入两难境地**：
- 我给了一个字节流（比如从网络下载的数据）。
- 但是我想把它当作文本，用字符流的方式读中文，怎么办？

这时候需要 **“桥梁”** —— `InputStreamReader`（字节转字符）和 `OutputStreamWriter`（字符转字节）。

> **用途**：当你手里只有字节流（`InputStream`），但想按字符读取时，套上这个转换流，就能**顺便指定编码**。

```java
import java.io.*;

public class TestBridge {
    public static void main(String[] args) throws Exception {
        // 1. 底层的字节流（水管）
        FileInputStream fis = new FileInputStream("test.txt");
        
        // 2. 套上转换流，变成字符流，并明确告诉它用 UTF-8 解码（防止乱码！）
        InputStreamReader isr = new InputStreamReader(fis, "UTF-8");
        
        // 3. 再套上缓冲流，提高效率（按行读）
        BufferedReader br = new BufferedReader(isr);
        
        String line = br.readLine();
        System.out.println(line); // 完美显示中文
        
        br.close();
    }
}
```
> **0基础核心口诀**：**文件（字节） → 转换桥 → 字符流（看懂文字）**。

---

### 6. 终极选型决策树（别再纠结了！）

在 IDEA 里写代码时，按这个顺序问自己：

1. **我要处理的是图片、视频、音频、压缩包吗？**  
   👉 **是** → 用 **字节流**（`Files.readAllBytes` / `FileInputStream`）。

2. **我要处理的是纯文本（.txt/.java/.json）吗？**  
   👉 **是** → 再看下一步。

3. **这个文本文件大不大？（比如 > 100MB）**  
   - **大** → 用字符流逐行读取（`BufferedReader` + `FileReader`）。
   - **小** → 直接 `Files.readAllLines`（底层自动帮你用了字符流）。

4. **我读中文出现了乱码？**  
   👉 **立刻**在代码里指定编码：`new InputStreamReader(new FileInputStream("文件"), StandardCharsets.UTF_8)`（Java 7+ 写法）。

---

### 7. 面试爱问的超级陷阱（你以后肯定会遇到）

> **问**：Java 中一个字符（char）占 2 个字节，为什么字符流读中文 `中` 却占了 3 个字节？
> 
> **答**：因为 **JVM 内存里的 char 是 2 字节（Unicode）**，但**硬盘/网络传输中的字符流**要按指定的编码（如 UTF-8）进行转换。UTF-8 编码下，一个中文占 3 个字节。字符流的作用就是在 **硬盘的 3 字节** 和 **内存的 2 字节** 之间来回翻译。

---


# 序列化


### 1. 先来生活比喻（快递打包与拆包）

想象一下：

- **你家的房子（JVM内存）** 里有一个**精美的手办（Java对象）**，它有胳膊、腿、颜色（属性值）。
- 你想把这个手办**寄给（保存到硬盘）** 远方的朋友，或者**通过网络传给**另一台电脑。

**问题来了**：快递小哥（网络/硬盘）不认识“手办”这种东西，他只认**扁平的纸箱（二进制字节流）**。

- **序列化（Serialization）** = **打包**。你把会散架的手办拆解成零件，按说明书整齐地码放进纸箱（变成一串 `010101` 字节）。
- **反序列化（Deserialization）** = **拆包**。朋友收到纸箱后，按照说明书把零件重新拼回一个完整的手办（从字节变回Java对象）。

> **官方定义**：序列化是**把Java对象转换成字节序列**的过程；反序列化是**把字节序列恢复成Java对象**的过程。

---

### 2. Java里怎么实现？只需要做一件事（标记）

在Java里，想让一个对象能被“打包”，代码简单得令人发指：**只要让你的类实现 `Serializable` 接口**。

> **0基础吃惊**：这个接口里面是**空的**！没有任何方法需要你重写！
> 它就像一个**“合格证”标签**或**“易碎品”贴纸**。贴上这个标签，JVM就知道：“哦，这个类允许被打包寄走。”如果不贴，强行打包会直接报错 `NotSerializableException`。

```java
import java.io.Serializable;

// 1. 定义一个用户类，贴上 Serializable 标签（合格证）
class User implements Serializable {
    // 注意：这个 id 下面会详细讲，先照抄
    private static final long serialVersionUID = 1L;
    
    public String name;
    public int age;
    
    public User(String name, int age) {
        this.name = name;
        this.age = age;
    }
}
```

---

### 3. 真正动手：打包（序列化）和拆包（反序列化）

我们使用两个“快递员”工具：`ObjectOutputStream`（打包员）和 `ObjectInputStream`（拆包员）。

```java
import java.io.*;

public class TestSerial {
    public static void main(String[] args) throws Exception {
        String path = "user.obj"; // 存文件的路径，后缀随便起
        
        // ----- 场景一：打包（把对象写到硬盘文件里） -----
        User user = new User("张三", 20);
        try (ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream(path))) {
            oos.writeObject(user); // 核心方法！把整个对象压成字节流写进去
            System.out.println("序列化成功，对象已打包到文件！");
        }
        
        // ----- 场景二：拆包（从硬盘文件里读出对象） -----
        try (ObjectInputStream ois = new ObjectInputStream(new FileInputStream(path))) {
            // 核心方法！把字节流恢复成对象（注意要强制转型）
            User savedUser = (User) ois.readObject();
            System.out.println("反序列化成功！");
            System.out.println("名字：" + savedUser.name); // 输出：张三
            System.out.println("年龄：" + savedUser.age);  // 输出：20
        }
    }
}
```
> **运行结果**：你的项目目录下会生成一个 `user.obj` 文件。虽然你用记事本打开它是乱码，但Java能完美地把它还原成对象！

---

### 4. 超级重点：`serialVersionUID`（版本指纹）

我们在上面写了 `private static final long serialVersionUID = 1L;`。**这玩意是干什么的？**

> **比喻**：假设你给朋友寄了一个**变形金刚（版本1.0）**。在快递路上，你突然觉得变形金刚的胳膊不好看，在家里把图纸改了（类代码改了，加了新属性）。
> 朋友收到快递后，拿着**新图纸（修改后的类）**去拼**旧零件（序列化数据）**，发现少了个胳膊，立刻**报错崩溃**（`InvalidClassException`）！

**`serialVersionUID` 就是“图纸版本号”**：
- 序列化时，会把版本号写入纸箱。
- 反序列化时，会比对当前类的版本号和纸箱里的版本号。
- **如果不写**：JVM会自动算一个哈希值当版本号。一旦你改了类（哪怕加个空格），哈希值就变了，导致新旧不兼容。
- **如果写了且固定**：即使你加了新属性，版本号一样，JVM就会尽量兼容（新属性赋默认值，比如 `null` 或 `0`），**程序就不会崩**。

> **0基础铁律**：**所有实现 `Serializable` 的类，都必须写 `private static final long serialVersionUID = 1L;`**。这是阿里开发手册的强制规定！

---

### 5. 核心关键字：`transient`（瞬态）

有时候，对象里有一些**敏感信息**（比如密码、银行卡号），或者一些**不需要保存**的临时数据（比如缓存计数器）。你不想把这些也打包进纸箱。

这时候用 `transient` 关键字修饰：

```java
class User implements Serializable {
    private static final long serialVersionUID = 1L;
    public String name;
    public transient String password; // 加上 transient，表示“不打包”
    public int age;
}

// 序列化后，password 不会被保存到文件
// 反序列化后，password 的值是 null（默认值）
```

> **比喻**：你寄快递时，手办上贴着一张写有密码的便利贴。打包前你把它撕掉（`transient`），等朋友收到后重新贴一张新的上去。

---

### 6. 新手最容易踩的坑（引用对象也要打包！）

如果 `User` 类里有一个 `Address` 类型的属性，那么 `Address` **也必须实现 `Serializable`**！否则报错。

```java
class Address implements Serializable { // 必须加！否则 User 序列化失败
    private static final long serialVersionUID = 1L;
    String city;
}

class User implements Serializable {
    private static final long serialVersionUID = 1L;
    String name;
    Address address; // 引用类型，Address 也必须是 Serializable
}
```
> **原理**：序列化会**递归**遍历对象中的所有引用，把所有关联对象都打包。但凡有一个没贴合格证，整个快递流程直接失败。

---

### 7. 什么时候用序列化？（实际开发场景）

| 场景 | 你在做什么？ |
| :--- | :--- |
| **对象保存到本地文件** | 游戏存档（保存角色等级、装备） |
| **Redis 缓存** | 把Java对象存进缓存，下次取出来直接是对象 |
| **Session 会话管理** | 把用户登录信息序列化存到服务器内存或硬盘 |
| **RPC远程调用** | 把 `User` 对象通过网络传给另一台服务器（Dubbo/gRPC底层） |

---

### 8. 终极总结（0基础大脑存档）

1. **序列化** = 对象 → 二进制流（打包快递）。
2. **反序列化** = 二进制流 → 对象（拆包收货）。
3. **怎么实现**：类实现 `Serializable` 接口（空标签）。
4. **必须写**：`private static final long serialVersionUID = 1L;`（防版本冲突）。
5. **忽略敏感字段**：加 `transient` 关键字，该字段不参与序列化。
6. **引用对象**：关联的所有类也必须是 `Serializable`。

---

# 管道

这个问题问得非常深入！作为0基础，你能一步步追问到“管道”，说明你的知识网络正在形成。在其他编程语言里，“管道”通常指进程间通信，但在 **Java原生I/O** 里，它特指 **`PipedInputStream` / `PipedOutputStream`（字节）** 和 **`PipedReader` / `PipedWriter`（字符）**。

老规矩，我们用**生活比喻 + 可视化代码**，一次性搞懂这个“极简版消息队列”。

---

### 1. 先来生活比喻（传声筒与水管）

想象你和你的同桌在**安静的图书馆**里：

- 你想偷偷告诉同桌一句话（**数据**），但不能大声喊。
- 你们在桌子底下拉了一根**软管（管道）**。
- 你对着软管这头**轻轻说（写入）**，同桌把耳朵贴在软管那头**听（读取）**。

**核心规则来了（非常重要）**：
1. **单向流动**：你只能往这头写，同桌只能从另一头读。如果同桌想回话，必须**再拉一根新管子**。
2. **必须连通**：管子没接好（没连接），话传不过去，直接报错。
3. **实时同步**：你说一个字，同桌立刻听到一个字。如果你说得太快，管子（内部缓冲区）塞满了，你就得**闭嘴等待（线程阻塞）**，等同桌听完几个字腾出空间，你才能继续往下说。
4. **唯一用途**：Java的管道专门用于**同一个程序内，两个不同线程（两个工人）** 之间的数据传递。

---

### 2. 代码实战：亲手搭建一根“传声筒”

我们用字符流 `PipedWriter`（写）和 `PipedReader`（读）来写一个**“生产者和消费者”**的经典例子。

```java
import java.io.PipedWriter;
import java.io.PipedReader;
import java.io.IOException;

public class TestPipe {

    public static void main(String[] args) throws Exception {
        // 1. 准备一根管子的两端（一头进，一头出）
        PipedWriter writer = new PipedWriter(); // 你嘴边的说话口
        PipedReader reader = new PipedReader(); // 同桌耳朵边的听筒口

        // 2. 必须把两端连接起来！(相当于把管子接上)
        // 注意：连接操作必须在读写之前完成，且只能连接一次
        writer.connect(reader); 
        // 或者用 reader.connect(writer); 效果一样

        // 3. 创建“说话的人（生产者）”线程
        Thread producer = new Thread(() -> {
            try {
                String message = "你好呀，这是通过管道传过来的消息！";
                System.out.println("[生产者] 开始往管子里说话...");
                writer.write(message); // 把整句话写进管子
                writer.close(); // 说完了，把嘴边的口封住（告诉对方没话说了）
                System.out.println("[生产者] 说完了，封口。");
            } catch (IOException e) {
                e.printStackTrace();
            }
        });

        // 4. 创建“听电话的人（消费者）”线程
        Thread consumer = new Thread(() -> {
            try {
                System.out.println("[消费者] 正在耳朵贴着管子听...");
                int data;
                // 循环读取，每次读一个字符，读到 -1 表示对方挂断了（close了）
                while ((data = reader.read()) != -1) {
                    // 把读到的数字转成 char 打印出来
                    System.out.print((char) data); 
                }
                System.out.println("\n[消费者] 听完了，对方挂断了。");
                reader.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        });

        // 5. 启动两个线程（注意：必须消费者先启动，或者生产者启动后稍微等一下）
        // 如果生产者启动太快，写完直接关闭管道，消费者可能读不到数据。
        // 更稳妥的做法：先启动消费者，再启动生产者。
        consumer.start();
        // 给消费者一点点启动时间（实际开发中用同步工具，这里为了演示加个sleep）
        Thread.sleep(100); 
        producer.start();
    }
}
```

**运行结果（控制台输出）**：
```
[消费者] 正在耳朵贴着管子听...
[生产者] 开始往管子里说话...
你好呀，这是通过管道传过来的消息！
[生产者] 说完了，封口。
[消费者] 听完了，对方挂断了。
```
> **注意看时序**：生产者写入的“你好呀...”，消费者在另一端完美地一个字一个字打印了出来。

---

### 3. 管道的致命弱点（面试常问）

既然管道这么好用，为什么**实际项目里很少直接用 `PipedReader`/`PipedWriter`** 呢？

| 管道的天生缺陷 | 解释（0基础版） |
| :--- | :--- |
| **缓冲区太小（默认只有1024字节）** | 生产者如果一口气写100万个字，管道会瞬间塞满。生产者必须停下来等消费者慢慢读，效率极低。 |
| **极易造成“死锁”** | 如果生产者和消费者**是同一个线程**（或者两个线程互相等待），生产者写完等着读，消费者读不着等着写，程序直接永久卡死。 |
| **异常处理麻烦** | 如果生产者写的时候抛异常，忘记调用 `close()`，消费者会永远傻乎乎地等下去，程序不会崩溃，但一直卡着。 |

---

### 4. 既然是鸡肋，为什么还要学它？（实际应用场景）

虽然日常CRUD不用它，但它在以下**两个底层领域**是绝对主角：

1. **进程间的标准输入输出（`ProcessBuilder`）**：
   当你的Java程序用 `Runtime.getRuntime().exec("ping baidu.com")` 调用操作系统的外部命令时，你需要获取命令行的输出结果。这时候，`Process` 对象会返回 `getInputStream()`，它本质上就是一根管道，把 `ping` 命令打印的内容“传”给你的Java程序。

2. **两个线程间极轻量级的即时通信**：
   如果你有两个线程，一个产生数据，一个消费数据，且数据量极小，不想引入重量级的MQ（消息队列），可以临时用管道。但**现实开发中，我们更推荐用 `BlockingQueue`（阻塞队列）**，因为它更安全、容量可控，且支持多对多。

---

### 5. 字节管道 vs 字符管道（快速分清）

和之前学的I/O分类一模一样：

| 管道类型 | 父类 | 传输单位 | 适用场景 |
| :--- | :--- | :--- | :--- |
| **字节管道** | `PipedInputStream` / `PipedOutputStream` | 字节 (byte) | 传递图片、视频、序列化后的对象等任意数据。 |
| **字符管道** | `PipedReader` / `PipedWriter` | 字符 (char) | **仅限**传递纯文本（中文、英文）。 |

> **0基础记住**：大多数情况下，如果你只在两个线程间传文字，用 **`PipedWriter` / `PipedReader`**（字符管道）最方便，不用操心编码问题。

---

### 6. 0基础避坑大总结（背下来保平安）

1. **必须先 `connect()`**，忘了连接直接报 `IOException`。
2. **读写要在两个线程里**，绝对**不能**在同一个线程中 `writer` 写完了立刻 `reader.read()`，百分百死锁。
3. **记得 `close()`**：生产者写完后一定要 `writer.close()`，这样消费者读到末尾才会返回 `-1`，否则 `reader.read()` 会永远阻塞在那里，程序退不出。
4. **报错 `Pipe broken`（管道破裂）**：这是因为你关闭了读取端，但写入端还在拼命写数据。就像你同桌把听筒摔了，你还在对着管子里喊，管子就破了。

---

### 7. 终极对比：管道 vs 文件（之前的I/O）

| 对比维度 | **文件流 (File I/O)** | **管道流 (Pipe I/O)** |
| :--- | :--- | :--- |
| **数据目的地** | 硬盘（持久化存储） | 内存（另一个线程的内存空间） |
| **生命周期** | 关机重启还在 | 程序一关，数据全丢 |
| **速度** | 慢（涉及磁盘读写） | **极快**（纯内存操作） |
| **必须关闭吗？** | 关掉释放系统句柄 | 关掉释放内存缓冲区，**更重要**（否则线程卡死） |

---


# 文件处理

## File类使用


### 2. 创建 File 对象（指哪打哪）

创建 `File` 对象非常简单，**不涉及硬盘操作，只是new一个对象**：

```java
import java.io.File;

public class TestFile {
    public static void main(String[] args) {
        // 方式1：绝对路径（Windows示例，注意用双斜杠 \\ 或 单斜杠 /）
        File file1 = new File("D:/myFolder/test.txt");
        
        // 方式2：相对路径（相对于你的项目根目录）
        File file2 = new File("data/config.properties");
        
        // 方式3：父路径 + 子路径（更规范）
        File parentDir = new File("D:/myFolder");
        File file3 = new File(parentDir, "test.txt");
        
        System.out.println("对象创建完了，但硬盘上不一定有这个文件哦！");
    }
}
```
> **0基础注意**：`new File(...)` 只是**在内存里造了一张“档案卡”**，这时候硬盘上可能根本没有这个文件！

---

### 3. 核心方法大全（背下这7个，日常够用）

我把方法分成**“判断类”**、**“操作类”** 和 **“获取类”**，这样好记。

#### 3.1 判断类（返回 boolean）

| 方法 | 作用 | 生活翻译 |
| :--- | :--- | :--- |
| `file.exists()` | 文件/文件夹存不存在？ | 查一下档案卡，这本书在不在架上？ |
| `file.isFile()` | 是不是一个**文件**？（不是文件夹） | 这是书，不是书架隔板？ |
| `file.isDirectory()` | 是不是一个**文件夹**？ | 这是书架隔板（目录），不是书？ |

#### 3.2 操作类（增删改）

| 方法 | 作用 | 注意点 |
| :--- | :--- | :--- |
| `file.createNewFile()` | **创建空文件** | 返回 true/false。如果父目录不存在，会抛异常！ |
| `file.mkdir()` | 创建**单层**文件夹 | 只能建一层。比如 `a/b`，如果 `a` 不存在会失败。 |
| `file.mkdirs()` | 创建**多层**文件夹（**强烈推荐**）| 不管中间缺几层，一口气全建好！ |
| `file.delete()` | 删除文件或空文件夹 | 删除后进回收站，**直接物理删除**（慎用！） |
| `file.renameTo(newFile)` | 重命名或移动文件 | 跨盘符可能失败 |

#### 3.3 获取类（查信息）

| 方法 | 作用 |
| :--- | :--- |
| `file.getAbsolutePath()` | 获取绝对路径（完整地址） |
| `file.getName()` | 获取文件名（如 `test.txt`） |
| `file.getParent()` | 获取父目录路径 |
| `file.length()` | 获取文件大小（**字节数**，文件夹返回 0） |
| `file.listFiles()` | **遍历子文件/文件夹**（返回 File[] 数组，极常用！） |

---

### 4. 实战代码：把档案卡用起来！

我们来写一个完整的例子，演示从**判断存在** → **创建** → **删除**的全流程。

```java
import java.io.File;
import java.io.IOException;

public class TestFileAction {
    public static void main(String[] args) throws IOException {
        // 1. 创建一张“档案卡”（指向项目根目录下的 test.txt）
        File file = new File("test.txt");
        
        // 2. 判断是否存在
        if (file.exists()) {
            System.out.println("文件已存在，大小：" + file.length() + " 字节");
            System.out.println("绝对路径：" + file.getAbsolutePath());
        } else {
            System.out.println("文件不存在，准备创建...");
            // 3. 创建新文件（注意：要抛异常，因为可能权限不足）
            boolean success = file.createNewFile();
            if (success) {
                System.out.println("创建成功！");
                // 创建完顺便看看绝对路径在哪（方便你去电脑上找）
                System.out.println("位置在：" + file.getAbsolutePath());
            }
        }
        
        // 4. 删掉它（演示完就删，保持项目干净）
        // file.delete(); 
        // System.out.println("已删除");
    }
}
```
> **运行结果**：你会看到项目根目录下凭空多出来一个 `test.txt` 空文件。

---

### 5. 超级重点：遍历文件夹（listFiles 的威力）

实际开发中，我们经常要**列出某个文件夹下的所有文件**，比如扫描 `D:/照片` 里的所有图片。

```java
import java.io.File;

public class TestListFiles {
    public static void main(String[] args) {
        File dir = new File("D:/myFolder"); // 指向一个文件夹
        
        // 1. 先判断是不是文件夹
        if (dir.isDirectory()) {
            // 2. 获取里面的所有子文件和子文件夹（返回 File 数组）
            File[] children = dir.listFiles();
            
            // 3. 如果文件夹是空的，children 为 null，必须判断！
            if (children != null) {
                System.out.println("该文件夹下有 " + children.length + " 个对象");
                for (File child : children) {
                    // 用 isFile 判断是文件还是文件夹
                    if (child.isFile()) {
                        System.out.println("【文件】" + child.getName() + " 大小：" + child.length());
                    } else if (child.isDirectory()) {
                        System.out.println("【文件夹】" + child.getName());
                    }
                }
            }
        } else {
            System.out.println("这不是一个文件夹，或者路径不存在！");
        }
    }
}
```

---

### 6. 0基础最容易踩的坑（保命三连）

#### 坑一：Windows 路径里的反斜杠 `\`
**错误**：`new File("C:\Users\name\test.txt")`（编译报错，因为 `\` 是转义字符）。  
**正确**：`new File("C:/Users/name/test.txt")`（推荐用正斜杠 `/`，Java跨平台兼容）或 `new File("C:\\Users\\name\\test.txt")`（双反斜杠）。

#### 坑二：`exists()` 返回 false，不代表不能创建！
如果路径中的**父目录不存在**，`createNewFile()` 会抛出 `IOException`。  
**解决**：创建文件前，先确保父目录存在：
```java
File file = new File("a/b/c/test.txt");
// 获取父目录，一口气创建多层父目录！
file.getParentFile().mkdirs(); 
// 父目录建好了，再创建文件
file.createNewFile();
```

#### 坑三：`delete()` 只能删空文件夹
如果你想删一个里面有子文件的文件夹，直接 `dir.delete()` 会返回 `false`（删不掉）。  
**解决**：必须用递归先删里面的文件，再删自己（或者用第三方工具如 Apache Commons IO）。

---

### 7. 终极解惑：`File` 和 `Files`（带 s）是什么关系？

你之前在学“现代I/O”时，我用过 `Files.readAllLines()`，这里有个**新老 API 交替**的概念：

| 对比      | **`java.io.File`（老）**                                                                                   | **`java.nio.file.Files`（新，Java 7+）**         |
| :------ | :------------------------------------------------------------------------------------------------------ | :------------------------------------------- |
| **本质**  | 一个类（操作档案卡）                                                                                              | 一个工具类（专门配合 `Path` 接口）                        |
| **怎么用** | `File f = new File("路径")`                                                                               | `Path p = Paths.get("路径")`；`Files.exists(p)` |
| **优点**  | 简单直观，老一辈最爱                                                                                              | 更强大，支持符号链接、文件属性、异常更明确                        |
| **缺点**  | 很多方法失败返回 `boolean`，看不出为什么失败                                                                             | 报错会抛具体异常（比如 `NoSuchFileException`）           |
| **建议**  | **两者都会用**。如果项目还在用老代码，读得懂 `File`；**新代码建议用 `Files` + `Path`**，但两者可互相转换：`file.toPath()` 或 `path.toFile()`。 |                                              |

> **0基础结论**：**先学好 `File`**，因为它是最直观的入门钥匙。等你熟练了，自然会过渡到 `Files` 工具类。

---

### 8. 一张图记住 `File` 和 `Stream` 的分工

```text
硬盘上的真实文件 (test.txt)
         ↑
    【File 档案卡】 ← 操作它的信息（存在？大小？删掉？）
         ↑
    【数据流】 ← 操作它的内容（读里面的字，写新的字进去）
         ↑
      你的程序
```

---

## 随机访问文件

### 1. 先来生活比喻（磁带 vs CD机）

- **之前的流（FileInputStream/BufferedReader）**：就像**听磁带**。你想听第5首歌，必须把前4首歌快进完，**从头到尾按顺序读**。这叫**顺序访问**。
- **今天的 `RandomAccessFile`**：就像**CD机/点读笔**。你想听第5首歌，直接按数字键 **“5”**，机器瞬间跳过去。你可以**随意跳来跳去**，先读结尾，再读开头，甚至直接在文件中间修改文字。

> **官方定义**：`RandomAccessFile` 是Java中**功能最强大**的文件访问类，它支持 **“随机访问”**——你可以随意移动一个叫做 **“文件指针”** 的光标，跳到文件的任意位置进行**读**或**写**。

---


### 3. 核心原理：文件指针（小光标）

`RandomAccessFile` 内部维护了一个 `long` 类型的**文件指针**（你可以想象成你在文档里的**输入光标**）。

- **刚打开时**：指针在开头（位置 0）。
- **`seek(位置)`**：把光标**瞬间移动**到指定位置（比如第 100 个字节处）。
- **`read()` / `write()`**：从光标当前位置开始读/写，操作完后光标自动后移。

> **记住**：你用 `seek` 跳到哪里，你的“手”就伸到哪里去操作。

---

### 4. 如何创建（2种模式）

构造方法：`new RandomAccessFile(File file, String mode)`

| 模式 (`mode`) | 含义 | 你能干什么？ |
| :--- | :--- | :--- |
| **`"r"`** | 只读模式 | 只能调用 `read()`，不能写。如果文件不存在，直接报错。 |
| **`"rw"`** | 读写模式（**最常用**） | 既可以读，也可以写。如果文件不存在，**会自动创建**它！ |

```java
import java.io.RandomAccessFile;

public class TestRAF {
    public static void main(String[] args) throws Exception {
        // 以读写模式打开（如果 test.dat 不存在，会新建一个）
        RandomAccessFile raf = new RandomAccessFile("test.dat", "rw");
        System.out.println("文件创建/打开成功！当前光标位置：" + raf.getFilePointer()); // 输出 0
        raf.close();
    }
}
```

---

### 5. 实战演示：在文件中间“插队”修改

假设我们要做一个**员工工资管理系统**，每条记录固定长度（比如100个字节）。想修改第3个员工的工资，用普通流得从头读到尾；用 `RandomAccessFile` 直接 **`seek(2 * 100)`** 跳过去改，**极快**！

我们先用最简单的方式演示：写一段文字，然后跳到中间去插入新内容。

```java
import java.io.RandomAccessFile;

public class TestRandomAccess {
    public static void main(String[] args) throws Exception {
        // 1. 创建并写入一段文字 (使用 "rw" 模式)
        try (RandomAccessFile raf = new RandomAccessFile("raf.txt", "rw")) {
            // 先写进去：ABCDEFGHIJ
            raf.write("ABCDEFGHIJ".getBytes());
            System.out.println("写入完成，当前指针位置：" + raf.getFilePointer()); // 输出 10

            // 2. 重点来了！将指针跳回到索引 3 的位置（也就是 D 的后面）
            raf.seek(3);
            System.out.println("跳转后指针位置：" + raf.getFilePointer()); // 输出 3

            // 3. 在 D 和 E 之间写入 "XYZ"
            raf.write("XYZ".getBytes());

            // 4. 关闭流，去看看 raf.txt 文件里的内容变成了什么？
            // 答案是：ABCXYZEFGHIJ 
            // 注意！它不是把 "DEFGHIJ" 往后挤，而是直接覆盖！原本的 D 没了，变成了 XYZ
            // 这就是随机访问的真相：它会覆盖原有的字节，不会自动插入！
        }
    }
}
```
> **0基础震惊**：原来 `seek` + `write` 是**覆盖写**！如果你想“插入”而不覆盖，你得自己把后面的内容先读出来缓存，写完新内容再写回去（这就是数据库底层的复杂之处）。

---

### 6. 怎么读取指定位置的内容？

假设你刚写完，现在只想读索引 `3` 到 `5` 的字符（即读取 "XYZ"）：

```java
import java.io.RandomAccessFile;

public class TestRead {
    public static void main(String[] args) throws Exception {
        try (RandomAccessFile raf = new RandomAccessFile("raf.txt", "r")) { // 只读模式
            // 跳过前3个字节（从开头数3个，即跳过 ABC）
            raf.seek(3);
            
            // 准备一个3字节的数组装数据
            byte[] buffer = new byte[3];
            // 读入3个字节
            raf.read(buffer);
            
            // 转成字符串打印
            System.out.println("索引3~5的内容是：" + new String(buffer)); // 输出：XYZ
        }
    }
}
```

---

### 7. 0基础必须掌握的常用方法清单

| 方法 | 作用 | 备注 |
| :--- | :--- | :--- |
| `seek(long pos)` | 把光标移动到文件的 `pos` 位置（从0开始计数） | **核心方法**，指哪打哪 |
| `getFilePointer()` | 获取当前光标的位置（距离开头多少个字节） | 调试时看进度用 |
| `length()` | 获取文件总大小（字节数） | 可以配合 `seek(length())` 跳到**末尾**追加内容 |
| `read(byte[] b)` | 从当前光标处读字节，填入数组 | 和普通流一样 |
| `write(byte[] b)` | 从当前光标处写入字节 | **覆盖模式**，注意备份 |
| `skipBytes(int n)` | 跳过 n 个字节（相当于 `seek(getFilePointer()+n)`，但更安全）| 当你不太确定当前位置时用 |

---

### 8. 关于编码的致命细节（中文乱码注意！）

上面的代码用了 `getBytes()`，在 Windows 下默认是 `GBK`（中文占2字节），在 Mac/Linux 下默认是 `UTF-8`（中文占3字节）。如果你用 `seek` 跳转，**必须按字节算位置**。

> **解决**：为了安全，**纯英文**随便用；如果有**中文**，强烈建议指定编码，并确保计算字节数准确：
> ```java
> // 写入时明确指定 UTF-8
> raf.write("中文".getBytes(StandardCharsets.UTF_8)); 
> // 此时 "中文" 占了 6 个字节 (UTF-8下每个中文3字节)
> ```

---

### 9. 经典应用场景（为什么必须学它？）

| 场景 | 解释 |
| :--- | :--- |
| **断点续传（下载工具）** | 文件下载到一半暂停了。下次开启时，`seek` 到已下载的大小位置，继续追加写，不用从头重下。 |
| **多线程下载** | 开10个线程，分别 `seek` 到文件的 0%、10%、20%... 各自负责下载一段，最后拼起来。 |
| **简单的键值对数据库** | 你想修改第 N 条记录，直接用 `seek(N * 固定长度)` 覆盖，无需读写整个文件。 |
| **大文件末尾追加日志** | `seek(raf.length())` 跳到末尾，`write` 写入新日志，比 `FileWriter` 更灵活。 |

---

### 10. 终极总结（对比记忆）

| 对比项 | **普通流（FileInputStream/Writer）** | **RandomAccessFile** |
| :--- | :--- | :--- |
| **访问方式** | 只能**从头到尾**顺序读/写 | **任意跳转**（随机访问） |
| **读写权限** | 只能读（InputStream）或只能写（OutputStream） | **既能读又能写**（rw模式） |
| **创建时** | 文件不存在会报错（读）或新建（写） | 根据模式决定，`rw`模式不存在则新建 |
| **本质** | 属于 I/O 流继承体系 | 独立继承 `Object`，是个“异类” |
| **适用场景** | 普通文本复制、读取配置文件 | **大文件局部修改、多线程下载、断点续传** |

# 线程
## 创建线程方法

### 1. 先来生活比喻（餐厅后厨 vs 流水线）

想象你开了一家**早餐店**：

- **单线程（之前学的普通程序）**：你是**全能大厨**。你要自己蒸包子、自己煮豆浆、自己收银。**做完A才能做B**，顾客排队排到崩溃。这叫**串行执行**。
- **多线程（今天要学的）**：你**雇了3个伙计**。
  - **伙计1（线程1）**：专门蒸包子。
  - **伙计2（线程2）**：专门煮豆浆。
  - **伙计3（线程3）**：专门收银。
  三个人**同时开工**，互不干扰，效率翻倍！这叫**并发执行**。

> **官方定义**：**线程**是操作系统能够进行运算调度的**最小单位**。它被包含在进程（程序）中，是进程中的**实际运作单位**。一个进程（如你的Java程序）可以包含多个线程。

---

### 2. 程序是怎么“同时”运行的？（时间片轮转）

很多0基础同学会问：“电脑CPU核心就那么几个，怎么可能同时跑几百个线程？”

**真相**：CPU就像只有一个“大脑”的超级快刀手。它**极快地在不同的任务之间来回切换**（毫秒级别）。比如1毫秒切到线程A，1毫秒切到线程B。因为切换太快，我们**肉眼感觉**它们是在同时运行。这叫**并发**。

---

### 3. 在Java里怎么创建一个线程？（3种方法，必学前2种！）

#### 方法一：继承 `Thread` 类（最直观，但单继承限制）
写一个类继承 `Thread`，重写 `run()` 方法，然后调用 `start()` 启动。

```java
// 1. 定义一个伙计（继承Thread）
class CookThread extends Thread {
    @Override
    public void run() {
        // 这个伙计要干的活（重写run方法）
        for (int i = 1; i <= 5; i++) {
            System.out.println("蒸包子师傅：" + i + " 笼包子出锅了！");
            try {
                Thread.sleep(1000); // 模拟做包子耗时1秒
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}

public class TestThread {
    public static void main(String[] args) {
        // 2. 招聘伙计（创建线程对象）
        CookThread cook = new CookThread();
        // 3. 让他开始干活！（调用start方法，注意不是run！）
        cook.start();

        // 主线程（老板）也在同时干别的活
        for (int i = 1; i <= 5; i++) {
            System.out.println("老板在收银：" + i + " 块钱");
            try {
                Thread.sleep(800);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```

**运行结果（交替出现）**：
```
蒸包子师傅：1 笼包子出锅了！
老板在收银：1 块钱
老板在收银：2 块钱
蒸包子师傅：2 笼包子出锅了！
...
```
> **注意**：`start()` 是告诉JVM去新开一条路跑`run()`里的代码；如果直接调用 `run()`，那就变成普通方法调用，**不会开启新线程**！

---

#### 方法二：实现 `Runnable` 接口（企业级推荐，因为可以多继承）
因为Java类是单继承，如果你的类已经继承了别的类，就不能继承`Thread`了。所以**更推荐**实现`Runnable`接口。

```java
// 1. 定义伙计干的活（实现Runnable）
class NoodleTask implements Runnable {
    @Override
    public void run() {
        for (int i = 1; i <= 5; i++) {
            System.out.println("煮面师傅：第 " + i + " 碗面好了！");
            try {
                Thread.sleep(1200);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}

public class TestRunnable {
    public static void main(String[] args) {
        // 2. 创建任务对象
        NoodleTask task = new NoodleTask();
        // 3. 把任务交给一个工人（线程），并命名
        Thread noodleThread = new Thread(task, "煮面师傅-A");
        // 4. 启动
        noodleThread.start();

        System.out.println("主线程继续干别的活...");
    }
}
```

---

#### 方法三：Lambda表达式（Java 8+，最简洁，只适合单行逻辑）
如果你的线程里代码很少，可以用Lambda（其实就是`Runnable`的语法糖）：

```java
public class TestLambda {
    public static void main(String[] args) {
        // 极简写法：()->{} 就是 run() 方法体
        new Thread(() -> {
            System.out.println("匿名线程在跑！");
        }).start();
    }
}
```

---

### 4. 线程的6大状态（生命周期——面试必问！）

一个线程从生到死，会经历这6个状态，就像人从“婴儿”到“死亡”：

| 状态 | 名字 | 比喻（工人） | 触发动作 |
| :--- | :--- | :--- | :--- |
| **New** | 新建 | 刚在招聘网站注册了简历，但**还没去上班**。 | `new Thread()` |
| **Runnable** | 就绪 | 已经打卡进公司了，**拿着工牌等**主管分配任务（等待CPU调度）。 | `start()` 调用后 |
| **Running** | 运行 | 正在**吭哧吭哧干活**（CPU正在执行你的`run()`代码）。 | CPU选中了该线程 |
| **Blocked** | 阻塞 | 发现没有面粉了（等待锁/资源），**先暂停干活**，等材料到了再继续。 | `synchronized`等待、`sleep()`、`wait()` |
| **Waiting** | 无限等待 | 被主管叫去“站着别动”（`wait()`），直到有人叫他（`notify()`）才能动。 | `Object.wait()` |
| **Terminated** | 终止 | 活干完了（`run()`结束），**正式离职**，不能再回来干活了。 | `run()`执行完毕 |

---

### 5. 新手最常踩的超级大坑（线程安全）

#### 场景：银行取钱
假设银行账户有 **1000** 块钱。你和女朋友**同时**用手机APP取钱（两个线程同时扣款），各取 **800** 块。

**正确结果**：应该只有一个人能取成功，另一个提示余额不足。

**实际可能发生（不加锁）**：
1. 线程A（你）看到余额=1000，够800，准备扣。
2. 线程B（女朋友）**在A还没扣完的一瞬间**，也看到余额=1000，也够800，准备扣。
3. 结果：A扣成200，B扣成200，**余额变成了200！总共取出了1600！银行亏了！**

> **这就是线程安全问题（数据不一致）**。

#### 解决方案：加锁（`synchronized`）
在方法上或代码块上加 `synchronized` 关键字，相当于给“取钱”这个动作加了一把**厕所门锁**。一个线程进去把门反锁（获得锁），其他线程只能在门外排队等。等它用完了开门（释放锁），下一个才能进去。

```java
class BankAccount {
    private int balance = 1000;

    // 方法一：给整个方法加锁（同步方法）
    public synchronized void withdraw(int money) {
        if (balance >= money) {
            System.out.println(Thread.currentThread().getName() + " 取走 " + money);
            balance -= money;
            System.out.println("剩余余额：" + balance);
        } else {
            System.out.println(Thread.currentThread().getName() + " 余额不足！");
        }
    }
}
```
> **0基础记住**：多个线程访问**同一个共享资源**（比如同一个账户余额）时，必须加锁！

---

### 6. 线程常用方法（背下来）

| 方法 | 作用 | 例子 |
| :--- | :--- | :--- |
| `start()` | 启动线程（**必须调这个，不能调`run()`**） | `thread.start();` |
| `sleep(ms)` | 让当前线程**休眠**指定毫秒（不释放锁） | `Thread.sleep(1000);` |
| `join()` | **插队**！等这个线程执行完，其他线程才能继续 | `t1.join();` // 等t1跑完 |
| `yield()` | 礼貌放弃CPU，让其他线程优先执行（但不一定成功） | `Thread.yield();` |
| `currentThread()` | 获取当前正在执行的线程对象 | `Thread.currentThread().getName()` |
| `setName()` | 给线程起名字（方便调试） | `new Thread("我的线程名")` |

---

### 7. 终极总结（大脑存档）

1.  **线程就是“伙计”**，让程序能同时干多件事。
2.  **创建方式**：`继承Thread`（简单）或 `实现Runnable`（**推荐**，避免单继承限制）。
3.  **启动命令**：**必须是 `start()`**，而不是 `run()`！
4.  **生命周期**：New → Runnable → Running → Terminated（途中会有Blocked/Waiting）。
5.  **最大天敌**：**线程安全**（数据抢乱了）。遇到共享资源，就用 `synchronized` 加锁保护。

---




## 用户进程和守护进程

### 1. 先来生活比喻（主角 vs 场务）

想象你在拍一部**电影**（Java程序），片场里有两种人：

- **用户线程（User Thread）** = **主角/演员**。
  导演（JVM）说了：**“只要还有一个主演在台上没演完，电影就不能散场，剧组就得一直开着。”** 主角负责演核心剧情（你的主要业务代码）。

- **守护线程（Daemon Thread）** = **场务/后勤人员**。
  比如：打光师、化妆师、扫地阿姨。他们围着主角转。**“一旦所有主演都演完下班了，场务也必须立刻收工，哪怕手里的扫把还没放下，片场也得强制关门。”** 主角都没了，后勤留着纯属浪费资源。

> **官方定义**：**用户线程**是JVM需要等待其执行完毕才会退出的线程；**守护线程**是服务于用户线程的后台线程，当所有用户线程结束时，JVM会**直接终止**所有守护线程并关闭程序。
---
### 2. 最核心的区别（背下来！）

| 对比维度 | **用户线程 (User Thread)** | **守护线程 (Daemon Thread)** |
| :--- | :--- | :--- |
| **JVM退出规则** | JVM **会等待**所有用户线程结束才退出。 | JVM **不会等待**，只要用户线程全没了，守护线程立即被“强行杀死”。 |
| **默认状态** | **默认就是**用户线程（`main`线程就是用户线程）。 | 必须手动设置（`setDaemon(true)`）。 |
| **生命周期** | 活到自己的`run()`方法跑完。 | 活到最后一个用户线程结束。 |
| **典型用途** | 处理核心业务（处理订单、接收用户请求）。 | 后台辅助任务（垃圾回收GC、自动保存草稿、心跳检测、日志监控）。 |

---

### 3. 代码实战：亲眼看看区别！

我们写两段代码，让你在控制台直观感受JVM的“无情”。

#### 场景一：默认情况（全是用户线程）
`main`线程是用户线程，我们新建的线程默认也是用户线程。

```java
public class TestUserThread {
    public static void main(String[] args) {
        // 新线程（默认是用户线程）
        Thread userThread = new Thread(() -> {
            for (int i = 1; i <= 10; i++) {
                System.out.println("用户线程（主角）还在演：" + i + " 幕");
                try {
                    Thread.sleep(500);
                } catch (InterruptedException e) {}
            }
            System.out.println("用户线程演完了！");
        });

        userThread.start();

        System.out.println("主线程（导演）喊了一声：卡！然后不管了...");
        // 注意：main方法执行到这里就结束了，但JVM不会关闭！
        // 因为userThread还在后台跑，JVM必须等它跑完10次才真正退出。
    }
}
```
> **结果**：你会看到主线程打印完“卡”后，程序**并没有退出**，控制台依然在打印 1~10，直到全部打印完，程序才停止。这就是“JVM等待用户线程”。

---

#### 场景二：设置为守护线程（配角）
我们把上面的线程设置为 **“守护线程”**，看看会发生什么。

```java
public class TestDaemonThread {
    public static void main(String[] args) {
        // 1. 创建线程
        Thread daemonThread = new Thread(() -> {
            int count = 0;
            while (true) { // 无限循环！死循环！
                count++;
                System.out.println("守护线程（场务）正在打扫：" + count + " 次");
                try {
                    Thread.sleep(300);
                } catch (InterruptedException e) {}
            }
        });

        // 2. 【核心】设置为守护线程！必须在 start() 之前调用！
        daemonThread.setDaemon(true); 
        
        // 3. 启动
        daemonThread.start();

        // 4. 主线程（用户线程）只活 2 秒钟
        try {
            Thread.sleep(2000); // 主线程睡2秒后结束
        } catch (InterruptedException e) {}

        System.out.println("主线程（唯一的主角）死了，电影散场！");
        // 主线程结束，程序里没有用户线程了，JVM立刻强制关门！
        // 守护线程虽然是个死循环，但会被 JVM 粗暴地直接掐断！
    }
}
```

**运行结果（可能打印如下）**：
```
守护线程（场务）正在打扫：1 次
守护线程（场务）正在打扫：2 次
...
守护线程（场务）正在打扫：5 次
守护线程（场务）正在打扫：6 次
主线程（唯一的主角）死了，电影散场！
```
> **震惊的真相**：守护线程只打印了几次，虽然它代码里是个 `while(true)`（永远不结束），但主线程一死，JVM直接“拔电源”，守护线程被迫中断，程序退出！**根本不会等它打印完**。

---

### 4. 超级致命陷阱（面试必问！）：`finally` 块不会执行！

**切记**：因为守护线程是被JVM**强行终止**的，所以它里面的 `finally` 代码块**根本不会被执行**！

```java
Thread daemon = new Thread(() -> {
    try {
        System.out.println("守护线程开始工作...");
        Thread.sleep(100000); // 模拟长时间工作
    } finally {
        // 这里永远不会被执行！！！
        System.out.println("我要保存数据...");
        // 如果你想在程序结束时把数据存到硬盘，千万不要放在守护线程的finally里，数据必丢！
    }
});
daemon.setDaemon(true);
daemon.start();
// 主线程很快结束...
```

> **0基础铁律**：**守护线程里绝对不能写“必须执行”的收尾操作**（比如关闭数据库连接、保存文件）。因为JVM退出时不会给你善后的机会！

---

### 5. 现实中的经典守护线程

- **垃圾回收（GC）**：它是JVM自带的守护线程。你写代码从不显式调用GC，它一直在后台默默巡视内存，当所有业务线程都结束了，程序关门，GC自然也跟着消失。
- **自动保存草稿箱**：IDE（如IDEA）每隔几秒自动保存你的代码，这个后台自动保存线程通常就是守护线程。
- **心跳检测**：长连接中，定时发Ping包探测对方是否在线的线程。

---

### 6. 怎么设置？只需一句代码

**注意：必须在 `start()` 之前设置！一旦启动了，就不能再改属性了！**

```java
Thread t = new Thread(() -> { /* 任务 */ });
t.setDaemon(true);  // 必须写在 start 前面！
t.start();
```

检查是否是守护线程：`t.isDaemon()`（返回 `true` 或 `false`）。

---

### 7. 终极总结（大脑存档）

1.  **默认规则**：我们自己 `new Thread()` 创建的都是 **用户线程**。
2.  **JVM的底线**：只要有一个用户线程活着，JVM就坚决不退出。
3.  **守护线程的宿命**：是“工具人”。当最后一个用户线程死亡，JVM就会立即终止所有守护线程（不管它们有没有干完活）。
4.  **`main` 线程**：它是一个特殊的用户线程。如果 `main` 方法执行完了，但其他用户线程还在，程序依然继续运行。
5.  **开发原则**：**重要数据（写入文件、数据库操作）绝对不能交给守护线程去做！** 必须交给用户线程，或者确保在守护线程结束前手动完成操作。

---

## synchronized ，volatile关键字

### 1. 先来两个生活比喻（刻进脑子里）

想象你有一个**共享办公室**（主内存），里面放着一个**小白板**（共享变量），门口有个**公告栏**。

- **`volatile`（公告栏）**：老板规定，**所有人在小白板上写字之前，必须先看一眼公告栏；写完字后，必须立刻把内容抄一份贴到公告栏上**。这保证了**可见性**——只要有人改了小白板，其他人进门前看一眼公告栏就知道改了。但**不锁门**！张三进去改的时候，李四也可以推门进去同时改，两人会互相覆盖（**不保证原子性**）。

- **`synchronized`（厕所门锁）**：你进办公室前，**必须把门反锁（获得锁）**，其他人在门外排队。你在里面随便改小白板，改完出门再把锁打开（释放锁）。这保证了**原子性**和**可见性**——同一时刻只有一个人能改，并且出门前会自动把内容贴到公告栏。

> **一句话总结**：`volatile` 解决“看见”问题（可见性），`synchronized` 解决“抢乱”问题（原子性 + 可见性）。

---

### 2. 为什么要它们？（现代CPU的“缓存坑”）

0基础必须知道一个硬件真相：**CPU 跑得超快，内存跑得慢**。所以每个CPU核心都有自己的**高速缓存（私人小本本）**。

**不加任何关键字时**：
线程A在自己的小本本上把 `flag = false` 改成了 `true`，但**还没来得及写回主内存**。线程B读的时候，直接从小本本里读到 `false`。于是两个线程看到的值不一样——这就是**可见性问题**（数据不一致）。

`volatile` 和 `synchronized` 都会强制让线程**直接从主内存读**，并**立刻写回主内存**，从而解决了这个问题。

---

### 3. 轻量级战士：`volatile`（仅限简单场景）

`volatile` 是Java最轻量级的同步机制，它只做两件事：
1. **保证可见性**：修改后立刻刷新到主内存，读的时候直接从主内存读。
2. **禁止指令重排序**（防止CPU优化代码导致逻辑错乱，这个先记住，后面遇到再深究）。

#### 什么时候用 `volatile`？
**当只有一个线程在写，多个线程在读，且操作是“纯赋值/读取”时**（比如开关标志位）。

```java
public class TestVolatile {
    // 加 volatile 保证：只要 running 变了，所有线程立马知道！
    private static volatile boolean running = true;

    public static void main(String[] args) throws Exception {
        Thread t = new Thread(() -> {
            System.out.println("子线程开始跑...");
            while (running) {
                // 死循环，一直等着 running 变成 false
                // 如果没有 volatile，子线程可能永远看不到 main 线程改了 running
            }
            System.out.println("子线程看到 running=false，退出！");
        });
        t.start();

        Thread.sleep(1000); // 主线程睡1秒
        System.out.println("主线程把 running 改成 false");
        running = false; // 修改标志
    }
}
// 运行结果：1秒后子线程完美退出。如果去掉 volatile，子线程可能永远卡死！
```

#### `volatile` 的死穴（致命伤！）
**它不保证原子性！** 什么叫原子性？就是“读-改-写”这一连串操作不能被打断。

**经典错误**：`volatile int count = 0;` 然后 100 个线程同时执行 `count++`（这相当于 `读count → 加1 → 写回count` 三步）。
结果最后 `count` 大概率小于 100。因为两个线程同时读到 `0`，都加1，都写回 `1`，丢失了一次加操作。

> **0基础铁律**：**`volatile` 只适用于简单的“赋值”或“开关”操作**，绝对不能用它来做计数器或累加！

---

### 4. 重型坦克：`synchronized`（万能锁）

`synchronized` 是Java的“大锁”，它保证**同一时刻，只有一个线程能进入被锁住的代码块**，进去后自带 `volatile` 的可见性效果。

#### 三种用法（0基础记住前两种）

| 用法 | 写法 | 锁的是谁？ | 场景 |
| :--- | :--- | :--- | :--- |
| **修饰实例方法** | `public synchronized void method()` | 当前对象 `this` | 同一对象的多线程调用 |
| **修饰静态方法** | `public static synchronized void method()` | 当前类的 `Class` 对象 | 全局锁，防止不同对象同时调用 |
| **修饰代码块**（最灵活） | `synchronized(obj) { ... }` | 括号里的任意对象 | 细粒度控制，只锁关键代码 |

```java
class Counter {
    private int count = 0;

    // 加锁：两个线程同时调用，必须排队！
    public synchronized void increment() {
        count++; // 现在是安全的原子操作了
    }

    public int getCount() {
        return count; // 读操作可以不加锁（如果对可见性要求不高）
    }
}

public class TestSynchronized {
    public static void main(String[] args) throws Exception {
        Counter c = new Counter();
        Thread t1 = new Thread(() -> { for (int i=0; i<10000; i++) c.increment(); });
        Thread t2 = new Thread(() -> { for (int i=0; i<10000; i++) c.increment(); });
        t1.start(); t2.start();
        t1.join(); t2.join();
        System.out.println("最终结果：" + c.getCount()); // 稳稳输出 20000
    }
}
```

> `synchronized` 除了保证原子性，还自动保证了可见性：线程A释放锁时，会把所有修改**刷回主内存**；线程B获取锁时，会**清空自己缓存**，强制从主内存重新读。

---

### 5. 终极对决：一张表让你永不混淆

| 对比维度 | **`volatile`** | **`synchronized`** |
| :--- | :--- | :--- |
| **核心作用** | 保证**可见性**（大家看到的值一致） | 保证**原子性**（操作不可打断）+ 可见性 |
| **是否加锁** | **不加锁**（无阻塞，性能极高） | **加锁**（阻塞排队，性能较低） |
| **能否修饰代码块** | 只能修饰**变量** | 修饰**方法** 或 **代码块** |
| **能否修饰局部变量** | **不能**（只能修饰成员变量/静态变量） | 能（锁定局部对象） |
| **适用场景** | **一个写，多个读**的开关标志、状态标记 | **多个写**的累加、复合操作（先读后改） |
| **底层实现** | 内存屏障（禁止重排序） | `monitorenter` / `monitorexit` 指令（JVM内置锁） |

---

### 6. 0基础必踩的三个巨坑（保命必看）

#### 坑一：`volatile` 不能解决 `i++`（已经强调过了，再提一遍）
**解决方案**：要么改用 `synchronized`，要么用 `AtomicInteger`（原子类，底层用CAS，比锁快）。

#### 坑二：`synchronized` 锁错了对象
```java
// 错误示范：两个线程操作两个不同的 Counter 对象，锁不住！
Counter c1 = new Counter();
Counter c2 = new Counter();
new Thread(() -> c1.increment()).start();
new Thread(() -> c2.increment()).start(); // 这两个锁的不是同一个 this，可以同时进去！

// 解决办法：用静态 synchronized 方法，或者锁定同一个对象，如 synchronized(Counter.class)
```

#### 坑三：`synchronized` 是可重入的（面试爱问）
**“可重入”** 的意思是：线程A拿到了锁，在锁里面又调用了另一个加锁的方法，**可以再次直接进入**，不会把自己锁死。
```java
public synchronized void a() { b(); } // 合法！A线程拿着锁，可以再次进入
public synchronized void b() { ... }
```
如果是非可重入锁，A在 `b()` 门口就会永远等自己释放锁，导致**死锁**。Java的 `synchronized` 很聪明，做了个计数器记录“进入次数”，安全得很。

---

### 7. 终极选择决策树（以后开发就这么选）

1.  **问**：这个变量是不是简单的 `true/false` 或 `int` 赋值，且只有 **1个线程写，N个线程读**？
    *   **是** → 用 `volatile`（轻量级，快！）。
2.  **问**：这个操作是 `count++`、`i = i + 1` 或者需要“先判断再修改”？
    *   **是** → **必须用 `synchronized`**（或者 `AtomicInteger`，原理我们以后学）。
3.  **问**：我是想锁定一大段业务逻辑，确保这几行代码完全串行执行？
    *   **是** → **必须用 `synchronized`** 锁住代码块。

---






## 线程启动、调度和挂起
### 1. 先来生活比喻（导演、场务与演员）

想象你在拍一部**动作大片**（Java程序）：

- **启动（Start）**：你作为导演，对着一个已经就位的演员（新建线程）喊了一声 **“Action！”**。演员开始进入状态，准备开演（进入就绪状态，等待CPU指令）。
    
- **调度（Scheduling）**：**场务（CPU调度器）** 拿着一个“聚光灯”（CPU时间片），在多个演员之间**疯狂切换**。给A演员照1毫秒，然后瞬间切给B演员照1毫秒。因为切换太快，观众（我们）感觉所有演员在同时演戏（并发）。
    
- **挂起（Suspending/Pausing）**：你突然喊 **“停！”**，让某个演员**暂时别动**（暂停执行），但你不能让他死掉（不能终止），因为他待会儿还要接着刚才的动作继续演。
    

---

### 2. 启动（`start()` vs `run()`）—— 复习与纠错

这里必须再强调一次**0基础最致命的错误**：

- **`new Thread().start()`（正确）**：导演喊“Action！”，JVM会去创建一个新的“执行管道”（系统线程），新管道会去执行 `run()` 里的代码。这是**真正的多线程**。
    
- **`new Thread().run()`（错误）**：你直接喊了演员的名字，让他去演，但这发生在**当前（主）管道**里。这只是普通的**方法调用**，没有新管道，**单线程**顺序执行。
    

java

public class TestStart {
    public static void main(String[] args) {
        Thread t = new Thread(() -> System.out.println("我在跑！"));
        
        t.run();  // 错误！在当前线程（main）里执行，没有启动新线程！
        t.start(); // 正确！JVM会开一条新路去执行打印语句。
    }
}

---

### 3. 调度（Scheduling）—— CPU的“雨露均沾”

调度不由我们程序员直接控制，而是由**操作系统的线程调度器**决定。但我们有4个“弱建议”可以影响它：

|方法/属性|作用|比喻|备注|
|---|---|---|---|
|**`setPriority(int)`**|设置优先级（1~10，默认5）。|给导演建议：“这位是主角，多给他点镜头。”|只是**建议**，底层OS不一定听你的（尤其在Windows上）。|
|**`Thread.yield()`**|当前线程**主动放弃**CPU时间片，让其他线程先跑。|演员说：“我有点累，让其他演员先演一会儿吧。”|不释放锁！只是把CPU位置让出来，下一秒可能又抢回来。|
|**`Thread.sleep(ms)`**|强制当前线程休眠指定毫秒，**不释放锁**。|演员闭眼休息5秒钟，醒后继续演。|必须处理 `InterruptedException`。|
|**`Object.wait()`**|释放锁并进入等待队列，直到被唤醒。|演员主动交出演戏资格，去后台睡觉，直到副导演喊他。|必须配合 `synchronized`（之前课程讲过）。|

> **0基础认知**：不要试图用 `setPriority` 来控制业务顺序，它不稳定。**真正的顺序控制靠 `wait/notify` 或 `BlockingQueue`**。

### 4. 挂起（Pausing）

```java
class PausableThread extends Thread {
    // volatile 保证多个线程之间能看到这个标志的最新值（后面课程会细讲）
    private volatile boolean running = true;   // 总开关
    private volatile boolean paused = false;   // 暂停标志
    private final Object pauseLock = new Object(); // 专门用来控制暂停的锁

    @Override
    public void run() {
        int count = 0;
        while (running) {
            // 1. 检查暂停标志
            synchronized (pauseLock) {
                while (paused) {
                    try {
                        System.out.println("线程被挂起，等待恢复...");
                        pauseLock.wait(); // 释放锁，安全地阻塞等待
                    } catch (InterruptedException e) {}
                }
            }

            // 2. 核心工作（模拟干活）
            System.out.println("工作中... 第 " + ++count + " 秒");
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {}
        }
        System.out.println("线程彻底终止。");
    }

    // 外部调用的“挂起”方法
    public void pauseThread() {
        synchronized (pauseLock) {
            paused = true;
            // 注意：这里不用notify，因为挂起时线程自己在wait里
        }
    }

    // 外部调用的“恢复”方法
    public void resumeThread() {
        synchronized (pauseLock) {
            paused = false;
            pauseLock.notify(); // 唤醒正在 wait 的线程
        }
    }

    // 外部调用的“停止”方法
    public void stopThread() {
        running = false;
        // 如果线程正在暂停等待中，必须唤醒它，否则它醒不来，也就看不到 running=false
        resumeThread(); 
    }
}

// 测试
public class TestPause {
    public static void main(String[] args) throws Exception {
        PausableThread t = new PausableThread();
        t.start(); // 启动

        Thread.sleep(3000); // 主线程等3秒
        
        System.out.println(">>> 主线程命令：挂起！");
        t.pauseThread(); // 挂起

        Thread.sleep(3000); // 挂起3秒

        System.out.println(">>> 主线程命令：恢复！");
        t.resumeThread(); // 恢复

        Thread.sleep(3000);

        System.out.println(">>> 主线程命令：彻底停止！");
        t.stopThread(); // 停止
    }
}
```
**运行结果分析**：

1. 前3秒：线程每秒打印一次。
    
2. 中间3秒：**彻底安静**（线程在 `wait()` 里老老实实睡觉，且释放了锁）。
    
3. 恢复后：继续每秒打印一次。
    
4. 停止：线程检测到 `running=false`，优雅退出。
    

> **这就是现代多线程安全挂起的标准写法！**

#### pack和unpack
#### 1. 先来生活比喻（停车券机制）

想象你把车开到一个**超级智能停车场**（JVM 内存）：

- **`park()`（停车/阻塞）**：你（线程）把车开到入口，栏杆是落下的。保安（JVM）过来问你：**“你有停车券吗？”**
    
- **`unpark(线程)`（给券/唤醒）**：保安突然往你手里塞了一张**停车券（许可证，Permit）**。
    

**这里有两个“反常识”的神奇规则（和 `wait/notify` 完全不同）：**

1. **如果保安提前把券塞给你（先 `unpark`），你再开车进去（后 `park`），栏杆会直接抬起，你**不需要等待**，直接通过！**（这解决了 `wait/notify` 如果先通知后等待，线程会永远挂起的致命缺陷）。
    
2. **券只有一张，不能累积**。保安给你塞了 100 张券，你也只能**用掉 1 张**就通过，剩下的作废。栏杆不会让你连续通过 100 次。
    

---

#### 2. 撕掉面具：`LockSupport` 到底是什么？

`LockSupport` 是 `java.util.concurrent.locks` 包下的一个工具类。它的核心方法就两个（全是静态方法）：

|方法|作用|比喻|
|---|---|---|
|`LockSupport.park()`|**阻塞**当前线程（让线程停在原地等待）。|把车停在栏杆前等通行证。|
|`LockSupport.unpark(Thread t)`|**唤醒**指定的线程（给目标线程发通行证）。|保安给某个司机塞一张通行证。|

> **0基础震惊**：你用 `LockSupport` 的时候，**完全不需要 `synchronized` 同步锁**！它直接操作线程本身，告别了 `wait/notify` 必须配对的繁琐。

---

#### 3. 核心特性一：先 `unpark` 再 `park`（绝对不会死等！）

这是 `LockSupport` 和 `wait/notify` 最本质的区别！

- **`wait/notify` 的痛点**：如果服务员 A 先调用了 `notify()`，厨师 B 后调用 `wait()`，B 会永远睡死过去（因为错过了通知）。**必须先 `wait` 后 `notify`，顺序颠倒就死锁。**
    
- **`LockSupport` 的神奇**：`unpark` 相当于提前把“通行证”塞给线程，线程执行 `park` 时发现有证，直接通过，**毫发无伤**！
    

**代码见证奇迹：**

```java
java

import java.util.concurrent.locks.LockSupport;
public class TestUnparkFirst {
    public static void main(String[] args) {
        // 1. 创建一个线程
        Thread t = new Thread(() -> {
            System.out.println("子线程：我准备停车了...");
            // 2. 尝试阻塞等待
            LockSupport.park();
            System.out.println("子线程：栏杆抬起了！我通过了！");
        });
        t.start();
        // 3. 主线程先给子线程发一张“通行证”（先 unpark）
        System.out.println("主线程：提前给子线程发一张停车券！");
        LockSupport.unpark(t); 
        // 即使子线程还没跑到 park，它手里已经有券了，等它跑到 park 时直接通过！
        // 运行结果：子线程成功打印 "我通过了！"，程序完美退出。
    }
}
```

> 如果是 `wait/notify`，上面这种写法必死锁。而 `LockSupport` 轻松化解。

---

#### 4. 核心特性二：许可证（Permit）不可累积

刚才说了，保安给券，一张通行证只能用一次。


```java
public class TestNoAccumulate {
    public static void main(String[] args) throws Exception {
        Thread t = new Thread(() -> {
            LockSupport.park(); // 第一次停车：用掉那张券，通过
            System.out.println("第一次通过");
            LockSupport.park(); // 第二次停车：没有券了，被永久阻塞！！！
            System.out.println("第二次通过（这句永远打印不出来）");
        });
        t.start();
        LockSupport.unpark(t); // 主线程只发了一张券
        // 结果：控制台只打印 "第一次通过"，然后程序卡死（因为没有第二张券了）
    }
}
```

> **0基础铁律**：`unpark` 相当于给线程发一个**布尔标记（true/false）**，发多次也只是 `true`，`park` 一次就把它改成 `false`。最多只能存 1 张，攒不了。

---

#### 5. 核心特性三：响应中断（不会抛异常，更柔和）

用过 `Thread.sleep()` 或 `Object.wait()` 的都知道，一旦线程被 `interrupt()`，会立刻抛出恶心的 `InterruptedException`，你必须写 `try-catch`。

**而 `LockSupport.park()` 不一样：**  
当线程在 `park()` 中被其他线程 `interrupt()` 时，它**不会抛异常**，而是**直接返回**（相当于自动醒过来了），但会保留中断状态。

```java
java

public class TestInterrupt {
    public static void main(String[] args) throws Exception {
        Thread t = new Thread(() -> {
            System.out.println("线程开始阻塞...");
            LockSupport.park(); // 在这里睡觉
            System.out.println("线程醒来了！中断标志：" + Thread.currentThread().isInterrupted());
            // 注意：这里不会抛异常，优雅地醒来了。
        });
        t.start();
        Thread.sleep(1000);
        t.interrupt(); // 中断它
        // 输出：线程醒来了！中断标志：true
    }
}
```

---

#### 6. `LockSupport` vs 之前的“老旧三件套”

我把你现在脑子里的知识全部串起来，对比一下：

|对比维度|**Object.wait/notify**|**Thread.suspend(已弃用)**|**LockSupport.park/unpark**|
|---|---|---|---|
|**是否需要持有锁？**|**必须**在 `synchronized` 块内（否则报错）|不需要|**不需要**（直接操作线程，极其自由）|
|**先通知后等待会怎样？**|**死锁**（错过信号，永远等）|不适用|**安全通过**（预发通行证机制）|
|**释放锁吗？**|释放锁（让出资源）|**不释放锁**（导致死锁，故弃用）|不涉及锁（只阻塞线程，不影响其他线程抢锁）|
|**处理中断**|抛出 `InterruptedException`|无|**静默返回**（不抛异常，检查标志位即可）|
|**使用难度**|复杂（容易死锁、虚假唤醒）|极度危险|**极简（业界标杆，JUC底层全用它）**|

---

#### 7. 实战升华：用 `LockSupport` 重写“可挂起线程”（秒杀之前写法）

还记得上节课我们用 `wait/notify` 写了一大堆 `synchronized` 和 `volatile` 标志位的“可挂起线程”吗？现在用 `LockSupport`，代码直接**减半**，且逻辑极其清晰！

```java

import java.util.concurrent.locks.LockSupport;
class EasyPausableThread extends Thread {
    private volatile boolean running = true;
    private volatile boolean paused = false;
    @Override
    public void run() {
        int count = 0;
        while (running) {
            // 检查暂停标志
            if (paused) {
                System.out.println("【挂起】线程进入阻塞状态...");
                LockSupport.park(); // 简单粗暴地停车！不需要任何锁！
                System.out.println("【恢复】线程被唤醒了！");
            }
            // 核心工作
            System.out.println("工作中... 第 " + ++count + " 秒");
            try { Thread.sleep(1000); } catch (Exception e) {}
        }
        System.out.println("线程终止。");
    }
    public void pauseThread() { this.paused = true; } // 改标志
    public void resumeThread() {
        this.paused = false;
        LockSupport.unpark(this); // 直接给当前线程发通行证，让它从 park 中弹出来！
    }
    public void stopThread() { 
        this.running = false;
        // 如果线程在 park 中睡着，必须唤醒来检查 running，否则停不掉
        LockSupport.unpark(this); 
    }
}
public class TestLockSupportPause {
    public static void main(String[] args) throws Exception {
        EasyPausableThread t = new EasyPausableThread();
        t.start();
        Thread.sleep(3000);
        System.out.println(">>> 主线程：挂起！");
        t.pauseThread();
        Thread.sleep(3000);
        System.out.println(">>> 主线程：恢复！");
        t.resumeThread();
        Thread.sleep(3000);
        System.out.println(">>> 主线程：停止！");
        t.stopThread();
    }
}
```

**总结优点**：你看，完全不用写 `synchronized`，不用担心锁释放问题，也不用写 `wait/notify` 的配对循环，代码干净得像白开水！

---

#### 8. 终极总结（大脑存档）

1. **`park()`** = 停车（阻塞），**`unpark()`** = 给停车券（唤醒）。
    
2. **无敌特性**：券可以**提前发**（先 `unpark` 后 `park` 依然有效），彻底解决了 `wait/notify` 的错过信号死锁问题。
    
3. **记清楚**：券最多只能存 **1 张**，不能累积。
    
4. **中断处理**：`park()` 被中断时**不会抛异常**，只是默默醒过来，这是 JUC 锁框架（`ReentrantLock`）底层实现的基础。
    
5. **使用场景**：你现在写代码几乎**不需要**直接手动 `wait/notify` 了，用 `LockSupport` 更安全。而且它是 `AQS`（抽象队列同步器，`ReentrantLock`、`CountDownLatch` 的底层）的基石。


## 线程通信

### 1. 先来生活比喻（厨师与服务员）

想象一个**繁忙的厨房**（程序），里面有一个**传菜台（共享数据区）**，最多只能放1道菜（缓冲区大小为1）。

- **厨师（Producer，生产者线程）**：负责炒菜（产生数据）。
- **服务员（Consumer，消费者线程）**：负责端菜（消费数据）。

**老板定下了两条死规定（线程通信规则）**：

1. **传菜台满了（已有1道菜）**：厨师必须**停下炒菜（阻塞/等待）**，直到服务员把菜端走（消费数据），厨师才能被叫醒继续炒下一道。
2. **传菜台空了（没菜可端）**：服务员必须**停下来等（阻塞/等待）**，直到厨师炒好新菜（生产数据），服务员才能被叫醒去端菜。

> 这里的“叫醒”和“等待”，就是Java里**`wait()`**和**`notify()`**要做的事情。

---

### 2. Java中的通信三件套（属于`Object`类，每个对象都有！）

这三个方法**极其特殊**，你必须记住它们的铁律：

| 方法 | 作用 | 比喻 | 铁律 |
| :--- | :--- | :--- | :--- |
| **`wait()`** | 让当前线程**进入等待**状态，并**释放**它所占有的锁。 | 服务员看没菜，把手中传菜台的钥匙（锁）放下，去旁边睡觉。 | **必须在 `synchronized` 块内调用！** |
| **`notify()`** | 随机**叫醒**一个正在 `wait()` 的线程。 | 厨师炒好菜，随机喊醒一个正在睡觉的服务员。 | **必须在 `synchronized` 块内调用！** |
| **`notifyAll()`** | **叫醒**所有正在 `wait()` 的线程。 | 炒了一堆菜，把所有人都喊起来干活。 | **必须在 `synchronized` 块内调用！** |

> **0基础震惊**：为什么这些方法是定义在 `Object`（祖宗类）里，而不是定义在 `Thread`（线程类）里？
> **答**：因为 `wait()` 和 `notify()` 操作的是**锁**（对象头里的标记），而锁是属于**对象**的，不是属于线程的。所以任何对象都能当“传菜台”的钥匙。

---

### 3. 终极重要：`wait()` 和 `sleep()` 的区别（面试必问！）

| 对比 | **`sleep(ms)`** | **`wait()`** |
| :--- | :--- | :--- |
| **属于谁？** | `Thread` 类的静态方法 | `Object` 类的实例方法 |
| **释放锁吗？** | **不释放锁**！抱着钥匙睡大觉 | **释放锁**！把钥匙交出来，让别人进去干活 |
| **谁叫醒它？** | 时间到了自动醒，或 `interrupt()` | 必须由别的线程调用 `notify()`/`notifyAll()` 叫醒 |
| **在哪用？** | 任何地方都可以用 | **必须在 `synchronized` 同步块里用** |

---

### 4. 代码实战：完美演示“厨师服务员”通信

我们来写一个**“一个放一个取”**的经典模型。为了让你看得更清楚，我们用 `Object` 当传菜台的锁。

```java
// 传菜台（共享对象）
class Desk {
    private String food;          // 当前菜名，null代表没菜
    private boolean hasFood = false; // 标志位：有没有菜

    // 厨师炒菜（生产者）
    public synchronized void cook(String name) {
        // 1. 如果已经有菜了，厨师必须等待（不能继续炒）
        while (hasFood) {
            try {
                System.out.println("【厨师】传菜台满了，等待服务员端走...");
                wait(); // 释放锁，厨师进入阻塞等待
            } catch (InterruptedException e) {}
        }
        // 2. 做菜（生产数据）
        this.food = name;
        hasFood = true;
        System.out.println("【厨师】炒好了：" + name);
        // 3. 叫醒一个正在等待的服务员（通知消费者来吃）
        notify();
        // 注意：这里虽然调用了 notify，但厨师还没释放锁，必须等厨师退出 synchronized 代码块，锁才真正释放。
    }

    // 服务员端菜（消费者）
    public synchronized void take() {
        // 1. 如果没有菜，服务员必须等待
        while (!hasFood) {
            try {
                System.out.println("【服务员】传菜台空空如也，等待厨师炒菜...");
                wait(); // 释放锁，服务员进入阻塞等待
            } catch (InterruptedException e) {}
        }
        // 2. 端菜（消费数据）
        System.out.println("【服务员】端走了：" + food);
        food = null;
        hasFood = false;
        // 3. 叫醒厨师（通知生产者来做菜）
        notify();
    }
}

// 测试代码
public class TestWaitNotify {
    public static void main(String[] args) {
        Desk desk = new Desk();

        // 厨师线程（生产5道菜）
        Thread cookThread = new Thread(() -> {
            String[] menu = {"红烧肉", "清蒸鱼", "麻婆豆腐", "回锅肉", "酸菜鱼"};
            for (String dish : menu) {
                desk.cook(dish);
                try { Thread.sleep(500); } catch (Exception e) {} // 模拟做菜耗时
            }
        }, "厨师-老王");

        // 服务员线程（消费5道菜）
        Thread waiterThread = new Thread(() -> {
            for (int i = 0; i < 5; i++) {
                desk.take();
                try { Thread.sleep(300); } catch (Exception e) {} // 模拟端菜耗时
            }
        }, "服务员-小李");

        // 启动线程（注意：必须先启动服务员等待，还是先启动厨师？其实都行，因为有 wait 循环判断）
        cookThread.start();
        waiterThread.start();
    }
}
```

**运行结果（完美交替）**：
```
【厨师】炒好了：红烧肉
【服务员】端走了：红烧肉
【服务员】传菜台空空如也，等待厨师炒菜...
【厨师】炒好了：清蒸鱼
【服务员】端走了：清蒸鱼
【厨师】炒好了：麻婆豆腐
...
```
> **关键点**：你会发现程序完美地“做一道，端一道”。如果没有 `wait/notify`，程序要么厨师疯狂炒（数据覆盖），要么服务员疯狂端（端到 `null`）。

---

### 5. 0基础最容易犯的死罪（避坑指南）

#### 避坑一：必须用 `while`，不能用 `if`（虚假唤醒）
上面的代码我用的是 `while (hasFood)`，而不是 `if (hasFood)`。

> **原理**：线程被 `notify` 叫醒后，**并不一定能立刻获得锁**。如果多个服务员在排队，一个服务员拿到锁时，菜可能已经被别的服务员端走了。用 `while` 醒过来后**再检查一次条件**，是绝对安全的；用 `if` 醒过来直接往下走，菜没了就会 `NullPointerException`。**永远用 `while` 包裹 `wait()`！**

#### 避坑二：`wait()` 必须放在 `synchronized` 里
如果你不把 `cook()` 和 `take()` 用 `synchronized` 修饰，直接写 `wait()`，程序会立刻抛出 `IllegalMonitorStateException`（非法监视器异常）。

> **原因**：线程必须先持有对象的“锁”，才能调用该对象的 `wait` 方法主动交枪。

#### 避坑三：`notify()` 不会立即释放锁
注意：厨师调用 `notify()` 后，服务员**并没有立刻干活**！因为厨师还在 `synchronized` 代码块里，锁还没释放。只有当厨师把 `synchronized` 方法执行完（或者主动 `wait()`），服务员才能抢到锁开始干活。

---

### 6. 高级替代品：`BlockingQueue`（你以后会遇到的“懒人神器”）

在真正的企业级开发中，我们几乎**不会**手写 `wait/notify`，因为它太容易出错了（虚假唤醒、死锁）。

Java 提供了封装好的 **`BlockingQueue`（阻塞队列）**，它内部已经帮我们写好了完美的 `wait/notify`。我们只需要 `put`（放）和 `take`（取）。

```java
// 以后你会这么写（极其简单，线程安全）
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.BlockingQueue;

public class TestBlockingQueue {
    public static void main(String[] args) throws Exception {
        // 容量为 1 的阻塞队列（相当于传菜台）
        BlockingQueue<String> queue = new ArrayBlockingQueue<>(1);

        // 生产者放菜：queue.put("红烧肉");（如果满了，自动阻塞等待）
        // 消费者取菜：String food = queue.take();（如果空了，自动阻塞等待）
    }
}
```

---

### 7. 终极总结（大脑存档）

1.  **线程通信的核心**：`wait()` 让出锁去睡觉，`notify()` 叫醒睡觉的线程。
2.  **三个铁律**：
    *   必须在 `synchronized` 块里。
    *   `wait()` 会释放锁，`sleep()` 不会释放锁。
    *   **永远用 `while`** 套住 `wait()`，防止虚假唤醒。
3.  **本质逻辑**：**生产者 — 消费者模型**（一个负责造，一个负责用，通过标志位协调）。
4.  **生产环境**：理解原理后，实际代码优先用 `BlockingQueue`，避免手写底层通信。

---

# 网络
## InetAddress

### 1. 先来生活比喻（门牌号 vs 人名）

想象你要给远方的朋友**寄快递**（通过网络发送数据）：

- **IP地址（如 `192.168.1.1`）**：是朋友家的**精确门牌号**。快递员（网络协议）靠这个找到他。计算机只认这个。
- **主机名（如 `www.baidu.com`）**：是朋友的名字/公司名。**人脑**记名字容易，但记 `110.242.68.4` 这种数字很困难。

**`InetAddress` 在Java里是什么？**
它就是一个**“通讯录名片”**，上面同时记着朋友的名字（主机名）和精确门牌号（IP地址）。Java程序通过这个名片，就能在网络上找到目标机器。

---

### 2. 撕开面具：`InetAddress` 的本质

`InetAddress` 位于 `java.net` 包下。它**没有公共构造方法**（你不能直接 `new InetAddress()`），而是要像查电话簿一样，通过**静态工厂方法**来获取。

> **0基础必背**：获取 `InetAddress` 对象的三种场景：

| 场景 | 方法 | 比喻 |
| :--- | :--- | :--- |
| **查本机**（我自己在哪？） | `InetAddress.getLocalHost()` | 你掏出身份证（本机配置），看自己家的门牌号。 |
| **查特定网站**（百度在哪？） | `InetAddress.getByName("www.baidu.com")` | 你在通讯录里查“百度”这个名字，找到它的门牌号。 |
| **查所有可能地址**（百度有多个服务器） | `InetAddress.getAllByName("www.baidu.com")` | 你发现百度有好几个仓库（多台服务器），拿到所有门牌号列表。 |

---

### 3. 代码实战（亲手查“门牌号”）

我们写一段最直观的代码，让你亲眼看到自己的电脑IP和百度的IP。

```java
import java.net.InetAddress;
import java.net.UnknownHostException;
import java.util.Arrays;

public class TestInetAddress {
    public static void main(String[] args) {
        try {
            // ========== 场景1：获取本机信息 ==========
            InetAddress local = InetAddress.getLocalHost();
            System.out.println("本机主机名：" + local.getHostName());      // 比如：DESKTOP-ABC123
            System.out.println("本机IP地址：" + local.getHostAddress());   // 比如：192.168.1.5

            // ========== 场景2：根据域名获取IP（单地址） ==========
            InetAddress baidu = InetAddress.getByName("www.baidu.com");
            System.out.println("\n百度主机名：" + baidu.getHostName());    // www.baidu.com
            System.out.println("百度IP地址：" + baidu.getHostAddress());   // 110.242.68.4

            // ========== 场景3：获取域名对应的所有IP（集群/负载均衡） ==========
            InetAddress[] allBaidu = InetAddress.getAllByName("www.baidu.com");
            System.out.println("\n百度所有IP地址列表：" + Arrays.toString(allBaidu));
            // 你会发现，百度有好几个IP！这就是为什么人家服务器不会崩。

            // ========== 场景4：判断是否可达（类似 ping 命令） ==========
            System.out.println("\n百度是否可达？" + baidu.isReachable(3000)); 
            // 注意：3000 是超时毫秒数。返回 true 表示能连通。

        } catch (UnknownHostException e) {
            // 如果域名写错了（比如 www.baiduuuu.com），就会抛这个异常！
            System.out.println("找不到这个主机！请检查域名。");
            e.printStackTrace();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```
**运行结果（在你的电脑上会略有不同）**：
```
本机主机名：DESKTOP-123
本机IP地址：192.168.1.100

百度主机名：www.baidu.com
百度IP地址：110.242.68.4

百度所有IP地址列表：[www.baidu.com/110.242.68.4, www.baidu.com/110.242.68.3]

百度是否可达？true
```

---

### 4. 核心方法大解析（0基础必记）

拿到 `InetAddress` 对象后，最常用的方法就这几个：

| 方法 | 返回值 | 解释 |
| :--- | :--- | :--- |
| `getHostName()` | 字符串 | 获取域名（如果没有，返回IP字符串）。 |
| `getHostAddress()` | 字符串 | 获取纯数字IP（如 `192.168.0.1`）。 |
| `getAddress()` | `byte[]` 数组 | 获取二进制格式的IP（存的是原始字节，用于底层计算）。 |
| `isReachable(int timeout)` | boolean | **尝试ping**这个地址，判断网络是否通（单位毫秒）。 |

> **关于 `getAddress()` 的坑**：它返回的是 `byte[]`（有符号），像 `192.168.1.1` 转成 byte 会变成负数。如果你要打印，建议直接用 `getHostAddress()`。

---

### 5. 0基础必须搞懂的两个概念（DNS 与 回环地址）

#### 概念一：DNS（域名系统，电话簿管理员）
- 你的电脑不认识 `www.baidu.com`，它只认 `110.242.68.4`。
- 当你执行 `getByName("www.baidu.com")` 时，Java底层会去问**DNS服务器**（相当于电信的114查号台）：“请问百度的门牌号是多少？” 查询需要**几十毫秒**，所以网络操作通常比较慢（要放在子线程里，避免卡住主界面）。

#### 概念二：回环地址（本地测试专用）
- **`127.0.0.1`** 和 **`localhost`** 永远代表**你自己的电脑**。
- 你在开发时，自己给自己发数据，不需要联网，用这个地址最安全。
```java
InetAddress loopback = InetAddress.getByName("127.0.0.1");
System.out.println(loopback.getHostName()); // 输出 localhost
```

---

### 6. 新手最容易踩的坑（保命警告）

1.  **找不到主机（`UnknownHostException`）**：
   - 代码里写的域名错了（比如把 `baidu` 拼成 `baiduu`）。
   - 或者你的电脑**没联网**，DNS查不到任何记录。
   - **解决**：必须 `try-catch` 捕获这个异常，否则编译报错（它是受检异常）。

2.  **`isReachable` 不一定管用（防火墙杀手）**：
   - 很多公司服务器禁用了 `ping`（ICMP协议），导致 `isReachable` 返回 `false`，但其实服务器是活着的（只是80端口开着）。
   - **解决**：真正的“通不通”要用下一节课学的 `Socket` 去连接指定端口（比如 80 或 8080），连接成功才是真的通。

3.  **缓存问题**：
   - `InetAddress` 会把查到的IP缓存起来。如果你改了百度的DNS映射，但Java程序之前已经查过了，它可能不会立刻生效（因为系统有DNS缓存）。

---

### 7. 把网络和之前的线程联系起来（思维飞跃）

你现在已经会了线程，应该能想到：**DNS查询（`getByName`）会阻塞线程！**

```java
// 错误示范（在main线程直接查，如果DNS挂了，程序会卡死在这里）
InetAddress addr = InetAddress.getByName("www.slow-website.com"); 

// 正确姿势（用我们之前学的线程，异步查询，不卡界面）
new Thread(() -> {
    try {
        InetAddress addr = InetAddress.getByName("www.baidu.com");
        System.out.println("查到IP：" + addr.getHostAddress());
    } catch (UnknownHostException e) {
        System.out.println("查不到！");
    }
}).start();
System.out.println("主线程可以继续干别的活，不会卡死！");
```

---

### 8. 终极总结（大脑存档）

1.  **`InetAddress`** 是Java里表示“IP地址+主机名”的对象，相当于一张网络名片。
2.  **怎么拿**：不能用 `new`，必须用 `getLocalHost()`（本机）或 `getByName()`（查域名）。
3.  **最常用的两个方法**：`getHostAddress()`（拿门牌号）和 `getHostName()`（拿名字）。
4.  **核心依赖**：它依赖操作系统的**DNS服务器**，所以必须联网且处理 `UnknownHostException`。
5.  **网络 + 线程**：任何网络操作都可能很慢，**千万**不要在GUI（图形界面）的主线程里直接调用 `getByName`，否则界面会卡住！要用之前学的多线程去跑。

---

## NetworkInterface
这个问题衔接得**天衣无缝**！学完 `InetAddress`（门牌号），你现在问的 `NetworkInterface` 就是**“门牌号所在的这栋楼”**——也就是你电脑上实实在在的**物理网卡或虚拟网卡**。

老规矩，我们继续用**生活比喻 + 极简代码**，把你电脑里“看不见的硬件”扒个底朝天！

---

### 1. 先来生活比喻（大门 vs 门牌号）

想象你的电脑是一栋**豪华写字楼**（操作系统），里面有好几个大门通向外界：

- **`InetAddress`（IP地址）**：是写在门上的**门牌号**（比如 `192.168.1.5`）。同一个大门可以有多个门牌号（一个网卡绑定多个IP）。
- **`NetworkInterface`（网络接口）**：是那扇**实实在在的大门**（物理硬件）。比如：
  - 大门1：**有线网卡（以太网）**，插着网线。
  - 大门2：**无线网卡（Wi-Fi）**，连着热点。
  - 大门3：**虚拟网卡（Loopback）**，只在自己楼里转悠（`127.0.0.1`）。
  - 大门4：**虚拟机网卡（VMware/VirtualBox）**，给虚拟机用的虚拟通道。

> **核心关系**：`NetworkInterface` 是 **“硬件/设备”**，`InetAddress` 是 **“配置在上面的逻辑地址”**。一个网卡可以有多个IP，一个IP也可能绑定在多个网卡上（但极少见）。

---

### 2. 为什么需要 `NetworkInterface`？

- 你想知道自己的 **MAC地址（物理硬件地址）**，它刻在网卡芯片上，全世界唯一。
- 你的服务器有 **双网卡**（内网IP + 外网IP），程序需要区分到底用哪个网卡发数据。
- 做 **网络嗅探**（如Wireshark）时，需要选择监听哪块网卡。
- 判断网卡是否 **插着网线/连着WiFi**（`isUp()` 方法）。

---

### 3. 代码实战：把你的电脑“扒光”，看看有几扇门

我们写一段代码，把你电脑上所有网卡信息全部打印出来！

```java
import java.net.NetworkInterface;
import java.net.InetAddress;
import java.util.Enumeration;

public class TestNetworkInterface {
    public static void main(String[] args) throws Exception {
        // 1. 获取电脑上所有的网卡（返回的是古老的 Enumeration，类似迭代器）
        Enumeration<NetworkInterface> interfaces = NetworkInterface.getNetworkInterfaces();

        System.out.println("========== 本机所有网络接口（大门）列表 ==========");
        int index = 0;
        
        while (interfaces.hasMoreElements()) {
            NetworkInterface ni = interfaces.nextElement();
            index++;
            
            System.out.println("\n--- 网卡 #" + index + " ---");
            // 2. 获取网卡的名字（系统内部名，如 eth0, wlan0, lo）
            System.out.println("系统名称 (Name)：" + ni.getName());
            // 3. 获取网卡的展示名（人类友好的名字，如 "Intel(R) Wi-Fi 6 AX200"）
            System.out.println("展示名称 (DisplayName)：" + ni.getDisplayName());
            
            // 4. 判断网卡是否启用（相当于网线插着没/无线连上没）
            System.out.println("是否启用 (isUp)：" + ni.isUp());
            // 5. 判断是否是回环网卡（127.0.0.1）
            System.out.println("是否回环 (isLoopback)：" + ni.isLoopback());
            // 6. 判断是否是虚拟网卡（比如虚拟机生成的）
            System.out.println("是否虚拟 (isVirtual)：" + ni.isVirtual());
            
            // 7. 获取 MAC 地址（硬件地址，刻在芯片上的全球唯一ID）
            byte[] mac = ni.getHardwareAddress();
            if (mac != null) {
                // 把字节转成常见的十六进制格式，如 AA:BB:CC:DD:EE:FF
                StringBuilder sb = new StringBuilder();
                for (int i = 0; i < mac.length; i++) {
                    sb.append(String.format("%02X", mac[i]));
                    if (i < mac.length - 1) sb.append(":");
                }
                System.out.println("MAC 地址：" + sb.toString());
            } else {
                System.out.println("MAC 地址：无（虚拟网卡或未启用）");
            }

            // 8. 获取这个网卡上绑定的所有 IP 地址（重点！）
            System.out.println("绑定的 IP 地址列表：");
            Enumeration<InetAddress> ipList = ni.getInetAddresses();
            if (!ipList.hasMoreElements()) {
                System.out.println("  （没有绑定IP）");
            }
            while (ipList.hasMoreElements()) {
                InetAddress ip = ipList.nextElement();
                System.out.println("  -> " + ip.getHostAddress());
            }
        }
        System.out.println("\n=========================================");
    }
}
```

**运行结果（每个人的电脑输出都不同，大概长这样）**：
```
========== 本机所有网络接口（大门）列表 ==========

--- 网卡 #1 ---
系统名称 (Name)：lo
展示名称 (DisplayName)：Software Loopback Interface 1
是否启用 (isUp)：true
是否回环 (isLoopback)：true
是否虚拟 (isVirtual)：false
MAC 地址：无（虚拟网卡或未启用）
绑定的 IP 地址列表：
  -> 127.0.0.1
  -> 0:0:0:0:0:0:0:1

--- 网卡 #2 ---
系统名称 (Name)：eth0
展示名称 (DisplayName)：Realtek PCIe GbE Family Controller
是否启用 (isUp)：true
是否回环 (isLoopback)：false
是否虚拟 (isVirtual)：false
MAC 地址：A1:B2:C3:D4:E5:F6
绑定的 IP 地址列表：
  -> 192.168.1.105
  -> 169.254.12.34

--- 网卡 #3 ---
系统名称 (Name)：wlan0
展示名称 (DisplayName)：Intel(R) Wi-Fi 6 AX201
是否启用 (isUp)：false  // 表示没连WiFi
...
```

---

### 4. 常用方法大清单（0基础速查表）

| 方法 | 返回值 | 解释（0基础版） |
| :--- | :--- | :--- |
| `getName()` | `String` | 系统内部代号（如 `eth0`，写代码时常用这个）。 |
| `getDisplayName()` | `String` | 用户看得懂的硬件品牌名（用于界面显示）。 |
| `isUp()` | `boolean` | 网卡是否“激活”（相当于网线插好了吗？WiFi连上了吗？）。 |
| `isLoopback()` | `boolean` | 是不是虚拟的本地环回（`127.0.0.1`那扇门）。 |
| `isVirtual()` | `boolean` | 是不是虚拟机/容器造的假网卡（如VMware）。 |
| `getHardwareAddress()` | `byte[]` | **MAC地址**（物理地址）。如果返回 `null`，说明没硬件（虚拟网卡）。 |
| `getInetAddresses()` | `Enumeration<InetAddress>` | 获取该网卡下所有的 IP 地址（一台电脑可以有多个IP）。 |
| `getMTU()` | `int` | 最大传输单元（网络包的大小限制，网络高手用的）。 |

---

### 5. 0基础最关注的：怎么判断“我连上网了没”？

实战场景：你的程序需要联网，如果没网就别发请求。可以这样写：

```java
public static boolean isNetworkConnected() {
    try {
        Enumeration<NetworkInterface> interfaces = NetworkInterface.getNetworkInterfaces();
        while (interfaces.hasMoreElements()) {
            NetworkInterface ni = interfaces.nextElement();
            // 过滤掉：虚拟网卡、回环网卡、未启用的
            if (ni.isLoopback() || ni.isVirtual() || !ni.isUp()) {
                continue;
            }
            // 如果有合法的硬件网卡且启动了，就算有网
            return true;
        }
    } catch (Exception e) {
        return false;
    }
    return false;
}
```

---

### 6. 新手最容易踩的坑（保命三连）

#### 坑一：`Enumeration` 是上古接口，用着不顺手
`getNetworkInterfaces()` 返回的是老式 `Enumeration`（枚举），不能用增强 `for` 循环。
**解决**：要么像上面那样 `while(hasMoreElements())`，要么转成 `ArrayList`：
```java
List<NetworkInterface> list = Collections.list(NetworkInterface.getNetworkInterfaces());
// 现在可以用 for (NetworkInterface ni : list) 了！
```

#### 坑二：MAC 地址在 Java 8 某些系统上返回 `null`
在 Windows 上通常没问题，但在某些 Linux 精简版或容器里，`getHardwareAddress()` 可能返回 `null`，一定要做 `if (mac != null)` 判断，否则会 `NullPointerException`。

#### 坑三：`isUp()` 不代表能上网！
`isUp()` 只代表网卡驱动加载成功，网线插着。但如果你没连路由器（DHCP没分配到IP），或者路由器没连外网，`isUp()` 依然是 `true`。**判断“真能上网”**，最保险的还是用我们上节课学的 `InetAddress.isReachable()` 去 ping 一下 `8.8.8.8`（谷歌DNS）。

---

### 7. 把脑中的知识串起来（`InetAddress` vs `NetworkInterface`）

| 对比维度 | **`InetAddress`** | **`NetworkInterface`** |
| :--- | :--- | :--- |
| **比喻** | 写在门上的**门牌号**（逻辑地址）。 | 那扇**物理大门**（硬件/驱动）。 |
| **获取方式** | `getByName("域名")` 或 `getLocalHost()` | `getNetworkInterfaces()`（枚举所有门） |
| **代表什么** | 一个具体的 IP（如 `192.168.1.5`）。 | 一张具体的网卡（如你的 Wi-Fi 芯片）。 |
| **底层依赖** | DNS服务器（远程查号台）。 | 操作系统驱动（本地硬件管理）。 |
| **典型用途** | 确定目标服务器在哪（连接对方）。 | 确定自己从哪扇门出去（选择路由）。 |

## URL

### 1. 先来生活比喻（快递地址单 vs 快递员）

想象你要从网上下载一张图片，或者访问一个网页：

- **`URL`（快递地址单）**：就是一张**标准格式的地址条**，上面清清楚楚写着：
  > `协议:// 服务器地址 : 端口 / 文件路径 ? 参数`
  > 比如：`https://www.baidu.com:443/s?wd=java`
  - **快递员用什么车送（协议）**：`https`（专车，加密安全）。
  - **收货城市（主机名）**：`www.baidu.com`。
  - **收货大楼门牌号（端口）**：`443`（Https的默认门，可以省略）。
  - **具体货架位置（路径）**：`/s`（表示搜索接口）。
  - **特殊备注（查询参数）**：`?wd=java`（表示搜“java”）。

- **同步请求（今天学的核心）**：你拿着这张地址单，**站在门口死死等快递员回来**。快递员没把货送到家，你就一直站着不动（**线程阻塞**），啥也别干。这叫**同步（阻塞）请求**。

> **官方定义**：`java.net.URL` 类代表一个统一资源定位符，它指向互联网上的某个“资源”（网页、图片、文件等）。通过它，我们可以**打开连接**并**读取数据**。

---

### 2. 撕开面具：URL 的组成部分（0基础拆解）

我们把 `https://www.example.com:8080/news/index.html?page=2#title` 大卸八块：

| 部分 | 方法 | 例子 | 生活翻译 |
| :--- | :--- | :--- | :--- |
| **协议** | `getProtocol()` | `https` | 用什么车送（货车/专车） |
| **主机名** | `getHost()` | `www.example.com` | 城市名 |
| **端口** | `getPort()` | `8080`（如果没写，返回 -1） | 大楼具体的门 |
| **路径** | `getPath()` | `/news/index.html` | 货架位置 |
| **查询参数** | `getQuery()` | `page=2` | 备注（选第2页） |
| **锚点（书签）** | `getRef()` | `title` | 具体看哪一段（#后面的） |

---

### 3. 如何创建一个 URL 对象？

创建 `URL` 对象**极其简单**，但注意它**会检查格式**，格式错了就抛异常。

```java
import java.net.URL;
import java.net.MalformedURLException;

public class TestCreateURL {
    public static void main(String[] args) {
        try {
            // 方式1：直接传完整的字符串（最常用）
            URL url1 = new URL("https://www.baidu.com/s?wd=Java");
            
            // 方式2：拆分开来写（协议, 主机, 路径）
            URL url2 = new URL("https", "www.baidu.com", "/s?wd=Java");
            
            // 方式3：父路径 + 子路径（拼接）
            URL base = new URL("https://www.baidu.com");
            URL url3 = new URL(base, "/s?wd=Python");

            System.out.println("url1 的完整地址：" + url1.toString());
            System.out.println("url3 的协议：" + url3.getProtocol()); // https

        } catch (MalformedURLException e) {
            // 如果协议写错了（比如 htttps://），就会抛这个异常
            System.out.println("URL格式非法！检查一下协议和路径");
            e.printStackTrace();
        }
    }
}
```

---

### 4. 终极实战：用 Java 发同步请求，把百度首页源码“扒”下来！

这是**最激动人心**的环节！我们不用浏览器，用纯Java代码把网页的HTML代码抓取到控制台。

```java
import java.net.URL;
import java.net.URLConnection;
import java.io.InputStream;
import java.io.BufferedReader;
import java.io.InputStreamReader;

public class TestSyncRequest {
    public static void main(String[] args) {
        // 注意：网络请求必须捕获异常，因为可能断网或服务器挂了
        try {
            // 1. 创建地址单（我要访问百度）
            URL url = new URL("https://www.baidu.com");

            // 2. 打开连接（相当于拨通电话，但还没说话）
            //    这一步在底层会做 DNS 查询（把域名转成 IP，即 InetAddress 的工作）
            URLConnection connection = url.openConnection();
            
            // 【重要】设置超时时间（同步请求必须加！）
            connection.setConnectTimeout(5000); // 连接超时 5 秒（连不上就放弃）
            connection.setReadTimeout(5000);    // 读取超时 5 秒（下载太慢就放弃）

            // 3. 拿到数据的“水管”（输入流）
            //    这一步才是真正开始发送 HTTP 请求，并把服务器返回的数据灌进来！
            try (InputStream is = connection.getInputStream();
                 // 用字符流包裹，指定 UTF-8 解码（防止中文乱码）
                 BufferedReader reader = new BufferedReader(new InputStreamReader(is, "UTF-8"))) {
                
                System.out.println("开始同步下载网页源码... (线程会卡在这里等待数据返回)");
                String line;
                int count = 0;
                // 4. 一行一行读（同步阻塞：没读完就一直卡在读操作上）
                while ((line = reader.readLine()) != null) {
                    System.out.println(line); // 打印每一行HTML代码
                    count++;
                    if (count > 20) { // 防止打印太多刷屏，只打前20行
                        System.out.println("... (内容过多，已截断)");
                        break;
                    }
                }
                System.out.println("下载完毕！");
            }
        } catch (java.net.SocketTimeoutException e) {
            System.out.println("网络请求超时！检查网络或目标服务器。");
        } catch (java.net.UnknownHostException e) {
            System.out.println("域名解析失败！检查域名是否写错（调用了InetAddress）。");
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

**运行结果（会打印出一堆眼花缭乱的HTML代码）**：
```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>百度一下，你就知道</title>
    ...
</head>
<body>
...
</body>
</html>
```
> **你看！** 这就是你的Java程序“卧底”到百度服务器，把它的首页偷回来了！

---

### 5. 什么是“同步请求”？（肉眼可见的阻塞）

运行上面的代码时，你会发现：
- 控制台**忽然停住**（感觉程序卡死了），过了几百毫秒后，才“哗啦啦”一下子喷出所有HTML代码。
- 在这个**等待期间**，`main` 线程被 `reader.readLine()` **死死卡住**（阻塞），什么别的代码都执行不了。
- **这就是同步（阻塞）I/O**：没拿到数据，我就站着死等。

> **0基础理解**：就像食堂排队打饭，前面的人不动，你就只能在原地傻站着，啥也干不了。

---

### 6. 0基础必须避开的三个巨坑（保命三连）

#### 坑一：忘记加超时（`setConnectTimeout`），程序永久卡死！
如果百度服务器突然宕机，或者你的网线掉了，`url.openConnection()` 可能会**无限期等待**（默认超时是无穷大！）。程序就永远卡在那里，像死机了一样。
**解决**：**必须写** `connection.setConnectTimeout(5000);` 和 `setReadTimeout(5000);`（单位毫秒）。

#### 坑二：中文乱码（GBK vs UTF-8）
很多国内网站（如百度）用 `UTF-8`，但有些老网站用 `GBK`。
**解决**：创建 `InputStreamReader` 时，**千万**要指定编码，不要用默认的 `new InputStreamReader(is)`（可能用操作系统编码导致乱码）。
```java
// 正确姿势（指定 UTF-8）
new InputStreamReader(is, "UTF-8");
// 如果是 GBK 网站
new InputStreamReader(is, "GBK");
```

#### 坑三：`openConnection()` 只是“拨号”，`getInputStream()` 才是“说话”
很多人以为 `openConnection()` 就发请求了，**大错特错**！
- `openConnection()`：只是创建了一个连接对象（填好了快递单），**还没寄出去**。
- `getInputStream()` / `getOutputStream()`：这一刻才真正**把包裹发出去**，开始传输数据。

---

### 7. 和之前知识的大串联（打通任督二脉）

你运行 `new URL("https://www.baidu.com").openConnection()` 时，底层悄悄做了这几件事：

1.  **`InetAddress`（查门牌号）**：JVM调用DNS，把 `www.baidu.com` 解析成 `110.242.68.4`（这就是你上节课学的）。
2.  **`NetworkInterface`（选哪扇门）**：操作系统路由表决定从你的 `eth0`（有线网卡）还是 `wlan0`（Wi-Fi）把数据包发出去（这是上上节课学的）。
3.  **Socket（打电话）**：在 `110.242.68.4:443` 之间建立一条TCP连接（下一课会学）。
4.  **HTTP协议（说话）**：发送 `GET / HTTP/1.1` 请求，接收服务器返回的HTML（就是上面代码干的事）。

---

### 8. 为什么我说“同步请求”有缺陷？（引出未来的“多线程”）

我们学过多线程。如果我在 **GUI（图形界面）** 的按钮点击事件里，直接写上面的同步请求代码：
> 用户点了“下载”按钮，程序界面立刻**卡死**，鼠标都动不了。直到下载完成，界面才“活”过来。

**正确的做法（我们现在的知识已经能实现了！）**：
用我们之前学的 **`Thread`（线程）** 包一层：
```java
new Thread(() -> {
    // 把上面请求百度的代码全部塞进这里
    // 这样主界面就不会卡了！
}).start();
```

---

### 9. 终极总结（大脑存档）

1.  **`URL`** = 标准网络地址单（包含协议、主机、路径、参数）。
2.  **如何创建**：`new URL("完整地址")`，记得处理 `MalformedURLException`。
3.  **如何同步请求**：
    - `url.openConnection()` → 设置超时 → `getInputStream()` → 用 `BufferedReader` 读。
4.  **同步（阻塞）**：线程在读取数据时会**卡住等待**，直到服务器返回或超时报错。
5.  **铁三角**：`URL` 负责拼地址，`InetAddress` 负责查IP，`NetworkInterface` 负责找网卡出口。
6.  **实践铁律**：**网络操作必须加超时，必须指定字符编码，最好放在子线程里跑！**

---

## 异步请求

### 1. 先来生活比喻（傻等 vs 叫号器）

想象你去一家**网红餐厅**（目标服务器）吃饭：

- **同步请求（你上节课学的）**：你站在柜台前**死死盯着厨师**（线程阻塞）。厨师不把菜端出来，你**一步都不走**，连手机都不敢玩。如果厨师做菜要 10 分钟，你就傻站 10 分钟。**极度浪费你的时间（CPU资源）**。

- **异步请求（今天学的）**：你拿了**叫号器（`CompletableFuture`）**，然后**回座位上玩手机（主线程继续干别的活）**。厨师做好菜，“叮”一声叫号器响了（回调/通知），你再去柜台端菜。**这 10 分钟里，你刷了短视频、回了微信，啥都没耽误！**

> **官方定义**：**异步请求**是指调用方发出请求后，**不阻塞等待**结果，而是立即返回一个“凭证”（`Future`）。当服务器处理完毕，通过回调（Callback）或轮询的方式，通知调用方来取结果。

---

### 2. 撕开面具：Java里怎么做异步？（两条路）

#### 路径一：“穷人的异步”（你现有的知识就能实现！）
直接套用你学过的 **`Thread`（线程）** 包装上节课的同步代码。主线程 `new Thread(...).start()` 后立刻跑走，子线程在后台阻塞等待。
> **缺点**：每发一个请求就要 new 一个线程，高并发下（1万个请求）会直接把内存撑爆。

#### 路径二：“王者的异步”（Java 11+ 官方标准）
从 Java 11 开始，官方提供了 **`java.net.http.HttpClient`**，配合 **`CompletableFuture`**（叫号器），实现真正的**非阻塞异步**。底层用的是操作系统级的“事件通知”，**一个线程可以管理成千上万个请求**。

---

### 3. 代码实战：用 Java 11 的 `HttpClient` 发异步请求

**注意**：如果你用的是 Java 8，这段代码跑不了（Java 8 要引入第三方库如 OkHttp）。**强烈建议你升级到 Java 11+**，这是现在的业界标配。

```java
import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.TimeUnit;

public class TestAsyncRequest {

    public static void main(String[] args) throws Exception {
        // 1. 创建一个 HTTP 客户端（相当于餐厅的总台）
        HttpClient client = HttpClient.newHttpClient();

        // 2. 构建请求（填写快递单）
        HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create("https://www.baidu.com"))
                .header("User-Agent", "Java-Async-Client") // 假装是浏览器
                .timeout(java.time.Duration.ofSeconds(5))  // 设置总超时
                .GET()
                .build();

        System.out.println("主线程：请求已发出（拿了叫号器），开始干别的活...");

        // 3. 【核心】异步发送！这行代码不阻塞，瞬间返回一个“叫号器”！
        CompletableFuture<HttpResponse<String>> future = client.sendAsync(request, 
                HttpResponse.BodyHandlers.ofString());

        // 4. 主线程可以继续干别的事（比如玩手机）
        for (int i = 1; i <= 3; i++) {
            System.out.println("主线程：刷短视频 " + i + " 秒...");
            Thread.sleep(1000);
        }

        // 5. 等叫号器响了（异步回调），我们来处理结果
        //    thenAccept 表示：数据到了之后，自动执行里面的代码（不阻塞主线程）
        future.thenAccept(response -> {
            System.out.println("\n【回调触发】叫号器响了！");
            System.out.println("状态码：" + response.statusCode());
            // 只打印前 200 个字符，防止刷屏
            String body = response.body();
            System.out.println("页面源码（前200字符）：" + body.substring(0, Math.min(200, body.length())));
        });

        // 6. 注意：因为异步是并行的，主线程如果提前结束，程序就退了。
        //    这里用 join() 让主线程等一等异步任务（相当于不急着结账走人）
        //    在实际的 GUI 或 Web 服务器中，主线程是无限循环的，不需要这行。
        System.out.println("\n主线程：我准备等异步任务完成再退出...");
        future.join(); // 阻塞等待异步任务执行完毕（演示用）
        System.out.println("主线程：异步任务完成，程序退出。");
    }
}
```

**运行结果（注意看时序）**：
```
主线程：请求已发出（拿了叫号器），开始干别的活...
主线程：刷短视频 1 秒...
主线程：刷短视频 2 秒...
主线程：刷短视频 3 秒...

【回调触发】叫号器响了！
状态码：200
页面源码（前200字符）：<!DOCTYPE html>...

主线程：我准备等异步任务完成再退出...
主线程：异步任务完成，程序退出。
```
> **看到没？** 主线程在刷短视频的时候，`thenAccept` 里的代码**根本没执行**。直到 3 秒后，数据返回了，回调代码才“啪”地一下自动打印出来。

---

### 4. `CompletableFuture` 到底是什么？（叫号器的说明书）

你拿到的 `future` 对象，就是那个“叫号器”。它有几种状态：

| 状态/方法 | 比喻 | 解释 |
| :--- | :--- | :--- |
| **未完成** | 叫号器没响，灯不亮。 | 服务器还在处理，数据没回来。 |
| **已完成** | 叫号器“叮”了，灯亮了。 | 数据回来了，或者出错了。 |
| **`.thenAccept()`** | 你设置：“响了就自动去端菜”。 | 注册回调函数，数据来了自动执行。 |
| **`.join()`** | 你拿着叫号器**站着死等**（同步阻塞）。 | 强行把异步变同步（仅用于演示，正常不用）。 |
| **`.completeExceptionally()`** | 叫号器显示“厨师把菜烧糊了”。 | 请求超时或服务器报错。 |

---

### 5. 异步的核心优势（为什么现代程序都这么干？）

| 对比维度 | **同步请求（上节课）** | **异步请求（这节课）** |
| :--- | :--- | :--- |
| **线程状态** | 线程**阻塞**（傻等），占着茅坑不拉屎。 | 线程**非阻塞**（发了请求就回来），继续干活。 |
| **资源消耗** | 1个请求占1个线程（线程昂贵，1万个请求就崩）。 | 1个线程可以管理**成千上万**个请求（极省资源）。 |
| **代码写法** | 代码是**顺序**写的（读-等-处理），容易理解。 | 代码是**回调**写的（发-注册回调），新手容易晕。 |
| **异常处理** | 直接 `try-catch`。 | 需要 `.exceptionally()` 或 `.handle()`。 |
| **适用场景** | 内部管理工具、低并发脚本。 | **互联网大厂**、高并发网关、微服务调用。 |

---

### 6. 0基础最容易踩的坑（回调地狱 & 提前退出）

#### 坑一：主线程提前结束，异步回调还没跑完！
**现象**：你运行代码，只看到“请求已发出”，**没看到 `thenAccept` 打印内容**，程序就退出了。
**原因**：异步请求是**后台线程**执行的。主线程（前台线程）一死，JVM 就会直接杀死所有后台线程，不等你回调。
**解决**：用 `future.join()`（阻塞等待）或者在 GUI/Web 容器里（主线程会一直存活）。

#### 坑二：异常被吞了，程序静默失败！
**现象**：域名写错了，但控制台什么都没打印，静悄悄的。
**原因**：异步回调中的异常，默认**不会**像同步那样直接打印堆栈。
**解决**：必须加 `.exceptionally()` 兜底！
```java
future.exceptionally(e -> {
    System.out.println("异步请求出错了：" + e.getMessage());
    return null; // 返回一个默认值
});
```

#### 坑三：回调里面写 `Thread.sleep` 或复杂循环？
**错误**：在 `thenAccept` 里写耗时的数据库操作或 `Thread.sleep`。
**后果**：虽然主线程不卡，但**回调线程（默认是ForkJoinPool的线程）会被你卡住**，影响其他异步任务的调度。
**正确**：如果回调里要干重活，再 `thenApplyAsync` 扔给另一个线程池去干。

---

### 7. 和你的知识大串联（高光时刻）

你现在已经能把**之前学的所有东西**串起来了：

1.  **`URL`**：定义地址。
2.  **`HttpClient`**：替代了上节课的 `URLConnection`，是 Java 11 的新版“快递员”，原生支持异步。
3.  **`CompletableFuture`**：和 `LockSupport` 原理相通，底层是 `park/unpark` 机制在支撑线程的挂起与唤醒。
4.  **线程池**：异步默认用的是 `ForkJoinPool.commonPool()`（公共线程池），你之前学的线程知识在这里发挥了底层作用。

---

### 8. 终极对比：异步 vs 多线程（别再搞混了！）

- **多线程（你之前学的）**：是**工具**。开启多条路同时跑。
- **异步（今天学的）**：是**策略**。不阻塞当前路，等结果回来了再开一条小路去处理。
- **组合技**：最牛逼的写法是 **`异步 + 回调`**（比如上面的代码），只占用极少的线程，处理海量的请求。这也是 **Node.js** 和 **Netty** 横扫高并发领域的底层密码。

---

## socket通信
### 1. 先来生活比喻（电话热线 vs 快递）

- **`URL` / `HttpClient`（快递/外卖）**：你发一个请求，服务器给你一个响应，**然后挂断**。你不能随时追加一句话，想说话得重新下一单。这叫**短连接（无状态）**。

- **`Socket`（电话热线）**：你给客服（服务器）拨通电话，**双方一直不挂断**。你可以随时说话（发数据），客服可以随时回应（收数据），直到你主动挂断。这叫**长连接（双向实时）**。

> **官方定义**：`Socket` 是**网络通信的端点**，它封装了 IP 地址 + 端口号。`Socket` 编程本质是 **“两台机器之间建立一条永不中断的管道，通过字节流互相灌水”**。

---

### 2. 撕开面具：IP + 端口 = 唯一的“电话号码”

- **`IP地址`**：找到了你家小区（目标机器）。
- **`端口号（Port）`**：找到了你家门口的信箱（具体应用程序）。比如 `80` 是网页，`443` 是加密网页，`3306` 是MySQL数据库。
- **`Socket`**：就是那根连着你家信箱和对方总机的**物理电话线**。

---

### 3. 必须分清的两个角色（服务端 vs 客户端）

- **服务端（Server）**：**24小时值守的客服总机**。它不能主动给你打电话，只能**“趴在电话旁等”（`accept` 阻塞等待）**。它有一个专门的 `ServerSocket` 用来“接客”。
- **客户端（Client）**：**打电话的人**。他知道对方的号码（IP+端口），主动拨号（`new Socket(...)`）建立连接。

---

### 4. 代码实战：亲手打造“回声服务器”（你说啥，它回啥）

这是学习 Socket 的“Hello World”。我们写两个类：**服务端**（等待呼叫）和 **客户端**（主动呼叫）。

#### 第一步：服务端代码（24小时接线员）
```java
import java.io.*;
import java.net.ServerSocket;
import java.net.Socket;

public class EchoServer {
    public static void main(String[] args) throws Exception {
        // 1. 接线员在 6666 端口趴着等（ServerSocket）
        System.out.println("【服务器】启动，在端口 6666 等待客户端连接...");
        try (ServerSocket serverSocket = new ServerSocket(6666)) {
            
            // 2. 阻塞等待客户来电（accept方法会卡在这里，直到有客户端连接）
            Socket clientSocket = serverSocket.accept();
            System.out.println("【服务器】检测到客户端连接！来自：" + clientSocket.getInetAddress());
            
            // 3. 建立“听筒”和“话筒”（输入流读数据，输出流写数据）
            try (BufferedReader reader = new BufferedReader(
                            new InputStreamReader(clientSocket.getInputStream()));
                 PrintWriter writer = new PrintWriter(clientSocket.getOutputStream(), true)) { // true表示自动刷新
                
                String line;
                // 4. 循环等待客户端发话（readLine会阻塞，直到客户端发来一行文字）
                while ((line = reader.readLine()) != null) {
                    System.out.println("【服务器】收到客户端说：" + line);
                    // 5. 原封不动回一句（回声）
                    writer.println("服务器已收到：" + line);
                }
                System.out.println("【服务器】客户端挂断了电话。");
            }
        }
    }
}
```

#### 第二步：客户端代码（主动拨号的人）
```java
import java.io.*;
import java.net.Socket;

public class EchoClient {
    public static void main(String[] args) throws Exception {
        // 1. 客户端拨号（连接本机的 6666 端口）
        System.out.println("【客户端】正在拨号连接 localhost:6666 ...");
        try (Socket socket = new Socket("127.0.0.1", 6666)) {
            System.out.println("【客户端】连接成功！");
            
            // 2. 建立“话筒”（写）和“听筒”（读）
            try (BufferedReader reader = new BufferedReader(
                            new InputStreamReader(socket.getInputStream()));
                 PrintWriter writer = new PrintWriter(socket.getOutputStream(), true)) {
                
                // 3. 模拟发送几条消息
                String[] messages = {"你好，服务器！", "今天天气不错", "bye"};
                for (String msg : messages) {
                    System.out.println("【客户端】发送：" + msg);
                    writer.println(msg); // 发送数据（注意：println会自动换行，方便readLine读取）
                    
                    // 4. 等待服务器回声（阻塞读）
                    String response = reader.readLine();
                    System.out.println("【客户端】收到服务器回声：" + response);
                    
                    Thread.sleep(1000); // 间隔1秒发一条
                }
                System.out.println("【客户端】发送完毕，断开连接。");
            }
        }
    }
}
```

#### 怎么运行？（关键步骤）
1. **先运行** `EchoServer` 的 `main` 方法（控制台显示“等待客户端连接...”）。
2. **再运行** `EchoClient` 的 `main` 方法。

**运行结果（完美对应）**：
```
【服务器】启动，在端口 6666 等待客户端连接...
【服务器】检测到客户端连接！来自：/127.0.0.1
【服务器】收到客户端说：你好，服务器！
【服务器】收到客户端说：今天天气不错
【服务器】收到客户端说：bye
【服务器】客户端挂断了电话。
```
```
【客户端】正在拨号连接 localhost:6666 ...
【客户端】连接成功！
【客户端】发送：你好，服务器！
【客户端】收到服务器回声：服务器已收到：你好，服务器！
...（后续类似）
```

---

### 5. 核心知识点：你之前学的技术全部汇聚于此！

运行 `Socket` 时，你之前学的所有知识都在协同工作：

| 你学的知识 | 在 Socket 里怎么体现？ |
| :--- | :--- |
| **I/O 流（字符流）** | `BufferedReader` 和 `PrintWriter` 就是之前学的读写文本的水管。 |
| **线程（Thread）** | **极其重要！** 上面写的服务器是**单线程**的（只能服务一个客户端）。如果有两个客户端同时连接，第二个会永远等待。**必须用线程**（如下方扩展）。 |
| **同步阻塞** | `accept()`、`readLine()` 都会**卡死**线程。所以要在子线程里跑，否则主界面会卡住（你上节课学的）。 |
| **`InetAddress`** | `socket.getInetAddress()` 获得到对方的IP（查门牌号）。 |
| **异常处理** | 网络随时可能断，必须 `try-catch`（`SocketException`）。 |

---

### 6. 0基础最容易踩的四大死坑（保命四连）

#### 坑一：端口被占用（`BindException: Address already in use`）
**现象**：你运行服务端，报错说端口 6666 被占用了。
**原因**：你之前运行的服务端没关，还趴在电话机上。
**解决**：在控制台按 `Ctrl + C` 关掉旧的，或者换个端口号（比如 6667）。

#### 坑二：忘记 `flush()` 或 `println` 的换行符
**现象**：客户端 `writer.println(msg)` 发了，但服务端 `reader.readLine()` 永远卡着读不到。
**原因**：`readLine()` 必须读到 `\n`（换行符）或 `\r\n` 才会返回。`println` 自带换行，但 `print` 不带。
**解决**：**服务器和客户端必须约定好协议**。最简单就是用 `println` 发，`readLine` 接。

#### 坑三：服务器只服务一个客户端（单线程死穴）
**现象**：客户端1连上开始聊天，客户端2连的时候，服务端没反应（因为服务器还在处理客户端1的 `while` 循环，没空回来执行 `accept`）。
**解决**：用你学过的**线程池**！每次 `accept` 到一个新客户端，就 `new Thread()` 扔进后台处理。
```java
// 多线程版服务器骨架（每个客户端独立线程）
while (true) {
    Socket clientSocket = serverSocket.accept();
    new Thread(() -> {
        // 处理这个 clientSocket 的读写
    }).start();
}
```

#### 坑四：半关闭（对方断网了，你还傻等）
**现象**：客户端拔了网线，服务端 `readLine()` 既不返回 `null`，也不抛异常，永远卡死。
**解决**：设置 **`setSoTimeout`（读超时）**。如果超过指定秒数没读到数据，就抛异常跳出循环。
```java
clientSocket.setSoTimeout(5000); // 5秒没数据就抛异常
```

---

### 7. 终极总结（大脑存档）

1.  **`Socket`** = 电话线（双向实时通信），**`ServerSocket`** = 总机（只负责等电话）。
2.  **TCP 三次握手**：你 `new Socket()` 时，底层自动完成“拨号-确认-建立连接”（你不用管）。
3.  **通信本质**：**就是 I/O 流**。往 `OutputStream` 写数据 = 说话，从 `InputStream` 读数据 = 听对方说话。
4.  **核心铁律**：**服务端必须多线程**（否则只能服务一个客户）；**读写必须约定格式**（比如用换行符分割消息）。
5.  **和 URL 的区别**：`URL` 是“一次性快递”，`Socket` 是“持续通话”。

---





# temp