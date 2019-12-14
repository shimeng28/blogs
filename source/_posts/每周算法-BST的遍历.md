---
title: 每周算法-BST的遍历
date: 2019-12-14 20:21:02
tags: 每周算法
---
BST的遍历，这是一个相对来说比较简单的问题，前序遍历就是每次在第一次访问该节点的时候做操作，而中序遍历可以理解从左孩子节点递归回溯过来的时候做操作，后序遍历同样能够理解为从右孩子节点递归回溯回来的时候做操作。针对整个二叉搜索树的遍历来说，可以理解为访问每个节点都访问了三次，在这三个时机的操作就是前中后的遍历。

当我们向下面用递归实现的时候
```js
const preOrder = (root) => {
  if (!root) return null;

  console.log(root);
  preOrder(root.left);
  preOrder(root.right);
};

const inOrder = (root) => {
  if (!root) return null;

  inOrder(root.left);
  console.log(root.val);
  inOrder(root.right);
};

const postOrder = (root) => {
  if (!root) return null;

  postOrder(root.left);
  console.log(root.val);
  postOrder(root.right);
};
```

可以看出来非常的简单，那么如果是非递归遍历呢？先看前序遍历

```js
const preOrder = (root) => {
  const result = [];
  if (!root) return result;
  const stack = [];
  stack.push(root);
  while (stack.length) {
    const currNode = stack.pop();
    if (!currNode) {
      continue;
    }
    result.push(currNode.val);
    if (currNode.right) {
      stack.push(right);
    }
    if (currNode.left) {
      stack.push(left);
    }
  }

  return result;
};
```
通过一个栈来深度遍历整颗BST，需要注意的是入栈的时候右孩子节点先入栈，最终返回遍历的顺序。接下来看下中序遍历

```js
const inOrder = (root) => {
  const result = [];
  if (!root) return result;
  const stack = [];
  while (root || stack.length) {
    while (root) {
      stack.push(root);
      root = root.left;
    } 
    root = stack.pop();
    result.push(root.val);
    root = root.right;
  }
  return result;
};
```

这是中序遍历的非递归代码，我的理解就是中序遍历会有一个强烈的左倾的渴望，除非万不得已绝不回头，这就叫做不撞南墙不回头，当然一旦回头来说明就是要出结果的时候了。所以说浪子回头金不换(ps: 此处纯属胡扯。。。)。回到正体，不过说实在的，中序遍历的功能是非常强大的，因为它返回的数据具有顺序性，因此我们可以直接求解给定一组数据，它在BST中的上界和下界，某个元素的排名以及查找第k大小的元素。

下面继续看下后序遍历, 根据后序遍历的定义我们可以写出这样的代码

```js
const postOrder = (root) => {
  const result = [];
  if (!root) {
    return result;
  }
  const visitedSet = new Set();
  const stack = [];
  stack.push(root);

  while (stack.length) {
    const currNode = stack[stack.length - 1];
    let leftVisited = true;
    let rightVisited = true;

    if (currNode.right && !visitedSet.has(currNode.right)) {
      rightVisited = false;
      stack.push(currNode.right);
    }

    if (currNode.left && !visitedSet.has(currNode.left)) {
      leftVisited = false;
      stack.push(currNode.left);
    }

    if (leftVisited && rightVisited) {
      result.push(currNode.val);
      visitedSet.add(currNode);
      stack.pop();
    }
  }

  return result;
};
```

左右子树都访问过了，我们才访问该节点，这里就需要一个Set集合。那么还有其他的实现方式吗？有的，那就是在输出结果的时候，将节点的结果从头部插入。具体看下代码来理解

```js
const postOrder = (root) => {
  const result = [];
  if (!root) {
    return result;
  }

  const stack = [];
  stack.push(root);

  while (stack.length) {
    const currNode = stack.pop();
    result.unshift(currNode.val);

    if (currNode.left) {
      stack.push(currNode.left);
    }

    if (currNode.right) {
      stack.push(currNode.right);
    }
  }

  return result;
};
```

以上便是针对BST的遍历，理解起来，各种算法还是非常好玩的。