## 一、Java 基础高频知识点清单（面试必考轮廓）

- 语言与面向对象
  - 值类型 vs 引用类型；自动装箱/拆箱坑点
  - 重载（overload）vs 重写（override）区别、场景
  - 面向对象三大特性：封装 / 继承 / 多态
  - 抽象类 vs 接口（Java 8 以后接口有 default / static 方法）
  - `==` 与 `equals`、`hashCode` 契约
- String 体系
  - `String`、`StringBuilder`、`StringBuffer` 区别和使用场景
  - 字符串常量池（intern）、`new String("a")` 有几份对象
  - `String` 为什么设计成不可变（线程安全、缓存、类加载等）
- 集合框架
  - `List` / `Set` / `Map` 常见实现：`ArrayList`、`LinkedList`、`HashSet`、`TreeSet`、`HashMap`、`LinkedHashMap`、`ConcurrentHashMap`
  - `HashMap` 底层结构（数组 + 链表 + 红黑树）、负载因子、扩容机制
  - `HashMap` 和 `Hashtable`、`ConcurrentHashMap` 区别
  - `ArrayList` vs `LinkedList` 的时间复杂度和典型使用场景
- 泛型与反射
  - 泛型擦除（编译期检查、运行时擦除）、`List<Object>` vs `List<String>` 是否可以互转
  - `? extends T` 和 `? super T` 区别
  - 反射能做什么（框架：Spring、MyBatis）、常见类：`Class` / `Constructor` / `Field` / `Method`
- 异常机制
  - checked 异常 vs unchecked 异常
  - 自定义业务异常的实践（类似你项目里的 `ServiceException`、`ClientException`）
  - `try-with-resources` 的作用
- IO 与序列化（简单掌握即可）
  - BIO / NIO 的基本概念（多路复用、缓冲区、通道）
  - Java 序列化的用途与缺点（`Serializable` 的问题）
- Java 8 常见特性
  - Lambda 表达式、函数式接口（`@FunctionalInterface`）
  - Stream 常用操作（map/filter/collect）、惰性求值
  - Optional 简单用法，避免 NPE

## 一、语言与面向对象

这些是 Java 后端面试中最核心的“基础内功”。在实际面试中，面试官不仅想听到你背诵概念，更想考察你是否真的在开发中踩过这些坑。我们逐一用图解和实战视角来拆解。

### 1. 值类型 vs 引用类型 ；自动装箱/拆箱坑点

Java 的数据类型分为两派：基本数据类型（值类型，如 `int`、`double`）和引用类型（如 `Integer`、`String`、自定义对象）。

我们可以用下面这个内存字符画来看看它们的本质区别：

Plaintext

```
代码：
int a = 10;
Integer b = new Integer(10); // 假设显式创建

【 栈内存 (Stack) 】                  【 堆内存 (Heap) 】
+-----------------+
| a [ 10 ]        |  <-- 值类型：变量直接拿着真实数据，全在栈里，访问极快。
+-----------------+
| b [ 0x00A1 ]    | -----> 指向 ----->  +--------------------+
+-----------------+                     | 对象头 | 数据 [ 10 ] |
                                        +--------------------+
                                 <-- 引用类型：变量拿的是一把“钥匙”（内存地址），真实数据存在堆里。
```

**自动装箱/拆箱的 2 大经典坑点：**

- **坑点一：整型缓存池（Integer Cache）的 `==` 陷阱**

  Java 为了省内存，默认把 `-128` 到 `127` 的 `Integer` 缓存了。

  Java

  ```
  Integer x = 100, y = 100; 
  System.out.println(x == y); // true，都在缓存池里，拿的是同一把钥匙
  
  Integer m = 200, n = 200; 
  System.out.println(m == n); // false，超出缓存范围，各自在堆里 new 了新对象，地址不同
  ```

- **坑点二：无情引发 NullPointerException (NPE)**

  当包装类为 `null` 时，如果触发自动拆箱（赋值给基本类型，或进行数学运算），底层会调用 `.intValue()`，直接抛出空指针异常。

  Java

  ```
  Integer price = null;
  int finalPrice = price; // 💥 编译不报错，运行直接 NullPointerException
  ```

------

### 2. 重载（Overload） vs 重写（Override）

这两个概念的名字很像，但发生的“时空”完全不同。

| **维度**        | **重载（Overload）**                           | **重写（Override）**                           |
| --------------- | ---------------------------------------------- | ---------------------------------------------- |
| **发生位置**    | 同一个类中                                     | 父子类之间（继承或实现接口）                   |
| **方法名**      | 必须相同                                       | 必须相同                                       |
| **参数列表**    | **必须不同**（个数、类型或顺序不同）           | **必须完全相同**                               |
| **返回值/异常** | 无限制                                         | 返回值必须相同或为其子类；抛出的异常不能更宽泛 |
| **多态表现**    | 编译时多态（写代码时根据参数就确定了调用哪个） | 运行时多态（运行到哪，看 new 的是哪个子类）    |

**实战场景：**

- **重载**：`String.valueOf()` 方法。不管你传 `int`、`char` 还是 `Object`，它都能处理，给调用者提供统一的 API 体验。
- **重写**：各大框架里的模板方法模式。父类定义了 `doGet()` 的骨架，子类重写 `doGet()` 注入自己的具体业务逻辑。

------

### 3. 面向对象三大特性：封装 / 继承 / 多态

- **封装（Encapsulation）**：**隐藏内部实现，暴露安全接口。** 把属性私有化（`private`），通过 `getter/setter` 控制读写权限，防止外部随意修改导致数据脏乱。
- **继承（Inheritance）**：**代码复用，确立层级关系。** 子类可以继承父类的非私有属性和方法（`extends`）。在 Java 中只能单继承，防止多重继承带来的“菱形调用冲突”。
- **多态（Polymorphism）**：**同一个接口，多种不同形态。**
  - **核心口诀**：“声明为父类，实例化为子类”。（例如 `List<String> list = new ArrayList<>();`）
  - **好处**：极致的扩展性。如果以后想换成 `LinkedList`，只需要改 `new` 的那部分，后面调用 `list.add()` 的业务代码一行都不用改。

------

### 4. 抽象类 vs 接口

在 Java 8 之前，它们的界限非常清晰：抽象类是“半成品”，接口是“纯契约”。但 Java 8 之后，接口越来越像抽象类了。

- **核心设计哲学区别**：

  - **抽象类**表达的是 **"Is-a" (是一个)** 的关系。比如 `Dog` 继承 `Animal`。它解决的是代码复用问题，可以有普通的成员变量。
  - **接口**表达的是 **"Can-do" (能做什么)** 的能力契约。比如 `Dog` 实现 `Runnable`（能跑），`Flyable`（能飞）。它解决的是系统解耦和扩展问题。

- **Java 8 的变革 (`default` / `static` 方法)**：

  过去，一旦接口被发布，你想在接口里加一个新方法，所有实现这个接口的类都会疯狂报错（必须实现新方法）。

  为了解决这个灾难，Java 8 引入了 `default` 方法。接口可以提供方法的默认实现，现有的实现类不需要改动任何代码就能直接继承这个默认能力。`static` 方法则允许在接口中写一些跟该接口相关的工具方法。

------

### 5. == 与 equals、hashCode 契约

这是集合框架（尤其是 `HashMap`）相关面试题的绝对前置知识点。

- `==`：如果是基本类型，比较的是**值**；如果是引用类型，比较的是**内存地址**。
- `equals()`：`Object` 类默认的 `equals` 其实内部就是用 `==` 实现的。但在 `String` 等类中，它被重写了，用来比较**内容逻辑上是否相等**。

**终极契约：为什么重写 `equals` 必须重写 `hashCode`？**

我们可以用一个“查字典”的字符画来理解这个契约的强制性。假设你用自定义对象 `Person(name, age)` 作为 `HashMap` 的 Key。

Plaintext

```
如果只重写 equals，不重写 hashCode：

步骤 1: map.put(new Person("张三", 18), "北京");
HashMap 算了一下这个对象的 hashCode，假设计算为 101，放进了【101号抽屉】。

   【101号抽屉】 --> [ Person("张三", 18) = "北京" ]
   【105号抽屉】 --> 空

步骤 2: String city = map.get(new Person("张三", 18));
你 new 了一个逻辑上完全一样的新对象去查询。
因为你没重写 hashCode，系统根据新的内存地址算出了 hashCode 为 105。
HashMap 直接去拉开【105号抽屉】，发现里面是空的！

结果：返回 null。明明"张三"存进去了，却死活查不出来。
```

**契约总结：**

1. **如果两个对象 `equals` 返回 `true`，它们的 `hashCode` 必须完全相等。**（保证找抽屉的时候找得准）。
2. **如果两个对象的 `hashCode` 相等，它们 `equals` 不一定返回 `true`。**（这就是哈希冲突/碰撞，不同的对象刚好被分到了同一个抽屉里，需要在抽屉里顺着链表继续用 `equals` 挨个比对）。

这 5 个问题梳理下来，你的 Java 基础八股文体系已经建立起骨架了。面试官在问完 `hashCode` 契约后，有极高的概率会顺理成章地深挖 **Java 8 中 HashMap 的底层数据结构（数组 + 链表 + 红黑树）和扩容机制**。

你需要我用字符画为你推演一遍 HashMap 从插入数据到触发红黑树转换的完整工作流吗？