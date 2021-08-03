# React Fiber

## 2021 SegmentFault D-Day 大前端前沿技术实践

![image-20210801084357543](/Users/windscy/Library/Application Support/typora-user-images/image-20210801084357543.png)

![17](https://segmentfault.com/img/bVcTPhP)

-   框架设计：React 是一个极致的运行时方案，NG、Svelte、Solid 等是极致的编译时方案，Vue 介于二者之间，可运可编；

![18](https://segmentfault.com/img/bVcTPhY)

-   React Hooks 本质是一种 React 的 ui=F(state)，即 ui 是一种数据的设计哲学践行，是函数式编程的最佳实践，也更适配 Serverless 的 lambda 函数践行

    ![20](https://segmentfault.com/img/bVcTPhQ)

-   CPU 瓶颈：1、提供性能优化 API（底层减少运行时流程）；2、时间分片，如批处理及 startTransition 等（减少用户感知）。

-   IO 瓶颈：Suspense，全新 SSR

![image-20210801131231078](/Users/windscy/Library/Application Support/typora-user-images/image-20210801131231078.png)

-   前端框架的本质：所有的框架的 render 逻辑，本质就是 appendChild

## React Fiber 是什么？

**React 是一个用于构建用于界面的 JavaScript 库，其核心机制在于跟踪一个组件状态上的更新，并将更新后的状态反应到屏幕上，在 React 中我们将此过程称为 reconciliation。**

> 最早的`Fiber`官方解释来源于[2016 年 React 团队成员 Acdlite 的一篇介绍](https://github.com/acdlite/react-fiber-architecture)。

> React Fiber is an ongoing reimplementation of React's core algorithm. It is the culmination of over two years of research by the React team.
> React Fiber 是 React 核心算法的持续重新实现。它是 React 团队两年多研究的结晶。

[官网](https://zh-hans.reactjs.org/docs/faq-internals.html#what-is-react-fiber)中介绍，**Fiber 是 React16 中新的协调引擎。React Fiber 的目标是提高其对动画、布局和手势等领域的适用性。它的主要目的是使 Virtual DOM 可以进行增量式渲染：能够将渲染工作分成 chunks 并将其分散到多个帧上。**

### 其他主要功能

TODO

-   在新更新到来时暂停、中止或重用工作(work)的能力
-   能够为不同类型的更新分配优先级
-   新的并发原语

### 从不同角度看 Fiber

Fiber 包含三层含义：

1. 作为架构来说，之前 React15 的 Reconciler 采用递归的方式执行，数据保存在递归调用栈中，所以被称为 stack Reconciler。React16 的 Reconciler 基于 Fiber 节点实现，也就是说 Fiber 是 Reconciler 的重新实现。被称为 Fiber Reconciler。Fiber 的架构还提供了一种方便的方式来 track，schedule，pause and abort the work。

2. 作为静态的数据结构来说，每个 Fiber 节点对应一个 React Element，保存了该组件的类型(函数组件/类组件/原生组件...)、对应的 DOM 节点等信息。而最后生成的 Fiber nodes 又通过链表的方式连接成一棵完整的 Fiber tree。

3. 作为动态的工作单元(unit of work)来说，每个 Fiber 节点保存了本次更新中该组件改变的状态、要执行的工作 work(需要被删除/被插入页面中/被更新)

## Fiber 架构

### 老的 React 架构

React15 架构可以分为两层：

-   Reconciler：协调器，负责找出变化的组件
-   Renderer：渲染器，负责将变化的组件渲染到页面上

#### Reconciler

我们知道，在 React 中通过 `this.setState`、`this.forceUpdate`、`ReactDOM.render` 等 API 触发更新。

每当有更新发生时，Reconciler 会做如下工作：

-   调用函数组件、或 class 组件的 render 方法，将返回的 JSX 转化为虚拟 DOM
-   将虚拟 DOM 和上次更新时的虚拟 DOM 对比
-   通过对比找出本次更新中变化的虚拟 DOM
-   通知 Renderer 将变化的虚拟 DOM 渲染到页面上

> 你可以在[这里](https://zh-hans.reactjs.org/docs/codebase-overview.html#reconcilers)看到 React 官方对 **Reconciler** 的解释

对于 React15 中的 Reconciler，可参见[官方文档](https://reactjs.org/docs/reconciliation.html?source=post_page---------------------------#recursing-on-children)，其中讨论了许多关于递归的内容：

> 默认情况下，在 DOM 节点的子节点上递归时，React 只会同时迭代两个子节点列表，并在出现差异时生成一个 mutation。

#### Renderer

由于`React`支持跨平台，所以不同平台有不同的**Renderer**。我们前端最熟悉的是负责在浏览器环境渲染的**Renderer** —— [ReactDOM](https://www.npmjs.com/package/react-dom)。

除此之外，还有：

-   [ReactNative](https://www.npmjs.com/package/react-native)渲染器，渲染 App 原生组件
-   [ReactTest](https://www.npmjs.com/package/react-test-Renderer)渲染器，渲染出纯 Js 对象用于测试
-   [ReactArt](https://www.npmjs.com/package/react-art)渲染器，渲染到 Canvas, SVG 或 VML (IE8)

在每次更新发生时，**Renderer**接到**Reconciler**通知，将变化的组件渲染在当前宿主环境。

> 你可以在[这里](https://zh-hans.reactjs.org/docs/codebase-overview.html#renderers)看到`React`官方对**Renderer**的解释

#### React15 架构的缺点

React15 采用递归更新子组件的方式，由于递归执行的缺陷，所以注定了 React15 只能是同步更新，中途无法中断，如果递归更新的时间超过了 16ms，那么也就意味着在这一帧里渲染线程也就没有执行，浏览器就卡顿了。

对于这个问题，React 新架构的思想是，在浏览器每一帧的时间中，预留一些时间给 JS 线程，React 利用这部分时间更新组件，在[源码](https://github.com/facebook/react/blob/1fb18e22ae66fdb1dc127347e169e73948778e5a/packages/scheduler/src/forks/SchedulerHostConfig.default.js#L119)中，预留时间初始为 5ms。

当预留的时间不够用时，React 将线程控制权，交还给浏览器使其有时间渲染 UI，React 则等待下一帧时间到来继续被中断的工作。这样浏览器就有剩余时间执行样式布局和绘制，减少了掉帧的可能性。

> 这种将长任务分拆到每一帧中，一次执行一小段任务的操作，被称为**时间切片(time slice)**

![](/Users/windscy/Desktop/Wind/React/time-slice.jpeg)

```jsx
function App() {
    const len = 3000;
    return (
        <ul>
            {Array(len)
                .fill(0)
                .map((_, i) => (
                    <li>{i}</li>
                ))}
        </ul>
    );
}

const rootEl = document.querySelector('#root');
ReactDOM.render(<App />, rootEl);
```

![](https://react.iamkasong.com/img/long-task.png)

![](https://react.iamkasong.com/img/time-slice.png)

也就是说，我们需要将`同步的更新`替换为`异步可中断的更新`。

而在 React15 的架构中，由于 Reconciler 和 Renderer 是交替工作的，当前一个需要更新的节点进行了 Reconciler 和 Renderer，才会进入下一个需要更新节点的 Reconciler 和 Renderer，而整个过程都是同步的。如果其中某个节点中断更新，那么后续的节点工作都会被阻断。所以 React15 并不支持异步可中断的更新，因此 React 决定重写整个架构。

### 新的 React 架构

React16 架构可以分为三层：

-   Scheduler：调度器，调度任务的优先级，高优任务优先进入 Reconciler。
-   Reconciler：协调器，负责找出变化的组件。
-   Renderer：渲染器，负责将变化的组件渲染到页面上。

#### Scheduler

既然我们想使用预留给 JS 线程的时间来更新组件，那么我们就要有一种机制，来作为在 React 中中断任务的标准，当浏览器有剩余时间时通知我们。

其实部分浏览器已经实现了这个 API，这就是`requestIdleCallback`。但是由于以下因素，React 放弃使用：

-   浏览器兼容性
-   触发频率不稳定，受很多因素影响。比如当我们切换 tab 后，之前 tab 注册的 `requestIdleCallback`触发的频率就会变得很低
-   极低的 fps，只有 20

基于以上原因，React 实现了功能更加完备的 `requestIdleCallback polyfill`，这就是 `Scheduler`。除了在空闲时触发回调的功能外，`Scheduler` 还提供了多种调度优先级任务设置。

> `Scheduler`是独立于 React 的库。

#### Reconciler

从[官网](https://zh-hans.reactjs.org/docs/codebase-overview.html#fiber-reconciler)中可以知道，Fiber Reconciler 是一个新尝试，致力于解决 stack Reconciler 中固有的问题，同时解决一些历史遗留问题。Fiber 从 React16 开始变成了默认的 Reconciler。

> 从依赖内置堆栈的同步递归模式到带有链表和指针的异步模型。

> 可以随意中断调用栈并实现游动操作栈帧。

> 可以将单个 fiber 视为虚拟堆栈帧。

它的主要目标是：

-   能够把可中断的任务切片处理。
-   能够调度优先级，重置并复用任务。
-   能够在父元素与子元素之间交错处理，以支持 React 中的布局。
-   能够在 `render()` 中返回多个元素。
-   更好的支持错误边界。

在 React15 中，Reconciler 是递归处理虚拟 DOM 的。而在 React16 以后重写了 diff 算法，也就是新的 Fiber Reconciler，它使用了链表来组织一个个由 React.createElement(JSX) 生成的 React Element，形成一棵 Fiber tree。

从如下代码可见，更新工作从递归变成了可中断的循环过程。每次循环都会调用 `shouldYield` 判断当前是否有剩余时间。

```js
/** @noinline */
function workLoopConcurrent() {
    // Perform work until Scheduler asks us to yield
    while (workInProgress !== null && !shouldYield()) {
        workInProgress = performUnitOfWork(workInProgress);
    }
}
```

那么 React16 是如何解决中断更新时 DOM 渲染不完全的问题呢？

在 React16 中，Reconciler 与 Renderer 不再是交替工作。当 Scheduler 将任务交给 Reconciler 后，Reconciler 会为变化的虚拟 DOM 打上代表增/删/更新的标记，类似这样：

```js
export const Placement = /*             */ 0b0000000000010;
export const Update = /*                */ 0b0000000000100;
export const PlacementAndUpdate = /*    */ 0b0000000000110;
export const Deletion = /*              */ 0b0000000001000;
```

整个 Scheduler 与 Reconciler 的工作都在内存中进行。只有当所有组件都完成 Reconciler 的工作，才会统一交给 Renderer。

#### Renderer

Renderer 根据 Reconciler 为虚拟 DOM 打的标记，同步执行对应的 DOM 操作。

> 实际上，由于 Scheduler 和 Reconciler 都是和平台无关的，所以 React 为他们单独发了一个包 [react-Reconciler](https://www.npmjs.com/package/react-reconciler)。

## Fiber 单链表树

-   child — 第一个子节点的引用
-   sibling — 第一个兄弟节点的引用
-   return — 父节点的引用

> 这里需要提一下，为什么父级指针叫做 return 而不是 parent 或者 father 呢？因为作为一个工作单元，return 指节点执行完 completeWork 后会返回的下一个节点。子 Fiber 节点及其兄弟节点完成工作后会返回其父级节点，所以用 return 指代父级节点。

在 React 新的 reconciliation 算法的上下文中，具有这些字段的数据结构称为 Fiber。

![](https://images.indepth.dev/images/2019/07/image-48.png)

```jsx
let root = fiber;
let node = fiber;
while (true) {
    // Do something with node
    if (node.child) {
        node = node.child;
        continue;
    }

    if (node === root) {
        return;
    }

    while (!node.sibling) {
        if (!node.return || node.return === root) {
            return;
        }
        node = node.return;
    }

    node = node.sibling;
}
```

**Fiber 单链表树可以暂停遍历并组织堆栈增长。**

如果我们使用这个实现检查调用堆栈，我们将看到一下内容：

![](https://images.indepth.dev/images/2019/07/callstack.gif)

如你所见，当我们沿着树向下走时，堆栈不会增长。但是，如果现在将 debugger 放入 doWork 函数和日志节点名称中，我们将看到一下内容：

![](https://images.indepth.dev/images/2019/07/callstack2.gif)

它看起来像浏览器的调用堆栈。因此，使用此算法，我们可以用我们自己的实现有效地替换浏览器对调用堆栈的实现。即：**Fiber 是堆栈的重新实现，专门用于 React 组件。可以将单个 Fiber 视为 Virtual Stack Frame。**

由于我们现在通过保持对充当顶帧的节点的引用来控制堆栈：

```jsx
function walk(o) {
  let root = o
  let current = o

  while(true) {
    ...
    current = child
    ...
    current = current.return
    ...
    current = current.sibling
  }
}
```

**我们可以随时停止遍历，稍后在继续。**这正是我们希望能够使用新 `requestIdleCallback`API 的条件。

**也就是说，我们了解了 Fiber 背后的设计思想，我们就顺带了解了调用堆栈的过程；我们了解了 Scheduler，我们就了解了浏览器执行时每一帧的工作过程。**

## Fiber 动态工作单元

TODO

## Fiber 静态数据结构

TODO

## Fiber 架构的工作原理

### React performs work 的两个阶段

React 执行工作主要分为两个主要阶段：render 和 commit。

-   Reconciler 工作的阶段被称为 render 阶段。因为在该阶段会调用组件的 render 方法。
-   Renderer 工作的阶段被称为 commit 阶段。就像你完成一个需求的编码后执行 git commit 提交代码。commit 阶段会把 render 阶段提交的信息渲染在页面上。
-   render 与 commit 阶段统称为 work，即 React 在工作中。相对应的，如果任务正在 Scheduler 内调度，就不属于 work。
-   Reconciler 阶段的工作可以异步执行。React 可以根据`可用的时间`处理一个或多个 fiber nodes，然后停下来，把已完成的工作藏起来，让一些事件执行。然后从它停止的地方继续。但有时，它可能需要放弃已完成的工作，并重新从头开始。由于在此阶段执行的工作不会导致任何用户可见的更改(例如 DOM 更新)，因此这些暂停成为可能。
-   Renderer 阶段总是同步的。这是因为在此阶段执行的工作会导致用户可见的更改，这也是 React 需要一次性完成更新的原因。

### 什么是双缓存？

当我们生成了 Fiber tree，我们就得到了对应的 DOM tree，接下来我们就探讨如何更新 DOM。

更新 DOM 需要用到被称为 "双缓存" 的技术。

当我们用 `canvas` 绘制动画，每一帧绘制前都会调用 ctx.clearRect 清除上一帧的画面。

如果当前帧画面计算量比较大，导致清除上一帧画面到绘制当前帧画面之间有较长间隙，就会出现白屏。

为了解决这个问题，我们可以在内存中绘制当前帧动画，绘制完毕后直接用当前帧替换上一帧画面，由于省去了两帧替换间的计算时间，不会出现从白屏到出现画面的闪烁情况。

这种**在内存中构建并直接替换**的技术叫做[双缓存](https://baike.baidu.com/item/%E5%8F%8C%E7%BC%93%E5%86%B2)。

React 使用"双缓存"来完成 Fiber tree 的构建与替换，对应着 DOM tree 的创建与更新。

### 双缓存 Fiber tree

在 React 中最多会存在两棵 `Fiber 树`。当前屏幕上显示内容对应的 `Fiber 树` 称为 `current Fiber 树`，正在内存中构建的 `Fiber 树` 称为 `workInProgress Fiber 树`。

`current Fiber树`中的 Fiber 节点被称为 `current fiber`，`workInProgress Fiber 树`中的 Fiber 节点被称为 `workInProgress fiber`，它们通过 `alternate` 属性连接。

```js
currentFiber.alternate === workInProgressFiber;
workInProgressFiber.alternate === currentFiber;
```

React 应用的根节点通过使 current 指针在不同 Fiber 树的 rootFiber 间切换来完成 current Fiber 树指向的切换。

即当 workInProgress Fiber 树构建完成交给 Renderer 渲染在页面上后，应用根节点的 current 指针指向 workInProgress Fiber 树，此时 workInProgress Fiber 树就变为 current Fiber 树。

每次状态更新都会产生新的 workInProgress Fiber 树，通过 current 与 workInProgress 的替换，完成 DOM 更新。

### mount 时的构建/替换流程

考虑中如下例子：

```jsx
function App() {
    const [num, add] = useState(0);
    return <p onClick={() => add(num + 1)}>{num}</p>;
}

ReactDOM.render(<App />, document.getElementById('root'));
```

1. 首次执行 ReactDOM.render 会创建 fiberRootNode(源码中叫 fiberRoot)和 rootFiber。其中 fiberRootNode 是整个应用的根节点，rootFiber 是 `<App />` 所在组件树的根节点。

> 之所以要区分 fiberRootNode 和 rootFiber，是因为在应用中我们可以多次调用 ReactDOM.render 渲染不同的组件树，他们会拥有不同的 rootFiber。但是整个应用的根节点只有一个，那就是 fiberRootNode。

fiberRootNode 的 current 会指向当前页面上已渲染内容对应 Fiber 树，即 current Fiber 树。

![](https://react.iamkasong.com/img/rootfiber.png)

```js
fiberRootNode.current = rootFiber;
```

由于是首屏渲染，页面中还没有挂载任何 DOM，所以 fiberRootNode.current 指向的 rootFiber 没有任何 `子Fiber节点`(即 current Fiber 树为空)。

2. 接下来进入 render 阶段，也就是 Reconciler 阶段，根据组件返回的 JSX 在内存中依次创建 Fiber 节点并连接在一起构建 Fiber 树，被称为 workInProgress Fiber 树。

![](https://react.iamkasong.com/img/workInProgressFiber.png)

在构建 workInProgress Fiber 树时会尝试复用 current Fiber 树中已有的 Fiber 节点内的属性，在首屏渲染时只有 rootFiber 存在对应的 current fiber(即 rootFiber.alternate)。

3. fiberRootNode 的 current 指针指向 workInProgress Fiber 树 使其变为 current Fiber 树。

![](https://react.iamkasong.com/img/wipTreeFinish.png)

### update 时的构建/替换流程

1. 当我们点击 p 节点触发状态更新，会开启一次新的 `render 阶段` 并构建一棵新的 `workInProgress Fiber 树`。

![](https://react.iamkasong.com/img/wipTreeUpdate.png)

和 mount 时一样，workInProgress fiber 的创建可以复用 current Fiber 对应的节点数据。

> 这个决定是否复用的过程就是 Diff 算法。

2. workInProgress Fiber 树在 render 阶段完成构建后进入 commit 阶段渲染到页面上。渲染完毕后，workInProgress Fiber 树变为 current fiber 树。

![](https://react.iamkasong.com/img/currentTreeUpdate.png)

## 从同步不可中断的递归到异步可中断的 Fiber，哪个耗时？

基本上 React Fiber 的想法是不使用 JavaScript 堆栈，而是展开我们通常在堆栈上执行的操作并将其放入堆对象中。

这受到 KC 在 OCaml 并发性方面的工作的启发。

```js
// noprotect

function fib1(n) {
    if (n <= 2) {
        return 1;
    } else {
        var a = fib1(n - 1);
        var b = fib1(n - 2);
        return a + b;
    }
}

function fib2(n) {
    var fiber = { arg: n, returnAddr: null, a: 0 /* b is tail call */ };
    rec: while (true) {
        if (fiber.arg <= 2) {
            var sum = 1;
            while (fiber.returnAddr) {
                fiber = fiber.returnAddr;
                if (fiber.a === 0) {
                    fiber.a = sum;
                    fiber = { arg: fiber.arg - 2, returnAddr: fiber, a: 0 };
                    continue rec;
                }
                sum += fiber.a;
            }
            return sum;
        } else {
            fiber = { arg: fiber.arg - 1, returnAddr: fiber, a: 0 };
        }
    }
}

var p = performance;

var c = 33;

for (var i = 0; i < 10; i++) {
    fib1(c);
    fib1(c);
    fib2(c);
    fib2(c);
}

fib1(c);
console.log('result stack', fib1(c));
fib2(c);
console.log('result fiber', fib2(c));

var s1 = p.now();
fib1(c);
var e1 = p.now();

var s2 = p.now();
fib2(c);
var e2 = p.now();

console.log('stack', e1 - s1);
console.log('fiber', e2 - s2);

var s1 = p.now();
fib1(c);
var e1 = p.now();

var s2 = p.now();
fib2(c);
var e2 = p.now();

console.log('stack', e1 - s1);
console.log('fiber', e2 - s2);
```

上面的例子表明，在最极端的示例中，展开堆栈的开销不到 2 倍。通过较低级别的调整，它可能会更加优化。

这种比较可能看起来很高，但对于实际代码，它要少得多，因为这基本上只是衡量开销。这也仅仅与使用堆栈进行渲染并且针对初始渲染进行优化的 React 版本进行比较。即不是 React 目前正在做的事情。

## Fiber Node 和 JSX

一旦模板通过 JSX 编译器，你最后会得到一堆 React Element，这才是 React Components 的 render 方法返回的真正的内容。

```jsx
class ClickCounter extends React.Component {
    render() {
        return [
            <button key="1" onClick={this.handleClick}>
                Update counter
            </button>,
            <span key="2">{this.state.count}</span>,
        ];
    }
}
```

```jsx
class ClickCounter {
    render() {
        return [
            React.createElement(
                'button',
                {
                    key: '1',
                    onClick: this.onClick,
                },
                'Update counter'
            ),
            React.createElement(
                'span',
                {
                    key: '2',
                },
                this.state.count
            ),
        ];
    }
}
```

在 reconciliation 过程中，从 render 方法返回的每个 React Element 的 data 都会合并到 Fiber Nodes tree 中。每个 React Element 都有一个对应的 Fiber Node。不同于 React Element，Fibers 并不会在每次渲染时都重新创建。这些是保存 components state 和 DOM 的稳定数据结构的根本。

当一个 React element 第一次被转换成一个 fiber node，React 使用来自 element 中的 data 通过 [createFiberFromTypeAndProps](https://github.com/facebook/react/blob/769b1f270e1251d9dbdce0fcbd9e92e502d059b8/packages/react-reconciler/src/ReactFiber.js#L414) 函数创建一个 fiber。在随后的更新中，React 重用了 fiber node 并仅使用来自 React element 中对应的 的 data 更新必要的 properties。React 可能还会根据 key prop 移动层级（hierarchy）上的 node，或者如果对应的 React element 不再从 render method 中返回，则删除它。

因为 React 为每个 React Element 创建了一个 fiber，并且因为我们有了这些 tree of elements，所以我们将有一个 tree of fiber nodes。在我们的示例中，他看起来像这样：

![](https://miro.medium.com/max/555/1*cLqBZRht7RgR9enHet_0fQ.png)

JSX 是一种描述当前组件内容的数据结构，他不包含组件 schedule、reconcile、render 所需的相关信息。

比如如下信息就不包括在 JSX 中：

-   组件在更新中的优先级
-   组件的 state
-   组件被打上的用于 Renderer 的标记

这些内容都包含在 Fiber 节点中。

所以，在组件 mount 时，Reconciler 根据 JSX 描述的组件内容生成组件对应的 Fiber 节点。

在 update 时，Reconciler 将 JSX 与 Fiber 节点保存的数据对比，生成组件对应的 Fiber 节点，并根据对比结果为 Fiber 节点打上标记。

## Fiber 的另外一个名字

TODO

## 代数效应

TODO
