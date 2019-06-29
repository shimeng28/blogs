---
layout: react16
title: React Ref的特性
date: 2019-06-01 14:14:32
tags: React
---

在react 16中，设置ref有三种方式

#### 1. 16之前的ref字符串
```jsx
  <span ref="span" >aaa</span>
  
  使用通过 this.refs.span能拿到该ref指向的实例
```

#### 2. 通过ref回调的方式

```jsx
this.textRef = null;

this.setTextRef = (element) => {
  this.textRef = element;
};

<input ref={this.setTextRef} />
```
这样的话，即可以通过this.textRef取到实例。同时，如果在元素里面使用内联函数，element会先为null,之后才是实例。官方的说明是在更新的时候会调用两次，第一次传入为null, 第二次传入的才是实例

#### 3. 最后一种方法是通过createRef

```jsx
import React, { Component } from 'react';

export default class RefLearn extends Component {
  constructor(props) {
    super(props);
    this.state = {
      value: 'aaa',
    };
    this.textRef = React.createRef();
  }

  componentDidMount() {
    this.textRef.current.focus();
    this.refs.span.textContent = 'bbb';
  }

  inputChange = (e) => {
    this.setState({
      value: e.target.value,
    });
  };

  render() {
    const { value } = this.state;
    return (
      <React.Fragment>
        <input 
          ref={this.textRef} 
          value={value} 
          onChange={this.inputChange}
        />
        <span ref="span">aaa</span>
      </React.Fragment>
    )
  }
}
```
通过上述代码可以看出，我们在constructor中通过createRef创建ref，之后将this.textRef赋值给html元素。同样我们也可以将ref赋值给class组件，修改一下代码
```jsx
export default class RefLearn extends Component {
  constructor(props) {
    super(props);
    this.state = {
      value: 'aaa',
    };
    this.textRef = React.createRef();
  }

  componentDidMount() {
    this.textRef.current.focusInput();
    this.refs.span.textContent = 'bbb';
  }

  inputChange = (e) => {
    this.setState({
      value: e.target.value,
    });
  };

  render() {
    const { value } = this.state;
    return (
      <React.Fragment>
        <TextInput 
          ref={this.textRef} 
          value={value} 
          onChange={this.inputChange}
        />
        <span ref="span">aaa</span>
      </React.Fragment>
    )
  }
}
```
这个时候我们通过this.textRef.current拿到的就是TextInput实例，这个时候是没有办法取到dom元素的，因此如果想要实现获取焦点的效果，这个时候就需要TextInput组件实现一个方法focusInput以便被调用。因此TextInput我们可以这样实现
```jsx
class TextInput extends Component {
  constructor(props) {
    super(props);
    this.input = React.createRef();
  }
  static propTypes = {
    value: PropTypes.string,
  };
  static defaultProps = {
    value: '',
  };

  focusInput = () => {
    this.input.current.focus();
  }

  render() {
    const { value, onChange } = this.props;
    return (
      <input 
        ref={this.input}
        value={value} 
        onChange={onChange}
      />
    );
  }
}
```

既然可以将ref赋值给类组件，那么对于函数组件是否也可以呢？使用代码测试一下
```jsx
function Input(props) {
  return <input />
}


export default class RefLearn extends Component {
  constructor(props) {
    super(props);
    this.funcRef = React.createRef();
  }

  componentDidMount() {
    console.log(this.funcRef.current);
  }

  render() {
    return (
      <React.Fragment>
        <Input ref={this.funcRef} />
      </React.Fragment>
    )
  }
}
```
结果为
```js
index.js:1437 Warning: Function components cannot be given refs. Attempts to access this ref will fail. Did you mean to use React.forwardRef()?
```
给一个警告，并且this.funcRef.current为null,也就是ref并未生效。对于函数组件，我们就可以使用Ref转发


#### 4. Ref转发
ref转发就是组件将传递给它的ref转发给它的子组件。修改一下Input组件，代码如下
```jsx
const Input = React.forwardRef((props, ref) => (
  <input ref={ref} {...props} />
));
```
这样就能通过this.funcRef.current访问到input元素了

以上就是React Ref的内容。通过上述也能看出，如果项目已经升级了，大部分情况下我们应该使用React.createRef或者ref回调，需要转发的时候使用React.forwardRef。尽量避免使用ref string。因为在React17中可能要被废弃了。