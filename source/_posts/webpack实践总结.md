---
title: webpack实践总结
date: 2019-11-23 21:13:51
tags: 前端工程化
---
webpack作为前端打包构建工具，在以前也写过几篇相关的文章，相对来说还是比较入门的内容。本文是对使用webpack的一个总结。

### sourceMap

webpack的作用是打包构建，同样内部可以通过使用babel-loader来编译最新的ES版本，从而可以兼容浏览器老版本。这里可以简单的说下，babel的编译分为两方面，一方面是语法的转化，另一方面是api的兼容。语法的方面比较容易理解，babel直接就能解决，但是对于api的兼容就有了两种方案，一种是babel-polyfill, 这种方式就是通过将新的api语法赋值到全局上，在浏览器中也就是window对象上，将一些方法直接在原型上实现，这样子就可以让我们的ES6+的代码能够运行在不兼容的浏览器中。另一种就是@babel/plugin-transform-runtime, 这种方式是转换我们的源代码，使得我们的项目代码能够在不兼容浏览器中运行。第一种polyfill方式适合在项目中使用，因为只有我们自己在用，而另一种transform-runtime就需要在组件开发中使用，因为组件会被别的项目中使用，我们不能通过改变全局变量的形式来发布我们的组件。

上面这些都是针对babel来说的，言归正传，来回到webpack中，webpack通过babel将我们的源代码编译转换从而兼容浏览器，就会改造我们的代码，那自然而然的就会有一个问题，那就是产生编译后的代码在源代码中的位置关系，这就是source-map。虽然babel已经编译过了，但是我们在调试，已经报错的时候都可以将源代码中的位置暴露出来，从而解决问题。 在webpack中通过

```js
devtool: 'source-map'
```
source-map的配置值有多个，分别有eval、hidden、inline、cheap、module五种关系，我们可以在他们之间互相组合

1. eval会把每个模块封装到eval函数中，并且会在末尾添加DataURL注释
2. hidden 会生成map文件，但是编译后的文件的末尾不会添加DataURL注释
3. inline是将sourceMap以DataURL的方式内嵌到文件末尾
4. cheap生成的sourceMap没有列信息，只有行信息
5. module包含loader的sourceMap

因此通过以上我们知道，在线下开发环境应当使用
`inline-eval-cheap-module-source-map`,而线上环境就不能将source-map打包到文件中了，可以使用`module-source-map`

### Tree Shaking
在我们引用一个封装的模块的时候，往往没有使用所有的代码，这个时候普通的打包就会将整个模块打包到项目文件中，而通过Tree Shaking，可以将未使用的代码不会被打包到项目文件中。那么它是如何做到的呢？这个跟ES Module有关，当使用ES Module中，模块的导入都在顶部，并且我们无法动态的导入一个模块，ES Module导入的模块和运行时的状态是无关的，因此可以通过静态检查的时候就知道我们会使用模块中的哪些内容，也就可以插去无用的代码，减少代码的体积。当然这个的前提就是使用ES Module。

### 代码分割
我们知道在浏览器中，js代码有两种，一种是同步的代码，根据代码的顺序依次放入到调用栈执行，另一种是异步的代码，事件回调，定时器等。那么它跟我们打包构建有什么关系呢？我们可以想象一个场景，webpack通过入口文件，依次深度遍历整个依赖模块，最终打包出一个目标文件。现在前端单页项目也越来越大，如果不做处理的话，整个项目js代码都在一个文件中，页面的加载会非常慢，同时首页的代码利用率会非常的低。因此我们需要通过一种思路，将代码在被需要的时候调用执行，这也叫做按需加载。

那么在webpack中，有两种代码分割，一种是同步的代码，一种是异步的代码。先来说异步的代码，以react框架为例，最常见的就是在路由中通过ES Module的动态import配合React.lazy动态导入一个模块，这样我们就可以在路由切换的时候再加载下一页的代码。使用代码如下

```jsx
import React from 'react';

export default {
  path: '/errors',
  routes: [
    {
      path: '/403',
      component: React.lazy(() => import('../pages/errors/403')),
    },
  ]
};
```


另一种那就是同步代码的分割，异步代码的分割很容易理解，按需加载减少不必要的js代码加载。同步代码为什么要有代码分割呢？其实主要是当把同步代码分割之后，由于我们现在将文件名增加了contentHash用来区分代码的更改，这样的话，内容未更改的文件名称没有变化，便于浏览器的缓存。在webpack中同步代码的分割主要是通过optimization的splitChunks属性的值来配置。

```js
const config = {
  optimization: {
    splitChunks: {
      chunks: 'all',
      minSize: 30000,
      maxSize: 0,
      minChunks: 1,
      maxAsyncRequests: 5,
      maxInitialRequests: 3,
      automaticNameDelimiter: '~',
      automaticNameMaxLength: 30,
      name: true,
      cacheGroups: {
        vendors: {
          test: /[\\/]node_modules[\\/]/,
          priority: -10
        },
        default: {
          minChunks: 2,
          priority: -10,
          reuseExistingChunk: true
        }
     }
  }
}
```

在splitChunks中，chunks的值默认为async，也就是说webpack其实默认支持异步代码的代码分割，minSize是指模块代码体积超过该值则分割代码，maxSize是指一个被分割的模块超过该值时是否继续分割下去，默认这里为0，也就是不再继续分割。minChunks是指要分割的代码最少被应用了多少次。可以主要看下cacheGroups，这个对象的内容主要是将代码分组，以我们上面的为例，在node_modules中的模块会被分割为vendors, 我们自己src中的模块被分割为default。

### webpack DLL
在项目代码中，我们会引入一些npm包，这些包会在使用require或者import的时候引入，而这个时候webpack会将其加载打包进当前文件。这个时候如果我们提前将这种库文件打包好，在项目打包构建的时候如果遇到需要导入的时候可以直接使用，这样的话就能减少我们构建的时间，这就是webpack DLL的作用。

```js
const path = require('path');
const webpack = require('webpack');
module.exports = {
  mode: 'production',
  entry: {
    react: [
      'react',
      'react-dom',
    ]
  }
  output: {
    filename: '[name].dll.js',
    path: path.resolve(process.cwd(), './webpack/dll'),
    library: '[name]',
  },
  plugins: [
    new webpack.DllPlugin({
      name: '[name]',
      path: path.resolve(process.cwd(), './webpack/dll/[name].manifest.json'),
    }),
  ],
};
```

这是dll构建的配置文件，其中将包的名称放到入口entry中，最终将其打包到output输出文件中，这里只是我们将整个库打包。在DllPlugin中，会生成一个json文件，这个文件包含了模块的映射，通过在output中library的配合可以将dll函数暴露到全局中。

那如何使用呢？通过添加两个webpack plugin，我们可以在打包构建的过程中如果发现了引入的模块已经被打包了，就可以直接将打包后的文件引入。
```js
plugins: [
  new AddAssetHtmlWebpackPlugin({
     filepath: path.resolve(process.cwd(), './webpack/dll/react.dll.js'),
  }),
  new webpack.DllReferencePlugin({
    manifest: path.resolve(process.cwd(), './webpack/dll/react.manifest.json'),
  })
]
```
AddAssetHtmlWebpackPlugin会将打包后的dll文件插入到HTML中，DllReferencePlugin会在构建的过程中将要打包的模块替换为已经被打包好dll函数。现在每当引用一个dll，就需要添加一遍AddAssetHtmlWebpackPlugin和DllReferencePlugin，在项目中有可能不是使用一个dll, 我们会在dll构建的配置文件的entry中添加多个入口，那在使用的时候就需要动态的引用。代码如下

```js
const fs = require('fs');
const dllPlugins = [];
  const files = fs.readdirSync(path.resolve(process.cwd(), './webpack/dll/'));

  files.forEach((file) => {
    if (/\.dll\.js$/.test(file)) {
      dllPlugins.push(new AddAssetHtmlWebpackPlugin({
        filepath: path.resolve(process.cwd(), './webpack/dll', file),
      }));
    }

    if (/\.manifest\.json$/.test(file)) {
      dllPlugins.push(new webpack.DllReferencePlugin({
        manifest: path.resolve(process.cwd(), './webpack/dll', file),
      }));
    }
  });

  return dllPlugins;
}
```

### 并行打包
当前在使用的过程中都是使用一个进程打包，可以充分利用计算机的性能，进行多进程并行打包，社区中也有解决方案，像happyPack， threadLoad，通过他们的文档可以非常容易的配置使用，我们以happyPack为例
```js
const os = require('os');
const HappyPack = require('happypack');
const happyThreadPool = HappyPack.ThreadPool({ size: os.cpus().length });

const config = {
  plugins: [
    new HappyPack({
      id: 'js',
      threadPool: happyThreadPool,
      loaders: [
        'babel-loader'
      ]
    })
  ],
};
```

这样我们就通过happyPack针对babel-Loader使用多进程并行打包了。

### Magic Comment
在webpack中还有一个比较有意思的点，那就是魔法注释，这一点主要在异步模块的分包加载中使用。通过使用webpackChunkName可以自定义异步模块的文件名，通过
使用webpackPrefetch或者webpackPreload，使得浏览器在网络请求空闲的时候请求，提高代码的加载速度，提升用户的性能体验，使用代码如下：
```js
import React from 'react';

export default {
  path: '/errors',
  routes: [
    {
      path: '/403',
      component: React.lazy(() => import(/*
        webpackChunkName: "errors-403",
        webpackPrefetch: true */'../pages/errors/403'
      )),
    }
  ]
};

```

### 总结
以上就是使用webpack打包构建的主要使用的一些特性，对于提升打包的速度来说，可以使用dll, 并行打包，对于减少代码的体积来说，可以使用代码分割， Tree Shaking。更多的趋势可能是多写异步组件，异步模块，这样才能真正的提高我们系统的第一次打开的性能。

















