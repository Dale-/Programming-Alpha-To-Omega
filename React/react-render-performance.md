## React渲染机制
React不直接操作真实DOM，它在内部维护了一套快速相应的虚拟DOM(本篇文章简称为VDOM)，`render`方法返回一个VDOM的描述，React会在reconsilation之后最小化的进行VDOM的更新，最终patch到真实的DOM。

下面官网给出的React组件渲染机制描述图
![](/source/img/javascript/react-update.png)

我们再来看看 触发组件更新的流程图

![](/source/img/javascript/react-render-flow.png)

通过上述流程图，再对比渲染机制描述图我们可以发现，如果进行性能优化，关键在：

* shouldComponentUpdate阶段，如果props与state与上一次相同，这个时候可以中断后续的生成虚拟DOM以及DOM DIFF的过程。
* 提高DOM DIFF的效率

## 性能检测工具
### Why-did-you-update

why-did-you-update是一个可以检测到潜在不必要的组件渲染的库。当发生比必要的更新时，它可以在控制台上打印出可能的可以避免多次 re-render 的操作。

它的原理是，通过比较每次 render 时组件的 prevProps 和 this.props，这里的比较是深比较，对 plain object 递归比较，对同为函数的相同属性只通过比较函数名来判断是否相同。

![](/source/img/javascript/why-did-you-update.png)

> Setup

```javascript
  npm install --save why-did-you-update
  or
  yarn add why-did-you-update
```

> Usage

```javascript
  import React from 'react';

  if (process.env.NODE_ENV !== 'production') {
    const {whyDidYouUpdate} = require('why-did-you-update');
    whyDidYouUpdate(React);
  }
```

More Options please click [why-did-you-update](https://github.com/maicki/why-did-you-update)

需要注意的是，要确保`why-did-you-update`在production环境中被禁用，因为它会减慢应用程序的速度。

### 性能时间表
React 15.4.0引入了性能时间轴的功能，可以更直观的了解可视化的组件生命周期。

React 官方文档里推荐的性能检测方法，是对 Chrome Devtool 的加强，可以将 Devtool 中 JS 部分的火焰图细分为组件各个声明周期的时间，在目前的开发模式下，不用输入 ?rect_perf 也已经开启了这个功能，生产模式下的页面仍然需要加上这个命令。

> 使用方法

* 打开你的应用程序并添加查询字符串?react_perf 。例如: http://localhost:3000
* 打开Chrome DevTools, 选择Performance，点击Record。
* 执行你想要分析的动作。
* 停止记录。
* React事件将会被归类在 **User Timing** 标签下。

![](/source/img/javascript/react_perf.png)

*这些数字是相对的，因此组件在生产版本中会运行更快*。然而，这也能够帮助你了解何时会有无关的组件被错误的更新，以及你的组件更新的深度和频率。

## 优化方法
### PureComponent
> Component的默认渲染行为

在React Component的生命周期中，又一个`shouldComponentUpdate`方法，这个方法的默认返回值为`true`。

这意味着Component不会比较对当前和下一个状态下的`props`和`state`, 因此每当`shouldComponentUpdate`被调用时，组件默认会重新渲染。从而导致组件因为不相关数据的改变导致重绘，这极大的降低了React的渲染效率。

> PureComponent

当组件更新时，如果组件的`prop`和`state`都没发生变化，`render`方法就不会被触发。从而省去了`Virtual DOM`的生成和对比过程，达到了提升渲染效率的目的。

下面是PureComponent对比`props`和`state`的浅比较
```JavaScript
  if (this._compositeType === CompositeTypes.PureClass) {
    shouldUpdate = !shallowEqual(prevProps, nextProps) || ! shallowEqual(inst.state, nextState);
  }
```

下面是shallowEqual的源码
```javascript
  'use strict';

  const hasOwnProperty = Object.prototype.hasOwnProperty;

  /**
   * inlined Object.is polyfill to avoid requiring consumers ship their own
   * https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/is
   */
  function is(x: mixed, y: mixed): boolean {
    // SameValue algorithm
    if (x === y) { // Steps 1-5, 7-10
      // Steps 6.b-6.e: +0 != -0
      // Added the nonzero y check to make Flow happy, but it is redundant
      return x !== 0 || y !== 0 || 1 / x === 1 / y;
    } else {
      // Step 6.a: NaN == NaN
      return x !== x && y !== y;
    }
  }
  /**
   * Performs equality by iterating through keys on an object and returning false
   * when any key has values which are not strictly equal between the arguments.
   * Returns true when the values of all keys are strictly equal.
   */
  function shallowEqual(objA: mixed, objB: mixed): boolean {
    if (is(objA, objB)) { // 如果 ===，返回 true
      return true;
    }


    if (typeof objA !== 'object' || objA === null ||
        typeof objB !== 'object' || objB === null) {
      return false;
    }

    const keysA = Object.keys(objA);
    const keysB = Object.keys(objB);

    if (keysA.length !== keysB.length) {
      return false;
    }

    // Test for A's keys different from B.
    for (let i = 0; i < keysA.length; i++) {
      if (
        !hasOwnProperty.call(objB, keysA[i]) || // A B 中包含相同元素且相等，函数也是直接用 === 来比较
        !is(objA[keysA[i]], objB[keysA[i]])
      ) {
        return false;
      }
    }

    return true;
  }
```

`shallowEqual` 会对Object的每个属性，如果是非引用类型，那么可以直接比较判断是否相等。如果是引用类型，则通过比较引用的地址进行严格比较（===）。当 props 和 state 是不可突变数据的时候，可以直接使用 PureComponent。

> 使用方法

将原组件继承自React.Component,替换为React.PureComponent

```JavaScript
  import React, { PureComponent } from React

  class Foo extend PureComponent {
    render() {
      // ...
    }
  }

  export default Foo
```

### 实现 shouldComponentUpdate
我们知道`PureComponent`的`shadowEqual`只会浅检查组件的`props`和`state`,所以嵌套对象和数组是不会被比较的。所以如果是需要深比较，我们也可以使用`shouldComponentUpdate`来手动单个比较是否需要重新渲染。

`shouldComponentUpdate`是`props`或`state`更改后、render前被调用的方法。如果`shouldComponentUpdate`返回`true`，`render`将被调用，如果它返回`false`，则组件不会重新渲染。

所以在一些情况下，你可以通过重写这个生命周期函数`shouldComponentUpdate`来提升渲染效率。这个函数默认返回`true`。如果你知道在某些情况下你的组件不需要更新，你可以在`shouldComponentUpdate`内返回`false`来跳过整个渲染进程。

```javascript
  shouldComponetUpdate(nextProps, nextState) {
    if (/* do some compare*/) {
        reture true;
    }  
  }
```

### plugin-transform-react-inline-elements
[(Reference)Optimizing Compiler: Inline React Elements](https://github.com/facebook/react/issues/3228)

在production环境，优化`React.createElement` function为`babelHelpers.jsx`

> In

```javascript
  <Baz foo="bar" key="1"></Baz>
```

> Out

```javascript
  babelHelpers.jsx(Baz, {
      foo: "bar"
  }, "1");

  /**
  * Instead of
  */

  React.createElement(Baz, {
    foo: "bar",
    key: "1",
  });

```

### 使用稳定的 `key`

> 使用稳定的 key, 对子组件进行唯一性识别，准确知道要操作的自组件，提高DOM DIFF的效率

* 唯一的: 元素的 key 在它的兄弟元素中应该是唯一的（而非全局唯一）
* 稳定的: 元素的 key 不应随时间、页面刷新或者元素重新排序而变

当使用数组中的索引作为 key时，若元素没有重排，该方法效果不错。但是如果其中的item被删除或移动导致重排,则整个key 就失去了与原项的对应关系，加大了diff的开销。

## 参考
[Why Did You Update](https://hackernoon.com/make-react-fast-again-part-2-why-did-you-update-43a89dc96b10)