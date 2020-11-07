`React16`除了新增了`Hooks`Api外，在核心架构上也发生了很大变化----同步更新重构为异步可中断更新。
也就是`React` 的 `Fiber`架构。

首先让我们来回忆一下React15的架构：
## React 架构演进
### React15 架构
#### React15架构可以分为两层：

- Reconciler（协调器）—— 负责找出变化的组件
- Renderer（渲染器）—— 负责将变化的组件渲染到页面上
##### Reconciler（协调器）
我们知道，在`React`中可以通过`this.setState`、`this.forceUpdate`、`ReactDOM.render`等API触发更新。

每当有更新发生时，`Reconciler`会做如下工作：

- 调用函数组件、或class组件的render方法，将返回的JSX转化为虚拟DOM
- 从父节点（Virtual DOM）开始向下遍历,将虚拟DOM和上次更新时的虚拟DOM对比
- 通过对比找出本次更新中变化的虚拟DOM
- 通知`Renderer`将变化的虚拟DOM渲染到页面上

它的执行过程很像函数的递归，所以，Reac15的`Reconciler`也称为`Stack Reconciler`。
##### Renderer（渲染器）
由于`React`支持跨平台，所以不同平台有不同的**Renderer**。我们前端最熟悉的是负责在浏览器环境渲染的Renderer —— `ReactDOM`。

除此之外，还有：

- ReactNative渲染器，渲染App原生组件
- ReactTest渲染器，渲染出纯Js对象用于测试
- ReactArt渲染器，渲染到Canvas, SVG 或 VML (IE8)
在每次更新发生时，Renderer接到Reconciler通知，将变化的组件渲染在当前宿主环境。
#### React15框架下状态更新的特点
- 一旦开始递归，无法停止
- Reconciler和Render交替工作
- mount的组件会调用mountComponent，update的组件会调用updateComponent。这两个方法都会递归更新子组件。
#### React15架构的缺点

主流的浏览器刷新频率为60Hz，即每（1000ms / 60Hz）16.6ms浏览器刷新一次。
在每16.6ms时间内，需要完成如下工作：
```
JS脚本执行---样式布局----样式绘制
```
我们知道，JS可以操作DOM，GUI渲染线程与JS线程是互斥的。所以JS脚本执行和浏览器布局、绘制不能同时执行。
当JS执行时间过长，超出了16.6ms，这次刷新就没有时间执行样式布局和样式绘制了。

对于用户在输入框输入内容这个行为来说，就体现为按下了键盘按键但是页面上不实时显示输入。

对于React的更新来说，由于递归执行，所以更新一旦开始，中途就无法中断。当层级很深时，递归更新时间超过了16ms，用户交互就会卡顿。

### React16架构

正式由于React15架构无法中断的同步更新，React团队重构了React的整体架构，实现了异步可中断更新。

在React 16 架构中，将更新过程切割为多个步骤，分批完成。也就是说在完成一部分任务后，看是否还有剩余时间，如果有继续下一个任务；如果没有，挂起当前任务，将控制权交回给浏览器。让浏览器有时间进行页面渲染，等浏览器完成任务之后再继续之前的未完成的任务。

React16架构可以分为三层：
- Scheduler（调度器）—— 产生更新，调度任务（优先级，高优任务优先进入Reconciler； 浏览器空闲）
- Reconciler（协调器）—— 负责找出变化的组件
- Renderer（渲染器）—— 负责将变化的组件渲染到页面上

#### Scheduler（调度器）

Scheduler的作用有两个：
- 调度任务的优先级，高优任务优先进入Reconciler；
- 浏览器空闲时触发回调 ---- 通过update.lane区分优先级


浏览器空闲触发回调的原理与[requestIdleCallback](https://developer.mozilla.org/zh-CN/docs/Web/API/Window/requestIdleCallback)相同，部分浏览器已经实现了这个API。

> window.requestIdleCallback()方法将在浏览器的空闲时段内调用的函数队列。这使开发者能够在主事件循环上执行后台和低优先级工作，而不会影响延迟关键事件，如动画和输入响应。函数一般会按先进先调用的顺序执行，然而，如果回调函数指定了执行超时时间timeout，则有可能为了在超时前执行函数而打乱执行顺序。

>你可以在空闲回调函数中调用requestIdleCallback()，以便在下一次通过事件循环之前调度另一个回调。


浏览器是一帧一帧执行的，在两个执行帧之间，主线程通常会有一小段空闲时间，requestIdleCallback可以在这个空闲期（Idle Period）调用空闲期回调（Idle Callback），执行一些任务。

![requestIdleCallback.png](https://notes.s3.cn-north-1.jdcloud-oss.com/requestIdleCallback.png?AWSAccessKeyId=A850436F52857FDD372AF2752FBD98B0&Expires=1666928069&Signature=LTIDbfU4BBP%2B7QPHts8e6C1DqOo%3D)

- 低优先级任务由requestIdleCallback处理；
- 高优先级任务，如动画相关的由requestAnimationFrame处理；
- requestIdleCallback可以在多个空闲期调用空闲期回调，执行任务；
- requestIdleCallback方法提供 deadline，即任务执行限制时间，以切分任务，避免长时间执行，阻塞UI渲染而导致掉帧；

##### Reconciler（协调器）
在React15中Reconciler是递归处理虚拟DOM的, 在React16中递归更新变成了可中断的循环过程，每次循环都会调用`shouldYiele`判断是否有剩余时间。

```js
/** @noinline */
function workLoopConcurrent() {
  // Perform work until Scheduler asks us to yield
  while (workInProgress !== null && !shouldYield()) {
    workInProgress = performUnitOfWork(workInProgress);
  }
}
```
那么React16是如何解决中断更新时DOM渲染不完全的问题呢？

在React16中，Reconciler与Renderer不再是交替工作。当Scheduler将任务交给Reconciler后，Reconciler会为变化的虚拟DOM打上代表增/删/更新的标记，类似这样：
```js
export const Placement = /*             */ 00000000000010;
export const Update = /*                */ 0b0000000000100;
export const PlacementAndUpdate = /*    */ 0b0000000000110;
export const Deletion = /*              */ 0b0000000001000;
```
整个Scheduler与Reconciler的工作都在内存中进行。只有当所有组件都完成Reconciler的工作，才会统一交给Renderer。
##### Renderer（渲染器）

Renderer根据Reconciler为虚拟DOM打的标记，同步执行对应的DOM操作。

我们用一个小例子来看下，在React16架构中整个更新流程：
```js
class App extends React.Component {
  constructor(...props) {
    super(...props);
    this.state = {
      count: 1
    }
  }
  onClick() {
    this.setState({
      count: this.state.count + 1
    })
  }
  render() {
     return (
       <ul>
         <button onClick={() => this.onClick()}>乘以{this.state.count}</button>
         <li>{1 * this.state.count}</li>
         <li>{2 * this.state.count}</li>
         <li>{3 * this.state.count}</li>
       </ul>
     )
  }
}

ReactDOM.render(<App/>, document.getElementById('app'));
```
![cd3fe2529c8c6f8d7f0bdbace8b84de4.png](evernotecid://4360BD77-FECC-45AB-A570-866A031A45EF/appyinxiangcom/22992045/ENResource/p64)

其中红框中的步骤随时可能由于以下原因被中断：

- 有其他更高优任务需要先更新
- 当前帧没有剩余时间
由于红框中的工作都在内存中进行，不会更新页面上的DOM，所以即使反复中断，用户也不会看见更新不完全的DOM




## React Fiber架构原理

### 什么是`React Fiber`


前面讲解React16架构时，我们说 React16 实现了异步可中断更新。异步可中断更新可以理解为：更新在执行过程中可能会被打断（浏览器时间分片用尽或有更高优任务插队），当可以继续执行时恢复之前执行的中间状态。
`React Fiber`就是实现这个过程的状态更新机制。

#### React Fiber的三层含义

1. 作为架构来说，之前React15的Reconciler采用递归的方式执行，数据保存在递归调用栈中，所以被称为stack Reconciler。React16的Reconciler基于Fiber节点实现，被称为Fiber Reconciler。

2. 作为静态的数据结构来说，每个Fiber节点对应一个React element，保存了该组件的类型（函数组件/类组件/原生组件...）、对应的DOM节点等信息。

3. 作为动态的工作单元来说，每个Fiber节点保存了本次更新中该组件改变的状态、要执行的工作（需要被删除/被插入页面中/被更新...）。

#### React Fiber结构

```js
function FiberNode(
  tag: WorkTag,
  pendingProps: mixed,
  key: null | string,
  mode: TypeOfMode,
) {
  // 作为静态数据结构的属性
  this.tag = tag;
  this.key = key;
  // 大部分情况同type，某些情况不同，比如FunctionComponent使用React.memo包裹
  this.elementType = null;
  // 对于 FunctionComponent，指函数本身，对于ClassComponent，指class，对于HostComponent，指DOM节点tagName

  this.type = null;
  // Fiber对应的真实DOM节点
  this.stateNode = null;

  // 用于连接其他Fiber节点形成Fiber树
  this.return = null;
  this.child = null;
  this.sibling = null;
  this.index = 0;

  this.ref = null;

  // 作为动态的工作单元的属性
  this.pendingProps = pendingProps;
  this.memoizedProps = null;
  this.updateQueue = null;
  this.memoizedState = null;
  this.dependencies = null;

  this.mode = mode;
  // 保存本次更新会造成的DOM操作
  this.effectTag = NoEffect;
  
  //单链表结构，方便遍历fiber树上有副作用的节点
  this.nextEffect = null;
  this.firstEffect = null;
  this.lastEffect = null;

  // 调度优先级相关
  this.lanes = NoLanes;
  this.childLanes = NoLanes;

  // 指向该fiber在另一次更新时对应的fiber
  this.alternate = null;
  }
```

作为架构，Fiber节点保存了将 Fiber连接成树的属性,
如下的组件，会形成下图的树形结构：
```js
function App() {
  return (
    <div>
      i am
      <span>xiaohong</span>
    </div>
  )
}
```

![82a1b7d4ba8c3116be50a6a0946423b4.png](evernotecid://4360BD77-FECC-45AB-A570-866A031A45EF/appyinxiangcom/22992045/ENResource/p65)

##### Fiber架构的链表（effectList）


React在对每个Fiber节点进行diff运算之后会为产生更新的节点打上effectTag,作为DOM更新的依据

实现高性能，React构建了一条带有effectTag的Fiber节点的线性列表

借用React团队成员Dan Abramov的话：effectList相较于Fiber树，就像圣诞树上挂的那一串彩灯。

![5d04ec0afb541cf931c75bff6813535b.png](evernotecid://4360BD77-FECC-45AB-A570-866A031A45EF/appyinxiangcom/22992045/ENResource/p66)
所有有effectTag的Fiber节点都会被追加在effectList中，最终形成一条以rootFiber.firstEffect为起点的单向链表。
![e55753e3a09d80741a5b136d3ab6b4e1.png](evernotecid://4360BD77-FECC-45AB-A570-866A031A45EF/appyinxiangcom/22992045/ENResource/p67)
##### Fiber架构的双缓存机制

在React中最多会同时存在两棵Fiber树。当前屏幕上显示内容对应的Fiber树称为current Fiber树，正在内存中构建的Fiber树称为workInProgress Fiber树。
current Fiber树中的Fiber节点被称为current fiber，workInProgress Fiber树中的Fiber节点被称为workInProgress fiber，他们通过alternate属性连接。


React应用的根节点通过current指针在不同Fiber树的rootFiber间切换来实现Fiber树的切换。

当workInProgress Fiber树构建完成交给Renderer渲染在页面上后，应用根节点的current指针指向workInProgress Fiber树，此时workInProgress Fiber树就变为current Fiber树。

每次状态更新都会产生新的workInProgress Fiber树，通过current与workInProgress的替换，完成DOM更新。

### React 工作流程

#### 首屏渲染



