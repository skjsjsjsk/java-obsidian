# 一、线程与进程基础
## 1. 进程和线程有什么区别？
### 15 秒简答
- 进程是操作系统资源分配的基本单位；线程是 CPU 调度和执行的基本单位。
- 进程拥有独立地址空间，线程共享进程的堆和方法区，但每个线程有独立的程序计数器、虚拟机栈。
- 进程切换开销大，线程切换开销小；多进程更健壮，多线程更轻量。
### 3 分钟详答
从面试视角讲，可以从“单位、资源、通信、健壮性”四个维度回答。
1. 单位不同  
   - 进程是操作系统资源分配的基本单位，每个进程有独立的虚拟地址空间。  
   - 线程是 CPU 调度的基本单位，是进程中的执行实体，一个进程内至少有一个线程。
2. 资源与开销  
   - 进程：有独立的代码段、数据段、堆栈，进程间内存相互隔离，切换需要切换页表等，开销较大。  
   - 线程：同一进程内的线程共享进程的堆和方法区（JDK 8 后的元空间），但每个线程有自己独立的程序计数器、虚拟机栈、本地方法栈，因此线程切换只需保存少量寄存器、栈帧等，开销小很多。
3. 通信与同步  
   - 进程间通信（IPC）需要依赖管道、消息队列、共享内存、信号量等显式机制。  
   - 线程间可直接读写共享变量，通信简单，但需要同步（如 synchronized、Lock）来避免竞态条件。
4. 健壮性  
   - 一个进程崩溃一般不会影响其他进程（保护模式）。  
   - 一个线程崩溃（尤其是访问非法内存等）可能导致整个进程崩溃，因此多进程通常更健壮，多线程更轻量、高效但需要更谨慎的并发控制。
追问：为什么说“线程共享堆，栈是线程私有”？  
简答：堆是进程级别分配的对象存储区域，所有线程共享；栈是每个线程方法调用链的栈帧空间，保存局部变量、操作数栈等，为了线程安全而设计为私有。
---
## 2. Java 线程有哪几种状态？如何流转？
### 15 秒简答
- 新建（NEW）、可运行（RUNNABLE）、阻塞（BLOCKED）、等待（WAITING）、计时等待（TIMED_WAITING）、终止（TERMINATED）。
- 调用 `start()` 从 NEW → RUNNABLE；争锁失败 → BLOCKED；`wait()`/`join()` 等 → WAITING；`sleep(timeout)` 等 → TIMED_WAITING；`run()` 结束或异常退出 → TERMINATED。
### 3 分钟详答
Java 线程状态定义在 `Thread.State` 枚举中，共 6 种状态：
1. NEW：线程对象已创建，尚未调用 `start()`。  
2. RUNNABLE：就绪/运行态，正在运行或等待 CPU 时间片。  
3. BLOCKED：等待监视器锁（synchronized），进入同步块/方法时争锁失败。  
4. WAITING：无限期等待，需要其他线程显式唤醒，例如：
   - `Object.wait()` / `Condition.await()`  
   - `Thread.join()`（无超时）  
   - `LockSupport.park()`  
5. TIMED_WAITING：有限期等待，时间到或被中断会自动唤醒：
   - `Thread.sleep(timeout)`  
   - `Object.wait(timeout)` / `Condition.awaitNanos(timeout)`  
   - `Thread.join(timeout)`  
   - `LockSupport.parkNanos()` / `parkUntil()`  
6. TERMINATED：`run()` 方法执行完毕或异常退出，线程终止。
典型流转：
- `new Thread()` → NEW  
- `t.start()` → RUNNABLE  
- 在 RUNNABLE 中争 synchronized 锁失败 → BLOCKED  
- 执行 `wait()` → WAITING  
- 执行 `sleep(ms)` 或 `wait(timeout)` → TIMED_WAITING  
- `run()` 正常返回或抛出未捕获异常 → TERMINATED
追问：BLOCKED 和 WAITING 有什么区别？  
简答：BLOCKED 是在等待锁进入同步块；WAITING 是在等待其他线程执行特定操作（如 notify、join 完成），本身已经持有锁或不需要锁。
---
## 3. 创建线程有哪几种方式？推荐哪种？
### 15 秒简答
- 继承 `Thread` 类、实现 `Runnable` 接口、实现 `Callable` 接口配合 `FutureTask`、使用线程池。
- 一般推荐实现 `Runnable` 接口，避免单继承限制，任务与线程解耦。
### 3 分钟详答
常见方式：
1. 继承 `Thread` 类  
   - 重写 `run()`，直接 `new MyThread().start()`。  
   - 缺点：Java 单继承，任务与线程强耦合。
2. 实现 `Runnable` 接口  
   - 实现 `run()`，通过 `new Thread(new MyRunnable()).start()`。  
   - 优点：任务类可以继承其他类；多个线程可共享同一个 `Runnable` 对象，适合共享资源场景。
3. 实现 `Callable` 接口  
   - 实现 `call()`，有返回值、可抛异常，通过 `FutureTask` 包装后交给 `Thread` 执行。  
   - 适用于需要获取结果或处理异常的任务。
4. 使用线程池  
   - 通过 `ExecutorService` 提交 `Runnable` 或 `Callable`，线程池内部管理线程创建和复用。  
   - 实际生产中推荐使用线程池，而不是显式 `new Thread()`。
追问：`Runnable` 和 `Callable` 有什么区别？  
简答：`Runnable` 的 `run()` 无返回值、不能抛受检异常；`Callable` 的 `call()` 有返回值、可抛异常，配合 `Future` 获取结果。
---
## 4. 为什么调用 `start()` 会执行 `run()`？能否直接调用 `run()`？
### 15 秒简答
- `start()` 会由 JVM 创建新线程，新线程再调用 `run()`；直接调用 `run()` 只是普通方法调用，还在原线程执行。
- `start()` 只能调用一次，`run()` 可以多次调用，但不会启动新线程。
### 3 分钟详答
- `start()` 的作用是启动线程，JVM 会调用本地方法创建线程，并在新线程中执行 `run()` 方法，实现真正的多线程。  
- 直接调用 `run()` 只是在当前线程（比如主线程）里执行一个普通方法，不会创建新线程，因此是多线程“伪启动”。
面试中常见追问：`start()` 可以多次调用吗？  
简答：不可以，线程一旦启动后就从 NEW 变为 RUNNABLE，再次调用 `start()` 会抛 `IllegalThreadStateException`。
---
## 5. `sleep()` 和 `wait()` 有什么区别？
### 15 秒简答
- `sleep()` 是 `Thread` 的静态方法，不释放锁，超时后自动恢复。
- `wait()` 是 `Object` 的方法，必须在同步块中调用，会释放锁，需要 `notify()`/`notifyAll()` 唤醒。
### 3 分钟详答
关键区别：
1. 所属类与调用位置  
   - `sleep()` 定义在 `Thread` 中，可在任何位置调用。  
   - `wait()` 定义在 `Object` 中，必须在 `synchronized` 方法/块中调用，否则抛 `IllegalMonitorStateException`。
2. 锁行为  
   - `sleep()`：当前线程休眠，但不会释放持有的任何锁。  
   - `wait()`：当前线程释放对象锁，进入该对象的等待集，让出 CPU。
3. 唤醒方式  
   - `sleep(timeout)`：时间到自动恢复。  
   - `wait(timeout)`：时间到或被其他线程 `notify()`/`notifyAll()` 唤醒。  
   - 无参 `wait()` 必须被 `notify()`/`notifyAll()` 唤醒。
追问：`wait()` 为什么要放在同步块里？  
简答：因为 `wait()` 会释放并重新获取监视器锁，只有在同步块中才能保证当前线程持有该对象锁，避免竞态条件。
---
# 二、线程安全与同步机制
## 6. 什么是线程安全？如何保证线程安全？
### 15 秒简答
- 线程安全：多线程访问同一对象时，无论运行时环境如何调度，都能得到正确结果。
- 常见手段：加锁（synchronized、Lock）、使用线程安全类、使用不可变对象、使用原子类、避免共享。
### 3 分钟详答
线程安全的本质是：对共享资源的访问，在并发环境下仍然保持正确性。常用方法：
1. 加锁同步  
   - `synchronized` 关键字修饰方法或代码块。  
   - 显式锁 `ReentrantLock` 等实现互斥访问。  
   - 保证原子性和可见性。
2. 使用线程安全类  
   - `ConcurrentHashMap`、`CopyOnWriteArrayList`、`AtomicInteger` 等。  
   - 这些类在内部已经处理了并发访问。
3. 不可变对象  
   - 状态不可变（如 `String`、包装类、`final` 字段构建的对象），天生线程安全，无需同步。
4. 避免共享  
   - 尽量使用局部变量、`ThreadLocal` 线程局部变量，减少共享资源。
5. 原子类 CAS  
   - 使用 `AtomicInteger` 等，通过 CAS 实现非阻塞的原子更新。
追问：`StringBuilder` 和 `StringBuffer` 谁是线程安全的？  
简答：`StringBuffer` 的方法大多用 `synchronized` 修饰，是线程安全的；`StringBuilder` 非线程安全，性能更高，一般在局部变量中使用。
---
## 7. `synchronized` 的底层原理是什么？
### 15 秒简答
- `synchronized` 通过对象的监视器锁（monitor）实现，基于操作系统的互斥锁（Mutex Lock）。
- 同步代码块使用 `monitorenter` / `monitorexit` 指令；同步方法通过 ACC_SYNCHRONIZED 标志。
### 3 分钟详答
从字节码和对象头角度说明：
1. 同步代码块  
   - 编译后，在同步块前后生成 `monitorenter` 和 `monitorexit` 指令。  
   - `monitorenter` 尝试获取对象 monitor 的所有权，计数器 +1；`monitorexit` 计数器 -1，为 0 时释放锁。  
   - 为保证异常情况下也能释放锁，会有两个 `monitorexit` 指令（正常退出和异常退出）。
2. 同步方法  
   - 方法表中设置 ACC_SYNCHRONIZED 标志，JVM 据此自动在方法调用前后进行 monitor 获取和释放，等价于包装了 `monitorenter` / `monitorexit`。
3. monitor 与对象头  
   - 每个对象关联一个 monitor，monitor 中记录锁状态、持有线程 ID、锁计数器等。  
   - 在 HotSpot 中，对象头的 Mark Word 会根据锁状态（无锁、偏向锁、轻量级锁、重量级锁）存储不同信息。
追问：`synchronized` 是可重入锁吗？  
简答：是，同一个线程可以多次获取同一把 monitor 锁，计数器递增，退出时递减，避免死锁。
---
## 8. JDK 1.6 对 `synchronized` 做了哪些优化？
### 15 秒简答
- 引入偏向锁、轻量级锁，自旋锁、自适应自旋、锁消除、锁粗化等优化。
- 在无竞争或轻度竞争场景，尽量避免昂贵的重量级锁。
### 3 分钟详答
JDK 1.6 对 `synchronized` 的优化主要包括：
1. 偏向锁  
   - 当只有一个线程访问同步块时，在对象头中记录线程 ID，后续该线程进入同步块时无需额外同步操作。
2. 轻量级锁  
   - 当有另一个线程竞争偏向锁时，升级为轻量级锁，通过 CAS 在栈帧中创建锁记录并更新对象头，避免使用互斥锁。
3. 自旋锁与自适应自旋  
   - 竞争失败时不立即阻塞，而是自旋（循环尝试）一段时间，可能很快获得锁。  
   - 自适应自旋根据前一次自旋结果动态调整次数。
4. 锁消除  
   - JIT 通过逃逸分析，发现对象只能被一个线程访问，则消除不必要的锁。
5. 锁粗化  
   - 对连续加锁/解锁操作进行合并，扩大锁范围，减少频繁上下文切换。
追问：锁可以降级吗？  
简答：在 HotSpot 中，锁升级一般是单向的（偏向 → 轻量级 → 重量级），不会在运行时主动降级，但 JVM 在安全点可能进行部分优化。
---
## 9. `volatile` 关键字有什么作用？和 `synchronized` 有什么区别？
### 15 秒简答
- `volatile` 保证可见性和禁止指令重排，但不保证原子性。
- `synchronized` 既保证原子性，又保证可见性，是重量级锁。
### 3 分钟详答
从 JMM 角度理解：
1. 可见性  
   - `volatile` 变量的写操作会直接刷新到主内存，读操作直接从主内存读取，避免工作内存缓存不一致。  
   - 普通变量可能被线程缓存，修改后对其他线程不可见。
2. 禁止指令重排  
   - `volatile` 通过插入内存屏障，禁止编译器和处理器对 volatile 操作进行重排序，保证有序性。  
   - 典型场景：单例模式的双重检查锁（DCL），需要 `volatile` 防止对象半初始化状态被其他线程看到。
3. 与 `synchronized` 的区别  
   - `volatile` 只修饰变量，不保证原子性，适用于“一个线程写、多个线程读”的场景。  
   - `synchronized` 可以修饰方法/代码块，保证原子性和可见性，是互斥锁。  
   - `volatile` 不会阻塞线程，`synchronized` 会阻塞。
追问：`volatile` 能保证 `i++` 的原子性吗？  
简答：不能，`i++` 是“读取-修改-写入”三步操作，多线程下仍然会出现竞态条件，需要 `synchronized` 或 `AtomicInteger`。
---
## 10. 什么是 CAS？有什么优缺点？
### 15 秒简答
- CAS（Compare-And-Swap）是比较并交换，是 CPU 级别的原子指令。
- 优点：非阻塞，性能好；缺点：ABA 问题、循环开销、只能保证一个变量的原子性。
### 3 分钟详答
CAS 是实现乐观锁和原子类的基础：
1. 基本思想  
   - CAS 包含三个值：内存地址 V、预期原值 A、新值 B。  
   - 当且仅当 V 中的值等于 A 时，才将 V 更新为 B，否则不做操作或重试。
2. 优点  
   - 不使用锁，不会线程阻塞，减少上下文切换。  
   - 在低竞争场景下性能较高。
3. 缺点  
   - ABA 问题：值从 A → B → A，CAS 无法感知中间变化，可通过版本号/AtomicStampedReference 解决。  
   - 循环开销：CAS 失败会自旋重试，占用 CPU。  
   - 只能保证一个共享变量的原子性。
追问：`AtomicInteger` 如何利用 CAS 实现 `incrementAndGet()`？  
简答：`AtomicInteger` 内部使用 `Unsafe` 类的 CAS 指令，在循环中不断尝试更新值，直到成功为止。
---
## 11. 什么是死锁？如何避免？
### 15 秒简答
- 死锁：两个或多个线程互相等待对方释放锁，导致无限期阻塞。
- 避免策略：破坏循环等待条件（按顺序加锁）、使用定时锁（tryLock）、一次性申请所有资源、死锁检测与恢复。
### 3 分钟详答
死锁产生的四个必要条件：
1. 互斥条件：资源一次只能由一个线程占用。  
2. 占有并等待：线程占有资源，同时等待其他资源。  
3. 不可抢占：资源不能被强制抢占。  
4. 循环等待：存在线程资源的循环等待链。
避免死锁的常见方法：
- 破坏循环等待：所有线程按相同顺序获取多个锁。  
- 限制锁的持有时间：尽量减少临界区长度。  
- 使用显式锁的 `tryLock(timeout)`，超时放弃，避免无限等待。  
- 使用工具进行死锁检测和线程转储分析。
追问：写一个简单的死锁示例？  
简答：两个线程分别先持有锁 A 再请求锁 B，另一个线程先持有锁 B 再请求锁 A，且都使用 `synchronized` 嵌套，就可能形成死锁。
---
## 12. `synchronized` 和 `ReentrantLock` 有什么区别？
### 15 秒简答
- `synchronized` 是 JVM 关键字，自动加锁/解锁；`ReentrantLock` 是 API 级别显式锁。
- `ReentrantLock` 支持公平锁、可中断、定时锁、多条件变量，但需要手动释放。
### 3 分钟详答
对比维度：
1. 层次与使用方式  
   - `synchronized`：JVM 内置关键字，自动加锁、解锁，代码块或方法结束自动释放。  
   - `ReentrantLock`：JDK 类，需要 `lock()` / `unlock()`，必须在 `finally` 中释放锁。
2. 功能特性  
   - 公平性：`ReentrantLock` 可选择公平/非公平；`synchronized` 非公平。  
   - 可中断：`ReentrantLock` 支持 `lockInterruptibly()`；`synchronized` 不可中断。  
   - 定时锁：`tryLock(timeout)`；`synchronized` 无。  
   - 多条件：`ReentrantLock` 可以创建多个 `Condition`；`synchronized` 只有一个隐式条件队列。
3. 性能与优化  
   - JDK 1.6 后 `synchronized` 进行了锁优化，在无竞争或轻度竞争场景性能接近 `ReentrantLock`。  
   - `ReentrantLock` 在复杂场景（需要公平性、可中断、多条件）更灵活。
追问：什么时候选择 `ReentrantLock`？  
简答：需要公平锁、可中断、定时获取锁，或需要多个条件变量时，使用 `ReentrantLock`；一般简单同步优先使用 `synchronized`。
---
## 13. 什么是 AQS？它如何实现锁？
### 15 秒简答
- AQS（AbstractQueuedSynchronizer）是 Java 并发包的基础框架，用于构建锁和同步器。
- 内部维护一个 `state` 状态和一个 CLH 队列，通过 CAS 和 `park`/`unpark` 管理线程排队与唤醒。
### 3 分钟详答
AQS 的核心思想：
1. `state` 状态  
   - 对于独占锁，`state` 表示锁是否被占用以及重入次数。  
   - 对于共享锁，`state` 表示共享资源数量（如信号量）。
2. CLH 队列  
   - AQS 使用一个双向链表队列存储等待获取锁的线程节点。  
   - 每个节点保存线程引用、等待状态等信息。
3. 模板方法模式  
   - 子类通过重写 `tryAcquire(int)`、`tryRelease(int)` 等方法，决定获取/释放逻辑。  
   - AQS 提供模板方法 `acquire()`、`release()`，负责排队、阻塞与唤醒。
4. 线程阻塞与唤醒  
   - 使用 `LockSupport.park()` 阻塞线程，`unpark()` 唤醒，避免底层监视器锁的开销。
追问：`ReentrantLock` 如何基于 AQS 实现？  
简答：`ReentrantLock` 内部组合一个 `Sync` 子类，继承 AQS，`tryAcquire` 使用 CAS 更新 `state`，失败则进入队列等待；`tryRelease` 将 `state` 减 1，为 0 时释放锁。
---
# 三、线程池与并发工具
## 14. 为什么要使用线程池？核心参数有哪些？
### 15 秒简答
- 线程池通过复用线程降低创建和销毁开销，提高响应速度，便于管理线程数量。
- 核心参数：`corePoolSize`、`maximumPoolSize`、`keepAliveTime`、`workQueue`、`threadFactory`、`handler`。
### 3 分钟详答
线程池的作用：
1. 降低资源消耗：复用已创建线程，减少线程创建/销毁的开销。  
2. 提高响应速度：任务到达时可直接使用已有线程执行。  
3. 提高可管理性：统一分配、调优和监控线程。
`ThreadPoolExecutor` 的核心参数：
- `corePoolSize`：核心线程数，即使空闲也会保留（除非设置 `allowCoreThreadTimeOut`）。  
- `maximumPoolSize`：最大线程数，当队列满时创建非核心线程。  
- `keepAliveTime`：非核心线程空闲存活时间。  
- `unit`：时间单位。  
- `workQueue`：阻塞队列，保存等待执行的任务。  
- `threadFactory`：线程工厂，可自定义线程名称、优先级等。  
- `handler`：拒绝策略，当队列满且线程数达到最大线程数时执行。
追问：`Executors` 创建线程池有什么问题？  
简答：`newFixedThreadPool` 和 `newSingleThreadExecutor` 使用无界队列，可能导致 OOM；`newCachedThreadPool` 允许线程数无限增长，也容易 OOM，因此阿里巴巴规范建议直接使用 `ThreadPoolExecutor` 明确参数。
---
## 15. 线程池的执行流程是怎样的？
### 15 秒简答
- 提交任务 → 核心线程数未满 → 创建线程执行；  
- 核心线程已满 → 入队；  
- 队列已满 → 创建非核心线程；  
- 线程数达最大且队列已满 → 执行拒绝策略。
### 3 分钟详答
`ThreadPoolExecutor` 的执行流程：
1. 如果运行的线程数少于 `corePoolSize`，即使有空闲线程，也会创建新线程执行任务。  
2. 如果运行的线程数达到或超过 `corePoolSize`，任务会被加入 `workQueue`。  
3. 如果队列已满，且运行的线程数小于 `maximumPoolSize`，创建非核心线程执行任务。  
4. 如果队列已满，且线程数已达 `maximumPoolSize`，执行拒绝策略（`RejectedExecutionHandler`）。  
5. 当线程数超过 `corePoolSize` 且空闲时间超过 `keepAliveTime`，销毁多余线程，回到核心线程数。
追问：先入队还是先创建最大线程？  
简答：`ThreadPoolExecutor` 默认策略是：先尝试加入队列，队列满再创建非核心线程，而不是直接创建最大线程数。
---
## 16. 线程池有哪些拒绝策略？
### 15 秒简答
- `AbortPolicy`：抛出 `RejectedExecutionException`（默认）。  
- `CallerRunsPolicy`：由调用线程执行任务。  
- `DiscardPolicy`：直接丢弃任务。  
- `DiscardOldestPolicy`：丢弃队列中最旧的任务，再尝试提交。
### 3 分钟详答
`ThreadPoolExecutor` 提供四种内置拒绝策略：
1. `AbortPolicy`：直接抛异常，通知调用方任务被拒绝，是默认策略。  
2. `CallerRunsPolicy`：只要线程池未关闭，就由提交任务的线程（如主线程）执行该任务，可以减缓提交速度。  
3. `DiscardPolicy`：默默丢弃任务，不做任何处理。  
4. `DiscardOldestPolicy`：丢弃队列中最老的一个任务，再尝试提交新任务。
追问：如何自定义拒绝策略？  
简答：实现 `RejectedExecutionHandler` 接口，在 `rejectedExecution` 方法中自定义逻辑，例如记录日志或降级处理。
---
## 17. `submit()` 和 `execute()` 有什么区别？
### 15 秒简答
- `execute()` 只提交 `Runnable`，无返回值。  
- `submit()` 可提交 `Runnable` 或 `Callable`，返回 `Future`，可获取结果或取消任务。
### 3 分钟详答
`ExecutorService` 中：
- `execute(Runnable)`：  
  - 只执行任务，不关心返回值。  
  - 任务抛出的异常会在 `Future.get()` 时包装为 `ExecutionException`，但如果忽略 `Future`，异常可能被“吞掉”。
- `submit(Runnable/Callable)`：  
  - 返回 `Future` 对象，可以：  
    - 通过 `get()` 获取结果（`Runnable` + result 时为传入的结果）。  
    - 通过 `cancel()` 尝试取消任务。  
    - 通过 `isDone()` 判断是否完成。
追问：`Future.get()` 会阻塞吗？  
简答：会，`get()` 会阻塞当前线程，直到任务完成或超时；`get(timeout)` 可以设置超时时间。
---
## 18. `CountDownLatch` 和 `CyclicBarrier` 有什么区别？
### 15 秒简答
- `CountDownLatch`：一次性计数器，等待其他线程完成一组操作，不能重置。  
- `CyclicBarrier`：可循环使用的屏障，一组线程互相等待到达屏障点后再一起继续。
### 3 分钟详答
对比：
1. 用途  
   - `CountDownLatch`：一个线程等待其他线程完成，例如主线程等待多个工作线程结束。  
   - `CyclicBarrier`：一组线程互相等待，所有线程到达后再一起继续，适合分阶段任务。
2. 计数方式  
   - `CountDownLatch`：构造时指定计数，`countDown()` 减 1，`await()` 等待计数到 0，到 0 后不再重置。  
   - `CyclicBarrier`：构造时指定 parties 数量，每个线程 `await()` 到达屏障，数量减 1，为 0 时触发屏障，所有线程释放，并可重置。
3. 可重用性  
   - `CountDownLatch` 是一次性使用的。  
   - `CyclicBarrier` 可以通过 `reset()` 重置，循环使用。
追问：`CyclicBarrier` 的 `await()` 抛出异常时，其他线程会怎样？  
简答：一个线程在 `await()` 中抛出异常或被中断，会导致屏障被打破，其他等待线程会抛出 `BrokenBarrierException`。
---
# 四、并发模型与 JMM
## 19. 什么是 Java 内存模型（JMM）？它解决什么问题？
### 15 秒简答
- JMM 规定了线程和主内存之间的抽象关系：线程间通信通过主内存，线程有工作内存。
- 主要解决可见性、有序性问题，定义 happens-before 规则。
### 3 分钟详答
JMM 是 Java 并发的理论基础：
1. 抽象结构  
   - 主内存：存储共享变量。  
   - 工作内存：每个线程保存变量的副本，线程对变量的操作在工作内存中进行，然后再刷新回主内存。  
   - 线程间通信必须通过主内存。
2. 解决的问题  
   - 可见性：一个线程修改共享变量，其他线程能立刻看到。  
   - 有序性：编译器和处理器会重排序指令，JMM 通过 happens-before 规则限制重排序。  
   - 原子性：由 JVM 和锁机制保证。
3. happens-before 规则  
   - 程序顺序规则：同一线程中，前面的操作 happens-before 后面的操作。  
   - 监视器锁规则：一个 `unlock` 操作 happens-before 后续对同一个锁的 `lock`。  
   - `volatile` 规则：对 `volatile` 变量的写 happens-before 后续的读。  
   - 线程启动规则：`Thread.start()` happens-before 该线程中的任何操作。  
   - 线程终止规则：线程中的操作 happens-before 其他线程检测到它终止。
追问：`volatile` 写-读是如何 happens-before 的？  
简答：JMM 要求对 `volatile` 变量的写操作先于后续的读操作，保证可见性和有序性。
---
## 20. 什么是原子性、可见性、有序性？
### 15 秒简答
- 原子性：一个或多个操作要么全部成功，要么全部失败，不可分割。  
- 可见性：一个线程修改共享变量，其他线程能立刻看到。  
- 有序性：程序执行顺序与代码逻辑顺序一致，防止指令重排导致错误。
### 3 分钟详答
并发编程三要素：
1. 原子性  
   - 例如 `i++` 不是原子操作（读取、修改、写入三步）。  
   - `synchronized`、`Lock`、原子类可以保证原子性。
2. 可见性  
   - 由于缓存一致性，线程可能使用旧值。  
   - `synchronized`、`volatile`、`Lock` 可以保证可见性。
3. 有序性  
   - 编译器和 CPU 会进行指令重排，提高性能，但可能破坏程序逻辑。  
   - `volatile` 和 `synchronized` 可以保证有序性，JMM 的 happens-before 规则定义了有序性边界。
追问：哪些操作是原子的？  
简答：对基本类型（除 `long`/`double` 外）的简单读取和赋值在大多数 JVM 上是原子的；`long`/`double` 的读写在 32 位 JVM 上不是原子，但可以通过 `volatile` 保证原子性。
---
## 21. `ThreadLocal` 的原理和使用场景是什么？会有什么内存泄漏问题？
### 15 秒简答
- `ThreadLocal` 为每个线程提供独立的变量副本，实现线程局部变量。  
- 典型场景：数据库连接、用户上下文、日期格式化等。  
- 内存泄漏：`ThreadLocalMap` 的 key 是弱引用，value 是强引用，线程池场景下若不调用 `remove()` 可能泄漏。
### 3 分钟详答
原理：
1. 数据结构  
   - 每个 `Thread` 内部维护一个 `ThreadLocalMap`，key 是 `ThreadLocal` 对象，value 是线程局部变量。  
   - `ThreadLocal` 作为 key，是弱引用，在 GC 时可能被回收。
2. 使用场景  
   - 数据库连接：每个线程持有自己的 Connection，避免并发访问同一连接。  
   - 用户上下文：在 Web 请求处理链中传递用户信息，无需显式传参。  
   - 日期格式化：每个线程使用独立的 `SimpleDateFormat`，避免线程安全问题。
3. 内存泄漏问题  
   - `ThreadLocalMap` 的 key（`ThreadLocal`）是弱引用，可能被 GC，但 value 是强引用。  
   - 如果线程长时间不结束（线程池），且没有调用 `remove()`，value 无法回收，导致内存泄漏甚至 OOM。  
   - 解决办法：使用完毕后显式调用 `remove()` 清理。
追问：为什么 `ThreadLocal` 的 key 要设计成弱引用？  
简答：为了防止 `ThreadLocal` 对象被强引用导致无法回收，造成内存泄漏；但弱引用 key 被回收后，value 仍需通过 `remove()` 清理。
---
## 22. 什么是上下文切换？对性能有什么影响？
### 15 秒简答
- 上下文切换：CPU 从一个线程切换到另一个线程，需要保存当前线程状态并加载另一个线程状态。
- 切换频繁会消耗大量 CPU 时间，影响性能，应尽量减少不必要的线程和阻塞。
### 3 分钟详答
上下文切换是多线程环境的常见开销：
1. 触发条件  
   - 时间片用完。  
   - 线程主动让出 CPU（`yield`）。  
   - 线程进入阻塞/等待状态（`sleep`、`wait`、I/O、争锁失败等）。
2. 切换过程  
   - 保存当前线程的程序计数器、栈帧等上下文到内核数据结构。  
   - 选择就绪线程，加载其上下文，恢复执行。
3. 性能影响  
   - 切换本身需要 CPU 时间，频繁切换会降低吞吐量。  
   - 大量线程竞争锁、频繁阻塞/唤醒，会导致严重的上下文切换开销。
追问：如何减少上下文切换？  
简答：合理设置线程数，避免过多线程竞争；使用非阻塞算法（CAS）减少锁竞争；缩短临界区，减少阻塞时间。
---
# 五、实战与综合题
## 23. 如何让 3 个线程按顺序执行（T1 → T2 → T3）？
### 15 秒简答
- 使用 `join()`：在 T2 中调用 `t1.join()`，在 T3 中调用 `t2.join()`。
- 或使用 `CountDownLatch`：T2 等待 T1 完成的 latch，T3 等待 T2 完成的 latch。
### 3 分钟详答
常见方案：
1. 使用 `join()`  
   - 主线程启动 T1、T2、T3。  
   - 在 T2 的 `run()` 中调用 `t1.join()`，确保 T1 执行完再执行 T2。  
   - 在 T3 的 `run()` 中调用 `t2.join()`，确保 T2 执行完再执行 T3。
2. 使用 `CountDownLatch`  
   - 定义 `latch1` 和 `latch2`，初始计数为 1。  
   - T1 执行完毕后调用 `latch1.countDown()`。  
   - T2 在开始时调用 `latch1.await()`，执行完再 `latch2.countDown()`。  
   - T3 开始时调用 `latch2.await()`。
追问：`join()` 的底层实现？  
简答：`join()` 本质是通过 `wait()`/`notifyAll()` 实现，在当前线程对象上等待，直到目标线程终止时调用 `notifyAll()`。
---
## 24. 生产者-消费者模型如何实现？
### 15 秒简答
- 使用 `BlockingQueue` 作为共享队列，生产者 `put()`，消费者 `take()`，内部已经处理了等待/通知。
- 或使用 `wait()`/`notifyAll()` 配合同步队列实现。
### 3 分钟详答
两种典型实现：
1. 使用 `BlockingQueue`  
   - 生产者：循环 `queue.put(item)`，队列满时自动阻塞。  
   - 消费者：循环 `item = queue.take()`，队列空时自动阻塞。  
   - 无需显式使用 `synchronized`，`BlockingQueue` 内部使用锁和条件队列。
2. 使用 `wait()`/`notifyAll()`  
   - 共享队列类中，维护一个 `List` 和最大容量。  
   - `put()` 时，队列满则 `wait()`，否则入队并 `notifyAll()`。  
   - `take()` 时，队列空则 `wait()`，否则出队并 `notifyAll()`。  
   - 需要在 `synchronized` 方法/块中调用，避免竞态条件。
追问：为什么推荐使用 `BlockingQueue`？  
简答：`BlockingQueue` 封装了复杂的等待/通知逻辑，避免手动使用 `wait()`/`notify()` 时易出错的问题，如漏掉同步、唤醒条件错误等。
---
## 25. 如何实现一个线程安全的单例模式？
### 15 秒简答
- 静态内部类：利用类加载机制保证线程安全。  
- 双重检查锁（DCL）：使用 `volatile` + `synchronized` 延迟初始化。
### 3 分钟详答
常见线程安全单例实现：
1. 静态内部类  
   - 外部类加载时不会加载内部类，只有调用 `getInstance()` 时才加载。  
   - JVM 保证类加载过程线程安全，且实例在类加载时创建。
2. 双重检查锁  
   - 使用 `volatile` 禁止指令重排，防止半初始化对象被其他线程看到。  
   - 第一次检查避免不必要的锁竞争，第二次检查在锁内保证只有一个线程创建实例。
追问：为什么 DCL 需要 `volatile`？  
简答：`new` 操作不是原子的，包含分配内存、初始化对象、赋引用三步，可能被重排为“分配内存 → 赋引用 → 初始化对象”，导致其他线程拿到未初始化的对象。
---
## 26. 如何优雅地停止一个线程？
### 15 秒简答
- 不推荐使用 `stop()`，它不安全，可能导致数据不一致。  
- 使用 `interrupt()` 设置中断标志，线程在循环或阻塞操作中检查中断并自行退出。
### 3 分钟详答
停止线程的正确方式：
1. 使用 `interrupt()`  
   - 调用 `thread.interrupt()` 设置中断标志。  
   - 线程在 `while (!Thread.currentThread().isInterrupted())` 循环中检查中断标志，主动退出。  
   - 如果线程在 `sleep`、`wait` 等可中断阻塞状态，会抛出 `InterruptedException`，需要在 `catch` 中重新设置中断或退出。
2. 使用 `volatile` 标志位  
   - 定义 `volatile boolean running = true;`，线程循环检查 `running`，外部通过设置为 `false` 停止线程。  
   - 适合不阻塞的任务。
3. 不推荐 `stop()`/`suspend()`  
   - `stop()` 会立即释放所有锁，可能导致对象状态不一致。  
   - `suspend()` 不会释放锁，容易导致死锁。
追问：`InterruptedException` 会清除中断标志吗？  
简答：会，当抛出 `InterruptedException` 时，中断标志会被清除，因此通常需要在 `catch` 中再次调用 `interrupt()` 恢复中断状态。
---
## 27. 如何实现一个简单的阻塞队列？
### 15 秒简答
- 使用 `LinkedList` + `synchronized` + `wait()`/`notifyAll()`，实现 `put()` 和 `take()`。
- 当队列满时 `put()` 等待，队列空时 `take()` 等待。
### 3 分钟详答
核心思路：
1. 成员变量  
   - `final LinkedList<E> queue = new LinkedList<>();`  
   - `final int capacity;`  
   - `final Object lock = new Object();`
2. `put(E e)`  
   - 在 `synchronized(lock)` 中：  
     - 当 `queue.size() == capacity` 时调用 `lock.wait()`。  
     - 否则 `queue.addLast(e)`，然后 `lock.notifyAll()`。
3. `take()`  
   - 在 `synchronized(lock)` 中：  
     - 当 `queue.isEmpty()` 时调用 `lock.wait()`。  
     - 否则 `return queue.removeFirst()`，并 `lock.notifyAll()`。
追问：为什么使用 `while` 而不是 `if` 判断条件？  
简答：`wait()` 可能被虚假唤醒，或者多个线程被唤醒后重新竞争，导致条件再次不满足，因此需要在循环中重新检查条件。
---
## 28. 如何排查死锁问题？
### 15 秒简答
- 使用 `jstack` 获取线程转储，查找“Found one Java-level deadlock”信息。  
- 或通过 `jconsole`、`jvisualvm` 等工具检测死锁。
### 3 分钟详答
排查步骤：
1. 使用 `jstack`  
   - `jstack pid` 打印线程堆栈，会显示死锁相关的线程和锁信息。  
   - `jstack` 会自动检测并打印死锁信息。
2. 使用 `jconsole`/`jvisualvm`  
   - 这些图形化工具可以检测死锁线程，直观展示线程状态和锁持有情况。
3. 代码层面  
   - 尽量避免嵌套锁，按顺序获取锁。  
   - 使用定时锁 `tryLock(timeout)` 减少死锁概率。  
   - 对关键业务增加死锁检测和告警机制。
追问：死锁的四个必要条件是什么？  
简答：互斥条件、占有并等待、不可抢占、循环等待，只有同时满足四个条件才可能发生死锁。
---
## 29. 如何理解“原子类”？
### 15 秒简答
- 原子类（如 `AtomicInteger`）基于 CAS 实现无锁原子操作，适合高并发场景下的简单计数器、标记等。
- 相比 `synchronized`，在低竞争场景性能更好。
### 3 分钟详答
原子类的特点：
1. 包路径  
   - `java.util.concurrent.atomic` 包下，如 `AtomicInteger`、`AtomicLong`、`AtomicReference` 等。
2. 实现原理  
   - 使用 `Unsafe` 类的 CAS 指令保证原子性。  
   - 对于 `AtomicInteger`，`incrementAndGet()` 内部是一个 `while` 循环，不断尝试 CAS 更新值。
3. 使用场景  
   - 高并发计数器、序列号生成、状态标记等。  
   - 适合“简单操作”，复杂事务逻辑仍需锁。
追问：`AtomicInteger` 和 `synchronized` 有什么区别？  
简答：`AtomicInteger` 使用 CAS 非阻塞，在竞争不激烈时性能更好；`synchronized` 是阻塞锁，竞争激烈时性能会下降，但更适合复杂同步块。
---
## 30. 如何在项目中合理配置线程池参数？
### 15 秒简答
- CPU 密集型：线程数接近 CPU 核心数（+1）。  
- I/O 密集型：线程数可以设置得较大，如 CPU 核心数 * (1 + 等待时间/计算时间)。  
- 结合任务类型、队列容量、系统资源进行压测调优。
### 3 分钟详答
配置线程池的一般思路：
1. 任务类型  
   - CPU 密集型：线程数过多会导致频繁上下文切换，一般可设为 CPU 核心数 + 1。  
   - I/O 密集型：线程大部分时间在等待 I/O，可适当增大线程数，提高 CPU 利用率。
2. 队列选择  
   - 有界队列（如 `ArrayBlockingQueue`）可以防止资源耗尽，但需要合理预估容量。  
   - 无界队列（如 `LinkedBlockingQueue`）容易导致任务堆积，OOM 风险较高。
3. 拒绝策略  
   - 根据业务重要程度选择：丢弃、记录日志、降级或由调用线程执行。
4. 动态调整  
   - 根据监控指标（队列长度、活跃线程数、任务执行时间）动态调整线程池参数，如 `setCorePoolSize()`。
追问：如何估算 I/O 密集型线程数？  
简答：一个经验公式是：线程数 = CPU 核心数 * (1 + 等待时间/计算时间)，其中等待时间可以通过压测或监控估算。
---