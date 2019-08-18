---
title: JavaScript对象的复制
date: 2017-10-30 21:07:17
tags: JavaScript
---
# JavaScript对象的复制

#### 假定一个对象为如下
<pre>
var obj1 = {
  a : 1,
  b : 'abc',
  c : true,
  d : [1,2,4],
  e : undefined,
  f : null,
  g : NaN,
  h : function(){},
  get i(){
  	return this.a;
  },
  set i(value){
    this.a = value;
  },
  j : {}
};
Object.defineProperty( obj1, 'd', {
  writable : false,
  configurable : false,
  enumerable   : false,	
} );
</pre>

### 整体思路
针对一个对象复制，思路即为遍历这个对象，然后将这个对象的属性和值拷贝给返回的对象上。

### 版本1 浅复制
因为js中对象的赋值是引用关系，所以如果我们之间以下面这种方式遍历复制一个对象的话
<pre>
  function shallowCopy( obj ){
    const copyObj = {};
    for( let key in obj ){
      copyObj[key] = obj[key];
    }
    return copyObj
  } 
</pre>
我们能够发现，这种复制是一种浅复制，两个对象之前是一种引用关系，改变其中一个，则会影响另一个对象。

### 深度复制
### 版本2.0 JSON方法复制对象
若一个对象是一个浅对象，即其值只含有JSON所包含的数据类型。有一种hack的写法。如下
<pre>
  function jsonCopy( obj ){
    return JSON.parse( JSON.stringify( obj ) );
  }
</pre>
但是这种情况有不足的地方，首要的一条便是他会省略值为undefined和函数的属性，并且将值为NaN的属性的值转换为null.

### 版本2.1 普通的深度复制
<pre>
  function deepClone1( copySource ){
    //使复制的对象copyObj和被复制对象有同样的原型
    const copyObj = Object.create( copySource.constructor.prototype );
    //遍历	
    function travel( dest, source ){  
      let keyType = null, value = null;
      for( let key in source ){ //可枚举属性
        if( source.hasOwnProperty(key) ){ //可枚举的自有属性
      	  value   = source[key];
          //属性值的类型
          keyType = Object.prototype.toString.call( value ).slice( 8, -1 );
          if( keyType === 'Array' ){
            dest[key] = [];
          }
          else if( keyType === 'Object' ){
            dest[key] = Object.create( value.constructor.prototype );
          }
          else{
            dest[key] = value;
            continue;
          }
          //递归调用
          travel( dest[key], source[key] )
        }
      }
    }
    travel( copyObj, copySource );
    return copyObj;
  }
</pre>  
这种方法看着比第一个好像要好点，但是写的不够优雅用到了(for...in)和对象的hasOwnProperty()方法  

### 版本2.2 稍微优雅一点的深度复制
<pre>
  function deepClone2( copySource ){
    const copyObj = Object.create( copySource.constructor.prototype );
    function travel( dest, source ){
      let keyType = null, value = null;
      //Object.keys()返回的是一个包含对象自有可枚举属性的数组
      Object.keys( source ).forEach( (key)=>{
        value = source[key];
        keyType = Object.prototype.toString.call( value ).slice( 8, -1 );
        if( keyType === 'Array' ) dest[key] = [];
          else if( keyType === 'Object' ) dest[key] = Object.create( value.constructor.prototype );
        else return dest[key] = value;
        travel( dest[key], source[key] );
      } );
    }
    travel( copyObj, copySource );
    return copyObj;
  }
</pre>
这种方法看着也好看了一点，不过好的也有限。我们可以发现，它对于getter和setter函数是无能为力的，并且不能复制对象属性的属性描述符。并且我们发现它不能复制一个对象的自有不可枚举的属性。复制后的对象如下<br />![](https://i.imgur.com/SVIFrR5.png)

### 版本2.3 更加健壮的深度复制
<pre>
  function deepClone3( copySource ){
    const copyObj = Object.create( copySource.constructor.prototype );
    function travel( dest, source ){
      let keyType = null, value = null, descript = null;
      //返回对象的所有自有属性
      Object.getOwnPropertyNames( source ).forEach( (key)=>{
        value     = source[key];
        keyType   = Object.prototype.toString.call( value ).slice( 8, -1 );
        //获得该对象上该属性的属性描述符
        descript  = Object.getOwnPropertyDescriptor( source, key );
        Object.defineProperty( dest, key, descript );
        if( keyType === 'Array' ) 
         dest[key] = [];
        else if( keyType === 'Object' ){
          dest[key] = Object.create( value.constructor.prototype );
        }
        else 
         return;
        travel( dest[key], source[key] );
      } );
    }
    travel( copyObj, copySource );
    return copyObj;	  
  }
</pre>
这个时候我们发现对于一个js中的一个对象，我们的这种方法好像完全没有问题了。但是........<br />
如果一个对象是一个环，也就是循环引用了的话，即对于对象obj1增加一条语句
`obj1.j.call = obj1.j;`<br />
看结果<br />
![](https://i.imgur.com/djVLXAI.png)<br />可以看出爆栈了................

###### **so,我们应该针对循环引用的对象做一些对策。**
我们知道在ES6中增加了Set和Map这两种数据类型，在ES6之前，我们不能将一个对象作为对象的属性key,对象只能作为value在对象中。但是利用Set和Map我们可以将对象作为key。对于对象循环引用的问题，我们可以用一个Set保存遍历的对象的记录，如果一个对象已经被遍历过，我们就用Set记录。如果一个对象已经被遍历过，说明发生了循环引用，这时候我们就不需要继续递归的深度遍历了。

### 版本2.4 支持对象循环引用的深度复制
<pre>
  function deepClone4( copySource ){
    const copyObj = Object.create( copySource.constructor.prototype );
    const visited = new WeakSet();
    visited.add( copySource );
    function travel( dest, source ){
      let keyType = null, value = null, descript = null;
      Object.getOwnPropertyNames( source ).forEach( (key)=>{
        value     = source[key];
        keyType   = Object.prototype.toString.call( value ).slice( 8, -1 );
        descript  = Object.getOwnPropertyDescriptor( source, key );
        Object.defineProperty( dest, key, descript );
        if( keyType === 'Array' ) dest[key] = [];
        else if( keyType === 'Object' ){
      	  if( visited.has(value) ){
      	    return;
      	  }
      	  dest[key] = Object.create( value.constructor.prototype );
      	  visited.add(value);
        }
        else return;
        travel( dest[key], source[key] );
  	  } );
    }
    travel( copyObj, copySource );
    return copyObj;	
  }
</pre>
这回我们发现我们写的最终的版本的JS对象深度复制解决了对象的深度遍历，对象属性的属性描述符，针对getter和setter函数也没有问题，同时也支持对象循环引用。OK，暂时就到这个版本了.......