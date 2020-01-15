---
title: NodeJS参数校验
date: 2020-01-12 20:14:21
tags: NodeJS
---
ES属于动态弱类型语言，因此只有在运行时才能确认变量的类型，所以往往在代码的实现中，往往能看到非常多的&&校验，当然这种校验是符合安全策略的，因为这样使用的时候就表明这个变量不是我们自己定义的，例如后端接口发送过来的内容，那么就没有办法保证任何时刻返回的数据类型都是和我们想要的一致。

那么当我们将ES放到NodeJS这个运行环境中去的时候，我们自己实现一个接口，而接口就会有参数，其中参数有三种，路由参数，query参数，请求体body中的参数。如果我们根据rest风格来设计我们的接口，那么路由参数就是必传的，query参数是可选的，传一个就表明过滤一种情况，而请求体中的参数是针对POST请求发送过来的，他的参数内容也是不定的。

这个时候如果我们用ES去实现一个接口，在路由中就会写一大堆&&校验，并且还是需要在每个接口中都要这么实现一遍，因此是否可以将他们封装抽离出来的，那当然是可以的，下面来看下如何实现。

第一步需要先思考🤔校验的逻辑，其实正常的逻辑都是先定义，后校验。定义就是定义参数的类型，是否可选，这两个条件就够了。而校验就是检查接口调用方传递过来的参数是否符合之前定义。这里我们将参数的定义这样设计

```
/*
* @param {
*   key: type? // ?表明该类型是可选的，否则都是必选的，其中key是参数名称，type是参数类型
* } paramDefine
*/
validate(paramDefine) 
```

在JS中由于typeof无法区分array类型，因此type的值是通过Object.prototype.toString获取到的，因此那就先将获取类型封装为一个函数
```js
const isType = (param) => Object.prototype.toString.call(param).slice(8, -1);
```
这样返回的类型值都是诸如Array，String，Object之类的，首字母大写。

我们以Koa2为例，那么validate这个方法应当挂载到哪个环境变量上比较合适呢？我的做法是validate挂载到app然后返回一个方法，这个方法的参数是context，这样的话相当于在服务启动的时候将准备工作做完，而在具体的每个连接的时候，再去校验每个连接的参数。这也是在服务端开发中提升性能的一个方法。

```js
app.validate = function (paramDefine) {
    assert.strictEqual(isType(paramDefine) === 'Object', true, new TypeError('validate: arguments must be object'));
 
    const typeDefineMap = parseDefine(paramDefine);
    return (ctx) => {
      const { params = {}, query = {}, request } = ctx;
      const { body = {} } = request;
      const data = {
        ...params,
        ...query,
        ...body
      };
      for (let [key, { type, required }] of typeDefineMap) {
        const currType = isType(data[key]);
        if (
          // 必填 但是没有参数
          (required && !data.hasOwnProperty(key))
          ||
          // 必填 参数格式错误
          (required && currType !== type)
          ||
          // 参数有值，格式不对
          (currType !== 'Undefined' && currType !== type)
        ) {
          ctx.fail(`参数格式错误key:value:type = ${key}:${JSON.stringify(data[key])}:${type}`, 400);
          return false;
        }
      }

      return true;
    };
  }
```
先将参数的类型定义解析，解析后的格式为
```
map<key, typeDescribe>

typeDescribe {
  type, // 类型
  required: Boolean // 是否必填
}
```
解析后的内容返回的是一个Map对象，其中key是参数的名称，value包含类型和是否必填的选项。那么接下来看下parseDefine如何实现呢？

```js
const parseDefine = (defineObj) => {
  const typeMap = new Map();
  //正则匹配
  const reg = /\b(\w)(\w*)(\?)?/g;
  Object.keys(defineObj).forEach((key) => {
    const typeStr = defineObj[key];
    typeStr.replace(reg, ($0, $1, $2, $3) => {
      const typeVal = {
        type: `${$1.toUpperCase()}${$2.toLowerCase()}`,
        required: true,
      };
      if (typeof $3 !== 'undefined' && $3 === '?') {
        typeVal.required = false;
      }
      typeMap.set(key, typeVal);
    })
  });
  return typeMap;
};
```
先用正则匹配每一个定义的参数类型数据信息，之后解析为type指类型，首字母大小，required是否为必填项

那如何使用呢？

```js
module.exports = (app) => {
  const router = new Router();
 
  const getCardValidate = app.validate({
    cinemaId: 'String?',
    memberCardId: 'String',
    beginDate: 'String',
    endDate: 'String',
  });

  router.get('/member-card/:memberCardId/getInfo', async (ctx) => {
    const validateResult = getCardValidate(ctx);
    if (!validateResult) return;
    //... 其他逻辑代码
  })
}
```

以上便是整个针对NodeJS接口做的参数校验。