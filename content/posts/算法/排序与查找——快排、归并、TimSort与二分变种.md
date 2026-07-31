---
title: "排序与查找——快排、归并、TimSort与二分变种"
date: 2026-07-31
description: 深入快排的pivot选择与双轴优化、归并排序与TimSort的run合并机制、二分查找的四种边界变种，给出Java/Python最优实现与JDK源码级的选型分析。
tags: ["算法","排序","快速排序","归并排序","TimSort","二分查找"]
categories: ["算法"]
---

# 排序与查找——快排、归并、TimSort 与二分变种

> Java `Arrays.sort()` 为什么基本类型用双轴快排而对象用 TimSort？Python `list.sort()` 为什么全是 TimSort？二分查找的四个变种怎么写才不会在边界条件上翻车？本文从底层原理到最优实现，讲清楚排序与查找在工程中真正需要的知识。

---

## 一、快速排序

### 1.1 核心原理

快排的递归骨架只有三步：

```
partition: 选一个 pivot → 把 < pivot 的放左边，> pivot 的放右边 → 返回 pivot 的最终位置
quicksort(left, pivot-1)  → 递归排左边
quicksort(pivot+1, right) → 递归排右边
```

**退化 O(n²) 的两种场景与应对**：

- **已排序数组 + 固定选最左为 pivot** → 每次 partition 只减少 1 个元素 → O(n²)。应对：**随机 pivot**（在 [l, r] 中随机选一个跟 l 交换）或**三数取中**（取 l、mid、r 三个值的中位数作为 pivot）
- **大量重复元素 + 简单二路划分** → 所有相等元素落在一侧 → 接近 O(n²)。应对：**三路快排**，把数组分为 `< pivot`、`= pivot`、`> pivot` 三部分，中间相等区间不再参与递归

### 1.2 双轴快排——JDK 对基本类型的排序选择

Java `Arrays.sort(int[])` 用的是 **DualPivotQuickSort**：

```
选择两个 pivot: p1 < p2
partition 分五段: < p1 | = p1 | p1 < x < p2 | = p2 | > p2
递归排三段: < p1、中间、> p2
```

双轴比单轴减少了比较次数。在元素分布接近均匀时，双轴每次比较可以同时缩小两个区间——效率接近单轴的 1.5 倍。JDK 还针对小数组（< 47）切到插入排序、对大数组加入优化的 pivot 选择策略。

**为什么基本类型用快排（不稳定），对象用 TimSort（稳定）？**

答案在于"排序结果是否依赖原始顺序"。基本类型（int/long/double）的两个相等值交换位置，没有任何语义差异——`3` 和 `3` 是无区别的。对象（Person/Order 等）的 `equals` 为 true 的元素不能随意交换——原始顺序可能蕴含业务语义（例如先创建的保留在前）。`Arrays.sort(Object[])` 的 Javadoc 明确承诺"stable"，所以必须用归并系列的稳定排序。

### 1.3 Java 最优实现

```java
// 基本类型排序
int[] arr = {3, 1, 4, 1, 5};
Arrays.sort(arr);              // DualPivotQuickSort，不稳定

// 部分排序（只排 [fromIndex, toIndex)）
Arrays.sort(arr, 0, 3);

// 并行排序（大数组，> 8192 元素时有优势）
Arrays.parallelSort(arr);

// 对象排序——稳定
Arrays.sort(users, Comparator.comparing(User::getAge).thenComparing(User::getName));

// 三路快排手写（应对大量重复元素——荷兰国旗问题）
void threeWaySort(int[] nums, int l, int r) {
    if (l >= r) return;
    int pivot = nums[l];
    int lt = l, i = l + 1, gt = r;   // [l,lt)<pivot; [lt,gt]=pivot; (gt,r]>pivot
    while (i <= gt) {
        if (nums[i] < pivot) swap(nums, lt++, i++);
        else if (nums[i] > pivot) swap(nums, i, gt--);
        else i++;
    }
    threeWaySort(nums, l, lt - 1);
    threeWaySort(nums, gt + 1, r);
}
```

### 1.4 Python 最优实现

```python
# Python 排序——总是 TimSort
arr = [3, 1, 4, 1, 5]
arr.sort()                      # 原地排序
sorted_arr = sorted(arr)        # 返回新列表

# 自定义排序
arr.sort(key=lambda x: x.age, reverse=True)

# Top K 快排 partition（O(n) 平均）
def find_kth_largest(nums, k):
    """基于快排 partition 找第 K 大——O(n) 平均，无需全排序"""
    def partition(l, r):
        pivot = nums[r]
        i = l
        for j in range(l, r):
            if nums[j] <= pivot:
                nums[i], nums[j] = nums[j], nums[i]
                i += 1
        nums[i], nums[r] = nums[r], nums[i]
        return i
    
    target = len(nums) - k   # 第 K 大 = 第 (n-k) 小
    l, r = 0, len(nums) - 1
    while l <= r:
        p = partition(l, r)
        if p == target:
            return nums[p]
        elif p < target:
            l = p + 1
        else:
            r = p - 1
    return -1
```

---

## 二、归并排序与 TimSort

### 2.1 归并排序原理

归并排序 = **分治 + 合并两个有序数组**。唯一一个任何情况下都是 O(n log n) 的排序算法，且**稳定**。

```
mergeSort(l, r):
  mid = (l + r) / 2
  mergeSort(l, mid)       # 排左半
  mergeSort(mid+1, r)     # 排右半
  merge(l, mid, r)        # 合并两个已排序的半区
```

代价是 O(n) 额外空间——合并时需要临时数组存放结果。

### 2.2 TimSort——工程中最精妙的排序算法

TimSort 是归并排序的超级增强版：**利用数据中已存在的有序片段（run）来加速**。

```
TimSort 的核心流程:
1. 扫描数组，识别自然有序的 run（minrun=32~64）
   - 如果 run 是降序 → 翻转成升序
   - 如果 run 太短 → 用二分插入扩展到 minrun
2. 把 run 压入栈，维护栈不变量（栈顶三个 run 的长度满足约束）
3. 违反不变量时合并栈顶的相邻 run
4. 全部 run 入栈后，依次合并栈中所有 run
```

**Google 在 Android 和 Python 都用了 TimSort**——因为真实数据很少是完全随机的。TimSort 在接近有序的数据上接近 O(n)（只需识别 run + 合并），而快排永远是 O(n log n)。

### 2.3 Java 实现

```java
// 归并排序手写
void mergeSort(int[] arr, int l, int r, int[] tmp) {
    if (l >= r) return;
    int mid = l + (r - l) / 2;
    mergeSort(arr, l, mid, tmp);
    mergeSort(arr, mid + 1, r, tmp);
    merge(arr, l, mid, r, tmp);
}

void merge(int[] arr, int l, int mid, int r, int[] tmp) {
    int i = l, j = mid + 1, k = l;
    while (i <= mid && j <= r) {
        tmp[k++] = arr[i] <= arr[j] ? arr[i++] : arr[j++];  // <= 保证稳定
    }
    while (i <= mid) tmp[k++] = arr[i++];
    while (j <= r)   tmp[k++] = arr[j++];
    System.arraycopy(tmp, l, arr, l, r - l + 1);
}
```

```java
// 外部排序——1TB 文件排序
// 1. 将大文件分成 N 个小块，每块可以一次读入内存
// 2. 每块单独排序后写入临时文件
// 3. 使用 PriorityQueue 做 N 路归并：
PriorityQueue<FileEntry> pq = new PriorityQueue<>();
for (BufferedReader reader : readers) {
    String line = reader.readLine();
    if (line != null) pq.offer(new FileEntry(line, reader));
}
while (!pq.isEmpty()) {
    FileEntry entry = pq.poll();
    writer.write(entry.line);
    String next = entry.reader.readLine();
    if (next != null) pq.offer(new FileEntry(next, entry.reader));
}
```

### 2.4 Python 实现

```python
def merge_sort(arr):
    if len(arr) <= 1:
        return arr
    mid = len(arr) // 2
    left = merge_sort(arr[:mid])
    right = merge_sort(arr[mid:])
    return merge(left, right)

def merge(left, right):
    result = []
    i = j = 0
    while i < len(left) and j < len(right):
        if left[i] <= right[j]:    # <= 保证稳定
            result.append(left[i])
            i += 1
        else:
            result.append(right[j])
            j += 1
    result.extend(left[i:])
    result.extend(right[j:])
    return result

# 简洁到极致的写法（非原地，O(n) 额外空间）
# 生产环境直接用 sorted() —— TimSort
```

### 2.5 实际场景

- **外部排序**：1TB 日志文件排序 → 拆成 1000 个小文件 → 每个排序 → K 路归并
- **数据库 ORDER BY**：MySQL `Using filesort` = 在 sort_buffer 中快排（如果数据小）或归并排序（外部文件排序）。`EXPLAIN` 中看到 `Using filesort` 不一定是坏事——索引已经避免全表扫描了
- **归并思想在数据库中的应用**：Nested Loop Join 的两个表如果都有序 → Merge Join O(n+m)

---

## 三、二分查找与四种变种

### 3.1 基础二分——`mid` 为什么这样写

```java
int binarySearch(int[] nums, int target) {
    int l = 0, r = nums.length - 1;
    while (l <= r) {                   // 闭区间 [l, r]
        int mid = l + (r - l) / 2;     // 防溢出，(l+r)/2 在 l+r > 2^31-1 时溢出
        if (nums[mid] == target) return mid;
        else if (nums[mid] < target) l = mid + 1;
        else r = mid - 1;
    }
    return -1;
}
```

**Python 内置的二分模块**：

```python
import bisect

arr = [1, 3, 4, 4, 7, 9]
# bisect_left:  第一个 >= x 的位置
bisect.bisect_left(arr, 4)   # → 2 (第一个 4)
# bisect_right: 第一个 > x 的位置
bisect.bisect_right(arr, 4)  # → 4 (最后一个 4 的后面)
# insort: O(n) 插入并保持有序
bisect.insort(arr, 5)        # arr = [1,3,4,4,5,7,9]
```

### 3.2 四种变种——一个模板搞定

四种变种本质都是缩小搜索区间的策略不同，关键在于 `l` 和 `r` 的语义以及何时 `+1`/`-1`：

```python
# 模板：while(l < r) + mid 偏向一边
# 区别在于 condition 和 mid 的计算方式

# 1. 第一个 == target（有重复时找最左）
def first_equal(nums, target):
    l, r = 0, len(nums) - 1
    while l < r:
        mid = l + (r - l) // 2      # 偏左
        if nums[mid] < target:
            l = mid + 1
        else:                        # nums[mid] >= target
            r = mid                  # 收缩右边界
    return l if nums[l] == target else -1

# 2. 最后一个 == target（有重复时找最右）
def last_equal(nums, target):
    l, r = 0, len(nums) - 1
    while l < r:
        mid = l + (r - l + 1) // 2  # 偏右！防止死循环
        if nums[mid] <= target:
            l = mid                  # 收缩左边界
        else:
            r = mid - 1
    return l if nums[l] == target else -1

# 3. 第一个 >= target（lower_bound）
def lower_bound(nums, target):
    l, r = 0, len(nums)
    while l < r:
        mid = l + (r - l) // 2
        if nums[mid] < target:
            l = mid + 1
        else:
            r = mid
    return l  # 返回值是插入位置

# 4. 第一个 > target（upper_bound）
def upper_bound(nums, target):
    l, r = 0, len(nums)
    while l < r:
        mid = l + (r - l) // 2
        if nums[mid] <= target:
            l = mid + 1
        else:
            r = mid
    return l
```

**防止死循环的关键**：`mid` 偏向哪边决定了 `l` 和 `r` 的更新方式。`mid = l + (r-l)//2` 偏向左边 → 当 `l` 和 `r` 相邻时 `mid = l` → 如果 `l = mid` 没有让区间缩小（因为 mid 已经等于 l），就会死循环。所以此时必须用 `l = mid + 1` 或让 `mid = l + (r-l+1)//2` 偏向右边。一句话：**mid 偏左则 l 必须 +1，mid 偏右则 r 必须 -1**。

### 3.3 Java `Arrays.binarySearch` 的返回值设计

```java
int idx = Arrays.binarySearch(arr, target);
if (idx >= 0) {
    // 找到：idx 是元素的索引
} else {
    // 没找到：返回 -(insertionPoint) - 1
    // 例: arr=[1,3,5], target=4 → idx=-3
    //      insertionPoint = -(-3) - 1 = 2
    //      意思是 4 应该插在索引 2 的位置
    int insertPos = -(idx + 1);
}
```

### 3.4 实际场景

- **InnoDB 页面内查找**：B+Tree 的每个节点页内二分查找定位 key → 找到子节点指针或具体行
- **配置中心版本查找**：`versions = [1.0, 2.0, 5.0]`，请求版本 3.0 → 二分找"最后一个 ≤ 3.0" → 返回 2.0
- **服务权重路由**：按权重区间分配请求 → 二分定位随机值落在哪个区间
- **IP 范围查找**：二分查找 IP 所属的 CIDR 段

---

## 四、排序算法选择决策树

```
数据量 < 47?
  → 插入排序（常量因子小，小数组最快）

数据接近有序？
  → TimSort（利用自然 run，接近 O(n)）

需要稳定性？
  → 是 → TimSort / 归并排序
  → 否 → 快排（DualPivotQuickSort）

数据有大量重复元素？
  → 三路快排（相等元素直接跳过，不等部分才递归）

数据完全在内存？
  → 是 → 上面前三种选一个
  → 否 → 外部排序（拆小文件 + 归并）
```

**工程铁律**：除非你要处理的数据有非常特殊的分布特征（如已知几乎有序），否则**三方库的默认排序就是最优选择**——JDK 的 `Arrays.sort` 和 Python 的 `sorted/timsort` 经过了 20+ 年的打磨，考虑了小数组、接近有序、重复元素等所有边界情况，比你手写的更快。
