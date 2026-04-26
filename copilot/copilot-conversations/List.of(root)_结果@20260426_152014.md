---
epoch: 1777188014047
modelKey: "gpt-5.4|openrouterai"
topic: "List.of(root) 结果"



tags:
  - copilot-conversation
---

**user**: root = [3,9,20,null,null,15,7], 对这个执行 List.of(root) 会咋样
[Context: Notes: 算法/11 102二叉树的层序遍历.md]
[Timestamp: 2026/04/26 15:20:14]

**ai**: `List.of(root)` 不会把整棵树“展开”。

它只是创建一个**长度为 1 的不可变 List**，里面唯一的元素就是 `root` 这个 `TreeNode` 对象本身。

也就是说：

- `root = [3,9,20,null,null,15,7]` 这是题目里描述二叉树的方式
- 真正在 Java 代码里，`root` 是一个 `TreeNode`
- 所以 `List.of(root)` 的结果相当于：

```java
cur = [ root节点 ]
```

也就是第一轮：

- `cur` 里只有一个节点：值为 `3` 的根节点
- 然后 `for(TreeNode node : cur)` 遍历时，只会遍历这个根节点
- 把 `3` 加进 `vals`
- 再把它的左右孩子 `9` 和 `20` 放进 `nxt`

所以执行过程大概是这样：

### 初始
```java
List<TreeNode> cur = List.of(root);
```

此时：
```java
cur = [3]
```

### 第一轮 while
- 遍历 `cur`
- `vals = [3]`
- `nxt = [9, 20]`
- `ans = [[3]]`
- `cur = nxt`

### 第二轮 while
此时：
```java
cur = [9, 20]
```

- 遍历 `9`、`20`
- `vals = [9, 20]`
- `nxt = [15, 7]`
- `ans = [[3], [9, 20]]`

### 第三轮 while
此时：
```java
cur = [15, 7]
```

- `vals = [15, 7]`
- `nxt = []`
- `ans = [[3], [9, 20], [15, 7]]`

最后返回：

```java
[[3], [9, 20], [15, 7]]
```

---

你这里最容易混淆的是：

- `[3,9,20,null,null,15,7]` 是**输入表示法**
- `root` 在代码里是**根节点对象**
- `List.of(root)` 是把这个**根节点对象放进一个 List 里**

---

你可以把这句理解成：

```java
List<TreeNode> cur = new ArrayList<>();
cur.add(root);
```

只是 `List.of(root)` 更简洁，而且创建的是**不可变列表**。

---

顺便提醒一下：

```java
List<TreeNode> cur = List.of(root);
```

这里 `cur` 一开始虽然指向不可变 List，但你后面并没有对这个 List 做 `add/remove`，而是直接：

```java
cur = nxt;
```

所以完全没问题。

如果你写成这样就会报错：

```java
cur.add(root.left);
```

因为 `List.of()` 返回的是不可变集合。

---

如果你愿意，我还能继续帮你把这道题里 `List.of(root)`、`new ArrayList<>()`、`ArrayDeque` 这几种写法的区别整理成适合记在 [[11 102二叉树的层序遍历]] 里的笔记。
[Timestamp: 2026/04/26 15:20:29]

**user**: Queue<TreeNode> queue = new LinkedList<>();
Queue<TreeNode> queue = new ArrayDeque<>(); 这个对吗
[Context: Notes: 算法/11 102二叉树的层序遍历.md]
[Timestamp: 2026/04/26 15:22:59]

**ai**: 对，这两种写法都对。

```java
Queue<TreeNode> queue = new LinkedList<>();
Queue<TreeNode> queue = new ArrayDeque<>();
```

它们都可以作为 `Queue` 来用，支持：

- `add()`
- `offer()`
- `poll()`
- `peek()`

在这道二叉树层序遍历里都能正常做 BFS。

---

### 1. `LinkedList` 可以当队列用
因为 `LinkedList` 实现了 `Queue` 接口，所以这样写没问题：

```java
Queue<TreeNode> queue = new LinkedList<>();
```

特点：

- 底层是链表
- 头删尾插方便
- 也允许存 `null`

---

### 2. `ArrayDeque` 也可以当队列用
```java
Queue<TreeNode> queue = new ArrayDeque<>();
```

特点：

- 底层是循环数组
- 作为队列/双端队列性能通常更好
- **不允许 `null`**

---

### 3. 这题更推荐 `ArrayDeque`
在 LeetCode 这种层序遍历题里，一般更推荐：

```java
Queue<TreeNode> queue = new ArrayDeque<>();
```

因为：

- 性能通常比 `LinkedList` 更好
- 更轻量
- Java 官方也更推荐它作为队列使用

所以实际写题时常见写法是：

```java
Queue<TreeNode> queue = new ArrayDeque<>();
queue.offer(root);
```

---

### 4. 面试里可以怎么说
如果面试官问你为什么不用 `LinkedList`，你可以答：

- `LinkedList` 和 `ArrayDeque` 都能实现队列
- 但在普通队列场景下，`ArrayDeque` 通常性能更好，开销更小
- 所以做 BFS 层序遍历时，一般优先用 `ArrayDeque`

---

### 5. 补一个小细节
虽然你写：

```java
queue.add(root);
TreeNode node = queue.poll();
```

也是对的，但更规范一点常写：

```java
queue.offer(root);
TreeNode node = queue.poll();
```

因为：

- `offer()` / `poll()` / `peek()` 是更标准的队列接口写法
- `add()` 在插入失败时可能抛异常
- `offer()` 更符合“入队”的语义

---

你可以记成一句话：

- `LinkedList` 能用
- `ArrayDeque` 更推荐
- BFS 常用 `Queue<TreeNode> queue = new ArrayDeque<>();`

如果你想，我可以顺手帮你把 [[11 102二叉树的层序遍历]] 这篇笔记补成“两个解法 + 队列知识点 + 面试表达”的整理版。
[Timestamp: 2026/04/26 15:23:14]