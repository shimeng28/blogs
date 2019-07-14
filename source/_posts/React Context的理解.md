---
layout: react16
title: React Context的理解
date: 2019-06-08 20:14:06
tags: React
---

React中父子组件的数据通过单向props传递，组件内部数据通过state维护。对于子组件可以通过回调函数向父组件传递数据，例如事件onChange回调。而像只有孙组件使用祖父组件的数据时，这时可以将孙组件提取出来和数据一起通过祖父组件传递下去。但是对于多个层级各不相同的组件共享一些数据的时候，就需要context来协助了。

本文仅考虑React16之后的写法，之前的最终都是要被废弃了，需要的时候可以看下官网。

第一种为类组件的时候

```jsx
const ChildContextDemo = React.createContext('default');
```

通过React.createContext创建一个context对象，传递的值即为defaultValue。

```jsx
class Parent extends Component {
  constructor(props) {
    super(props);

    this.state = {
      childContext: '123',
    };
  }
  render() {
    return (
      <Fragment>
        <label>childContext:</label>
        <input
          type="text"
          value={this.state.childContext}
          onChange={e => this.setState({ childContext: e.target.value })}
        />
        <ChildContextDemo.Provider value={this.state.childContext}>
          {this.props.children}
        </ChildContextDemo.Provider>
      </Fragment>
    )
  }
}

export default () => (
  <Fragment>
    <Parent>
      <Child />
      <Child2 />
    </Parent>
  </Fragment>
)
```
Child, Child2组件在后面有其定义。对于context的提供者来说，通过Provider的value值，设置context的值，而子组件如何使用context值呢？以下代码展示如下：

```jsx
class Child extends Component {
  static contextType = ChildContextDemo;
  render() {
    return (
      <div>{this.context}</div>
    )
  }
}
```

通过将ChildContextDemo赋值给Child的静态属性contextType，我们即可在任一声明周期（包括render方法）通过this.context访问到context数据。那么对于函数组件呢？我们如何消费context的数据？

```jsx
const Child2 = () => (<ChildContextDemo.Consumer>
  {value => (
    <span>Function Context: {value}</span>
  )}
</ChildContextDemo.Consumer>)
```

因此可以看出，对于函数式组件来说，通过Consumer包裹，将context数据传递下来。

通过上面我们也能看出，对于多个context的，同样可以通过这种方式来使用
```jsx
const Child3 = () => (
  <ChildContextDemo2.Consumer>
    {
      value2 => (<ChildContextDemo.Consumer>
        {value => (
          <span>Function Context: {value}</span>
        )})
    }
  </ChildContextDemo2.Consumer>
)
```

