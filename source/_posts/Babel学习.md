---
title: Babel学习
date: 2022-02-19 13:18:29
tags: 前端工程化
---

Babel的主要功能就是将ES6+的代码转换成ES5等向后兼容的代码，代码向后转换就是两种途径，一种是改变代码，一种是改变代码运行的环境。Babel7相对于Babel6更加的精简和聚焦，核心npm包都在@babel这个域下, 主要是@babel/core、@babel/cli。cli安装之后就可以在shell命令行使用，@babel/core是babel的核心功能，@babel/cli也是依赖在@babel/core上面的。

##插件和预设
Babel的配置文件有.babelrc、.babelrc.js、babel.config.js、package.json中babel字段，主要配置有两项，分别是plugins和presets，preset是一系列plugin的集合，Babel7中preset有@babel/preset-env、@babel/preset-flow、@babel/typescript、@babel/react。各个详细内容官网都有，这里主要通过@babel/preset-env来深入学习Babel的使用。

##@babel/preset-env
@babel/preset-env可以通过配置目标环境转换代码语法，首先可以配置目标环境，这个可以直接读取browserslist的配置，项目根目录下.browserslistrc或者package.json中browserslist字段，当然@babel/preset-env的配置项中也有targets，这个也是用来配置目标环境。这个配置完了只是能转换语法，比如箭头函数，class语法，那对于API，像Promise、Symbol、Async、Generator以及String和Array原型上的新加入的方法，@babel/preset-env就没有办法。这里有一种方式就是在全局环境添加垫片polyfill。

##垫片polyfill
添加垫片的方式有多种，
  1. 在html中添加polyfill.js
  2. 在前端项目的入口文件引入polyfill.js或者@babel/polyfill
  3. 在构建工具的配置文件入口引入polyfill.js或者@babel/polyfill
