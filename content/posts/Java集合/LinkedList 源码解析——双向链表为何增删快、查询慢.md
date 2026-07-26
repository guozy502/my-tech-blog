---
title: "LinkedList 源码解析——双向链表为何增删快、查询慢"
date: 2026-07-27
description: 从 Node 的双向指针结构、linkFirst/linkLast 的头尾 O(1) 增删、node(index) 的折半遍历优化、到为什么实践中 ArrayList 几乎总是比 LinkedList 更快（CPU Cache + GC 开销的碾压），拆解 LinkedList 的真实性能画像。
tags: ["Java","集合","LinkedList","双向链表","源码","Deque"]
categories: ["Java集合"]
---

# 一、"增删快、查询慢"——一个需要纠正的面试答案

几乎所有 Java 面试都这样教：

> ArrayList 查询 O(1) 增删 O(N)，LinkedList 查询 O(N) 增删 O(1)

**这个答案在一半场景下是错的。** LinkedList 的"增删 O(1)"只在头尾成立。在中间插入，你得先遍历 N/2 个节点找到位置（O(N)），然后才是 O(1) 的指针操作。而 ArrayList 的中间增删虽然是 O(N)，`System.arraycopy` 的批量内存搬运比逐个遍历节点**快得多**。

本文拆解 LinkedList 的源码，并给出实践中正确的选型决策。建议对照 [ArrayList](/posts/java集合/arraylist-源码解析动态扩容与-fail-fast-机制/) 阅读。

---

# 二、双向链表的 Node 结构

```java
public class LinkedList<E> extends AbstractSequentialList<E>
    implements List<E>, Deque<E> {  // ← 实现了 Deque！
    
    transient Node<E> first;  // 链表头
    transient Node<E> last;   // 链表尾
    transient int size;
    
    private static class Node<E> {
        E item;
        Node<E> next;
        Node<E> prev;
    }
}
```

**LinkedList 实现了 `Deque` 接口**——这其实是它最重要的用途。作为双端队列，它提供了 `addFirst`/`addLast`/`removeFirst`/`removeLast` 等方法。

---

# 三、头尾操作——真正的 O(1)

## 3.1 头部插入（linkFirst）

```java
private void linkFirst(E e) {
    final Node<E> f = first;
    final Node<E> newNode = new Node<>(null, e, f);
    first = newNode;
    if (f == null) last = newNode;  // 原来是空链表 → last 也指向新节点
    else f.prev = newNode;          // 旧 first 的 prev 指向新节点
    size++;
}
```

**仅 4 次指针赋值**： `newNode.next=f`、`f.prev=newNode`、`first=newNode`、`size++`。真正的 O(1)。

## 3.2 尾部追加（linkLast）

```java
void linkLast(E e) {
    final Node<E> l = last;
    final Node<E> newNode = new Node<>(l, e, null);
    last = newNode;
    if (l == null) first = newNode;
    else l.next = newNode;
    size++;
}
```

同样 O(1)——这就是 LinkedList "增删快"的唯一成立场景。

## 3.3 头部删除（unlinkFirst）

```java
private E unlinkFirst(Node<E> f) {
    final E element = f.item;
    final Node<E> next = f.next;
    f.item = null;      // 断开引用，help GC
    f.next = null;      // 断开引用，help GC
    first = next;       // 下一个成为新的 first
    if (next == null) last = null;  // 链表变空
    else next.prev = null;
    size--;
    return element;
}
```

---

# 四、中间操作——"先找后改"，O(N)

## 4.1 node(index)——折半优化的遍历

```java
Node<E> node(int index) {
    if (index < (size >> 1)) {   // 前半段 → 从 first 往后找
        Node<E> x = first;
        for (int i = 0; i < index; i++)
            x = x.next;          // 逐个跳指针！
        return x;
    } else {                     // 后半段 → 从 last 往前找
        Node<E> x = last;
        for (int i = size - 1; i > index; i--)
            x = x.prev;
        return x;
    }
}
```

**最多遍历 size/2 个节点**。这是 LinkedList 各种操作的瓶颈——`get(index)`、`add(int, E)`、`remove(int)` 都依赖 `node(index)` 先定位。

## 4.2 中间插入——先找到位置，再 O(1) 插

```java
public void add(int index, E element) {
    checkPositionIndex(index);
    if (index == size)
        linkLast(element);          // 尾部 → O(1)
    else
        linkBefore(element, node(index));  // 中间 → O(N) + O(1)
}
```

**"O(1)"只指指针操作本身**——但找到位置花的时间是 O(N)。

---

# 五、为什么在实践中 ArrayList 总是更快？

## 5.1 内存布局——CPU Cache 的碾压

```
ArrayList 的元素： [A][B][C][D][E][F][G][H]...  ← 一块连续内存
LinkedList 的元素： [A|next|prev] → [B|next|prev] → [C|next|prev] → ... ← 分散在堆中

当 CPU 从内存加载 ArrayList 的一个元素时：
  → 同时加载了相邻的 64 字节（一个 Cache Line）
  → 包含接下来的 8-16 个引用 → 遍历极快

当 CPU 加载 LinkedList 的一个节点时：
  → 加载了节点数据和 next/prev 指针
  → next 指向另一个随机的堆地址 → Cache Miss → 下一次加载又要从主存读
```

## 5.2 System.arraycopy vs 指针跳转

```java
// ArrayList 中部插入：System.arraycopy 后移 N 个元素
// → JVM 原生方法 → 可能用 SIMD 向量化 → 数微秒完成

// LinkedList 中部插入：node(index) 遍历 N/2 个节点
// → 每个节点一次指针跳转 → 可能 Cache Miss → 数十微秒
```

**即使 `add(int, E)` 在 ArrayList 中的 arraycopy 搬运了 10 万个元素，也可能比遍历 LinkedList 的 5 万个节点快**——因为连续内存操作 vs 随机指针跳转。

## 5.3 GC 行为

```
ArrayList: 1 个大数组 → GC 扫描时 1 个对象
LinkedList: N 个 Node 对象 + N 个元素引用 → GC 扫描时 N 个对象

对于 10 万元素的 List：
  ArrayList → 1 个数组对象（加上 10 万个元素对象本身）
  LinkedList → 10 万个 Node 对象 → GC 标记/整理时明显更慢
```

## 5.4 真正应该用 LinkedList 的场景

| 场景 | 用 LinkedList？ |
|------|----------------|
| 大量 `add(0, e)` 头部插入 | ✅（但 `ArrayDeque` 更好） |
| 需要双向遍历 | ✅（LinkedList 是双向链表） |
| 需要 `Deque` 接口但 JDK < 6 | ✅（没有 ArrayDeque） |
| 在头部/尾部频繁增删 | ✅（但 ArrayDeque 同样好且更省内存） |
| `get(i)` 频繁 | ❌ ArrayList |
| 中部插入/删除 | ❌ ArrayList（arraycopy 比遍历节点快） |
| 遍历所有元素 | ❌ ArrayList（CPU Cache 友好） |
| 通用 List | ❌ ArrayList |

---

# 六、作为 Deque 的使用——这才是 LinkedList 的正确用法

```java
// LinkedList 实现了 Deque<E> 接口
// 作为双端队列的头尾操作都是 O(1)
Deque<String> deque = new LinkedList<>();

// 头部操作
deque.addFirst("A");    // 插入头部
deque.removeFirst();    // 移除头部
deque.peekFirst();      // 查看头部

// 尾部操作
deque.addLast("Z");     // 插入尾部（等价于 add）
deque.removeLast();     // 移除尾部
deque.peekLast();       // 查看尾部

// 栈操作（LIFO）
deque.push("X");        // addFirst
deque.pop();            // removeFirst

// 队列操作（FIFO）
deque.offer("Y");       // addLast
deque.poll();           // removeFirst
```

**注意**：Java 6 引入了 `ArrayDeque`（环形数组实现的双端队列），在绝大多数场景下比 `LinkedList` 更适合当 Deque——性能更好、内存更省。LinkedList 仅在需要**同时**使用 List 和 Deque 接口（双接口需求）时才优先。

---

# 七、总结

| 特性 | 说明 |
|------|------|
| **底层** | 双向链表 `Node`（item + next + prev） |
| **头尾增删** | O(1)——真正的"增删快" |
| **中间增删** | O(N) 定位 + O(1) 操作 |
| **随机访问** | O(N)——`node(index)` 折半遍历 |
| **内存** | 分散存储，Cache 不友好，N 个 Node 对象 → GC 压力大 |
| **正确用法** | 作为 **Deque**（双端队列），而非通用 List |
| **替代品** | `ArrayDeque`（队列/栈首选）、`ArrayList`（通用 List 首选） |

**一句话总结**：除非你需要 Java 1.2 兼容性或同时需要 List 和 Deque 接口，**永远优先用 ArrayList（List）或 ArrayDeque（Deque）**。
