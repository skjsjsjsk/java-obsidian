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