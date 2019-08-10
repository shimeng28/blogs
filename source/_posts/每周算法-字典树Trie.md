---
title: 每周算法---- 字典树Trie
date: 2019-08-10 17:53:49
tags: 每周算法
---
Trie 也叫字典树，读作Tree, 不过为了区分一般会在后面加一个E的音。其作为专门针对字符串的查询搜索，因为一般来说针对搜索可以使用二叉树，其搜索时间复杂度是logN，而如果使用Trie这种数据结构的话，其搜索的时间复杂度就和整个的查询的数量没有关系，而和查询的字符串长度相关，其时间复杂度为O(M),若M为查询的字符串长度。当然有得就有失，作为代价就是它的空间复杂度，和字符串中每个字符的类型数量相关。

### 字典树介绍

#### 定义

```js
class Node {
  constructor(isWord = false) {
    this.isWord = isWord;
    this.next = new Map();
  }
}

class Trie {
  constructor() {
    this.root = new Node();
    this.size = 0;
  }

  getSize() {
    return this.size;
  }
}
```
先定义一个Node类，作为每个节点的构造函数，isWord最为是否是字符串的最后一个字符节点，next指向下一个节点，它是一个Map值，是字符和Node节点的映射。之后定义的Trie类，包含两个属性，root根节点，size是节点的数量。那如何添加一个字符串呢？

#### 添加字符串

```js
add(word) {
  let curr = this.root;
  for (let i = 0; i < word.length; i++) {
    if (!curr.next.has(word[i])) {
      curr.next.set(word[i], new Node());
    }
    curr = curr.next.get(word[i]);
  }

  if (!curr.isWord) {
    curr.isWord = true;
    this.size++;
  }
}
```

根据添加的字符串的长度，依次将每个字符节点插入，之后在最后一个节点的时候给它添加isWord为true的属性。最终size自增。查询操作和添加操作类似。

#### 查询字符串

```js
search(word) {
  let curr = this.root;
  for (let i = 0; i < word.length; i++) {
    if (!curr.next.has(word[i])) {
      return false;
    }
    curr = curr.next.get(word[i]);
  }

  return curr.isWord;
}
```

通过遍历字符串，如果其中一个字符不存在，则直接返回false，否则看最后一个字符节点的isWord是否为true。那如果查询一个前缀是否存在呢？

#### 查询前缀

```js
isPrefix(prefix) {
  let curr = this.root;
  for (let i = 0; i < prefix.length; i++) {
    if (!curr.next.has(prefix[i])) {
      return false;
    }
    cur = cur.next.get(prefix[i]);
  }

  return true;
}
```
和查询类似，遍历prefix结束，即可认为包含该前缀。添加和查询功能都有了，那么删除功能呢？

#### 删除字符串

```js
remove(word) {
  let curr = this.root;
  const length = word.length;
  for (let i = 0; i < length; i++) {
    if (!curr.next.has(word[i])) {
      return false;
    }
    curr = curr.next.get(word[i]);
  }
  this.size--;
  return this._remove(word, length);
}

_remove(word, length) {
  if (!length) return true;

  let parent = this.root;
  let curr = this.root;

  for (let i = 0; i < length; i++) {
    parent = curr;
    curr = curr.next.get(word[i]);
  }

  if (curr.next.keys().length) {
    curr.isWord = false;
    return true;
  } else {
    parent.next.delete(word[length - 1]);
    return this._remove(word, length - 1);
  }
}
```

先判断是否包含有该字符串，之后执行删除操作，递归的判断最后一个节点是否有子节点，没有子节点则删除，否则将其isWord属性置为false。


### 算法题 简单的模式匹配

#### 定义

设计一个支持以下两种操作的数据结构：

```js
addWord(word)
search(word)
```
search(word) 可以搜索文字或正则表达式字符串，字符串只包含字母 . 或 a-z 。 . 可以表示任何一个字母。

#### 思路

通过之前Trie树的理解，这道题就是让实现一个字典树，但是不同的是search操作，.能匹配任何一个字符。因此可以理解为当匹配到'.'时，需要遍历该节点所在的子树，也就是树的遍历操作。


#### 代码

```js
class Node {
  constructor(isWord = false) {
    this.isWord = isWord;
    this.next = new Map();
  }
}

class WordDictionary {
  constructor() {
    this.root = new Node();
  }

  addWord(word) {
    let curr = this.root;
    for (let i = 0; i < word.length; i++) {
      if (!curr.next.has(word[i])) {
        curr.next.set(word[i], new Node());
      }

      curr = curr.next.get(word[i]);
    }

    curr.isWord = true;
  }

  match(node, word, index) {
    const length = word.length;
    if (length === index) {
      return node.isWord;
    }

    let curr = node;
    for (let i = index; i < length; i++) {
      const char = word[i];
      if (char === '.') {
        for (let nodeItem of curr.next.values()) {
          if(this.match(nodeItem, word, i+1)) {
            return true;
          }
        }
        return false;
      } else {
        if (!curr.next.has(char)) {
          return false;
        }

        curr = curr.next.get(char);
      }
    }

    return true;
  }

  search(word) {
    return this.match(this.root, word, 0);
  }
}
```

addWord和之前的添加操作一样，主要看下search，通过调用递归match方法，最终返回是否包含该值。

### 算法题 键值映射

#### 定义

实现一个 MapSum 类里的两个方法，insert 和 sum。

对于方法 insert，你将得到一对（字符串，整数）的键值对。字符串表示键，整数表示值。如果键已经存在，那么原来的键值对将被替代成新的键值对。

对于方法 sum，你将得到一个表示前缀的字符串，你需要返回所有以该前缀开头的键的值的总和。

```
输入: insert("apple", 3), 输出: Null
输入: sum("ap"), 输出: 3
输入: insert("app", 2), 输出: Null
输入: sum("ap"), 输出: 5
```

#### 思路

同样，这也是一道关于字典树的算法题，对于添加操作，没有任何的区别。对于sum操作，我们需要先遍历是否存在这样的前缀，之后再将包含该前缀的所有键值对的值相加。这也是树的遍历问题，只需要将表示前缀的节点的子树遍历一遍即可。看代码

#### 代码
```js
class Node {
  constructor(value = 0) {
    this.value = value;
    this.next = new Map();
  }
}

/**
 * Initialize your data structure here.
 */
class MapSum {
  constructor() {
    this.root = new Node();
  }

  /** 
   * @param {string} key 
   * @param {number} val
   * @return {void}
   */
  insert(key, val) {
    let curr = this.root;
    for (let i = 0; i < key.length; i++) {
      const char = key[i];
      if (!curr.next.has(char)) {
        curr.next.set(char, new Node());
      }

      curr = curr.next.get(char);
    }   
    curr.value = val;
  }

  /** 
   * @param {string} prefix
   * @return {number}
   */
  sum(prefix) {
    let curr = this.root;
    let result = 0;
    for (let i = 0; i < prefix.length; i++) {
      const char = prefix[i];
      if (!curr.next.has(char)) {
        return result;
      }
      curr = curr.next.get(char);
    } 
  
    result += this.match(curr);
    return result;
  }

  match(node) {
    let result = node.value;
    
    for (let nodeItem of node.next.values()) {
      result += this.match(nodeItem);
    }
  
    return result;
  }
};
```