---
title: React Hooks 能给我们带来什么
date: 2019-07-18
categories: ["前端"]
tags: ["React", "react-redux", "JavaScript", "TypeScript"]
---

# TL;DR

很多 React 的新技术，我们大多数人也只是听说而已，实际上投入生产的机会非常少。首先很多旧项目不会去升级 React 的版本，很多团队因为不精通技术而且只需要专注于做业务，所以也没有这样的成本来升级 React 的版本。而我刚好遇到了好的机会，让我有足够的技术自由度，我也趁着一个重构的机会，顺便把负责组件全部用 hooks 重构了。这次重构是在 2019 年 6 月下旬开始的，并且也要感谢 react-redux 在 6 月上旬的时候发布了 v7.1.0，Hooks API 正式投入生产了，让我能够大展身手。

这篇文章不会涉及到 React 中所有 Hooks 的细节，而是就开发而言谈谈常用的一些 Hooks，特别是最近推出的、让 redux app 能够使用 functional component 替代 connect 的 Hooks。

# Before & After Hooks

使用 Hooks 对代码的修改不少，但是完成重构后代码看着会非常简洁，因为少了非常多的代码，我们维护一个组件只需要看一个 function 即可。

## Before

```TypeScript
/** container.ts */
import { connect } from 'react-redux'
import { injectIntl } from 'react-intl'
import ComponentA from './componentA'
import * as actions from 'your/path/to/actions'
import { bindActionCreators, Dispatch } from 'redux'

function mapStateToProps(state: any): any {
  return {
    variableA: state.stateA.variableA
  }
}

function mapDispatchToProps(dispatch: Dispatch): any {
  return {
    actionA: bindActionCreators(actions.actionA, dispatch)
  }
}

export default connect(
  mapStateToProps,
  mapDispatchToProps
)(ComponentA) as any
```

```tsx
/** component.tsx */
interface Props {
  actionA: () => void;
  variableA: string;
}

export default class ComponentA extends React.Component {
  public render(): JSX.Element {
    return <div onClick={() => this.props.actionA}>{this.props.variableA}</div>;
  }
}
```

## After

```tsx
/** index.tsx */
import * as React from "react";
import { useDispatch, useSelector, shallowEqual } from "react-redux";
import * as actions from "path/to/actions";

interface Props {
  stateA: string;
}

export const ComponentA = (props: Props) => {
  const dispatch = useDispatch();
  const variableA = useSelector(
    (state: any) => state.stateA.variableA,
    shallowEqual
  );

  return <div onClick={() => dispatch(actions.actionA())}>{variableA}</div>;
};

export default injectIntl(React.memo(ComponentA));
```

## So?

从改造前的代码和改造后的代码中，我们可以看出，使用 Hooks 确实节省了很多代码的行数。在 connect 的写法中，上面的例子并未指定 mapDispatchToProps 的返回值类型。而在 dumb component 中，组件的 action 类型其实是遵循于组件内定义的 props 的类型的，如果我们在组件内定义的 action 类型和 action creator 中定义的类型不同，例如传入参数不同，那么实际上 TypeScript 是检测不出这个问题的。这样就会引起不必要的 Bug。而使用 Hooks 完全可以避免这个问题，因为我们是直接把 action creator 引进来的，TypeScript 可以识别出 action creator 中定义的类型。也就是说如果你传入了错误的参数，那么此时 TypeScript 编译是不会通过的。所以可见，使用 Hooks 可以减少开发中产生 bug 的几率，也简化了代码，更易读更易维护。

# Tech Spec

## useEffect

Class Component 对于我们来说有足够的功能新，因为我们可以通过自定义生命周期钩子来实现很多复杂的功能，这是在 React v16.8 前 Functional Component 无法实现的。这也是之前我喜欢用 Class Component 的原因。而 useEffect 这个近乎“万能”的 hook 可以实现几乎所有的生命周期钩子，这对于 Functional Component 非常关键，也是我重构组件最依赖的一个 hook。useEffect 的类型签名如下:

```TypeScript
function useEffect(effect: EffectCallback, deps?: DependencyList): void;

type EffectCallback = () => (void | (() => void | undefined));
type DependencyList = ReadonlyArray<any>;
```

可以看到 `useEffect` 有两个参数，第一个参数是一个 callback，里面写的是我们的逻辑。而第二个参数不要小看它，虽然它是可选参数，但是如何玩转 `useEffect` 完全就靠它了！它是“依赖”，React 官网也有不小的篇幅来描述怎样使用它。简而言之，这个“依赖”，就是**当它改变的时候才会执行里面的 callback**。先说下不传这个参数的情况，那就是每次 re-render 都会调用这个 hook。而我们在实际开发中往往不会希望我们所有的 `useEffect` 的 callback 在每次渲染都被调用一次。所以“依赖”的作用就在于此，例如：

```TypeScript
useEffect(() => {
  console.log(props.name)
}, [props.name])
```

这个 hook 只会在 `props.name` 发生变化的时候把 `props.name` 在控制台打印出来。这个相当于 Class Component 中的：

```TypeScript
public componentDidUpdate(prevProps) {
  if (this.props.name !== prevProps.name) {
    console.log(this.props.name)
  }
}
```

或者是即将被废弃的 `componentWillReceiveProps`。

而如果我们修改一下这个依赖呢？这其实挺纠结的，因为如果你在你的项目中使用了 eslint，那么这里它会提示一个 `exhaustive-deps` 的 warning。而如果你开了 Prettier 的 eslint 错误自动修复。。。那不好意思，要么你只能不写，要么会被强制改成里面所有用到的依赖，如 state、props 或者其他变量等。起初我也陷入了深思，到底是不允许这么写，还是 eslint 有毛病呢？但是实际上，我们应该先抛弃 eslint，先在这将它 disable 了，因为 `useEffect` 能够做到的远比我们想象的多。

上面既然我们实现了 `componentDidUpdate`，那其他的钩子呢？有两个很关键的钩子 `componentDidMount` 和 `componentWillUnmount` 我们还不知道怎么实现呢。而我之前提到它是“近乎‘万能’”的，是的，它就是可以实现这些生命周期钩子的效果。

为了实现 `componentDidMount`，我们只需要在依赖中传入一个空数组 `[]`。

而为了实现 `componentWillUnmount`，我们只需要在 `useEffect` 中 return 一个 callback，那个 callback 便是在 `componentWillUnmount` 时执行的。上面提到的

```TypeScript
type EffectCallback = () => (void | (() => void | undefined));
```

是有写返回值类型的，而它也不是白写的，正是用在这种场景。

---

依赖很万能，但是不要乱用，因为乱写依赖会导致你的组件渲染的行为异常，甚至会导致 Infinite Loop (`useEffect` 的 callback 触发了某个变量的变化导致了 re-render, 而 re-render 又触发了 useEffect 并运行了 callback)。因为 re-render 大概是我们在 React 开发中最经常需要处理的一件事了，所以用好 `useEffect` 是用好 React hooks 的关键。

## Hooks in react-redux

[Hooks](https://react-redux.js.org/api/hooks) 在 react-redux 中有专门的文档介绍。常用的 API 其实就两个：用于获取 state 的 `useSelector` 和用户 dispatch action 的 `useDispatch`。使用的方式很简单，参考官方文档就可以了。

### useSelector

`useSelector` 允许我们使用 `reselect` 的 selector 进行 memorize 的操作，也可以直接从 redux store 中获取相应的 property。但是有个“比较”的问题，就是官方文档使用了 === 的比较（[参考源码](https://github.com/reduxjs/react-redux/blob/316467a07e29911d82ba0342364a907e05d9066c/src/hooks/useSelector.js#L17)），而如果我们 select 出来的是 Object，那每次比较出来的 Object 都是不同的。因为 reducer 遵循了纯函数的原则，每次都会 return 一个新的 state，而 state 是一个 JavaScript Plain Object，也就是说每次 return 出来的 Object 指向的内存地址都是不同的，那么 select 出来的 Object 也是不同的。这样就会导致**每次 redux store 有变化，这个组件就会重新渲染**。这时候我们就需要浅比较了。官方给我们提供了一个 `shallowEqual` 函数，让我们可以做浅比较。这个方法其实在早前的 react-redux 版本中就有了，因为之前 react-redux 的 connect 就是用了 shallowEqual 的。当然我们也可以使用诸如 lodash 中的浅比较方法。

### useDispatch

`useDispatch` 其实没有什么好说的，它就是一个函数，不接收任何参数，只能在组件中使用：

```JavaScript
const dispatch = useDispatch()
```

它的功能就是 redux 中的 `store.dispatch`。

## React.memo

`React.memo` 其实并不是 Hooks 的内容，但是由于我们只能在 Functional Component 中使用 Hooks，Class Component 中能用的生命周期钩子也都不能用，所以之前的
`React.PureComponent` 和 `shouldComponentUpdate()` 也都无法使用。那么对于应用优化，我们常用的手段在 Functional Component 中都无法使用了。`React.memo` 的出现就很有效地解决了这个问题。它能实现与 `React.PureComponent` 和 `shouldComponentUpdate()` 相同的功能，就是防止父组件 re-render 导致的子组件的不必要的 re-render。

对于“不必要的 re-render”，举个例子来说，就是父组件使用 `useSelector` 订阅了 redux store 的某个属性，但是它与子组件无关。如果这个属性有变化，那么按照 React 的 diff 算法，它的子组件都会 re-render 一遍。这也不能说 React 本身有问题，因为 diff 算法已经足够好，能够高效地解决 React 渲染计算的问题。那么我们其实是不希望与该属性无关的子组件去 re-render 的，因为对我们来说它其实一点变化都没有。所以 `React.memo`、`React.PureComponent` 和 `shouldComponentUpdate()` 就是用于解决这种问题的。

它在 react-redux 中也挺重要的，因为官方说 `useSelector` 不会阻止子组件的不必要的 re-render，而官方也推荐使用 `React.memo` 进行优化。

## 单元测试

其实 Hooks 并不是完美的，比如单元测试。

Jest + Enzyme 是 React 中常用的单元测试库，但是 Enzyme 官方文档中提到：`With React 16 and above, instance() returns null for stateless functional components.`。对于 Hooks 而言（使用了 Hooks 的组件必然是 functional components），虽然官方文档也说明了[对 Hooks 的支持](https://github.com/airbnb/enzyme#react-hooks-support)：目前只在 `shallow()` 里提供有限的支持，但是想要照搬 class component 中用的测试套路显然是不行的。React 官方则推荐使用 [react-testing-library](https://github.com/testing-library/react-testing-library) 来测试。**但是这些目前只能测试 React 官方的 Hooks，对于 react-redux 的 `useDispatch` 和 `useSelector`，目前我还没有找到测试的方法，也没有找到相关的教程或者文档，甚至 react-redux 官方也没有详细的教程。**

相比之下，class component 可以直接把所有的 connect 以及其他高阶组件和 dumb component 剥离开来测试，因为我们测试组件仅仅关注的是 dumb component 上的逻辑，高阶组件注入的 props 完全可以通过 mock 来做（通常我会直接无视 TypeScript 的类型错误，因为很多 props 其实测试中是不必要的，例如 react-intl 的高阶组件注入的那些 props）。但是对于 Hooks，没有了 connect，也就是说 redux 和 dumb component 其实是耦合在一起了，做单元测试首先就会提示，必须用 react-redux 的 `<Provider>` 把组件包裹起来。这样其实无形中增加了测试的难度。

# 总结

Hooks 是个新技术，它确实给我们的开发带来了便利，但是单元测试上却并不是那么容易。这也是我们经常遇到的问题，就是只在意功能而忽视了代码质量。只有功能和质量同时有保障的代码才是好的代码。关于单元测试这部分，我会继续研究并继续关注社区上的相关动态。
