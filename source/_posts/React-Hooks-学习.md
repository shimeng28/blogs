---
title: React Hooks 学习
date: 2019-06-30 17:36:09
tags: React 
---

在React16.8之前，虽然同样有函数组件，不过只是作为无状态组件的一种使用方法，对于需要涉及到React生命周期函数，维护组件内在的状态的能力就需要使用类组件了。并且React有别于其他框架的特点一方面是其可以使用标准化的声明式的模版语言jsx，另一方面就是其的组件化，也就是说涉及到组件的封装和复用，封装就是内部维护自己的状态，这就需要有state的能力，之前的复用一种是通过高阶组件，另一种是通过props，这两种的复用方式并没法复用组件内部的逻辑状态，因为我们没有一种合适的方法把逻辑状态抽离出来。因此基于此React推出了React hooks。使得同样在开发的过程中可以通过自定义hooks复用逻辑状态，其赋予了函数组件类组件的能力，包括state，生命周期函数，获取context, ref等。

### 所有的hooks
  1. useState
  2. useEffect
  3. useContext
  4. useReducer
  5. useLayoutEffect
  6. useCallback
  7. useMemo
  8. useRef
  9. useImprerativeHanle
  10. useDebugValue 
  
#### useState
先来一段代码看看
```jsx
import React, {
  Fragment, useState
} from 'react';

export default () => {
  const [name, setName] = useState('jokking');
  const [count, setCount] = useState(0);
  
  return (
    <Fragment>
      <input 
        type="text"
        value={name}
        onChange={(e) => setName(e.target.name)}
       />
      <p>My name is {name} </p>
      <div onClick={() => setCount(count + 1)}>button {count}</div>
    </Fragment>
  );
};
```
上面就是使用hooks实现的一个简单的demo，通过useState传入一个initialValue， 返回的一个是state的值name，一个是更新state的方法setName。同样可以使用多个useState，react会根据useState调用的顺序知道每个state，这也是为什么需要把hooks函数放到函数的顶层，而不要放到if判断语句以及子函数中，因为其调用顺序不可控来。

我们看到我们上面代码使用了内联回调函数, 每一次父组件的更新都会造内联回调函数都会不一样，就会触发子组件的更新。把上面的代码再修改一下可能就能看出是什么意思了。

```jsx
import React, {
  Fragment, useState
} from 'react';
const Count = (props) => {
  useEffect(() => {
    console.log('Count update');
  });

  const updateCount = () => {
    props.onClick();
  };

  return (
    <div onClick={updateCount}>button {props.count}</div>
  );
};

export default () => {
  const [name, setName] = useState('jokking');
  const [count, setCount] = useState(0);
  
  // const inputChange = useCallback((e) => {
  //  setName(e.target.name + count);
  // }, [count]);
  
  // const countClick = useCallback(() => {
  //  setCount(count + 1);
  // }, []);
  
  return (
    <Fragment>
      <input 
        type="text"
        value={name}
        onChange={inputChange}
       />
      <p>My name is {name} </p>
      <Count 
        count={count}
        onClick={() => setCount(count + 1)} 
       />
    </Fragment>
  );
};
```
上面的代码把展示count的div抽离出去新建一个Count组件，通过内联回调函数() => setCount(count + 1)传递给Count组件，这时如果input元素更新，会造成Count的useEffect也执行。对于input元素也是同样的道理，如果我们把注释的地方取消注释，每一次都会是memorized函数，只有依赖项更改后才会更新。

现在我们又使用了两种hooks函数，useCallback和useEffect, 通过之前代码也能有个理解了，对于useCallback来说，是对子组件回调函数的一种记忆更新，也就是说除非依赖项更新，否则每次更新返回的回调函数都是一样的，这对于通过比较props是否相等来更新组件从而优化性能是非常适合的一种法子。

那么对于useEffect呢？它是在渲染完成之后执行副作用的，它的第二个参数是它的依赖项，只有在依赖项更新的之后才会调用effect函数。那么由此而产生了一种便是传入空数组，岂不就是相当于只会在componetDidMount 和 componentWillUnmount的时候调用，虽然是这个道理，但是需要注意的是如果传入空数组，那么useEffect函数中的props和state都只是最初始的值。

现在继续看剩下的hooks，先来一段代码
```jsx
import React, { 
  createContext, forwardRef,
  useState, useEffect, useCallback, 
  useContext, useRef, useImperativeHandle,
} from 'react';

const MyNameContext = createContext('hello');

const Test = forwardRef((props, ref) => {
  const context = useContext(MyNameContext);
  
  useImperativeHandle(ref, () => ({
    getContent() {
      console.log('ref content');
    }
  }));
  
  return (
    <p>{context}</p>
  );
});


export default () => {
  const [name, setName] = useState('jokking');
  const [count, setCount] = useState(0);

  const ref = useRef();
  
  useEffect(() => {
    console.log('invoke ref method');
    ref.current.getContent();
  }, [name]);
  
  const countFeature = useMemo(() => {
    if (count % 2) {
      return 'odd';
    } 
    return 'even';
  }, [count]);

  const inputChange = useCallback((e) => {
    setName(e.target.name);
  });

  return (
    <>
      <input 
        type="text"
        value={name}
        onChange={inputChange}
      />
      <p>My name is {name} {countFeature} </p>
      <MyNameContext.Provider value={name}>
         <Test ref={ref}>
      </MyNameContext.Provider>
    </>
  );
};
```
我们先是通过createContext生成了一个context，在Test自定义组件中使用了useContext消费了context数据。这里我们就能看出来useContext的作用了，相当于是ClassComp.contextType = TestContext。在之后通过useRef生成一个ref对象，并且将ref指向Test组件，由于Test组件是一个函数组件，并没有实例this对象不存在。我们又通过useImperativeHandle自定义暴露给父组件的实例方法。useMemo返回一个变量，会参与到组件渲染中，主要作用就是优化渲染时的计算，除非依赖项更新，否则不会重新计算。

对于useLayoutEffect来说，其和useEffect传参和定义是一样的，只是执行的时机不一样。useLayoutEffect和componentDidUpdate和componentDidMount的时机是一样的，是在Dom变更后立即同步执行，会阻塞页面渲染，而useEffect在每轮渲染结束后执行


最后来看一下useReducer
```jsx
import React, {
  useReducer
} from 'react';

const initialState = {
  name: 'jokking',
  age: 24,
};

function reducer(state, action) {
   switch(action.type) {
     case 'addAge': 
       return {
         ...state,
         age: ++state.age,
       };
     case 'modifiedName':
       return {
         ...state,
         name: action.payload,
       };
   }
}

export default () => {
  const [state, dispatch] = useReducer(reducer, initialState);
  
  return (
    <>
      <p>name: {state.name}, age: {state.age}</p>
      <button onClick={() => dispatch({type: 'addAge'})}>increment age</button>
      <input 
        type="text"
        value={state.name}
        onChange={(e) => {
          dispatch({
            type: 'modifiedName',
            payload: e.target.value,
          })
        }}
      />
    </>
  );
}
```

使用useReducer返回一个state， 以及一个dispatch，每次更新都会调用dispatch，从而通过reducer函数生成新的state。useState和useReducer来说，useState只是useReducer的一种特殊情况。

  
以上就是react hooks的内容。 也许有一天React框架慢慢的落寞了，但是其中使用的原理，思想一定会被继承下来