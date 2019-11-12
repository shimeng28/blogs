---
title: 在react中使用echarts
date: 2019-11-11 20:45:02
tags: React
---
echarts作为一个图表库，在我们的项目中，尤其是B端的项目是缺少不了的。因此，我们这次的商户i版同样也有需要引入图表的地方，于是，我这就先将echarts引入到项目中, 虽然同样有一个echarts-for-react, 不过它还是使用react16之前的方式实现的。我们的项目是基本是使用hooks开发，因此我便针对hooks的版本简单封装了一下。

```jsx
import React, { useRef, useEffect } from 'react';
import PropTypes from 'prop-types';
import * as echarts from 'echarts/src/echarts';

const Chart = React.memo((props) => {});

Chart.propTypes = {
  options: PropTypes.object,
  className: PropTypes.string,
  style: PropTypes.object,
  eventsMap: PropTypes.object
};

Chart.defaultProps = {
  options: {},
  className: '',
  style: {},
  eventsMap: {}
};
```

正常的引入hooks方法, 以及引入echarts主模块，这里为什么引入echarts/src/echarts目录下，在官网能看到有一段关于lib目录下和src目录下的区别

>原因是：目前，echarts/src/** 中是采用 ES Module 的源代码，echarts/lib/** 中是 echarts/src/** 编译成为 CommonJS 后的产物（编译成 CommonJS 是为了向后兼容一些不支持 ES Module 的老版本 NodeJS 和 webpack）。

可以看出为什么要使用src，因为我们的项目是使用webpack打包，src目录下的代码是采用ES Module的源代码，和CommonJS相比，ES Module可以通过Tree Shaking插去未使用的代码。

暴露出了四个属性，options作为图表的配置，className作为额外的类名，style是图表的DOM元素的内联样式，eventMap是我们要在图表上添加的事件。

接下来主要看Chart内部如何实现

```jsx
const Chart = React.memo((props) => {
  const { options, eventsMap, className, style } = props;

  const chartNodeRef = useRef(null);
  const myCharts = useRef(null);

  useEffect(() => {
    const { current: node } = chartNodeRef;

    if (!node) {
      return;
    }

    // 第一次
    if (!myCharts.current) {
      myCharts.current = echarts.init(node);
      for (let [ event, handler ] of Object.entries(eventsMap)) {
        if (typeof event !== 'string' && typeof handler !== 'function') {
          throw new Error('事件类型必须是string类型，事件方法必须是function类型');
        }
        myCharts.current.on(event, handler);
      }
    }

    // options数据更新
    myCharts.current.setOption(options, true);
  }, [ options ]);

  return (<div
    ref={chartNodeRef}
    events={eventsMap}
    style={style}
    className={`chart ${className}`}
  />);
});
```

这里定义了两个ref，一个指向图表的DOM元素，一个作为Echarts的实例, 当第一次渲染的时候，初始化echars以及绑定事件方法。

```js
// options数据更新
myCharts.current.setOption(options, true);
```
setOption有三个传参
>(option: Object, notMerge?: boolean, lazyUpdate?: boolean)
or
(option: Object, opts?: Object)

如果是options的notMerge不为true的情况下，在增加一个维度的数据的时候并不能正常的更新，所以在这里直接就设置为true了。

以上就是针对echarts在react之上的封装，还是相对比较简单的。
    
