---
title: Jade模板学习
date: 2018-05-23 18:49:56
tags: 模板引擎
---
当前流行的MVVM框架如Vue、React适合作为单页应用，但是当遇到SEO问题的时候就不是特别友好。作为js模板引擎，学回了一个，其他的用法大致都一样。

```jade
!doctype html
html
  head
    meta(charset="utf-8")
    title  jade learning
  body
    h1   jade leanring
  </html>
```
以上代码编译后为
```html
!doctype html
<html>
  <head>
    <meta charset="utf-8"/>
    <title>jade learning</title>
  </head>
  <body>
    <h1>jade leanring</h1>
  </body>
</html>
```
因此很容易看出，其和HTML非常类似。而想元素的属性添加
```jade
  h1(class="jade", id="jade-learning" data-uid="aaa") jade leanring
```
则会编译为
```html
<h1 id="jade-learning" data-uid="aaa" class="jade">jade leanring</h1>
```
因此在元素后面紧跟着用括号括起来的就是各个元素的属性，而像class和id，由于其比较特殊，有简便 的方式
```jade
h1.jade#jade-learning jade leanring
```
编译为: ```<h1 id="jade-learning" class="jade"> jade leanring</h1>```

如果想要注释代码，有两种方式单行注释和多行注释，都是`//`，
```jade
 //
      h1.jade#jade-learning 
      h1(class="jade", id="jade-learning" data-uid="aaa") jade leanring
      
//  h1.jade#jade-learning
```
编译后的结果就是
```html
<!--
    h1.jade#jade-learning 
    h1(class="jade", id="jade-learning" data-uid="aaa") jade leanring
    -->
    
  <!-- h1.jade#jade-learning -->
```
当如果不想让编译后的文件出现注释, 则用`//-`

模板引擎主要的作用是减少重复，因此，当然就会有代码块
```jade
block desc
  p desc from test
```
如果我们想要继承该代码块，可以使用`extends filename`,将该模板文件继承过来即可。

另一种用法是mixin。如果我们创建一个mixin，即可以一直复用
```jade
mixin student(name, courses)
  p #{name}
  ul.courses
     each course in courses 
       li= course
    
    - var name = "Jerery"
    - var courses = ["jade", "node"];
    +student(name, courses)
```
编译后代码为
```html
<p>Jerery</p>
  <ul class="courses">
    <li>jade</li>
    <li>node</li>
  </ul>
```
`-`是作为定义一个js变量，在后面的使用中我们可以通过`#{name}`以及直接在元素后面用`=name`,这两者的区别就是第一种会转义其值，第二种不会转义。mixin用法可以在后面直接调用，和代码块的区别就是mixin可以传入参数。

同样我们也可以使用if-else if -else语句
```jade
- var lessons = ['jade', 'node']
    if lessons
      if lessons.length > 2
        p more than 2: #{lessons.join(", ")}
      else if lessons.length > 1
        p more than 1: #{lessons.join(", ")}
      else
        p no lesson
    else 
      p no lesson
```
编译后为`<p>more than 1: jade, node</p>`

我们也可以使用unless语法，就是其代表的含义就是如果不为真。
```jade
unless isImooc
  p no imooc
```
```html
<p>no imooc</p>
```

同样也可以使用case语句
```jade
-var name = "jade"
case name
  when "node"
    p Hi node!
  when "jade": p Hi jade!
  default: p Hi Welcome
```
```html
    <p>Hi jade!</p>
```

最后介绍的一个特性就是filter特性，首先我们下载markdown包`cnpm install markdown -g`
```jade
:markdown
   Hi this is **Jade**	
```
转义后代码为
```html
<p>Hi this is <strong>Jade</strong></p>
```

以上就是`jade`的基本语法，所有的模板引擎都大致差不多，学完一个之后，其余的都差不多了。
