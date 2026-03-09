## Java 基础高频知识点清单（面试必考轮廓）

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

## 二、String 体系

### 1. String、StringBuilder、StringBuffer 区别与场景

这是最基础的送分题，用一张表就能完美总结：

| **维度**     | **String**                            | **StringBuilder**                  | **StringBuffer**                           |
| ------------ | ------------------------------------- | ---------------------------------- | ------------------------------------------ |
| **可变性**   | **不可变**（每次修改都会生成新对象）  | **可变**（在原对象上修改）         | **可变**（在原对象上修改）                 |
| **线程安全** | 绝对安全（因为不可变）                | **不安全**                         | **安全**（核心方法加了 `synchronized` 锁） |
| **执行效率** | 最低（频繁拼接会导致内存溢出）        | **最高**                           | 较低（因为锁的开销）                       |
| **使用场景** | 少量字符串操作、作为 `HashMap` 的 key | **单线程**下频繁进行大量字符串拼接 | **多线程**下共享缓冲区的字符串拼接         |

**实战避坑**：在现代 Java 开发中，`StringBuffer` 已经极少使用了。如果在单线程方法内部拼接字符串，闭着眼睛用 `StringBuilder`；如果是多线程环境，通常会使用 `ThreadLocal` 包装 `StringBuilder` 或者使用其他并发组件，因为 `StringBuffer` 的方法级粗粒度锁性能太差。

------

### 2. 字符串常量池与 `new String("a")` 到底创建了几个对象？

这是八股文里的高频陷阱题。答案是：**1 个或 2 个**。

这取决于**字符串常量池（String Pool）**中是否已经存在 `"a"` 这个字面量。

我们用字符画来看看 `String s = new String("a");` 在 JVM 底层（以 JDK 8 为例）到底发生了什么：

Plaintext

```
代码执行：String s = new String("a");

【 堆内存 (Heap) 】
                                
   +------------------------------------+
   | 2. new 出来的 String 对象 (s 指向它)  |
   |    内部包含一个指针，指向常量池的 "a"    |
   +------------------------------------+
                   | 引用
                   v
==================================================

【 字符串常量池 (String Pool) - 也是在堆中 】

   +------------------------------------+
   | 1. 字面量对象 "a"                    | <-- 如果常量池以前没有 "a"，这一步就会创建它。
   +------------------------------------+       如果已经有了，就省去这一步，直接复用。
```

**解析：**

1. **编译期/类加载期**：JVM 看到字面量 `"a"`，会检查常量池。如果没有，就在常量池里创建一个 `"a"` 对象。
2. **运行期**：遇到 `new` 关键字，必定会在堆内存中开辟一块新空间，创建一个全新的 `String` 对象，并让这个新对象底层的字符数组引用指向常量池中的 `"a"`。

**关于 `intern()` 方法：**这是一个“主动白嫖”常量池的方法。`s.intern()` 的逻辑是：去常量池里找找有没有和 `s` 内容一样的字符串。如果有，直接返回常量池里的那个引用；如果没有，就把 `s` 的引用塞进常量池，并返回。

------

### 3. String 为什么设计成不可变？

面试官问这个问题，是想看你的架构思维。Java 之父 James Gosling 如此设计，主要基于以下三大基石：

#### A. 线程安全 (Thread Safety)

不可变对象天生就是线程安全的。

在并发场景下，多个线程可以毫无顾忌地共享同一个 `String` 实例，完全不需要加锁 `synchronized`，也不用担心它被其他线程悄悄修改。这极大简化了多线程编程的复杂度。

#### B. 缓存与性能优化 (Caching & String Pool)

正因为 `String` 是不可变的，**字符串常量池**才得以存在。

设想一下，如果 `String` 是可变的：

Java

```
String s1 = "hello";
String s2 = "hello";
// s1 和 s2 指向常量池里的同一个 "hello"
s1.setValue("world"); // 如果允许修改
```

这时候 `s2` 的值也会莫名其妙变成 `"world"`，整个 Java 世界就全乱套了。只有不可变，才能放心地让多个变量共享同一块内存。

另外，`String` 极常被用作 `HashMap` 的 key。因为不可变，它的 `hashCode` 只需要在第一次使用时计算一次，然后**缓存**起来，以后直接读取。这也是 `String` 作为散列表 key 性能极高的根本原因。

#### C. 安全性与类加载 (Security & Class Loading)

`String` 广泛用于 Java 的核心底层机制中，比如：

- **网络连接**：传入主机名、URL。
- **数据库连接**：传入 DB 的 URL、用户名、密码。
- **文件系统**：传入文件路径。
- **反射与类加载机制**：`Class.forName("com.example.MyClass")`。

如果 `String` 是可变的，黑客就可以在系统校验完文件路径或 SQL 语句后，在执行前的一瞬间开个新线程将其篡改掉（这就是著名的 TOCTOU 漏洞 - Time of Check to Time of Use）。不可变性彻底封死了这条利用链。

------

## 三、集合框架

### 1. ArrayList vs LinkedList：时间复杂度与场景

这是最经典的基础对比。面试官通常不仅要听复杂度，还要听你对底层内存结构的理解。

| **操作**               | **ArrayList (动态数组)**       | **LinkedList (双向链表)**          |
| ---------------------- | ------------------------------ | ---------------------------------- |
| **底层结构**           | 连续内存空间                   | 分散内存，通过指针（引用）连接     |
| **随机访问 (get/set)** | **$O(1)$** (根据索引直接定位)  | $O(N)$ (需从头/尾遍历)             |
| **头部插入/删除**      | $O(N)$ (需移动后续所有元素)    | **$O(1)$** (只需改变指针)          |
| **尾部插入**           | $O(1)$ (偶尔触发扩容需 $O(N)$) | $O(1)$                             |
| **中间插入/删除**      | $O(N)$ (需移动后续元素)        | $O(N)$ (查找耗时，改变指针 $O(1)$) |
| **空间开销**           | 尾部可能预留部分空闲空间       | 需额外存储 `prev` 和 `next` 指针   |

**典型使用场景：**

- **`ArrayList`**：99% 的日常开发场景！极度适合**查询多、尾部追加多**的场景。由于内存连续，它对 CPU 缓存（CPU Cache）极其友好，实际运行速度往往吊打 `LinkedList`。
- **`LinkedList`**：只有在**需要频繁在头部或中间插入/删除**，且数据量极大时才会考虑。也可以被用作栈（Stack）、队列（Queue）或双端队列（Deque）。

------

### 2. HashMap 深度拆解：结构、负载因子与扩容

`HashMap` 是整个 Java 集合的心脏，它的底层设计融合了多种数据结构的精华。

#### A. 底层结构（JDK 1.8）

**数组 + 链表 + 红黑树**。

当多个 Key 经过哈希计算后落在数组的同一个槽位时（哈希冲突），它们会形成链表。当链表太长影响查询效率时，会转化为红黑树。

Plaintext

```
【 HashMap 底层架构简图 】

数组索引
 [ 0 ] -> null
 [ 1 ] -> [Node] -> [Node] -> [Node]  (哈希冲突，形成单向链表)
 [ 2 ] -> null
 [ 3 ] -> [Node]                      (红黑树：当链表长度 >= 8 且数组长度 >= 64 时触发转换)
            /   \
        [Node] [Node]
         /       \
      [Node]    [Node]
```

#### B. 负载因子（Load Factor）

默认值是 **0.75**。

- **为什么是 0.75？** 这是空间利用率和时间复杂度（哈希冲突概率）之间的一个完美折中。根据泊松分布，在 0.75 的负载因子下，链表长度达到 8 的概率极小（千万分之一级别）。
- 如果设为 1.0：空间利用率高，但极易发生冲突，链表变长，查询变慢。
- 如果设为 0.5：冲突少了，但动不动就扩容，极其浪费内存。

#### C. 扩容机制（Resize）

当 `HashMap` 中的元素总数超过了 `数组容量 × 负载因子`（例如 $16 \times 0.75 = 12$），就会触发扩容。

1. **开辟新数组**：容量直接翻倍（比如从 16 变成 32），保证容量永远是 2 的次幂。
2. **数据迁移（Rehash）**：JDK 1.8 做了一个极其优雅的优化。因为容量翻倍，节点在新数组中的位置只有两种可能：
   - **原位置**（索引不变）。
   - **原位置 + 旧容量**（比如原来在索引 3，旧容量 16，新位置就是 19）。
   - 底层通过高位运算 `(e.hash & oldCap) == 0` 来以 $O(1)$ 的速度决定节点去哪，彻底避免了 JDK 1.7 中死循环的 Bug，且不再需要重新计算 Hash 值。

------

### 3. HashMap vs Hashtable vs ConcurrentHashMap

这三者是针对“线程安全”这个考点量身定制的演进路线。

| **特性**           | **HashMap**                           | **Hashtable (已淘汰)**            | **ConcurrentHashMap**                          |
| ------------------ | ------------------------------------- | --------------------------------- | ---------------------------------------------- |
| **线程安全**       | ❌ 绝对不安全                          | ✅ 安全 (全表加 `synchronized` 锁) | ✅ 安全 (分段锁/节点锁，极高并发)               |
| **Null Key/Value** | ✅ 允许 1 个 Null Key，多个 Null Value | ❌ 抛 NullPointerException         | ❌ 抛 NullPointerException                      |
| **并发性能**       | N/A                                   | 极低（多线程串行访问）            | **极高**                                       |
| **底层实现**       | 数组+链表+红黑树                      | 数组+链表                         | JDK 1.8: 数组+链表+红黑树 (CAS + Synchronized) |

**关于 `ConcurrentHashMap` 的进化：**

- **JDK 1.7**：采用 `Segment` 分段锁。把大数组切分成多个小数组，每次只锁一个小数组，并发度默认 16。
- **JDK 1.8**：废弃了 `Segment`，结构变得和普通 `HashMap` 一样。锁的粒度细化到了**每一个数组槽位的头节点（Node）**。大量使用无锁算法 `CAS` 来插入新节点，只有在发生哈希冲突需要挂载链表或红黑树时，才用 `synchronized` 锁住头节点。性能直接拉满！

------

### 4. 其他核心集合速览

为了保证知识体系的完整，这几个实现的底层逻辑你也必须做到脱口而出：

- **`HashSet`**：底层其实就是一个 `HashMap`！它的所有元素都存放在 `HashMap` 的 Key 里，Value 则统一填充一个名叫 `PRESENT` 的虚无 Object 占位符。
- **`TreeSet`**：底层是一个 `TreeMap`（红黑树）。它能保证存入的元素是有序的（自然排序或自定义 Comparator 排序），插入和查询的时间复杂度都是 $O(\log N)$。
- **`LinkedHashMap`**：继承自 `HashMap`。在 `HashMap` 的结构之外，**额外维护了一个双向链表**，把所有节点串起来。这就保证了它可以按照**插入顺序**或者**访问顺序**（开启 `accessOrder = true` 时，常用于手写 LRU 缓存淘汰算法）进行遍历。

## 泛型与反射

### 1. 泛型擦除与 `List<Object>` vs `List<String>`

**泛型擦除（Type Erasure）**是 Java 泛型最核心的底层机制：**Java 的泛型只存在于编译期，运行期会被彻底抹掉。**

- **编译期检查**：编译器会检查你往 `List<String>` 里塞的是不是字符串，如果塞 `Integer`，直接编译报错，这叫“语法糖”，为了让你写代码时少踩坑。
- **运行时擦除**：一旦编译成 `.class` 字节码文件，所有的 `<String>`、`<T>` 统统消失，全部变成裸类型（Raw Type），通常退化为 `Object`（如果定义了上限比如 `<T extends Number>`，则退化为 `Number`）。

Plaintext

```
【 泛型擦除直观演示 】

开发者写的 Java 代码：
List<String> list = new ArrayList<>();
list.add("Hello");
String s = list.get(0);

编译器处理后的字节码（反编译看本质）：
List list = new ArrayList();       <-- 泛型没了！变成了 Object
list.add("Hello");
String s = (String) list.get(0);   <-- 编译器偷偷帮你加了强转！
```

**`List<Object>` vs `List<String>` 是否可以互转？**

**绝对不可以！** 这是面试高频坑题。

虽然 `String` 是 `Object` 的子类，但 `List<String>` **绝不是** `List<Object>` 的子类。Java 泛型是**不型变（Invariant）**的。

假设允许互转，会发生什么灾难？

Java

```
List<String> strList = new ArrayList<>();
// 假设下面这行不报错（实际上编译会直接拦截）：
List<Object> objList = strList; 

// 灾难来了：
objList.add(100); // objList 认为自己装的是 Object，塞个 Integer 合情合理
String s = strList.get(0); // 💥 strList 去取的时候，拿到个 100 强转 String，直接 ClassCastException！
```

------

### 2. `? extends T` 与 `? super T`（PECS 原则）

既然 `List<String>` 不能赋给 `List<Object>`，那我们要写一个通用的方法去处理不同类型的集合怎么办？这就引入了**通配符上下界**。

这里有一个业界公认的神级口诀：**PECS (Producer Extends, Consumer Super)**。

Plaintext

```
继承关系： Food (食物) -> Fruit (水果) -> Apple (苹果)

1. 上界通配符： <? extends Fruit>  (它是生成器 Producer)
   【含义】：这是一个篮子，里面装的 全是 Fruit 或它的子类。
   【限制】：你只能往外拿 (get)，绝对不能往里存 (add)！
   【原因】：编译器只知道里面是 Fruit 的子类，但不知道具体是 Apple 还是 Banana。
            如果你试图 add(new Banana())，但篮子里本来装的是 Apple，就崩了。所以干脆禁止存。
   【用法】：适合作为参数，只提供数据供读取。

2. 下界通配符： <? super Fruit>    (它是消费者 Consumer)
   【含义】：这是一个篮子，里面装的是 Fruit 或它的父类。
   【限制】：你可以放心地往里存 (add) Fruit 及其子类，但往外拿 (get) 的时候，只能用 Object 接收。
   【原因】：因为篮子里要求的下限是 Fruit，你塞 Apple、Banana 都是安全的。
            但你往外拿的时候，编译器不知道拿出来的到底是 Fruit 还是 Food 甚至是 Object，只能用顶级父类 Object 兜底。
   【用法】：适合用来接收数据，不断往里塞东西。
```

------

### 3. 反射能做什么？

如果说正常的面向对象开发是“按规矩办事”，那么反射（Reflection）就是 Java 赋予程序的“上帝视角”和“黑客特权”。

**反射的核心能力：在运行状态中，对于任意一个类，都能够知道这个类的所有属性和方法；对于任意一个对象，都能够调用它的任意方法和属性（甚至连 `private` 私有属性都能强行扒开）。**

**框架中的典型应用场景：**

- **Spring (IoC / DI)**：你在类上写个 `@Service`，在属性上写个 `@Autowired`。Spring 启动时，通过反射扫描所有的类，发现有这些注解，就用反射调用 `Constructor.newInstance()` 帮你把对象 `new` 出来，然后再用 `Field.set()` 强行把依赖项塞进你的私有变量里。
- **MyBatis (ORM)**：数据库查出来一行数据（比如叫 `user_name`），MyBatis 用反射解析你传入的 `User` 类，找到对应的 `userName` 属性，利用 `Method.invoke()` 调用 `setUserName()` 方法，把数据完美封装成对象返回给你。

------

### 4. 反射常见四大核心类

反射的本质，就是把 Java 类中的各种成分映射成相应的 Java 对象。

Plaintext

```
假设我们有一个类：
class User {
    private String name;               <-- 映射为 Field 对象
    public User() {}                   <-- 映射为 Constructor 对象
    public void sayHello() {}          <-- 映射为 Method 对象
}
```

- **`Class` (类对象)**：这是反射的入口。每个被加载的类在 JVM 里都有唯一的一个 `Class` 实例（比如 `User.class`），里面保存了该类的所有结构信息。
- **`Constructor` (构造器对象)**：代表类的构造方法。通过 `constructor.newInstance()` 可以绕过普通的 `new` 关键字来动态创建对象。
- **`Field` (成员变量对象)**：代表类的属性。最霸道的方法是 `field.setAccessible(true)`（暴力反射），调用后可以直接修改 `private` 修饰的私有变量，打破封装性！
- **`Method` (方法对象)**：代表类的方法。核心操作是 `method.invoke(对象, 参数)`，可以在运行时动态执行对象的某个方法。

反射和泛型在实际框架中经常结合使用（比如通过反射获取泛型的真实类型，虽然泛型被擦除了，但在类的签名、方法参数上依然保留了泛型信息，可以通过 `ParameterizedType` 拿到）。

## 异常机制

异常处理是 Java 后端开发中最日常的编码环节，也是面试中考察系统设计规范的重要切入点。我们先用一张字符画理清 Java 异常家族的宏观家谱：

Plaintext

```
【 Java 异常体系大满贯 】

              Throwable (顶级大 Boss)
             /         \
      Error (系统级)   Exception (程序级)
     (如 OOM,        /         \
    StackOverflow)  /           \
                   /             \
        RuntimeException     非 RuntimeException
         (Unchecked 异常)     (Checked 异常)
        (如 NPE, IndexOut)   (如 IOException, SQLException)
```

### 1. Checked 异常 vs Unchecked 异常

- **Checked 异常（受检异常）**：

  在编译阶段就强制要求处理（要么 `try-catch` 捕获，要么在方法签名上 `throws` 抛出，否则代码直接爆红无法运行）。它们通常代表那些**程序之外、程序员无法绝对控制的外部客观条件**，比如目标文件不存在、网络突然断开、数据库连接失败。

- **Unchecked 异常（非受检异常）**：

  即所有的 `RuntimeException` 及其子类。编译器完全不会管它。它们通常代表**代码逻辑本身存在的 Bug**，比如空指针（NPE）、数组越界、参数格式错误。这类异常理应通过更严谨的代码逻辑（如判空、校验）来避免，而不是到处写 `try-catch`。

**🎯 核心面试陷阱（Spring 事务机制）**：

在 Spring 框架中，默认情况下 `@Transactional` 事务只会对 **Unchecked 异常 (RuntimeException / Error)** 触发回滚！如果你的代码抛出了 Checked 异常，事务默认是**不会回滚**的。这也是为什么我们在业务开发中，遇到异常基本都会包装成 `RuntimeException` 往外抛。

------

### 2. 自定义业务异常的实践

在企业级项目中，Java 原生的异常类型完全不够用。我们需要自定义业务异常来精准表达业务语义。设计 `ClientException` 和 `ServiceException` 是一种非常经典且专业的架构分层实践。

**为什么要继承 `RuntimeException`？**

绝大多数自定义业务异常都会继承 `RuntimeException`。因为业务报错不需要在每一层方法调用上都写烦人的 `throws`，而且能完美契合刚才提到的 Spring 默认事务回滚机制。

- **ClientException (客户端异常)**：对应 HTTP 状态码 `4xx` 系列。

  代表**调用方（前端或外部系统）犯的错**，比如参数校验失败、密码错误、余额不足。这类异常抛出后，系统通常不需要报警，只需记录 info 级别日志，并直接把明确的错误信息和错误码返回给前端即可。

- **ServiceException (服务端异常)**：对应 HTTP 状态码 `5xx` 系列。

  代表**服务器内部出了严重问题**，比如数据库宕机、第三方 API 调用超时、磁盘写满。这类异常必须记录 error 级别日志，保留完整的堆栈信息，甚至触发系统的监控报警（如发送钉钉/飞书通知）。

代码结构通常会包含明确的错误码：

Java

```
// 基础的业务异常类
public abstract class BaseException extends RuntimeException {
    private final String errorCode;
    private final String errorMessage;
    
    public BaseException(String errorCode, String errorMessage) {
        super(errorMessage);
        this.errorCode = errorCode;
        this.errorMessage = errorMessage;
    }
}

// 客户端导致的异常
public class ClientException extends BaseException {
    public ClientException(String errorCode, String errorMessage) {
        super(errorCode, errorMessage);
    }
}
```

------

### 3. try-with-resources 的作用

在 Java 7 之前，处理文件流（Stream）、数据库连接（Connection）等需要手动关闭的底层资源时，代码就像又臭又长的裹脚布。`try-with-resources` 的出现，简直是拯救代码洁癖的终极语法糖。

我们用对比字符画来看一下它的威力：

Plaintext

```
【 旧时代：臃肿的 try-catch-finally 】        【 新时代：try-with-resources 】

InputStream in = null;                      |  try (InputStream in = new FileInputStream("a.txt")) {
try {                                       |      // 正常读取数据
    in = new FileInputStream("a.txt");      |  } catch (IOException e) {
    // 读取数据                              |      // 处理读取时的异常
} catch (IOException e) {                   |  } 
    // 处理异常                              |  
} finally {                                 |  // 🚀 核心点：完全不需要写 finally 块！
    if (in != null) {                       |  // 只要资源类实现了 java.lang.AutoCloseable 接口，
        try {                               |  // 离开 try 代码块时，JVM 必定会自动调用 in.close()。
            in.close(); // 极度臃肿！       |  // 哪怕 try 里面发生了异常，或者执行了 return，也会先关门！
        } catch (IOException e) {           |
            // 连关个门都要再写一次捕获！    |
        }                                   |
    }                                       |
}                                           |
```

**隐藏的高级功能：异常抑制 (Suppressed Exceptions)**

在旧写法中有一个极其恶心的 Bug：如果 `try` 块里抛了异常，同时 `finally` 块里的 `close()` 执行时也抛了异常，那么 `finally` 里的异常会把真正引发问题的 `try` 块异常给**吞掉（覆盖）**，导致排查极其困难。

而 `try-with-resources` 会保留 `try` 块里的核心主体异常，把 `close()` 时发生的次要异常通过 `e.addSuppressed()` 追加到主体异常后面，保证了完整的排查线索绝不丢失。

## IO 与序列化

### 1. BIO 与 NIO 的基本概念

要理解这两个概念，最经典的类比就是**“餐厅点餐模型”**。

- **BIO (Blocking I/O - 同步阻塞)**：

  就像去一家老式餐厅，**一个服务员只能服务一桌客人**。客人如果拿着菜单犹豫半小时，服务员就只能傻站在旁边等（线程阻塞），什么也干不了。并发量一上来，餐厅就得雇佣海量的服务员（疯狂开启新线程），最终导致系统资源耗尽。

- **NIO (Non-blocking I/O - 同步非阻塞)**：

  现代餐厅的点餐方式。**一个服务员可以管理几十桌客人**。客人看菜单时，服务员去忙别的；客人决定好了，按一下桌上的呼叫铃，服务员再过去处理。这就是**多路复用**的精髓。

#### NIO 的三大核心组件

在 NIO 中，数据的流转不再是直接对着流（Stream）傻等，而是通过下面三个核心件配合完成：

1. **Buffer（缓冲区）**：NIO 的数据都是在 Buffer 中处理的。它本质上是一块内存（底层是数组），可以进行读写双向操作，而不是像传统 BIO 那样单向流动。
2. **Channel（通道）**：类似于 BIO 中的流（Stream），但 Channel 是双向的。数据必须从 Channel 读取到 Buffer，或者从 Buffer 写入到 Channel。
3. **Selector（多路复用器）**：**NIO 的灵魂**。它是那个“眼观六路的服务员”。一个线程只需要管理一个 Selector，Selector 会不断轮询注册在其上的多个 Channel。只有当 Channel 真正有读写事件发生时，线程才去处理，极大地降低了系统开销。

Plaintext

```
【 NIO 多路复用 (I/O Multiplexing) 核心运行图 】

        客户端 1 -----(连接)-----> [ Channel 1 ] --+
                                                  |
        客户端 2 -----(连接)-----> [ Channel 2 ] --+---> 【 Selector 】 (多路复用器)
                                                  |           |
        客户端 3 -----(连接)-----> [ Channel 3 ] --+           | (不断轮询谁有动静)
                                                              V
                                                      【 单一工作线程 】
                                                  (发现 Channel 2 有数据，
                                                   通过 Buffer 进行读写)
```

------

### 2. Java 序列化的用途与致命缺点

**序列化（Serialization）的本质就是把内存里的 Java 对象，变成一串二进制的“字节序列”，目的是为了把对象存入磁盘（持久化）或者通过网络发送给别的机器**。反序列化则是把字节序列拼回 Java 对象。

在 Java 中，最原生的做法是让类实现一个空接口 `java.io.Serializable`作个标记。

#### 为什么现在大家都在疯狂吐槽并弃用原生 `Serializable`？

虽然它用起来简单，但在企业级开发和面试中，它满身都是雷点：

1. **无法跨语言（语言绑定）**：

   Java 原生序列化出来的字节码，只有 Java 语言自己能看懂。如果你想把对象传给前端的 JavaScript，或者后端的 Python 服务，对方直接原地抓瞎。

2. **序列化后的体积非常臃肿**：

   Java 序列化不仅保存数据本身，还会保存类名、属性类型、复杂的继承结构等大量元数据。这导致它在网络传输时极其消耗带宽，性能极差。

3. **极其脆弱的版本控制 (`serialVersionUID` 的坑)**：

   如果你序列化了一个对象存到硬盘上，第二天给这个类新增了一个字段。再次反序列化时，如果之前没有手动固定 `serialVersionUID`，JVM 会认为类的版本变了，直接抛出 `InvalidClassException` 拒绝反序列化。

4. **严重的安全漏洞（反序列化攻击）**：

   这是最致命的一点。如果攻击者伪造了一段恶意的字节序列传给服务器，Java 在反序列化时会极其“听话”地实例化对象并可能触发某些恶意类的构造函数或静态代码块，导致服务器被直接拿走最高权限（RCE 漏洞）。

## Java 8 常见特性

### 1. Lambda 表达式与函数式接口 (@FunctionalInterface)

**核心理念：把“行为（方法）”当作参数传递。**

在 Java 8 之前，如果我们想给某个方法传递一段逻辑，只能傻傻地 `new` 一个匿名内部类。Lambda 的出现彻底终结了这个臃肿的写法。

**什么是函数式接口？**

只要一个接口**有且仅有一个抽象方法**（SAM, Single Abstract Method），它就是一个函数式接口。`@FunctionalInterface` 只是一个编译期检查注解（类似 `@Override`），如果你在里面写了第二个抽象方法，编译器会直接报错，防止别人破坏你的契约。

Java

```
@FunctionalInterface
public interface MathOperation {
    int operate(int a, int b);
}

// 传统写法（匿名内部类）：
MathOperation addOld = new MathOperation() {
    @Override
    public int operate(int a, int b) {
        return a + b;
    }
};

// Lambda 优雅写法：
// (参数) -> { 执行逻辑 }
MathOperation addNew = (a, b) -> a + b; 
```

------

### 2. Stream 常用操作与“惰性求值”的底层智慧

Stream 不是数据结构，它不存数据！它是一条**流水线（Pipeline）**。

掌握 Stream 最常用的三板斧：

- **`filter`**：过滤。留下符合条件的，踢掉不要的。
- **`map`**：映射/转换。把 A 类型变成 B 类型（比如把 `User` 对象流变成 `String` 用户名流）。
- **`collect`**：终结。把流水线上的结果打包收集起来（比如转成 `List` 或 `Map`）。

#### 💥 核心考点：什么是惰性求值（Lazy Evaluation）？

面试官极爱问：“如果在 Stream 里写了好多 `filter` 和 `map`，但是不写最后的 `collect`，代码会执行吗？”

答案是：**绝对不会执行！**

Plaintext

```
【 Stream 惰性求值字符画演示 】

假设有一堆水果：[ 🍎, 🍏, 🍎, 🍌 ]

代码：
Stream<水果> stream = fruits.stream()
    .filter(f -> f.颜色 == 红)    <-- 中间操作 (阀门1：只放行红色的)
    .map(f -> 榨汁(f));           <-- 中间操作 (阀门2：把水果变成汁)

此时，机器根本没有启动！所有的动作只是把“处理图纸”画好了。

只有当你调用【终端操作】时：
List<果汁> result = stream.collect(Collectors.toList()); <-- 启动开关！

流水线才开始【串行】拉动数据：
进 🍎 -> 阀门1(通过) -> 阀门2(变🍎汁) -> 收集！
进 🍏 -> 阀门1(卡住, 丢弃)
进 🍎 -> 阀门1(通过) -> 阀门2(变🍎汁) -> 收集！
进 🍌 -> 阀门1(卡住, 丢弃)

(这种拉动式设计极其高效，如果终端操作是 findFirst()，它找到第一个 🍎汁 就直接停机了，后面的数据根本不会走流水线！)
```

------

### 3. Optional 的优雅用法（消灭 NPE）

`NullPointerException` 是 Java 程序员一生的宿敌。`Optional` 本质上是一个“可能包含也可能不包含非空值”的**包装盒**。

**面试防坑指南：千万不要把 `Optional` 用成了高级版的 `if (obj != null)`！**

- **反面教材（被面试官嫌弃的写法）**：

  Java

  ```
  Optional<User> optUser = Optional.ofNullable(user);
  if (optUser.isPresent()) { // 这和写 user != null 有什么区别？？
      return optUser.get().getName();
  } else {
      return "Unknown";
  }
  ```

- **满分写法（利用函数式链式调用）**：

  利用 `map` 来解包，利用 `orElse` 或 `orElseGet` 来提供默认兜底策略。

  Java

  ```
  // 语义：把 user 装进盒子里，如果盒子不为空，就把名字抽出来，如果为空，就返回默认值。
  String name = Optional.ofNullable(user)
                        .map(User::getName)
                        .orElse("Unknown");
  ```

- **高阶场景：如果为空就直接抛出我们上一轮聊过的自定义业务异常！**

  Java

  ```
  User result = Optional.ofNullable(userRepository.findById(id))
                        .orElseThrow(() -> new ServiceException("404", "用户不存在"));
  ```

Java 8 的这三大件在日常开发中经常会被组合在一起使用（比如把数据库查出来的一个 `List<User>` 变成一个以 `userId` 为 Key，`User` 为 Value 的 `Map`）。

在真实的笔试或面试手撕代码环节，经常会让你用 Stream API 来完成这种数据结构的转换。你需要我马上给你出一个常见的 **“List 转 Map 且处理 Key 冲突”** 的实战小测验，带你把这段代码敲熟吗？