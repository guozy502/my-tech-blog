---
title: "ArrayList 源码解析——动态扩容与 fail-fast 机制"
date: 2026-07-27
description: 从 elementData 数组的 1.5 倍扩容策略与 C++/Rust 的差异、System.arraycopy 的浅拷贝行为、modCount 驱动的 fail-fast 迭代器在并发场景的局限、到 ArrayList vs LinkedList 的选型决策，拆解 ArrayList 的核心源码设计。
tags: ["Java","集合","ArrayList","源码","fail-fast","扩容"]
categories: ["Java集合"]
---

# 一、你在 foreach 里 remove，抛了 ConcurrentModificationException

```java
List<String> list = new ArrayList<>(Arrays.asList("a", "b", "c", "d"));
for (String s : list) {
    if (s.equals("b")) list.remove(s);  // 💀 ConcurrentModificationException
}
```

99% 的 Java 开发者都踩过这个坑。根因不是"并发修改"（虽然名字叫这个），而是 `modCount` 机制——ArrayList 用这个计数器防止**任何结构性修改**（包括单线程中的修改）在迭代期间不被察觉。

本文从底层结构、扩容策略、到 fail-fast 原理拆解 ArrayList。建议对照 [LinkedList](/posts/java集合/linkedlist-源码解析双向链表为何增删快查询慢/) 阅读——两篇合在一起给出的选型决策才完整。

---

# 二、底层结构——一个会变长的 Object[]

```java
public class ArrayList<E> extends AbstractList<E> {
    transient Object[] elementData;  // 真正存储元素的数组
    private int size;                 // 当前元素个数（注意：不是 elementData.length！）
    
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
    // 默认构造创建的空数组——不分配内存，懒加载
}
```

**`elementData.length` ≠ `size`**。数组可能更长（扩容后预留的空间），`size` 是实际存放的元素数。

**`transient` 修饰**：表示序列化时跳过此字段。ArrayList 自己实现了 `writeObject`/`readObject`，只序列化 `elementData[0..size-1]` 中的有效元素，而不是整个数组（避免序列化空槽位的 null）。

---

# 三、动态扩容——1.5 倍，不是 2 倍

## 3.1 扩容代码

```java
private Object[] grow(int minCapacity) {
    int oldCapacity = elementData.length;
    if (oldCapacity > 0 || elementData != DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        int newCapacity = ArraysSupport.newLength(oldCapacity,
            minCapacity - oldCapacity,   // 最少增长量
            oldCapacity >> 1);            // 优先增长量 = oldCapacity/2
        return elementData = Arrays.copyOf(elementData, newCapacity);
    } else {
        // 默认构造的空数组 → 首次分配
        return elementData = new Object[Math.max(DEFAULT_CAPACITY, minCapacity)];
    }
}
```

`newLength` 的逻辑：优先按 `oldCapacity + oldCapacity/2`（即 1.5 倍）扩容，如果还不够则直接扩到 `minCapacity`：

```mermaid
flowchart LR
    A["new ArrayList()\nelementData = {}"] -->|"第 1 次 add"| B["扩容到 10\nnew Object[10]"]
    B -->|"加到第 11 个"| C["扩容到 15"]
    C -->|"加到第 16 个"| D["扩容到 22"]
    D -->|"加到第 23 个"| E["扩容到 33"]
    
    style B fill:#fff3e0
```

## 3.2 为什么是 1.5 倍？

这是一个有趣的设计决策。不同语言/框架做了不同选择：

| 实现 | 扩容因子 | 理由 |
|------|---------|------|
| **Java ArrayList** | **1.5x** | 平衡内存浪费和扩容频率 |
| C++ `std::vector` | **2x** | 确保每次扩容的**摊还成本**是 O(1) |
| Rust `Vec` | **2x** | 和 C++ 一样，保证摊还 O(1) |
| Go `slice` | **2x**（小切片）→ **1.25x**（大切片） | 小切片快速成长，大切片控制内存 |
| Python `list` | **~1.125x** | 非常保守，内存优先 |

**1.5x 和 2x 的真正区别**：不是"扩容快慢"，而是**内存碎片化**。

- 2x 扩容时，每次扩容的新大小是之前所有已分配大小之和 + 1（16→32→64→128→256→...）。这意味着旧内存释放后无法被新分配复用——每次都需要一块更大的连续内存
- 1.5x 扩容时，旧内存释放后可以逐渐积累到足够大的连续空间，理论上可以复用之前的地址空间

**结论**：1.5x 是"减少扩容次数"（需要 1.5x 以上的因子）和"允许内存复用"（需要 2x 以下的因子）的折中点。

## 3.3 扩容的实际代价

```java
// Arrays.copyOf 底层调用 System.arraycopy——一个 native 方法
// 它是内存拷贝（memcpy/memmove），不是逐个元素赋值
// 对于 10 万个元素的数组 → ~0.8MB → 微秒级完成
// 对于 1000 万个元素的数组 → ~80MB → 毫秒级 → 可能引发 GC
```

如果预先知道数据量，**总是设置 `initialCapacity`**：

```java
// ❌ 默认构造 → 10→15→22→33→49→... → 20+ 次扩容
new ArrayList<>();

// ✅ 预设容量 → 0 次扩容
new ArrayList<>(10_000);
```

---

# 四、add/remove —— "O(1)" 的真相

| 操作 | 头部 | 中间 | 尾部 |
|------|------|------|------|
| `add(E)`（追加） | — | — | O(1) 摊还* |
| `add(int, E)`（插入） | O(N) | O(N) | O(1) |
| `get(int)` | O(1) | O(1) | O(1) |
| `set(int, E)` | O(1) | O(1) | O(1) |
| `remove(int)` | O(N) | O(N) | O(1) |
| `remove(Object)` | O(N) 查找 + O(N) 搬运 | | |

```java
// add(int, E)：中间插入 → System.arraycopy 后移所有元素
// 在 index=5 的位置插入 → 5 之后的所有元素 [5..size-1] 全部后移一位
System.arraycopy(elementData, index,
                 elementData, index + 1,
                 size - index);      // 一次内存批量拷贝
```

**关键认知**：`System.arraycopy` 虽然也是 O(N)，但它是 JVM 原生优化（向量化、CPU 缓存友好的连续内存操作），比手动 for 循环快一个数量级。100 万元素的 arraycopy 可能在几毫秒完成。

**`remove(Object)` 的陷阱**：

```java
// remove(Object) 先调用 equals 遍历查找 → O(N)
// 找到后再 arraycopy 搬运 → O(N)
// 最坏情况：元素在最后 → 遍历 N 次 equals + arraycopy(N)
// 不算慢（都是连续内存操作），但比 remove(int) 多了查找开销
```

---

# 五、fail-fast——`modCount` 的真相

## 5.1 它检测的是"结构性修改"，不是"并发"

```java
// ArrayList 的迭代器
private class Itr implements Iterator<E> {
    int expectedModCount = modCount;  // 迭代器创建时的"快照"
    
    public E next() {
        checkForComodification();  // 每次 next/remove 都检查
        // ...
    }
    
    final void checkForComodification() {
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
    }
}
```

**名为"并发修改异常"，但它不检测并发**——只检测 `modCount` 是否变了。在单线程中 `foreach` 循环里调 `list.remove()` 同样会触发。

## 5.2 结构性修改 vs 非结构性修改

```java
// 结构性修改（导致 modCount++）：
list.add(e);
list.remove(index);
list.add(index, e);

// 非结构性修改（不改变 modCount）：
list.set(index, value);  // ✅ 迭代期间可以 set——modCount 不变
```

## 5.3 如何在遍历时安全删除？

```java
// ✅ 方法 1：用迭代器的 remove()
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    if (it.next().equals("bad")) 
        it.remove();  // 同步更新 expectedModCount
}

// ✅ 方法 2：removeIf（Java 8+）
list.removeIf(s -> s.equals("bad"));

// ✅ 方法 3：倒序遍历（利用 remove(int) 只搬运尾部 + 下标移位）
for (int i = list.size() - 1; i >= 0; i--) {
    if (list.get(i).equals("bad")) 
        list.remove(i);  // 不影响前面未遍历的元素下标
}
```

---

# 六、ArrayList vs LinkedList——用数据说话

| | ArrayList | LinkedList |
|------|----------|-----------|
| **底层** | `Object[]` 连续内存 | `Node` 链表分散内存 |
| **get(i)** | **O(1)** | O(N)——需要遍历 |
| **add(E) 尾部** | O(1) 摊还 | O(1) |
| **add(int,E) 中间** | O(N) 但 `System.arraycopy`（快） | O(N) 但需要逐个遍历（慢） |
| **内存** | 连续，CPU Cache 友好 | 分散，Cache Miss 多 |
| **迭代** | 极快（连续内存） | 较慢（指针跳转） |
| **GC** | 一个大对象 | N 个小对象（GC 压力大） |

> **实践中，LinkedList 几乎没有比 ArrayList 快的场景。** 即使是在"头部插入"，`ArrayDeque` 也比 `LinkedList` 更快（环形数组复用空间）。LinkedList 的唯一优势在于：**作为 Deque/Queue 使用时，如果 java 版本小于 6（没有 ArrayDeque），它是唯一的选择**。现在请优先使用 `ArrayDeque`。

---

# 七、总结

| 特性 | 说明 |
|------|------|
| **底层** | `Object[]` 数组，懒加载 |
| **扩容** | 1.5 倍，`Arrays.copyOf → System.arraycopy` 批量迁移 |
| **随机访问** | O(1) |
| **增删中间** | O(N) 搬运（但 `arraycopy` 很快） |
| **线程安全** | 否。需要 `Collections.synchronizedList()` 或 `CopyOnWriteArrayList` |
| **fail-fast** | `modCount` 检测结构性修改，迭代期间 remove 抛 CME |
| **序列化** | 只序列化 `[0..size-1]` 的有效元素 |
