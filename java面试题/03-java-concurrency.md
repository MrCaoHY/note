# 03 - Java 并发编程 (JUC) 面试深度指南

> 本章覆盖：进程/线程/协程与虚拟线程、线程状态与生命周期、JMM 与 Happens-Before、`volatile` 底层与缓存行、`synchronized` 与锁升级、CAS 与 `LongAdder`、AQS 源码体系、`ReentrantLock` / 读写锁 / `StampedLock`、并发工具类、线程池核心原理与动态调优、`CompletableFuture` 与 `ForkJoinPool`、JDK 21 虚拟线程与 `ScopedValue`。
> 共计 **41 道高频深度面试题**，每题均包含：**核心结论 + 底层原理 + 代码/图解 + 面试官连环追问 + 实战避坑**。

---

## 目录

- [1. 线程基础与生命周期 (Q1 - Q7)](#1-线程基础与生命周期-q1---q7)
- [2. Java 内存模型 (JMM) 与 volatile (Q8 - Q12)](#2-java-内存模型-jmm-与-volatile-q8---q12)
- [3. synchronized 与锁底层原理 (Q13 - Q17)](#3-synchronized-与锁底层原理-q13---q17)
- [4. CAS、原子类与 Unsafe (Q18 - Q21)](#4-cas原子类与-unsafe-q18---q21)
- [5. AQS 与 JUC 显式锁体系 (Q22 - Q28)](#5-aqs-与-juc-显式锁体系-q22---q28)
- [6. JUC 并发工具类与协同模型 (Q29 - Q32)](#6-juc-并发工具类与协同模型-q29---q32)
- [7. 线程池、异步编排与 ForkJoinPool (Q33 - Q38)](#7-线程池异步编排与-forkjoinpool-q33---q38)
- [8. JDK 21 虚拟线程与现代并发趋势 (Q39 - Q41)](#8-jdk-21-虚拟线程与现代并发趋势-q39---q41)

---

## 1. 线程基础与生命周期 (Q1 - Q7)

### Q1：进程、线程、协程与 JDK 21 虚拟线程的区别是什么？为什么要慎用“过多线程”？
- **核心结论**：**进程**是操作系统资源分配的最小单位，**线程**是 CPU 调度的最小单位，**协程/虚拟线程**是用户态实现的轻量级调度单元（无需内核态切换）。并非线程越多越快，过多线程会引发剧烈的**上下文切换开销**与**内存爆炸**。
- **底层对比**：
  | 维度 | 进程 (Process) | 平台线程 (Platform Thread) | 协程 / JDK 21 虚拟线程 (Virtual Thread) |
  | :--- | :--- | :--- | :--- |
  | **调度主体** | 操作系统内核 | 操作系统内核（1:1 映射内核线程） | 用户态 / JVM 调度器（M:N 映射到载体线程） |
  | **资源隔离** | 独立虚拟地址空间、文件句柄 | 共享进程堆/方法区，独立栈/PC | 共享堆，栈帧按需在堆上动态伸缩 |
  | **单实例内存** | 几十 MB ~ GB 级 | 默认栈大小约 **1 MB**（`-Xss`） | 初始仅 **几百字节** |
  | **切换开销** | 极高（切换页表、刷新 TLB/Cache） | 较高（用户态 ↔ 内核态切换、保存寄存器） | 极低（仅在用户态切换栈指针与少量寄存器） |
  | **单机上限** | 数百 ~ 数千 | 数千（受内存与内核调度限制） | **百万级** |
- **为什么线程过多反而变慢**：
  1. **上下文切换（Context Switch）**：CPU 时间片轮转时需保存当前线程寄存器/程序计数器并加载新线程状态，频繁切换导致 CPU 缓存（L1/L2 Cache）和 TLB 大量失效。
  2. **内存占用**：2000 个平台线程光栈空间就需要约 2GB 堆外本地内存，极易触发 `OutOfMemoryError: unable to create new native thread`。
- **面试官连环追问**：
  - *问：怎么在 Linux 上查看上下文切换次数？*  
    答：使用 `vmstat 1` 查看 `cs`（Context Switch）列，或 `pidstat -w -p <PID> 1` 区分自愿切换 `cswch/s`（等待 IO/锁）和非自愿切换 `nvcswch/s`（时间片耗尽被抢占，说明线程过多或 CPU 过载）。
- **实战避坑**：不要为每个请求 `new Thread()`，平台线程必须由线程池统一托管；高并发 IO 密集型场景可升级至 JDK 21 虚拟线程。

---

### Q2：Java 线程的 6 种状态及完整转换流转是怎样的？与操作系统线程状态有何映射？
- **核心结论**：Java 在 `Thread.State` 枚举中定义了 **6 种状态**：`NEW`、`RUNNABLE`、`BLOCKED`、`WAITING`、`TIMED_WAITING`、`TERMINATED`。Java 将操作系统的“就绪”和“运行中”合并为 `RUNNABLE`。
- **状态流转全景图**：
  ```text
                                     等待进入 synchronized 同步块/方法
                     ┌───────────────────────────────────────────┐
                     ▼                                           │
  [ NEW ] ──start()──▶ [ RUNNABLE ] ◀──── notify() / notifyAll() / park唤醒 ─── [ WAITING ]
                         │   ▲                                                   ▲
            OS 调度(Ready ⇄ Running)                                     wait() / join() / park()
                         │   │                                                   │
                         │   └─────── 超时自动唤醒 / notify() ────── [ TIMED_WAITING ]
                         │                                           ▲
                         │         sleep(t) / wait(t) / join(t)      │
                         └───────────────────────────────────────────┘
                         │
              run() 正常结束或抛出未捕获异常
                         ▼
                   [ TERMINATED ]
  ```
- **关键映射与细节**：
  1. **`RUNNABLE`**：包含 OS 层的 `Ready`（等待 CPU 时间片）和 `Running`（正在 CPU 上执行）。注意：**传统 BIO 阻塞（如 `socket.read()` 等待网络数据）在 Java 层面状态依然是 `RUNNABLE`**，因为 JVM 认为线程仍在等待操作系统返回，而非阻塞在 JVM 锁上。
  2. **`BLOCKED`**：**专指**线程等待获取 `synchronized` 监视器锁（Monitor）的状态。注意：JUC 的 `ReentrantLock.lock()` 底层调用 `LockSupport.park()`，等待锁时线程状态是 **`WAITING` 或 `TIMED_WAITING`**，而不是 `BLOCKED`！
- **面试官连环追问**：
  - *问：线上 `jstack` 看到大量 `WAITING (parking)` 是什么情况？*  
    答：通常是线程空闲在线程池队列取任务（`BlockingQueue.take()`），或等待 `ReentrantLock` / `CompletableFuture.get()`。大量 `BLOCKED (on object monitor)` 才是 `synchronized` 锁竞争激烈。
- **实战避坑**：排查锁竞争时，不能只搜索 `BLOCKED`，还要结合堆栈检查处于 `WAITING` 状态的 `ReentrantLock` 争用。

---

### Q3：`start()` 和 `run()` 的区别是什么？为什么不能对同一个线程调用两次 `start()`？
- **核心结论**：`start()` 负责向操作系统申请并启动新线程，异步执行 `run()`；直接调用 `run()` 只是一个普通的同步方法调用，仍在当前线程执行。重复调用 `start()` 会抛出 `IllegalThreadStateException`。
- **底层原理（`Thread.start()` 源码解析）**：
  1. 检查线程状态：`if (threadStatus != 0) throw new IllegalThreadStateException();`（`0` 对应 `NEW` 状态）。
  2. 将线程加入 `ThreadGroup`。
  3. 调用本地方法 **`start0()`**（C++ 实现 `JVM_StartThread`），在底层创建 OS 线程（Linux 下调用 `pthread_create`），并将该新线程置于就绪状态，由 OS 调度器回调 Java 层的 `run()` 方法。
  4. 调用 `start()` 后 `threadStatus` 已变为非 0（如 `RUNNABLE` 或结束后的 `TERMINATED`），因此二次调用必抛 `IllegalThreadStateException`。
- **面试官连环追问**：
  - *问：如果一个线程执行完了（`TERMINATED`），能再次 `start()` 复用吗？*  
    答：不能。线程生命周期不可逆，要复用线程资源必须使用线程池（线程池内部通过让 `Worker` 线程在一个 `while` 循环中不断从 `workQueue` 拉取 `Runnable` 并直接调用 `task.run()` 来复用线程）。
- **实战避坑**：实现多线程任务优先实现 `Runnable` 或 `Callable` 接口（解耦任务与线程载体），不要盲目继承 `Thread` 类。

---

### Q4：`sleep()`、`wait()`、`yield()`、`join()` 的核心区别是什么？为什么 `wait/notify` 必须在 `synchronized` 中调用且定义在 `Object` 上？
- **核心结论**：`sleep` 持锁休眠、`wait` 释锁等待、`yield` 礼让 CPU、`join` 等待目标线程结束。`wait/notify` 定义在 `Object` 上是因为锁绑定在对象头 Monitor 上，必须在 `synchronized` 中调用是为了防止“信号丢失（Lost Wakeup）”并确保已持有 Monitor。
- **四者核心对比表**：
  | 方法 | 所属类 | 是否释放锁 | 进入状态 | 唤醒条件 |
  | :--- | :--- | :--- | :--- | :--- |
  | `Thread.sleep(ms)` | `Thread` (静态) | **不释放锁** | `TIMED_WAITING` | 超时或被 `interrupt()` |
  | `obj.wait()` / `wait(ms)` | `Object` (实例) | **释放当前对象锁** | `WAITING` / `TIMED_WAITING` | `notify/notifyAll`、超时、中断 |
  | `Thread.yield()` | `Thread` (静态) | **不释放锁** | 保持 `RUNNABLE`（提示让出 CPU） | 仅暗示调度器，可能立刻再次被选中 |
  | `t.join()` / `join(ms)` | `Thread` (实例) | 释放 `t` 对象锁（不释放其它锁） | `WAITING` / `TIMED_WAITING` | `t` 线程终止、超时、中断 |
- **底层深度剖析**：
  1. **为什么 `wait/notify` 在 `Object` 中**：Java 中任意对象都可以作为 `synchronized` 的锁（每个对象在底层都可以关联一个 `ObjectMonitor`，包含 `_WaitSet` 等待队列），因此唤醒/等待对象锁的方法必须属于 `Object` 而非 `Thread`。
  2. **为什么必须在 `synchronized` 中调用（Lost Wakeup 问题）**：假设先判断条件 `while (!ready)` 再调用 `wait()`。如果不加锁，判断为 false 后、进入 `wait()` 前，另一线程修改 `ready = true` 并执行了 `notify()`，此时唤醒信号就彻底丢失，当前线程将永久沉睡。加锁使“条件判断 + 入队等待”成为原子操作，且未持有锁时调用会直接抛出 `IllegalMonitorStateException`。
  3. **`join()` 底层源码**：本质就是 `synchronized` 修饰的方法内部循环判断 `while (isAlive()) { wait(0); }`。当目标线程 `t` 在 JVM 层面终止时，C++ 底层会调用 `lock.notify_all(thread)` 唤醒所有 `join()` 等待者。
- **面试官连环追问**：
  - *问：为什么等待条件判断要用 `while` 而不是 `if`？*  
    答：防止**虚假唤醒（Spurious Wakeup）**以及唤醒后条件被其他线程再次改变。线程从 `wait()` 返回后必须重新检查业务条件是否满足。
- **实战避坑**：调用 `t.join()` 前确保 `t.start()` 已执行；不要在持有全局锁的情况下调用 `Thread.sleep()`，否则会阻塞所有等待该锁的线程。

---

### Q5：Java 线程如何优雅停止？`interrupt()`、`isInterrupted()`、`interrupted()` 的区别是什么？
- **核心结论**：Java 采用**协作式中断机制**而非暴力强杀（`stop()` 已废弃）。调用 `t.interrupt()` 只是将线程的中断标志位设为 `true`（若处于阻塞态则抛出 `InterruptedException` 并清除标志位），由目标线程自行决定何时安全退出。
- **三个方法对比**：
  | 方法 | 作用 | 是否静态方法 | 是否清除中断标志位 |
  | :--- | :--- | :--- | :--- |
  | `t.interrupt()` | 将目标线程的中断标志位设为 `true` | 实例方法 | 否（仅在抛 `InterruptedException` 时由 JVM 清除） |
  | `t.isInterrupted()` | 查询目标线程当前是否处于中断状态 | 实例方法 | **不清除** |
  | `Thread.interrupted()` | 查询**当前正在执行**线程的中断状态 | **静态方法** | **会清除**（复位为 `false`） |
- **与各类阻塞方法的交互**：
  - **`sleep` / `wait` / `join`**：响应中断，立即唤醒并抛出 `InterruptedException`，同时**清除**中断标志位（设为 `false`）。
  - **`synchronized` 阻塞与传统 BIO**：**不响应**中断（`synchronized` 会一直等锁，BIO 需关闭底层 `Socket` 才会抛出异常）。
  - **`LockSupport.park()`**：**响应**中断立即返回，但**不抛异常**，且**不清除**中断标志位（保持为 `true`）。
- **优雅停止线程标准模板**：
  ```java
  Thread worker = new Thread(() -> {
      while (!Thread.currentThread().isInterrupted()) {
          try {
              // 执行业务逻辑或可中断阻塞操作
              Thread.sleep(1000);
          } catch (InterruptedException e) {
              // 关键：捕获异常后标志位已被清除，必须重新恢复中断状态或直接 break 退出！
              Thread.currentThread().interrupt();
              break;
          }
      }
      // 释放资源收尾工作
  });
  ```
- **面试官连环追问**：
  - *问：为什么 `Thread.stop()` 被废弃？*  
    答：`stop()` 会强制终止线程并**立刻释放该线程持有的所有对象锁**，极易导致共享数据处于修改到一半的残缺状态（破坏数据一致性）。
- **实战避坑**：捕获 `InterruptedException` 绝对不能“生吞”（空 catch 块），要么向上抛出，要么在 catch 块中调用 `Thread.currentThread().interrupt()` 恢复中断信号，让上层调用栈感知到中断请求。

---

### Q6：什么是守护线程（Daemon Thread）？JVM 何时退出？守护线程中 `finally` 块一定执行吗？
- **核心结论**：守护线程是为其他线程提供后台服务的线程（如 GC 线程、JIT 编译线程、Finalizer 线程）。**当 JVM 中所有非守护线程（用户线程）都结束时，JVM 就会直接退出，不会等待守护线程执行完毕**。因此守护线程中的 `finally` 块**不一定执行**。
- **核心要点**：
  1. `t.setDaemon(true)` 必须在 `t.start()` **之前**调用，否则抛出 `IllegalThreadStateException`。
  2. 在守护线程中创建的新线程，默认依然是守护线程（继承父线程的 `daemon` 属性）。
  3. 当最后一个用户线程退出，JVM 会直接中止所有正在运行的守护线程，**既不会执行它们的 `finally` 代码块，也不会优雅释放 IO/连接资源**。
- **面试官连环追问**：
  - *问：为什么 `main` 方法结束后，如果还有非守护子线程在跑，JVM 不会退出？*  
    答：因为 `main` 线程本身只是众多用户线程之一。JVM 退出的唯一判断条件是“存活的非守护线程数 == 0”（可通过 `DestroyJavaVM` 线程监控）。注意，自定义线程池如果不调用 `shutdown()`，其核心用户线程会一直存活导致进程无法退出。
- **实战避坑**：严禁将涉及文件写入、数据库连接关闭、事务提交等关键业务逻辑放在守护线程中执行；后台异步监控、心跳检测或缓存清理（配合安全退出钩子）才适合使用守护线程。

---

### Q7：如何实现多线程按顺序执行（如 T1 -> T2 -> T3 顺序执行，或交替打印 A/B/C）？
- **核心结论**：按顺序执行本质是线程间的**同步编排**，单次顺序执行可用 `join()`、`CountDownLatch`、`CompletableFuture`；循环交替打印可用 `synchronized + wait/notifyAll` 或 `ReentrantLock + 多 Condition` 精准唤醒。
- **代码示例（`ReentrantLock` + 多 `Condition` 精准唤醒循环打印 ABC）**：
  ```java
  public class OrderedPrint {
      private int flag = 1; // 1->A, 2->B, 3->C
      private final ReentrantLock lock = new ReentrantLock();
      private final Condition condA = lock.newCondition();
      private final Condition condB = lock.newCondition();
      private final Condition condC = lock.newCondition();

      public void print(int targetFlag, int nextFlag, String msg, Condition cur, Condition next) {
          for (int i = 0; i < 10; i++) {
              lock.lock();
              try {
                  while (flag != targetFlag) cur.await();
                  System.out.print(msg);
                  flag = nextFlag;
                  next.signal(); // 精准唤醒下一个线程
              } catch (InterruptedException e) {
                  Thread.currentThread().interrupt();
              } finally {
                  lock.unlock();
              }
          }
      }
  }
  ```
- **五种常见顺序控制方案选型**：
  1. **`t1.join()`**：最简单，适合主线程串行等待少数子线程。
  2. **`CountDownLatch`**：适合一个或多个线程等待前序步骤全部完成。
  3. **`Executors.newSingleThreadExecutor()`**：单线程线程池天然保证 FIFO 顺序执行。
  4. **`CompletableFuture.runAsync(t1).thenRun(t2).thenRun(t3)`**：优雅的异步编排链。
  5. **`LockSupport.park/unpark`**：无需加锁即可指定唤醒具体线程对象。
- **实战避坑**：循环交替唤醒场景中，如果用 `synchronized` 配合 `notify()` 极易因为随机唤醒错误线程导致“假死”（所有线程都在 wait），必须用 `notifyAll()` 或 `ReentrantLock` 的独立 `Condition` 对象。

---

## 2. Java 内存模型 (JMM) 与 volatile (Q8 - Q12)

### Q8：什么是 JMM（Java 内存模型）？主内存、工作内存与 8 种内存交互操作是什么？
- **核心结论**：JMM（Java Memory Model）是一套**抽象的内存规范**（并非物理内存划分），用于屏蔽不同 CPU 架构（x86/ARM）和操作系统在内存访问、缓存一致性上的差异，定义了线程与主内存之间的交互规则，保证并发的**原子性、可见性、有序性**。
- **JMM 内存抽象结构**：
  ```text
  [线程 A] ---> [工作内存 A (共享变量副本)] ──┐
                                              ├──(save/load/use/assign)──▶ [主内存 (Main Memory)]
  [线程 B] ---> [工作内存 B (共享变量副本)] ──┘
  ```
  *(注：物理上主内存对应 RAM，工作内存对应 CPU 寄存器、L1/L2/L3 高速缓存与写缓冲区 Store Buffer)*
- **8 种原子内存交互操作**（JSR-133 规范前经典模型，助于理解底层逻辑）：
  - 作用于主内存：`lock`（锁定）、`unlock`（解锁）、`read`（读取到工作内存）、`write`（从工作内存写入主内存）。
  - 作用于工作内存：`load`（放入副本）、`use`（传给执行引擎）、`assign`（执行引擎赋值回工作内存）、`store`（传送到主内存准备写入）。
  - **严格约束**：不允许单独出现 `read`/`load` 或 `store`/`write`；一个变量同一时刻只允许一个线程对其 `lock`；对变量 `unlock` 前必须先 `store` + `write` 同步回主内存。
- **三大特性保证手段**：
  - **原子性**：`synchronized`、`Lock`、`Atomic*` / CAS。
  - **可见性**：`volatile`、`synchronized`、`Lock`、`final`。
  - **有序性**：`volatile`、`synchronized`、`Happens-Before` 原则。
- **面试官连环追问**：
  - *问：JMM 中的工作内存、主内存和 JVM 运行时数据区的堆、栈是一回事吗？*  
    答：**不是一个维度的概念**。堆/栈是 JVM 物理/逻辑内存划分；JMM 是并发访问语义模型。粗略对应上：主内存主要对应堆中对象实例数据，工作内存主要对应虚拟机栈部分区域及硬件 CPU 缓存/寄存器。

---

### Q9：什么是指令重排序？`as-if-serial` 语义与 `Happens-Before` 8 大规则是什么？
- **核心结论**：为了提升性能，编译器和处理器会在不影响单线程结果的前提下对指令重排序。`as-if-serial` 保障**单线程**看起来是顺序执行的；`Happens-Before` 则是 JMM 用来向程序员承诺**跨线程可见性与有序性**的核心法则。
- **三级重排序**：
  1. **编译器优化重排**：在不改变单线程语义下调整语句顺序。
  2. **指令级并行重排**：现代 CPU（超标量流水线）将无数据依赖的机器指令重叠/乱序执行。
  3. **内存系统重排**：CPU 引入 Store Buffer（写缓冲区）和 Invalidate Queue（无效队列），导致“读/写”在外部核心看来像发生了乱序。
- **`Happens-Before` 核心规则（如果 A Happens-Before B，则 A 的结果对 B 一定可见，且 A 逻辑上排在 B 之前）**：
  1. **程序次序规则**：同一线程中，前面的操作 HB 后面的操作（控制流顺序）。
  2. **监视器锁规则**：对一个锁的解锁（`unlock`）HB 随后对这个锁的加锁（`lock`）。
  3. **`volatile` 变量规则**：对一个 `volatile` 变量的写 HB 随后对这个变量的读。
  4. **传递性规则**：A HB B，且 B HB C，则 A HB C（非常关键！比如线程1写普通变量 `x=1` 再写 `volatile flag=true`，线程2读 `flag==true` 后一定能看到 `x==1`）。
  5. **线程启动规则**：`Thread.start()` HB 此线程的每一个动作。
  6. **线程终止规则**：线程中所有操作 HB 其他线程检测到该线程终止（如 `join()` 返回）。
  7. **线程中断规则**：对线程 `interrupt()` 的调用 HB 被中断线程检测到中断事件。
  8. **对象终结规则**：对象构造函数执行结束 HB 它的 `finalize()` 方法开始。
- **实战避坑**：多线程共享状态传递时，不一定非要把所有字段都加 `volatile`。利用**传递性规则**，只需给状态标志位加 `volatile`，就能保证写标志位之前的所有普通变量修改对读线程可见。

---

### Q10：`volatile` 关键字的作用和底层实现原理是什么？内存屏障、LOCK 前缀与 MESI 如何配合？
- **核心结论**：`volatile` 保证**内存可见性**与**禁止指令重排序**。JVM 通过在读写前后插入**内存屏障（Memory Barrier）**来禁止编译重排与约束 CPU 乱序，在 x86 硬件层面主要体现为对写操作生成带 **`LOCK` 前缀的指令**，结合 **MESI 缓存一致性协议**强制刷新 Store Buffer 并使其他核心缓存行失效。
- **JVM 内存屏障插入策略（保守策略）**：
  - **写操作**：在每个 `volatile` 写前面插入 **`StoreStore` 屏障**（禁止上面普通写与它重排）；在后面插入 **`StoreLoad` 屏障**（禁止它与下面 `volatile` 读/写重排）。
  - **读操作**：在每个 `volatile` 读后面插入 **`LoadLoad` 屏障**（禁止下面普通读与它重排）和 **`LoadStore` 屏障**（禁止下面普通写与它重排）。
- **硬件底层原理解密（以 x86 为例）**：
  x86 是强内存模型（TSO），不会发生 LoadLoad、LoadStore、StoreStore 乱序，唯一可能发生的是 **StoreLoad 乱序**（因为 CPU 有写缓冲区 Store Buffer，写操作异步进缓冲区，后面的读先执行了）。
  当对 `volatile` 变量执行写操作时，JIT 编译器会在汇编指令前加上 **`lock addl $0x0,(%rsp)`**（`LOCK` 前缀空操作，触发完整 `StoreLoad` 屏障效果）：
  1. 强制将当前核心 **Store Buffer** 中的所有脏数据立即刷入主内存（或共享 L3 Cache）。
  2. 通过总线嗅探机制与 **MESI 协议**，向其他 CPU 核心发送 Cache Line 失效消息，并等待它们确认失效（置为 `Invalid` 状态）后才继续执行。
  3. 其他 CPU 核心再次读取该变量时发现缓存行已失效，只能从主内存/共享缓存重新加载最新值。
- **面试官连环追问**：
  - *问：什么是 MESI 协议的 4 种状态？*  
    答：**M（Modified，已修改）**、**E（Exclusive，独占）**、**S（Shared，共享）**、**I（Invalid，已失效）**。
  - *问：既然 CPU 有 MESI 缓存一致性协议，为什么还需要 `volatile`？*  
    答：因为 MESI 本身存在异步优化——**Store Buffer** 和 **Invalidate Queue**，导致硬件层依然存在短暂的可见性延迟和 StoreLoad 乱序；此外编译器优化也可能把变量缓存在 CPU 寄存器中根本不去读内存。`volatile` 强制禁止寄存器缓存并刷空缓冲区。

---

### Q11：为什么 `volatile` 不能保证原子性？如何手写双重检查锁（DCL）单例模式？去掉 `volatile` 会怎样？
- **核心结论**：`volatile` 只能保证单次读或单次写的可见性与有序性，无法保证复合操作（如 `count++` 的“读-改-写”三步）的原子性。DCL 单例必须加 `volatile`，以防止 `new Singleton()` 指令重排导致其他线程拿到**未初始化完全的半成品对象**。
- **标准 DCL 单例代码**：
  ```java
  public class Singleton {
      // 必须加 volatile 禁止指令重排！
      private static volatile Singleton instance;

      private Singleton() {}

      public static Singleton getInstance() {
          if (instance == null) {                     // 第一次检查：避免已创建后频繁加锁
              synchronized (Singleton.class) {
                  if (instance == null) {             // 第二次检查：防止多线程同时通过第一次检查
                      instance = new Singleton();     // 关键点：3步操作可能重排
                  }
              }
          }
          return instance;
      }
  }
  ```
- **去掉 `volatile` 的致命原因**：
  `instance = new Singleton()` 在字节码与 CPU 层面分为 3 步：
  1. 分配对象的堆内存空间。
  2. 初始化对象（调用 `<init>` 构造方法赋值成员变量）。
  3. 将 `instance` 引用指向分配的内存地址（此时 `instance != null`）。
  若没有 `volatile`，第 2 步和第 3 步可能被重排序（变成 `1 -> 3 -> 2`）。当线程 A 执行完 `1 -> 3` 但尚未执行 `2` 时，线程 B 进入第一次检查 `if (instance == null)` 发现不为 null，直接返回这个**字段全是默认值（0/null）的半成品对象**，导致空指针或业务逻辑错误！
- **面试官连环追问**：
  - *问：除了 DCL，还有哪种线程安全且优雅的单例实现？*  
    答：**静态内部类**（利用 JVM 类加载 `<clinit>()` 阶段天然的线程安全与懒加载特性）或 **枚举单例**（《Effective Java》推荐，天然防反射攻击与反序列化破坏）。

---

### Q12：什么是伪共享（False Sharing）？如何解决？（`@Contended` 与缓存行填充）
- **核心结论**：CPU 缓存以**缓存行（Cache Line，通常为 64 字节）**为单位从内存加载数据。当两个毫无关系的 `volatile` 变量恰好被分配在**同一个缓存行**中，被不同 CPU 核心频繁修改时，会互相导致对方的整个缓存行失效，性能退化如单核，这就是**伪共享**。
- **原理示意图**：
  ```text
  [ Core 1 ] ──修改 x──▶ [ 缓存行 (64 Bytes): | volatile long x (8B) | volatile long y (8B) | ... ] ◀──修改 y── [ Core 2 ]
                             └─ Core1 写 x 导致整个缓存行失效，Core2 读 y 发生 Cache Miss！
  ```
- **解决方案**：
  1. **缓存行填充（Padding）**：在变量前后填充无用的 `long` 字段（7 个 `long` 占 56B + 自身 8B = 64B），确保独占一个缓存行（如 Disruptor 的 `RingBufferPad`）。
  2. **`@Contended` 注解（JDK 8+）**：JVM 会在字段前后自动填充 128 字节避免伪共享（考虑到某些 CPU 相邻缓存行预取机制）。**经典源码应用**：`LongAdder` 中的 `Cell` 类、`Thread` 中的 `threadLocalRandomSeed`、`ConcurrentHashMap` 的 `CounterCell`。
- **实战避坑**：用户自定义类使用 `@sun.misc.Contended`（JDK 9+ 在 `jdk.internal.vm.annotation.Contended`）默认被 JVM 忽略，必须加上启动参数 **`-XX:-RestrictContended`** 才会生效！

---

## 3. synchronized 与锁底层原理 (Q13 - Q17)

### Q13：`synchronized` 修饰代码块、实例方法、静态方法的底层字节码与锁对象有什么区别？
- **核心结论**：三者锁住的对象不同（`this`/指定对象 vs `Class` 对象）；底层实现上，代码块通过 **`monitorenter` / `monitorexit`** 字节码指令实现，而同步方法通过方法表上的 **`ACC_SYNCHRONIZED`** 访问标志实现。
- **对比详情**：
  1. **同步代码块**：锁住括号内指定对象（通常是 `this` 或自定义 `private final Object lock = new Object()`）。编译后生成成对的 `monitorenter` 与两个 `monitorexit`（第二个在异常表兜底块中，确保抛出异常也能释放锁）。
  2. **普通同步方法**：锁住当前实例对象 `this`。方法常量池标识 **`ACC_SYNCHRONIZED`**，JVM 执行方法时自动隐式获取/释放锁。
  3. **静态同步方法**：锁住当前类的 **`Class` 对象**（全局唯一，与实例锁互不干扰）。方法标识 **`ACC_SYNCHRONIZED | ACC_STATIC`**。
- **面试官连环追问**：
  - *问：如果一个线程调用类的 `static synchronized` 方法，另一个线程能同时调用同一个对象的普通 `synchronized` 方法吗？*  
    答：**可以**。前者锁的是 `Xxx.class` 对象，后者锁的是实例对象 `this`，两把锁完全独立，互不互斥。
- **实战避坑**：绝对不要用字符串常量（如 `"LOCK"`，在字符串常量池全局共享）或装箱整数（如 `Integer` 在 `-128~127` 有缓存池）作为 `synchronized` 的锁对象，极易引发不同业务间的意外死锁！

---

### Q14：什么是 `ObjectMonitor`？`_owner`、`_EntryList`、`_WaitSet`、`_cxq` 如何协同工作？
- **核心结论**：`synchronized` 重量级锁底层依赖 HotSpot C++ 编写的 **`ObjectMonitor`** 管程对象（对应操作系统互斥量 `mutex`）。它通过维护三个队列和持有者指针来管理线程竞争、阻塞与等待唤醒。
- **核心结构与流转图**：
  ```text
  竞争线程 ──(CAS抢锁失败)──▶ [ _cxq (竞争队列, LIFO栈) ] ──(释放锁时迁移)──▶ [ _EntryList (阻塞队列) ]
                                                                                     │
                                                                          出队竞争 _owner
                                                                                     ▼
                            [ _WaitSet (等待集合) ] ◀──调用 wait()── [ _owner (当前持锁线程) ]
                                    │                                       ▲
                                    └──────调用 notify() / notifyAll()──────┘
  ```
- **关键字段解析**：
  - `_owner`：指向当前持有 `ObjectMonitor` 的线程；为 `null` 时表示无线程持有。
  - `_recursions`：锁的重入次数（每重入一次 +1，退出一次 -1，减到 0 才真正释放）。
  - `_cxq`（ContentionList）：所有竞争锁失败的线程先通过 CAS 压入这个单向链表栈。
  - `_EntryList`：真正有资格参与下一轮锁争夺的阻塞线程队列（默认当 `_EntryList` 为空时，释放锁的线程会把 `_cxq` 里的节点移到 `_EntryList`）。
  - `_WaitSet`：调用 `wait()` 主动释放锁进入等待状态的线程集合，被 `notify()` 唤醒后通常重新放入 `_EntryList`（或 `_cxq`）再次排队抢锁，而非直接获得锁。

---

### Q15：64 位 JVM 下对象头 Mark Word 的结构是怎样的？锁状态标志位如何表示？
- **核心结论**：64 位 JVM（开启指针压缩）下，对象头占 **12 字节**（8B Mark Word + 4B Klass Pointer）。**Mark Word（64 bit）** 是一个动态复用的位域结构，通过最后 **2 bit（锁标志位）+ 1 bit（偏向锁标志位）** 来区分 5 种状态。
- **Mark Word 位图结构（64 位 JVM，最后 3 位为核心标志）**：
  | 锁状态 | 前 61 位存储内容 | 偏向位 (1 bit) | 锁标志位 (2 bit) | 组合后缀 |
  | :--- | :--- | :--- | :--- | :--- |
  | **无锁** | unused(25) + **HashCode(31)** + unused(1) + **分代年龄(4)** | `0` | `01` | **`001`** |
  | **偏向锁** | **ThreadID(54)** + **Epoch(2)** + unused(1) + **分代年龄(4)** | `1` | `01` | **`101`** |
  | **轻量级锁** | **指向线程栈中 Lock Record 的指针 (62 bit)** | — | `00` | **`00`** |
  | **重量级锁** | **指向堆中/C++ ObjectMonitor 的指针 (62 bit)** | — | `10` | **`10`** |
  | **GC 标记** | 空（用于 GC 标记清除/整理阶段） | — | `11` | **`11`** |
- **面试官连环追问**：
  - *问：如果一个对象计算过 `hashCode()`，它还能进入偏向锁状态吗？*  
    答：**不能**！从上表可见，无锁时 `HashCode(31bit)` 存在 Mark Word 中，而偏向锁需要把这 54 bit 用来存 `ThreadID`，没有空间再存一致性哈希码。因此：
    1. 如果对象在无锁态调用过 `System.identityHashCode()` / `Object.hashCode()`，它会**直接跳过偏向锁**，竞争时直接升级为轻量级锁。
    2. 如果对象已经处于**偏向锁**状态，此时突然调用 `hashCode()`，偏向锁会立刻**撤销并膨胀为重量级锁**（把原始 Mark Word 连同 HashCode 备份进 `ObjectMonitor` 内部）！

---

### Q16：详细描述 JDK 6 引入的 `synchronized` 锁升级全过程？为什么只能升级不能降级？JDK 15+ 为什么废弃偏向锁？
- **核心结论**：JDK 6 优化后的锁升级路径为：**无锁 (`001`) → 偏向锁 (`101`) → 轻量级锁 (`00`) → 重量级锁 (`10`)**。锁通常只升不降（严格来说 HotSpot 在 STW 安全点下存在极少数的 Monitor 降级回收机制，但面试标准口径指运行时并发路径单向不可逆）。JDK 15+ 默认禁用偏向锁（JEP 374），因为现代多核并发应用竞争普遍，维护偏向锁撤销的 STW 开销得不偿失。
- **完整升级过程**：
  1. **无锁 → 偏向锁**：首个线程进入同步块，通过一次 CAS 将自身 `ThreadID` 写入对象头 Mark Word，把标志位改为 `101`。后续该线程再次进入时，只需比对 `ThreadID` 是否为自己，**零 CAS 开销**。
  2. **偏向锁 → 轻量级锁（存在轻微交替竞争）**：当另一个线程 B 尝试获取锁，发现偏向的不是自己，触发**偏向锁撤销**：
     - 暂停原持有线程（等待到达安全点 **Safepoint**）。
     - 检查线程 A 是否还活着或仍在同步块内：若已退出同步块，重置为无锁再由线程 B 竞争；若仍在执行，升级为**轻量级锁**。
     - 线程在各自栈帧创建 **Lock Record（锁记录）**，将对象头原 Mark Word 拷贝进锁记录（Displaced Mark Word），再用 CAS 尝试把对象头替换为指向自己锁记录的指针。
  3. **轻量级锁 → 重量级锁（高并发竞争）**：线程 B CAS 自旋抢锁失败（达到自适应自旋阈值，或有第三个线程同时来抢），锁膨胀为**重量级锁**：向 OS 申请 `ObjectMonitor`，Mark Word 改为指向 Monitor 指针（`10`），未抢到锁的线程进入 `_EntryList` 挂起（陷入内核态阻塞）。
- **为什么 JDK 15+ 废弃偏向锁（JEP 374）**：
  - 偏向锁当年是为了优化 `Vector`/`Hashtable` 等单线程重复加锁的老代码而设计；现代代码已广泛使用无锁集合（`ArrayList`/`HashMap`）和并发容器（`ConcurrentHashMap`）。
  - 偏向锁撤销需要等待全球安全点（STW），且与各种现代锁优化耦合极其复杂，维护成本远超收益。**JDK 15 起 `-XX:-UseBiasedLocking` 默认关闭，JDK 18+ 彻底移除相关代码**。

---

### Q17：什么是锁消除、锁粗化与自适应自旋？
- **核心结论**：三者均是 HotSpot JIT 即时编译期或运行期对 `synchronized` 做的无感知性能优化，旨在减少无谓的加锁解锁和线程挂起开销。
- **核心原理**：
  1. **锁消除（Lock Elimination）**：JIT 编译器通过**逃逸分析**（Escape Analysis），发现同步块锁住的对象仅在当前线程内部使用、根本不会逃逸出方法（如方法内局部创建的 `StringBuffer.append()`），直接把底层加锁/解锁指令彻底消除。
  2. **锁粗化（Lock Coarsening）**：如果探测到一连串操作在循环内反复对**同一个对象**加锁解锁（例如 `for (int i=0; i<1000; i++) { synchronized(lock){...} }`），JIT 会把锁的范围扩大（粗化）到整个循环外部，只加锁一次、解锁一次。
  3. **自适应自旋（Adaptive Spinning）**：轻量级锁竞争失败后，线程不立刻挂起，而是空跑忙等一会（自旋，赌持有锁的线程很快释放）。JDK 6 改为“自适应”：如果上次在同一个锁上自旋刚成功过，下次就多自旋几次；如果很少成功，就干脆直接省略自旋过程进入挂起，避免浪费 CPU。

---

## 4. CAS、原子类与 Unsafe (Q18 - Q21)

### Q18：什么是 CAS？`Unsafe` 类在 CAS 中起什么作用？底层 CPU 指令如何保证原子性？
- **核心结论**：CAS（Compare-And-Swap，比较并交换）是一种**无锁乐观并发算法**，包含 3 个操作数：内存地址 `V`、预期旧值 `A`、新值 `B`。Java 通过 `Unsafe`（JDK 9+ 逐步由 `VarHandle` 替代）调用 C++ 内联汇编 **`LOCK CMPXCHG`** 指令，借助硬件总线锁或缓存行锁保证“读取-比较-写入”不可分割。
- **底层调用链**：
  `AtomicInteger.incrementAndGet()` → `Unsafe.getAndAddInt()`（内部是 `do { v = getIntVolatile(); } while (!compareAndSwapInt(o, offset, v, v+delta));` 自旋循环）→ HotSpot C++ `Atomic::cmpxchg` → x86 汇编 **`lock cmpxchg`**。
- **为什么 `CMPXCHG` 前必须加 `LOCK` 前缀**：
  单条 `cmpxchg` 在多核 CPU 上并不是原子的（可能同时有两个核读取了内存并写回）。加上 `LOCK` 前缀后，CPU 会采取**缓存锁（Cache Locking）**（如果数据跨缓存行或不支持缓存锁则退化为总线锁）：锁住对应缓存行直到指令执行完成，并利用 MESI 协议阻止其他核心同时修改该内存区域，同时具备完整的内存屏障效果。

---

### Q19：CAS 有哪三大经典问题？什么是 ABA 问题，如何用 `AtomicStampedReference` 解决？
- **核心结论**：CAS 的三大缺陷是：**ABA 问题**、**高并发自旋开销大**、**只能保证单个共享变量的原子操作**。ABA 问题可通过绑定**版本号/时间戳**（`AtomicStampedReference`）或**布尔标记**（`AtomicMarkableReference`）解决。
- **ABA 问题详解**：
  线程 1 读取变量值为 `A` 后被挂起；期间线程 2 将变量改为 `B`，随后又改回 `A`；线程 1 恢复执行 CAS，比对内存值依然是 `A`，误以为从未被修改过而执行成功。在链表头节点指针操作或库存/余额流水中可能引发严重逻辑错乱（比如栈弹出又压入导致链表节点关系错乱）。
- **`AtomicStampedReference` 解决方案**：
  ```java
  // 初始值 100，初始版本号 1
  AtomicStampedReference<Integer> ref = new AtomicStampedReference<>(100, 1);

  int stamp = ref.getStamp();
  Integer oldVal = ref.getReference();
  // 只有当当前引用 == 100 且 当前版本号 == stamp 时才修改为 101 并让版本号 +1
  boolean success = ref.compareAndSet(oldVal, 101, stamp, stamp + 1);
  ```
- **其他两个问题的解法**：
  - **自旋开销大**：改用分段累加 `LongAdder` / `DoubleAdder`，或改用悲观锁让线程阻塞而非空耗 CPU。
  - **多变量原子性**：将多个字段封装到一个不可变对象中，配合 `AtomicReference` 整体替换引用。

---

### Q20：`LongAdder` 为什么在高并发下比 `AtomicLong` 快得多？底层源码核心原理是什么？
- **核心结论**：`AtomicLong` 在高并发下所有线程都对**同一个 `value`** 进行 CAS 自旋，冲突率极高导致大量 CPU 空转；`LongAdder` 采用**空间换时间 + 分段累加（热点分散）**思想，内部维护一个基础值 `base` 和一个 `@Contended` 修饰的 `Cell[]` 槽数组，各线程分别 CAS 修改不同槽位，最后 `sum()` 时求和。
- **架构对比图**：
  ```text
  AtomicLong:  T1, T2, T3, T4 ──(全部 CAS 争抢同一变量)──▶ [ volatile long value ] (严重自旋失败)

  LongAdder:   T1 ──▶ [ base ]
               T2 ──▶ [ Cell[0] (@Contended 避免伪共享) ]
               T3 ──▶ [ Cell[1] (@Contended 避免伪共享) ]  ──▶ sum() = base + ∑Cell[i]
               T4 ──▶ [ Cell[2] (@Contended 避免伪共享) ]
  ```
- **核心源码流程（`Striped64` 父类）**：
  1. **无竞争阶段**：直接 CAS 更新 `base`，成功即返回（与 `AtomicLong` 性能相当）。
  2. **发生竞争阶段**：`base` CAS 失败后，根据线程的探针哈希（`getProbe()`）映射到 `Cell[hash & (len - 1)]`：
     - 如果 `cells` 未初始化，加自旋锁 `cellsBusy` 初始化长度为 2 的 `Cell[]`。
     - 如果对应的 `Cell[i]` 存在，对其 `value` 执行 CAS 累加。
     - 如果对 `Cell[i]` 再次 CAS 失败，先通过 `advanceProbe()` 重新随机哈希换一个槽位重试；若还碰撞且数组长度小于 CPU 核数，触发 **2 倍扩容**。
- **面试官连环追问**：
  - *问：`LongAdder.sum()` 返回的结果一定是绝对精确的实时快照吗？*  
    答：**不是强一致快照**！`sum()` 遍历 `Cell[]` 累加时**没有加全局锁**，累加过程中可能有其他线程正在修改前面已扫过或未扫过的 `Cell`。因此 `LongAdder` 适合**高并发写、低频读且允许短暂最终一致的统计场景**（如 QPS 计数、监控埋点）；若要强一致原子生成递增序列号（如订单 ID），必须用 `AtomicLong`。

---

### Q21：JUC 提供了哪些常用原子类？`AtomicIntegerFieldUpdater` 和 `VarHandle` 有什么用？
- **核心结论**：JUC `atomic` 包按作用对象可分为 **5 类**：基本类型、数组类型、引用类型、字段更新器、高性能累加器（JDK 8+）。
- **5 大分类清单**：
  1. **基本类型**：`AtomicBoolean`、`AtomicInteger`、`AtomicLong`。
  2. **数组类型**：`AtomicIntegerArray`、`AtomicLongArray`、`AtomicReferenceArray`（针对数组内某下标元素做原子更新）。
  3. **引用类型**：`AtomicReference`、`AtomicStampedReference`（带版本号）、`AtomicMarkableReference`（带 boolean 标记）。
  4. **对象属性修改器**：`AtomicIntegerFieldUpdater`、`AtomicLongFieldUpdater`、`AtomicReferenceFieldUpdater`。
     - **核心价值**：当海量对象中某个属性需要偶尔原子修改时（如 `ConcurrentLinkedQueue`、`Netty` 中的引用计数 `refCnt`），使用全局唯一的静态 `Updater` 直接修改普通 `volatile` 字段，**避免每个对象实例额外创建一个 `AtomicInteger` 对象带来的内存与 GC 压力**！（要求目标字段必须为 `volatile` 且非 `static`、非 `final`）。
  5. **高性能累加器**：`LongAdder`、`DoubleAdder`、`LongAccumulator`、`DoubleAccumulator`。
- **JDK 9+ `VarHandle`（变量句柄）**：
  因为 `sun.misc.Unsafe` 过于危险计划被限制，JDK 9 引入了官方安全的 `MethodHandles.lookup().findVarHandle(...)`。它能以接近 `Unsafe` 的性能对字段执行 `getAcquire`、`setRelease`、`compareAndSet` 等细粒度内存序操作，并支持强类型安全检查（`Atomic*` 和 AQS 在 JDK 9+ 内部已大量改为基于 `VarHandle` 实现）。

---

## 5. AQS 与 JUC 显式锁体系 (Q22 - Q28)

### Q22：什么是 AQS（`AbstractQueuedSynchronizer`）？它的核心架构与设计模式是什么？
- **核心结论**：AQS 是 JUC 绝大多数锁与同步器（`ReentrantLock`、`ReentrantReadWriteLock`、`Semaphore`、`CountDownLatch`、线程池 `Worker`）的底层核心骨架。它采用**模板方法模式**，内部由 **`volatile int state`**（同步状态）和 **CLH 变体双向队列**构成。
- **核心架构图**：
  ```text
  [ volatile int state ] (0=空闲, >=1=占用或剩余许可数)
  [ Thread exclusiveOwnerThread ] (独占模式当前持锁线程)
  
  CLH 双向等待队列 (FIFO):
  [ head (虚节点/当前持锁哨兵) ] ⇄ [ Node(T1, waitStatus=SIGNAL) ] ⇄ [ Node(T2, waitStatus=0) ] ◀── tail
  ```
- **Node 节点关键状态（`waitStatus`）**：
  - `0`：初始默认状态。
  - `SIGNAL (-1)`：表示**后继节点**正处于或即将处于阻塞状态，当前节点释放锁时必须 `unpark` 唤醒后继节点。
  - `CANCELLED (1)`：当前节点因超时或中断被取消，将被移出队列。
  - `CONDITION (-2)`：当前节点在 `Condition` 条件队列中等待。
  - `PROPAGATE (-3)`：共享模式下，向后传播唤醒信号。
- **模板方法模式**：AQS 实现了繁琐的线程排队、阻塞（`LockSupport.park`）、唤醒、取消等公共机制；子类只需重写 `tryAcquire` / `tryRelease`（独占）或 `tryAcquireShared` / `tryReleaseShared`（共享）来定义如何修改 `state` 即可。

---

### Q23：AQS 独占模式获取与释放锁的源码流程是怎样的？为什么释放锁时要从 `tail` 向前遍历找唤醒节点？
- **核心结论**：获取锁流程为 `acquire(arg)` → `tryAcquire`（失败则入队）→ `addWaiter`（CAS 尾插）→ `acquireQueued`（自旋尝试抢锁 + 将前驱置为 `SIGNAL` + `park` 挂起）。释放时调用 `unparkSuccessor`，**从 `tail` 向前遍历寻找最靠前的有效节点**，是因为入队 `addWaiter` 时 `prev` 指针先于 `next` 指针建立。
- **入队与挂起核心流程（`acquire`）**：
  1. `tryAcquire(arg)`：子类实现，尝试修改 `state` 抢锁，成功直接返回。
  2. `addWaiter(Node.EXCLUSIVE)`：构造 `Node`，通过 `CAS` 将其设为 `tail`；若队列未初始化或 CAS 失败，进 `enq()` 死循环 CAS 初始化一个空的哑节点（Dummy Head）再尾插。
  3. `acquireQueued(node, arg)`：在死循环中，如果自己的前驱节点刚好是 `head`，再试一次 `tryAcquire`（可能前驱刚释放）；若还失败，调用 `shouldParkAfterFailedAcquire` 将前驱的 `waitStatus` 改为 `SIGNAL (-1)`（同时顺便剔除已 `CANCELLED` 的前驱），然后调用 `LockSupport.park(this)` 阻塞自己。
- **为什么释放锁时（`unparkSuccessor`）从后往前找**：
  观察 `addWaiter` 尾插代码顺序：
  ```java
  node.prev = pred;                 // 1. 先连接新节点的 prev 指针
  if (compareAndSetTail(pred, node)) { // 2. CAS 把新节点设为 tail
      pred.next = node;             // 3. 最后才连接旧尾节点的 next 指针！
  }
  ```
  在第 2 步完成、第 3 步尚未执行的并发瞬间，如果另一个线程正在释放锁并从 `head` 往后通过 `next` 遍历，会看到 `pred.next == null` 而漏掉新入队的节点！而此时 `node.prev` 早已建立完毕，从 `tail` 通过 `prev` 指针向前遍历则**绝不会漏掉任何刚入队的节点**。

---

### Q24：AQS 共享模式与独占模式有何区别？`Condition` 条件队列如何与同步队列协同？
- **核心结论**：独占模式一次只唤醒 1 个后继节点；共享模式（如 `Semaphore`、`CountDownLatch`）支持多线程同时持有资源，当线程成功获取共享资源且 `state` 仍有富余时，会触发 **`setHeadAndPropagate` 向后链式传播唤醒**。`Condition` 内部维护一条**单向条件队列**，调用 `await()` 释放锁入条件队列，调用 `signal()` 将节点转移回 AQS 同步队列。
- **`Condition` 双队列流转图**：
  ```text
  Condition 条件队列 (单向链表, nextWaiter):
    firstWaiter ──▶ [ Node(T1, CONDITION) ] ──▶ [ Node(T2) ] ◀── lastWaiter
                           │
                     调用 signal() 转移
                           ▼
  AQS 同步队列 (双向链表, prev/next):
    [ head ] ⇄ [ Node(T0) ] ⇄ [ Node(T1, waitStatus=0) ] ◀── tail (准备重新抢锁)
  ```
- **关键机制**：
  - 每个 `ReentrantLock` 可以 `newCondition()` 多个实例（比如阻塞队列经典的 `notFull` 和 `notEmpty` 两个条件队列），实现**分组精准唤醒**，而 `synchronized` 只有一个 `_WaitSet`。
  - 调用 `await()` 前必须先持有锁，进入条件队列时调用 `fullyRelease(node)` 一次性释放所有重入次数。被 `signal()` 转移到同步队列后，必须**重新抢到锁**才会从 `await()` 返回。

---

### Q25：`ReentrantLock` 如何实现可重入、公平锁与非公平锁？与 `synchronized` 全方位对比？
- **核心结论**：`ReentrantLock` 通过记录 `exclusiveOwnerThread` 与递增 `state` 实现可重入；公平锁与非公平锁的唯一区别在于：**非公平锁在 `lock()` 入口和 `tryAcquire()` 时，不管同步队列里有没有人在排队，直接先尝试 CAS 抢一次锁**；公平锁则会先调用 `hasQueuedPredecessors()` 检查队列是否已有等待者。
- **为什么默认是非公平锁**：
  假设线程 A 刚释放锁并准备唤醒队头线程 B（线程唤醒存在内核调度延迟），此时新来的线程 C 刚好处于运行态直接 CAS 抢锁成功并快速执行完释放，随后 B 才被唤醒拿到锁。非公平锁**大大减少了线程挂起与唤醒的上下文切换开销，吞吐量显著更高**（代价是极端情况下队列线程可能饥饿）。
- **`synchronized` vs `ReentrantLock` 深度对比**：
  | 对比维度 | `synchronized` | `ReentrantLock` |
  | :--- | :--- | :--- |
  | **实现层面** | JVM 关键字（字节码 + C++ `ObjectMonitor`） | Java API 类（AQS + CAS + `LockSupport`） |
  | **锁释放** | 自动释放（即使异常也保证释放） | 必须在 `finally` 中手动 `unlock()` |
  | **锁公平性** | 仅支持非公平锁 | 支持非公平（默认）和公平锁 |
  | **可中断 / 超时** | 不支持（死等） | 支持 `lockInterruptibly()` 与 `tryLock(timeout, unit)` |
  | **多条件精确唤醒** | 不支持（单 `_WaitSet`） | 支持绑定多个 `Condition` |
  | **性能差异** | JDK 6 优化后与 `ReentrantLock` 在中低竞争下基本持平 | 高并发复杂场景或配合虚拟线程（避免 Pinning）更有优势 |
- **实战避坑**：使用 `ReentrantLock` 时，**`lock()` 必须写在 `try` 块外面**！如果写在 `try` 里面，万一加锁阶段抛出异常（或被中断）而并未成功持锁，执行 `finally` 里的 `unlock()` 会抛出 `IllegalMonitorStateException` 并覆盖原始异常。

---

### Q26：`ReentrantReadWriteLock` 读写锁的底层原理是什么？什么是锁降级？为什么不支持锁升级？
- **核心结论**：`ReentrantReadWriteLock` 遵循“**读读共享、读写互斥、写写互斥**”，巧妙地将 AQS 的 32 位 `int state` 拆为两半：**高 16 位记录读锁持有总次数，低 16 位记录写锁重入次数**。**锁降级**是指当前线程持有写锁时再获取读锁，随后释放写锁（降级为读锁）；不支持锁升级是为了防止多线程同时持读锁升级导致的死锁。
- **高/低 16 位拆分位运算**：
  - 读锁次数：`state >>> 16`（同时用 `ThreadLocal` `HoldCounter` 记录每个读线程自己的重入次数）。
  - 写锁次数：`state & ((1 << 16) - 1)`（即 `state & 0xFFFF`）。
- **锁降级标准代码与意义**：
  ```java
  rwLock.writeLock().lock();
  try {
      updateCache(); // 1. 修改共享数据
      rwLock.readLock().lock(); // 2. 在释放写锁前先获取读锁（降级开始）
  } finally {
      rwLock.writeLock().unlock(); // 3. 释放写锁，依然持有读锁
  }
  try {
      useCache(); // 4. 安全使用刚修改的数据（防止写锁一释放就被其他写线程篡改）
  } finally {
      rwLock.readLock().unlock(); // 5. 最终释放读锁
  }
  ```
- **为什么不支持锁升级（读锁 → 写锁）**：
  假设线程 A 和线程 B 同时持有读锁，现在两者都试图在持有读锁的前提下升级为写锁：A 等 B 释放读锁才能拿写锁，B 等 A 释放读锁才能拿写锁，**必然立刻发生死锁**！

---

### Q27：JDK 8 `StampedLock` 为什么比 `ReentrantReadWriteLock` 更快？乐观读的原理与致命陷阱是什么？
- **核心结论**：`ReentrantReadWriteLock` 在读多写极少场景下，每次读仍要通过 CAS 修改 `state` 高 16 位，引发高频缓存行失效（伪共享风暴）且写线程易饥饿；`StampedLock` 引入了**无锁的乐观读（`tryOptimisticRead`）**——读操作**不修改任何内存状态，只读取版本号 `stamp`**，读完再校验版本号是否被写线程改过。
- **乐观读标准范式**：
  ```java
  long stamp = stampedLock.tryOptimisticRead(); // 1. 获取乐观读戳（不加锁，无CAS写）
  double curX = x, curY = y;                    // 2. 拷贝共享变量到方法局部变量
  if (!stampedLock.validate(stamp)) {           // 3. 校验期间是否有写线程介入（含内存屏障）
      stamp = stampedLock.readLock();           // 4. 若校验失败，升级为悲观读锁重读
      try {
          curX = x; curY = y;
      } finally {
          stampedLock.unlockRead(stamp);
      }
  }
  return Math.hypot(curX, curY);
  ```
- **`StampedLock` 三大致命陷阱（实战必背）**：
  1. **不可重入！** 同一个线程连续两次调用 `writeLock()` 会直接把自己死锁！
  2. **不支持 `Condition`**（没有 `newCondition()` 方法）。
  3. **严禁对阻塞在 `StampedLock` 上的线程调用 `interrupt()`**！在 JDK 8 中，被中断的 `StampedLock` 等待线程会从内部循环苏醒但未退出循环，直接导致 **CPU 100% 飙高**（若需响应中断，必须使用 `readLockInterruptibly()` / `writeLockInterruptibly()`）。

---

### Q28：`LockSupport.park()` / `unpark()` 的底层原理是什么？与 `wait/notify` 有何本质区别？
- **核心结论**：`LockSupport` 底层调用 `Unsafe.park/unpark`，每个线程在 JVM（`Parker` C++ 对象）中关联一个**二元信号量（许可 `_counter`，取值只能为 0 或 1）**。`unpark` 将许可置为 1，`park` 消费许可（置为 0）并返回；若许可原本就是 0 则挂起等待。
- **与 `wait/notify` 的核心区别**：
  1. **无需提前加锁**：`LockSupport` 可在任意位置调用，不需要包裹在 `synchronized` 中。
  2. **先 `unpark` 后 `park` 不会丢信号**：如果线程 B 先对线程 A 执行了 `unpark(A)`（把许可提前置为 1），随后线程 A 执行 `park()` 会**直接消费这个许可并立即返回，不会阻塞**！（而 `notify` 先于 `wait` 发生则会永久死锁）。注意：连续调用多次 `unpark` 许可上限仍为 1，不会累加。
  3. **精准唤醒**：`unpark(thread)` 可以直接唤醒指定的某一个线程；`notify()` 只能随机唤醒一个。
  4. **不抛中断异常**：被中断时 `park` 只是悄然返回并保留线程中断标志位。

---

## 6. JUC 并发工具类与协同模型 (Q29 - Q32)

### Q29：`CountDownLatch`、`CyclicBarrier`、`Semaphore` 的底层原理与使用场景对比？
- **核心结论**：三者均用于多线程协同控制。`CountDownLatch` 基于 AQS 共享模式实现**一次性倒计时等待**；`CyclicBarrier` 基于 `ReentrantLock + Condition` 实现**可循环使用的多方屏障**；`Semaphore` 基于 AQS 共享模式实现**并发资源限流许可证**。
- **对比矩阵**：
  | 维度 | `CountDownLatch` | `CyclicBarrier` | `Semaphore` |
  | :--- | :--- | :--- | :--- |
  | **底层实现** | AQS 共享模式 (`state` = 计数) | `ReentrantLock` + `Condition` | AQS 共享模式 (`state` = 许可数) |
  | **核心语义** | 一个(或多个)线程等待 N 个任务完成 | N 个线程互相等待，全部到齐后一起出发 | 同一时刻最多允许 N 个线程访问资源 |
  | **是否可复用** | **不可复用**（归零后失效） | **可复用**（支持 `reset()` 或自动重置进入下一代） | **可复用**（`acquire` 减，`release` 还） |
  | **阻塞主体** | 通常是主线程调 `await()`，子线程只管 `countDown()` 不阻塞 | 所有参与线程各自调用 `await()` 并阻塞 | 获取许可失败的线程阻塞 |
  | **典型场景** | 聚合多个微服务的并发查询结果 | 大数据分批计算、并发压测模拟百人同时起跑 | 数据库连接池、接口并发限流 |
- **面试官连环追问**：
  - *问：`CountDownLatch` 的一个线程可以调用两次 `countDown()` 吗？*  
    答：可以，每次 `countDown()` 只是把 AQS `state` 减 1；同理，必须注意把 `countDown()` 放进 **`finally`** 块，防止子线程抛异常导致主线程永远卡死在 `await()`（建议给 `await(timeout, unit)` 设置超时时间兜底）。

---

### Q30：`Phaser` 和 `Exchanger` 适合什么场景？如何用 `Semaphore` 或 `Condition` 手写一个阻塞队列？
- **核心结论**：`Phaser`（JDK 7+）是增强版可动态调整参与方数量的多阶段屏障；`Exchanger` 用于两个线程间安全交换数据（如双缓冲或校对场景）。手写有界阻塞队列标准做法是 **1 把 `ReentrantLock` + 2 个 `Condition`（`notFull` 和 `notEmpty`）**（即 `ArrayBlockingQueue` 源码思路）。
- **`Phaser` vs `CyclicBarrier`**：`CyclicBarrier` 的参与线程数在构造时就写死了；`Phaser` 允许运行期动态调用 `register()` 增加参与者或 `arriveAndDeregister()` 注销参与者，并且支持分阶段回调 `onAdvance(phase, registeredParties)`，适合多步骤游戏关卡或分布式阶段作业。
- **手写核心有界阻塞队列（面试高频白板题）**：
  ```java
  public class SimpleBlockingQueue<T> {
      private final Object[] items;
      private int putIdx, takeIdx, count;
      private final ReentrantLock lock = new ReentrantLock();
      private final Condition notFull = lock.newCondition();
      private final Condition notEmpty = lock.newCondition();

      public SimpleBlockingQueue(int capacity) { this.items = new Object[capacity]; }

      public void put(T x) throws InterruptedException {
          lock.lock();
          try {
              while (count == items.length) notFull.await();
              items[putIdx] = x;
              if (++putIdx == items.length) putIdx = 0;
              count++;
              notEmpty.signal();
          } finally { lock.unlock(); }
      }

      @SuppressWarnings("unchecked")
      public T take() throws InterruptedException {
          lock.lock();
          try {
              while (count == 0) notEmpty.await();
              T x = (T) items[takeIdx];
              items[takeIdx] = null; // 防止内存泄漏
              if (++takeIdx == items.length) takeIdx = 0;
              count--;
              notFull.signal();
              return x;
          } finally { lock.unlock(); }
      }
  }
  ```

---

### Q31：并发场景下如何选择 JUC 线程安全容器？
- **核心结论**：没有绝对最好的并发容器，必须根据**读写比例、阻塞语义、有序性**选型。
- **生产选型速查表**：
  | 容器 | 底层锁/无锁机制 | 适用场景 |
  | :--- | :--- | :--- |
  | `ConcurrentHashMap` | CAS + `synchronized` 锁桶头节点 | 绝大部分高并发 KV 读写（取代 `Hashtable` / `synchronizedMap`） |
  | `CopyOnWriteArrayList` | 写时复制（`ReentrantLock` + `Arrays.copyOf`），读完全无锁 | **极度读多写少**且数据量不大（如白名单配置、监听器列表） |
  | `ConcurrentLinkedQueue` | 基于 CAS 的无锁非阻塞队列 | 高并发异步日志、无需阻塞等待的非阻塞入队出队 |
  | `ArrayBlockingQueue` | 有界数组 + 单锁双 `Condition` | 固定容量削峰、避免内存无限扩张（有界安全） |
  | `LinkedBlockingQueue` | 链表 + **读写分离双锁**（`putLock` / `takeLock`） | 吞吐量要求更高（必须显式传容量，防无界 OOM） |
  | `SynchronousQueue` | 不存储元素的移交队列（`CachedThreadPool` 核心） | 生产者与消费者直接一对一握手传递任务 |

---

### Q32：生产中并发安全的常见设计策略有哪些？（局部化、不可变、分段锁、ThreadLocal）
- **核心结论**：最高效的并发控制不是加更快的锁，而是**尽量不共享**或**只读共享**。
- **4 大架构级并发优化策略**：
  1. **线程封闭（Thread Confinement）**：栈封闭（局部变量）或使用 `ThreadLocal` / JDK 21 `ScopedValue`（如 `SimpleDateFormat` 线程不安全，可改用 `DateTimeFormatter` 或 `ThreadLocal` 封闭）。
  2. **不可变对象（Immutability）**：所有字段设为 `private final` 且不提供修改器，或使用 JDK 17 `record`、`List.of()`，天然绝对线程安全，无需任何锁。
  3. **缩小锁粒度 / 分段锁**：比如扣减热点商品库存时，将 1 个商品的 1000 库存拆分为 `stock_1 ~ stock_10` 10 个分段桶，并发请求哈希到不同桶上扣减，吞吐量提升 10 倍。
  4. **无锁并发（CAS / ThreadLocal 累加）**：计数器改用 `LongAdder`，状态机扭转用 CAS 乐观锁（DB 层 `UPDATE ... WHERE version = old_ver`）。

---

## 7. 线程池、异步编排与 ForkJoinPool (Q33 - Q38)

### Q33：`ThreadPoolExecutor` 的 7 大核心参数、5 种状态与完整工作流程是什么？
- **核心结论**：`ThreadPoolExecutor` 用一个 `AtomicInteger ctl`（高 3 位存线程池状态，低 29 位存当前有效线程数）实现生命周期与线程数的原子绑定管理。任务提交严格遵循 **“核心线程 → 阻塞队列 → 最大线程 → 拒绝策略”** 的四步决策顺序。
- **7 大核心参数**：
  1. `corePoolSize`：常驻核心线程数（默认空闲不销毁，除非设置 `allowCoreThreadTimeOut(true)`）。
  2. `maximumPoolSize`：最大线程数（核心 + 临时线程总上限）。
  3. `keepAliveTime`：非核心空闲线程的存活时间。
  4. `unit`：存活时间单位。
  5. `workQueue`：用于暂存等待执行任务的阻塞队列。
  6. `threadFactory`：线程工厂（**生产必配**：指定清晰业务前缀如 `order-async-%d`，设置是否守护线程及 `UncaughtExceptionHandler`）。
  7. `handler`：队列满且达到最大线程数时的拒绝策略。
- **5 种状态流转（由 `ctl` 高 3 位表示）**：
  - `RUNNING (-1)`：接收新任务并处理队列任务。
  - `SHUTDOWN (0)`：调用 `shutdown()` 进入，不再接收新任务，但会处理完队列中已有的任务。
  - `STOP (1)`：调用 `shutdownNow()` 进入，不再接收新任务，清空队列并中断正在执行的线程。
  - `TIDYING (2)`：所有任务已终止，有效线程数为 0，准备执行 `terminated()` 钩子方法。
  - `TERMINATED (3)`：`terminated()` 执行完毕。
- **完整执行流程图**：
  ```text
  execute(task)
     │
     ├─▶ 1. workerCount < corePoolSize? ──(是)──▶ addWorker(task, true) 创建核心线程执行
     │
     ├─▶ 2. workQueue.offer(task) 入队成功? ──(是)──▶ double-check 池状态(若非RUNNING则回滚拒绝；若线程数为0则补建1个非核心线程)
     │
     ├─▶ 3. workerCount < maximumPoolSize? ──(是)──▶ addWorker(task, false) 创建临时线程执行
     │
     └─▶ 4. 执行拒绝策略 handler.rejectedExecution(task, this)
  ```
- **面试官连环追问**：
  - *问：为什么入队成功后还要进行一次 Double-Check（二次检查状态）？*  
    答：防止任务入队那瞬间，外部刚好调用了 `shutdown()`，或此时所有核心线程刚好因为异常死掉变成 0，导致任务永远躺在队列里无人消费！
  - *问：Tomcat 的线程池和 JDK 原生线程池执行顺序有什么区别？*  
    答：JDK 池是**先入队、队列满再扩临时线程**；Tomcat 面向高并发 IO 响应，重写了 `TaskQueue.offer()`，改为**核心线程满后先直接创建线程直到 `maxThreads`，满了之后才放入阻塞队列**（避免先放无界/大队列导致请求响应慢）。

---

### Q34：线程池有哪 4 种默认拒绝策略？生产如何自定义拒绝策略？
- **核心结论**：JDK 默认提供 4 种拒绝策略：`AbortPolicy`（默认抛异常）、`CallerRunsPolicy`（调用者运行）、`DiscardPolicy`（静默丢弃新任务）、`DiscardOldestPolicy`（丢弃队头老任务再重试）。生产核心业务往往需要自定义策略实现**告警 + 降级持久化补偿**。
- **4 种策略详解与避坑**：
  1. `AbortPolicy`（默认）：抛出 `RejectedExecutionException`。适合必须感知失败并快速失败的核心调用链。
  2. `CallerRunsPolicy`：交由**提交任务的调用者线程**（如 Tomcat HTTP 工作线程、Netty EventLoop 线程）直接执行。**避坑**：如果任务耗时很长（比如 5 秒超时），用 `CallerRunsPolicy` 会直接卡死 Web 容器请求线程或 Netty IO 线程导致雪崩！
  3. `DiscardPolicy`：静默丢弃当前任务，毫无报错。仅适合非核心日志上报/监控埋点。
  4. `DiscardOldestPolicy`：丢弃队列头部的最老任务并再次调用 `execute()` 提交当前任务（若队列是优先级队列会丢弃优先级最高的！）。
- **生产自定义拒绝策略示例**：
  ```java
  public class CustomAlertRejectHandler implements RejectedExecutionHandler {
      @Override
      public void rejectedExecution(Runnable r, ThreadPoolExecutor executor) {
          // 1. 触发监控指标打点与企业微信/钉钉告警
          Metrics.counter("threadpool.rejected", "pool", "order").increment();
          log.error("线程池负载过载触发拒绝! active={}, queueSize={}",
                    executor.getActiveCount(), executor.getQueue().size());
          // 2. 降级策略：若为关键任务，序列化写入本地磁盘/Redis/MQ 等待后续补偿重试
          // compensationService.saveRejectedTask(r);
          // 3. 或者带短暂超时阻塞入队（反向背压）
          try {
              if (!executor.isShutdown() && !executor.getQueue().offer(r, 200, TimeUnit.MILLISECONDS)) {
                  throw new RejectedExecutionException("线程池队列阻塞超时，执行降级拒绝");
              }
          } catch (InterruptedException e) {
              Thread.currentThread().interrupt();
          }
      }
  }
  ```

---

### Q35：为什么阿里《Java 开发手册》禁止使用 `Executors` 创建线程池？线程池内线程抛异常了会怎样？
- **核心结论**：`Executors` 隐藏了资源上限细节，极易引发 **OOM 或线程数失控**。此外，`execute()` 提交的任务若抛出未捕获异常会**直接打印堆栈并销毁当前工作线程**，而 `submit()` 会将异常**封装吞掉**直到调用 `Future.get()` 才抛出。
- **`Executors` 四大工厂方法的 OOM 隐患**：
  1. `newFixedThreadPool` / `newSingleThreadExecutor`：底层使用无界 `LinkedBlockingQueue`（容量 `Integer.MAX_VALUE` ≈ 21 亿），任务堆积耗尽堆内存 → OOM。
  2. `newCachedThreadPool` / `newScheduledThreadPool`：允许的最大线程数 `maximumPoolSize` 为 `Integer.MAX_VALUE`，突发流量会疯狂创建数万线程耗尽本地栈内存 → OOM。
- **线程池异常处理机制**：
  - `execute(Runnable)`：线程执行 `runWorker` 发生未捕获异常，会跳出循环并在 `processWorkerExit` 中销毁该线程，再补建一个新的替补线程。若不配 `UncaughtExceptionHandler` 或内部 `try-catch`，仅会在控制台打印（在定时线程池 `ScheduledThreadPoolExecutor` 中更致命：**一旦某次定时任务抛异常未捕获，后续周期调度直接彻底停止，且无任何报错！**）。
  - `submit(Callable/Runnable)`：返回 `FutureTask`，异常被捕获保存在 `outcome` 字段中，调用 `future.get()` 时才包装成 `ExecutionException` 抛出，如果不调 `get()` 则异常彻底静默消失！
- **实战避坑**：定时任务内部必须最外层 `try-catch(Throwable t)` 兜底打印日志；所有 `ThreadPoolExecutor` 创建必须显式传入有界队列。

---

### Q36：线程池参数如何科学配置与动态调优？美团动态线程池（DynamicTp）原理是什么？
- **核心结论**：理论计算公式只是起点，真实生产参数必须依赖**压测 + 运行期动态调优**。DynamicTp 的核心原理是将线程池注册为 Bean，利用**配置中心（Nacos/Apollo）监听变更**并调用 JDK 原生 `setCorePoolSize()` 等 API 实时修改参数，同时克服了 `LinkedBlockingQueue.capacity` 为 `final` 无法动态修改的限制。
- **经验计算公式（$N$ 为 CPU 核心数）**：
  - **CPU 密集型**（加密、图像编解码、复杂计算）：$N + 1$（多出 1 个防止线程因缺页中断等偶尔停顿导致 CPU 空转）。
  - **IO 密集型**（DB、Redis、RPC 调用）：传统公式 $2N$；更精准公式为 $N \times \left(1 + \frac{WT}{ST}\right)$（$WT$ 为 IO 等待时间，$ST$ 为 CPU 计算时间）。若接口 90ms 在等 RPC、10ms 在算 CPU，理论可配 $N \times 10$（注意同时受限于下游 DB 连接池大小）。
- **美团 DynamicTp 核心揭秘**：
  1. **动态修改核心/最大线程数**：直接调用 `ThreadPoolExecutor.setCorePoolSize()` 与 `setMaximumPoolSize()`（底层实时生效：调大 `corePoolSize` 若有任务会补建线程；调小后空闲线程到期自动回收；**避坑**：先调大 `max` 再调大 `core`，先调小 `core` 再调小 `max`，保证 `core <= max` 约束）。
  2. **动态修改队列容量**：JDK 原生 `LinkedBlockingQueue` 的 `capacity` 是 `final` 字段！DynamicTp 自定义了 **`ResizableCapacityLinkedBlockingQueue`**，去掉 `final` 并暴露 `setCapacity()` 动态调整队列水位线。
  3. **监控预警**：定时采集活性（`activeCount / maximumPoolSize`）、队列使用率（`queueSize / capacity`）、拒绝次数，超过阈值（如 80%）立刻推送报警。

---

### Q37：`FutureTask` 原理是什么？`CompletableFuture` 有哪些常用编排方法及致命生产陷阱？
- **核心结论**：`FutureTask` 实现了 `RunnableFuture`（既是任务又是 `Future`），内部通过 `volatile int state`（`NEW`、`COMPLETING`、`NORMAL`、`EXCEPTIONAL` 等）和 Treiber 栈维护等待结果的线程列表，`get()` 时未完成就 `LockSupport.park()` 阻塞。`CompletableFuture` 是基于异步回调的声明式编排神器，但存在**默认公共池阻塞**与**父子任务死锁**两大生产杀手。
- **`CompletableFuture` 核心 API 速览**：
  - **创建异步**：`supplyAsync(Supplier, executor)`（有返回值）、`runAsync(Runnable, executor)`（无返回值）。
  - **结果转换**：`thenApply`（同步/异步转换，扁平）、`thenCompose`（连接下一个 `CompletableFuture`，相当于 `flatMap`）。
  - **双任务合并**：`thenCombine`（两任务并行，均完成后合并结果）、`applyToEither`（谁先完成用谁）。
  - **多任务聚合**：`CompletableFuture.allOf(f1, f2, f3).join()`（全完成才继续）、`anyOf(...)`（任一完成即返回）。
  - **异常兜底**：`exceptionally`（仅异常时降级）、`handle`（无论成功或异常都执行）。
- **生产四大避坑**：
  1. **必须传入自定义线程池**：不传 executor 参数时，默认使用全局共享的 **`ForkJoinPool.commonPool()`**（线程数为 `CPU核数 - 1`，如果是双核容器甚至只有 1 个线程！），一旦执行阻塞 IO，会拖垮全应用所有 `parallelStream()` 和其他默认异步任务。
  2. **父子任务共用同一线程池导致饥饿死锁**：
     ```java
     // 致命死锁：如果线程池核心=最大=2，同时来了2个父任务把仅有的2个线程占满，
     // 且在里面调用子任务 .join() 等待结果，而子任务正排队在阻塞队列里永远等不到线程执行！
     executor.submit(() -> {
         CompletableFuture.supplyAsync(() -> queryDB(), executor).join(); // 死锁！
     });
     ```
     **解法**：父任务与子任务严格使用**隔离的不同线程池**。
  3. **`allOf` 拿不到单个异常细节的坑**：建议对每个子 `CompletableFuture` 提前用 `.exceptionally()` 兜底或统一封装返回包装对象。
  4. **JDK 9+ 超时保护**：务必配合 `.orTimeout(500, TimeUnit.MILLISECONDS)` 或 `.completeOnTimeout(defaultVal, 500, TimeUnit.MILLISECONDS)`，防止下游挂死永久阻塞。

---

### Q38：`ForkJoinPool` 的工作窃取（Work-Stealing）算法原理是什么？与普通线程池有何区别？
- **核心结论**：`ForkJoinPool` 专为**分治递归大任务**（如大数组排序、大数据汇总、`parallelStream()`）设计。每个工作线程维护自己的**双端队列（WorkQueue）**，自己从队尾 `push`/`pop`（LIFO，利用 CPU 缓存局部性），空闲线程则去其他忙碌线程的**队头窃取任务 `poll`（FIFO）**，从而最大限度消除队列锁竞争与线程闲置。
- **工作窃取架构图**：
  ```text
  Thread 1 (忙碌):  [ Task A ] [ Task B ] [ Task C ] ◀── 自己从队尾(Top) push/pop (LIFO)
                        ▲
                        │ (Thread 2 队列空了，从队头 Base 窃取 Task A，减少碰撞!)
  Thread 2 (空闲):  [   空   ]
  ```
- **核心优势与区别**：
  - 普通 `ThreadPoolExecutor` 所有线程争抢**同一个全局阻塞队列**，锁竞争激烈且父任务等子任务时线程被白白占死。
  - `ForkJoinPool` 在父任务调用 `forkJoinTask.join()` 时，工作线程**不会傻等阻塞**，而是主动去执行自己队列里的子任务或去窃取其他队列的任务（借助 `ManagedBlocker` 接口甚至能在阻塞时临时补偿新线程）。

---

## 8. JDK 21 虚拟线程与现代并发趋势 (Q39 - Q41)

### Q39：JDK 21 虚拟线程（Virtual Threads）底层如何实现 M:N 调度？`mount` 和 `unmount` 是怎么回事？
- **核心结论**：JDK 21 正式发布的虚拟线程（JEP 444）由 JVM 在用户态调度，底层依托一个全局共享的 FIFO 模式 **`ForkJoinPool` 作为载体线程池（Carrier Threads）**，并结合 **`Continuation`（续体，本质是堆上可挂起/恢复的栈帧块）** 实现非阻塞式 `mount`（挂载）与 `unmount`（卸载）。
- **核心运行机制**：
  ```text
  [ 虚拟线程 VT-1 ] ──mount(挂载)──▶ [ 载体平台线程 Carrier-1 ] (正在执行 VT-1 代码)
         │
    遇到阻塞 IO (如 Socket.read / sleep / ReentrantLock.lock)
         │
         ▼
    unmount(卸载)：将 VT-1 的栈帧快照保存到 Java 堆中 (Continuation.yield)
    Carrier-1 立刻自由，去执行队列里的 [ 虚拟线程 VT-2 ]！
         │
    IO 就绪回调唤醒 → VT-1 重新进入 ForkJoinPool 队列，等待 mount 到空闲的 Carrier 上恢复执行。
  ```
- **JDK 内部改造**：JDK 21 几乎重写了标准库中所有的阻塞点（`java.io`、`java.nio`、`java.net.Socket`、`LockSupport.park`、`Thread.sleep`、`BlockingQueue`），当检测到当前处于虚拟线程时，不再走 OS 内核挂起，而是挂起 Continuation 并向操作系统注册非阻塞 Epoll 事件，等事件就绪再重新提交回载体池。
- **核心价值**：让你用**同步阻塞的直观代码风格（Thread-per-Request）**，直接写出接近 Netty/WebFlux 异步非阻塞编程的**百万级并发吞吐能力**！

---

### Q40：什么是虚拟线程的 Pinning（固定/锚定）问题？如何排查与解决？
- **核心结论**：在 JDK 21 中，如果虚拟线程在执行期间进入了 **`synchronized` 代码块/方法**（或调用了 JNI 本地 C/C++ 方法），此时再触发阻塞 IO 操作，虚拟线程**无法从载体线程卸载（unmount）**，从而死死钉住（Pin）宝贵的底层 OS 平台线程，这种现象称为 **Pinning**。如果大量虚拟线程同时被钉住，会导致载体线程池耗尽、整个系统吞吐量雪崩。
- **为什么 JDK 21 会发生 Pinning**：
  因为 JDK 21 中 `synchronized` 的 Monitor 锁持有者记录的是**底层操作系统平台线程 ID（或栈帧指针绑死 OS 栈）**，如果此时卸载换另一个虚拟线程上来执行，会导致锁持有状态混乱。
- **排查与解决之道**：
  1. **排查参数**：启动时加上 **`-Djdk.tracePinnedThreads=full`**（或 `short`），一旦发生 Pinning 阻塞就会打印完整栈追踪；生产低开销排查可使用 **JFR 事件 `jdk.VirtualThreadPinned`**（默认阈值为 20ms）。
  2. **JDK 21 临时解法**：将高耗时 IO 操作外层的 `synchronized` 替换为 **`ReentrantLock`**（JUC 显式锁底层基于 `LockSupport`，完全支持安全卸载！）。注意：纯内存快速操作（无 IO）的 `synchronized` 问题不大。
  3. **彻底解决（JDK 24+ 新进展）**：**JEP 491（Synchronize Virtual Threads without Pinning）** 已在 JDK 24 中彻底重做了 JVM `ObjectMonitor`，让虚拟线程自身直接拥有监视器锁，从此 `synchronized` 内部执行阻塞 IO 也能自由 `unmount`！

---

### Q41：为什么虚拟线程要配合 `ScopedValue` 和结构化并发（`StructuredTaskScope`）使用，而要慎用 `ThreadLocal` 和线程池？
- **核心结论**：千万级动态创建销毁的虚拟线程直接打破了平台线程“重资源池化”的假设。**严禁池化虚拟线程**，**慎用大对象 `ThreadLocal`**，推荐改用内存开销极小且自动清理的 **`ScopedValue`** 与生命周期受控的 **结构化并发**。
- **三大最佳实践原理解析**：
  1. **为什么不要池化虚拟线程**：创建虚拟线程只需几百字节内存和微秒级耗时，像普通对象一样用完即毁（`Executors.newVirtualThreadPerTaskExecutor()`）即可，无需复用；池化反而会导致资源限制或逻辑复杂化。如果真需要限制下游并发数（比如 DB 最大连接数只有 50），应当使用 **`Semaphore(50)`** 来限流，而不是用线程池！
  2. **为什么慎用 `ThreadLocal`（改用 `ScopedValue`）**：
     - 平台线程只有几百个，用 `ThreadLocal` 缓存一个 1MB 的缓冲区或数据库连接还能接受；如果百万个虚拟线程每个都存一份大对象副本，会瞬间炸穿堆内存！
     - `ThreadLocal` 可变且依赖手动 `remove()` 防止内存泄漏；而 **`ScopedValue`（JDK 21 预览 / 23-25 演进转正）** 是**不可变（Immutable）且有严格作用域边界**的上下文传参神器：
       ```java
       static final ScopedValue<User> CURRENT_USER = ScopedValue.newInstance();
       // 仅在 run() 作用域内绑定可读，离开代码块自动销毁，零泄漏风险，子虚拟线程还能零拷贝继承！
       ScopedValue.where(CURRENT_USER, user).run(() -> handleRequest());
       ```
  3. **结构化并发（`StructuredTaskScope`）**：解决传统线程池启动多个子任务时，如果其中一个抛异常或主线程取消了，其他子任务沦为“孤儿线程”仍在后台白跑浪费资源的问题。它确保子任务的生命周期严格嵌套在父任务代码块作用域内（`ShutdownOnFailure` 任一失败即全部取消；`ShutdownOnSuccess` 任一成功即返回并取消其余）。
