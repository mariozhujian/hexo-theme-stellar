---
title: synchronized 和 ReentrantLock
date: 2026-07-14 10:43:00
categories:
- 技术
- 面试

cover: https://69zvfy2e8b.ucarecd.net/cf6ef270-1ddb-4de6-9331-c0aa8295ee95/lockcover.png
---
1. 它解决什么问题？ 
2. 底层原理是什么？ 
3. 常见问题是什么？ 
4. 常见方案是什么？ 
5. 项目里怎么用？ 
6. 面试官可能追问什么？

按照这 **6 个问题** 维度来讲，面试时可以这样回答：

# synchronized / ReentrantLock

先给一句总览：

> `synchronized` 和 `ReentrantLock` 都是 Java 里解决 **多线程并发访问共享资源** 的锁机制。  
> `synchronized` 是 JVM 内置锁，使用简单；`ReentrantLock` 是 JUC 包里的显式锁，功能更强，但需要手动释放。

---

# 1. 它解决什么问题？

它们主要解决 **多线程并发安全问题**。

比如多个线程同时修改同一个变量、同一个集合、同一份缓存数据时，可能会出现：

```java
count++;
```

这行代码看起来是一步，其实底层至少包含：

```text
读取 count
计算 count + 1
写回 count
```

如果两个线程同时执行，就可能出现丢失更新。

例如：

```text
count = 0

线程 A 读取 count = 0
线程 B 读取 count = 0
线程 A 写回 1
线程 B 写回 1

理论结果应该是 2，实际结果变成 1
```

所以锁要解决的问题就是：

1. **原子性**：同一时间只能有一个线程执行关键代码。
    
2. **可见性**：一个线程修改的数据，其他线程能看到。
    
3. **有序性**：避免指令重排带来的并发问题。
    
4. **线程协作**：让多个线程按规则竞争资源。
    

---

# 2. 底层原理是什么？

## 一、synchronized 的底层原理

`synchronized` 是 JVM 层面的锁。

常见用法有三种：

```java
public synchronized void method() {
    // 锁当前对象 this
}
```

```java
public static synchronized void method() {
    // 锁当前 Class 对象
}
```

```java
synchronized (lock) {
    // 锁指定对象
}
```

它的核心原理可以理解为：

> 每个 Java 对象都可以作为一把锁，对象头里有一块区域叫 **Mark Word**，里面会记录锁状态、线程 ID、哈希码等信息。

当线程进入同步代码块时，会尝试获取对象的 Monitor 锁。

字节码层面，代码块形式的 `synchronized` 会生成：

```text
monitorenter
monitorexit
```

大致流程是：

```text
线程进入 synchronized
        ↓
尝试获取对象 Monitor
        ↓
获取成功：执行同步代码
        ↓
执行完毕：monitorexit 释放锁
        ↓
其他线程再竞争
```

`synchronized` 是 **可重入锁**。

例如：

```java
public synchronized void a() {
    b();
}

public synchronized void b() {
    // 同一个线程可以重复拿到同一把锁
}
```

同一个线程已经拿到了锁，再次进入同一把锁保护的方法时，不会被自己阻塞。

---

## 二、synchronized 的锁升级

在 JDK 6 以后，`synchronized` 做了很多优化，不再是早期那种很重的锁。

常见锁状态可以理解为：

```text
无锁
 ↓
偏向锁
 ↓
轻量级锁
 ↓
重量级锁
```

不过面试时要注意补一句：

> 在 JDK 8 面试语境里，通常会讲偏向锁、轻量级锁、重量级锁；但在较新的 JDK 版本中，偏向锁已经被弱化甚至移除，所以重点理解轻量级锁和重量级锁即可。

简单理解：

## 1. 偏向锁

适合只有一个线程反复进入同步代码的场景。

比如一直是线程 A 在访问：

```text
锁偏向线程 A
线程 A 再进来，几乎不用竞争
```

## 2. 轻量级锁

适合多个线程交替访问，但竞争不激烈。

它主要依赖 CAS 操作尝试获取锁。

## 3. 重量级锁

如果竞争激烈，线程就会被阻塞、唤醒，涉及操作系统层面的线程调度，成本比较高。

```text
线程竞争失败
   ↓
进入阻塞队列
   ↓
等待操作系统唤醒
```

---

## 三、ReentrantLock 的底层原理

`ReentrantLock` 是 `java.util.concurrent.locks` 包下的显式锁。

典型用法：

```java
private final ReentrantLock lock = new ReentrantLock();

public void update() {
    lock.lock();
    try {
        // 临界区代码
    } finally {
        lock.unlock();
    }
}
```

它的底层核心是：

```text
AQS：AbstractQueuedSynchronizer
```

AQS 可以理解为 JUC 并发包里的基础框架，很多工具都基于它实现，比如：

```text
ReentrantLock
CountDownLatch
Semaphore
ReentrantReadWriteLock
ThreadPoolExecutor 里的部分同步控制
```

`ReentrantLock` 主要依赖 AQS 里的一个变量：

```java
private volatile int state;
```

对于 `ReentrantLock` 来说：

```text
state = 0：没有线程持有锁
state > 0：锁已经被某个线程持有
```

加锁过程大概是：

```text
线程尝试 CAS 修改 state，从 0 改成 1
        ↓
成功：获取锁
        ↓
失败：进入 AQS 队列等待
        ↓
前面的线程释放锁后，唤醒队列里的线程
```

可以画成：

```text
线程 A 获取锁成功
state = 1
owner = 线程 A

线程 B 获取锁失败
进入 AQS 队列等待

线程 C 获取锁失败
进入 AQS 队列等待

A 释放锁
state = 0

唤醒 B 或 C
```

---

## 四、ReentrantLock 也是可重入锁

例如：

```java
lock.lock();
try {
    lock.lock();
    try {
        // 同一个线程可以重复加锁
    } finally {
        lock.unlock();
    }
} finally {
    lock.unlock();
}
```

同一个线程每加一次锁，`state` 就加 1。

```text
第一次 lock：state = 1
第二次 lock：state = 2
第一次 unlock：state = 1
第二次 unlock：state = 0，真正释放锁
```

所以加锁和释放锁次数必须匹配。

---

## 五、ReentrantLock 支持公平锁和非公平锁

默认是非公平锁：

```java
ReentrantLock lock = new ReentrantLock();
```

公平锁：

```java
ReentrantLock lock = new ReentrantLock(true);
```

区别：

```text
公平锁：先来先得，排队获取锁
非公平锁：后来的线程也可以插队抢锁
```

非公平锁性能通常更好，因为减少了线程切换和排队成本。

面试回答可以说：

> ReentrantLock 默认是非公平锁，因为非公平锁吞吐量更高；公平锁更符合先来先得，但性能相对差一些。

---

# 3. 常见问题是什么？

## 问题一：死锁

两个线程互相等待对方释放锁。

```java
synchronized (lockA) {
    synchronized (lockB) {
        // ...
    }
}
```

另一个线程：

```java
synchronized (lockB) {
    synchronized (lockA) {
        // ...
    }
}
```

结果：

```text
线程 1 拿着 lockA，等待 lockB
线程 2 拿着 lockB，等待 lockA
双方都不释放，死锁
```

---

## 问题二：锁粒度过大

例如：

```java
public synchronized void handle() {
    queryDb();
    callRemoteApi();
    updateCache();
}
```

这个方法里既有数据库查询，又有远程接口调用，还加了整方法锁。

问题是：

```text
锁持有时间太长
并发能力下降
线程大量阻塞
接口变慢
```

---

## 问题三：ReentrantLock 忘记释放锁

错误写法：

```java
lock.lock();

// 如果这里抛异常，unlock 不会执行
doSomething();

lock.unlock();
```

正确写法：

```java
lock.lock();
try {
    doSomething();
} finally {
    lock.unlock();
}
```

---

## 问题四：误以为 synchronized 可以锁住分布式场景

`synchronized` 和 `ReentrantLock` 只能锁住 **当前 JVM 内部的线程**。

如果服务部署了多台机器：

```text
服务 A 实例 1
服务 A 实例 2
服务 A 实例 3
```

每个 JVM 里的锁都是自己的，不能跨机器生效。

所以：

```text
单机多线程：synchronized / ReentrantLock 可以
多机分布式：不够，需要 Redis 分布式锁、数据库锁、Zookeeper 等
```

---

## 问题五：锁对象使用错误

例如：

```java
synchronized (new Object()) {
    // 错误
}
```

每次都 new 一个新的锁对象，线程之间锁的不是同一把锁，完全没有意义。

还有这种：

```java
synchronized ("LOCK") {
    // 不推荐
}
```

字符串常量池可能被其他地方复用，存在风险。

更推荐：

```java
private final Object lock = new Object();

synchronized (lock) {
    // ...
}
```

---

## 问题六：锁和事务边界混乱

比如：

```java
synchronized (lock) {
    updateDb();
    sendMq();
}
```

如果锁里包含数据库事务、MQ 发送、远程调用，可能导致锁持有时间变长，也可能出现事务提交前后数据不一致的问题。

---

# 4. 常见方案是什么？

## 方案一：优先缩小锁范围

不推荐：

```java
public synchronized void handle() {
    validate();
    queryDb();
    updateMemory();
}
```

推荐：

```java
public void handle() {
    validate();
    queryDb();

    synchronized (lock) {
        updateMemory();
    }
}
```

只锁真正需要保护的共享资源。

---

## 方案二：锁对象要固定

推荐：

```java
private final Object lock = new Object();

public void update() {
    synchronized (lock) {
        // 修改共享数据
    }
}
```

不要锁临时对象、字符串、包装类。

---

## 方案三：ReentrantLock 必须 finally 释放

标准模板：

```java
lock.lock();
try {
    // 业务逻辑
} finally {
    lock.unlock();
}
```

面试时可以直接说：

> 使用 ReentrantLock 时，必须在 finally 中释放锁，否则异常会导致锁无法释放，后续线程永久阻塞。

---

## 方案四：避免死锁

常见办法：

1. 固定加锁顺序。
    
2. 减少嵌套锁。
    
3. 使用 `tryLock()` 设置超时时间。
    
4. 不要在锁里调用远程接口。
    
5. 不要在锁里执行耗时任务。
    

例如：

```java
if (lock.tryLock(3, TimeUnit.SECONDS)) {
    try {
        // 获取锁成功
    } finally {
        lock.unlock();
    }
} else {
    // 获取锁失败，走降级逻辑
}
```

---

## 方案五：读多写少用读写锁

如果是读多写少场景，可以用：

```java
ReentrantReadWriteLock
```

例如缓存场景：

```text
大量线程读缓存
少量线程更新缓存
```

读读之间不互斥，读写、写写之间互斥。

---

## 方案六：高并发计数用 LongAdder

如果只是做计数，不一定要用锁。

不推荐：

```java
synchronized (lock) {
    count++;
}
```

高并发下可以用：

```java
LongAdder adder = new LongAdder();

adder.increment();
```

`LongAdder` 通过分段累加降低竞争，适合高并发计数。

---

## 方案七：优先使用并发工具类

很多场景不需要自己加锁。

例如：

```text
HashMap 多线程不安全
可以换 ConcurrentHashMap

ArrayList 多线程不安全
可以换 CopyOnWriteArrayList

普通队列不安全
可以换 BlockingQueue
```

常见替代方案：

```java
ConcurrentHashMap
AtomicInteger
LongAdder
BlockingQueue
Semaphore
CountDownLatch
CyclicBarrier
CompletableFuture
```

---

# 5. 项目里怎么用？

结合你 Java 后端项目，可以这样回答：

## 场景一：本地缓存初始化防止重复加载

例如项目中有本地缓存：

```java
private volatile Map<String, Object> localCache;
private final Object lock = new Object();

public Map<String, Object> getCache() {
    if (localCache == null) {
        synchronized (lock) {
            if (localCache == null) {
                localCache = loadFromDb();
            }
        }
    }
    return localCache;
}
```

面试表达：

> 在项目中，如果某些配置、字典、规则数据需要懒加载到本地缓存，我会使用 synchronized 做双重检查，避免多个线程同时初始化缓存。

---

## 场景二：防止定时任务在单 JVM 内重复执行

例如 XXL-Job 或本地定时任务中，有些任务不能并发执行：

```java
private final ReentrantLock taskLock = new ReentrantLock();

public void executeTask() {
    if (!taskLock.tryLock()) {
        return;
    }

    try {
        // 执行任务
    } finally {
        taskLock.unlock();
    }
}
```

面试表达：

> 在定时任务中，我会使用 ReentrantLock 的 tryLock 防止同一个 JVM 内任务重入。如果上一次任务还没执行完，下一次调度直接跳过，避免重复处理数据。

---

## 场景三：设备状态、规则配置更新

比如 MES、WMS、IOT、设备健康度这类系统里，经常会有：

```text
设备状态缓存
规则配置缓存
指标统计缓存
告警状态缓存
```

如果多个线程同时更新同一个设备状态，可以使用锁保护。

```java
private final ConcurrentHashMap<String, ReentrantLock> lockMap = new ConcurrentHashMap<>();

public void updateDeviceStatus(String deviceId) {
    ReentrantLock lock = lockMap.computeIfAbsent(deviceId, k -> new ReentrantLock());

    lock.lock();
    try {
        // 更新某个设备的状态
    } finally {
        lock.unlock();
    }
}
```

这里不是所有设备共用一把大锁，而是按 `deviceId` 分段加锁。

面试表达：

> 如果是按设备维度更新状态，我不会使用一把全局锁，而是按 deviceId 做细粒度锁，降低不同设备之间的锁竞争。

---

## 场景四：库存、订单类场景

比如秒杀、库存扣减，单 JVM 内可以用锁保护：

```java
synchronized (lock) {
    if (stock > 0) {
        stock--;
    }
}
```

但真实分布式系统中不能只靠这个。

面试表达：

> 如果是单机应用，可以使用 synchronized 或 ReentrantLock 控制库存扣减；但如果服务是多实例部署，就需要 Redis 分布式锁、数据库乐观锁、库存预扣、MQ 异步削峰等方案。

---

## 场景五：接口幂等控制

比如防止重复提交：

```text
用户连续点击提交
接口被重复调用
MQ 消息重复消费
定时任务重复处理
```

单 JVM 内可以用锁，但更常用的是：

```text
Redis setnx
数据库唯一索引
状态机控制
消息幂等表
```

面试表达：

> 本地锁只能解决当前 JVM 的并发问题，对于分布式重复提交，我一般会使用 Redis SETNX、数据库唯一索引或者业务状态机来保证幂等。

---

# 6. 面试官可能追问什么？

## 追问一：synchronized 和 ReentrantLock 有什么区别？

可以这样答：

|对比点|synchronized|ReentrantLock|
|---|---|---|
|实现层面|JVM 内置|JUC，基于 AQS|
|是否可重入|是|是|
|是否需要手动释放|不需要|需要手动 unlock|
|是否支持公平锁|不支持直接设置|支持|
|是否支持中断|不灵活|支持 lockInterruptibly|
|是否支持超时获取锁|不支持|支持 tryLock|
|条件队列|wait/notify|Condition|
|使用难度|简单|稍复杂|
|适用场景|普通同步场景|复杂并发控制场景|

总结一句：

> 简单同步优先用 synchronized；需要公平锁、可中断、超时等待、多条件队列时，用 ReentrantLock。

---

## 追问二：synchronized 是公平锁吗？

答：

> synchronized 不是公平锁。线程释放锁后，等待线程会竞争锁，JVM 不保证严格按照先来先得的顺序获取锁。

---

## 追问三：ReentrantLock 默认是公平锁还是非公平锁？

答：

> 默认是非公平锁。非公平锁允许新来的线程插队竞争锁，吞吐量通常更高。

代码：

```java
new ReentrantLock();       // 非公平锁
new ReentrantLock(true);   // 公平锁
new ReentrantLock(false);  // 非公平锁
```

---

## 追问四：什么是可重入锁？

答：

> 可重入锁就是同一个线程已经获取了一把锁之后，可以再次获取这把锁，不会被自己阻塞。

例如：

```java
public synchronized void a() {
    b();
}

public synchronized void b() {
    // 可以正常进入
}
```

如果锁不可重入，`a()` 里面调用 `b()` 就会自己把自己锁死。

---

## 追问五：ReentrantLock 为什么必须 finally 释放？

答：

> 因为 ReentrantLock 是显式锁，不像 synchronized 那样由 JVM 自动释放。如果业务代码抛异常，unlock 没执行，就会导致锁永远不释放，其他线程一直阻塞。

标准写法：

```java
lock.lock();
try {
    // 临界区
} finally {
    lock.unlock();
}
```

---

## 追问六：synchronized 锁的是什么？

答：

取决于用法：

```java
public synchronized void method()
```

锁的是：

```text
当前对象 this
```

```java
public static synchronized void method()
```

锁的是：

```text
当前类的 Class 对象
```

```java
synchronized (lock) {}
```

锁的是：

```text
括号里的 lock 对象
```

---

## 追问七：synchronized 可以保证可见性吗？

答：

> 可以。线程释放锁时，会把工作内存中的数据刷新到主内存；线程获取锁时，会从主内存读取最新数据。所以 synchronized 既保证原子性，也保证可见性。

---

## 追问八：volatile 和 synchronized 有什么区别？

|对比点|volatile|synchronized|
|---|---|---|
|原子性|不保证复合操作原子性|保证|
|可见性|保证|保证|
|有序性|一定程度保证|保证|
|阻塞|不阻塞|可能阻塞|
|适用场景|状态标记、配置刷新|临界区互斥|

例如：

```java
volatile boolean running = true;
```

适合做线程开关。

但是：

```java
volatile int count;
count++;
```

这样不安全，因为 `count++` 不是原子操作。

---

## 追问九：ReentrantLock 的底层 AQS 是什么？

答：

> AQS 是 AbstractQueuedSynchronizer，是 Java 并发包里的同步器框架。它通过一个 volatile 的 state 状态变量和一个 FIFO 等待队列来实现线程的排队、阻塞和唤醒。

对于 ReentrantLock：

```text
state = 0：锁空闲
state > 0：锁被持有
```

线程获取锁失败后，会进入 AQS 队列，通过 `LockSupport.park()` 挂起；锁释放后，再通过 `unpark()` 唤醒后续线程。

---

## 追问十：什么时候不用锁？

可以这样回答：

> 如果能用无锁结构、并发容器或者原子类解决，就不一定要手写锁。

例如：

```text
计数：AtomicInteger / LongAdder
Map 并发访问：ConcurrentHashMap
队列：BlockingQueue
读多写少：ReadWriteLock
单线程异步消费：MQ + 单消费者
分布式幂等：Redis / DB 唯一索引
```

---

# 面试总结版

你可以这样背：

> `synchronized` 和 `ReentrantLock` 都是为了解决多线程访问共享资源时的线程安全问题。  
> `synchronized` 是 JVM 内置锁，基于对象头 Mark Word 和 Monitor 实现，进入同步块对应 monitorenter，退出对应 monitorexit，使用简单，异常时会自动释放锁。  
> `ReentrantLock` 是 JUC 提供的显式锁，底层基于 AQS，通过 volatile state、CAS 和等待队列实现，支持公平锁、非公平锁、可中断、tryLock 超时获取锁以及多个 Condition 条件队列。  
> 实际项目中，简单同步场景我会优先使用 synchronized；如果需要超时获取锁、防止任务重复执行、支持公平性或者更复杂的线程协作，我会使用 ReentrantLock。  
> 需要注意的是，这两种锁都只在单 JVM 内生效，如果是分布式部署，就需要 Redis 分布式锁、数据库唯一索引、乐观锁或者 MQ 幂等方案。