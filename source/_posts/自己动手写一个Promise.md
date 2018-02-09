---
title: 自己动手写一个Promise
date: 2017-11-03 14:47:14
tags: JavaScript 
---
### 第一阶段：先分析Promise
1. 每一个Promise有一个短暂的生命周期,刚开始处于**Pending**,表示为处理；异步操作结束，则有两种结果状态**Fulfilled(表示成功完成)**和**Rejected(表示因为某些原因异步操作未能完成)**，这两种状态是互斥的，其结果只能为其中的一个，并且**Fulfilled可以转化为Rejected**(通过内部throw new Error（）)，但是**Rejected不能转化为Fulfilled**。<br /><br />
2. Promise的原型中有then()和catch方法表示处理异步操作结果程序。then()方法可以有两个参数，**resolve和reject**分别代表**当异步操作结果状态为fulfilled和rejected时的处理程序**，若then()只有一个参数，则为resolve.而catch()方法则是表示异步操作结果状态出错为rejected时的处理结果。若是不传递参数，则表示不做处理。<br /><br />
3. Promise有四个静态方法：resolve()、reject()、race()、all().
   * resolve()返回一个状态为fulfilled的已完成promise
   * reject() 返回一个状态为rejected的已完成promise
   * race() 接收一个promise数组,当其中一个最先解决时，若这个最先解决的是状态为fulfilled的Promise，则返回状态fulfilled的promise; 若这个最先解决的是状态为rejected的Promise，则返回为状态为rejected的proimse;
   * all() 接收一个promise数组，若所有promise都被成功解决后，返回一个状态为fulfilled的promise; 只要有一个promise的处理结果状态为rejected，则返回为状态为rejected的proimse；
   <br/>
4. 调用then()和catch()后返回一个promise，因此支持链式调用。并且可以给同一个链上的后面的promise传递数据。

### 第二阶段：写一个Promise构造函数
1. 根据阶段一的1可以知道一个promise应该有一个属性表示状态，同时根据2，我们也需要一个属性保存异步操作结果处理程序。当然它还要有一个属性保存传给操作结果处理程序的参数值。又因为Promise构造函数传入一个函数参数，并且该函数参数接收两个函数参数resolve()和reject()。so 代码如下：
<pre>
function _Promise(fn){
    //默认状态
    this.promiseStatus  = 'Pending';  
    //默认值
    this.promiseValue   = null; 
    //用作存放处理程序
    this.statusQueue    = {
      Fulfilled : [],
      Rejected  : []	
    };
    //保存this值
    const that = this;  
    //执行传入的函数参数
    if( typeof fn !== 'function' ){
      //传入的参数如果不是函数
      throw new Error( 'Promise resolver is not a function' );
    }
    fn.call( null, resolve, reject );
    //resolve函数
    function resolve(value){
      that.promiseStatus = 'Fulfilled';
      that.promiseValue  = value;
    }
    //reject函数
    function reject(value){
      that.promiseStatus = 'Rejected';
      that.promiseValue  = value;
    }
  }
</pre>
为了区分官方的Promise，我们在Promise前加一个横杠作为区分。并且代码中我们之所以要保存this的值也就是promise对象，主要是因为当执行resolve()和reject()时，我们是在改变promise对象上属性值，因此需要外面的this。<br />
上面代码比较简单，因此这一阶段完成。<br/>

### 第三阶段：完成原型上的then()和catch()方法
1.思考一下，then()方法可以接收两个参数(resolve,reject)，或者一个参数(resolve)作为异步操作结果处理程序,那么then()执行过后应当执行传入resolve和reject函数。so 代码如下：
<pre>
  _Promise.prototype.then = function( resolve, reject ){
    //如果当前状态为fulfilled并且resolve为函数
    if( typeof resolve === 'function' ){
      this.statusQueue.Fulfilled.push( resolve );	
    }
    if( typeof reject === 'function' ){
      this.statusQueue.Rejected.push( reject );	
    }
    //若状态为fulfilled或者为Rejected,则statusQueue中有该状态的响应事件,执行队列中的函数
    this.statusQueue[this.promiseStatus] && util.execQueue(this, this.promiseStatus);

    //链式调用
    return this;
  };
</pre>
2.我为什么要这样写呢？  首先我们要检查传入的是不是函数，只有是函数了我们就将它压入响应状态的处理程序队列中去。其次，我们要思考一个问题，如果传入Promise构造函数中的参数异步执行resolve和reject，那么当执行到then()方法时,这时候promise的状态还是Pending，如果直接执行，则会报错。这也是前面为什么statusQueue中的两个属性key与状态一致的原因，这样我们就可以检查状态是否改变，如果状态改变，则执行处理程序队列中的函数；若状态没有改变，就不执行。这里用到了与操作(&&)的一个技巧。<br />

3.这里我们就又发现了一个问题，当构造函数中的参数异步执行resolve或reject的时候，我们在then()方法内有没有执行处理程序，这时候我们就需要在第二阶段resolve()和reject()函数中也要尝试执行一次处理程序队列中的函数。因此需要在第二阶段resolve()和reject()加入一下代码:
<pre>
  util.execQueue( that, 'Fulfilled' );
</pre>

4.为了重复利用代码，我将执行处理程序队列的代码放到了一个函数中。具体代码如下：
<pre>
 const util = {
    //负责执行处理程序队列的函数
    execQueue(that, status){ 
      setTimeout( ()=>{
        let returnValue = null;	
      	that.statusQueue[status].forEach( (fn, index, arr)=>{
      	  returnValue = fn.call(null, that.promiseValue);
          //如果promise处理程序有返回值，则将这个值传给链后续的promise
      	  if(returnValue) {
            that.promiseValue = returnValue;
      	  }
      	} );
      	//执行完要清除处理程序队列
      	that.statusQueue[status].length = 0
      }, 0 );
    }	
  };
</pre>

5.解释一下上面的代码，执行处理程序队列中的处理程序并不是立即执行，我们要把它放到js任务队列的最后，也就是先执行promise后面的代码，之后再调用处理程序。为了给同一链上的其他promise处理程序传递数据，我们将处理程序的返回值赋值给promiseValue。因为promiseValue就是传递给处理程序的参数。<br />

6.**catch()方法的实现** catch()的代码思路和then()的一样，直接贴代码:
<pre>
  _Promise.prototype.catch = function( reject ){
    if( typeof reject === 'function' && this.statusQueue.Rejected.length ===0 ){
      this.statusQueue.Rejected.push( reject );
    }   
    //若状态为fulfilled或者为Rejected,则statusQueue中有该状态的响应事件,执行队列中的函数
    this.statusQueue[this.promiseStatus] && util.execQueue(this, this.promiseStatus);
    return this; 
  };
</pre>

7.本阶段最后的最后，为了能够表示出我们写的这个Promise，使用es6里面的方法重写一下当用toString()识别时的返回值.
<pre>
  //识别自定义的_Promise对象
  _Promise.prototype[Symbol.toStringTag] = '_Promise';  
</pre>

### 第四阶段: 完成静态方法resolve(),reject(),race(),all()
#### 1.Promise.resolve(value)
直接返回一个状态为fulfilled的promise。
如果传入的是一个promise对象，则直接返回给对象。
如果传入的是一个thenable的非promise对象，可将其转换为promise对象。
代码如下：
<pre>
  _Promise.resolve = function( value ){
    //非_Promised的thenalbe对象
    if( typeof value.then === 'function' ){
      return new _Promise( value.then );	
    }
    //传入一个自定义的_Promise对象
    if( Object.prototype.toString.call(value).slice( 8, -1 ) === '_Promise' ){
      return value;	
    }
    //fulfilled状态的promise
    function resolved_Promise(resolve){
      resolve( value );   	
    }

    return new _Promise( resolved_Promise );
  }
</pre>

#### 2.Promise.reject(value)
与Promise.resolve()类似.代码如下:
<pre>
  _Promise.reject = function( value ){
    if( Object.prototype.toString.call(value).slice( 8, -1 ) === '__Promise' ){
      return value;	
    }
    //rejected状态的promise
    function rejected_Promise(resolve, reject){
      reject( value );   	
    }

    return new _Promise( rejected_Promise );
  };
</pre>

#### 3.Promise.all([promise1, promise2, promise3...]);
Promise.all()接收一个具有多个promise的可迭代对象。根据阶段一的分析：
>  all() 接收一个promise数组，若所有promise都被成功解决后，返回一个状态为fulfilled的promise; 只要有一个promise的处理结果状态为rejected，则返回为状态为rejected的proimse；

代码实现如下：
<pre>
  _Promise.all = function( promiseList ){
    return new Promise( function(resolve, reject){
      const resolveResult = [], len = promiseList.length;
      //状态为resolve的promise的个数
      let num = 0;   
      //promise resolved时
      let wrapResolve = function(index){
        return function(value){
      	  num++;
          resolveResult[index] = value;
          if( num === len ) resolve(resolveResult);
        }	
      }
      //promise rejected时
      let wrapReject = (function(){
      	let isReject = false;
      	return function(value){
      	  if( !isReject ){
            reject( value ); 
            isReject = true;
      	  }	
      	}
      })();
      promiseList.forEach( (promise, index)=>{
        promise.then( wrapResolve(index), wrapReject );	
      } );    
    } );
  };
</pre>
代码解释：我们首先返回一个promise，其状态取决于传入的多个promise的状态。wrapResolve()表示当所有的promise都为fulfilled状态时，执行resolve()方法。
当第一个promise状态为rejected时，执行reject()方法。

#### 4.Promise.race([promise1, promise2, promise3...])
Promise.race()也接收含有多个promise的可迭代对象，根据阶段一的分析：
> all() 接收一个promise数组，若所有promise都被成功解决后，返回一个状态为fulfilled的promise; 只要有一个promise的处理结果状态为rejected，则返回为状态为rejected的proimse；

代码如下：
<pre>
   _Promise.race = function( promiseList ){
    return new Promise( function(resolve, reject){
      let isResult = false;
      function wrapResolve(value){
        if( !isResult ){
      	  isResult = true;
      	  resolve( value );
        }  
      }
      function wrapReject(value){
        if( !isResult ){
      	  isResult = true;
      	  reject( value );
        }
      }
      promiseList.forEach( (promise)=>{
        promise.then( wrapResolve, wrapReject );	
      } );
    } );
  }
</pre>
Promise.race()方法是只要传入的promise中有一个promise状态发生改变，则返回一个异步操作结果。

### 总结
以上就是针对ES6中Promise的代码模拟，整体来说，实现一遍对整个Promise的理解更全面。