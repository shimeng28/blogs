---
title: 在Electron中实现JS Bridge容器
date: 2020-06-06 12:28:26
tags: Electron
---
这是工作中分享的一篇文章，替换了代码，只保留了技术实现。

容器的底层实现都是一致的，都是通过webview。如下所示就是一个简单的webview组件。

```html
  <webview src={url} preload={sdk} />
```
preload的内容会在嵌入页面加载前加载执行，因此可以在这个地方将JS Bridge需要的内容注入进去(ps: 这里有个点需要注意的就是preload的内容只能是一个本地文件路径)。


### 基础实现

JS Bridge的基础是基于两个方法，一个方法作为客户端容器来实现，作用是内嵌页面调用客户端能力；一个方法是内嵌页面自己来实现，用来让客户端向内嵌页面传递数据。我们可以自己命名，例如可以分别是execNative和_handleFromNative，都挂载到window.JSBridge对象上。_handleFromNative是内嵌页面自己实现的，作为客户端只需要实现execNative就可以了。

从客户端的角度来思考🤔，有两个场景，内嵌页面通过execNative调用我们支持的方法，比如请求，调用客户端功能；和客户端向内嵌页面触发一些事件, 比如状态数据的变化。

第一种情况是通过execNative调用的时候将方法名称、参数以入参的形式传递过来，而客户端再通过_handleFromNative将结果以入参的形式回传过去就可以啦。

而第二种向内嵌页面触发事件，这里先简单介绍一下Electron的机制，Electron有两种进程，主进程和渲染进程，渲染进程就是窗口，webview是作为一个组件放到渲染进程的，在整个Electron过程中，会有一个主窗口一直存在，这里可以先认为webview就是放到主窗口里面的。webview向内嵌页面注入代码可以通过webview.executeJavaScript(code);

而向webview内嵌页面传递信息可以通过调用_handleFromNative, 如下

```js
function execJs = (webview, eventName, eventArgs) => {
  if (!webview) return;
  const code = `
    if (window.JSBridge._handleFromNative) {
      window.JSBridge._handleFromNative({
        name: ${eventName},
        args: ${JSON.stringify(eventArgs)}
      });
    }
  `;
  webview.executeJavaScript(code)
}
```

向内嵌页面触发事件非常适合一种设计模式，那就是订阅发布模式

```js
forIn(WEBVIEW_EVENT, (name) => {
  WebviewEvent.getInstance().on(name, (webview, eventName, eventArgs) => {
    execJs(webview, eventName, eventArgs);
  });
});
```

WEBVIEW_EVENT包含所有事件,通过订阅的回调函数可以看出，在触发事件的时候是传递webview实例、事件名称和事件内容。 真实调用如下:

```js
WebviewEvent.getInstance().emit(
  webview,
  WEBVIEW_EVENT.TEST,
  {
    value: 'test'
  }
);
```
### webview事件
在我们的程序中，webview的事件调用中有两个事件需要特别了解 dom-ready 和 did-finish-load, dom-ready是说明内嵌页面dom加载完毕, 只有在这个事件时候才可以调用webview上的方法，如webview. executeJavaScript 向内嵌页面执行代码。而did-finish-load则是表明整个内嵌页面已经加载完成，完全渲染出来之后。

那这就会有一个问题，内嵌页面未加载完全，我们就向内嵌页面触发事件了。因此这就需要加一个使用一个队列，在内嵌页面未完全加载之前，将事件名称和参数入队列，在did-finish-load的时候再执行这一系列事件。 代码如下
```js
const webview = document.querySelector('webview');
const emitEvent = ({ eventName, eventArgs }) => {
  if (!ready) {
    prependQueue.push({ eventName, eventArgs });
    return;
  }
  WebviewEvent.getInstance().emit(webview, eventName, eventArgs)
};
```
在触发事件的时候如果没有ready，则进队列。
```js
  webview.addEventListener('did-finish-load', () => {
    forEach(prependQueue, ({ eventName, eventArgs }) => {
      WebviewEvent.getInstance().emit(webview, eventName, eventArgs)
    })
  });
```
触发did-finish-load的时候将未执行的事件执行完毕并清空队列。

### 多窗口多webview

在实际的使用中，不可能只是在主窗口有一个webview, 因此这就引发一个问题，整个app中有多个包含webview组件的页面，甚至于一个页面中包含了多个webview组件。如上面所说的向webview中推送事件的时候，最合适的方式就是在一个地方统一管理，也就是放在主渲染进程，主窗口。一是因为主窗口会一直存在，另一个是app中的数据状态都可以通过主窗口监听到。但是因为webview也在渲染进程，从一个渲染进程发送消息到另一个渲染进程，我们是没有办法直接通信，必须要走主进程中转。因此我们的实现大致如下

在主进程维护一个Map映射表，在webview所在组件的挂载的时候注册，卸载的时候移除。当在另一个渲染进程向webview所在渲染进程发送消息，只需要通过ipcRender.send触发事件就可以。

### 注册自定义scheme

我们的jsb使用了一些自定义的协议，在Electron中注册自定义scheme的方法如下
```js
protocol.registerHttpProtocol('aaa', (request, callback) => {
  const url = request.url;
  console.log('url...', url);
}
```

protocol是Electron的一个模块, 协议的注册必须放到主进程，主要原因有两点， 一是如果放到渲染进程，为了拦截webview内嵌页面的所有自定义协议请求，必须放到preload中，那么就把所有渲染进程中的自定义协议请求都会在第一个webview的preload中被拦截了，想要每个渲染进程处理自己的自定义协议无法完美实现。第二个原因就是会阻塞did-finish-load事件的触发，延长了webview中内嵌页面的加载时间。
