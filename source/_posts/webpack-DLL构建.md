---
title: webpack DLL构建
date: 2019-10-04 11:52:26
tags: 前端工程化
---
DLL是起源于微软的windows上，即Dynamic Library Link,翻译过来就是动态链接库，其包含多个程序同时使用的代码和数据，主要是在Windows系统可以实现共享函数库。那么在webpack中的DLL和Windows的DLL作用是类似的，是将构建过程中多个模块同时使用的类库抽离出来，通过暴露出一个dll方法，实现在构建的过程中实现共享，从而提升构建速度。

既然是要将类库抽离出来，最简单的一种方式就是在entry中增加入口，但是这样和业务代码的构建耦合。因此需要单独抽离出一份配置文件，专属DLL构建。

```js
module.exports = {
  mode: 'production',
  entry: {
    react: [
      'react'
    ]
  },
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

以上就是一份基本的DLL配置，在入口文件中entry中配置了react类库，在output中将react打包为一个js文件, 通过在output.library暴露出dll方法。在plugins中，使用DllPlugin生成的是一份映射文件, 映射文件内容如下

```json
{
  "name":"react",
  "content":{
    "./node_modules/react/index.js": {
      "id": 1,
      "buildMeta":{
        "providedExports":true
      }
    },
  }
}
```
这样针对一个DLL构建就完成了，那如何在我们项目的webpack配置中使用呢？可以细想一下，一个js文件，一个json文件。js文件是源码构建后的内容，这部分整个项目都需要，因此我们需要挂载到html文件内。json文件属于一份映射关系，在构建的过程中需要，而这份工作webpack为我们提供了一个插件DllReferencePlugin。

因此使用只需要在我们项目的webpack配置文件的plugins中添加两个插件就可以了。

```js
plugins: [
  new AddAssetHtmlWebpackPlugin({
    filepath: path.resolve(process.cwd(), './webpack/dll/react.dll.js'),
  }),
  new webpack.DLLReferencePlugin({
    context: path.join(__dirname),
    manifest: path.resolve(process.cwd(), './webpack/dll/react.manifest.json'),
  })
]
```

以上内容便是我们使用webpack dll构建所用到的内容。

