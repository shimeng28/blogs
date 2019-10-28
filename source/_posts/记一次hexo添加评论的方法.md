---
title: 记一次hexo添加评论的方法
date: 2019-07-21 15:20:01
tags: 代码人生
---

### 注册

先到[https://github.com/settings/applications/new](https://github.com/settings/applications/new) 上面注册， Homepage URL 和 Authorization callback URL 填写自己的域名，像我的就是 http://shimeng.info/  之后会生成一个  client ID 和一个 client secret。

### 主题文件下配置

在你使用的主题的目录下的_config.yml文件下配置信息。我的就是在 themes/next/\_config.yml文件里面。

```yml
# Gitment
# Introduction: https://imsun.net/posts/gitment-introduction/
# You can get your Github ID from https://api.github.com/users/<Github username>
gitment:
  enable: true
  mint: true # RECOMMEND, A mint on Gitment, to support count, language and proxy_gateway
  count: true # Show comments count in post meta area
  lazy: false # Comments lazy loading with a button
  cleanly: false # Hide 'Powered by ...' on footer, and more
  language: # Force language, or auto switch by theme
  github_user: shimeng28 # MUST HAVE, Your Github ID 
  github_repo: blogs # MUST HAVE, The repo you use to store Gitment comments
  client_id:  # MUST HAVE, Github client id for the Gitment
  client_secret:  # EITHER this or proxy_gateway, Github access secret token for the Gitment
  proxy_gateway: # Address of api proxy, See: https://github.com/aimingoo/intersect
  redirect_protocol: # Protocol of redirect_uri with force_redirect_protocol when mint enabled

```

这上面主要修改的是 enable 设置为true。client_id和client_secret就是刚才得到的两个值。github_repo是你准备作为评论的仓库。 github_user虽然说是要你填github Id,但是实际上是你的github name。 因为需要github_user的内容去向github请求issues内容。github提供的API接口是：

```yml
GET /repos/:owner/:repo/issues
```

其实它就是通过这个接口请求issue内容作为评论内容，路由参数owner就是我们的github name，填写github id是请求不成功的。像我的请求的URL就是 

```js
https://api.github.com/repos/shimeng28/blogs/issues
```

### 接口跪了

到这的时候按道理就是可以了。可是当你hexo d -g部署之后，点击登入会发现有个弹窗报错 [object ProgressEvent]。如果我们查看网络请求面板就会发现是https://gh-oauth.imsun.net这个接口挂了。 具体原因直接访问这个地址就能看出来了是https证书过期了。由于这个服务的提供者不维护了，那就只能靠我们自己了。不过这个接口的提供者开源了他的服务代码[https://github.com/imsun/gh-oauth-server](https://github.com/imsun/gh-oauth-server)。 这个服务的内容就是向 https://github.com/login/oauth/access_token 转发数据。解决方案看个人吧，如果自己有一台服务器可以把这个服务部署上去替换一下就好了。因此自己就部署了一下，访问接口是/api/oauth;

同样发现https://imsun.github.io/gitment/dist/gitment.browser.js不可用，都部署到自己到服务器上

具体的解决方法就是在 themes/next/layout/\_third-party/comments/gitment.swig 文件内部 将

```js
 https://imsun.github.io/gitment/dist/gitment.browser.js
```

替换为

```js
/static/gitment.browser.js
```
就可以了。这个时候我们再通过 hexo clean; hexo d -g; 清楚缓存再重新部署发现就能登陆了。

### 文件名太长

登陆之后会展示 初始化创建 按钮，点击就会创建针对上面填写的repo创建一个issue。但是突然你会发现又出问题了。具体看下就是Error: Validation Failed.错误码 422。GitHub提供的接口是非常符合RESTFul的。422 是请求格式错误，具体搜索一下就会发现前人已经踩过坑了。那就是传输的数据太长，这个数据就是你的日期 + 博客名，它的限制是50个长度。最简单的思路就是重新命名，我看了下我的博客，基本上都超过了这个限制。

这应该怎么办呢？突然在一个issue里有个人的思路非常棒，那就是将数据转MD5，之后长度就是32个，完全在50内。

做法是 通过[https://www.bootcdn.cn/](https://www.bootcdn.cn/) 找到blueimp-md5 这个开源库的cdn路径，然后放到博客里面。我的是在themes/next/layout/\_scripts/pages/post-details.swig这个文件里，添加一个script标签

```
<script src="https://cdn.bootcss.com/blueimp-md5/2.11.0/js/md5.js"></script>
```

然后在themes/next/layout/\_third-party/comments/gitment.swig文件内 

```
function renderGitment(){
  var gitment = new {{CommentsClass}}({
      id: md5(window.location.pathname), 
      owner: '{{ theme.gitment.github_user }}',
      repo: '{{ theme.gitment.github_repo }}',
      {% if theme.gitment.mint %}
      lang: "{{ theme.gitment.language }}" || navigator.language || navigator.systemLanguage || navigator.userLanguage,
      {% endif %}
      oauth: {
      {% if theme.gitment.mint and theme.gitment.redirect_protocol %}
          redirect_protocol: '{{ theme.gitment.redirect_protocol }}',
      {% endif %}
      {% if theme.gitment.mint and theme.gitment.proxy_gateway %}
          proxy_gateway: '{{ theme.gitment.proxy_gateway }}',
      {% else %}
          client_secret: '{{ theme.gitment.client_secret }}',
      {% endif %}
          client_id: '{{ theme.gitment.client_id }}'
      }});
  gitment.render('gitment-container');
}
```

将window.location.pathname的值转MD5，然后在赋值给id。之后就一切大功告成了。不过唯一的糟点就是每篇博客都需要点击 初始化创建 别人才能评论。