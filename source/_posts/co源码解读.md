---
title: co源码解读
date: 2019-06-15 21:54:58
tags: JavaScript
---
在JS中，状态控制的有三种，分别是Generator、Promise、Async/Await。co库是专门将Generator以promise的形式输出的，核心代码非常简洁.

首先的需要明白的是，Generator是将通过调用next将执行内部的yield状态，而Promise是通过调用resolve和reject来改变该promise的状态。

```js
function* testGen() {
  console.log(111);
  let result = yield 123;
  console.log(123);
}

const testProm = new Promise((resolve, reject) => {
  setTimeout(() => {
    resolve(456);
  }, 300);
});
```

那么如何将两个看着差别如此之大的两种状态机从Generator转换为Promise呢？接下看它们之间的调用的区别

```js
const testGenDemo = testGen();
testGenDemo.next();

testProm.then((res) => console.log(res));
```

因此我们发现，想要转换Generator，需要不停的迭代它的next方法，而为了转化为Promise，那么转换方法必须应该返回的是一个Promise。

```js
function co(gen) {
  const ctx = this;
  const args = Array.prototype.slice.call(arguments, 1);
  return new Promise((resolve, reject) => { 
  });
}
```

那接下来呢？我们需要先判断传入的gen方法是一个GeneratorFunction还是一个Generator

```js
function co(gen) {
  const ctx = this;
  const args = Array.prototype.slice.call(arguments, 1);
  return new Promise((resolve, reject) => {
     if (typeof gen === 'function') {
        gen = gen.apply(ctx, args);
     }
  });
}
```
通过这种判断，那么我们现在能够肯定gen一定是一个Generator实例了，如果不是，那就是使用者写错了。其实对于我们来说，gen的零界条件就是它要不是一个GeneratorFunction要不就是一个Generator实例。

针对Generator的部分暂时先告一段落，那么针对Promise，我们要什么时候调用resolve和reject啊？这就是一个值得思考的问题了，对于resolve和reject我们又不能直接调用，那就将它们封装一层，这样我们才能控制住Promise的状态。当前其实我们发现现在最主要做的先开始执行Generator内部的代码，那也就是说要先执行
```js 
gen.next() 
```
。那我们就将这段代码写到对resolve的封装里面吧

```js
function co(gen) {
  const ctx = this;
  const args = Array.prototype.slice.call(arguments, 1);
  return new Promise((resolve, reject) => {
     if (typeof gen === 'function') {
        gen = gen.apply(ctx, args);
     }
     
     function onFullFilled(res) {
       let ret;
       try {
        ret = gen.next(res);
       } catch(e) {
         return reject(e);
       }
     }
     
     function onRejected(res) {
       let ret;
       try {
         ret = gen.throw(res);
       } catch(e) {
         return reject(e);
       }
     }
  });
}
```
以上便是针对resolve，和reject的封装，先来看下代码，如果走到onFullFilled的话，那么就先执行gen.next(), 如果gen内部没有针对错误try catch,就会走到onFullFilled的catch里面，从而将Promise的状态改为rejected。这时候发现，好像没有地方调用resolve。继续看下内部next方法的实现

```js
function co(gen) {
  const ctx = this;
  const args = Array.prototype.slice.call(arguments, 1);
  return new Promise((resolve, reject) => {
     if (typeof gen === 'function') {
        gen = gen.apply(ctx, args);
     }
     
     function next(ret) {
       if (ret.done) return resolve(ret.value);
       let value = ret.value;
       if (
        (typeof value === 'function' && value.constructor.name === 'GeneratorFunction') // is Generator Function
        || 
        (typeof value.next === 'function' && typeof value.throw === 'function') // is Generator object
       ) {
         value = co.call(ctx, value);
       }
       
       Promise.resolve(value).then(onFullFilled, onRejected);
     }
     
     function onFullFilled(res) {
       let ret;
       try {
        ret = gen.next(res);
       } catch(e) {
         return reject(e);
       }
       next(ret);
     }
     
     function onRejected(res) {
       let ret;
       try {
         ret = gen.throw(res);
       } catch(e) {
         return reject(e);
       }
       next(ret);
     }
  });
}
```
通过上面的代码可以看出，对于next方法，我们接收到的是每次yield执行后的内容作为参数。首先判断是否执行完Generator的代码，否则将递归的执行内部的Generator函数。最终将结果以promise输出。这里还差一点，那就是需要执行gen的next的方法，因此我们需要自己在内部调用onFullFilled。整体的全部代码为：

```js
function co(gen) {
  const ctx = this;
  const args = Array.prototype.slice.call(arguments, 1);
  return new Promise((resolve, reject) => {
     if (typeof gen === 'function') {
        gen = gen.apply(ctx, args);
     }
     onFullFilled();
     function next(ret) {
       if (ret.done) return resolve(ret.value);
       let value = ret.value;
       if (
        (typeof value === 'function' && value.constructor.name === 'GeneratorFunction') // is Generator Function
        || 
        (typeof value.next === 'function' && typeof value.throw === 'function') // is Generator object
       ) {
         value = co.call(ctx, value);
       }
       
       Promise.resolve(value).then(onFullFilled, onRejected);
     }
     
     function onFullFilled(res) {
       let ret;
       try {
        ret = gen.next(res);
       } catch(e) {
         return reject(e);
       }
       next(ret);
     }
     
     function onRejected(res) {
       let ret;
       try {
         ret = gen.throw(res);
       } catch(e) {
         return reject(e);
       }
       next(ret);
     }
  });
}
```

那么对于co.wrap方法就更简单了
```js
co.wrap = function(fn) {
  return function() {
    co.call(this, fn.apply(this, arguments);
  }
}
```