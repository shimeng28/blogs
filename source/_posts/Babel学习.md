---
title: Babel学习
date: 2021-01-03 13:18:29
tags: 前端工程化
---

Babel的主要功能就是将ES6+的代码转换成ES5等向后兼容的代码，代码向后转换就是两种途径，一种是改变代码，一种是改变代码运行的环境。Babel7相对于Babel6更加的精简和聚焦，核心npm包都在@babel这个域下, 主要是@babel/core、@babel/cli。cli安装之后就可以在shell命令行使用，@babel/core是babel的核心功能，@babel/cli也是依赖在@babel/core上面的。

##插件和预设
Babel的配置文件有.babelrc、.babelrc.js、babel.config.js、package.json中babel字段，主要配置有两项，分别是plugins和presets，preset是一系列plugin的集合，Babel7中preset有@babel/preset-env、@babel/preset-flow、@babel/typescript、@babel/react。各个详细内容官网都有，这里主要通过@babel/preset-env来深入学习Babel的使用。

##@babel/preset-env
@babel/preset-env可以通过配置目标环境转换代码语法，首先可以配置目标环境，这个可以直接读取browserslist的配置，项目根目录下.browserslistrc或者package.json中browserslist字段，当然@babel/preset-env的配置项中也有targets，这个也是用来配置目标环境。这个配置完了只是能转换语法，比如箭头函数，class语法，那对于API，像Promise、Symbol、Async、Generator以及String和Array原型上的新加入的方法，@babel/preset-env就没有办法。这里有一种方式就是在全局环境添加垫片polyfill。

##垫片polyfill
添加垫片就是在项目代码执行之前，给运行环境添加其没有的API特性，添加的方式有多种，
  1. 在html中添加polyfill.js
  2. 在前端项目的入口文件引入polyfill.js或者@babel/polyfill
  3. 在构建工具的配置文件入口引入polyfill.js或者@babel/polyfill

##减少引入的polyfill体积
上面添加polyfill的方式是将整个polyfill文件添加到项目里，这会大大增加页面加载的时间，这里就有了@babel/preset-env第二个重要的配置项useBuiltIns, 它的值可以有false，entry，usage。默认值就是false，就是将所有的polyfill文件全部引入到项目中; entry就会根据配置的目标环境引入缺少的API特性，不过使用这个配置我们需要在入口文件手动引入@babel/polyfill。usage就是会更近一步，只会引入我们代码中使用了的目标环境缺少的API特性。(ps: 这里补充一点，Babel官网建议，babel7.4之后建议使用core-js@3和regenerator-runtime两个包替换@babel/polyfill, @babel/polyfill是包含core-js@2和regenerator-runtime两部分内容，core-js@2和3的区别就是core-js@2不再更新，core-js@3包含了更新的特性，regenerator-runtime是引入Generator和Aysnc函数)。

##减少语法转换引入的代码
通过@babel/preset-env转换后的代码看，多了一些辅助方法_classCallCheck、_defineProperties、_createClass等，如果每个文件都有大量相同的辅助函数，那会大大增加转换后代码的体积，而像webpack这类工具，都是以模块的方式去重，是没有办法去重不同文件内相同的函数的。所以这个时候需要有一个npm包包含这些方法，它就是@babel/runtime, 有了@babel/runtime之后，那就需要有一个工具能够帮我们替换Babel转换语法添加的辅助函数，那就是@babel/plugin-transform-runtime,它可以自动帮我们将辅助函数替换为@babel/runtime中的方法，构建工具就会只引入一份代码，减少代码体积。

##@babel/plugin-transform-runtime
自动替换辅助函数只是@babel/plugin-transform-runtime的一个功能, 还有一个功能就是替换我们源代码使用的ES API特性，上面用polyfill的方式是直接在运行环境中添加其不支持的特性，通过@babel/plugin-transform-runtime，可以替换我们使用的新的API，例如Promise，对于不支持的浏览器来货，@babel/plugin-transform-runtime会将我们使用的Promise替换为babel实现的Promise，这个可以通过corejs的配置参数来确定，corejs的默认参数是false，就是关闭这个功能，另外两个参数分别是2和3，和@babel/preset-env的corejs参数一样，分别代表core-js@2、core-js@3。不过这里有个区别，如果我们开启了替换core-js相关的API，需要把@babel/rumtime替换为@babel/runtime-corejs2或者@babel/runtime-corejs3。

剩下的参数helpers，默认是开启的，就是替换为@babel/runtime中的辅助函数。regenerator默认开启，这个的意思就是替换Generator和Async函数，这里和polyfill方式的regenerator-runtime的功能是一样的，只不过用@babel/plugin-transform-runtime不污染全局环境。version默认是7.0.0，可以设置成我们安装的@babel/runtime或者@babel/runtime-corejs(2、3)的版本号，可以减少打包的体积，
  



