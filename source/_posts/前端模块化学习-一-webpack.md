---
title: 前端模块化学习(一)webpack
date: 2018-05-22 22:01:33
tags: JavaScript模块化
---
  webpack作为一个前端打包器，现在在开发中已经必不可少了。以前学习中往往没有模块化的实践，认为遇不到，从而没有踏出舒适区，没有深入学习。在实习的过程中，学习了企业的代码发现工程化是当前最欠缺的一种能力。于是，这几天在毕业论文初稿定了的情况下，稍微深入学习了下webpack。
    
作为一个代码打包器，其必定需要输入和输出的配置。又因为其基于Node.js，配置文件必不可少的也是一个模块。因此先新建一个配置文件`webpack.config.js`, 很容易理解一下代码

```js
     const path = require("path");
     module.exports = {
       entry: "./src/index.js",
       output: {
         filename: "bundle.js",
         path: path.resolve(__dirname, "./dist")
       }
     }
```
以上代码中entry作为一个入口，表示webpack将从这个模块开始加载；同样output也容易理解就是作为最终打包后的文件，至于说filename和path，生成的一个文件其名称和位置作为必不可少的两个属性，我们需要告诉webpack。

ok, 现在我们将js文件打包了，那么html文件呢？既然我们知道了js的最终的打包文件，那就自己新建一个HTML文件，自己引用呗。这是一种方法，不过不符合我们的预期，因为其健壮性不够，一方面不够自动化，另一方面如果几十几百个文件，岂不是累死。因此，这个时候plugins出场了。

plugins作为一系列的插件，它能够帮我们做很多辅助的功能，如生成HTML文件。因此
```js
  plugins: [
    new HtmlWebpackPlugin(
      {  
        title: "webpack learning"
      }
    );
  ]
```
当然我们需要先`cnpm install html-webpack-plugin -D`  (cnpm是作为从淘宝npm镜像下载)。这个时候HTML解决了，那么作为前端三剑客，还有CSS呢？它我们又将如何处理？我们完成一个模块的时候，一个CSS文件也是一个模块，这个时候我们必须也能够处理CS内容，webpack利用
```js
    module: {
      rules: [
        {
            test: /\.css$/,
            use: [
              "style-loader",
              "css-loader"
            ]
        },
        {
           test:/\.js$/,
           use: "babel-loader",
           exclude: /(node_modules)/
        },
        {
           test:/\.html$/,
           use: "html-loader"
        }
      ]
    }
```
以上我把HTML，CSS，JS的处理都写出来的，同样需要先执行`cnpm install html-loader css-loader style-loader babel-loader html-loader -D`以上代码就是说遇到代码我用各个loader处理一下。

好了，看样子我们好像最基本的功能就是完成了。不过作为一名有追求的开发者，我们不可能每次修改一部分就在命令行里面敲一个`npm run build`。解决方案有多种，一种是观察者模式直接用 `webpack --watch`,  它会跟踪所有文件，进行更改时便会重新编译，缺点就是每次我们必须重新刷新浏览器才能看到修改结果。第二种方法是用`webpack-dev-server`,先`cnpm install webpack-dev-server -D`安装这个包。同样配置文件需要添加内容
```js
  devServer: {
    contentBase: "./dist",
    host: "127.0.0.1",
    port: "9527"
  }
```
以上就会开启一个服务器，端口为9527。之后当我们修改任何文件的时候，都会重新编译自动刷新。至于第三种使用`webpack-dev-middleware`都差不多。

当然最后少不了利用source-map, 能够在源代码上debug.即在配置文件webpack.config.js中添加
```js
  devtool: "inline-source-map"
```

以上就是一个简单的开发环境，至于更加深入的内容，下次再总结。