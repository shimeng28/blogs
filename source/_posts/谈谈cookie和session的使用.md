---
title: 斐波那契数列的一次实验
date: 2019-04-15 15:35:10
tags:
---

记一次斐波那契数列的实现

斐波那契数列 1 1 2 3 5 8.....

普通的实现方法
```js
  function fib_normal(n) {
    if (n === 0) return 0;
    if (n <= 2) return 1;
    return fib_norma(n - 1) + fib_norma(n - 2);
  };
```

增加计算缓存的实现方法
```
  const fib_cache = (() => {
    const cache = {};
    return function _fib(n) {
      if (typeof cache[ n ] !== 'undefined') {
        return cache[ n ];
      }
      let result = 0;
      if (n === 0) result = 0;
      else if (n <= 2) result = 1;
      else result = _fib(n - 1) + _fib(n - 2);
      cache[ n ] = result;
 
      return result;
    };
  })();
```

为了表明两者的区别，可以看下每种需要执行的次数
因此修改一下代码
```
const fib_normal = (() => {
  let i = 0;
  return function _fib(n) {
    i++;
    console.log(i);
    if (n === 0) return 0;
    if (n <= 2) return 1;
    return _fib(n - 1) + _fib(n - 2);
  };
})();  
```

```
const fib_cache = (() => {
  const cache = {};
  let i = 0;
  return function _fib(n) {
    if (typeof cache[ n ] !== 'undefined') {
      return cache[ n ];
    }
    let result = 0;
    if (n === 0) result = 0;
    else if (n <= 2) result = 1;
    else result = _fib(n - 1) + _fib(n - 2);
    cache[ n ] = result;
    i++;
    console.log(i);
    return result;
  };
})();
```

因此可以看下结果 
 1. 当n=10时 fib_normal 执行了109次 fib_cache执行了10次
 2. 当n=20时 fib_normal 执行了13529次 fib_cache执行了20次
 3. 当n=30时 fib_normal 执行了1664079次 fib_cache执行了30次

 ......后面不用比了，反倒是最开始是他俩差距最小的时候







