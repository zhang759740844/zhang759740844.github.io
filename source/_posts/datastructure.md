title: 数据结构与算法
date: 2017/7/31 14:07:12  
categories: 计算机
tags: 

 - 学习笔记
	
---

记录一些数据结构和算法的学习

<!--more-->

## 极客时间数据结构与算法之美—王争

### 数组

#### 为什么编程语言中数组多以 0 作为开始？

数组的特点是随机访问，定位一个数据需要通过数组首地址和偏移量来确定。如果数组以 0 开始，那么计算 a[k] 的内存地址的公式为：

```
a[k]_address = base_address + k * t
```

如果以 1 作为开始，那么公式就变为了：

```
a[k]_address = base_address + (k - 1) * t
```

会需要多一次的计算量。

### 排序

#### 如何找到数组总第k大的数

如果说先排序再直接过去第k大未免有些太慢了。可以参考快排的思路。选取最后一个数的值 x 作为标识，然后经过一次遍历把大于 x 的都放到 x 的右边，小于 x 的都放到 x 的左边。如果 x 所在位置大于 k，那么就对 x 左边继续之前的操作。否则对 x 右边进行之前的操作。这样就不需要像快排一样，每次都要对两边重复进行操作了。

时间复杂度为 O(n)，不过需要比较复杂的证明。

#### 归并排序的思想是什么？为什么计算机不使用归并排序？

归并排序是一种自下而上的排序，将两个分别排好序的队列 merge 为一个排好序的队列，不断重复，直到生成最终排好序的队列。

归并排序并不是一种原地排序，在 merge 的时候是需要申请额外的存储地址的，空间复杂度为 O(n)。因此，没有快排应用广泛。

#### 什么是桶排序，时间复杂度是多少，适用场景是什么？

桶排序的核心是将排序的数据分到有序的几个桶中，每个桶内的数据再单独进行快排。排完序后再把数据依次取出，组成有序序列。

时间复杂度为 O(n)

适用于外部排序。即数据存储在外部磁盘中，数据量较大，内存有限，无法将数据全部加载到内存中。（所以就先把外部数据先分到一个个桶中，然后将桶加载到内存中排序。）

### 跳表

#### 什么是跳表，时间复杂度是多少？

有序链表如何能像数组一样快速找到第 n 个节点？增加一些索引节点，使时间复杂度从 O(n) 减少到 O(logn)

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/tiaobiao.png?raw=true)

### 二叉树

#### 什么是二叉查找树

二叉查找树要求树中的任意一个节点，其左子树中的每个节点的值都要小于这个节点，而右子树的值都要大于这个节点。

#### 红黑树有什么用

**红黑树是一种不完全平衡的二叉查找树**，红黑树的高度也就是红黑树的查找时间复杂度为 logn。

### 堆

#### 什么是堆

- 堆是一个完全二叉树
- 堆中的每一个节点的值都必须大于等于或小于等于子树中每个节点的值

#### 堆排序的时间复杂度

堆排序的时间复杂度是 nlogn。堆排序总共分两步：

第一步是建堆，就是把数组中的元素以满足堆的方式排列。办法就是将数据一个一个的插入堆中。

第二步是排序，依次将堆顶的元素取出，然后将其左右子重新构建堆。

#### 什么是大顶堆什么是小顶堆

每个节点都大于等于左右子树的堆叫大顶堆，反之则是小顶堆

#### 堆的应用场景

堆主要用在**不需要所有数全排列**的情况，比如主要找到最大，或者最大的几个：

- 优先级队列
- 求 Top k 和求中位数

不需要全排序的就可以用一个 k 的大顶堆或者小顶堆实现，好处是避免了排列其他不用的数据产生的时间复杂度。

#### 什么是优先级队列

优先级队列就是按照优先级大小排好序的队列。

优先级队列可以是直接排好序的数组。但一般来说，我们只要保证每次出队列的是优先级最高的元素即可。

因此优先级队列是使用堆的最适宜的场景，即每次求出堆中最大的元素。

### 回溯法

#### 回溯法的核心思想

回溯法的思想不是纯粹的for循环，而是用 for 循环确定当前情况，然后运用递归，确定之后的情况。在确定当前情况的时候可以利用剪枝去除明显错误的情况。

例如八皇后问题，for循环确定当前列的位置，再判断当然选择是否符合规则，然后递归后面的列执行相同的操作。

### 动态规划

动态规划的核心是存在**重复子问题**。

当存在重复子问题的时候可以尝试找到**状态转移方程**。当找到状态转移方程，问题基本就能解决了。不过还要注意**递归终止条件**



### 最短路径算法

#### Dijkstra 算法

第一步，找出与起点 O 相邻的点，设定这些点的权值是它们各自与起点 O 之间的距离。找出权值最小的点 a，Oa 就是最短路径。第二步，找出与点 a 相邻的所有点，这些点的权值等于点 a 的权值加上这些点到点 a 的距离。在这两步中获得的所有有权重的点中，找到权重最小的点 b。点 b 可能直接和起点 O 相连，那么 Ob 就是其最短路径。点 b 也可能与点 a 相连，那么 Oab 就是 b 到起点 O 的最短路径。如此反复，直到所有点都被遍历完。

迪科斯彻算法是 BFS 的拓展。它是一种贪心算法的实现。每次找距离起点最近的点。

迪科斯彻算法每次获得一个距离起点最近的点，不能存在负权值。只有在权值为正的情况下，`a 到 O 权值最小，所以不存在任意点 x 使得 ax+xO < aO，即 aO 是最短路径` 这一假设才能够成立

迪科斯彻算法是路由选择协议 ospf 的解决方式

### 利用位图判重

使用 hashmap 的判重将会占用比较大的内存空间。比如对一个文本先进行 hash，得到一个 32 位的 hash 值，将会占用 4 个字节。当数据量非常大的时候将占用极大的内存空间。

利用位图来判重，将使 32 位的占用量减少为 1 位。但是信息量的减少必然会造成大量的误判，如何解决这一问题？可以使用布隆过滤器

#### 布隆过滤器

布隆过滤器要求我们使用 k 个哈希函数，对同一个值求哈希值，将会得到 k 个不同的哈希值 X1 X2 X3 … Xn。我们把这 k 个数字作为位图的下标，将对应的 BitMap[X1],BitMap[x2],BitMap[x3]… 都设置为 true。这样就用 k 个二进制位，表示一个值的存在。

当要查询某个值是否存在的时候，使用同样的 k 个哈希函数，对这个数字求哈希，然后在位图中相应的位置查看是否都为 true。如果都是 true，那么说明这个值存在，如果存在一个为 false，那么就说明这个数字不存在。

布隆过滤器其实也存在误判(误判某个数存在)，但是几率比较低

## 解题技巧

### 线性表

- 对数组排序

如果**题目中是一个无序数组**，那么通常可以先将数组进行排序。如果对时间复杂度有要求，可以使用 hashmap

- 二分查找

对于一个**已经排了序的数组**，可以通过二分法来降低时间复杂度

- 从中间向两边

匹配的通用做法。例如求最大连续序列，最长回文，最长有序括号。

- 双指针夹逼

例如找到序列中和为特定值的两个数

- 快慢指针

例如是否有环等问题

### 堆

#### 找最大的 k 个数

使用堆比每次都排序快

### 树

#### 枚举所有的可能

当题干中出现枚举所有可能的时候，要快速转换为树的形式，通过递归实现。

#### 是否存在某种可能

使用 DFS 或者 BFS 完成。一定要立刻反应到用树实现。

### 动态规划

递归+备忘录和动态规划的差别在于，递归是通过递推公式，从结果往条件的反向递推。而动态规划是基于 for 循环的正向迭代。

#### 核心

核心是能写出状态转移方程，使之能转为求解一个个子问题。

在写状态转移方程的时候要考虑，当前问题需要多少个子问题共同参与求解。比如爬楼梯问题，每次只能爬一层或者两层，那么每个问题就只需要两个子问题参与。再比如求最长上升子序列，那么当前位置的最长上升子序列，就和之前的所有位置的最长上升子序列相关。再比如买卖股票的问题，求当前天的最大收益，就不只是求之前每一天的最大收益，还要分别计入每一天的买进卖出状态。