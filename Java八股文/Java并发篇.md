# Java并发

按你简历里写的「并发」这条，整理成**知识点清单 + 一轮拷打题**，方便你对着背和练。

---

## 一、并发知识点清单（对应简历）

### 1. JUC 与线程基础
- 线程的创建方式（Thread、Runnable、Callable）、线程状态与转换
- `synchronized` 用法（实例锁、类锁）、锁升级（偏向 → 轻量级 → 重量级）、锁粗化/锁消除
- `volatile` 语义：可见性、禁止指令重排；不保证原子性
- `ThreadLocal` 原理、内存泄漏风险；与 TTL 的关系（TTL 解决的是线程池场景下的继承问题）

### 2. 线程池
- 核心参数：`corePoolSize`、`maximumPoolSize`、`keepAliveTime`、`unit`、`workQueue`、`threadFactory`、`rejectedExecutionHandler`
- 执行流程：先 core → 入队 → 再开非核心线程 → 满则拒绝
- 常见队列：`LinkedBlockingQueue`、`ArrayBlockingQueue`、`SynchronousQueue`、`DelayedWorkQueue`
- 拒绝策略：AbortPolicy、CallerRunsPolicy、DiscardPolicy、DiscardOldestPolicy；自定义策略（如你项目里排队限流）
- 为什么用线程池：复用、控并发、统一管理；和「8 个专用线程池」的设计思路

### 3. CAS 与 AQS
- **CAS**：Compare-And-Swap，无锁更新；ABA 问题及解决（版本号/StampedReference）；`Unsafe` 与原子类（如 `AtomicInteger`）
- **AQS**：抽象队列同步器；state + 双向 CLH 队列；独占/共享模式；`ReentrantLock`、`ReentrantReadWriteLock`、`Semaphore`、`CountDownLatch`、`CyclicBarrier` 与 AQS 的关系
- **ReentrantLock**：可重入、公平/非公平、`lock()`/`unlock()`、`tryLock()`、Condition；与 `synchronized` 的对比

### 4. 悲观锁 vs 乐观锁
- 悲观锁：假定会冲突，先加锁再操作（synchronized、ReentrantLock、DB 行锁/表锁）
- 乐观锁：假定冲突少，先改再校验（版本号、CAS）；典型场景：库存扣减、更新带 version 的实体
- 选型：冲突多、临界区大倾向悲观；读多写少、冲突少倾向乐观

### 5. 分布式锁
- 要解决的问题：多进程/多机下互斥
- Redis 实现：SET key value NX EX、Lua 保证「加锁+过期」原子性、删除时校验 value 防误删；Redisson 看门狗续期
- 注意点：过期时间、可重入、锁续期、单点/红锁
- 你项目里：限流用 ZSET+Lua，和「分布式锁用 Lua」是同一类「原子性」考点

### 6. CompletableFuture
- 创建：`supplyAsync`、`runAsync`（可指定线程池）
- 编排：`thenApply`/`thenAccept`/`thenRun`、`thenCompose`、`thenCombine`、`allOf`/`anyOf`
- 异常：`handle`、`exceptionally`、`whenComplete`
- 项目对应：多路检索并行（多通道用 CompletableFuture 并行，再合并结果）

### 7. TTL（TransmittableThreadLocal）跨线程透传
- 问题：`ThreadLocal` 在线程池里，任务结束线程不销毁，下次任务拿不到上一任务的上下文
- TTL：包装 `ThreadLocal`，在提交到线程池时「捕获」当前值，在子线程执行前「回放」，执行后「恢复」
- 项目对应：用户上下文、TraceId 在 8 个专用线程池里透传，保证全链路追踪

---

## 二、第一轮「并发」拷打题

按面试官口吻问，你可以先心里答一遍，再对照下面「答题要点」自检。

---

**1. 线程池**

- 说一下线程池的 corePoolSize、maximumPoolSize、workQueue 在任务提交流程里是怎么配合的？  
  比如 core=2、max=5、队列容量=10，连续提交 20 个任务，前 2、第 3～12、第 13～17、第 18～20 分别会怎样？
- 为什么阿里规范里建议用 `ThreadPoolExecutor` 构造函数而不是 `Executors`？  
  结合 `FixedThreadPool`/`CachedThreadPool` 的队列或线程数说明。

**2. synchronized 与 volatile**

- `synchronized` 和 `volatile` 的区别？什么场景必须用 `synchronized` 而不能只用 `volatile`？
- 简单说一下 synchronized 的锁升级过程（偏向锁、轻量级锁、重量级锁）。

**3. CAS 与 AQS**

- CAS 是什么？有什么缺点？ABA 问题怎么解决？
- AQS 大致原理（state + 队列）。  
  说一个基于 AQS 的类（如 ReentrantLock 或 Semaphore），并说明它怎么用 state 和队列。

**4. 锁**

- 悲观锁和乐观锁的区别？各举一个你在项目或学习中见过的使用场景。
- 你们项目里分布式锁是怎么实现的？  
  （若用 Redis）为什么要用 Lua？不用 Lua 会有什么问题？

**5. CompletableFuture 与 TTL（结合项目）**

- 你们多路检索是怎么做并行的？  
  为什么用 CompletableFuture 而不是直接起多个 Thread 或一个大的线程池？
- 用户上下文（如 userId）和 TraceId 在线程池里是怎么传递的？  
  用普通 ThreadLocal 会有什么问题？TTL 是怎么解决这个问题的？

---

## 三、自检时的答题要点（不背原话，理解后能说出来即可）

| 题                       | 建议答到的点                                                 |
| ------------------------ | ------------------------------------------------------------ |
| 线程池流程               | 先占满 core → 任务入队 → 队满再开线程到 max → 再满走拒绝策略；18 个进队列、2 个在跑，第 13～17 开非核心线程，第 18～20 触发拒绝。 |
| 不用 Executors           | 固定/缓存线程池要么队列无界 OOM，要么线程数无界 OOM 或把系统打满；显式指定参数可控。 |
| synchronized vs volatile | volatile 只保证可见性和有序性，不保证复合操作的原子性；多线程写同一变量必须用锁或 CAS。 |
| 锁升级                   | 无竞争偏向锁 → 轻量级锁（CAS）→ 重量级锁（Monitor）。        |
| CAS 缺点                 | ABA（加版本号）、自旋开销、只能管一个变量。                  |
| AQS                      | state 表示资源/锁状态；抢不到就进 CLH 队列阻塞；释放时唤醒后继。 |
| 分布式锁 + Lua           | 加锁（SET NX EX）和设置过期要原子，否则进程挂掉会死锁；删锁要校验 value，防止删了别人的锁。 |
| CompletableFuture        | 多路检索多通道并行，用 `CompletableFuture.supplyAsync` 投到线程池，`allOf` 等结果再合并；避免手写线程和同步。 |
| TTL                      | 线程池复用导致 ThreadLocal 串值；TTL 在提交时 capture、执行前 replay、执行后 restore，保证每次任务拿到当前请求的上下文。 |

你可以先选 2～3 道（比如：线程池流程、synchronized vs volatile、CompletableFuture+TTL 结合项目）按自己的话答一遍，发给我，我可以按「面试官视角」帮你改成更顺、更专业的说法。

#  JUC 与线程基础

下面按「理论 + 你项目里的对应用法」把这块串起来讲，方便你理解和面试时结合项目说。

---

## 一、线程的创建方式 & 线程状态

### 1. 三种创建方式

| 方式         | 写法                              | 特点                   | 项目里怎么用                                                 |
| ------------ | --------------------------------- | ---------------------- | ------------------------------------------------------------ |
| **Thread**   | 继承 `Thread`，重写 `run()`       | 单继承，不利于扩展     | 一般不直接 new Thread，而是交给线程池                        |
| **Runnable** | 实现 `Runnable`，重写 `run()`     | 无返回值，可多实现     | 线程池 `execute(Runnable)` 用的就是它                        |
| **Callable** | 实现 `Callable<V>`，重写 `call()` | 有返回值、可抛受检异常 | `CompletableFuture.supplyAsync(() -> {...})` 里本质是 Callable 语义 |

项目里：**不手写 `new Thread()`**，而是用**线程池 + Executor**。例如 `ThreadPoolExecutorConfig` 里定义的 8 个线程池（mcp 批处理、RAG 检索、意图识别、记忆摘要、模型流式、chat 入口、知识库分块等），任务都是以 `Runnable`/`Callable` 的形式提交给这些池子，由池子里的线程去跑。所以面试时可以说：我们通过**线程池 + Runnable/Callable** 来用多线程，而不是直接 new Thread。

### 2. 线程状态与转换（Java 的 6 种状态）

- **NEW**：`new Thread()` 后、未 `start()`
- **RUNNABLE**：可运行（包括在 CPU 上跑或在就绪队列里等）
- **BLOCKED**：等 **synchronized** 监视器锁（没抢到锁时进这个状态）
- **WAITING**：`wait()` / `join()` / `LockSupport.park()`，要别人唤醒
- **TIMED_WAITING**：带超时的等，如 `sleep(ms)`、`wait(timeout)`、`LockSupport.parkNanos()`
- **TERMINATED**：`run()` 结束

项目里：像 `ragRetrievalThreadPoolExecutor` 里的工作线程，平时在 `LinkedBlockingQueue` 上 `take()` 时可能进入 **TIMED_WAITING**；拿到任务后进入 **RUNNABLE** 执行；若任务里用了锁，抢不到锁就会 **BLOCKED**。所以「线程状态」在你项目里就体现在**线程池工作线程**的生命周期上。

---

## 二、synchronized：用法 & 锁升级

### 1. 两种用法（实例锁 vs 类锁）

- **实例锁**：锁的是**当前对象**  
  - `synchronized` 实例方法、`synchronized(this)`、`synchronized(某个对象)`
- **类锁**：锁的是 **Class 对象**  
  - `synchronized` 静态方法、`synchronized(Xxx.class)`  
  类锁全局只有一把，所有实例共享。

项目里：Ragent 没有到处用 synchronized，但像**限流、Trace、用户上下文**都是「请求级」的，用 **ThreadLocal/TTL** 做隔离，避免共享变量，所以不需要对业务对象加实例锁。你可以说：我们更多用**线程池隔离 + TTL 透传**来保证线程安全，而不是依赖大量 synchronized。

### 2. 锁升级（偏向 → 轻量级 → 重量级）

- **偏向锁**：认为多数时间只有一个线程用这把锁，在对象头里记「偏向线程 ID」，该线程再来直接进，几乎无 CAS。
- **轻量级锁**：有第二个线程来争，升级为轻量级锁，用 **CAS** 改对象头里的 Lock Record，自旋几次。
- **重量级锁**：自旋超过阈值或等待线程多，升级为 **Monitor**（操作系统 mutex），线程进内核态阻塞（**BLOCKED**）。

面试点：synchronized 不是一上来就重量级，而是根据竞争情况升级，减少在无竞争或低竞争时的开销。

### 3. 锁粗化 & 锁消除

- **锁粗化**：一连串加锁/解锁连在一起，JIT 可能把锁范围扩大成一大块，减少加解锁次数。
- **锁消除**：逃逸分析发现某对象只在当前线程用，不会逃逸，则对该对象的锁会被优化掉。

项目里不一定要举具体类，只要说：我们业务里共享状态不多，主要靠线程池和 TTL；若真有共享变量，会用 synchronized 或 JUC 锁，并知道 JVM 会做锁优化即可。

---

## 三、volatile：可见性、有序性，不保证原子性

### 1. 作用

- **可见性**：一个线程改了这个变量，别的线程能马上看到（通过内存屏障刷新/失效缓存）。
- **禁止指令重排**：保证「写 volatile」之前的操作不会被重排到写之后，「读 volatile」之后的操作不会被重排到读之前（如单例双重检查（DCL）里的 instance 用 volatile）。

  ```
  public class Singleton {
      
      // 1. 必须加 volatile，这是防翻车的最后一道防线！
      private static volatile Singleton instance;
  
      // 2. 构造方法私有化，防止别人在外面 new
      private Singleton() {
      }
  
      // 3. 全局唯一获取实例的方法
      public static Singleton getInstance() {
          // 第一重检查：如果不为空，直接返回，连锁都不用抢，性能拉满！
          if (instance == null) {
              // 抢锁：只在第一次初始化时才需要排队
              synchronized (Singleton.class) {
                  // 第二重检查：拿到锁之后再查一次，防止别人刚才已经帮你建好了！
                  if (instance == null) {
                      instance = new Singleton();
                  }
              }
          }
          return instance;
      }
  }
  ```

  

### 2. 不保证原子性

`count++` 这种「读-改-写」在多线程下不是原子操作，**仅用 volatile 不能保证线程安全**，需要配合 CAS 或锁。  
项目里：没有明显用 volatile 的地方，但你可以说：我们异步场景多，共享状态主要靠 TTL 做线程隔离，若以后有简单的状态标志位，会考虑用 volatile 保证可见性。

---

## 四、ThreadLocal 原理 & 内存泄漏 & 和 TTL 的关系

### 1. ThreadLocal 原理（一句话 + 结构）

- 每个线程有一个 **ThreadLocalMap**（在 `Thread.threadLocals` 里），key 是 **ThreadLocal 对象**，value 是你存的值。
- `get()/set()` 实际是：当前线程 → 自己的 map → 用当前 ThreadLocal 做 key 读写。
- 所以**同一条线程**里，不同请求如果复用这条线程（例如线程池），上一个请求设的 value 没 remove，下一个请求还能拿到，就会**串请求**。

### 2. 内存泄漏风险

- **Key**：是弱引用（WeakReference），key 指向的 ThreadLocal 会被 GC 回收，但 **value 是强引用**。
- 若 ThreadLocal 不再用，但线程一直活着（如线程池线程），则 map 里会留下 `key=null, value=大对象` 的 entry，**value 一直不被回收** → 内存泄漏。
- 正确姿势：用完后 **`threadLocal.remove()`**；或至少保证线程生命周期短（不长期复用时风险小）。

项目里：`UserContext`、`RagTraceContext` 用的是 **TransmittableThreadLocal**，封装里会在合适的时机做 copy/restore，但**请求结束**时仍然要 **clear/remove**（例如在过滤器或拦截器里调 `UserContext.clear()`、`RagTraceContext.clear()`），避免线程复用时把上个请求的 user/trace 带到下个请求。

### 3. TTL 解决的是「线程池场景下的继承」问题

- **问题**：  
  - 请求在 **Tomcat 线程 A** 里，把 `UserContext.set(loginUser)`、`RagTraceContext.setTraceId(...)` 设好了。  
  - 然后 `executor.submit(() -> { ... })` 把任务丢到**线程池里的线程 B**。  
  - 线程 B 是复用的，它的 **ThreadLocal** 里要么是空的，要么是**上一个任务**留下的，所以拿不到当前请求的 user 和 traceId。
- **TTL 做的事**：  
  - **提交任务时**：把当前线程（A）里所有 TTL 的 value **复制一份**（capture）。  
  - **在子线程执行前**：把这份复制出来的值 **设进子线程（B）** 的 TTL（replay）。  
  - **子线程执行完**：把子线程的 TTL **恢复成执行前的状态**（restore），避免污染下一个任务。  
  这样，**线程池里的任务**也能拿到「提交时所在请求」的 user、traceId、nodeStack。

```
【 原生线程池的上下文丢失惨案 】

主线程 (TraceId = 1001)  --提交任务-->  线程池队列
                                       |
                                       v
                                线程池 (核心线程 A)
                                (线程 A 早就创建了，身上还残留着上一次任务的 TraceId = 0099)
                                (业务代码执行时报错：拿不到 1001，链路追踪彻底断裂！)

================================================================================

【 TtlExecutors 的包装与拦截机制 (Capture -> Replay -> Restore) 】

主线程 (TraceId = 1001)
      |
      | 1. Capture (抓取)：在提交任务的瞬间，快照抓取主线程当前的 TTL 上下文。
      v
TtlExecutors 包装层  ---(将真实任务 + TraceId=1001 封装成 TtlRunnable)---> 线程池队列
                                                                          |
                                                                          v
                                                                 线程池 (核心线程 A)
                                                                          |
                                        2. Replay (回放)：在真实业务执行前，把 1001 强行塞进线程 A，
                                                          并把线程 A 原本的脏数据 0099 暂存起来。
                                                                          |
                                        3. 执行真实业务逻辑 (此时代码完美读取到 TraceId = 1001)
                                                                          |
                                        4. Restore (恢复)：业务执行完，把 1001 抽走，把暂存的 0099 
                                                           还给线程 A，打扫战场，深藏功与名。
```

项目里的对应代码：

- **UserContext**（`framework/context/UserContext.java`）：用 `TransmittableThreadLocal<LoginUser>` 存当前用户，在 Tomcat 线程里 set，在 RAG 检索、意图识别、模型流式等**线程池**里 get，都能拿到同一用户。
- **RagTraceContext**（`framework/trace/RagTraceContext.java`）：用三个 TTL 存 `traceId`、`taskId`、`NODE_STACK`，在异步链路里 push/pop 节点，全链路追踪不断。
- **ThreadPoolExecutorConfig**：每个 Bean 最后都 `return TtlExecutors.getTtlExecutor(executor)`，这样**所有通过这些 Executor 执行的任务**都会自动做 TTL 的 capture/replay/restore。

面试时可以这样说：

- 「我们用户上下文和 Trace 是用 **TransmittableThreadLocal** 存的，请求在 Tomcat 线程里 set，但实际业务会丢到 8 个专用线程池里执行，普通 ThreadLocal 在子线程里拿不到父线程的值，所以用了 TTL；并且所有线程池都用 **TtlExecutors.getTtlExecutor** 包装，这样异步任务里也能拿到当前请求的 userId 和 traceId，保证全链路追踪和鉴权。」

---

## 五、和简历/面试的对应关系

| 简历上的点     | 对应项目/知识点                                              |
| -------------- | ------------------------------------------------------------ |
| 线程创建方式   | 用线程池 + Runnable/Callable，不直接 new Thread；8 个线程池各司其职。 |
| 线程状态       | 线程池工作线程在取任务、执行、等锁时的状态转换。             |
| synchronized   | 知道实例锁/类锁、锁升级；项目里少用，主要靠线程隔离 + TTL。  |
| volatile       | 可见性、有序性、不保证原子性；项目里可提「需要时可用来做状态标志位」。 |
| ThreadLocal    | 原理（每线程一个 map，key 是 ThreadLocal）；内存泄漏（弱引用 key + 强引用 value，要 remove）。 |
| TTL 跨线程透传 | UserContext、RagTraceContext 用 TTL；线程池用 TtlExecutors 包装；解决「线程池复用导致子线程拿不到请求上下文」的问题。 |

如果你愿意，我可以再按「面试官问一句、你答一句」的格式，把上面浓缩成 5～8 个 Q&A，你直接背/练就行。

下面是按「面试官问一句、你答一句」整理的 Q&A，可以直接用来练口述。

---

## 一、线程创建 & 线程池

**Q1：Java 里创建线程有哪几种方式？你们项目里怎么用？**

答：三种。一是继承 `Thread` 重写 `run()`；二是实现 `Runnable` 重写 `run()`，无返回值；三是实现 `Callable` 重写 `call()`，有返回值、可抛异常。我们项目里不直接 `new Thread()`，而是用线程池。像 RAG 检索、意图识别、模型流式输出等都配置了独立的线程池，任务以 `Runnable` 或通过 `CompletableFuture` 以类似 `Callable` 的方式提交，由线程池里的工作线程执行，这样能复用线程、控制并发、也方便用 TTL 做上下文透传。

---

**Q2：线程有哪几种状态？怎么转换？**

答：Java 里线程有六种状态：NEW（刚 new 未 start）、RUNNABLE（可运行或正在运行）、BLOCKED（等 synchronized 的监视器锁）、WAITING（无超时等待，如 wait/join）、TIMED_WAITING（有超时等待，如 sleep、带超时的 wait）、TERMINATED（run 结束）。比如我们检索线程池里的工作线程，在队列上取任务时可能处于 TIMED_WAITING，拿到任务后进入 RUNNABLE 执行，如果任务里有锁没抢到就会进入 BLOCKED。

---

## 二、synchronized & volatile

**Q3：synchronized 和 volatile 有什么区别？什么场景必须用 synchronized？**

答：synchronized 是锁，保证同一时刻只有一个线程执行临界区，能保证可见性、有序性和原子性；volatile 只保证可见性和禁止指令重排，不保证复合操作的原子性。所以像 `i++` 这种读-改-写，多线程下光用 volatile 不行，必须用 synchronized 或 CAS。我们项目里请求级状态主要用 ThreadLocal/TTL 做线程隔离，共享变量不多，若以后有共享计数或标志位，会视情况用 volatile 做可见性或 synchronized/ReentrantLock 做互斥。

---

**Q4：synchronized 的锁升级过程简单说一下。**

答：从无竞争到高竞争会经历三个阶段。一开始是偏向锁，在对象头里记录偏向线程 ID，同一线程再次进入几乎无开销。一旦有第二个线程来争，就升级为轻量级锁，用 CAS 改对象头的 Lock Record，并自旋几次。如果自旋超过阈值或等待线程多了，就升级为重量级锁，也就是 Monitor，线程会进入内核态阻塞。这样在低竞争时能减少加锁开销。

---

## 三、ThreadLocal & TTL

**Q5：ThreadLocal 的原理是什么？为什么会内存泄漏？**

答：每个线程里有一个 ThreadLocalMap，存在 Thread 的 threadLocals 里，key 是 ThreadLocal 对象本身，value 是存进去的值。get/set 时就是拿当前线程的 map，用当前 ThreadLocal 当 key 读写，所以不同线程之间互不干扰。内存泄漏主要是因为 map 里 key 是弱引用，ThreadLocal 可能被 GC 掉变成 null，但 value 是强引用，如果线程一直活着（比如线程池里的线程）且没有 remove，这个 value 就一直不会被回收。所以用完后要主动调用 remove，尤其在线程池场景下。

---

**Q6：你们用户上下文和 TraceId 是怎么在线程池里传递的？为什么不用普通 ThreadLocal？**

答：我们用的是 TransmittableThreadLocal，也就是 TTL。请求在 Tomcat 线程里会 set 当前用户和 traceId，但后续 RAG 检索、意图识别、模型流式等会提交到多个专用线程池执行。普通 ThreadLocal 只在当前线程有效，子线程是池子里复用的线程，拿不到父线程设的值，会串请求或 trace 断掉。TTL 在任务提交时会把当前线程的 TTL 值复制一份，在子线程执行前塞进子线程的 TTL，执行完再恢复，这样池里的任务也能拿到当前请求的用户和 traceId。我们所有线程池都用 TtlExecutors.getTtlExecutor 包装，所以透传是统一的。

---

**Q7：TTL 具体是怎么实现「透传」的？**

答：可以简单理解成三步：提交任务时 capture，把当前线程里所有 TTL 的值复制出来；子线程执行任务前 replay，把这些值设进子线程的 TTL；子线程执行完后 restore，把子线程的 TTL 恢复成执行前的状态，避免把当前请求的上下文留给下一个任务。这样既能在异步链路里拿到 user、traceId，又不会污染线程池里后续任务。

---

## 四、综合（结合项目一句话）

**Q8：你们项目里在并发和线程安全这块主要做了哪些设计？**

答：主要有三点：一是按业务拆了多个线程池，比如检索、意图识别、记忆摘要、模型流式等各用各的池，避免互相阻塞、也方便限流和排查；二是用 TransmittableThreadLocal 存用户上下文和 Trace，所有线程池用 TtlExecutors 包装，保证异步任务里也能拿到当前请求的 user 和 traceId，全链路追踪和鉴权不断；三是请求级状态尽量用 TTL 做线程隔离，减少共享变量，需要共享时再考虑锁或 volatile。这样既保证正确性，也方便扩展和观测。

---

建议：先按 Q1→Q8 自己出声答一遍，卡住的地方对照上面的答再练一两遍，面试时就能自然结合 Ragent 讲清楚。

# 线程池

按「知识点讲解 + 项目对应 + 面试 Q&A」整理好了，你直接按顺序看、再练口述即可。

---

## 一、线程池 7 个核心参数（一句话 + 项目里的用法）

| 参数                         | 含义                                                       | 项目里怎么用                                                 |
| ---------------------------- | ---------------------------------------------------------- | ------------------------------------------------------------ |
| **corePoolSize**             | 核心线程数，常驻不回收（除非 allowCoreThreadTimeOut=true） | 如 ragContext 用 2、ragRetrieval 用 CPU_COUNT、memorySummary 用 1，按任务量区分 |
| **maximumPoolSize**          | 最大线程数，包含核心 + 非核心                              | 多为 core 的 2 倍或 4 倍（如 CPU_COUNT << 1、CPU_COUNT << 2） |
| **keepAliveTime**            | 非核心线程空闲多久被回收                                   | 统一 60 秒，和 unit 一起用                                   |
| **unit**                     | 上面时间的单位                                             | TimeUnit.SECONDS                                             |
| **workQueue**                | 任务队列，核心满后先入队                                   | 见下「常见队列」                                             |
| **threadFactory**            | 创建线程的工厂，可设名字、是否守护线程等                   | Hutool ThreadFactoryBuilder，前缀如 `rag_retrieval_executor_`、`model_stream_executor_`，出问题时好排查 |
| **rejectedExecutionHandler** | 队列也满时的拒绝策略                                       | 见下「拒绝策略」                                             |

---

## 二、执行流程（先 core → 入队 → 再开非核心 → 满则拒绝）

1. 线程数 &lt; corePoolSize → **新建核心线程**执行该任务（不先入队）。
2. 线程数 ≥ corePoolSize → 任务**入 workQueue**（不立刻开新线程）。
3. 队列**已满**且线程数 &lt; maximumPoolSize → **新建非核心线程**执行该任务。
4. 队列满且线程数 = maximumPoolSize → 走 **rejectedExecutionHandler**（拒绝或抛异常）。

注意：只有队列是有界且已满时，才会创建非核心线程；若用的是无界队列，永远到不了第 3、4 步，maximumPoolSize 相当于不起作用。

**项目对应**：  
- 用 **SynchronousQueue** 的池（如 mcpBatch、ragRetrieval）：不存任务，有任务且核心都在忙就会立刻开非核心线程，所以适合「不想堆积、希望快速执行」的检索、MCP 等。  
- 用 **LinkedBlockingQueue(100/200)** 的池（如 ragInnerRetrieval、memorySummary、modelStream）：核心满后任务先入队，队列满才开非核心，能缓冲一定波峰。

---

## 三、常见队列（项目里用到的两种）

| 队列                    | 特点                                                         | 项目里用在哪儿                                               |
| ----------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **SynchronousQueue**    | 容量为 0，不存任务，offer 必须有人 take 才能成功，否则直接去创建非核心线程 | mcpBatch、ragContext、ragRetrieval、intentClassify：偏「即时执行」、不堆积 |
| **LinkedBlockingQueue** | 可选有界（如 100、200），链表，FIFO，默认无界时容易 OOM      | ragInnerRetrieval(100)、memorySummary/modelStream/chatEntry/kbChunk(200)：允许排队、缓冲 |
| **ArrayBlockingQueue**  | 有界数组，必须指定容量                                       | 项目未用                                                     |
| **DelayedWorkQueue**    | 延迟队列，用于 ScheduledThreadPoolExecutor                   | 项目未用                                                     |

面试可答：我们检索、意图识别这类希望尽快跑完的用 SynchronousQueue；内部检索、记忆摘要、模型流式、聊天入口这类可以适当排队的用有界 LinkedBlockingQueue，并设 100/200 防止无界导致 OOM。

---

## 四、拒绝策略（项目里两种）

| 策略                    | 行为                                               | 项目里用在哪儿                                               |
| ----------------------- | -------------------------------------------------- | ------------------------------------------------------------ |
| **AbortPolicy**（默认） | 抛 RejectedExecutionException                      | modelStreamExecutor、chatEntryExecutor：流式和入口希望「满了就失败」，由上层做重试或限流 |
| **CallerRunsPolicy**    | 用**调用者线程**执行被拒绝的任务，相当于降级、背压 | 其他 6 个池：检索、MCP、意图、记忆摘要、知识库分块等，让提交任务的线程自己跑，减缓提交速度 |
| DiscardPolicy           | 静默丢弃                                           | 未用                                                         |
| DiscardOldestPolicy     | 丢弃队头再提交                                     | 未用                                                         |

面试可答：我们大部分池用 CallerRunsPolicy，被拒绝时由调用线程执行，形成背压；模型流式和聊天入口用 AbortPolicy，满了就拒绝，配合前面的排队/限流逻辑处理。

---

## 五、为什么用线程池 + 为什么拆 8 个专用线程池

- **用线程池**：复用线程减少创建/销毁开销；用 core、max、队列控制并发和堆积；统一用 TtlExecutors 包装，方便 TTL 透传；线程名统一前缀，便于排查。
- **拆 8 个池**：不同业务隔离，检索打满不会把记忆摘要、模型流式拖死；可按业务设不同 core/max/队列和拒绝策略；出问题时看线程名就知道是哪个环节。

---

## 六、面试 Q&A（可直接练）

**Q1：线程池的 7 个核心参数分别是什么？各自作用？**

答：corePoolSize 核心线程数、maximumPoolSize 最大线程数、keepAliveTime 和 unit 控制非核心线程空闲多久回收、workQueue 任务队列、threadFactory 创建线程的工厂、rejectedExecutionHandler 队列和线程都满时的拒绝策略。我们项目里用 CPU_COUNT 动态算 core/max，队列有用 SynchronousQueue 的也有用有界 LinkedBlockingQueue 的，线程工厂统一设了名称前缀方便排查，拒绝策略大部分用 CallerRunsPolicy，流式和入口用 AbortPolicy。

---

**Q2：任务提交后，线程池的执行顺序是怎样的？**

答：先看当前线程数是否小于 corePoolSize，是就新建核心线程执行；否则尝试把任务放进 workQueue；只有队列满了才会新建非核心线程直到达到 maximumPoolSize；再满就走拒绝策略。所以用无界队列时，最大线程数基本用不上；我们有的池用 SynchronousQueue，核心一忙就会开非核心，有的用有界 LinkedBlockingQueue 先缓冲再开非核心。

---

**Q3：SynchronousQueue 和 LinkedBlockingQueue 在线程池里有什么区别？你们为什么有的池用 SynchronousQueue、有的用 LinkedBlockingQueue？**

答：SynchronousQueue 不存任务，一提交就等着被线程取走，否则会去开非核心线程；LinkedBlockingQueue 可以设容量，任务先入队，队满才开非核心。我们检索、意图识别、MCP 批处理这类希望尽快执行、不想堆积的用 SynchronousQueue；内部检索、记忆摘要、模型流式、聊天入口、知识库分块这类可以排队的用有界 LinkedBlockingQueue(100/200)，既缓冲又避免无界导致 OOM。

---

**Q4：拒绝策略有哪几种？你们项目里怎么选的？**

答：常见四种：AbortPolicy 抛异常、CallerRunsPolicy 用调用者线程执行、DiscardPolicy 静默丢弃、DiscardOldestPolicy 丢队头再提交。我们大部分池用 CallerRunsPolicy，被拒绝时由提交任务的线程自己执行，形成背压；模型流式和聊天入口用 AbortPolicy，满了就拒绝，配合前面限流和排队逻辑处理。

---

**Q5：为什么不用 Executors 的 FixedThreadPool、CachedThreadPool，而要自己 new ThreadPoolExecutor？**

答：FixedThreadPool 用的是无界 LinkedBlockingQueue，任务多时队列会无限增长，有 OOM 风险；CachedThreadPool 最大线程数是 Integer.MAX_VALUE，高并发时可能创建大量线程把系统拖垮。所以我们显式 new ThreadPoolExecutor，指定有界队列或 SynchronousQueue，以及合理的 core、max 和拒绝策略，资源和行为都可控。同时用 ThreadFactoryBuilder 设线程名前缀，出问题时好排查。

---

**Q6：你们为什么拆成多个线程池而不是用一个大的？**

答：不同业务对延迟和堆积的容忍度不一样。检索、意图识别希望尽快跑完，用 SynchronousQueue 和小 core/max；记忆摘要、模型流式可以适当排队，用有界队列。拆开之后，检索打满不会占满记忆摘要或流式输出的线程，互相隔离；而且每个池可以单独调参、单独看监控，线程名也能直接看出是哪个业务，排查方便。再加上所有池都用 TtlExecutors 包装，每个池里的任务都能正确拿到当前请求的上下文和 Trace。

---

# CAS 与 AQS

按「知识点 + 项目里的用法 + 面试 Q&A」把 CAS 和 AQS 串起来讲。

---

## 一、CAS（Compare-And-Swap）

### 1. 是什么、解决什么问题

- **含义**：三个操作数——变量当前值 V、期望值 A、新值 B。只有当前值等于 A 时才把内存更新为 B，否则不更新；整个过程是**一条 CPU 指令**级别的原子操作。
- **作用**：无锁更新共享变量，多线程下做「读-改-写」时不用 synchronized，用 CAS 保证只在一个线程上更新成功，其它线程重试或放弃。
- **实现**：Java 里通过 `Unsafe` 的 native 方法调 CPU 的 CAS 指令；我们平时用 **原子类**（如 `AtomicInteger`、`AtomicBoolean`、`AtomicReference`）的 `compareAndSet` / `getAndSet` 等，底层就是 CAS。

### 2. ABA 问题与解决

- **问题**：线程 1 读到 A，准备改成 B 时，线程 2 把 A→C→A，线程 1 再 CAS 时发现还是 A 就更新成功，但中间状态已经被改过，在「只关心值是否变化」的场景可能有问题（如链表头被换过又换回）。
- **解决**：加版本号或 stamp。Java 里用 **AtomicStampedReference**（版本号）或 **AtomicMarkableReference**（boolean 标记），比较时同时比较「引用 + 版本」，只有都一致才更新。

### 3. 项目里 CAS 用在哪

| 场景               | 类                              | 用法                                                         | 目的                                           |
| ------------------ | ------------------------------- | ------------------------------------------------------------ | ---------------------------------------------- |
| SSE 只关一次       | `SseEmitterSender`              | `closed.compareAndSet(false, true)`                          | 多线程关连接时只真正关一次，幂等               |
| 首包只触发一次     | `FirstPacketAwaiter`            | `eventFired.compareAndSet(false, true)` 再 `latch.countDown()` | 多个回调（内容/完成/错误）里只 countDown 一次  |
| 取消只执行一次     | `StreamCancellationHandles`     | `once.compareAndSet(false, true)` 再 cancel                  | 流式取消幂等，避免重复 cancel                  |
| 释放许可防重复     | `ChatQueueLimiter`              | `permitRef.compareAndSet(permitId, null)` 再 release         | 只有持有该 permitId 的线程能释放，防止重复释放 |
| 通知只由一个线程跑 | `ChatQueueLimiter.PollNotifier` | `firing.compareAndSet(false, true)` 再执行 notify            | 多线程触发时只让一个线程执行一轮通知           |

原子类：`AtomicLong`（HttpMCPClient 的 requestId）、`AtomicBoolean`（closed、eventFired、once、firing）、`AtomicReference`（permitRef、error）、`AtomicInteger`（pendingNotifications）。  
面试时可以说：我们多处用 **CAS 做「只执行一次」或「谁持有谁释放」**，避免重复关闭、重复 countDown、重复释放许可；原子类保证这些状态更新的线程安全。

---

## 二、AQS（AbstractQueuedSynchronizer）

### 1. 是什么、核心结构

- **角色**：JUC 里很多同步组件的基础，用「一个 int state + 一个双向 CLH 队列」实现「抢不到资源就进队、释放时唤醒后继」。
- **state**：由子类定义含义。如 ReentrantLock 表示重入次数，Semaphore 表示剩余许可数，CountDownLatch 表示剩余未 countDown 次数。
- **队列**：双向链表，节点里是等待的线程；独占模式下一个节点代表一个线程，共享模式下可能一次唤醒多个（如 Semaphore 释放多个许可）。

### 2. 独占 vs 共享

- **独占**：同一时刻只有一个线程能拿到资源。如 **ReentrantLock**：tryAcquire 把 state 从 0 改成 1，拿到锁；unlock 时 release 把 state 减回去，唤醒队头。
- **共享**：多个线程可以同时持有。如 **Semaphore**：tryAcquireShared 扣减 state（许可数）；**CountDownLatch**：state 表示剩余次数，countDown 减 1，减到 0 时唤醒所有等待的线程。

### 3. 常见基于 AQS 的类（简要）

| 类                     | state 含义                                    | 典型用法                        |
| ---------------------- | --------------------------------------------- | ------------------------------- |
| ReentrantLock          | 重入次数，0=未占用                            | lock/unlock、tryLock、Condition |
| ReentrantReadWriteLock | 高 16 位读锁、低 16 位写锁                    | 读多写少时的读写锁              |
| Semaphore              | 许可数量                                      | 限流、控制并发数                |
| CountDownLatch         | 剩余次数                                      | 主线程等 N 个子任务完成         |
| CyclicBarrier          | 通过 Generation + count，内部用 ReentrantLock | 多线程一起到达栅栏再继续        |

项目里：没有直接用 ReentrantLock，但用了 **CountDownLatch**（FirstPacketAwaiter 里 `new CountDownLatch(1)`，等首包或错误/完成，只 countDown 一次）；限流用的是 **Redisson 的 RPermitExpirableSemaphore**（分布式信号量），和 JUC 的 Semaphore 思想一致，都是「有限许可」。面试时可说：首包探测用 CountDownLatch + CAS 保证只触发一次；限流用分布式信号量，和 AQS 的 Semaphore 是同一类「许可数」模型。

---

## 三、ReentrantLock 要点（与 synchronized 对比）

- **可重入**：同一线程多次 lock，state 累加，unlock 次数要对等，否则不会真正释放。
- **公平/非公平**：构造函数传 true 为公平锁（先到先得），默认非公平（新来的可能直接抢，吞吐更高）。
- **API**：`lock()` / `unlock()`、`tryLock()` / `tryLock(timeout)`、`newCondition()` 得到 Condition，可 `await` / `signal`，实现多条件等待（类似 wait/notify 但更灵活）。
- **和 synchronized 对比**：  
  - 都是可重入；  
  - ReentrantLock 可非阻塞 tryLock、可公平、可多 Condition、可跨方法加解锁；  
  - synchronized 是 JVM 内置、自动加解锁、锁升级，代码更简单。  
  项目里没有用 ReentrantLock，但可以按「高并发时需要 tryLock 或 Condition 时我们会选 ReentrantLock」来答。

---

## 四、面试 Q&A

**Q1：CAS 是什么？有什么缺点？ABA 怎么解决？**  
答：CAS 是 Compare-And-Swap，用「当前值、期望值、新值」做原子比较并更新，底层是一条 CPU 指令，无锁。缺点有：ABA（值被改过又改回）、自旋可能浪费 CPU、只能管一个变量。ABA 用带版本号的引用解决，如 AtomicStampedReference，比较时同时看版本号。

---

**Q2：你们项目里 CAS 用在哪些地方？**  
答：好几处「只执行一次」或「谁持有谁操作」都用 CAS。比如 SSE 关闭用 `closed.compareAndSet(false, true)` 保证只关一次；首包等待器用 `eventFired.compareAndSet(false, true)` 保证只 countDown 一次；流式取消用 `once.compareAndSet(false, true)` 保证取消只执行一次；排队限流里释放许可时用 `permitRef.compareAndSet(permitId, null)` 保证只有持有该 permit 的线程能释放。这些都是用 AtomicBoolean、AtomicReference 的 compareAndSet 做的。

---

**Q3：AQS 的原理？说一个基于 AQS 的类。**  
答：AQS 里有一个 int state 和一个双向队列。子类定义 state 含义，抢资源时先改 state，改成功就拿到，改失败就封装成节点入队阻塞；释放时改回 state 并唤醒后继。比如 CountDownLatch，state 表示剩余次数，countDown 减 1，减到 0 时唤醒所有 await 的线程；我们项目里首包探测就用 CountDownLatch(1)，等第一个包或错误/完成，配合 CAS 保证只 countDown 一次。

---

**Q4：ReentrantLock 和 synchronized 有什么区别？**  
答：都是可重入。ReentrantLock 是 API 级锁，可以 tryLock、设公平锁、一个锁多个 Condition、跨方法加解锁；synchronized 是关键字，JVM 实现，自动加解锁、有锁升级。需要非阻塞获取或复杂等待条件时用 ReentrantLock，一般简单互斥用 synchronized 就够。我们项目里共享状态不多，没用 ReentrantLock，但高并发需要 tryLock 或精细条件等待时会考虑用它。

---

**Q5：CountDownLatch 和 CyclicBarrier 区别？你们用 CountDownLatch 做什么？**  
答：CountDownLatch 是一次性的，state 减到 0 后不能重置，主线程等 N 个子任务完成；CyclicBarrier 可重复用，多线程一起到栅栏再一起继续。我们首包探测用 CountDownLatch(1)，流式回调里收到首包或错误/完成时 CAS 一次再 countDown，等待线程 await 到就继续，用来做「等第一个事件」的同步。

---

按上面顺序理解一遍再练这几道 Q&A，CAS、AQS 和项目就能一起讲清楚。

# 悲观锁 vs 乐观锁

按「概念 + 项目里的用法 + 面试 Q&A」把悲观锁和乐观锁串起来讲。

---

## 一、悲观锁 vs 乐观锁（概念）

### 1. 悲观锁

- **思路**：默认会冲突，**先加锁再操作**，别人要等。
- **实现**：
  - JVM：`synchronized`、`ReentrantLock`
  - DB：`SELECT ... FOR UPDATE`（行锁）、表锁
  - 分布式：Redis/Redisson 分布式锁（如你们用的 RLock）
- **特点**：互斥强、一致性好；并发度低、容易阻塞、死锁要小心。

### 2. 乐观锁

- **思路**：默认冲突少，**先读再改，用条件校验是否被改过**，冲突就重试或放弃。
- **实现**：
  - **版本号**：表里加 `version`，更新时 `UPDATE ... SET version=version+1 WHERE id=? AND version=?`，影响行数=0 表示被别的事务改过，重试或返回失败。
  - **CAS**：内存里用原子类；DB 里可用「条件更新」类似思路（如 `UPDATE ... WHERE stock>=?`）。
- **典型场景**：库存扣减、高并发更新带 version 的实体、多实例抢同一批任务（只允许一个实例抢到）。
- **特点**：无锁等待、读多写少时并发好；冲突多时重试多，可能影响吞吐。

### 3. 选型（面试可答）

| 维度       | 更偏悲观             | 更偏乐观                          |
| ---------- | -------------------- | --------------------------------- |
| 冲突       | 冲突多、临界区大     | 冲突少、读多写少                  |
| 一致性要求 | 强一致、不能接受重试 | 可接受「更新失败再重试」          |
| 实现       | 先拿锁再干活         | 先改再校验（版本号/CAS/条件更新） |

---

## 二、项目里怎么用的

### 1. 悲观锁：Redisson RLock

- **文档分块**（`KnowledgeDocumentServiceImpl.startChunk`）：同一文档只允许一个分块在执行，用 `RLock lock = redissonClient.getLock("knowledge:chunk:lock:"+docId)`，`tryLock(5, 30, SECONDS)`，先拿到锁再查库、改状态、提交分块任务。
- **防重复提交**（`IdempotentSubmitAspect`）：同一用户同一请求只执行一次，用 RLock + tryLock()，拿不到直接报重复提交。
- **会话摘要**（`MySQLConversationMemorySummaryService`）：同一会话同一用户只一个线程做摘要，用 RLock，tryLock 不到就跳过。

共性：**先抢到锁再执行业务**，属于悲观锁。

### 2. 乐观锁：定时任务调度用 DB 条件更新「抢锁」

- **场景**：`KnowledgeDocumentScheduleJob`，多实例跑定时任务，同一批待执行记录只能被一个实例执行。
- **做法**：
  - 表里有 `lock_until`、`lock_owner`。
  - 扫描条件：`lock_until IS NULL OR lock_until < now()`（未锁或已过期）。
  - **抢锁**：`tryAcquireLock` 里用一条 **UPDATE**：
    - `WHERE id=? AND (lock_until IS NULL OR lock_until < now)`
    - `SET lock_owner=instanceId, lock_until=?`
  - 只有「当前没人占或已过期」的那一行会被更新，所以 **update 影响行数 > 0 的实例才算抢到**，其他实例更新不到，自然跳过。
- **续期 / 释放**：执行过程中用 `renewLock` 更新 `lock_until`；执行完或异常时 `releaseLock` 把 `lock_owner`、`lock_until` 清空。

和「版本号乐观锁」是一类思想：**不先加互斥锁，而是用「条件更新」保证只有一个人能改成功**；这里是「谁先满足 WHERE 条件谁抢到」，属于乐观锁。

---

## 三、面试 Q&A

**Q1：悲观锁和乐观锁的区别？分别适合什么场景？**  
答：悲观锁假定会冲突，先加锁再操作，如 synchronized、ReentrantLock、DB 的 FOR UPDATE、Redis 分布式锁，适合冲突多、临界区大的场景。乐观锁假定冲突少，先读再改，用版本号或条件更新校验，冲突就重试或失败，适合读多写少、冲突少的场景，比如库存扣减、带 version 的实体更新。我们项目里文档分块、防重、会话摘要用 Redisson 分布式锁（悲观）；定时任务多实例抢同一批 schedule 用 DB 的 lock_until 条件更新，只有 update 成功的那台执行（乐观）。

---

**Q2：乐观锁在数据库里一般怎么实现？**  
答：常见两种：一是加 version 字段，更新时 `UPDATE ... SET version=version+1 WHERE id=? AND version=?`，影响行数为 0 说明被别的事务改过；二是用业务字段做条件，比如库存 `UPDATE ... SET stock=stock-1 WHERE id=? AND stock>=1`。我们定时任务调度没用 version，而是用 `lock_until` 和 `lock_owner`：只有 `lock_until 为 null 或已过期` 的行才能被 update 成自己的 lock_owner 和 lock_until，多实例里只有一个能 update 成功，相当于乐观抢锁。

---

**Q3：你们项目里哪些地方用悲观锁、哪些用乐观锁？为什么这么选？**  
答：悲观锁用在：同一文档分块（RLock 按 docId）、防重复提交（RLock 按请求）、同一会话摘要（RLock 按会话+用户），这些都要「先占住再干活」，用分布式锁串行。乐观锁用在：知识库文档定时任务调度，多实例扫描待执行记录，用 DB 的 lock_until 条件更新抢行，谁先 update 成功谁执行，避免用分布式锁把调度逻辑和 Redis 绑太死，也适合「抢任务」这种冲突相对少的场景。

---

**Q4：乐观锁如果冲突很多会怎样？怎么缓解？**  
答：冲突多会导致大量 update 影响行数为 0、业务层要重试，吞吐会下来。缓解方式：缩短持锁时间、把热点拆开（如按用户/单号分片）、或对真正高冲突的写用悲观锁。我们定时任务是按 schedule 行抢的，任务分散在不同行，冲突本来就不高，所以用乐观锁；真正强互斥的像「同一文档分块」就用悲观锁。

---

上面把「悲观/乐观概念 + 选型 + 项目里的 RLock 和 schedule 条件更新」都串好了，你按顺序理解再练这几道 Q&A 即可。

# 分布式锁

按「知识点 + 项目用法 + 面试 Q&A」把分布式锁串起来讲，并带上你项目里的 Redisson 和 ZSET+Lua。

---

## 一、要解决的问题

- **场景**：多进程 / 多机部署，同一时刻只允许一个实例执行某段逻辑（如扣库存、同一文档分块、同一会话做摘要、防重复提交）。
- **本质**：在分布式环境下实现**互斥**，用一把「全局唯一的锁」来串行化。

---

## 二、Redis 实现要点

### 1. 基本写法（单条命令不够原子）

- **加锁**：`SET key value NX PX 过期毫秒`  
  - NX：仅当 key 不存在时设置  
  - PX：毫秒级过期，避免进程挂了导致死锁  
- **问题**：若拆成「先 SET NX 再 PEXPIRE」两条命令，中间宕机会出现「抢到锁但没设过期」→ 死锁。所以**加锁 + 设过期**必须在同一原子操作里（一条命令或 Lua）。

### 2. 用 Lua 保证原子性

- 加锁：在 Lua 里 `SET key value NX PX ttl` 一条完成。  
- 删锁：不能直接 `DEL key`，否则可能删掉**别人**的锁（自己持有的锁已过期，被别的实例抢到，再 DEL 就误删）。  
- **正确做法**：value 存**唯一标识**（如 UUID），删锁时用 Lua：**先 GET 比较 value 是否等于当前实例的标识，相等才 DEL**，这样「判断 + 删除」原子，不会误删。

### 3. Redisson 的 RLock（你项目里用的）

- **加锁**：底层用 Lua：SET key hash(线程/客户端标识) NX PX ttl，保证加锁+过期原子。  
- **可重入**：同一线程多次 lock，用 hash 存重入次数，加 1/减 1。  
- **看门狗（Watchdog）**：未显式传 leaseTime 时，Redisson 不依赖你传的过期时间，而是用**看门狗**：默认 30 秒过期，后台每 10 秒（约 1/3 过期时间）续期一次；只要业务没执行完且进程活着，锁就不会因为「业务执行超过过期时间」被提前释放。  
- **释放**：unlock 时用 Lua：校验 value（或重入计数）是否属于当前线程，是才 DEL 或减 1，防止误删别人锁。

所以：**原子性（加锁+过期、判断+删锁）、value 防误删、可重入、续期** 这几条你都可以结合 Redisson 答。

---

## 三、注意点（面试常问）

| 点                 | 说明                                                         |
| ------------------ | ------------------------------------------------------------ |
| **过期时间**       | 设太短：业务没跑完锁就没了，被别的实例抢走；设太长：宕机后恢复慢。Redisson 用看门狗自动续期缓解。 |
| **可重入**         | 同一线程多次加锁要支持，否则自己把自己锁死。Redisson 的 RLock 支持。 |
| **锁续期**         | 业务可能很长，看门狗或自己定时续期，避免执行中锁过期。       |
| **删锁校验 value** | 必须「只删自己的锁」，用 value/UUID + Lua 判断再 DEL。       |
| **单点/红锁**      | 单 Redis 宕机锁全丢；Redisson 提供 RedLock：多台 Redis 独立加锁，过半成功才算拿到，降低单点影响（有争议，但面试能说清楚即可）。 |

---

## 四、项目里怎么用的

### 1. 分布式锁（Redisson RLock）

| 场景                 | 类                                      | 用法                                                         | 目的                                               |
| -------------------- | --------------------------------------- | ------------------------------------------------------------ | -------------------------------------------------- |
| **防重复提交**       | `IdempotentSubmitAspect`                | `getLock(lockKey)`，`tryLock()` **不等待**，拿不到直接抛异常，finally `unlock()` | 同一用户同一请求（path+userId+args MD5）只执行一次 |
| **同一文档分块互斥** | `KnowledgeDocumentServiceImpl`          | `getLock("knowledge:chunk:lock:"+docId)`，`tryLock(5, 30, TimeUnit.SECONDS)`，锁内起异步分块任务 | 同一文档不会并发分块                               |
| **同一会话摘要互斥** | `MySQLConversationMemorySummaryService` | `getLock(SUMMARY_LOCK_PREFIX+conversationId+userId)`，`tryLock(0, TTL, MILLISECONDS)`，finally 里 `isHeldByCurrentThread()` 再 `unlock()` | 同一会话同一用户只一个线程做摘要                   |

- **Key 设计**：防重是「用户+请求」维度；分块是「文档」维度；摘要是「会话+用户」维度，避免不同资源争同一把锁。  
- **tryLock 区别**：防重不等待（0）；分块等 5 秒、锁 30 秒；摘要不等待、锁用配置 TTL。  
- **安全释放**：摘要里用 `isHeldByCurrentThread()` 再 unlock，避免在异常或异步场景下误释放在别的线程持有的锁（Redisson 可重入时当前线程持有）。

### 2. ZSET + Lua 原子「出队/claim」（和分布式锁同一类考点）

- **场景**：`ChatQueueLimiter` 排队限流，请求入 ZSET 排队，只有排到「队头窗口内」才能去抢信号量并出队。  
- **Lua**：`queue_claim_atomic.lua` 里：`ZRANK` 看排名 → 若 `rank >= maxRank` 直接 return → 否则 `ZSCORE` 再 `ZREM`。**判断排名 + 出队** 在 Redis 里一条脚本执行，避免「判断通过后、ZREM 前」被别的请求插队。  
- **和分布式锁的共性**：都是「多步操作在 Redis 上必须原子」，要么用单条命令（SET NX PX），要么用 **Lua**；你们限流用 Lua 做 ZSET 的原子 claim，和锁用 Lua 做「判断 value 再 DEL」是同一类「原子性」考点。

---

## 五、面试 Q&A

**Q1：Redis 做分布式锁，为什么要用 Lua？不用会有什么问题？**

答：两处需要原子：一是**加锁+设过期**，若先 SET NX 再 PEXPIRE，中间宕机会加锁成功但没过期时间，导致死锁，所以要用一条 SET NX PX 或 Lua；二是**删锁**，若先 GET 再 DEL，中间锁可能已过期被别的实例抢走，再 DEL 就误删别人锁，所以要用 Lua「先判断 value 是否属于当前实例再 DEL」。我们项目里用 Redisson 的 RLock，底层加锁和释放都是 Lua，保证原子；限流里 ZSET 出队也是用 Lua 做 ZRANK+判断+ZREM 原子执行。

---

**Q2：删锁时为什么要校验 value？不校验会怎样？**

答：锁有过期时间，可能自己执行时间超过 ttl，锁已经过期被别的实例抢到了，此时若直接 DEL key，会把别人刚抢到的锁删掉，导致多个实例同时持有「逻辑上的锁」。所以要把 value 设成唯一标识（如 UUID 或 Redisson 的 hash），删锁时用 Lua 只在自己的 value 匹配时才 DEL，避免误删。

---

**Q3：Redisson 的看门狗是干什么的？**

答：当不传 leaseTime 用默认加锁时，Redisson 不会只依赖一次过期时间，而是启动一个看门狗任务，在锁快过期时（默认约 1/3 时间）自动续期，这样只要业务没执行完、进程还在，锁就不会因为执行时间超过初始 ttl 而被释放。我们项目里像知识库文档分块会传 tryLock(5, 30, SECONDS)，是显式指定了 30 秒过期；防重和会话摘要是 tryLock(0, TTL)，也是显式过期，看门狗在未指定 leaseTime 的 lock() 场景下更常用。

---

**Q4：你们项目里分布式锁用在哪些场景？Key 怎么设计？**

答：三个场景：一是防重复提交，用 Redisson getLock，key 是 path+userId+参数 MD5，tryLock 不等待，拿不到就报重复提交；二是同一文档分块，key 是 knowledge:chunk:lock:docId，tryLock 等 5 秒、锁 30 秒，避免同一文档被并发分块；三是同一会话的摘要，key 是会话+用户，tryLock 不等待，拿不到就跳过，保证同一会话同一用户只有一个线程做摘要。释放时摘要那里会先判断 isHeldByCurrentThread 再 unlock，避免误释放。

---

**Q5：你们限流里 ZSET 和 Lua 是怎么用的？和分布式锁的 Lua 有什么共同点？**

答：排队限流时请求按顺序进 ZSET，只有排名在许可数以内的才能去抢信号量并出队。出队时要先 ZRANK 看排名，再判断是否在窗口内，再 ZREM；这三步若分多条命令执行，中间可能被别的请求插队，所以用 Lua 把 ZRANK、判断、ZREM 放在一个脚本里原子执行。和分布式锁的共通点是：多步操作在 Redis 上必须要么单条命令、要么 Lua 保证原子，否则会有并发问题；我们锁用 Redisson 的 Lua，限流用自己写的 queue_claim_atomic.lua，都是这类「原子性」需求。

---

上面把「分布式锁 + Redis/Lua + Redisson + 项目里的 RLock 和 ZSET+Lua」串在一起了，你按顺序过一遍再练这几道 Q&A 即可。

# CompletableFuture

按「知识点 + 项目用法 + 面试 Q&A」把 CompletableFuture 串起来讲。

---

## 一、CompletableFuture 要点

### 1. 创建

| 方法                                         | 含义                 | 返回值                        |
| -------------------------------------------- | -------------------- | ----------------------------- |
| **supplyAsync(Supplier&lt;U&gt;)**           | 异步执行，有返回值   | CompletableFuture&lt;U&gt;    |
| **supplyAsync(Supplier&lt;U&gt;, Executor)** | 指定线程池，有返回值 | CompletableFuture&lt;U&gt;    |
| **runAsync(Runnable)**                       | 异步执行，无返回值   | CompletableFuture&lt;Void&gt; |
| **runAsync(Runnable, Executor)**             | 指定线程池，无返回值 | CompletableFuture&lt;Void&gt; |

不传 Executor 时用 **ForkJoinPool.commonPool()**；项目里一般**显式传线程池**（如 ragRetrievalExecutor、intentClassifyExecutor），便于控制并发、且配合 TtlExecutors 做上下文透传。

### 2. 编排

| 方法             | 含义                                                         |
| ---------------- | ------------------------------------------------------------ |
| **thenApply**    | 上一步有结果，转换成新结果，继续返回 CF                      |
| **thenAccept**   | 消费上一步结果，无返回值                                     |
| **thenRun**      | 上一步完成后执行，不关心结果                                 |
| **thenCompose**  | 上一步结果是 CF，扁平化成一串 CF（类似 flatMap）             |
| **thenCombine**  | 当前 CF 与另一个 CF 都完成后，用 BiFunction 合并两个结果     |
| **allOf(CF...)** | 多个 CF 都完成才完成，无聚合结果，常配合 thenApply + join 取各结果 |
| **anyOf(CF...)** | 多个 CF 中任意一个完成即完成，返回第一个完成的结果           |

### 3. 异常

| 方法                                            | 含义                                         |
| ----------------------------------------------- | -------------------------------------------- |
| **handle(BiFunction&lt;T,Throwable,U&gt;)**     | 无论成功失败都执行，用结果或异常生成新值     |
| **exceptionally(Function&lt;Throwable,T&gt;)**  | 仅失败时执行，返回兜底值                     |
| **whenComplete(BiConsumer&lt;T,Throwable&gt;)** | 无论成功失败都执行，不改变结果，用于打日志等 |

---

## 二、项目里怎么用的

### 1. 多路检索并行（主场景）

**类**：`MultiChannelRetrievalEngine`

- **流程**：先筛出启用的检索通道，再**并行执行**每个通道，最后把结果交给后处理链。
- **实现**：
  - `List<CompletableFuture<SearchChannelResult>> futures = enabledChannels.stream().map(channel -> CompletableFuture.supplyAsync(() -> channel.search(context), ragRetrievalExecutor)).toList();`
  - 每个通道在 **ragRetrievalExecutor** 里跑，带 TTL 透传。
  - 收集结果：`futures.stream().map(CompletableFuture::join).filter(Objects::nonNull).toList()`，再合并 chunk、走去重/重排等后处理。
- **为什么用 CF**：多通道互不依赖，并行能缩短总耗时；用 CF + 指定线程池，比手写 Thread 或只用一个通用线程池更清晰、也方便和 TTL 结合。

### 2. 子问题 / 子意图并行

- **IntentResolver**：多个子问题各自做意图分类。  
  `subQuestions.stream().map(q -> CompletableFuture.supplyAsync(() -> new SubQuestionIntent(q, classifyIntents(q)), intentClassifyExecutor)).toList()`，再 `map(CompletableFuture::join)` 得到 subIntents。
- **RetrievalEngine**：多个子问题各自构建检索上下文。  
  `subIntents.stream().map(si -> CompletableFuture.supplyAsync(() -> buildSubQuestionContext(si, topK), ragContextExecutor)).toList()`，再 `map(CompletableFuture::join)` 得到 contexts，后面再合并 KB/MCP 上下文。

### 3. 多子问题 / 多目标检索并行

- **RetrievalEngine**：多个 MCP 请求并行执行。  
  `requests.stream().map(request -> CompletableFuture.supplyAsync(() -> executeSingleMcpTool(request), mcpBatchExecutor)).toList()`，再 `map(CompletableFuture::join)` 得到 MCP 响应列表。
- **AbstractParallelRetriever**：多个 target（如多 collection）并行检索。  
  每个 target 一个 `CompletableFuture.supplyAsync(() -> createRetrievalTask(question, target, topK), executor)`，再对每个 future join 收集 chunk。

### 4. allOf + thenApply 合并两个异步结果

**类**：`DefaultConversationMemoryService.load`

- **需求**：同时加载「摘要」和「历史消息」，都完成后再合并成对话记忆。
- **实现**：
  - `CompletableFuture<ChatMessage> summaryFuture = CompletableFuture.supplyAsync(() -> loadSummaryWithFallback(...));`
  - `CompletableFuture<List<ChatMessage>> historyFuture = CompletableFuture.supplyAsync(() -> loadHistoryWithFallback(...));`
  - `CompletableFuture.allOf(summaryFuture, historyFuture).thenApply(v -> { ChatMessage summary = summaryFuture.join(); List<ChatMessage> history = historyFuture.join(); return attachSummary(summary, history); }).join();`
- **要点**：用 **allOf** 等两个都完成，在 **thenApply** 里再分别 join 取结果并合并，避免串行等待。

### 5. runAsync 异步执行、不关心返回值

- **MySQLConversationMemorySummaryService**：`CompletableFuture.runAsync(() -> doCompressIfNeeded(conversationId, userId), memorySummaryExecutor)`，摘要压缩丢到专用线程池异步执行。
- **StreamAsyncExecutor**：`CompletableFuture.runAsync(() -> streamTask.accept(cancelled), executor)`，流式任务异步跑。

### 6. 异常处理（项目里的写法）

- **MultiChannelRetrievalEngine**：在 supplyAsync 的 lambda 里 **try-catch**，单个通道失败时返回空的 `SearchChannelResult`，外层 join 时再 catch 转成 null 并 filter，保证一个通道挂了不影响其他通道结果汇总。
- 没有用 **exceptionally** / **handle**，而是「任务内吞异常 + 返回兜底结果」，逻辑简单、也方便打日志。

---

## 三、面试 Q&A

**Q1：CompletableFuture 的 supplyAsync 和 runAsync 区别？为什么要传线程池？**  
答：supplyAsync 有返回值，runAsync 无返回值。不传线程池时用 ForkJoinPool.commonPool()，我们项目里都传线程池：一是和业务绑定的线程池（如 ragRetrievalExecutor、intentClassifyExecutor、mcpBatchExecutor）便于限流和监控；二是这些线程池都用 TtlExecutors 包装，CF 里跑的任务能拿到当前请求的 user 和 traceId，不会丢上下文。

---

**Q2：你们多路检索是怎么做并行的？为什么用 CompletableFuture 而不是直接起多个 Thread？**  
答：多路检索引擎里会先筛出启用的检索通道，然后对每个通道 `CompletableFuture.supplyAsync(() -> channel.search(context), ragRetrievalExecutor)` 投到检索线程池，得到一列 CompletableFuture，再用 stream map(CompletableFuture::join) 等全部完成并收集结果，最后合并 chunk 再走后处理链。用 CF 而不是自己 new Thread：一是复用线程池、控制并发；二是和现有 TTL 线程池一致，上下文能透传；三是写法简单，不需要自己维护 List&lt;Thread&gt; 和 join。

---

**Q3：allOf 和 thenApply 在你们项目里怎么用的？**  
答：加载对话记忆时要同时拉「摘要」和「历史消息」，用两个 supplyAsync 分别跑，再用 `CompletableFuture.allOf(summaryFuture, historyFuture).thenApply(v -> { ... summaryFuture.join(); historyFuture.join(); return attachSummary(...); }).join()`。allOf 表示两个都完成才往下走，thenApply 里再分别 join 取结果并合并，这样摘要和历史是并行的，总耗时接近较慢的那一个，而不是两者相加。

---

**Q4：如果某个通道或子任务抛异常，你们怎么处理？**  
答：多路检索里在每个通道的 supplyAsync lambda 里 try-catch，失败时返回一个空的 SearchChannelResult（0 chunk），外层 join 时再 catch 转成 null 并 filter 掉，这样单通道失败不会导致整次检索失败，其他通道结果照常合并。没有用 CF 的 exceptionally，而是任务内吞异常并返回兜底结果，便于打日志和统计成功/失败通道数。

---

**Q5：thenApply 和 thenCompose 的区别？**  
答：thenApply 是上一步的结果类型 T 映射成 U，得到 CompletableFuture&lt;U&gt;；thenCompose 是上一步的结果本身就是 CompletableFuture&lt;U&gt;，把它「展平」成一层 CF，避免嵌套 CF&lt;CF&lt;U&gt;&gt;。我们项目里多是一组 CF 用 allOf 或 stream + join 收集结果，没有用到 thenCompose；如果以后有「先异步查 A，再根据 A 异步查 B」的链式调用，会用 thenCompose 把两段异步串起来。

---

按上面顺序理解一遍再练这几道 Q&A，CompletableFuture 和你们多路检索、记忆加载、MCP 并行等就能在面试里说清楚。

# TTL

按「问题 → 原理 → 项目用法 → 面试 Q&A」把 TTL 跨线程透传讲清楚。

---

## 一、问题：ThreadLocal 在线程池里会「断档」

- **ThreadLocal**：每个线程一份副本，本线程 `get/set` 只影响当前线程，线程间互不干扰。
- **线程池**：工作线程会**复用**，一个任务跑完线程不销毁，继续跑下一个任务。
- **冲突**：
  - 请求在 **Tomcat 线程 A** 里：`UserContext.set(loginUser)`、`RagTraceContext.setTraceId(...)`，然后 `executor.submit(task)`。
  - 任务在**线程池里的线程 B** 执行，B 是复用的，它的 ThreadLocal 里要么是**空的**，要么是**上一个任务**留下的。
  - 所以在 B 里 `UserContext.get()`、`RagTraceContext.getTraceId()` 拿不到**当前请求**的 user/traceId，会出现鉴权错、Trace 断链、甚至串请求。

也就是说：**父线程（请求线程）设的上下文，子线程（池中线程）拿不到**，这就是「ThreadLocal 在线程池里无法继承」的问题。

---

## 二、TTL 的思路：捕获 → 回放 → 恢复

TTL（TransmittableThreadLocal）在**任务被提交到线程池时**和**在子线程执行前后**插了三个步骤：

| 阶段                | 时机                             | 做什么                                                       |
| ------------------- | -------------------------------- | ------------------------------------------------------------ |
| **Capture（捕获）** | 任务提交时，在**提交者线程**执行 | 把当前线程里所有 TTL 变量的值**复制**出来，和任务一起「带走」 |
| **Replay（回放）**  | 任务在**子线程**里真正执行**前** | 把捕获到的值**设置进**子线程的 TTL，这样子线程 get 就能拿到  |
| **Restore（恢复）** | 任务在子线程执行**后**           | 把子线程的 TTL **恢复成执行前的状态**（通常是清空或还原），避免把本请求的上下文留给**下一个**任务 |

这样：

- 子线程执行期间能拿到「提交时」的 user、traceId 等；
- 执行完后子线程被池回收，不会把当前请求的上下文留给下一个任务，避免串请求、串 Trace。

要生效，需要**执行任务的线程池**被 TTL 包装：用 **TtlExecutors.getTtlExecutor(executor)** 包装后，提交到该 executor 的 Runnable/Callable 会被 TTL 的包装类在 run 前后自动做 replay/restore；capture 在提交时由包装的 submit 完成。

---

## 三、项目里怎么用的

### 1. 用 TTL 存「请求级」上下文

- **UserContext**（`framework/context/UserContext.java`）  
  - `private static final TransmittableThreadLocal<LoginUser> CONTEXT = new TransmittableThreadLocal<>();`  
  - 在请求入口（如拦截器）里 `UserContext.set(loginUser)`，业务里 `UserContext.get()` / `requireUser()` / `getUserId()`。  
  - 若业务在 Tomcat 线程里执行，直接 get 即可；若在**线程池**里执行，就必须配合 TtlExecutors，否则 get 不到或拿到的是别的请求的 user。

- **RagTraceContext**（`framework/trace/RagTraceContext.java`）  
  - 三个 TTL：`TRACE_ID`、`TASK_ID`、`NODE_STACK`（Deque，用于节点进栈出栈）。  
  - 在请求入口设置 traceId/taskId，在 AOP 或检索/生成等节点里 push/pop 节点，实现全链路层级。  
  - 这些逻辑会跑到多个线程池（检索、意图、记忆摘要、模型流式等），只有用 TTL + TtlExecutors，子线程里才能拿到同一个 traceId 和节点栈，Trace 不断。

### 2. 所有业务线程池都用 TtlExecutors 包装

- **ThreadPoolExecutorConfig** 里：mcpBatch、ragContext、ragRetrieval、ragInnerRetrieval、intentClassify、memorySummary、modelStream、chatEntry、knowledgeChunk 等 **8 个**（或更多）Executor，每个都是：
  - `ThreadPoolExecutor executor = new ThreadPoolExecutor(...);`
  - `return TtlExecutors.getTtlExecutor(executor);`
- 这样，所有通过这些 Bean 提交的 `Runnable`/`Callable`（包括 `CompletableFuture.supplyAsync(..., executor)` 里用的 executor）都会走 TTL 的包装逻辑：**执行前 replay、执行后 restore**，提交时由包装类做 **capture**。

### 3. 效果

- **用户上下文**：检索、意图分类、记忆摘要、模型流式、MCP 等只要在业务里调 `UserContext.getUserId()` 等，拿到的都是**当前请求**的用户，不会串号。
- **全链路 Trace**：从入口到检索、重写、意图、生成、MCP，只要在同一个请求链路上且任务在这些线程池里执行，traceId 和 NODE_STACK 都能透传，便于排查和监控。

---

## 四、面试 Q&A

**Q1：ThreadLocal 在线程池里会有什么问题？TTL 是怎么解决的？**  
答：ThreadLocal 是线程隔离的，而线程池会复用线程，任务结束后线程不销毁，下一个任务再跑时拿到的是上一个任务留下的值，会串请求或拿不到当前请求的上下文。TTL 在任务提交时把当前线程的 TTL 值**捕获**，在子线程执行**前**把值**回放**到子线程，执行**后**再**恢复**子线程原来的 TTL 状态，这样子线程执行期间能拿到提交时的上下文，执行完又不会污染下一个任务。我们项目里用户上下文和 Trace 都用 TTL 存，所有业务线程池用 TtlExecutors 包装，保证透传。

---

**Q2：TTL 的 capture、replay、restore 分别发生在什么时候？**  
答：Capture 在任务**提交到线程池时**，在提交者线程里把当前所有 TTL 的值复制出来跟任务一起传下去。Replay 在**子线程执行任务前**，把复制出来的值设进子线程的 TTL。Restore 在**子线程执行完任务后**，把子线程的 TTL 恢复成执行前的状态，避免把本请求的上下文留给下一个任务。这样既能在异步链路里拿到 user、traceId，又不会串请求。

---

**Q3：你们用户上下文和 TraceId 在线程池里是怎么传递的？**  
答：用 TransmittableThreadLocal 存：UserContext 存 LoginUser，RagTraceContext 存 traceId、taskId 和节点栈。请求在 Tomcat 线程里 set，但检索、意图、摘要、模型流式等会提交到多个专用线程池执行。普通 ThreadLocal 在子线程拿不到父线程设的值，所以用 TTL；并且所有线程池都用 TtlExecutors.getTtlExecutor 包装，提交到这些池的任务会自动做 capture/replay/restore，所以池里的任务能拿到当前请求的 userId 和 traceId，全链路追踪和鉴权不断。

---

**Q4：如果不用 TTL，只用 ThreadLocal，会怎样？**  
答：在 Tomcat 线程里 set 的 user 和 traceId，到了线程池里的线程 get 会是 null 或是别的请求的值。结果是：异步环节鉴权可能失败、Trace 断链、日志里 traceId 对不上，甚至把 A 用户的信息算到 B 用户头上。所以只要请求会经过线程池，就需要 TTL（或类似机制）做跨线程透传，我们项目里 8 个专用线程池都包装了 TtlExecutors，保证透传一致。

---

**Q5：为什么要用 8 个专用线程池并都包装 TtlExecutors？**  
答：不同业务对延迟和堆积的容忍度不同，拆成多个池可以隔离：检索打满不会把记忆摘要或流式输出拖死，且每个池可以单独调参、看监控。这些池都会执行「当前请求」的后续步骤，都需要拿到同一个 user 和 traceId，所以每个池都用 TtlExecutors 包装，这样无论任务在哪个池执行，都能拿到当前请求的 TTL 上下文，保证全链路追踪和用户隔离。

---

小结：**问题**是 ThreadLocal 在线程池复用下无法继承；**TTL** 通过提交时 capture、执行前 replay、执行后 restore，在池中线程里「临时」拥有提交时的上下文；**项目里**用 TTL 存 UserContext 和 RagTraceContext，8 个业务线程池统一用 TtlExecutors 包装，实现用户和 Trace 的跨线程透传。按上面几道 Q&A 练几遍，面试就能说清楚。