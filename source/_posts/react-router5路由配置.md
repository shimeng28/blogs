---
title: react-router5路由配置
date: 2019-09-26 21:30:17
tags: React
---
最近在做商家中心的移动端，主要负责的就是webpack编译，路由，以及移动端适配和echarts的引入。本文主要作为引入react路由的方法。在React中，路由需要我们自己去添加，相应的模块为react-router。react-router已经升级到5，一切皆是组件的思想更加的明显，并且其针对react native和i版作了区分。对于本次来说，我们只需要引入react-router-dom即可，相应的安装就是``` yarn add react-router-dom ```。

根据[react-router官方文档](https://reacttraining.com/react-router/web/guides/quick-start)的指引，最终的路由模块应该是这样：

```js
<BrowserRouter>
  <React.Suspense fallback={fallback}>
    <Switch>
      <Route 
        key={'home'} 
        path={'/home'} 
        render={(props) => <Home {...props} />}
      />
      
      <Route 
        key={'login'} 
        path={'/login'} 
        render={(props) => <Login {...props} />}
      />
    </Switch>
  </React.Suspense>
</BrowserRouter>
```
可以先不用管React.Suspense, 在一个大的项目中，如果我们真的按照这样配置路由的话，那么最终的结果就是这个文件会非常的长，因为所有的路由都要写在这么一个文件中, 并且如果再在每个路由中加上权限判断，那么整个结构就非常的乱了。因此需要做成配置化的路由。那么当配置化的路由是什么样子呢？(ps: 数据均为伪造的)
```js
import React from 'react';

export default {
  path: '/errors',
  access: (perm) => Number(perm.ERROR_PAGE) === 1,
  component: React.lazy(() => import('../pages/errors/index')),
  routes: [
    {
      path: '/404',
      component: React.lazy(() => import('../pages/errors/404')),
    },
    {
      path: '/403',
      component: React.lazy(() => import('../pages/errors/403')),
    }
  ]
};
```

如上这是一个配置好的路由，下一次如果想要添加其他的错误页面如402， 只需要在添加两行代码就行

```js
{
  path: '/402',
  component: React.lazy(() => import('../pages/errors/402')),
}
```

是不是很简单了，当然啦，配置的时候简单了，就需要增加如何将配置化的路由转成react-router-dom需要的路由格式。

第一步：每一个一级路由都应该是一个单独的路由文件，旗下有不同的多级路由，因此我们需要先把所有的一级路由文件引入到一个地方。如下

```js
import userRoute from './user';
import errorsRoute from './errors';
import homeRoute from './home';

const routesList = [
  userRoute,
  errorsRoute,
  homeRoute
];

const wrapRoutesList = wrapRoute(routesList);
```

应该通过wrapRoute将刚才配置好的路由转换成下面这样格式

```js
[
  { 
    path: "/errors/404"
    component: Component404,
  },
  { 
    path: "/errors/403"
    component: Component403,
  },
  { 
    path: "/"
    component: Home,
  },
]
```

那我们来看下wrapRoute如何实现

```js
const wrapRoute = (routesList) => {
  const wrapRoutesList = [];
  routesList.forEach((routeItem) => wrapRoutesList.push(..._wrapRoute(routeItem)));
  return wrapRoutesList;
};

// 添加分割线
const addSlash = (pagePath) => pagePath.startsWith('/') ? pagePath : `/${pagePath}`;

// 包装路由
const _wrapRoute = (route, parentRoute = { path: '' }) => {
  const parentPath = addSlash(parentRoute.path);
  const currentPath = addSlash(route.path);
  const path = `${parentPath}${currentPath}`.replace(/\/\//g, '/');
  route.path = path;

  // 子路由继承父路由的权限
  if (!('access' in route) && ('access' in parentRoute)) {
    route.access = parentRoute.access;
  }

  // 是否强制重定向
  if (route.forbiddenRedirect && typeof route.forbiddenRedirect !== 'string') {
    throw new TypeError('forbiddenRedirect 必须是字符串链接');
  }

  let routesList = [];

  // 递归地包装子路由
  if (route.routes && route.routes.length) {
    for (const childRoute of route.routes) {
      routesList.push(..._wrapRoute(childRoute, route));
    }
  }

  if (route.component) {
    routesList.push(route);
  }

  return routesList;
};
```

通过一个递归的方法，依次深度遍历路由，最终将其放到一个List里面。现在还只是一个List的形式，需要改成组件的形式，看接下来的代码

```js
<BrowserRouter>
  <React.Suspense fallback={fallback}>
    <Switch>
      {renderRoutes(wrapRoutesList, userInfo)}
    </Switch>
  </React.Suspense>
</BrowserRouter>
```
这个时候就和最开始react-router-dom需要的一样了，那么我们就剩下最后一个renderRoutes方法了

```js
// 渲染路由方法
const renderRoutes = (routes, userInfo = {}, switchProps = {}) => routes
  ? (
    <Switch {...switchProps}>
      {
        routes.map((route, i) => {
          const { key, path, exact = true, strict, component: RouteComponent, access } = route;
          return (
            <Route
              key={key || i}
              path={path}
              exact={exact}
              strict={strict}
              render={(props) => {
                // 是否有权限
                const hasRight = (access instanceof Function) ? access(userInfo) : true;
                return (
                  hasRight
                  ? <RouteComponent {...props} route={route} />
                  : <Redirect to={{
                    pathname: routeInfo.forbiddenRedirect || FORBIDDEND_PAGE,
                    state: { from: props.location }
                  }} />
                );
            }}
            />
          );
        })
      }
      <Route component={NotFound} />
    </Switch>
  )
  : null;
```
renderRoutes就将我们的List形式的路由配置渲染出来了，整个的路由配置也就完成了。支持多级路由，支持权限配置。
