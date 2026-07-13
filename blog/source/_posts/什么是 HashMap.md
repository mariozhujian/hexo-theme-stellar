---
title: 什么是 HashMap 和 ConcurrentHashMap
date: 2026-07-07 16:43:00
categories:
- 技术
- 面试

cover: https://69zvfy2e8b.ucarecd.net/90aee3db-e29f-4e79-839f-5de4f632e0d0/hashmap.png
---


下面可以直接当成 **高级 Java 面试复习模板** 来背。
核心一句话：

> **HashMap 解决单线程下快速查找问题；ConcurrentHashMap 解决多线程下安全、高性能地共享 Map 问题。**

---

# 一、HashMap

## 1. 它解决什么问题？

HashMap 解决的是：

> **根据 key 快速存取 value 的问题。**

比如：

```java
Map<Long, User> userMap = new HashMap<>();
userMap.put(1001L, user);
User user = userMap.get(1001L);
```

它适合：

| 场景      | 说明                  |
| ------- | ------------------- |
| 单线程数据缓存 | 方法内部临时缓存            |
| 快速查找    | 根据 id、code、type 找对象 |
| 数据分组    | 按状态、类型、日期分组         |
| 去重辅助    | 用 key 判断是否已存在       |
| 策略映射    | 根据类型找到对应处理器         |

它的核心优势是：

> **查询、插入平均时间复杂度是 O(1)。**

但是注意：
**HashMap 不是线程安全的。**

---

## 2. 底层原理是什么？

JDK 8 之后，HashMap 底层结构是：

```text
数组 + 链表 + 红黑树
```

可以理解成：

```text
HashMap
 └── table 数组
      ├── bucket[0]
      ├── bucket[1] -> Node -> Node
      ├── bucket[2]
      └── bucket[3] -> 红黑树
```

### put 流程

```java
map.put("name", "张三");
```

大概流程是：

```text
1. 计算 key 的 hash 值
2. 根据 hash 找到数组下标
3. 如果位置为空，直接放入
4. 如果位置不为空：
   - key 相同，覆盖 value
   - key 不同，放到链表后面
5. 链表过长，可能转成红黑树
6. 元素数量超过阈值，触发扩容
```

### get 流程

```java
map.get("name");
```

大概流程是：

```text
1. 计算 key 的 hash 值
2. 定位数组下标
3. 如果 bucket 第一个节点就是目标，直接返回
4. 否则遍历链表或红黑树
5. 找到 key 相等的节点，返回 value
```

### 为什么数组长度必须是 2 的幂？

HashMap 定位数组下标用的是：

```java
index = (n - 1) & hash;
```

如果数组长度是 2 的幂，`n - 1` 的二进制低位全是 1，可以让 hash 分布更均匀，同时位运算比取模 `%` 更快。

例如：

```text
数组长度 n = 16
n - 1 = 15
二进制：1111

index = hash & 15
```

### 为什么需要红黑树？

链表太长时，查询会退化成：

```text
O(n)
```

为了提升性能，JDK 8 引入红黑树。

关键阈值：

| 参数                        | 含义                |
| ------------------------- | ----------------- |
| TREEIFY_THRESHOLD = 8     | 链表长度大于等于 8，可能树化   |
| UNTREEIFY_THRESHOLD = 6   | 红黑树节点少于等于 6，退化为链表 |
| MIN_TREEIFY_CAPACITY = 64 | 数组长度至少 64，才会树化    |

也就是说：

> 链表长度达到 8 时，不一定马上树化。
> 如果数组长度小于 64，HashMap 会优先扩容，而不是树化。

---

## 3. 常见问题是什么？

### 问题一：Hash 冲突

不同 key 可能算出相同数组下标。

```text
key1 -> index 3
key2 -> index 3
key3 -> index 3
```

冲突多了之后，bucket 里的链表会变长，查询性能下降。

---

### 问题二：扩容性能开销

HashMap 默认容量是 16，默认负载因子是 0.75。

```text
阈值 = 容量 * 负载因子
16 * 0.75 = 12
```

当元素数量超过 12，就会扩容。

扩容时会：

```text
1. 创建新数组
2. 容量变成原来的 2 倍
3. 重新分配原来的节点位置
```

扩容成本比较高，所以如果一开始知道数据量，最好提前设置容量。

---

### 问题三：线程不安全

HashMap 在多线程下可能出现：

| 问题       | 说明                                       |
| -------- | ---------------------------------------- |
| 数据覆盖     | 多个线程同时 put，数据丢失                          |
| 数据不可见    | 一个线程 put，另一个线程未必马上看到                     |
| size 不准确 | 并发修改导致 size 异常                           |
| 遍历异常     | 遍历时修改可能抛 ConcurrentModificationException |

JDK 7 中 HashMap 并发扩容还可能出现链表环，导致 CPU 飙高。
JDK 8 改进了扩容方式，但它依然不是线程安全的。

---

### 问题四：key 的 equals 和 hashCode 写错

HashMap 判断 key 是否相等，依赖两个方法：

```java
hashCode()
equals()
```

规则是：

```text
两个对象 equals 相等，hashCode 必须相等。
两个对象 hashCode 相等，equals 不一定相等。
```

如果只重写 equals，不重写 hashCode，就可能出现：

```java
map.put(user1, "A");
map.get(user2); // 明明 user1 和 user2 逻辑相等，但取不到
```

---

### 问题五：使用可变对象作为 key

比如：

```java
User user = new User(1, "张三");
map.put(user, "data");

user.setId(2);

map.get(user); // 可能取不到
```

因为 key 的字段变了，hashCode 也可能变了，导致定位不到原来的 bucket。

---

## 4. 常见方案是什么？

### 方案一：合理设置初始容量

如果预计要放 1000 个元素，不建议直接：

```java
new HashMap<>();
```

可以设置容量：

```java
Map<String, Object> map = new HashMap<>(2048);
```

为什么不是 1000？

因为 HashMap 有负载因子 0.75。

大概容量可以这样估：

```text
初始容量 = 预计元素数量 / 0.75 + 1
```

1000 个元素：

```text
1000 / 0.75 ≈ 1334
```

HashMap 会调整到最近的 2 的幂，所以最终可能是 2048。

---

### 方案二：key 使用不可变对象

推荐 key 用：

```java
String
Long
Integer
Enum
```

不推荐用频繁变化的业务对象作为 key。

---

### 方案三：重写 equals 和 hashCode

实体对象作为 key 时，一定要保证：

```java
equals 和 hashCode 逻辑一致
```

比如根据 userId 判断唯一性：

```java
@Override
public boolean equals(Object obj) {
    if (this == obj) return true;
    if (!(obj instanceof User)) return false;
    User user = (User) obj;
    return Objects.equals(this.userId, user.userId);
}

@Override
public int hashCode() {
    return Objects.hash(userId);
}
```

---

### 方案四：多线程场景不要用 HashMap

多线程共享 Map，优先使用：

```java
ConcurrentHashMap
```

而不是：

```java
HashMap
Collections.synchronizedMap(new HashMap<>())
Hashtable
```

---

## 5. 项目里怎么用？

### 场景一：按 id 快速查找数据

比如从数据库查出用户列表后，转成 Map：

```java
List<User> users = userMapper.selectList();

Map<Long, User> userMap = users.stream()
        .collect(Collectors.toMap(User::getId, Function.identity()));
```

后续查询：

```java
User user = userMap.get(userId);
```

比每次遍历 List 快很多。

---

### 场景二：数据分组

比如按订单状态分组：

```java
Map<Integer, List<Order>> orderMap = orders.stream()
        .collect(Collectors.groupingBy(Order::getStatus));
```

结果类似：

```text
1 -> 待支付订单列表
2 -> 已支付订单列表
3 -> 已取消订单列表
```

---

### 场景三：策略模式

项目里经常会根据类型选择不同处理器：

```java
Map<String, PayHandler> handlerMap = new HashMap<>();

handlerMap.put("ALI_PAY", aliPayHandler);
handlerMap.put("WECHAT_PAY", wechatPayHandler);
handlerMap.put("BANK_CARD", bankCardPayHandler);
```

使用时：

```java
PayHandler handler = handlerMap.get(payType);
handler.pay(order);
```

在 Spring 项目中，也可以结合 Bean 自动装配成策略 Map。

---

### 场景四：接口返回数据组装

比如订单里有 userId，需要补充用户名称：

```java
List<Long> userIds = orders.stream()
        .map(Order::getUserId)
        .distinct()
        .collect(Collectors.toList());

List<User> users = userMapper.selectByIds(userIds);

Map<Long, User> userMap = users.stream()
        .collect(Collectors.toMap(User::getId, Function.identity()));

for (Order order : orders) {
    User user = userMap.get(order.getUserId());
    order.setUserName(user.getName());
}
```

这类场景非常常见：

```text
订单列表 + 用户信息
设备列表 + 产线信息
告警列表 + 字典信息
权限列表 + 菜单信息
```

---

## 6. 面试官可能追问什么？

### 追问 1：HashMap 为什么线程不安全？

可以这样答：

> HashMap 的 put、resize、size++ 等操作都不是原子操作，也没有加锁或 volatile 保证可见性。多线程同时修改时，可能出现数据覆盖、size 不准确、遍历异常等问题，所以 HashMap 不适合多线程共享写场景。

---

### 追问 2：HashMap 为什么容量是 2 的幂？

可以这样答：

> 因为 HashMap 通过 `(n - 1) & hash` 计算数组下标。容量是 2 的幂时，`n - 1` 的二进制低位都是 1，可以让 hash 分布更均匀，同时位运算比取模更快。

---

### 追问 3：HashMap 什么时候扩容？

可以这样答：

> 当元素数量超过 `capacity * loadFactor` 时触发扩容。默认容量是 16，默认负载因子是 0.75，所以默认阈值是 12。扩容后容量变为原来的 2 倍。

---

### 追问 4：HashMap 什么时候树化？

可以这样答：

> 当某个 bucket 中链表长度达到 8，并且数组容量至少为 64 时，链表会转成红黑树。如果数组容量小于 64，会优先扩容，而不是树化。

---

### 追问 5：HashMap 和 Hashtable 有什么区别？

| 对比             | HashMap | Hashtable |
| -------------- | ------- | --------- |
| 线程安全           | 不安全     | 安全        |
| 性能             | 较高      | 较低        |
| null key/value | 允许      | 不允许       |
| 出现时间           | 较新      | 较老        |
| 推荐程度           | 常用      | 基本不推荐     |

Hashtable 是老类，方法基本都加了 synchronized，性能较差。

---

# 二、ConcurrentHashMap

## 1. 它解决什么问题？

ConcurrentHashMap 解决的是：

> **多线程环境下，安全、高效地读写 Map 的问题。**

它适合：

| 场景      | 说明                  |
| ------- | ------------------- |
| 多线程共享缓存 | 多线程读写本地缓存           |
| 幂等控制    | 防止重复提交、重复处理         |
| 在线用户状态  | 保存用户连接、session      |
| 任务状态管理  | 记录任务执行状态            |
| 统计计数    | 配合 LongAdder 做高并发统计 |
| 策略注册表   | 初始化后读多写少            |

它相比 HashMap 最大的区别是：

```text
HashMap：单线程高效
ConcurrentHashMap：多线程安全且高效
```

---

## 2. 底层原理是什么？

ConcurrentHashMap 在不同 JDK 版本里实现不同。

### JDK 7：Segment 分段锁

JDK 7 结构大概是：

```text
ConcurrentHashMap
 └── Segment[]
      ├── Segment 1 -> HashEntry[]
      ├── Segment 2 -> HashEntry[]
      └── Segment 3 -> HashEntry[]
```

每个 Segment 类似一个小 HashMap，并且每个 Segment 有自己的锁。

好处是：

```text
不同 Segment 可以并发写
```

比如：

```text
线程 A 写 Segment 1
线程 B 写 Segment 2
```

两者互不影响。

这叫：

```text
分段锁
```

---

### JDK 8：CAS + synchronized + volatile

JDK 8 取消了 Segment 分段锁，底层结构更接近 HashMap：

```text
数组 + 链表 + 红黑树
```

但是它通过下面机制保证线程安全：

| 机制             | 作用               |
| -------------- | ---------------- |
| CAS            | 空 bucket 插入时无锁更新 |
| synchronized   | bucket 冲突时锁住头节点  |
| volatile       | 保证数组和节点的可见性      |
| ForwardingNode | 扩容时标记迁移状态        |
| sizeCtl        | 控制初始化和扩容         |
| CounterCell    | 高并发统计 size       |

### put 流程

```java
map.put(key, value);
```

大概流程：

```text
1. 判断 key/value 是否为空，为空直接抛异常
2. 计算 hash
3. 如果 table 没初始化，先初始化
4. 根据 hash 定位 bucket
5. 如果 bucket 为空，用 CAS 插入
6. 如果 bucket 正在扩容，当前线程帮忙扩容
7. 如果 bucket 不为空，synchronized 锁住 bucket 头节点
8. 插入链表或红黑树
9. 判断是否需要树化或扩容
```

注意这里锁的粒度很小：

```text
不是锁整个 Map，而是锁某个 bucket
```

所以性能比 Hashtable 好很多。

---

### get 为什么不用加锁？

ConcurrentHashMap 的 get 通常不加锁。

原因是：

```text
1. table 使用 volatile 保证可见性
2. Node 的 value、next 等关键字段有 volatile 修饰
3. 读操作只需要根据 hash 定位并遍历，不修改结构
```

所以 get 可以高并发执行。

这也是 ConcurrentHashMap 性能好的重要原因。

---

## 3. 常见问题是什么？

### 问题一：不允许 null key 和 null value

ConcurrentHashMap 不允许：

```java
map.put(null, "A");
map.put("A", null);
```

否则会抛：

```java
NullPointerException
```

为什么？

因为多线程下，null 会带来歧义。

例如：

```java
map.get("A") == null
```

你无法判断：

```text
1. key 不存在
2. key 存在，但 value 就是 null
```

在并发环境下，这个判断会更复杂，所以 ConcurrentHashMap 直接禁止 null。

---

### 问题二：复合操作不是线程安全的

虽然单个操作是线程安全的：

```java
put()
get()
remove()
```

但下面这种组合操作不一定安全：

```java
if (!map.containsKey(key)) {
    map.put(key, value);
}
```

两个线程可能同时进入 if，然后都 put。

应该使用：

```java
map.putIfAbsent(key, value);
```

或者：

```java
map.computeIfAbsent(key, k -> value);
```

---

### 问题三：size 不一定适合强一致判断

ConcurrentHashMap 在高并发修改时，`size()` 得到的是一个尽量准确的值，但不适合做严格并发控制。

不推荐：

```java
if (map.size() == 0) {
    // 判断没有任务
}
```

高并发下，刚判断完，其他线程可能就 put/remove 了。

如果要判断是否为空，优先：

```java
map.isEmpty()
```

但即便如此，在并发环境下也要理解它只是当前瞬间状态。

---

### 问题四：computeIfAbsent 使用不当

比如：

```java
map.computeIfAbsent(key, k -> {
    // 里面做远程调用、查数据库、复杂计算
    return queryFromRemote(k);
});
```

这个函数执行期间，可能会影响同一个 bucket 上其他线程的操作。

所以不建议在 `computeIfAbsent` 里面做太重的逻辑。

更稳的做法是：

```java
Value value = map.get(key);
if (value == null) {
    Value newValue = queryFromDb(key);
    Value oldValue = map.putIfAbsent(key, newValue);
    value = oldValue == null ? newValue : oldValue;
}
```

---

### 问题五：value 本身不一定线程安全

ConcurrentHashMap 只能保证 Map 结构安全，不代表 value 对象内部也安全。

比如：

```java
ConcurrentHashMap<String, List<String>> map = new ConcurrentHashMap<>();
```

这里 Map 是线程安全的，但 `List` 不是线程安全的。

多个线程同时修改这个 List，依然可能有问题。

可以改成：

```java
ConcurrentHashMap<String, CopyOnWriteArrayList<String>> map = new ConcurrentHashMap<>();
```

或者使用同步控制。

---

## 4. 常见方案是什么？

### 方案一：用 putIfAbsent 解决并发初始化

```java
ConcurrentHashMap<String, Object> cache = new ConcurrentHashMap<>();

Object old = cache.putIfAbsent(key, value);
if (old == null) {
    // 当前线程放入成功
} else {
    // 已经有其他线程放入了
}
```

适合：

```text
缓存初始化
幂等处理
任务注册
```

---

### 方案二：用 computeIfAbsent 简化缓存逻辑

```java
User user = userCache.computeIfAbsent(userId, id -> userMapper.selectById(id));
```

含义是：

```text
如果 userId 不存在，就查询数据库并放入缓存；
如果已经存在，直接返回已有值。
```

不过刚才说了，里面不要写特别重、特别慢、可能阻塞很久的逻辑。

---

### 方案三：高并发计数使用 LongAdder

不要这样：

```java
ConcurrentHashMap<String, Integer> countMap = new ConcurrentHashMap<>();

countMap.put(key, countMap.getOrDefault(key, 0) + 1);
```

这不是线程安全的复合操作。

可以用：

```java
ConcurrentHashMap<String, LongAdder> countMap = new ConcurrentHashMap<>();

countMap.computeIfAbsent(key, k -> new LongAdder()).increment();
```

读取：

```java
long count = countMap.get(key).sum();
```

这个在高并发统计里非常常见。

---

### 方案四：本地缓存要考虑过期和容量

ConcurrentHashMap 经常被拿来做本地缓存：

```java
private final ConcurrentHashMap<String, User> userCache = new ConcurrentHashMap<>();
```

但是它没有：

```text
过期时间
最大容量
自动淘汰
统计命中率
```

如果是正式缓存，推荐：

```text
Caffeine
Guava Cache
Redis
```

简单场景可以用 ConcurrentHashMap，复杂缓存别硬造轮子。

---

## 5. 项目里怎么用？

### 场景一：本地缓存

比如缓存设备信息：

```java
private final ConcurrentHashMap<Long, DeviceInfo> deviceCache = new ConcurrentHashMap<>();

public DeviceInfo getDevice(Long deviceId) {
    return deviceCache.computeIfAbsent(deviceId, id -> deviceMapper.selectById(id));
}
```

适合读多写少的场景。

例如你项目里的：

```text
设备信息
产线配置
字典数据
告警规则
模型算子配置
```

都可以用这种思路做本地缓存。

---

### 场景二：防重复提交 / 幂等控制

比如防止同一个任务重复执行：

```java
private final ConcurrentHashMap<String, Boolean> runningTaskMap = new ConcurrentHashMap<>();

public void execute(String taskId) {
    Boolean old = runningTaskMap.putIfAbsent(taskId, true);
    if (old != null) {
        throw new RuntimeException("任务正在执行，请勿重复提交");
    }

    try {
        doExecute(taskId);
    } finally {
        runningTaskMap.remove(taskId);
    }
}
```

这类代码在项目里很实用。

比如：

```text
防止重复导入
防止重复计算
防止同一个设备重复下发任务
防止同一批次重复处理
```

---

### 场景三：统计接口调用次数

```java
private final ConcurrentHashMap<String, LongAdder> apiCountMap = new ConcurrentHashMap<>();

public void record(String apiName) {
    apiCountMap.computeIfAbsent(apiName, k -> new LongAdder()).increment();
}
```

可以统计：

```text
接口访问次数
设备告警次数
模型调用次数
用户操作次数
```

---

### 场景四：WebSocket 连接管理

```java
private final ConcurrentHashMap<Long, Session> sessionMap = new ConcurrentHashMap<>();

public void onOpen(Long userId, Session session) {
    sessionMap.put(userId, session);
}

public void onClose(Long userId) {
    sessionMap.remove(userId);
}

public void sendMsg(Long userId, String msg) {
    Session session = sessionMap.get(userId);
    if (session != null) {
        session.getBasicRemote().sendText(msg);
    }
}
```

这就是典型的多线程共享 Map 场景。

---

## 6. 面试官可能追问什么？

### 追问 1：ConcurrentHashMap 和 Hashtable 有什么区别？

| 对比             | ConcurrentHashMap | Hashtable |
| -------------- | ----------------- | --------- |
| 锁粒度            | bucket 级别         | 整个方法加锁    |
| 性能             | 高                 | 低         |
| null key/value | 不允许               | 不允许       |
| 并发能力           | 强                 | 弱         |
| 是否推荐           | 推荐                | 不推荐       |

可以这样回答：

> Hashtable 是整个方法加 synchronized，锁粒度太大；ConcurrentHashMap 通过 CAS、synchronized 锁桶节点、volatile 等机制降低锁粒度，所以并发性能更好。

---

### 追问 2：ConcurrentHashMap JDK 7 和 JDK 8 有什么区别？

| 版本    | 实现方式                                    |
| ----- | --------------------------------------- |
| JDK 7 | Segment 分段锁                             |
| JDK 8 | CAS + synchronized + Node 数组 + 链表 + 红黑树 |

可以这样回答：

> JDK 7 使用 Segment 分段锁，每个 Segment 类似一个小 HashMap。JDK 8 取消了 Segment，改成数组 + 链表 + 红黑树结构，插入空桶时用 CAS，发生冲突时用 synchronized 锁住桶头节点，锁粒度更细。

---

### 追问 3：ConcurrentHashMap 的 get 为什么不加锁？

可以这样回答：

> 因为 ConcurrentHashMap 中 table、Node 的 value 和 next 等字段通过 volatile 保证可见性。get 操作只读不改结构，所以一般不需要加锁，直接根据 hash 定位 bucket 后遍历即可。

---

### 追问 4：ConcurrentHashMap 为什么不允许 null？

可以这样回答：

> 为了避免并发环境下的语义歧义。如果 get 返回 null，无法判断是 key 不存在，还是 key 存在但 value 为 null。为了让并发判断更明确，ConcurrentHashMap 直接禁止 null key 和 null value。

---

### 追问 5：ConcurrentHashMap 一定线程安全吗？

这个问题很容易被坑。

正确回答：

> ConcurrentHashMap 保证的是 Map 自身结构的线程安全，比如 put、get、remove 这些单个操作是线程安全的。但如果是多个操作组合，比如 containsKey 后 put，就不是原子的。另外，如果 value 本身是非线程安全对象，比如 ArrayList，也需要额外保证 value 内部的线程安全。

---

### 追问 6：ConcurrentHashMap 扩容时怎么保证线程安全？

可以这样回答：

> ConcurrentHashMap 扩容时不是单线程独占完成，而是多个线程可以协助迁移数据。迁移中的 bucket 会被设置成 ForwardingNode，其他线程访问到这个节点时，就知道当前正在扩容，要么帮助迁移，要么去新表查找。通过 sizeCtl、CAS、ForwardingNode 等机制协调扩容过程。

---

# 三、HashMap 和 ConcurrentHashMap 对比总结

| 对比项        | HashMap       | ConcurrentHashMap |
| ---------- | ------------- | ----------------- |
| 线程安全       | 不安全           | 安全                |
| 底层结构       | 数组 + 链表 + 红黑树 | 数组 + 链表 + 红黑树     |
| null key   | 允许一个 null key | 不允许               |
| null value | 允许            | 不允许               |
| 适用场景       | 单线程、本地临时数据    | 多线程共享数据           |
| 性能         | 单线程性能高        | 并发性能高             |
| 扩容         | 普通 resize     | 多线程协助扩容           |
| 遍历         | fail-fast     | 弱一致性              |
| 典型用途       | 数据组装、分组、临时缓存  | 本地并发缓存、幂等控制、状态管理  |

---

# 四、面试时可以这样整体回答

面试官问：**HashMap 和 ConcurrentHashMap 的区别？**

你可以这样答：

> HashMap 是非线程安全的 Map，适合单线程场景，底层是数组、链表和红黑树。它通过 key 的 hash 值定位数组下标，冲突时使用链表或红黑树解决，默认负载因子是 0.75，超过阈值会扩容。
>
> ConcurrentHashMap 是线程安全的 Map，适合多线程共享场景。JDK 7 使用 Segment 分段锁，JDK 8 改成 CAS + synchronized + volatile 的方式。put 时如果 bucket 为空，会通过 CAS 插入；如果发生冲突，会锁住 bucket 头节点；get 通常不加锁，通过 volatile 保证可见性。
>
> 另外，HashMap 允许 null key 和 null value，而 ConcurrentHashMap 不允许。ConcurrentHashMap 只能保证单个 Map 操作线程安全，如果是 containsKey 后 put 这种复合操作，还需要使用 putIfAbsent、computeIfAbsent 等原子方法。

---

# 五、最容易被问的源码细节

## HashMap 高频源码点

| 问题        | 答案                  |
| --------- | ------------------- |
| 默认容量      | 16                  |
| 默认负载因子    | 0.75                |
| 为什么 2 的幂  | 方便 `(n - 1) & hash` |
| 链表多长树化    | 8                   |
| 树节点少于多少退化 | 6                   |
| 最小树化容量    | 64                  |
| 是否线程安全    | 否                   |
| 是否允许 null | 允许                  |

---

## ConcurrentHashMap 高频源码点

| 问题             | 答案                 |
| -------------- | ------------------ |
| JDK 7 实现       | Segment 分段锁        |
| JDK 8 实现       | CAS + synchronized |
| get 是否加锁       | 通常不加锁              |
| 是否允许 null      | 不允许                |
| 空桶插入           | CAS                |
| 冲突插入           | synchronized 锁桶头节点 |
| 扩容方式           | 多线程协助扩容            |
| 复合操作是否安全       | 不一定，要用原子方法         |
| value 是否自动线程安全 | 不是                 |

---

# 六、项目表达模板

面试里讲项目时，可以这样说：

> 在项目中，我一般会根据线程场景选择 Map。如果只是方法内部做数据组装、列表转 Map、字典映射、按 id 快速查询，我会使用 HashMap，因为它简单高效。比如订单列表需要补充用户信息时，我会先批量查询用户，再转成 `Map<Long, User>`，避免循环里反复查数据库。
>
> 如果这个 Map 会被多个线程共享，比如本地缓存、任务执行状态、防重复提交、WebSocket session 管理，我会使用 ConcurrentHashMap。比如我做过任务防重复执行，使用 `putIfAbsent(taskId, true)` 保证同一个任务同一时间只能有一个线程执行，任务结束后在 finally 中 remove，避免异常导致状态无法清理。
>
> 对于高并发计数，我不会用 `get + put`，而是使用 `ConcurrentHashMap<String, LongAdder>`，通过 `computeIfAbsent` 初始化计数器，然后调用 `increment`，这样并发性能更好。

---

# 七、最后记忆版

```text
HashMap：
解决单线程快速查找问题。
底层是数组 + 链表 + 红黑树。
默认容量 16，负载因子 0.75。
链表长度 8 且数组容量 >= 64 时树化。
线程不安全，允许 null。

ConcurrentHashMap：
解决多线程安全 Map 问题。
JDK7 是 Segment 分段锁。
JDK8 是 CAS + synchronized + volatile。
get 通常不加锁，put 空桶用 CAS，冲突锁桶头。
不允许 null。
单个操作安全，复合操作要用 putIfAbsent、compute、merge。
```

面试里最稳的一句话：

> **HashMap 关注单线程性能，ConcurrentHashMap 关注并发安全和并发性能；前者没有锁，后者通过 CAS、volatile 和细粒度 synchronized 保证线程安全。**
