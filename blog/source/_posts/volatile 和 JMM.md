---
title: volatile 和 JMM
date: 2026-07-14 17:43:00
categories:
- 技术
- 面试

cover: https://69zvfy2e8b.ucarecd.net/4b83db61-c1bd-4cc0-baa5-f996d21a5b27/jmmcover.png
---

![](https://69zvfy2e8b.ucarecd.net/4b83db61-c1bd-4cc0-baa5-f996d21a5b27/jmmcover.png)

# volatile 和 JMM 深度解析

先记住一句话：

> **JMM 是 Java 并发程序的底层规则，volatile 是 JMM 提供的一种轻量级同步手段。**

三者关系：

```text
JMM 规定并发规则
    ↓
volatile 利用这些规则
    ↓
解决线程之间的可见性和有序性问题
```

但 `volatile` **不能保证复合操作的原子性**。

---

# 1. 它解决什么问题？

## 1.1 JMM 解决什么问题？

JMM，全称 Java Memory Model，Java 内存模型。

现代计算机中，每个线程执行代码时，可能会经过：

```text
Java 代码
  ↓
JIT 编译器优化
  ↓
CPU 指令重排序
  ↓
CPU 缓存、写缓冲区
  ↓
主内存
```

如果没有统一规则，同一段多线程 Java 代码，在不同 CPU、操作系统或 JVM 上，可能表现不同。

JMM 主要解决三个问题：

### ① 可见性

一个线程修改了变量，其他线程什么时候能够看到？

```java
boolean running = true;

new Thread(() -> {
    while (running) {
        // 执行任务
    }
}).start();

Thread.sleep(1000);
running = false;
```

主线程将 `running` 修改为 `false`，工作线程却可能一直读取自己之前缓存的 `true`，导致循环无法停止。

这就是**可见性问题**。

---

### ② 原子性

一个操作是否能够作为不可分割的整体完成。

```java
count++;
```

看起来只有一行，实际上大致包含：

```text
1. 读取 count
2. count + 1
3. 写回 count
```

两个线程可能同时读取到 `count = 0`：

```text
线程 A：读取 0
线程 B：读取 0
线程 A：写入 1
线程 B：写入 1
```

执行两次自增，最终却只有 `1`。

这就是**原子性问题**。

---

### ③ 有序性

编译器和 CPU 为了提升性能，可能调整指令执行顺序。

例如：

```java
int data = 0;
boolean ready = false;

// 线程 A
data = 100;
ready = true;
```

在没有同步约束时，底层可能将其重排序为：

```java
ready = true;
data = 100;
```

线程 B：

```java
if (ready) {
    System.out.println(data);
}
```

线程 B 可能看到：

```text
ready = true
data = 0
```

这就是**有序性问题**。

---

## 1.2 volatile 解决什么问题？

`volatile` 主要提供两个保证：

```text
1. 保证变量的可见性
2. 禁止特定类型的指令重排序
```

例如：

```java
private volatile boolean running = true;
```

当一个线程执行：

```java
running = false;
```

其他线程再次读取 `running` 时，能够看到最新值。

### volatile 不解决什么问题？

`volatile` 不保证复合操作的原子性。

```java
private volatile int count = 0;

public void increment() {
    count++;
}
```

这段代码仍然线程不安全，因为 `count++` 是“读取、计算、写入”三个步骤。

---

# 2. 底层原理是什么？

## 2.1 JMM 的抽象模型

JMM 将内存抽象为：

```text
              主内存
        保存所有共享变量
          ↑          ↑
          ↓          ↓
    线程 A 工作内存  线程 B 工作内存
```

每个线程都有自己的工作内存。

线程操作共享变量时，可以抽象为：

```text
主内存中的变量
      ↓
复制到线程工作内存
      ↓
线程读取、修改
      ↓
同步回主内存
```

需要注意：

> 工作内存、主内存是 JMM 的抽象概念，不是简单等同于 CPU 缓存和物理内存。

它们在底层可能对应：

- CPU 寄存器
    
- CPU 多级缓存
    
- 写缓冲区
    
- 主内存
    
- 编译器优化产生的临时变量
    

---

## 2.2 happens-before 规则

JMM 使用 `happens-before` 定义两个操作之间的可见性和顺序关系。

如果操作 A happens-before 操作 B，那么可以保证：

```text
1. A 的执行结果对 B 可见
2. A 的执行顺序排在 B 之前
```

常见 happens-before 规则如下。

### ① 程序顺序规则

同一个线程中，前面的操作 happens-before 后面的操作。

```java
int a = 1;
int b = a + 1;
```

`a = 1` happens-before `b = a + 1`。

---

### ② volatile 规则

对一个 volatile 变量的写操作，happens-before 后续对这个变量的读操作。

```java
int data;
volatile boolean ready;

// 线程 A
data = 100;
ready = true;

// 线程 B
if (ready) {
    System.out.println(data);
}
```

因为：

```text
data = 100
    happens-before
ready = true

ready = true
    happens-before
读取 ready

读取 ready
    happens-before
读取 data
```

根据传递性：

```text
data = 100
    happens-before
线程 B 读取 data
```

因此，只要线程 B 读取到 `ready == true`，就能够看到线程 A 之前写入的 `data = 100`。

---

### ③ 锁规则

一个锁的解锁操作，happens-before 后续对同一个锁的加锁操作。

```java
synchronized (lock) {
    data = 100;
}
```

线程 A 释放锁后，线程 B 再获得同一把锁，可以看到线程 A 在临界区中的修改。

---

### ④ 线程启动规则

调用 `Thread.start()` 之前的操作，happens-before 新线程中的操作。

```java
int data = 100;

Thread thread = new Thread(() -> {
    System.out.println(data);
});

thread.start();
```

新线程能够看到 `data = 100`。

---

### ⑤ 线程终止规则

线程中的操作，happens-before 其他线程从 `join()` 成功返回。

```java
Thread thread = new Thread(() -> {
    data = 100;
});

thread.start();
thread.join();

System.out.println(data);
```

`join()` 返回后，主线程能够看到子线程的执行结果。

---

### ⑥ 传递性规则

如果：

```text
A happens-before B
B happens-before C
```

那么：

```text
A happens-before C
```

---

## 2.3 volatile 的底层实现

`volatile` 的底层原理可以概括为：

```text
volatile 写：Release 语义
volatile 读：Acquire 语义
```

### volatile 写

```java
data = 100;
ready = true; // volatile 写
```

volatile 写操作保证：

```text
它前面的普通读写操作
不能被重排序到 volatile 写之后
```

因此不能变成：

```java
ready = true;
data = 100;
```

同时，线程之前的修改会以符合 JMM 语义的方式对其他线程可见。

---

### volatile 读

```java
if (ready) { // volatile 读
    System.out.println(data);
}
```

volatile 读操作保证：

```text
它后面的普通读写操作
不能被重排序到 volatile 读之前
```

不能先读取 `data`，再判断 `ready`。

---

## 2.4 内存屏障

JVM 通常通过内存屏障限制编译器和 CPU 的重排序。

常见屏障可以理解为：

|屏障|作用|
|---|---|
|LoadLoad|前面的读完成后，才能执行后面的读|
|LoadStore|前面的读完成后，才能执行后面的写|
|StoreStore|前面的写完成后，才能执行后面的写|
|StoreLoad|前面的写完成并对其他处理器可见后，才能执行后面的读|

volatile 写通常具备类似：

```text
普通操作
StoreStore 屏障
volatile 写
StoreLoad 屏障
```

volatile 读通常具备类似：

```text
volatile 读
LoadLoad 屏障
LoadStore 屏障
后续普通操作
```

具体生成什么机器指令，与 JVM、JIT 编译器和 CPU 架构有关。

面试时不要绝对地说：

> volatile 一定生成某一条固定汇编指令。

更加准确的说法是：

> JVM 根据具体硬件架构，通过编译器屏障和 CPU 内存屏障，实现 volatile 的 Acquire/Release 语义。

---

## 2.5 volatile 为什么不能保证 count++ 原子性？

```java
volatile int count = 0;

count++;
```

虽然每次读取和写入 `count` 都具备 volatile 语义，但整个自增操作仍然可以交叉执行：

```text
线程 A volatile 读：count = 0
线程 B volatile 读：count = 0

线程 A 本地计算：0 + 1
线程 B 本地计算：0 + 1

线程 A volatile 写：count = 1
线程 B volatile 写：count = 1
```

因此最终结果仍然可能是 `1`。

volatile 只能保证：

```text
单次读能够看到符合 JMM 规则的最新写入
```

不能保证：

```text
读取 → 修改 → 写回
```

整个过程不可被其他线程插入。

---

# 3. 常见问题是什么？

## 3.1 用 volatile 修饰计数器

错误写法：

```java
private volatile int count;

public void increment() {
    count++;
}
```

问题：

```text
count++ 不是原子操作
```

应该使用：

```java
private final AtomicInteger count = new AtomicInteger();

public void increment() {
    count.incrementAndGet();
}
```

高并发统计场景可以使用：

```java
private final LongAdder count = new LongAdder();
```

---

## 3.2 volatile 引用不等于对象内部线程安全

```java
private volatile List<String> list = new ArrayList<>();
```

volatile 只保证：

```text
list 这个引用的读取和替换具备 volatile 语义
```

但不保证：

```java
list.add("A");
list.remove("B");
```

是线程安全的。

例如：

```java
private volatile List<String> list = new ArrayList<>();

public void add(String value) {
    list.add(value); // 仍然线程不安全
}
```

可以改为：

```java
private final List<String> list = new CopyOnWriteArrayList<>();
```

或者使用锁：

```java
public synchronized void add(String value) {
    list.add(value);
}
```

---

## 3.3 volatile 数组不保证数组元素具备 volatile 语义

```java
private volatile int[] array = new int[10];
```

这里 volatile 修饰的是 `array` 引用。

下面的操作：

```java
array[0]++;
```

仍然不是线程安全的。

可以使用：

```java
AtomicIntegerArray array = new AtomicIntegerArray(10);

array.incrementAndGet(0);
```

---

## 3.4 多个 volatile 变量无法维护整体不变式

例如账户有两个变量：

```java
private volatile int balance;
private volatile int frozenAmount;
```

业务要求：

```text
可用金额 = balance - frozenAmount
```

虽然两个变量分别可见，但线程可能读取到不一致的组合：

```text
旧 balance + 新 frozenAmount
新 balance + 旧 frozenAmount
```

如果多个变量必须整体保持一致，应使用：

```java
synchronized
Lock
不可变对象 + volatile 引用
```

例如：

```java
public final class AccountSnapshot {

    private final int balance;
    private final int frozenAmount;

    public AccountSnapshot(int balance, int frozenAmount) {
        this.balance = balance;
        this.frozenAmount = frozenAmount;
    }

    public int getBalance() {
        return balance;
    }

    public int getFrozenAmount() {
        return frozenAmount;
    }
}
```

通过 volatile 一次性替换整个快照：

```java
private volatile AccountSnapshot snapshot =
        new AccountSnapshot(0, 0);

public void update(int balance, int frozenAmount) {
    snapshot = new AccountSnapshot(balance, frozenAmount);
}
```

读取时只读取一次引用：

```java
AccountSnapshot current = snapshot;
```

这样可以保证读到的是一个完整快照。

---

## 3.5 误认为 volatile 等于 synchronized

两者能力不同：

|对比项|volatile|synchronized|
|---|---|---|
|可见性|支持|支持|
|有序性|支持特定约束|支持|
|原子性|仅单次读写|支持临界区原子性|
|是否阻塞|不阻塞|可能阻塞|
|适用场景|状态标记、快照发布|复合操作、共享资源修改|

不能简单认为：

```text
volatile 是 synchronized 的高性能替代品
```

volatile 的能力比 synchronized 弱，只适用于特定场景。

---

## 3.6 忙等待导致 CPU 占满

```java
while (!finished) {
}
```

即使 `finished` 是 volatile，这段代码也可能让一个 CPU 核心长期保持高负载。

可以使用：

```java
Thread.sleep()
LockSupport.park()
CountDownLatch
Condition
BlockingQueue
CompletableFuture
```

例如：

```java
CountDownLatch latch = new CountDownLatch(1);

// 工作线程完成
latch.countDown();

// 等待线程
latch.await();
```

---

## 3.7 在多写场景中使用 volatile

volatile 适合：

```text
一个或少量线程修改
多个线程读取
```

如果多个线程都需要根据旧值进行修改：

```java
value = value + 1;
```

volatile 不够，应考虑：

```text
AtomicInteger
AtomicLong
LongAdder
synchronized
ReentrantLock
```

---

# 4. 常见方案是什么？

可以根据问题选择对应工具。

|问题|常见方案|
|---|---|
|一个线程修改停止标记，多个线程读取|volatile|
|发布一个不可变配置快照|volatile 引用|
|简单计数器|AtomicInteger、AtomicLong|
|超高并发统计|LongAdder|
|多个变量需要整体一致|synchronized、Lock|
|多线程共享集合|ConcurrentHashMap 等|
|等待某个任务完成|CountDownLatch、Future|
|线程间传递任务|BlockingQueue|
|无锁更新一个对象引用|AtomicReference|
|读多写少的配置|不可变对象 + volatile 引用|
|复杂状态机|synchronized、Lock、CAS|

---

## 4.1 状态标记：volatile

```java
public class Worker {

    private volatile boolean running = true;

    public void run() {
        while (running) {
            doWork();
        }
    }

    public void stop() {
        running = false;
    }

    private void doWork() {
        // 业务逻辑
    }
}
```

适用条件：

```text
新值不依赖旧值
只有简单读取和赋值
不需要与其他变量共同保持一致
```

---

## 4.2 计数：AtomicInteger

```java
private final AtomicInteger count = new AtomicInteger();

public void increment() {
    count.incrementAndGet();
}
```

底层通常使用 CAS：

```text
读取旧值
计算新值
比较内存中的值是否仍然等于旧值
相等则更新
不相等则重试
```

---

## 4.3 高并发统计：LongAdder

```java
private final LongAdder requestCount = new LongAdder();

public void recordRequest() {
    requestCount.increment();
}

public long getCount() {
    return requestCount.sum();
}
```

它通过分散热点，降低多个线程竞争同一个变量的问题。

适合：

```text
接口调用次数
监控指标
访问量统计
成功失败次数
```

不适合要求每次读取都绝对精确、并且依赖旧值做业务判断的场景。

---

## 4.4 复合操作：synchronized

```java
private int balance;

public synchronized boolean deduct(int amount) {
    if (balance < amount) {
        return false;
    }

    balance -= amount;
    return true;
}
```

这里“判断余额”和“扣减余额”必须作为一个整体，不能只给 `balance` 加 volatile。

---

## 4.5 读多写少：不可变对象加 volatile 引用

```java
public final class SystemConfig {

    private final int timeout;
    private final int retryCount;

    public SystemConfig(int timeout, int retryCount) {
        this.timeout = timeout;
        this.retryCount = retryCount;
    }

    public int getTimeout() {
        return timeout;
    }

    public int getRetryCount() {
        return retryCount;
    }
}
```

配置持有类：

```java
public class ConfigManager {

    private volatile SystemConfig config =
            new SystemConfig(3000, 3);

    public SystemConfig getConfig() {
        return config;
    }

    public void refresh(SystemConfig newConfig) {
        config = newConfig;
    }
}
```

优势：

```text
读取不加锁
配置更新时整体替换
不会出现读取到一半新、一半旧的情况
```

---

# 5. 项目里怎么用？

## 5.1 服务优雅停止

例如后台消费线程：

```java
@Component
public class MessageConsumer {

    private volatile boolean running = true;

    public void consume() {
        while (running) {
            pollAndProcess();
        }
    }

    @PreDestroy
    public void shutdown() {
        running = false;
    }

    private void pollAndProcess() {
        // 拉取和处理消息
    }
}
```

但如果 `pollAndProcess()` 会长时间阻塞，仅设置 volatile 不够，还需要：

```text
中断线程
关闭连接
唤醒阻塞队列
设置拉取超时
```

---

## 5.2 动态配置刷新

例如 Nacos 配置更新后，替换本地配置快照：

```java
@Component
public class RuleManager {

    private volatile RuleConfig currentConfig =
            RuleConfig.defaultConfig();

    public RuleConfig getCurrentConfig() {
        return currentConfig;
    }

    public void refresh(RuleConfig newConfig) {
        currentConfig = newConfig;
    }
}
```

请求线程直接读取：

```java
RuleConfig config = ruleManager.getCurrentConfig();
```

适合：

```text
规则配置
限流参数
超时时间
功能开关
算法参数
设备阈值
```

最好让 `RuleConfig` 成为不可变对象。

---

## 5.3 本地功能开关

```java
private volatile boolean featureEnabled;

public void enable() {
    featureEnabled = true;
}

public void disable() {
    featureEnabled = false;
}

public void handleRequest() {
    if (featureEnabled) {
        executeNewLogic();
    } else {
        executeOldLogic();
    }
}
```

需要注意：

> volatile 只能保证单个 JVM 内线程之间的可见性，不能解决分布式节点之间的数据一致性。

多个微服务节点需要通过：

```text
Nacos
Apollo
Redis
配置中心推送
消息队列广播
```

同步配置。

---

## 5.4 双重检查单例

```java
public class Singleton {

    private static volatile Singleton instance;

    private Singleton() {
    }

    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton();
                }
            }
        }

        return instance;
    }
}
```

为什么必须加 volatile？

```java
instance = new Singleton();
```

可以抽象为：

```text
1. 分配对象内存
2. 初始化对象
3. 将 instance 指向对象内存
```

如果没有 volatile，步骤 2 和步骤 3 可能重排序：

```text
1. 分配内存
2. instance 指向内存
3. 初始化对象
```

另一个线程可能看到：

```text
instance != null
```

但对象尚未初始化完成。

volatile 禁止这种不安全发布。

不过项目中更推荐：

```java
public class Singleton {

    private Singleton() {
    }

    private static class Holder {
        private static final Singleton INSTANCE = new Singleton();
    }

    public static Singleton getInstance() {
        return Holder.INSTANCE;
    }
}
```

或者直接交给 Spring 容器管理。

---

## 5.5 缓存快照切换

例如加载一份设备规则：

```java
private volatile Map<String, DeviceRule> ruleCache =
        Collections.emptyMap();

public void reload() {
    Map<String, DeviceRule> newCache = loadFromDatabase();

    ruleCache = Collections.unmodifiableMap(newCache);
}

public DeviceRule getRule(String deviceCode) {
    return ruleCache.get(deviceCode);
}
```

这里不是在原 Map 上逐条修改，而是：

```text
1. 创建一份新的完整 Map
2. 加载全部数据
3. 使用 volatile 一次性替换引用
```

请求线程要么看到旧版本，要么看到新版本，不会看到只加载一半的数据。

这种模式也叫：

```text
Copy-On-Write
不可变快照
整体替换
```

---

## 5.6 主备状态、连接状态标记

```java
private volatile boolean master;
private volatile boolean connected;
private volatile ServiceState state;
```

适合表示：

```text
服务是否可用
连接是否建立
节点是否为主节点
任务是否关闭
配置是否已加载
```

但如果状态切换涉及复杂判断：

```text
INIT → STARTING → RUNNING → STOPPING → STOPPED
```

多个线程可能同时修改时，应考虑：

```java
AtomicReference<ServiceState>
```

例如：

```java
private final AtomicReference<ServiceState> state =
        new AtomicReference<>(ServiceState.INIT);

public boolean start() {
    return state.compareAndSet(
            ServiceState.INIT,
            ServiceState.STARTING
    );
}
```

---

# 6. 面试官可能追问什么？

## 6.1 volatile 能保证什么？

回答：

```text
1. 保证可见性
2. 保证 volatile 变量单次读写的原子性
3. 禁止特定类型的指令重排序
4. 不能保证复合操作的原子性
```

注意不要简单回答：

> volatile 不保证原子性。

更准确的说法是：

> volatile 变量的单次读取和单次写入具有原子性，但 `count++` 这样的复合操作不具有原子性。

---

## 6.2 volatile 和 synchronized 有什么区别？

回答：

```text
volatile 是轻量级的可见性、有序性工具，不会形成临界区；
synchronized 同时保证可见性、有序性和临界区内的原子性。

volatile 不会阻塞线程，适合状态标记和快照发布；
synchronized 可能阻塞线程，适合复合操作和共享资源保护。
```

---

## 6.3 volatile 为什么不能保证 count++ 安全？

因为：

```java
count++;
```

包含：

```text
读取 count
执行加一
写回 count
```

多个线程之间仍然可以交叉执行。

应使用：

```java
AtomicInteger.incrementAndGet()
```

或者：

```java
synchronized
```

---

## 6.4 volatile 如何保证可见性？

回答时分三层：

```text
JMM 层面：
volatile 写 happens-before 后续对同一个变量的 volatile 读。

JVM 层面：
通过编译器屏障限制 JIT 重排序。

硬件层面：
通过 CPU 内存屏障、缓存一致性等机制实现跨核心可见性。
```

不要只回答：

> 每次都从主内存读取。

这种说法适合入门理解，但不够严谨。JMM 只规定最终必须满足的语义，不要求每次真的绕过所有缓存访问物理主存。

---

## 6.5 什么是 happens-before？

回答：

> happens-before 是 JMM 判断两个操作之间是否存在可见性和顺序保证的规则。如果 A happens-before B，那么 A 的执行结果对 B 可见，并且从内存模型角度看，A 排在 B 之前。

常见规则：

```text
程序顺序规则
锁规则
volatile 规则
线程启动规则
线程终止规则
传递性规则
```

---

## 6.6 volatile 可以禁止所有重排序吗？

不能。

volatile 只禁止会破坏其内存语义的重排序。

例如：

```java
int a = 1;
int b = 2;
volatile boolean ready = true;
```

`a = 1` 和 `b = 2` 之间，如果互不依赖，仍然可能调整顺序。

但它们不能被移动到：

```java
ready = true;
```

之后。

---

## 6.7 volatile 修饰对象后，对象内部字段是否线程安全？

不安全。

```java
volatile User user;
```

只保证：

```text
user 引用的可见性和发布安全
```

不保证：

```java
user.setName()
user.setAge()
```

这些操作线程安全。

如果对象不可变，然后整体替换引用，则比较适合 volatile。

---

## 6.8 volatile 能保证立即可见吗？

更准确的回答是：

> volatile 提供的是内存模型上的可见性保证，而不是现实时间意义上的“多少纳秒内立即传播”。

当线程执行后续 volatile 读时，必须遵守 JMM 的可见性和同步顺序规则，但线程什么时候被 CPU 调度执行，不由 volatile 保证。

---

## 6.9 volatile 和 AtomicInteger 有什么区别？

|对比|volatile int|AtomicInteger|
|---|---|---|
|读取最新值|支持|支持|
|普通赋值|支持|支持|
|自增原子性|不支持|支持|
|CAS|不支持|支持|
|条件更新|不支持|支持|

例如：

```java
volatile int count;
```

不能安全执行：

```java
count++;
```

而：

```java
AtomicInteger count = new AtomicInteger();
count.incrementAndGet();
```

可以。

---

## 6.10 volatile 和 final 有什么关系？

`final` 也有特殊的内存语义。

只要对象没有在构造方法中发生逸出，并且被正确发布，其他线程看到对象引用时，应当能够看到构造方法中对 `final` 字段的初始化结果。

```java
public final class Config {

    private final int timeout;

    public Config(int timeout) {
        this.timeout = timeout;
    }
}
```

因此并发项目中经常组合使用：

```text
不可变对象
+
final 字段
+
volatile 引用
```

既保证对象内部状态固定，也保证新快照能够安全发布。

---

## 6.11 单例为什么必须使用 volatile？

因为：

```java
instance = new Singleton();
```

不是逻辑上的单一步骤，可能发生“引用已经发布，但对象还没初始化完成”的重排序。

volatile 保证：

```text
对象初始化完成
    happens-before
instance 引用对其他线程可见
```

---

## 6.12 volatile 的性能一定比 synchronized 好吗？

不一定。

volatile 的单次读写通常比竞争激烈的锁轻量，但它也会：

```text
限制编译器优化
限制 CPU 重排序
产生内存屏障开销
增加缓存一致性通信
```

如果变量被大量线程频繁写入，可能造成缓存行频繁失效。

而现代 JVM 对 synchronized 做了大量优化。选择并发工具应该以正确性和适用场景为准，而不是只看“有没有锁”。

---

# 最后总结

## JMM 的核心

JMM 主要规定：

```text
线程之间如何看到共享变量
哪些指令允许重排序
什么情况下操作具有可见性和顺序保证
```

核心关键词：

```text
可见性
原子性
有序性
happens-before
安全发布
```

## volatile 的核心

volatile 能够保证：

```text
可见性
特定的有序性
单次读取和写入的原子性
```

volatile 不能保证：

```text
复合操作原子性
多个变量整体一致性
对象内部操作线程安全
分布式节点之间的数据一致性
```

## 选择口诀

```text
状态标记用 volatile
简单计数用 AtomicInteger
高并发统计用 LongAdder
复合逻辑用 synchronized 或 Lock
读多写少用不可变快照 + volatile 引用
多字段一致性不要只靠 volatile
```

面试时可以用一句话收尾：

> JMM 定义了 Java 多线程程序的内存可见性和执行顺序规则；volatile 基于 JMM 的 happens-before 关系，通过 Acquire/Release 语义和内存屏障，保证变量的可见性并限制重排序，但不能保证读取、修改、写回这类复合操作的原子性。