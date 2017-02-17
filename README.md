# React Performance Optimization

> 近日读了《React 全栈》这本书，其中最后一章讲述了 React 性能优化方面的技术，更是介绍了一些做 React 性能优化的误区，对我的帮助挺大的。本文即对这一部分的内容进行归纳总结。

## React 性能误区

### "短路"式的性能优化

基于 shouldComponentUpdate 的“短路”式优化应该被无限制的（甚至默认的）使用。

基于 shouldComponentUpdate 的优化是最常见的 React 性能优化手段。然而要注意的是这一手段有两个前提：

1. 组件本身的 render 方法是参数 state 和 props 的纯函数，即对于未变化的 state 与 props，render 方法将返回相同的结果。
1. state 与 props 是不可变的。

对于第 2 点，出于对复杂参数比对代价的考量，实现中一般都只会进行浅层比较，即对 state 与 nextState、props 与 nextProps，遍历其键值对，要求每个键名对应的值严格相等。因此，如果其中有值是可变的数据，如 object，则其内容的变化是无法被检查出来的。从而导致数据变化了，但界面并未更新。

另外，对于第 2 点，可以使用 immutable.js 来简化比较流程，代价是引入一个 56KB 的库文件。对于客户端或者 RN 这样的应用场景倒是没问题，如果是 web 端，则需要衡量下得失。

在保证两个前提下使用“短路”式优化也一定都能达到最好的效果。这种方式比较像缓存，有命中，也有失败。失败的时候，则这种优化就变成了“负优化”。对于大多数情况，这种“负优化”的开销很低，但对更新即为频繁的组件来说，其累计损失不可忽略。

*所以，“短路”式优化的应用与否，需要先对应用进行分析，在组件确实成为瓶颈且这种优化可以带来明显性能提升再使用。*

### 无状态函数式组件的性能

无状态函数式组件（Stateless Functional Component，简称 SFC）是官方推荐尽可能采取的组件写法。官方声称将来可以对这种写法的组件进行针对性的性能优化，但截止 React v15.0.1，SFC 尚未得到承诺的针对性优化。另外，虽然 SFC 不存在 shouldComponentUpdate 这样的优化，但是可以使用高阶函数来达到同样的效果。

```js
function PureRenderEnhance (component) {
  return class Enhanced extends Component {
    shouldComponentUpdate(nextState, nextProps) {
      return shallowCompare(this, nextState, nextProps);
    }

    render() {
      return React.createElement(component, this.props, this.props.children);
    }
  }
}
```

值得注意，上一节提到的“短路”式优化的问题仍然存在，使用时需要注意。

## 反模式

在软件工程中，一个反面模式（anti-pattern 或 antipattern）指的是在实践中明显出现但又低效或是有待优化的设计模式，是用来解决问题的带有共性的不良方法。这里只着重介绍 React 中常用的反模式做法。

### 基于 props 得到初始 state

不论是 ES5 还是 ES6，基于组件的 props 计算得到初始 state 往往都是典型的反模式。准确地说，这个行为本身并不一定是不对的，但这一行为的出现往往意味着 state 设计或是别的地方存在问题。以一个名字组件为例：

```js
class UserName extends React.Component {
  constructor(props) {
    super(props);

    this.state = {
      fullName: `${props.firstName} ${props.lastName}`
    };
  }

  render() {
    return (
      <span>{ this.state.fullName }</span>
    );
  }
}
```

这个例子比较简单，存在的问题：constructor 只会在组件初始化时执行一次，在后续发生 props 更新时不会被调用，因此 this.state.fullName 的值不会随着组件 props 的变化而更新，界面也就不会更新。这时常见的做法是在 componentWillReceiveProps 中更新 state。问题也随之而来，要在两个地方维护 state 的正确性。此时再回头检查，不难发现，问题的根源在于这里的 fullName 本不应该是组件 state 的一部分，即它并不是组件的内部状态，它只是一个一居组件的 props 可以计算出的一个结果。对于这样的数据，合适的做法是将计算过程放到 render 里。

```js
class UserName extends React.Component {
  render() {
    const fullName = `${this.props.firstName} ${this.props.lastName}`;
    return (
      <span>{fullName}</span>
    )
  }
}
```

*一个设计良好的 React 组件，其 state 与 props 包含的信息应该是正交的。如果 state 需要基于 props 计算得到，那么他们很有可能包含了重复的信息，这时就需要回头检视 state 与 props 的设计是否合理。*

最后，需要说明一下，前面提到“反模式”时强调了“往往”、“很可能”。*在一些例子中，某些 props 只是用来作为初始化 state 的“种子”信息的，二者不存在需要同步的关系，这种情况下在生成初始 state 时使用 props 就是合理的。* 比如计数器初始数的设定、输入控件的 defaultValue 属性，这种情况的特点是，在 props 的命名里往往就说明了它只用于初始化。props 中的 initialCount 与 state 中的 count 显然没有包含重复的信息，因此这与上面提到的 “state 与 props 包含的信息应该是正交的”并不矛盾。

### 使用 refs 获取子组件

习惯了传统的 JavaScript 组件化方案的开发者，往往会很熟悉手动实例化组件，然后读取其属性或调用其方法。在刚刚接触 React 时会较难接受这种通过 JSX 进行声明、React 自动实例化的方式，因为这样父组件没有持有子组件的引用，习惯的做法也就没法套用过来。因此，在开发他们的头几个 React 项目时，React 提供的 refs 是极常见的被滥用的功能。同样通过一个很典型的例子进行说明，在这个例子里有两个组件：通知组件（Notification）与某业务组件（Sample），业务组件通过通知组件展示通知信息。

```js
class Notification extends React.Component {
  // 添加通知的方法
  notify(content) {
    // ...
  }
  render() {
    // ...
  }
}

class Sample extends React.Component {
  constructor(props) {
    super(props);
    this.onBtnClick = this.onBtnClick.bind(this);
  }

  onBtnClick() {
    this.refs.notification.notify('Button clicked!');
  }

  render() {
    return (
      <section>
        <button onClick={this.onBtnClick}>Click me!</button>
        <Notification ref="notification" />
      </section>
    );
  }
}
```

上例中，父组件 Sample 通过拿到子组件 Notification 实例的引用，调用其实例方法来达到展示特定通知信息的效果。这里的通知组件也可能是个模态框（Modal），notify 方法可能是模态框的 show/hide 方法，很多人应该都见过类似的例子。然而，这在 React 的项目里并不是一个最合适的做法。首先看它的问题，如下：

- 子组件多出了除 props 外的其他形式的对外接口，使用更复杂。
- 命令式的方法调用行为与 React 声明式的编程风格不一致。
- 调用行为不便追溯，数据本身也没有记录，增加了跨组件调试的难度。
- 无状态函数式组件在渲染时是不存在对应的实例的，也无法定义实例方法，这样的做法不通用。

那么更合适的做法是什么呢？很简单，用 React 提供的工具来解决这个问题。在 React 里，我们希望尽可能所有组件间的通信都是通过props 完成的。

```js
class Notification extends React.Component {
  render() {
    // 将通知内容作为 props 的一部分由外部传入
    const content = this.props.content;
    // ...
  }
}

class Sample extends React.Component {
  constructor(props) {
    super(props);
    this.state = {
      notification: null
    };
    this.onBtnClick = this.onBtnClick.bind(this);
  }

  onBtnClick() {
    // 通过 setState 触发更新
    this.setState({
      notification: 'Button Clicked!'
    });
  }

  render() {
    return (
      <section>
        <button onClick={this.onBtnClick}>Click me!</button>
        <Notification content={this.state.notification} />
      </section>
    );
  }
}
```

*与前面的情况类似的是，使用 refs 本身是没有问题的，但是这种使用 refs 获得子组件的引用以调用其实例方法来触发状态变更的做法往往是有问题的，因为它往往意味着这里得到了变更的状态信息，并没有把它抽取记录在某个地方（某个上层组件的 state 里或全局的 Redux store 里），在 React 的世界里显得用很不自然的方式来实现组件间的通信。不过，有时不得已需要操作原生 DOM（如获取 input 的 value），这种情况下通过 refs 获得实例（真是的 DOM 节点）是没有问题的。*

### 冗余事实

React 是一个界面库，并不关心数据怎么组织：Redux 是一个状态容器，也不关心状态的具体形状。不过在组织与管理 React + Redux 项目的状态/数据时，有一个很简单的原则：事实只有一份，但这也是一个很容易被忽视的原则，与之对应地，冗余事实是典型的反模式。

前面的一节中提到的“基于 props 得到初始 state”可以看成是冗余事实的某一个具体表现，它对应的那些事实在 props 与 state 中各有一份情况。同理，这种冗余也可能存在于 Redux 的 reducer 与 reducer 之间（即状态数据的不同部分之间）、同一组件的不同 props 或不同 state 之间。简单地说，当我们消费数据的时候，如果某一份数据可以从不止一个来源（可能性需要通过额外的计算）得到，那么这往往是有问题的。对 Redux 与 store state 的形状设计是这一问题的高发区，下面将就此举例说明。假设数据中包含一系列的书籍信息，不过每本书都有作者信息，所以拿到的书籍信息的结构是如下这样的。

```json
// books
[
  {
    id: '134',
    name: 'book A',
    price: 20,
    author: {
      id: '462',
      name: 'author X',
      age: 30,
      country: 'China'
    }
  }
]
```

如果只是简单地这样存储书籍信息，那么就犯了一个典型的冗余事实的错误。在可能需要点击作者名字的时候，弹出一个卡片展示作者的信息，做法是通过点击作者这一行为携带的作者 ID，去 store state 中查询得到这个作者的完整信息，但这种形式的 state 要求遍历所有 book，检查其 author 信息。在不同的书籍对应同一作者的情况下，会出现冗余，将不能决定应该使用哪一份 book 中的 author 信息。此时一般都会增加一个与 book 信息评级的 author 这样的 reducer，将作者信息存放在其中，当需要查找作者信息时，就从 store state 的这一部分获取。

```json
// authors
[
  {
    id: '462',
    name: 'author X',
    age: 30,
    country: 'China'
  }
]
```

这样是不是就够了呢？还不够。因为作者信息还同时存在于 books 与 authors 这两处，还应该对书籍信息做一下调整，如下。

```json
// books
[
  {
    id: '134',
    name: 'book A',
    price: 20,
    author: '462'
  },
  // ...
]
```

这样，当需要消费作者信息时，便有了明确的来源：在需要消费书籍信息时，也可以通过查找 authors 获取书籍对应的作者信息。要注意的是，这样的结构是对前端使用更友好的数据组织格式，但它不一定对前后端通信友好，很多时候需要在与后端通信时进行一些额外处理，以便维持前端应用状态的合理性。

### 组件的隐式数据源

隐式数据源是指，那些仅仅通过观察组件对外暴露的接口无法发现的数据来源。最典型的例子就是组件在实现代码中直接 require 某个模块，并从中获得数据，这个模块可能是一个全局的事件触发器，也可能是一个 model 实现。

事实上，“隐式数据源”在传统的基于模块的前端项目开发中是很常见的做法。其缺点是：
- 组件行为不可预测。对于完全相同的输入（props），输出（组件行为）不一定相同。
- 组件可复用性下降。隐式的数据源导致组件可复用的场景受限，如组件直接从某 store 中读取数据，在该 store 不适用的环境里，组件无法被复用。

React 的组件方案提供了简单清晰的数据传入方式：props 与 context，且提供了如 propTypes、contextTypes 这样的工具来帮助发现数据流传递中潜在的问题。因此，在 React 项目中，使用 props 或 context 来传递应用状态、对象模型数据等内容是更好的方法。

至于前面提到的，组件也可能自行依赖某个全局的事件触发器，这是另一种传统前端项目常用的手段，用来实现组件间的通信。在这种情况下，组件隐式依赖的一般不是数据本身，而是应用行为与行为所包含的数据。但是在 React 项目中，组件间实现通信较好的做法是，在更高层维护状态并基于 props 或 context 传递数据与回调函数。

大部分情况下不需要 context，即便用也要有克制的使用。

### 不被预期的副作用

React 与 Redux 为前端开发领域带来了很多函数式编程的理念：不可变数据、纯函数、函数组装等。在函数式编程中，副作用的产生是被严格限制的，纯函数应该是无副作用的。

一般项目中，除了对应用其他部分的行为产生影响，DOM 操作、AJAX 请求等也算是副作用。一个应用没有副作用是不可能的，不过限制副作用产生的位置可以帮助抽取应用中的纯逻辑，享受函数式理念带来的好处。

在一般的 React 项目里，会期望组件本身（尤其是 render 方法）是无副作用的；在 Redux 项目里，会要求 reducer 是无副作用的。

基于上面提到的副作用和常见表现和那些被期望无副作用的位置，可以得到一个结论：在组件的 render 方法或 store 的 reducer 实现中，触发 action 或调用组件的 setState（对应用其他部分的行为产生影响）、操作 DOM或发送 AJAX 请求等行为，都是反模式的。

## 性能优化

对于规模不大的 React 项目，基本不需要考虑性能问题，因为 Virtual DOM 和高效的 Diff 算法带来的性能提升足够了。但是当应用较为复杂或数据量更大的时候，性能问题就会很容易的显露出来了。那么问题来了，基于 React 的项目，如何做性能优化呢？

### 优化原则

关于优化，有一些普适的原则可以给出好的指导，Rick Cook 在文章《Don't Cut Yourself: Code Optimization as a Double-Edged Sword》中提到了很多有价值的观点，这里强调两点：

1. 避免过早优化
过早的优化不仅常常无法起到预期的效果，还会给项目引入额外的复杂度，影响项目的开发与维护效率。在开发时，首先考虑的应该是逻辑拆分的合理性及代码的可读性、可维护性，在需求基本完成、性能成为无法避开的挑战时，再着手做针对性的优化措施。
1. 着眼瓶颈
性能优化本身是在做权衡，绝大部分情况下，性能上的提升需要损失部分代码的简洁与可读性。最合理的做法是找到对性能影响最大的部分（性能瓶颈），以对其他部分代码影响尽可能小的方法进行优化。过度的优化会容易导致额外的损失。

### 性能分析

性能分析的一个比较合适的执行方案：
1. 在应用基本开发完成后考虑优化
1. 先找到性能瓶颈，在针对优化

对于 React 项目来说，耗时可能存在两个地方：
- 业务代码的行为
- React 的行为

*对于前者，可以借助 Chrome DevTool 提供的 JS Profiler 工具来找到调用频繁、耗时较多的代码。对于后者，如果借助 React 官方提供的辅助工具，会有事半功倍的效果。*

#### react-addons-perf
这是官方提供的性能分析工具，可以通过 npm 安装。安装为 devDependencies 后，在应用的入口文件中引入以下代码：

```js
import Perf from 'react-addons-perf';
window.Perf = Perf;
```

因为这个工具主要是在命令行中完成，所以要暴露到 window 下。这个工具的用法是：

```js
// do something ...
// 开始记录
Perf.start();
// some actions ...
// 结束记录
Perf.stop();
// 获取本次记录的行为（介于 start 与 stop 之间的所有应用行为）的测量数据
var measurements = Perf.getLastMeasurements();
```

对于记录的行为数据，一般不直接阅读，有额外的几个方法可以以更友好的方式来打印所关心的信息。

1. printInclusive：打印总体花费的时间。

| (index) | Owner > Component | Inclusive render time (ms) | Instance count | Render count |
| -- | -- | -- | -- | -- |
| 0 | "App" | 90.97 | 1 | 1 |
| 1 | "App > List" | 90 | 1 | 1 |
| 2 | "List > ListItem" | 64.8 | 1000 | 1000 |
| 3 | "App > ItemShowLayer" | 0.53 | 1 | 1 |
| 4 | "App > CreateBar" | 0.06 | 1 | 1 |

2. printExclusive：打印“独占”的时间信息，即不包括在加载组件上花费的时间：处理 props、getInitialState，调用 componentWillMount 及 componentDidMount 等。

| (index) | Component | Total time (ms) | Instance count | Total render time (ms) | Average render time (ms) | Render count | Total lifecycle |
| -- | -- | -- | -- | -- | -- | -- | -- |
| 0 | "ListItem" | 64.8 | 1000 | 64.8 | 0.06 | 1000 | 0 |
| 1 | "List" | 25.2 | 1 | 25.2 | 25.2 | 1 | 0 |
| 2 | "ItemShowLayer" | 0.53 | 1 | 0.53 | 0.53 | 1 | 0 |
| 3 | "App" | 0.37 | 1 | 0.37 | 0.37 | 1 | 0 |
| 4 | "CreateBar" | 0.06 | 1 | 0.06 | 0.06 | 1 | 0 |

3. printWasted：打印那些被“浪费”的时间，这个方法非常有用。“浪费”是指那些执行了 render，但没有导致 DOM 更新（渲染结果与前次相同）的行为。该方法往往能有效地发现可以被避免的 render 调用。

| (index) | Owner > Component | Inclusive waste time (ms) | Instance count | Render count |
| -- | -- | -- | -- | -- |
| 0 | "App > List" | 90 | 1 | 1 |
| 1 | "List > ListItem" | 64.8 | 1000 | 1000 |
| 2 | "App > CreateBar" | 0.06 | 1 | 1 |

4. printOperations：打印最终发生的真实 DOM 操作。它有助于发现那些可以被避免的真实 DOM 操作。

| (index) | Owner > Node | Operation | Payload | Flush index | Owner Component ID | DOM Component ID |
| -- | -- | -- | -- | -- | -- | -- |
| 0 | "ItemShowLayer > h2" | "replace text" | "test-1" | 0 | 5047 | 5050 |
| 1 | "ItemShowLayer > div" | "replace children" | "<p>test-content-1</p>" | 0 | 5047 | 5054 |

以上就是性能瓶颈的定位方法。

### 生产环境版本

准确说这不能算是性能优化，但对于应用的性能影响很大。注意生产环境要设置环境变量 NODE_ENV，代码压缩工具会移除无用代码，代码运行效率更高，体积也会缩小。比如：

```js
if (process.env.NODE_ENV !== 'production') {
  // 开发环境相关代码
}
```

上面的代码替换后为：

```js
if ('production' !== 'production') {
  // 开发环境相关代码
}
```

这样的代码会被代码压缩工具自动移除的。

### 避免不必要的 render

在优化 React 应用时，绝大部分的优化空间在于避免不必要的 render，这不仅可以节省执行 render 的时间，还可以节省对 DOM 节点做 Diff 的时间。

1. shouldComponentUpdate

shouldComponentUpdate 是一个会被频繁调用的方法，所以对于 Object 等复杂类型进行神比较的代价很大。如果数据结构很复杂，深比较甚至会不如直接 render 一次。相比深比较，浅比较最多只遍历一层子节点。所以推荐使用浅比较。

2. Mixin 与 HoC

对于浅比较，React 官方提供了两种实现，分别是 Mixin 形式的 PureRenderMixin 和辅助工具 shallow-compare。Mixin 是 ES5 的写法，但 ES6 推荐使用 shallow-compare。

```js
// ES5
var PureRenderMixin = require('react-addons-pure-render-mixin');
React.createElement({
  mixins: [PureRenderMixin],
  render： function() {
    return <div className={this.props.className}>foo</div>;
  }
});
```

```js
// ES6
import shallowCompare from 'react-addons-shallow-compare';
export class FooComponent extends React.Component {
  shouldComponentUpdate(nextProps, nextState) {
    return shallowCompare(this, nextProps, nextState);
  }

  render() {
    return <div className={this.props.className}>foo</div>;
  }
}
```

上面两种方法本质上是一样的。

另外，也有以高阶组件形式提供这种能力的工具，如库 recompose 提供的 pure 方法，用法更简单，很适合 ES6 写法的React 组件。

```js
import { pure } from 'recompose';

class FooComponent extends React.Component {
  render() {
    return <div className={this.props.className}>foo</div>;
  }
}

const OptimizedComponent = pure(FooComponent);
```

这种方式还有一个好处是支持函数式组件，如下：

```js
const FunctionalComponent = ({ className }) => (
  <div className={className}>foo</div>;
);

const OptimizedComponent = pure(FunctionalComponent);
```

3. 不可变数据

前面使用 shouldComponentUpdate 与浅比较优化性能的一个前提是数据（props 与 state）是不可变的。但是 JavaScript 中是可变的。为了解决这一问题，facebook 退出了 immutable.js。使用的代价是引入一个体积不算小（55.7KB，Gzip 后 16.3KB）的库，并且数据的操作方式也有所改变。幸好大部分情况下可以使用 JavaScript 原生语法或方法中对不可变数据更友好的部分这种代价较小的做法。

```js
const newObj = Object.assign({}, obj, {a: 1});
```

或

```js
const newObj = {
  ...obj,
  a: 1
}
```

对于数组，可以使用 `concat`、`slice`、`map`、`filter`、`from` 等方法。也可以借助 ES6 的 Array Rest/Spread 语法。

React 官方也提供一个便于修改较复杂数据结构深层次内容的工具 —— react-addons-update，它借鉴了 MongoDB 的 query 语法。

```js
var update = require('react-addons-update');

var newData = update(myData, {
  x: {y: {z: {$set: 7}}},
  a: {b: {$push: [9]}}
});
```

4. 记忆计算结果

基于 immutable data 的性能优化能够减少重复渲染，但也不能覆盖所有的场景。很多时候，组件依赖的数据不是简单的记录在全局 state 中，而是需要通过对 state 数据的计算得来。看一个 TodoList 的例子：

```js
const stateToProps = state => {
  const list = state.list;
  const visibleFilter = state.visibleFilter;
  const visibleList = list.filter(
    item => (item.status === visibleFilter)
  );

  return {
    list: visibleList
  };
}

function List({list}) { /* ... */}
const VisibleList = connect(stateToProps)(List);
```

如上，数据 visibleList 是通过对 list 进行计算得到的。当 state 变化时，及时 state.list 与 state.visibleFilter 没变，每次执行 stateToProps 都会生成一个新的 visibleList 数组。此时 shouldComponentUpdate 的“短路”优化宣告失败。

为了解决这一问题，记忆计算结果的办法出现了。一般把从 state 计算得到一份可用数据的行为称为 selector。

```js
const visibleListSelector = state => state.list.filter(
  item => (item.status === state.visibleFilter)
);
```

如果这样的 selector 具备记忆能力，即在其结果所依赖的部分数据未变更的情况下，直接返回先前的计算结果，那么前面提到的问题将迎刃而解。

reselect 就是实现了这样一个能力的 JavaScript 库。它的使用很简单，下面来改写一下上边的几个 selector。

```js
import { createSelector } from 'reselect';

const listSelector = state => state.list;
const visibleFilterSelector = state => state.visibleFilter;
const visibleListSelector = createSelector(
  listSelector,
  visibleFilterSelector,
  (list, visibleFilter) => list.filter(
    item => (item.status === visibleFilter)
  )
);
```

这里实现了 3 个 selector，visibleListSelector 由 listSelector 与 visibleFilterSelector 通过 createSelector 组合而成。reselect 的价值是提供了这种组合 selector 的能力，而且通过 createSelector 组合产生的 selector 具有记忆能力。

5. 容易忽视的细节

最后，在组件的实现中，一些很容易被忽视的细节，会趋于让相关组件的 shouldComponentUpdate 失效。他们的特点是，对于相同的内容，每次都创造并使用一个新的对象、函数，这一行为存在于前面提到的 selector 之外，典型的位置包括父组件的 render 方法、生成容器组件的 stateToProps 方法等。常见的例子：

- 在 render 中声明函数

```js
const onItemClick = id => console.log(id);
function List({ list }) {
  const items = list.map(
    item => (
      <Item key={item.id} onClick={() => onItemClick(item.id)}>{item.name}</Item>
    )
  );
  return (
    <p>{ items }</p>
  );
}
```

解决办法是可以将 onItemClick 传给 onClick，调用的时候，有 Item 将需要的数据传给 onItemClick。

但不得不承认，对于子组件 Item 来说，拿到一个通用的 onRemove 方法是不太合理的。所以也有一些其他解决方案的思路：提供一个具有记忆能力的绑定方法，相同参数返回相同绑定结果。

- 在 render 中使用函数绑定
使用 bind 的函数绑定与函数声明类似，例子略。解决办法是可以在 constructor 中 bind，或者将绑定行为实现为一个组件，参数不变的情况下不会重新 render。

笔者认为，绝大部分情况下，都不至于需要为了性能做这么多的妥协。

- object/array 字面量

```js
function Foo() {
  return (
    <Bar options={['a', 'b', 'c']} />
  );
}
```

解决办法就是将字面量保存在常量中即可。

### 合理拆分组件
