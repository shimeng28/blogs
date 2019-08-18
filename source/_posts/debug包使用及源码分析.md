---
title: debug包使用及源码分享
date: 2018-12-16 15:11:21
tags: JavaScript
---
### 1. 使用介绍
debug包是一款跨环境的npm包，主要作用如它的名字所示辅助代码debug [GitHub地址](https://github.com/visionmedia/debug)。 在我们的项目中可以通过npm安装
> npm install debug

之后就可以通过

```js
const createDebug = require('debug');
const debugA = createDebug('a');
debugA('%o', 'i am a');
```
在命令行环境下为
![使用介绍](https://res.cloudinary.com/duoxjcv8x/image/upload/v1544944970/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7_2018-12-16_%E4%B8%8B%E5%8D%883.22.35_uqkbvf.png)

通过传入的命令行参数配置debug生效模块。

### 2. 源码分析
基本的使用大体就是以上所介绍的，debug包作为一个可以兼容浏览器和node环境的npm包，其主要的作用就是打印日志，当然在其上还添加了时间戳，命名空间，颜色等特性，相比原生的console.log功能更加强大。下面主要分析其源码。

#### 2.1 Node和浏览器共用的JavaScript包
兼容两个环境，必然涉及到环境的识别判断。在浏览器和node中其最大的区别便是对process的判断。正如源码中所看到的一样。
![环境判断](https://res.cloudinary.com/duoxjcv8x/image/upload/v1544946186/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7_2018-12-16_%E4%B8%8B%E5%8D%883.42.54_hcb7dg.png)
process如果为undefined，那么必然不是在node环境下。

1. 相比process.type === 'renderer', 那是因为Electron中的process对象type属性值为browser(ie main process)或renderer，在Electron中 main进程关注应用的开启和运行，是Electron应用的首要入口。而renderer进程作为Chromium浏览器运行环境，因此如果process.type的值为'renderer'，我们就能认为当前环境为浏览器环境。
2. process.browser === true 则是因为在前端构建工具中，会将process.browser设置为true。
3. process.\__nwjs 是因为在NW.JS中有两种环境，node环境和浏览器环境，如果process.__nwjs不为假值，我们就可以认为当前环境为浏览器环境。

以上就是区分两种环境的方式，通过上面代码我们如果想要发布一个兼容两个环境的包时的方案。

#### 2.2 如何打印出颜色

在node中 我们可以通过以下格式添加颜色
![ANSI escape code](https://res.cloudinary.com/duoxjcv8x/image/upload/v1544947957/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7_2018-12-16_%E4%B8%8B%E5%8D%884.12.07_ettukx.png)

有两种类别，其中当颜色值小于8时，可以通过
> \033[31;1m i am message \033[0m

将 i am message打印出红色。通过colorValue的取值不同，其打印出的颜色也不同。但是这种格式只能打印0-7共8中颜色。
![颜色值](https://res.cloudinary.com/duoxjcv8x/image/upload/v1544948284/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7_2018-12-16_%E4%B8%8B%E5%8D%884.17.45_nmc7dv.png)

如果想要打印更多的颜色时，可以使用第一种格式。
> \033[38;5;10;1m i am message \033[0m

![颜色值](https://res.cloudinary.com/duoxjcv8x/image/upload/v1544948352/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7_2018-12-16_%E4%B8%8B%E5%8D%884.18.58_r8xerk.png)

在浏览器中，颜色生成比较简单只是需要
> console.log('%c i am red', 'color: #FF0066')

就可以打印出彩色的log。

#### 2.3 源码分析
整个包的结构分为4个模块其中browser.js作为适配浏览器环境，node.js适配node环境，common.js作为包的核心，两种环境适配均是基于它之上的，而index.js则是作为环境识别的入口文件。

index.js 中的内容在`2.1 Node和浏览器共用的JavaScript包` 中已经了解，这里不作赘叙。主要学习下common.js中的代码。

```js

function setup(env) {
  function createDebug(namespace) {
    function debug() {}
    return debug;
  };
  
  return createDeubg;
}

module.exports = setup;
```

由此可以看到，这里嵌套了两层高阶函数。setup方法中为createDebug添加静态方法，并将命令行中配置的DEBUG值。

```js
function setup(env) {
  createDebug.debug = createDebug;
  createDebug.default = createDebug;
  createDebug.coerce = coerce;  // 过滤 Error 对象
  createDebug.disable = disable; // disable 所有debug
  createDebug.enable = enable;  // 根据黑名单和白名单的内容生成debug命名空间
  createDebug.enabled = enabled;  // 判断传入的name空间是否为生效状态
  createDebug.humanize = require('ms');
  
  Object.keys(env).forEach(key => {
    createDebug[key] = env[key];
  });
  
  createDebug.instances = []; // debug 实体数组
  
  createDebug.names = [];  // 白名单
  createDebug.skips = [];  // 黑名单
  
  createDebug.formatters = {}; // %n n => [a-zA-Z]处理方法
  
  createDebug.enable(createDebug.load()); 设置传入的debug模块

  return createDebug;
}
```

对于debug方法，其作用为为debug生成其所需的属性和方法。

```js
function debug(...args) {
  if (!debug.enabled) {
    return;
  }

  const self = debug;
  const curr = Number(new Date());
  const ms = curr - (prevTime || curr); // 时间差
  self.diff = ms;
  self.prev = prevTime;
  self.curr = curr;
  prevTime = curr;

  args[0] = createDebug.coerce(args[0]);

  if (typeof args[0] !== 'string') {
    args.unshift('%O');
  }

  // 调用formatters中的自定义方法
  let index = 0;
  args[0] = args[0].replace(/%([a-zA-Z%])/g, (match, format) => {
    if (match === '%%') { // 如果为 %% 略过
      return match;
    }
    index++;
    const formatter = createDebug.formatters[format];
    if (typeof formatter === 'function') {
      const val = args[index];
      match = formatter.call(self, val);
      args.splice(index, 1);
      index--;
    }
    return match;
  });
  
  // 格式化 参数
  createDebug.formatArgs.call(self, args);
  
  // 打印日志
  const logFn = self.log || createDebug.log;
  logFn.apply(self, args);
}
```

对于createDebug方法而言，它的作用和setup的作用是类似的，也是为debug方法添加静态方法

```js

function createDebug(namespace) {
  let prevTime;

  function debug(...args) {
  }

  debug.namespace = namespace; // 名称
  debug.enabled = createDebug.enabled(namespace); // 是否开启
  debug.useColors = createDebug.useColors(); // 是否使用颜色
  debug.color = selectColor(namespace); // 颜色值
  debug.destroy = destroy;  // 将当前debug从instances中删去
  debug.extend = extend;  // 生成一个新的debug模块 名称为当前的名称：新名称

  if (typeof createDebug.init === 'function') {
	createDebug.init(debug); // 初始化
  }

  createDebug.instances.push(debug);

  return debug;
}
```

node.js和browser.js文件中的内容，其作用便是适配环境并定义createDeug静态方法
node.js

```js
exports.init = init;
exports.log = log;
exports.formatArgs = formatArgs;
exports.save = save;
exports.load = load;
exports.useColors = useColors;

exports.colors = [6, 2, 3, 4, 5, 1];

module.exports = require('./common')(exports);
```

browser.js

```js
exports.log = log;
exports.formatArgs = formatArgs;
exports.save = save;
exports.load = load;
exports.useColors = useColors;
exports.storage = localstorage();

exports.colors = [];
module.exports = require('./common')(exports);
```

以上便是整个debug包的内容，这里只能大致介绍一下，源码只有自己读一遍才能体会到它的奥妙。













