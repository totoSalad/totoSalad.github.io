---
title: 如何优雅使用 hooks
date: 2019-11-28 21:11:59
tags: hooks
---

使用了 hooks 一小段时间，时常有非预期的bug，还要小心翼翼担心重复渲染导致的性能瓶颈，令人头发稀少，十分抓狂。本文从函数式组件、性能问题等角度解除大家的疑惑，放心玩耍~
<!-- more -->

![image](https://pic4.zhimg.com/v2-fbb866c98bd27c5a44cad02f4eeb0f1c_1200x500.jpg)

## 我们不一样！函数式组件与 class 组件

首先不得不提 hooks 依附的函数式组件。
两者有什么不同呢？class 组件具体更多的feature，比如生命周期、state。function 组件因为少了这些生命周期会渲染地更快。
然而这些都不是最大的区别，Dan 认为是:

> **Function components capture the rendered values.**

直接翻译是 **function 捕捉的是渲染的值**，说着抽象，show me some code。

```jsx
// function 实现的 ProfilePage
function ProfilePage(props) {
  const showMessage = () => {
    alert('Followed ' + props.user);
  };

  const handleClick = () => {
    setTimeout(showMessage, 3000);
  };

  return (
    <button onClick={handleClick}>Follow</button>
  );
}
```

```jsx
// class 实现的 ProfilePage
class ProfilePage extends React.Component {
  showMessage = () => {
    alert('Followed ' + this.props.user);
  };

  handleClick = () => {
    setTimeout(this.showMessage, 3000);
  };

  render() {
    return <button onClick={this.handleClick}>Follow</button>;
  }
zai
```

两个组件似乎只是同一功能的两种实现，看似没有区别，[live demo](https://codesandbox.io/s/pjqnl16lm7)

执行操作：
1. 选择 profile 为 dan
2. 点击 follow (function) / follow (class)
3. 修改 profile 为 sophie

两个组件 alter 的内容竟然不一致！function 的行为是正确的，user 依旧是点击时的 Dan。而 class 组件因为 `this.props` 是动态变化的，所以获得了更新后的值。

如果希望 class 的行为表现一致的话，则应该按以下的方式实现:

```jsx
// 行为一致的 class 组件实现
class ProfilePage extends React.Component {
  render() {
    // Capture the props!
    const props = this.props;

    // Note: we are *inside render*.
    // These aren't class methods.
    const showMessage = () => {
      alert('Followed ' + props.user);
    };

    const handleClick = () => {
      setTimeout(showMessage, 3000);
    };

    return <button onClick={handleClick}>Follow</button>;
  }
}
```

现在我们可以很好的理解 **function 捕捉的是渲染的值** 了吧~ 

如果希望 function 组件获取最新的 state，不与具体渲染有关。我们也可以借助 useRef hooks 实现类似 **instance field** 的效果

```jsx
// function 实现的 ProfilePage
function ProfilePage(props) {
  const user = useRef(props.user)
  
  useEffect(() => {
    user.current = props.user
  })

  const showMessage = () => {
    alert('Followed ' + user.current);
  };

  const handleClick = () => {
    setTimeout(showMessage, 3000);
  };

  return (
    <button onClick={handleClick}>Follow</button>
  );
}
```

## 令人恐惧？函数式组件的重复渲染

函数组件还有一个特别让人紧张的特性，组件 update 阶段时函数组件会重新执行，函数组件内部的变量、方法均会重新生成。

### Impedance Mismatch （阻碍不匹配）问题处理

setInterval API 在 react 中使用属于一个典型的阻碍不匹配。两者在描述状态变化方便有很大的差异。

* react 渲染结果与变化的状态保持一致
```
// 描述每次渲染
return <h1>{delay}</h1>
```

* setInterval 无法同步状态变化，只能清除重建
```
// delay 发生变化
setInterval(tick, delay);
```

所以在 `react` 中使用 `setInterval`，并且 `delay` 发生变化。class 的处理非常复杂，需要手动反复挂载 `setInterval` 事件。完整对比，[请点击](https://overreacted.io/zh-hans/making-setinterval-declarative-with-react-hooks)

{% asset_img hooks-image1.png quote from http://www.thisgirlcan.co.uk/activities/hooks-image1/ %}

而 hooks 因为函数式组件会重新执行，则不需要手动挂载。

```jsx
function Counter(props) {
  const [count, setCount] = useState(0);
  const savedCallback = useRef();

  // 每次更新组件，callback 重新生成，与不同的 count 绑定。
  function callback() {
    setCount(count + 1);
  }

  // 每次更新时同步 callback 函数，避免 stale state 问题。
  useEffect(() => {
    savedCallback.current = callback;
  });

  // delay 更新时，更新 setInterval 函数。
  useEffect(() => {
    function tick() {
      savedCallback.current();
    }

    let id = setInterval(tick, 1000);
    return () => clearInterval(id); // componentDidUpdate 与 unmount 的情况下销毁现有 interval.
  }, [props.delay]);

  return <h1>{count}</h1>;
}
```

> 值得注意的是每次组件更新时 `callback` 需要重新挂载。因为 `callback` 函数与 `count` 之间形成了闭包关系，每次组件更新时 `callback` 都会重新生成，与最新的 `count` 绑定。

在 setInterval 的场景下，因为 hooks 的组件更新则函数更新的行为，不需要做手动同步，更加自然。

### 直面性能恐惧

让我们来面对我们最担心的性能问题，最大的疑惑主要集中在两方面：

1. 在渲染时创建函数会影响性能？

在现代浏览器中，闭包和类的原始性能只有在极端场景下才会有明显的差别。反倒因为 hooks 基于 `function component` 减少了很多 class component 的开支。


2. 子组件无关渲染如何解决？

如下例子：`handleClick` 函数表现的类似 class 组件中的 inline 函数。

如果 `handleClick` 属性发生变化，子组件 `<Button>` 即便使用了 `React.memo` （类似 pureComponent）也得被迫更新，最终整个组件树发生了更新。

```jsx
function App() {
  const [isOn, setIsOn] = useState(false);
  const handleClick = () => setIsOn(!isOn);  // 每次渲染，重新生成 handleClick
  return (
    <div className="App">
      <h1>{isOn ? "On" : "Off"}</h1>
      <Button handleClick={handleClick} /> 
      // button 因为属性不一样一定会重新渲染
    </div>
  );
}
```

**如何解决**

1. useCallback 与 React.memo 
`useCallback` 指定 depends，可以避免多余渲染。 

```
const handleClick = callBack(() => setIsOn(isOn => !isOn), []);  
// 组件 rerender 时，函数不会再次生成
```

但是对所有的函数使用 `useCallback` 记住旧值，绝对不是一个最佳实践。因为 `useCallback` 本质是一个 memorized 函数，空间换效率，使用过多反倒会造成内存泄漏等问题。
其次如果 `useCallback` 有遗漏的依赖项会出现 stale data 的情况。会在下一章详细介绍。


2. useReducer
  `useReducer` 产生的 `dispatch` 不会发生变化，方便处理深层更新。[相关 issue](https://github.com/facebook/react/issues/14099)

```jsx
const TodosDispatch = React.createContext(null);

function TodosApp() {
  // 提示：`dispatch` 不会在重新渲染之间变化
  const [todos, dispatch] = useReducer(todosReducer);

  return (
    <TodosDispatch.Provider value={dispatch}>
      <DeepTree todos={todos} />
    </TodosDispatch.Provider>
  );
}
```

如果对性能还有疑问请看 [react hooks faq](https://zh-hans.reactjs.org/docs/hooks-faq.html#are-hooks-slow-because-of-creating-functions-in-render)




## hooks 使用
#### instance field
在函数式组件中所有的变量、方法都会在更新时重新赋值，如果期望实现 `this` (instance field)，useRef 可以实现。

> useRef 也可以处理 element ref 的情况。

```jsx
// 将 count 绑定在 this 上
function AutoCount(props) {
  const count = useRef(0)

  // 无需更新的 increase 函数，但是 count 值是最新的
  const increase = callback(() => {
		count.current = count.current++
  }, [])
  

  return (
    <p>{count}</p>
    <Button onClick={increase}>increaee</Button>
  );
}
```

#### stale closures 陷阱
useCallback / useMemo 不是万能的，使用时很容易与 stale state (旧state) 绑定，我们称之为 stale closures 陷阱。

`useCallback` 内部会形成闭包，如果依赖项（第二个参数）不更新， `useCallback` 的闭包也不会更新，最终函数可能执行永远得不到我们期望的结果。

```jsx
const [isOn, setIsOn] = useState(false);
// 错误写法！
// 因为 dependence 是 [], 组件 rerender 时，callback 函数不会更新
// 闭包不更新，isOn 也永远是 false
const handleClick = callBack(() => setIsOn(!isOn), []);  
```
可以把闭包值作为 `callback` 依赖项，在需要时更新闭包。

```jsx
// 正确写法 1
const handleClick = callBack(() => setIsOn(!isOn), [isOn]);  
```

也可以利用 useState 的另一种用法，避开使用闭包值。

```jsx
// 正确写法 2
const handleClick = callBack(() => setIsOn(isOn => !isOn), []);  
```

> eslint-plugin-react-hooks 会校验 useCallback / useEffect 的 dependence  https://github.com/facebook/react/issues/14920


#### 其他注意点
* `const [arr, setArr] = useState()` 的 `setArr` 需要返回一个全新的数组否则组件不会发生更新，应该是 hooks 内部做了优化会做一层浅比较。

* 使用 `useReducer` 时，reducer 不能放在自定义 hooks 内部，否则 action 会触发多次.[相关segmentfault](https://stackoverflow.com/questions/54892403/usereducer-action-dispatched-twice?answertab=oldest#tab-top)   

**错误写法**

```
import { useReducer } from "react";

export function useApiCallReducer() {
  // 因为放在函数内部，useApiCallReducer 执行时 reducer 就会更新
  function reducer(state, action) {
   switch (action.type) {
      ...
    }
  }
  return useReducer(reducer, initialState);
}
```

**正确写法**

```
import { useReducer } from "react";

// 放在函数外部，reducer 保持不变
function reducer(state, action) {
 switch (action.type) {
   ...
 }
}
```


## 总结

react hooks 在 react 内部实现了组件逻辑复用，避免了相关逻辑被生命周期割裂的情况。但是 hooks 依附函数式组件，更新方式与我们常用的 class 组件不一样。
我们可以不需要担心函数式组件更新时需要重新初始化函数，会影响性能。在必要时可以采用 `useCallback` + `React.memo` / `useReducer`。但是还是那句老话，过早优化是万恶之源。
同时 hooks 也带来了新的问题，instance 使用 、stale data 、除此之外 hooks 内部做了很多所谓的优化，导致了在实践中有很多坑。


## 其他

业界目前木有 hooks 最佳实践，本文不敢夸口最佳实践，只是给出 hooks 使用的注意点，希望能帮助大家优雅使用 hooks ~ 在使用中遇到问题可以在 [相关链接](http://git.caimi-inc.com/huluman/doc/issues/73) 记录。



## 资料

- [how-are-function-components-different-from-classes](https://overreacted.io/how-are-function-components-different-from-classes/)
- [如何錯誤地使用-react-hooks-usecallback-來保存相同的-function-instance](https://medium.com/@as790726/%E5%A6%82%E4%BD%95%E9%8C%AF%E8%AA%A4%E5%9C%B0%E4%BD%BF%E7%94%A8-react-hooks-usecallback-%E4%BE%86%E4%BF%9D%E5%AD%98%E7%9B%B8%E5%90%8C%E7%9A%84-function-instance-7744984bb0a6)
- [阻碍不匹配](https://haacked.com/archive/2004/06/15/impedance-mismatch.aspx/)

