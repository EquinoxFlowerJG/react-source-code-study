# 四、初始化渲染

要将 React 元素渲染到页面中，分为两个阶段，render 阶段和 commit 阶段。

render 阶段负责创建 Fiber 数据结构并为 Fiber 节点打标记，标记当前 Fiber 节点要进行的 DOM 操作。

commit 阶段负责根据 Fiber 节点标记 ( effectTag ) 进行相应的 DOM 操作。

大体的流程是：

1. render 阶段：

   - render 阶段就是协调层负责的阶段，在这个阶段要为每个 React 元素构建对应的 Fiber 对象，在构建 Fiber 对象的过程中，还要为此 Fiber 对象创建 DOM 对象，并且还要为 fiber 对象添加 effectTag 属性（标注 Fiber 对象对应的 DOM 对象要进行什么操作）。

   - 这个新构建的 Fiber 对象，我们称之为 workInProgress Fiber 树，也可以理解为待提交的 Fiber 树。

   - 当 render 阶段结束后，这个 Fiber 树会被保存在 fiberRoot 对象中。

   - 然后就可以进入 commit 阶段了。

2. commit 阶段：

   - 在 commit 阶段，会先获取 render 阶段的工作结果（就是获取到保存在 fiberRoot 对象中新构建的 workInProgress Fiber 树）。

   - 接下来就是根据 Fiber 对象中的 effectTag 属性进行相应的 DOM 操作了。

大体的原理是这样的，只不过源码中还处理了很多细节的问题。

## （一）render 阶段

### 一）render

```js
// src/react/packages/react-dom/src/client/ReactDOMLegacy.js
// render 方法

// ⭐️ Line 287~315 ⭐️
/**
 * React 向用户开放的渲染入口
 * @param {React$Element<any>} element 要进行渲染的 ReactElement，实际上就是 createElement 的返回值
 * @param {Container} container 渲染容器，就是 id 为 root 的 div 元素
 * @param {?Function} callback 渲染完成后执行的回调函数，可选参数
 * @return {*}
 */
export function render(
  element: React$Element<any>,
  container: Container,
  callback: ?Function
) {
  // 检测 container 是否是符合要求的渲染容器
  // 即检测 container 是否是真实的 DOM 对象
  // 如果不符合要求就报错
  invariant(
    isValidContainer(container),
    "Target container is not a DOM element."
  );
  // 在开发环境下（暂时忽略）
  if (__DEV__) {
    const isModernRoot =
      isContainerMarkedAsRoot(container) &&
      container._reactRootContainer === undefined;
    if (isModernRoot) {
      console.error(
        "You are calling ReactDOM.render() on a container that was previously " +
          "passed to ReactDOM.createRoot(). This is not supported. " +
          "Did you mean to call root.render(element)?"
      );
    }
  }
  return legacyRenderSubtreeIntoContainer(
    // 父组件：初始渲染没有父组件就传递 null 占位
    null,
    element,
    container,
    // 是否为服务器端渲染： false-不是服务器端渲染；true-是服务器端渲染
    // 如果是服务器端渲染，就需要复用 container 内部的 DOM 元素；如果不是服务器端渲染，则不需要复用，需要删除 container 内部的内容
    false,
    callback
  );
}
```

### 二）isValidContainer

```js
// src/react/packages/react-dom/src/client/ReactDOMRoot.js
// isValidContainer 方法

// ⭐️ Line 162~171 ⭐️
/**
 * 判断 node 是否是符合要求的 DOM 节点
 * 1. node 可以是元素节点
 * 2. node 可以是 document 节点
 * 3. node 可以是文档碎片节点
 * 4. node 可以是注释节点，但注释内容必须是 react-mount-point-unstable ：
 *    react 内部可以找到注释节点的父级，通过调用父级元素的 insertBefore 方法将 element 插入到注释节点的前面
 */
export function isValidContainer(node: mixed): boolean {
  return !!(
    node &&
    (node.nodeType === ELEMENT_NODE ||
      node.nodeType === DOCUMENT_NODE ||
      node.nodeType === DOCUMENT_FRAGMENT_NODE ||
      (node.nodeType === COMMENT_NODE &&
        (node: any).nodeValue === " react-mount-point-unstable "))
  );
}
```

### 三）初始化 FiberRoot

#### 1、legacyRenderSubtreeIntoContainer

```js
// src/react/packages/react-dom/src/client/ReactDOMLegacy.js

// legacyRenderSubtreeIntoContainer 方法
// ⭐️ Line 175~222 ⭐️
/**
 * 将子树渲染到容器中（初始化 Fiber 数据结构：创建 fiberRoot 及 rootFiber）
 *
 * @param {?React$Component<any, any>} parentComponent 父组件，初始渲染传入了 null
 * @param {ReactNodeList} children render 方法中的第一个参数，要渲染的 ReactElement
 * @param {Container} container 渲染容器
 * @param {boolean} forceHydrate true 为服务器端渲染，false 为客户端渲染
 * @param {?Function} callback 组件渲染完成后需要执行的回调函数
 * @return {*}
 */
function legacyRenderSubtreeIntoContainer(
  parentComponent: ?React$Component<any, any>,
  children: ReactNodeList,
  container: Container,
  forceHydrate: boolean,
  callback: ?Function
) {
  if (__DEV__) {
    topLevelUpdateWarnings(container);
    warnOnInvalidCallback(callback === undefined ? null : callback, "render");
  }

  /**
   * 检测 container 是否是已经初始化过的渲染容器
   * react 在初始渲染时会为最外层容器添加 _reactRootContainer 属性
   * react 会根据此属性采取不同的渲染方式
   * root 不存在表示初始渲染
   * root 存在表示更新
   */
  // 获取 container 容器对象下是否有 _reactRootContainer 属性
  let root: RootType = (container._reactRootContainer: any);
  // 即将存储根 Fiber 对象
  let fiberRoot;
  if (!root) {
    // 初始渲染
    // 初始化根 Fiber 数据结构
    // 为 container 容器添加 _reactRootContainer 属性
    // 在 _reactRootContainer 对象中有一个属性叫做 _internalRoot
    // _internalRoot 属性值即为 fiberRoot ，表示根节点 Fiber 数据结构
    // legacyCreateRootFromDOMContainer 方法创建了 fiberRoot 和 rootFiber，方法中调用了其他方法：
    // createLegacyRoot 方法中也调用了其他方法：
    // new ReactDOMBlockingRoot -> this._internalRoot 方法中也调用了其他方法：
    // createRootImpl
    root = container._reactRootContainer = legacyCreateRootFromDOMContainer(
      container,
      forceHydrate
    );
    fiberRoot = root._internalRoot;
    if (typeof callback === "function") {
      const originalCallback = callback;
      callback = function () {
        const instance = getPublicRootInstance(fiberRoot);
        originalCallback.call(instance);
      };
    }
    // Initial mount should not be batched.
    unbatchedUpdates(() => {
      updateContainer(children, fiberRoot, parentComponent, callback);
    });
  } else {
    fiberRoot = root._internalRoot;
    if (typeof callback === "function") {
      const originalCallback = callback;
      callback = function () {
        const instance = getPublicRootInstance(fiberRoot);
        originalCallback.call(instance);
      };
    }
    // Update
    updateContainer(children, fiberRoot, parentComponent, callback);
  }
  return getPublicRootInstance(fiberRoot);
}
```

#### 2、legacyCreateRootFromDOMContainer

```js
// src/react/packages/react-dom/src/client/ReactDOMLegacy.js

// legacyCreateRootFromDOMContainer 方法
// ⭐️ Line 113~160 ⭐️
/**
 * 判断是否是服务器端渲染
 * 如果不是服务器端渲染，则清空 container 容器中的节点
 */
function legacyCreateRootFromDOMContainer(
  container: Container,
  forceHydrate: boolean
): RootType {
  // container => <div id="root"></div>
  // 检测是否为服务器端渲染
  const shouldHydrate =
    forceHydrate || shouldHydrateDueToLegacyHeuristic(container);
  // 如果不是服务器端渲染
  if (!shouldHydrate) {
    let warned = false;
    let rootSibling;
    // 开启循环，删除 container 容器中的节点
    while ((rootSibling = container.lastChild)) {
      // 在开发环境中
      if (__DEV__) {
        if (
          !warned &&
          rootSibling.nodeType === ELEMENT_NODE &&
          (rootSibling: any).hasAttribute(ROOT_ATTRIBUTE_NAME)
        ) {
          warned = true;
          console.error(
            "render(): Target node has markup rendered by React, but there " +
              "are unrelated nodes as well. This is most commonly caused by " +
              "white-space inserted around server-rendered markup."
          );
        }
      }
      // 删除 container 容器中的节点
      container.removeChild(rootSibling);
      /**
       * 思考：为什么要清除 container 中的元素？
       * 因为：
       * 有时需要在 container 中放置一些占位图或者 loading 图以提高首屏加载体验
       * 就无可避免的要向 container 中加入 html 标记
       * 在讲 ReactElement 渲染到 container 之前，必然要先清空 container
       * 因为占位图和 ReactElement 不能同时显示
       *
       * 在加入占位代码时，最好只有一个父级元素，可以减少内部代码的循环次数以提高性能
       */
    }
  }
  if (__DEV__) {
    if (shouldHydrate && !forceHydrate && !warnedAboutHydrateAPI) {
      warnedAboutHydrateAPI = true;
      console.warn(
        "render(): Calling ReactDOM.render() to hydrate server-rendered markup " +
          "will stop working in React v17. Replace the ReactDOM.render() call " +
          "with ReactDOM.hydrate() if you want React to attach to the server HTML."
      );
    }
  }

  return createLegacyRoot(
    container,
    shouldHydrate
      ? {
          hydrate: true,
        }
      : undefined
  );
}
```

#### 3、createLegacyRoot

```js
// src/react/packages/shared/ReactRootTags.js

// ⭐️ Line 12~14 ⭐️
export const LegacyRoot = 0; // ReactDOM.render
export const BlockingRoot = 1; // ReactDOM.createBlockingRoot
export const ConcurrentRoot = 2; // ReactDOM.createRoot
```

```js
// src/react/packages/react-dom/src/client/ReactDOMRoot.js

// createLegacyRoot 方法
// ⭐️ Line 155~160 ⭐️
/**
 * 通过实例化 ReactDOMBlockingRoot 类创建 LegacyRoot
 */
export function createLegacyRoot(
  container: Container,
  options?: RootOptions
): RootType {
  // container => <div id="root"></div>
  // LegacyRoot 常量，值为 0
  // 通过 render 犯法创建的 container 就是 LegacyRoot
  return new ReactDOMBlockingRoot(container, LegacyRoot, options);
}
```

#### 4、ReactDOMBlockingRoot

```js
// src/react/packages/react-dom/src/client/ReactDOMRoot.js
// ⭐️ Line 56~62 ⭐️
/**
 * 创建 ReactDOMBlockingRoot 的类
 * 通过它可以创建 LegacyRoot 的 Fiber 数据结构
 */
function ReactDOMBlockingRoot(
  container: Container,
  tag: RootTag,
  options: void | RootOptions
) {
  // tag => 0 => legacyRoot
  // container => <div id="root"></div>
  // container._reactRootContainer = {_internalRoot: {}}
  this._internalRoot = createRootImpl(container, tag, options);
}
```

#### 5、createRootImpl

```js
// src/react/packages/react-dom/src/client/ReactDOMRoot.js

// ⭐️ Line 110~129 ⭐️
function createRootImpl(
  container: Container,
  tag: RootTag,
  options: void | RootOptions
) {
  // container => <div id="root"></div>
  // tag => 0
  // options => undefined
  // 检测是否是服务器端渲染 —— false
  const hydrate = options != null && options.hydrate === true;
  // 服务器端渲染相关 —— null
  const hydrationCallbacks =
    (options != null && options.hydrationOptions) || null;
  const root = createContainer(container, tag, hydrate, hydrationCallbacks);
  markContainerAsRoot(root.current, container);
  // 服务器端渲染相关
  if (hydrate && tag !== LegacyRoot) {
    const doc =
      container.nodeType === DOCUMENT_NODE
        ? container
        : container.ownerDocument;
    eagerlyTrapReplayableEvents(container, doc);
  }
  return root;
}
```

#### 6、createContainer

```js
// src/react/packages/react-reconciler/src/ReactFiberReconciler.js

// ⭐️ Line 219~226 ⭐️
export function createContainer(
  containerInfo: Container,
  tag: RootTag,
  hydrate: boolean,
  hydrationCallbacks: null | SuspenseHydrationCallbacks
): OpaqueRoot {
  // containerInfo => <div id="root"></div>
  // tag: 0
  // hydrate: false
  // hydrationCallbacks: null
  return createFiberRoot(containerInfo, tag, hydrate, hydrationCallbacks);
}
```

#### 7、createFiberRoot

```js
// src/react/packages/react-reconciler/src/ReactFiberRoot.js

// ⭐️ Line 137~157 ⭐️
// 创建根节点对应的 Fiber 对象
export function createFiberRoot(
  containerInfo: any,
  tag: RootTag,
  hydrate: boolean,
  hydrationCallbacks: null | SuspenseHydrationCallbacks
): FiberRoot {
  // 创建 fiberRoot
  const root: FiberRoot = (new FiberRootNode(containerInfo, tag, hydrate): any);
  // false
  if (enableSuspenseCallback) {
    root.hydrationCallbacks = hydrationCallbacks;
  }

  // 创建根节点对应的 rootFiber
  const uninitializedFiber = createHostRootFiber(tag);
  // 为 fiberRoot 添加 current 属性，值为 rootFiber
  root.current = uninitializedFiber;
  // 为 rootFiber 添加 stateNode 属性，值为 fiberRoot
  uninitializedFiber.stateNode = root;

  // 为 fiber 对象添加 updateQueue 属性，初始化 updateQueue 对象
  // updateQueue 用于存放 Update 对象
  // Update 对象用于记录组件状态的改变
  initializeUpdateQueue(uninitializedFiber);
  // 返回 root
  return root;
}
```

#### 8、FiberRootNode

```js
// src/react/packages/react-reconciler/src/ReactFiberRoot.js

// ⭐️ Line 106~135 ⭐️
// FiberRootNode 的构造函数
function FiberRootNode(containerInfo, tag, hydrate) {
  this.tag = tag;
  this.current = null;
  this.containerInfo = containerInfo;
  this.pendingChildren = null;
  this.pingCache = null;
  this.finishedExpirationTime = NoWork;
  this.finishedWork = null;
  this.timeoutHandle = noTimeout;
  this.context = null;
  this.pendingContext = null;
  this.hydrate = hydrate;
  this.callbackNode = null;
  this.callbackPriority = NoPriority;
  this.firstPendingTime = NoWork;
  this.firstSuspendedTime = NoWork;
  this.lastSuspendedTime = NoWork;
  this.nextKnownPendingLevel = NoWork;
  this.lastPingedTime = NoWork;
  this.lastExpiredTime = NoWork;

  if (enableSchedulerTracing) {
    this.interactionThreadID = unstable_getThreadID();
    this.memoizedInteractions = new Set();
    this.pendingInteractionMap = new Map();
  }
  if (enableSuspenseCallback) {
    this.hydrationCallbacks = null;
  }
}
```

#### 9、initializeUpdateQueue

```js
// src/react/packages/react-reconciler/src/ReactUpdateQueue.js

// ⭐️ Line 154~164 ⭐️
export function initializeUpdateQueue<State>(fiber: Fiber): void {
  const queue: UpdateQueue<State> = {
    // 上一次更新之后的 state，作为下一次更新的基础
    baseState: fiber.memoizedState,
    baseQueue: null,
    shared: {
      pending: null,
    },
    effects: null,
  };
  fiber.updateQueue = queue;
}
```

### 四）获取 rootFiber.child 的实例对象

#### 1、 legacyRenderSubtreeIntoContainer

```js
// src/react/packages/react-dom/src/client/ReactDOMLegacy.js

// legacyRenderSubtreeIntoContainer 方法
// ⭐️ Line 175~222 ⭐️
/**
 * 将子树渲染到容器中（初始化 Fiber 数据结构：创建 fiberRoot 及 rootFiber）
 */
function legacyRenderSubtreeIntoContainer(
  parentComponent: ?React$Component<any, any>,
  children: ReactNodeList,
  container: Container,
  forceHydrate: boolean,
  callback: ?Function
) {
  if (__DEV__) {
    topLevelUpdateWarnings(container);
    warnOnInvalidCallback(callback === undefined ? null : callback, "render");
  }

  let root: RootType = (container._reactRootContainer: any);
  // 即将存储根 Fiber 对象
  let fiberRoot;
  if (!root) {
    // 初始渲染
    // 初始化根 Fiber 数据结构
    // 为 container 容器添加 _reactRootContainer 属性
    // 在 _reactRootContainer 对象中有一个属性叫做 _internalRoot
    // _internalRoot 属性值即为 fiberRoot ，表示根节点 Fiber 数据结构
    // legacyCreateRootFromDOMContainer 方法创建了 fiberRoot 和 rootFiber，方法中调用了其他方法：
    // createLegacyRoot 方法中也调用了其他方法：
    // new ReactDOMBlockingRoot -> this._internalRoot 方法中也调用了其他方法：
    // createRootImpl
    root = container._reactRootContainer = legacyCreateRootFromDOMContainer(
      container,
      forceHydrate
    );
    // 获取 fiberRoot 对象
    fiberRoot = root._internalRoot;
    /**
     * 改变 callback 函数中的 this 指向
     * 使其指向 render 方法第一个参数的真实 DOM 对象
     */
    // 如果 callback 参数是函数类型
    if (typeof callback === "function") {
      // 使用 originalCallback 存储 callback 函数
      const originalCallback = callback;
      // 为 callback 函数重新赋值
      callback = function () {
        // 获取 render 方法第一个参数的真实 DOM 对象
        // 实际上就是 id="root" 的 div 子元素
        // rootFiber.child.stateNode
        // rootFiber 就是 id="root" 的 div
        const instance = getPublicRootInstance(fiberRoot);
        // 调用原始 callback 函数并改变函数内部 this 指向
        originalCallback.call(instance);
      };
    }
    // 初始化渲染不执行批量化更新
    // 因为批量更新是异步的，是可以被打断的，但是初始化渲染应该是尽快完成不能被打断
    // 所以不执行批量更新
    unbatchedUpdates(() => {
      updateContainer(children, fiberRoot, parentComponent, callback);
    });
  } else {
    fiberRoot = root._internalRoot;
    if (typeof callback === "function") {
      const originalCallback = callback;
      callback = function () {
        const instance = getPublicRootInstance(fiberRoot);
        originalCallback.call(instance);
      };
    }
    // Update
    updateContainer(children, fiberRoot, parentComponent, callback);
  }
  return getPublicRootInstance(fiberRoot);
}
```

#### 2、getPublicRootInstance

```js
// src/react/packages/react-reconciler/src/ReactFiberReconciler.js

// ⭐️ Line 316~329 ⭐️
// getPublicRootInstance 方法
/**
 * 获取 container 的第一个子元素的实例对象
 */
export function getPublicRootInstance(
  container: OpaqueRoot
): React$Component<any, any> | PublicInstance | null {
  // 获取 rootFiber
  const containerFiber = container.current;
  // 如果 rootFiber 没有子元素
  // 指的是 id="root" 的 div 没有子元素
  if (!containerFiber.child) {
    // 返回 null
    return null;
  }
  // 匹配子元素的类型
  switch (containerFiber.child.tag) {
    // 普通
    case HostComponent:
      return getPublicInstance(containerFiber.child.stateNode);
    default:
      // 返回子元素的真实 DOM 元素
      return containerFiber.child.stateNode;
  }
}
```

#### 3、getPublicInstance

```js
// src/react/packages/react-dom/src/client/ReactDOMHostConfig.js

// ⭐️ Line 208~210 ⭐️
// getPublicInstance 方法
export function getPublicInstance(instance: Instance): * {
  return instance;
}
```

### 五）updateContainer

#### 1、updateContainer

```js
// src/react/packages/react-reconciler/src/ReactFiberReconciler.js

// ⭐️ Line 228~300 ⭐️
// updateContainer 方法
/**
 * 计算任务的过期时间
 * 再根据任务过期时间创建 Update 任务
 * 通过任务的过期时间还可以计算出任务的优先级
 */
export function updateContainer(
  // element：要渲染的 ReactElement
  element: ReactNodeList,
  // container：fiberRoot 对象
  container: OpaqueRoot,
  // parentComponent：父组件，初始渲染为 null
  parentComponent: ?React$Component<any, any>,
  // ReactElement 渲染完成执行的回调函数
  callback: ?Function
): ExpirationTime {
  if (__DEV__) {
    onScheduleRoot(container, element);
  }
  // container 获取 rootFiber
  const current = container.current;
  // 获取当前距离 react 应用初始化的时间 1073741805
  const currentTime = requestCurrentTimeForUpdate();
  if (__DEV__) {
    // $FlowExpectedError - jest isn't a global, and isn't recognized outside of tests
    if ("undefined" !== typeof jest) {
      warnIfUnmockedScheduler(current);
      warnIfNotScopedWithMatchingAct(current);
    }
  }
  // 异步加载设置
  const suspenseConfig = requestCurrentSuspenseConfig();

  // 计算任务过期时间
  // 为防止任务因为优先级的原因一直被打断而未能执行
  // react 会设置一个过期时间，当时间到了过期时间的时候
  // 如果任务还未执行的话，react 将会强制执行该任务
  // 初始化渲染时，任务同步执行不涉及被打断的问题
  // 过期时间被设置成了 1073741823 ，这个数值表示当前任务为同步任务
  const expirationTime = computeExpirationForFiber(
    currentTime,
    current,
    suspenseConfig
  );
  // 设置 fiberRoot.context ，首次执行返回一个 emptyContext ，是一个 {} （因为初始化渲染时 parentComponent 为 null）
  const context = getContextForSubtree(parentComponent);
  // 初始化渲染时 fiberRoot 对象中的 context 属性值为 null
  // 所以会进入到 if 中
  if (container.context === null) {
    // 初始化渲染时将 context 属性值设置为 {}
    container.context = context;
  } else {
    container.pendingContext = context;
  }

  if (__DEV__) {
    if (
      ReactCurrentFiberIsRendering &&
      ReactCurrentFiberCurrent !== null &&
      !didWarnAboutNestedUpdates
    ) {
      didWarnAboutNestedUpdates = true;
      console.error(
        "Render methods should be a pure function of props and state; " +
          "triggering nested component updates from render is not allowed. " +
          "If necessary, trigger nested updates in componentDidUpdate.\n\n" +
          "Check the render method of %s.",
        getComponentName(ReactCurrentFiberCurrent.type) || "Unknown"
      );
    }
  }

  /** 核心代码 */
  // 创建一个待执行任务
  const update = createUpdate(expirationTime, suspenseConfig);
  // 将要更新的内容挂载到更新对象中的 payload 中
  // 将要更新的组件存储在 payload 对象中，方便后期获取
  update.payload = { element };
  // 判断 callback 是否存在
  callback = callback === undefined ? null : callback;
  // 如果 callback 存在
  if (callback !== null) {
    if (__DEV__) {
      if (typeof callback !== "function") {
        console.error(
          "render(...): Expected the last optional `callback` argument to be a " +
            "function. Instead received: %s.",
          callback
        );
      }
    }
    // 将 callback 挂载到 update 对象中
    // 其实就是一层层传递，方便 ReactElement 元素渲染完成调用
    // 回调函数执行完成后会被清除，可以在代码的后面加上 return 进行验证
    update.callback = callback;
  }

  // 将 update 对象加入到当前 Fiber 的更新队列当中（updateQueue）
  // 待执行的任务都会被存储在 fiber.updateQueue.shared.pending 中
  enqueueUpdate(current, update);
  // 调度和更新 current 对象
  scheduleWork(current, expirationTime);

  // 返回过期时间
  return expirationTime;
}
```

#### 2、getContextForSubtree

```js
// src/react/packages/react-reconciler/src/ReactFiberReconciler.js

// ⭐️ Line 122~140 ⭐️
// getContextForSubtree 方法

function getContextForSubtree(
  parentComponent: ?React$Component<any, any>
): Object {
  if (!parentComponent) {
    return emptyContextObject;
  }

  const fiber = getInstance(parentComponent);
  const parentContext = findCurrentUnmaskedContext(fiber);

  if (fiber.tag === ClassComponent) {
    const Component = fiber.type;
    if (isLegacyContextProvider(Component)) {
      return processChildContext(fiber, Component, parentContext);
    }
  }

  return parentContext;
}
```

### 六）enqueueUpdate

```js
// src/react/packages/react-reconciler/src/ReactUpdateQueue.js

// ⭐️ Line 205~237 ⭐️
// enqueueUpdate 方法
// 将任务（Update）存放于任务队列（updateQueue）中
// 创建单向链表结构存放 update，next 用来串联 update
export function enqueueUpdate<State>(fiber: Fiber, update: Update<State>) {
  // 获取当前 Fiber 的更新队列
  const updateQueue = fiber.updateQueue;
  // 如果更新队列不存在就返回 null
  if (updateQueue === null) {
    // 这种情况仅发生在 fiber 已经被卸载的时候
    return;
  }
  // 获取待执行的 Update 任务
  // 初始渲染时没有待执行的任务
  const sharedQueue = updateQueue.shared;
  const pending = sharedQueue.pending;
  // 如果没有待执行的 Update 任务
  if (pending === null) {
    // This is the first update. Create a circular list.
    update.next = update;
  } else {
    update.next = pending.next;
    pending.next = update;
  }
  // 将 Update 任务存储在 pending 属性中
  sharedQueue.pending = update;

  if (__DEV__) {
    if (
      currentlyProcessingQueue === sharedQueue &&
      !didWarnUpdateInsideUpdate
    ) {
      console.error(
        "An update (setState, replaceState, or forceUpdate) was scheduled " +
          "from inside an update function. Update functions should be pure, " +
          "with zero side-effects. Consider using componentDidUpdate or a " +
          "callback."
      );
      didWarnUpdateInsideUpdate = true;
    }
  }
}
```

### 七）scheduleUpdateOnFiber

#### 1、updateContainer

```js
// src/react/packages/react-reconciler/src/ReactFiberReconciler.js

// ⭐️ Line 228~300 ⭐️
// updateContainer 方法
/**
 * 计算任务的过期时间
 * 再根据任务过期时间创建 Update 任务
 * 通过任务的过期时间还可以计算出任务的优先级
 */
export function updateContainer(
  // element：要渲染的 ReactElement
  element: ReactNodeList,
  // container：fiberRoot 对象
  container: OpaqueRoot,
  // parentComponent：父组件，初始渲染为 null
  parentComponent: ?React$Component<any, any>,
  // ReactElement 渲染完成执行的回调函数
  callback: ?Function
): ExpirationTime {
  if (__DEV__) {
    onScheduleRoot(container, element);
  }
  // container 获取 rootFiber
  const current = container.current;
  // 获取当前距离 react 应用初始化的时间 1073741805
  const currentTime = requestCurrentTimeForUpdate();
  if (__DEV__) {
    // $FlowExpectedError - jest isn't a global, and isn't recognized outside of tests
    if ("undefined" !== typeof jest) {
      warnIfUnmockedScheduler(current);
      warnIfNotScopedWithMatchingAct(current);
    }
  }
  // 异步加载设置
  const suspenseConfig = requestCurrentSuspenseConfig();

  // 计算任务过期时间
  // 为防止任务因为优先级的原因一直被打断而未能执行
  // react 会设置一个过期时间，当时间到了过期时间的时候
  // 如果任务还未执行的话，react 将会强制执行该任务
  // 初始化渲染时，任务同步执行不涉及被打断的问题
  // 过期时间被设置成了 1073741823 ，这个数值表示当前任务为同步任务
  const expirationTime = computeExpirationForFiber(
    currentTime,
    current,
    suspenseConfig
  );
  // 设置 fiberRoot.context ，首次执行返回一个 emptyContext ，是一个 {} （因为初始化渲染时 parentComponent 为 null）
  const context = getContextForSubtree(parentComponent);
  // 初始化渲染时 fiberRoot 对象中的 context 属性值为 null
  // 所以会进入到 if 中
  if (container.context === null) {
    // 初始化渲染时将 context 属性值设置为 {}
    container.context = context;
  } else {
    container.pendingContext = context;
  }

  if (__DEV__) {
    if (
      ReactCurrentFiberIsRendering &&
      ReactCurrentFiberCurrent !== null &&
      !didWarnAboutNestedUpdates
    ) {
      didWarnAboutNestedUpdates = true;
      console.error(
        "Render methods should be a pure function of props and state; " +
          "triggering nested component updates from render is not allowed. " +
          "If necessary, trigger nested updates in componentDidUpdate.\n\n" +
          "Check the render method of %s.",
        getComponentName(ReactCurrentFiberCurrent.type) || "Unknown"
      );
    }
  }

  /** 核心代码 */
  // 创建一个待执行任务
  const update = createUpdate(expirationTime, suspenseConfig);
  // 将要更新的内容挂载到更新对象中的 payload 中
  // 将要更新的组件存储在 payload 对象中，方便后期获取
  update.payload = { element };
  // 判断 callback 是否存在
  callback = callback === undefined ? null : callback;
  // 如果 callback 存在
  if (callback !== null) {
    if (__DEV__) {
      if (typeof callback !== "function") {
        console.error(
          "render(...): Expected the last optional `callback` argument to be a " +
            "function. Instead received: %s.",
          callback
        );
      }
    }
    // 将 callback 挂载到 update 对象中
    // 其实就是一层层传递，方便 ReactElement 元素渲染完成调用
    // 回调函数执行完成后会被清除，可以在代码的后面加上 return 进行验证
    update.callback = callback;
  }

  // 将 update 对象加入到当前 Fiber 的更新队列当中（updateQueue）
  // 待执行的任务都会被存储在 fiber.updateQueue.shared.pending 中
  enqueueUpdate(current, update);
  // 调度和更新 current 对象
  scheduleWork(current, expirationTime);

  // 返回过期时间
  return expirationTime;
}
```

#### 2、scheduleUpdateOnFiber

```js
// src/react/packages/react-reconciler/src/ReactFiberWorkLoop.js

// ⭐️ Line 379~449 ⭐️
// scheduleUpdateOnFiber 方法
/**
 * 判断任务是否为同步，如果是，则调用同步任务入口
 *
 * fiber：初始化渲染时为 rootFiber ，即 <div id="root"></div> 对应的 Fiber 对象
 * expirationTime：任务过期时间，初始化渲染时任务过期时间值为 1073741823
 */
export function scheduleUpdateOnFiber(
  fiber: Fiber,
  expirationTime: ExpirationTime
) {
  /**
   * 判断是否是无限循环的 update ，如果是就报错。nested => 嵌套的
   * 无限循环的 update 在什么时候发生呢？
   * 在 componentWillUpdate 或者 componentDidUpdate 生命周期函数中重复调用 setState 方法时，可能会发生这种情况。
   * React 限制了嵌套更新的数量以防止无限制的嵌套。
   * 限制的嵌套更新数量为 50 ，可通过 NESTED_UPDATE_LIMIT 全局变量获取。
   */
  checkForNestedUpdates();
  // 开发环境下执行的代码，忽略
  warnAboutRenderPhaseUpdatesInDEV(fiber);
  // 遍历更新子节点的过期时间，返回 FiberRoot
  const root = markUpdateTimeFromFiberToRoot(fiber, expirationTime);
  if (root === null) {
    // 开发环境下执行，忽略
    warnAboutUpdateOnUnmountedFiberInDEV(fiber);
    return;
  }
  // 判断是否有高优先级任务打断当前正在执行的任务
  // 内部判断条件不成立，内部代码没有得到执行
  checkForInterruption(fiber, expirationTime);
  // 报告调度更新，测试环境执行，忽略
  recordScheduleUpdate();

  // 获取当前调度任务的优先级，数值类型，从 90 开始，数值越大，优先级越高。
  // 97 — 普通优先级任务
  const priorityLevel = getCurrentPriorityLevel();
  // 判断任务是否是同步任务，Sync的值为：1073741823
  if (expirationTime === Sync) {
    if (
      // 检测是否处于非批量更新模式
      (executionContext & LegacyUnbatchedContext) !== NoContext &&
      // 检测是否没有处于正在进行渲染的任务
      (executionContext & (RenderContext | CommitContext)) === NoContext
    ) {
      // 在根上注册待处理的交互，以避免丢失跟踪的交互数据
      // 初始渲染时内部条件判断不成立，内部代码没有得到执行
      schedulePendingInteractions(root, expirationTime);

      // 同步任务入口点
      performSyncWorkOnRoot(root);
    } else {
      ensureRootIsScheduled(root);
      schedulePendingInteractions(root, expirationTime);
      if (executionContext === NoContext) {
        // Flush the synchronous work now, unless we're already working or inside
        // a batch. This is intentionally inside scheduleUpdateOnFiber instead of
        // scheduleCallbackForFiber to preserve the ability to schedule a callback
        // without immediately flushing it. We only do this for user-initiated
        // updates, to preserve historical behavior of legacy mode.
        flushSyncCallbackQueue();
      }
    }
  } else {
    ensureRootIsScheduled(root);
    schedulePendingInteractions(root, expirationTime);
  }

  // 初始渲染不执行
  if (
    (executionContext & DiscreteEventContext) !== NoContext &&
    // Only updates at user-blocking priority or greater are considered
    // discrete, even inside a discrete event.
    (priorityLevel === UserBlockingPriority ||
      priorityLevel === ImmediatePriority)
  ) {
    // This is the result of a discrete event. Track the lowest priority
    // discrete update per root so we can flush them early, if needed.
    if (rootsWithPendingDiscreteUpdates === null) {
      rootsWithPendingDiscreteUpdates = new Map([[root, expirationTime]]);
    } else {
      const lastDiscreteTime = rootsWithPendingDiscreteUpdates.get(root);
      if (lastDiscreteTime === undefined || lastDiscreteTime > expirationTime) {
        rootsWithPendingDiscreteUpdates.set(root, expirationTime);
      }
    }
  }
}
```

#### 3、getCurrentPriorityLevel

```js
// src/react/packages/react-reconciler/src/SchedulerWithReactIntegration.js
// ⭐️ Line 84~99 ⭐️
// getCurrentPriorityLevel 方法
// 获取当前调度任务的优先级
export function getCurrentPriorityLevel(): ReactPriorityLevel {
  switch (Scheduler_getCurrentPriorityLevel()) {
    // 99，立即执行的任务
    case Scheduler_ImmediatePriority:
      return ImmediatePriority;
    // 98，用户交互任务
    case Scheduler_UserBlockingPriority:
      return UserBlockingPriority;
    // 97，普通优先级
    case Scheduler_NormalPriority:
      return NormalPriority;
    // 96，低优先级任务
    case Scheduler_LowPriority:
      return LowPriority;
    // 95，闲时任务
    case Scheduler_IdlePriority:
      return IdlePriority;
    // 缺少优先级
    default:
      invariant(false, "Unknown priority level.");
  }
}
```

### 八）构建 Fiber 对象

#### 1、performSyncWorkOnRoot

```js
// src/react/packages/react-reconciler/src/ReactFiberWorkLoop.js

// ⭐️ Line 988~1063 ⭐️
// performSyncWorkOnRoot 方法
/**
 * 调用了此方法，则标志着我们正式进入 render 阶段了，这个方法用于构建 workInProgress Fiber 树
 * root：FiberRoot 对象
 */
function performSyncWorkOnRoot(root) {
  // 检查是否有过期的任务
  // 如果没有过期的任务，值为 0
  // 初始化渲染没有过期的任务待执行
  const lastExpiredTime = root.lastExpiredTime;
  // NoWork 值为 0
  // 如果有过期的任务，将过期时间设置为 lastExpiredTime ，否则将过期时间设置为 Sync
  // 初始渲染过期时间被设置成了 Sync
  const expirationTime = lastExpiredTime !== NoWork ? lastExpiredTime : Sync;
  invariant(
    (executionContext & (RenderContext | CommitContext)) === NoContext,
    "Should not already be working."
  );

  // 处理 useEffect 这个钩子函数相关的内容
  flushPassiveEffects();

  // 如果 root 和 workInProgressRoot 不相等，说明 workInProgressRoot 不存在，说明还没有建构 workInProgress Fiber 树
  // workInProgressRoot 为全局变量，默认值为 null ，初始渲染时值为 null
  // expirationTime => 1073741823
  // renderExpirationTime => 0
  // true
  if (root !== workInProgressRoot || expirationTime !== renderExpirationTime) {
    // 构建 workInProgress Fiber 树及 rootFiber
    prepareFreshStack(root, expirationTime);
    // 初始渲染不执行，内部条件判断不成立
    startWorkOnPendingInteractions(root, expirationTime);
  }

  // If we have a work-in-progress fiber, it means there's still work to do
  // in this root.
  if (workInProgress !== null) {
    const prevExecutionContext = executionContext;
    executionContext |= RenderContext;
    const prevDispatcher = pushDispatcher(root);
    const prevInteractions = pushInteractions(root);
    startWorkLoopTimer(workInProgress);

    do {
      try {
        workLoopSync();
        break;
      } catch (thrownValue) {
        handleError(root, thrownValue);
      }
    } while (true);
    resetContextDependencies();
    executionContext = prevExecutionContext;
    popDispatcher(prevDispatcher);
    if (enableSchedulerTracing) {
      popInteractions(((prevInteractions: any): Set<Interaction>));
    }

    if (workInProgressRootExitStatus === RootFatalErrored) {
      const fatalError = workInProgressRootFatalError;
      stopInterruptedWorkLoopTimer();
      prepareFreshStack(root, expirationTime);
      markRootSuspendedAtTime(root, expirationTime);
      ensureRootIsScheduled(root);
      throw fatalError;
    }

    if (workInProgress !== null) {
      // This is a sync render, so we should have finished the whole tree.
      invariant(
        false,
        "Cannot commit an incomplete root. This error is likely caused by a " +
          "bug in React. Please file an issue."
      );
    } else {
      // We now have a consistent tree. Because this is a sync render, we
      // will commit it even if something suspended.
      stopFinishedWorkLoopTimer();
      root.finishedWork = (root.current.alternate: any);
      root.finishedExpirationTime = expirationTime;
      // 结束 render 阶段
      // 进入 commit 阶段
      finishSyncRender(root);
    }

    // Before exiting, make sure there's a callback scheduled for the next
    // pending level.
    ensureRootIsScheduled(root);
  }

  return null;
}
```

#### 2、prepareFreshStack

```js
// src/react/packages/react-reconciler/src/ReactFiberWorkLoop.js
// ⭐️ Line 1235~1273 ⭐️
// prepareFreshStack 方法
// 构建 workInProgressFiber 树及 rootFiber
function prepareFreshStack(root, expirationTime) {
  // 为 FiberRoot 对象添加 finishedWork 属性
  // finishedWork 表示 render 阶段执行完成后构建的待提交的 FIber 对象
  root.finishedWork = null;
  // 初始化 finishedExpirationTime 值为 0
  root.finishedExpirationTime = NoWork;

  const timeoutHandle = root.timeoutHandle;
  // 初始化渲染不执行 timeoutHandle => -1 noTimeout => -1
  if (timeoutHandle !== noTimeout) {
    // The root previous suspended and scheduled a timeout to commit a fallback
    // state. Now that we have additional work, cancel the timeout.
    root.timeoutHandle = noTimeout;
    // $FlowFixMe Complains noTimeout is not a TimeoutID, despite the check above
    cancelTimeout(timeoutHandle);
  }

  // 初始化渲染不执行 workInProgress 全局变量，初始化为 null
  // false
  if (workInProgress !== null) {
    let interruptedWork = workInProgress.return;
    while (interruptedWork !== null) {
      unwindInterruptedWork(interruptedWork);
      interruptedWork = interruptedWork.return;
    }
  }
  // 建构 workInProgress Fiber 树的 fiberRoot 对象
  workInProgressRoot = root;
  // 建构 workInProgress Fiber 树中的 rootFiber
  workInProgress = createWorkInProgress(root.current, null);
  renderExpirationTime = expirationTime;
  workInProgressRootExitStatus = RootIncomplete;
  workInProgressRootFatalError = null;
  workInProgressRootLatestProcessedExpirationTime = Sync;
  workInProgressRootLatestSuspenseTimeout = Sync;
  workInProgressRootCanSuspendUsingConfig = null;
  workInProgressRootNextUnprocessedUpdateTime = NoWork;
  workInProgressRootHasPendingPing = false;
  // true
  if (enableSchedulerTracing) {
    spawnedWorkDuringRender = null;
  }

  if (__DEV__) {
    ReactStrictModeWarnings.discardPendingWarnings();
  }
}
```

#### 3、createWorkInProgress

```js
// src/react/packages/react-reconciler/src/ReactFiber.js
// ⭐️ Line 403~506 ⭐️
// createWorkInProgress 方法
/**
 * 构建 workInProgress Fiber 树中的 rootFiber
 * 构建完成后会替换 current fiber
 * current：current Fiber 中的 rootFiber
 * 初始渲染 pendingProps 为 null
 */
export function createWorkInProgress(current: Fiber, pendingProps: any): Fiber {
  // 获取 current Fiber 对应的 workInProgress Fiber
  let workInProgress = current.alternate;
  // 如果 workInProgress 不存在
  if (workInProgress === null) {
    // 创建 fiber 对象
    workInProgress = createFiber(
      current.tag,
      pendingProps,
      current.key,
      current.mode
    );
    // 属性复用
    workInProgress.elementType = current.elementType;
    workInProgress.type = current.type;
    workInProgress.stateNode = current.stateNode;

    if (__DEV__) {
      // DEV-only fields
      if (enableUserTimingAPI) {
        workInProgress._debugID = current._debugID;
      }
      workInProgress._debugSource = current._debugSource;
      workInProgress._debugOwner = current._debugOwner;
      workInProgress._debugHookTypes = current._debugHookTypes;
    }
    // 使用 alternate 存储 current
    workInProgress.alternate = current;
    // 使用 alternate 存储 workInProgress
    current.alternate = workInProgress;
  } else {
    workInProgress.pendingProps = pendingProps;

    // We already have an alternate.
    // Reset the effect tag.
    workInProgress.effectTag = NoEffect;

    // The effect list is no longer valid.
    workInProgress.nextEffect = null;
    workInProgress.firstEffect = null;
    workInProgress.lastEffect = null;

    if (enableProfilerTimer) {
      // We intentionally reset, rather than copy, actualDuration & actualStartTime.
      // This prevents time from endlessly accumulating in new commits.
      // This has the downside of resetting values for different priority renders,
      // But works for yielding (the common case) and should support resuming.
      workInProgress.actualDuration = 0;
      workInProgress.actualStartTime = -1;
    }
  }

  workInProgress.childExpirationTime = current.childExpirationTime;
  workInProgress.expirationTime = current.expirationTime;

  workInProgress.child = current.child;
  workInProgress.memoizedProps = current.memoizedProps;
  workInProgress.memoizedState = current.memoizedState;
  workInProgress.updateQueue = current.updateQueue;

  // Clone the dependencies object. This is mutated during the render phase, so
  // it cannot be shared with the current fiber.
  const currentDependencies = current.dependencies;
  workInProgress.dependencies =
    currentDependencies === null
      ? null
      : {
          expirationTime: currentDependencies.expirationTime,
          firstContext: currentDependencies.firstContext,
          responders: currentDependencies.responders,
        };

  // These will be overridden during the parent's reconciliation
  workInProgress.sibling = current.sibling;
  workInProgress.index = current.index;
  workInProgress.ref = current.ref;

  if (enableProfilerTimer) {
    workInProgress.selfBaseDuration = current.selfBaseDuration;
    workInProgress.treeBaseDuration = current.treeBaseDuration;
  }

  if (__DEV__) {
    workInProgress._debugNeedsRemount = current._debugNeedsRemount;
    switch (workInProgress.tag) {
      case IndeterminateComponent:
      case FunctionComponent:
      case SimpleMemoComponent:
        workInProgress.type = resolveFunctionForHotReloading(current.type);
        break;
      case ClassComponent:
        workInProgress.type = resolveClassForHotReloading(current.type);
        break;
      case ForwardRef:
        workInProgress.type = resolveForwardRefForHotReloading(current.type);
        break;
      default:
        break;
    }
  }

  return workInProgress;
}
```

#### 4、workLoopSync

```js
// src/react/packages/react-reconciler/src/ReactFiberWorkLoop.js
// ⭐️ Line 1457~1464 ⭐️
// workLoopSync 方法
// 以同步的方式构建 workInprogress Fiber 对象
/** @noinline */
function workLoopSync() {
  // workInProgress 是一个 fiber 对象
  // 它的值不为 null 意味着该 fiber 对象上仍然有更新要执行
  // while 方法支撑 render 阶段所有 fiber 节点的构建
  while (workInProgress !== null) {
    workInProgress = performUnitOfWork(workInProgress);
  }
}
```

#### 5、performUnitOfWork

```js
// src/react/packages/react-reconciler/src/ReactFiberWorkLoop.js
// ⭐️ Line 1474~1501 ⭐️
// performUnitOfWork 方法
// 构建 Fiber 对象
function performUnitOfWork(unitOfWork: Fiber): Fiber | null {
  // The current, flushed, state of this fiber is the alternate. Ideally
  // nothing should rely on this, but relying on it here means that we don't
  // need an additional field on the work in progress.
  const current = unitOfWork.alternate;

  startWorkTimer(unitOfWork);
  setCurrentDebugFiberInDEV(unitOfWork);

  let next;
  if (enableProfilerTimer && (unitOfWork.mode & ProfileMode) !== NoMode) {
    startProfilerTimer(unitOfWork);
    next = beginWork(current, unitOfWork, renderExpirationTime);
    stopProfilerTimerIfRunningAndRecordDelta(unitOfWork, true);
  } else {
    next = beginWork(current, unitOfWork, renderExpirationTime);
  }

  resetCurrentDebugFiberInDEV();
  unitOfWork.memoizedProps = unitOfWork.pendingProps;
  if (next === null) {
    // If this doesn't spawn new work, complete the current work.
    next = completeUnitOfWork(unitOfWork);
  }

  ReactCurrentOwner.current = null;
  return next;
}
```

#### 6、beginWork

```js
// src/react/packages/react-reconciler/src/ReactFiberBeginWork.js
// ⭐️ Line 2874~3301 ⭐️
//beginWork 方法
// 从父到子，构建 Fiber 节点对象
function beginWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderExpirationTime: ExpirationTime
): Fiber | null {
  const updateExpirationTime = workInProgress.expirationTime;

  if (__DEV__) {
    if (workInProgress._debugNeedsRemount && current !== null) {
      // This will restart the begin phase with a new fiber.
      return remountFiber(
        current,
        workInProgress,
        createFiberFromTypeAndProps(
          workInProgress.type,
          workInProgress.key,
          workInProgress.pendingProps,
          workInProgress._debugOwner || null,
          workInProgress.mode,
          workInProgress.expirationTime
        )
      );
    }
  }
  // 判断是否有旧的 Fiber 对象
  // 初始化渲染时，只有 rootFiber 节点存在 current
  if (current !== null) {
    // 获取旧的 props 对象
    const oldProps = current.memoizedProps;
    // 获取新的 props 对象
    const newProps = workInProgress.pendingProps;
    // 初始渲染时，false
    if (
      oldProps !== newProps ||
      hasLegacyContextChanged() ||
      // Force a re-render if the implementation changed due to hot reload:
      (__DEV__ ? workInProgress.type !== current.type : false)
    ) {
      // If props or context changed, mark the fiber as having performed work.
      // This may be unset if the props are determined to be equal later (memo).
      didReceiveUpdate = true;
    } else if (updateExpirationTime < renderExpirationTime) {
      didReceiveUpdate = false;
      // This fiber does not have any pending work. Bailout without entering
      // the begin phase. There's still some bookkeeping we that needs to be done
      // in this optimized path, mostly pushing stuff onto the stack.
      switch (workInProgress.tag) {
        case HostRoot:
          pushHostRootContext(workInProgress);
          resetHydrationState();
          break;
        case HostComponent:
          pushHostContext(workInProgress);
          if (
            workInProgress.mode & ConcurrentMode &&
            renderExpirationTime !== Never &&
            shouldDeprioritizeSubtree(workInProgress.type, newProps)
          ) {
            if (enableSchedulerTracing) {
              markSpawnedWork(Never);
            }
            // Schedule this fiber to re-render at offscreen priority. Then bailout.
            workInProgress.expirationTime = workInProgress.childExpirationTime =
              Never;
            return null;
          }
          break;
        case ClassComponent: {
          const Component = workInProgress.type;
          if (isLegacyContextProvider(Component)) {
            pushLegacyContextProvider(workInProgress);
          }
          break;
        }
        case HostPortal:
          pushHostContainer(
            workInProgress,
            workInProgress.stateNode.containerInfo
          );
          break;
        case ContextProvider: {
          const newValue = workInProgress.memoizedProps.value;
          pushProvider(workInProgress, newValue);
          break;
        }
        case Profiler:
          if (enableProfilerTimer) {
            // Profiler should only call onRender when one of its descendants actually rendered.
            const hasChildWork =
              workInProgress.childExpirationTime >= renderExpirationTime;
            if (hasChildWork) {
              workInProgress.effectTag |= Update;
            }
          }
          break;
        case SuspenseComponent: {
          const state: SuspenseState | null = workInProgress.memoizedState;
          if (state !== null) {
            if (enableSuspenseServerRenderer) {
              if (state.dehydrated !== null) {
                pushSuspenseContext(
                  workInProgress,
                  setDefaultShallowSuspenseContext(suspenseStackCursor.current)
                );
                // We know that this component will suspend again because if it has
                // been unsuspended it has committed as a resolved Suspense component.
                // If it needs to be retried, it should have work scheduled on it.
                workInProgress.effectTag |= DidCapture;
                break;
              }
            }

            // If this boundary is currently timed out, we need to decide
            // whether to retry the primary children, or to skip over it and
            // go straight to the fallback. Check the priority of the primary
            // child fragment.
            const primaryChildFragment: Fiber = (workInProgress.child: any);
            const primaryChildExpirationTime =
              primaryChildFragment.childExpirationTime;
            if (
              primaryChildExpirationTime !== NoWork &&
              primaryChildExpirationTime >= renderExpirationTime
            ) {
              // The primary children have pending work. Use the normal path
              // to attempt to render the primary children again.
              return updateSuspenseComponent(
                current,
                workInProgress,
                renderExpirationTime
              );
            } else {
              pushSuspenseContext(
                workInProgress,
                setDefaultShallowSuspenseContext(suspenseStackCursor.current)
              );
              // The primary children do not have pending work with sufficient
              // priority. Bailout.
              const child = bailoutOnAlreadyFinishedWork(
                current,
                workInProgress,
                renderExpirationTime
              );
              if (child !== null) {
                // The fallback children have pending work. Skip over the
                // primary children and work on the fallback.
                return child.sibling;
              } else {
                return null;
              }
            }
          } else {
            pushSuspenseContext(
              workInProgress,
              setDefaultShallowSuspenseContext(suspenseStackCursor.current)
            );
          }
          break;
        }
        case SuspenseListComponent: {
          const didSuspendBefore =
            (current.effectTag & DidCapture) !== NoEffect;

          const hasChildWork =
            workInProgress.childExpirationTime >= renderExpirationTime;

          if (didSuspendBefore) {
            if (hasChildWork) {
              // If something was in fallback state last time, and we have all the
              // same children then we're still in progressive loading state.
              // Something might get unblocked by state updates or retries in the
              // tree which will affect the tail. So we need to use the normal
              // path to compute the correct tail.
              return updateSuspenseListComponent(
                current,
                workInProgress,
                renderExpirationTime
              );
            }
            // If none of the children had any work, that means that none of
            // them got retried so they'll still be blocked in the same way
            // as before. We can fast bail out.
            workInProgress.effectTag |= DidCapture;
          }

          // If nothing suspended before and we're rendering the same children,
          // then the tail doesn't matter. Anything new that suspends will work
          // in the "together" mode, so we can continue from the state we had.
          let renderState = workInProgress.memoizedState;
          if (renderState !== null) {
            // Reset to the "together" mode in case we've started a different
            // update in the past but didn't complete it.
            renderState.rendering = null;
            renderState.tail = null;
          }
          pushSuspenseContext(workInProgress, suspenseStackCursor.current);

          if (hasChildWork) {
            break;
          } else {
            // If none of the children had any work, that means that none of
            // them got retried so they'll still be blocked in the same way
            // as before. We can fast bail out.
            return null;
          }
        }
      }
      return bailoutOnAlreadyFinishedWork(
        current,
        workInProgress,
        renderExpirationTime
      );
    } else {
      // An update was scheduled on this fiber, but there are no new props
      // nor legacy context. Set this to false. If an update queue or context
      // consumer produces a changed value, it will set this to true. Otherwise,
      // the component will assume the children have not changed and bail out.
      didReceiveUpdate = false;
    }
  } else {
    didReceiveUpdate = false;
  }

  // Before entering the begin phase, clear pending update priority.
  // TODO: This assumes that we're about to evaluate the component and process
  // NoWork 常量，值为 0 ，清空过期时间
  workInProgress.expirationTime = NoWork;

  // 根据当前 Fiber 的类型决定如何构建起子级 Fiber 对象
  // 文件位置：shared/ReactWorkTags.js
  switch (workInProgress.tag) {
    // 2
    // 函数组件在第一次渲染时使用
    case IndeterminateComponent: {
      return mountIndeterminateComponent(
        current,
        workInProgress,
        workInProgress.type,
        renderExpirationTime
      );
    }
    // 16
    case LazyComponent: {
      const elementType = workInProgress.elementType;
      return mountLazyComponent(
        current,
        workInProgress,
        elementType,
        updateExpirationTime,
        renderExpirationTime
      );
    }
    // 0
    case FunctionComponent: {
      const Component = workInProgress.type;
      const unresolvedProps = workInProgress.pendingProps;
      const resolvedProps =
        workInProgress.elementType === Component
          ? unresolvedProps
          : resolveDefaultProps(Component, unresolvedProps);
      return updateFunctionComponent(
        current,
        workInProgress,
        Component,
        resolvedProps,
        renderExpirationTime
      );
    }
    // 1
    case ClassComponent: {
      const Component = workInProgress.type;
      const unresolvedProps = workInProgress.pendingProps;
      const resolvedProps =
        workInProgress.elementType === Component
          ? unresolvedProps
          : resolveDefaultProps(Component, unresolvedProps);
      return updateClassComponent(
        current,
        workInProgress,
        Component,
        resolvedProps,
        renderExpirationTime
      );
    }
    // 3
    case HostRoot:
      return updateHostRoot(current, workInProgress, renderExpirationTime);
    // 5
    case HostComponent:
      return updateHostComponent(current, workInProgress, renderExpirationTime);
    // 6
    case HostText:
      return updateHostText(current, workInProgress);
    case SuspenseComponent:
      return updateSuspenseComponent(
        current,
        workInProgress,
        renderExpirationTime
      );
    case HostPortal:
      return updatePortalComponent(
        current,
        workInProgress,
        renderExpirationTime
      );
    case ForwardRef: {
      const type = workInProgress.type;
      const unresolvedProps = workInProgress.pendingProps;
      const resolvedProps =
        workInProgress.elementType === type
          ? unresolvedProps
          : resolveDefaultProps(type, unresolvedProps);
      return updateForwardRef(
        current,
        workInProgress,
        type,
        resolvedProps,
        renderExpirationTime
      );
    }
    case Fragment:
      return updateFragment(current, workInProgress, renderExpirationTime);
    case Mode:
      return updateMode(current, workInProgress, renderExpirationTime);
    case Profiler:
      return updateProfiler(current, workInProgress, renderExpirationTime);
    case ContextProvider:
      return updateContextProvider(
        current,
        workInProgress,
        renderExpirationTime
      );
    case ContextConsumer:
      return updateContextConsumer(
        current,
        workInProgress,
        renderExpirationTime
      );
    case MemoComponent: {
      const type = workInProgress.type;
      const unresolvedProps = workInProgress.pendingProps;
      // Resolve outer props first, then resolve inner props.
      let resolvedProps = resolveDefaultProps(type, unresolvedProps);
      if (__DEV__) {
        if (workInProgress.type !== workInProgress.elementType) {
          const outerPropTypes = type.propTypes;
          if (outerPropTypes) {
            checkPropTypes(
              outerPropTypes,
              resolvedProps, // Resolved for outer only
              "prop",
              getComponentName(type),
              getCurrentFiberStackInDev
            );
          }
        }
      }
      resolvedProps = resolveDefaultProps(type.type, resolvedProps);
      return updateMemoComponent(
        current,
        workInProgress,
        type,
        resolvedProps,
        updateExpirationTime,
        renderExpirationTime
      );
    }
    case SimpleMemoComponent: {
      return updateSimpleMemoComponent(
        current,
        workInProgress,
        workInProgress.type,
        workInProgress.pendingProps,
        updateExpirationTime,
        renderExpirationTime
      );
    }
    case IncompleteClassComponent: {
      const Component = workInProgress.type;
      const unresolvedProps = workInProgress.pendingProps;
      const resolvedProps =
        workInProgress.elementType === Component
          ? unresolvedProps
          : resolveDefaultProps(Component, unresolvedProps);
      return mountIncompleteClassComponent(
        current,
        workInProgress,
        Component,
        resolvedProps,
        renderExpirationTime
      );
    }
    case SuspenseListComponent: {
      return updateSuspenseListComponent(
        current,
        workInProgress,
        renderExpirationTime
      );
    }
    case FundamentalComponent: {
      if (enableFundamentalAPI) {
        return updateFundamentalComponent(
          current,
          workInProgress,
          renderExpirationTime
        );
      }
      break;
    }
    case ScopeComponent: {
      if (enableScopeAPI) {
        return updateScopeComponent(
          current,
          workInProgress,
          renderExpirationTime
        );
      }
      break;
    }
    case Block: {
      if (enableBlocksAPI) {
        const block = workInProgress.type;
        const props = workInProgress.pendingProps;
        return updateBlock(
          current,
          workInProgress,
          block,
          props,
          renderExpirationTime
        );
      }
      break;
    }
  }
  invariant(
    false,
    "Unknown unit of work tag (%s). This error is likely caused by a bug in " +
      "React. Please file an issue.",
    workInProgress.tag
  );
}
```

#### 7、updateHostRoot

```js
// src/react/packages/react-reconciler/src/ReactFiberBeginWork.js
// ⭐️ Line 987~1053 ⭐️
//updateHostRoot 方法
// 更新 hostRoot （即 <div id="root"></div> 对应的 Fiber 对象
function updateHostRoot(current, workInProgress, renderExpirationTime) {
  pushHostRootContext(workInProgress);
  // 获取更新队列
  const updateQueue = workInProgress.updateQueue;
  invariant(
    current !== null && updateQueue !== null,
    "If the root does not have an updateQueue, we should have already " +
      "bailed out. This error is likely caused by a bug in React. Please " +
      "file an issue."
  );

  // 获取新的 props 对象，初始化渲染时值为 null
  const nextProps = workInProgress.pendingProps;
  // 获取上一次渲染使用的 state ，初始化渲染时值为 null
  const prevState = workInProgress.memoizedState;
  // 获取上一次渲染使用的 children ，初始化渲染时值为 null
  const prevChildren = prevState !== null ? prevState.element : null;
  // 浅复制更新队列，防止引用属性互相影响
  // workInProgress.updateQueue 浅拷贝 current.updateQueue
  cloneUpdateQueue(current, workInProgress);
  // 获取 updateQueue.payload 并赋值到 workInProgress.memoizedState
  // 要更新的内容就是 element 就是 rootFiber 的子元素
  processUpdateQueue(workInProgress, nextProps, null, renderExpirationTime);
  // 获取 element 所在对象
  const nextState = workInProgress.memoizedState;
  // 从对象中获取 element
  const nextChildren = nextState.element;
  // 在计算 state 后如果前后两个 children 相同的情况
  // prevChildren => null
  // nextState => App
  // false
  if (nextChildren === prevChildren) {
    // If the state is the same as before, that's a bailout because we had
    // no work that expires at this time.
    resetHydrationState();
    return bailoutOnAlreadyFinishedWork(
      current,
      workInProgress,
      renderExpirationTime
    );
  }
  // 获取 fiberRoot 对象
  const root: FiberRoot = workInProgress.stateNode;
  // 服务器端渲染走 if
  if (root.hydrate && enterHydrationState(workInProgress)) {
    // If we don't have any current children this might be the first pass.
    // We always try to hydrate. If this isn't a hydration pass there won't
    // be any children to hydrate which is effectively the same thing as
    // not hydrating.

    let child = mountChildFibers(
      workInProgress,
      null,
      nextChildren,
      renderExpirationTime
    );
    workInProgress.child = child;

    let node = child;
    while (node) {
      // Mark each child as hydrating. This is a fast path to know whether this
      // tree is part of a hydrating tree. This is used to determine if a child
      // node has fully mounted yet, and for scheduling event replaying.
      // Conceptually this is similar to Placement in that a new subtree is
      // inserted into the React tree here. It just happens to not need DOM
      // mutations because it already exists.
      node.effectTag = (node.effectTag & ~Placement) | Hydrating;
      node = node.sibling;
    }
  } else {
    // 客户端渲染走 else
    // 构建子节点 fiber 对象
    reconcileChildren(
      current,
      workInProgress,
      nextChildren,
      renderExpirationTime
    );
    resetHydrationState();
  }
  // 返回子节点 fiber 对象
  return workInProgress.child;
}
```

#### 8、reconcileChildren

```js
// src/react/packages/react-reconciler/src/ReactFiberBeginWork.js
// ⭐️ Line 212~243 ⭐️
// reconcileChildren 方法
// 构建子级 Fiber 对象
export function reconcileChildren(
  // 旧 Fiber
  current: Fiber | null,
  // 父级 Fiber
  workInProgress: Fiber,
  // 子级 vdom 对象
  nextChildren: any,
  // 初始渲染，整型最大值，代表同步任务
  renderExpirationTime: ExpirationTime
) {
  /**
   * 为什么要传递 current ？
   * 如果不是初始渲染的情况，要进行新旧 Fiber 对比
   * 初始渲染时则用不到 current
   */
  // 如果旧 Fiber 为 null 表示初始渲染
  if (current === null) {
    workInProgress.child = mountChildFibers(
      workInProgress,
      null,
      nextChildren,
      renderExpirationTime
    );
  } else {
    // If the current child is the same as the work in progress, it means that
    // we haven't yet started any work on these children. Therefore, we use
    // the clone algorithm to create a copy of all the current children.

    // If we had any progressed work already, that is invalid at this point so
    // let's throw it out.
    workInProgress.child = reconcileChildFibers(
      workInProgress,
      current.child,
      nextChildren,
      renderExpirationTime
    );
  }
}
```

#### 9、mountChildFibers（ChildReconciler）

```js
// src/react/packages/react-reconciler/src/ReactChildFiber.js
// ⭐️ Line 264~1418 ⭐️
// mountChildFibers（ChildReconciler） 方法
function ChildReconciler(shouldTrackSideEffects) {
  // ……（省略若干个方法）

  // 处理子元素是数组的情况
  function reconcileChildrenArray(
    // 父级 Fiber
    returnFiber: Fiber,
    currentFirstChild: Fiber | null,
    // 子级 vdom 数组
    newChildren: Array<*>,
    expirationTime: ExpirationTime
  ): Fiber | null {
    // ……

    /**
     * 存储第一个子节点 Fiber 对象
     * 方法返回的也是第一个子节点 Fiber 对象
     * 因为其他子节点 Fiber 对象都存储在上一个子 Fiber 节点对象的 sibling 属性上
     */
    let resultingFirstChild: Fiber | null = null;
    // 上一次创建的 Fiber 对象
    let previousNewFiber: Fiber | null = null;

    // 初始渲染没有旧的子级，所以为 null
    let oldFiber = currentFirstChild;
    let lastPlacedIndex = 0;
    let newIdx = 0;
    let nextOldFiber = null;
    // 初始渲染， oldFiber 为 null ，循环不执行
    for (; oldFiber !== null && newIdx < newChildren.length; newIdx++) {
      if (oldFiber.index > newIdx) {
        nextOldFiber = oldFiber;
        oldFiber = null;
      } else {
        nextOldFiber = oldFiber.sibling;
      }
      const newFiber = updateSlot(
        returnFiber,
        oldFiber,
        newChildren[newIdx],
        expirationTime
      );
      if (newFiber === null) {
        // TODO: This breaks on empty slots like null children. That's
        // unfortunate because it triggers the slow path all the time. We need
        // a better way to communicate whether this was a miss or null,
        // boolean, undefined, etc.
        if (oldFiber === null) {
          oldFiber = nextOldFiber;
        }
        break;
      }
      if (shouldTrackSideEffects) {
        if (oldFiber && newFiber.alternate === null) {
          // We matched the slot, but we didn't reuse the existing fiber, so we
          // need to delete the existing child.
          deleteChild(returnFiber, oldFiber);
        }
      }
      lastPlacedIndex = placeChild(newFiber, lastPlacedIndex, newIdx);
      if (previousNewFiber === null) {
        // TODO: Move out of the loop. This only happens for the first run.
        resultingFirstChild = newFiber;
      } else {
        // TODO: Defer siblings if we're not at the right index for this slot.
        // I.e. if we had null values before, then we want to defer this
        // for each null value. However, we also don't want to call updateSlot
        // with the previous one.
        previousNewFiber.sibling = newFiber;
      }
      previousNewFiber = newFiber;
      oldFiber = nextOldFiber;
    }

    // 初始渲染不执行
    if (newIdx === newChildren.length) {
      // We've reached the end of the new children. We can delete the rest.
      deleteRemainingChildren(returnFiber, oldFiber);
      return resultingFirstChild;
    }

    // oldFiber 为空，说明是初始渲染
    if (oldFiber === null) {
      // 遍历子 vdom 对象
      for (; newIdx < newChildren.length; newIdx++) {
        // 创建子 vdom 对应的 fiber 对象
        const newFiber = createChild(
          returnFiber,
          newChildren[newIdx],
          expirationTime
        );
        // 如果 newFiber 为 null
        if (newFiber === null) {
          // 进入下次循环
          continue;
        }
        // 初始渲染时值为 newFiber 添加了 index 属性，其他事没敢。
        // lastPlacedIndex 被原封不动的返回了
        lastPlacedIndex = placeChild(newFiber, lastPlacedIndex, newIdx);
        // 为当前节点设置下一个兄弟节点
        if (previousNewFiber === null) {
          // 存储第一个子 Fiber ，发生在第一次循环时
          resultingFirstChild = newFiber;
        } else {
          // 为节点设置下一个兄弟 Fiber
          previousNewFiber.sibling = newFiber;
        }
        // 在循环的过程中更新上一个创建的 Fiber 对象
        previousNewFiber = newFiber;
      }
      // 返回创建好的子 Fiber
      // 其他 Fiber 都作为 sibling 存在
      return resultingFirstChild;
    }

    // 下面的代码初始渲染不执行
    const existingChildren = mapRemainingChildren(returnFiber, oldFiber);

    // Keep scanning and use the map to restore deleted items as moves.
    for (; newIdx < newChildren.length; newIdx++) {
      const newFiber = updateFromMap(
        existingChildren,
        returnFiber,
        newIdx,
        newChildren[newIdx],
        expirationTime
      );
      if (newFiber !== null) {
        if (shouldTrackSideEffects) {
          if (newFiber.alternate !== null) {
            // The new fiber is a work in progress, but if there exists a
            // current, that means that we reused the fiber. We need to delete
            // it from the child list so that we don't add it to the deletion
            // list.
            existingChildren.delete(
              newFiber.key === null ? newIdx : newFiber.key
            );
          }
        }
        lastPlacedIndex = placeChild(newFiber, lastPlacedIndex, newIdx);
        if (previousNewFiber === null) {
          resultingFirstChild = newFiber;
        } else {
          previousNewFiber.sibling = newFiber;
        }
        previousNewFiber = newFiber;
      }
    }

    if (shouldTrackSideEffects) {
      // Any existing children that weren't consumed above were deleted. We need
      // to add them to the deletion list.
      existingChildren.forEach((child) => deleteChild(returnFiber, child));
    }

    return resultingFirstChild;
  }

  // ……（省略几个方法）

  // 处理子元素是单个对象的情况
  function reconcileSingleElement(
    // 父 Fiber 对象
    returnFiber: Fiber,
    // 备份子 Fiber
    currentFirstChild: Fiber | null,
    // 子 vdom 对象
    element: ReactElement,
    expirationTime: ExpirationTime
  ): Fiber {
    const key = element.key;
    let child = currentFirstChild;
    // 初始渲染 currentFirstChild 为 null
    // false
    while (child !== null) {
      // ……
    }

    // 查看子 vdom 对象是否表示 fragment
    // false
    if (element.type === REACT_FRAGMENT_TYPE) {
      // ……
    } else {
      // 根据 React Element 创建 Fiber 对象
      // 返回创建好的 Fiber 对象
      const created = createFiberFromElement(
        element,
        // 用来表示当前组件下的所有子组件要用处于何种渲染模式
        // 文件位置： ./ReactTypeOfMode.js
        // 0    同步渲染模式
        // 100  异步渲染模式
        returnFiber.mode,
        expirationTime
      );
      // 添加 ref 属性 { current: DOM }
      created.ref = coerceRef(returnFiber, currentFirstChild, element);
      // 添加父级 Fiber 对象
      created.return = returnFiber;
      // 返回创建好的子 Fiber
      return created;
    }
  }

  // ……（省略一个方法）

  function reconcileChildFibers(
    // 父 Fiber 对象
    returnFiber: Fiber,
    // 旧的第一个子 Fiber ，初始渲染为 null
    currentFirstChild: Fiber | null,
    // 新的子 vdom 对象
    newChild: any,
    // 初始渲染，整型最大值，代表同步任务
    expirationTime: ExpirationTime
  ): Fiber | null {
    // 这是入口方法，根据 newChild 类型进行相应处理

    // 判断新的子 vdom 是否为占位组件，比如 <></>
    // false
    const isUnkeyedTopLevelFragment =
      typeof newChild === "object" &&
      newChild !== null &&
      newChild.type === REACT_FRAGMENT_TYPE &&
      newChild.key === null;
    // 如果 newChild 为占位符，使用占位符组件的子元素作为 newChild
    if (isUnkeyedTopLevelFragment) {
      newChild = newChild.props.children;
    }

    // 检测 newChild 是否为对象类型（单个子元素就是对象，多个子元素就是数组）
    const isObject = typeof newChild === "object" && newChild !== null;

    // newChild 是单个对象的情况（只有一个子元素）
    if (isObject) {
      // 匹配子元素的类型
      switch (newChild.$$typeof) {
        // 子元素为 ReactElement
        case REACT_ELEMENT_TYPE:
          // 为 Fiber 对象设置 effectTag 属性
          // 返回创建好的子 Fiber
          return placeSingleChild(
            // 处理单个 React Element 的情况
            // 内部会调用其他方法创建对应的 Fiber 对象
            reconcileSingleElement(
              returnFiber,
              currentFirstChild,
              newChild,
              expirationTime
            )
          );
        // ……（省略一个 case ）
      }
    }

    // ……（省略部分代码）

    // children 是数组的情况
    if (isArray(newChild)) {
      // 返回创建好的子 Fiber
      return reconcileChildrenArray(
        returnFiber,
        currentFirstChild,
        newChild,
        expirationTime
      );
    }

    // Remaining cases are all treated as empty.
    return deleteRemainingChildren(returnFiber, currentFirstChild);
  }

  return reconcileChildFibers;
}

/**
 * shouldTrackSideEffects 标识，是否为 Fiber 对象添加 effectTag
 * true 添加，false 不添加
 * 对于初始渲染来说，只有根组件需要添加，其他元素不需要添加，防止过多的 DOM 操作
 */
// 用于更新
export const reconcileChildFibers = ChildReconciler(true);
// 用于初始渲染
export const mountChildFibers = ChildReconciler(false);
```

#### 10、completeUnitOfWork

```js
// src/react/packages/react-reconciler/src/ReactFiberWorkLoop.js
/**
 *
 * 从下至上移动到该节点的兄弟节点, 如果一直往上没有兄弟节点就返回父节点, 最终会到达 root 节点
 * 1. （从子级到父级）创建其他节点的 Fiber 对象
 * 2. 创建每一个节点的真实 DOM 对象并将其添加到 stateNode 属性中
 * 3. 构建 effect 链表结构
 */
function completeUnitOfWork(unitOfWork: Fiber): Fiber | null {
  // 为 workInProgress 全局变量重新赋值
  workInProgress = unitOfWork;
  do {
    // 获取备份节点
    // 初始化渲染 非根 Fiber 对象没有备份节点 所以 current 为 null
    const current = workInProgress.alternate;
    // 父级 Fiber 对象, 非根 Fiber 对象都有父级
    const returnFiber = workInProgress.return;
    // 判断传入的 Fiber 对象是否构建完成, 任务调度相关
    // & 是表示位的与运算, 把左右两边的数字转化为二进制
    // 然后每一位分别进行比较, 如果相等就为1, 不相等即为0
    // 此处应用"位与"运算符的目的是"清零"
    // true
    if ((workInProgress.effectTag & Incomplete) === NoEffect) {
      let next;
      // 如果不能使用分析器的 timer, 直接执行 completeWork
      // enableProfilerTimer => true
      // 但此处无论条件是否成立都会执行 completeWork
      if (
        !enableProfilerTimer ||
        (workInProgress.mode & ProfileMode) === NoMode
      ) {
        // 重点代码(二)
        // 创建节点真实 DOM 对象并将其添加到 stateNode 属性中
        next = completeWork(current, workInProgress, renderExpirationTime);
      } else {
        // 创建节点真实 DOM 对象并将其添加到 stateNode 属性中
        next = completeWork(current, workInProgress, renderExpirationTime);
      }
      // 重点代码(一)
      // 如果子级存在
      if (next !== null) {
        // 返回子级 一直返回到 workLoopSync
        // 再重新执行 performUnitOfWork 构建子级 Fiber 节点对象
        return next;
      }

      // 构建 effect 链表结构
      // 如果不是根 Fiber 就是 true 否则就是 false
      // 将子树和此 Fiber 的所有 effect 附加到父级的 effect 列表中
      if (
        // 如果父 Fiber 存在 并且
        returnFiber !== null &&
        // 父 Fiber 对象中的 effectTag 为 0
        (returnFiber.effectTag & Incomplete) === NoEffect
      ) {
        // 将子树和此 Fiber 的所有副作用附加到父级的 effect 列表上

        // 以下两个判断的作用是搜集子 Fiber的 effect 到父 Fiber
        if (returnFiber.firstEffect === null) {
          // first
          returnFiber.firstEffect = workInProgress.firstEffect;
        }

        if (workInProgress.lastEffect !== null) {
          if (returnFiber.lastEffect !== null) {
            // next
            returnFiber.lastEffect.nextEffect = workInProgress.firstEffect;
          }
          // last
          returnFiber.lastEffect = workInProgress.lastEffect;
        }

        // 获取副作用标记
        // 初始渲染时除[根组件]以外的 Fiber, effectTag 值都为 0, 即不需要执行任何真实DOM操作
        // 根组件的 effectTag 值为 3, 即需要将此节点对应的真实DOM对象添加到页面中
        const effectTag = workInProgress.effectTag;

        // 创建 effect 列表时跳过 NoWork(0) 和 PerformedWork(1) 标记
        // PerformedWork 由 React DevTools 读取, 不提交
        // 初始渲染时 只有遍历到了根组件 判断条件才能成立, 将 effect 链表添加到 rootFiber
        // 初始渲染 FiberRoot 对象中的 firstEffect 和 lastEffect 都是 App 组件
        // 因为当所有节点在内存中构建完成后, 只需要一次将所有 DOM 添加到页面中
        if (effectTag > PerformedWork) {
          // false
          if (returnFiber.lastEffect !== null) {
            returnFiber.lastEffect.nextEffect = workInProgress;
          } else {
            // 为 fiberRoot 添加 firstEffect
            returnFiber.firstEffect = workInProgress;
          }
          // 为 fiberRoot 添加 lastEffect
          returnFiber.lastEffect = workInProgress;
        }
      }
    } else {
      // 忽略了初始渲染不执行的代码
    }
    // 获取下一个同级 Fiber 对象
    const siblingFiber = workInProgress.sibling;
    // 如果下一个同级 Fiber 对象存在
    if (siblingFiber !== null) {
      // 返回下一个同级 Fiber 对象
      return siblingFiber;
    }
    // 否则退回父级
    workInProgress = returnFiber;
  } while (workInProgress !== null);

  // 当执行到这里的时候, 说明遍历到了 root 节点, 已完成遍历
  // 更新 workInProgressRootExitStatus 的状态为 已完成
  if (workInProgressRootExitStatus === RootIncomplete) {
    workInProgressRootExitStatus = RootCompleted;
  }
  return null;
}
```

#### 11、completeWork

```js
// src/react/packages/react-reconciler/src/ReactFiberCompleteWork.js

function completeWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderExpirationTime: ExpirationTime
): Fiber | null {
  // 获取待更新 props
  const newProps = workInProgress.pendingProps;
  switch (workInProgress.tag) {
    // 0
    case FunctionComponent:
      return null;
    // 3
    case HostRoot: {
      updateHostContainer(workInProgress);
      return null;
    }
    // 5
    case HostComponent: {
      // 获取 rootDOM 节点 <div id="root"></div>
      const rootContainerInstance = getRootHostContainer();
      // 节点的具体的类型 div span ...
      const type = workInProgress.type;
      // false current = null
      if (current !== null && workInProgress.stateNode != null) {
        // 初始渲染不执行
      } else {
        // false
        if (wasHydrated) {
          // 初始渲染不执行
        } else {
          // 创建节点实例对象 <div></div> <span></span>
          let instance = createInstance(
            type,
            newProps,
            rootContainerInstance,
            currentHostContext,
            workInProgress
          );
          /**
           * 将所有的子级追加到父级中
           * instance 为父级
           * workInProgress.child 为子级
           */
          appendAllChildren(instance, workInProgress, false, false);
          // 为 Fiber 对象添加 stateNode 属性
          workInProgress.stateNode = instance;
        }
        // 处理 ref DOM 引用
        if (workInProgress.ref !== null) {
          markRef(workInProgress);
        }
      }
      return null;
    }
  }
}
```

#### 12、appendAllChildren

```js
// src/react/packages/react-reconciler/src/ReactFiberCompleteWork.js

let appendAllChildren;

appendAllChildren = function (
  parent: Instance,
  workInProgress: Fiber,
  needsVisibilityToggle: boolean,
  isHidden: boolean
) {
  // 获取子级
  let node = workInProgress.child;
  // 如果子级不为空 执行循环
  while (node !== null) {
    // 如果 node 是普通 ReactElement 或者为文本
    if (node.tag === HostComponent || node.tag === HostText) {
      // 将子级追加到父级中
      appendInitialChild(parent, node.stateNode);
    } else if (node.child !== null) {
      // 如果 node 不是普通 ReactElement 又不是文本
      // 将 node 视为组件, 组件本身不能转换为真实 DOM 元素
      // 获取到组件的第一个子元素, 继续执行循环
      node.child.return = node;
      node = node.child;
      continue;
    }
    // 如果 node 和 workInProgress 是同一个节点
    // 说明 node 已经退回到父级 终止循环
    // 说明此时所有子级都已经追加到父级中了
    if (node === workInProgress) {
      return;
    }
    // 处理子级节点的兄弟节点
    while (node.sibling === null) {
      // 如果节点没有父级或者节点的父级是自己, 退出循环
      // 说明此时所有子级都已经追加到父级中了
      if (node.return === null || node.return === workInProgress) {
        return;
      }
      // 更新 node
      node = node.return;
    }
    // 更新父级 方便回退
    node.sibling.return = node.return;
    // 将 node 更新为下一个兄弟节点
    node = node.sibling;
  }
};
```

## （二）commit 阶段

### 一）finishSyncRender

```js
// src/react/packages/react-reconciler/src/ReactFiberWorkLoop.js

function finishSyncRender(root) {
  // 销毁 workInProgress Fiber 树
  // 因为待提交 Fiber 对象已经被存储在了 root.finishedWork 中
  workInProgressRoot = null;
  // 进入 commit 阶段
  commitRoot(root);
}
```

### 二）commitRoot

```js
// sre/react/packages/react-reconciler/src/ReactFiberWorkLoop.js

function commitRoot(root) {
  // 获取任务优先级 97 => 普通优先级
  const renderPriorityLevel = getCurrentPriorityLevel();
  // 使用最高优先级执行当前任务, 因为 commit 阶段不可以被打断
  // ImmediatePriority, 优先级为 99, 最高优先级
  runWithPriority(
    ImmediatePriority,
    commitRootImpl.bind(null, root, renderPriorityLevel)
  );
  return null;
}
```

### 三）commitRootImpl

commit 阶段可以分为三个子阶段：

- before mutation 阶段（执行 DOM 操作前）
- mutation 阶段（执行 DOM 操作）
- layout 阶段（执行 DOM 操作后）

```js
// src/react/packages/react-reconciler/src/ReactFiberWorkLoop.js

function commitRootImpl(root, renderPriorityLevel) {
  // 获取待提交 Fiber 对象 rootFiber
  const finishedWork = root.finishedWork;
  // 如果没有任务要执行
  if (finishedWork === null) {
    // 阻止程序继续向下执行
    return null;
  }
  // 重置为默认值
  root.finishedWork = null;
  root.callbackNode = null;
  root.callbackExpirationTime = NoWork;
  root.callbackPriority = NoPriority;
  root.nextKnownPendingLevel = NoWork;

  // 获取要执行 DOM 操作的副作用列表
  let firstEffect = finishedWork.firstEffect;

  // true
  if (firstEffect !== null) {
    // commit 第一个子阶段
    nextEffect = firstEffect;
    // 处理类组件的 getSnapShotBeforeUpdate 生命周期函数
    do {
      invokeGuardedCallback(null, commitBeforeMutationEffects, null);
    } while (nextEffect !== null);

    // commit 第二个子阶段
    nextEffect = firstEffect;
    do {
      invokeGuardedCallback(
        null,
        commitMutationEffects,
        null,
        root,
        renderPriorityLevel
      );
    } while (nextEffect !== null);
    // 将 workInProgress Fiber 树变成 current Fiber 树
    root.current = finishedWork;

    // commit 第三个子阶段
    nextEffect = firstEffect;
    do {
      invokeGuardedCallback(
        null,
        commitLayoutEffects,
        null,
        root,
        expirationTime
      );
    } while (nextEffect !== null);

    // 重置 nextEffect
    nextEffect = null;
  }
}
```

### 四）第一子阶段

#### 1、commitBeforeMutationEffects

```js
// src/react/packages/react-reconciler/src/ReactFiberWorkLoop.js

// commit 阶段的第一个子阶段
// 调用类组件的 getSnapshotBeforeUpdate 生命周期函数
function commitBeforeMutationEffects() {
  // 循环 effect 链
  while (nextEffect !== null) {
    // nextEffect 是 effect 链上从 firstEffect 到 lastEffec 的每一个需要commit的 fiber 对象

    // 初始化渲染第一个 nextEffect 为 App 组件
    // effectTag => 3
    const effectTag = nextEffect.effectTag;
    // console.log(effectTag);
    // nextEffect = null;
    // return;

    // 如果 fiber 对象中里有 Snapshot 这个 effectTag 的话
    // Snapshot 和更新有关系 初始化渲染 不执行
    // false
    if ((effectTag & Snapshot) !== NoEffect) {
      // 获取当前 fiber 节点
      const current = nextEffect.alternate;
      // 当 nextEffect 上有 Snapshot 这个 effectTag 时
      // 执行以下方法, 主要是类组件调用 getSnapshotBeforeUpdate 生命周期函数
      commitBeforeMutationEffectOnFiber(current, nextEffect);
    }
    // 更新循环条件
    nextEffect = nextEffect.nextEffect;
  }
}
```

#### 2、commitBeforeMutationLifeCycles

```js
// src/react/packages/react-reconciler/src/ReactFiberCommitWork.js

function commitBeforeMutationLifeCycles(
  current: Fiber | null,
  finishedWork: Fiber
): void {
  switch (finishedWork.tag) {
    case FunctionComponent:
    case ForwardRef:
    case SimpleMemoComponent:
    case Block: {
      return;
    }
    // 如果该 fiber 类型是 ClassComponent
    case ClassComponent: {
      if (finishedWork.effectTag & Snapshot) {
        if (current !== null) {
          // 旧的 props
          const prevProps = current.memoizedProps;
          // 旧的 state
          const prevState = current.memoizedState;
          // 获取 classComponent 组件的实例对象
          const instance = finishedWork.stateNode;
          // 执行 getSnapshotBeforeUpdate 生命周期函数
          // 在组件更新前捕获一些 DOM 信息
          // 返回自定义的值或 null, 统称为 snapshot
          const snapshot = instance.getSnapshotBeforeUpdate(
            finishedWork.elementType === finishedWork.type
              ? prevProps
              : resolveDefaultProps(finishedWork.type, prevProps),
            prevState
          );
        }
      }
      return;
    }
    case HostRoot:
    case HostComponent:
    case HostText:
    case HostPortal:
    case IncompleteClassComponent:
      // Nothing to do for these component types
      return;
  }
}
```

### 五）第二子阶段

#### 1、commitMutationEffects

```js
// src/react/packages/react-reconciler/src/ReactFiberWorkLoop.js

// commit 阶段的第二个子阶段
// 根据 effectTag 执行 DOM 操作
function commitMutationEffects(root: FiberRoot, renderPriorityLevel) {
  // 循环 effect 链
  while (nextEffect !== null) {
    // 获取 effectTag
    // 初始渲染第一次循环为 App 组件
    // 即将根组件及内部所有内容一次性添加到页面中
    const effectTag = nextEffect.effectTag;

    // 根据 effectTag 分别处理
    let primaryEffectTag =
      effectTag & (Placement | Update | Deletion | Hydrating);
    // 匹配 effectTag
    // 初始渲染 primaryEffectTag 为 2 匹配到 Placement
    switch (primaryEffectTag) {
      // 针对该节点及子节点进行插入操作
      case Placement: {
        commitPlacement(nextEffect);
        // effectTag 从 3 变为 1
        // 从 effect 标签中清除 "placement" 重置 effectTag 值
        // 以便我们知道在调用诸如componentDidMount之类的任何生命周期之前已将其插入。
        nextEffect.effectTag &= ~Placement;
        break;
      }
      // 插入并更新 DOM
      case PlacementAndUpdate: {
        // 插入
        commitPlacement(nextEffect);
        nextEffect.effectTag &= ~Placement;

        // 更新
        const current = nextEffect.alternate;
        commitWork(current, nextEffect);
        break;
      }
      // 服务器端渲染
      case Hydrating: {
        nextEffect.effectTag &= ~Hydrating;
        break;
      }
      // 服务器端渲染
      case HydratingAndUpdate: {
        nextEffect.effectTag &= ~Hydrating;

        // Update
        const current = nextEffect.alternate;
        commitWork(current, nextEffect);
        break;
      }
      // 更新 DOM
      case Update: {
        const current = nextEffect.alternate;
        commitWork(current, nextEffect);
        break;
      }
      // 删除 DOM
      case Deletion: {
        commitDeletion(root, nextEffect, renderPriorityLevel);
        break;
      }
    }
    nextEffect = nextEffect.nextEffect;
  }
}
```

#### 2、commitPlacement

```js
// src/react/packages/react-reconciler/src/ReactFiberCommitWork.js

// 挂载 DOM 元素
function commitPlacement(finishedWork: Fiber): void {
  // finishedWork 初始化渲染时为根组件 Fiber 对象
  // 获取非组件父级 Fiber 对象
  // 初始渲染时为 <div id="root"></div>
  const parentFiber = getHostParentFiber(finishedWork);

  // 存储真正的父级 DOM 节点对象
  let parent;
  // 是否为渲染容器
  // 渲染容器和普通react元素的主要区别在于是否需要特殊处理注释节点
  let isContainer;
  // 获取父级 DOM 节点对象
  // 但是初始渲染时 rootFiber 对象中的 stateNode 存储的是 FiberRoot
  const parentStateNode = parentFiber.stateNode;
  // 判断父节点的类型
  // 初始渲染时是 hostRoot 3
  switch (parentFiber.tag) {
    case HostComponent:
      parent = parentStateNode;
      isContainer = false;
      break;
    case HostRoot:
      // 获取真正的 DOM 节点对象
      // <div id="root"></div>
      parent = parentStateNode.containerInfo;
      // 是 container 容器
      isContainer = true;
      break;
    case HostPortal:
      parent = parentStateNode.containerInfo;
      isContainer = true;
      break;
    case FundamentalComponent:
      if (enableFundamentalAPI) {
        parent = parentStateNode.instance;
        isContainer = false;
      }
  }
  // 查看当前节点是否有下一个兄弟节点
  // 有, 执行 insertBefore
  // 没有, 执行 appendChild
  const before = getHostSibling(finishedWork);
  // 渲染容器
  if (isContainer) {
    // 向父节点中追加节点 或者 将子节点插入到 before 节点的前面
    insertOrAppendPlacementNodeIntoContainer(finishedWork, before, parent);
  } else {
    // 非渲染容器
    // 向父节点中追加节点 或者 将子节点插入到 before 节点的前面
    insertOrAppendPlacementNode(finishedWork, before, parent);
  }
}
```

#### 3、getHostParentFiber

```js
// src/react/packages/react-reconciler/src/ReactFiberCommitWork.js

// 获取 HostRootFiber 对象
function getHostParentFiber(fiber: Fiber): Fiber {
  // 获取当前 Fiber 父级
  let parent = fiber.return;
  // 查看父级是否为 null
  while (parent !== null) {
    // 查看父级是否为 hostRoot
    if (isHostParent(parent)) {
      // 返回
      return parent;
    }
    // 继续向上查找
    parent = parent.return;
  }
}
```

#### 4、insertOrAppendPlacementNodeIntoContainer

```js
// src/react/packages/react-reconciler/src/ReactFiberCommitWork.js

// 向容器中追加 | 插入到某一个节点的前面
function insertOrAppendPlacementNodeIntoContainer(
  node: Fiber,
  before: ?Instance,
  parent: Container
): void {
  const { tag } = node;
  // 如果待插入的节点是一个 DOM 元素或者文本的话
  // 比如 组件fiber => false div => true
  const isHost = tag === HostComponent || tag === HostText;

  if (isHost || (enableFundamentalAPI && tag === FundamentalComponent)) {
    // 获取 DOM 节点
    const stateNode = isHost ? node.stateNode : node.stateNode.instance;
    // 如果 before 存在
    if (before) {
      // 插入到 before 前面
      insertInContainerBefore(parent, stateNode, before);
    } else {
      // 追加到父容器中
      appendChildToContainer(parent, stateNode);
    }
  } else {
    // 如果是组件节点, 比如 ClassComponent, 则找它的第一个子节点(DOM 元素)
    // 进行插入操作
    const child = node.child;
    if (child !== null) {
      // 向父级中追加子节点或者将子节点插入到 before 的前面
      insertOrAppendPlacementNodeIntoContainer(child, before, parent);
      // 获取下一个兄弟节点
      let sibling = child.sibling;
      // 如果兄弟节点存在
      while (sibling !== null) {
        // 向父级中追加子节点或者将子节点插入到 before 的前面
        insertOrAppendPlacementNodeIntoContainer(sibling, before, parent);
        // 同步兄弟节点
        sibling = sibling.sibling;
      }
    }
  }
}
```

#### 5、insertInContainerBefore

```js
// src/react/packages/react-dom/src/client/ReactDOMHostConfig.js

export function insertInContainerBefore(
  container: Container,
  child: Instance | TextInstance,
  beforeChild: Instance | TextInstance | SuspenseInstance
): void {
  // 如果父容器是注释节点
  if (container.nodeType === COMMENT_NODE) {
    // 找到注释节点的父级节点 因为注释节点没法调用 insertBefore
    (container.parentNode: any).insertBefore(child, beforeChild);
  } else {
    // 将 child 插入到 beforeChild 的前面
    container.insertBefore(child, beforeChild);
  }
}
```

#### 6、appendChildToContainer

```js
// src/react/packages/react-dom/src/client/ReactDOMHostConfig.js

export function appendChildToContainer(
  container: Container,
  child: Instance | TextInstance
): void {
  // 监测 container 是否注释节点
  if (container.nodeType === COMMENT_NODE) {
    // 获取父级的父级
    parentNode = (container.parentNode: any);
    // 将子级节点插入到注释节点的前面
    parentNode.insertBefore(child, container);
  } else {
    // 直接将 child 插入到父级中
    parentNode = container;
    parentNode.appendChild(child);
  }
}
```

### 六）第三子阶段

#### 1、commitLayoutEffects

```js
// src/react/packages/react-reconciler/src/ReactFiberWorkLoop.js

// commit 阶段的第三个子阶段
function commitLayoutEffects(
  root: FiberRoot,
  committedExpirationTime: ExpirationTime
) {
  while (nextEffect !== null) {
    // 此时 effectTag 已经被重置为 1, 表示 DOM 操作已经完成
    const effectTag = nextEffect.effectTag;
    // 调用生命周期函数和钩子函数
    // 前提是类组件中调用了生命周期函数 或者函数组件中调用了 useEffect
    if (effectTag & (Update | Callback)) {
      // 类组件处理生命周期函数
      // 函数组件处理钩子函数
      commitLayoutEffectOnFiber(
        root,
        current,
        nextEffect,
        committedExpirationTime
      );
    }
    // 更新循环条件
    nextEffect = nextEffect.nextEffect;
  }
}
```

#### 2、commitLifeCycles

```js
// src/react/packages/react-reconciler/src/ReactFiberCommitWork.js

function commitLifeCycles(
  finishedRoot: FiberRoot,
  current: Fiber | null,
  finishedWork: Fiber,
  committedExpirationTime: ExpirationTime,
): void {
  switch (finishedWork.tag) {
    case FunctionComponent: {
      // 处理钩子函数
      commitHookEffectListMount(HookLayout | HookHasEffect, finishedWork);
      return;
    }
    case ClassComponent: {
      // 获取类组件实例对象
      const instance = finishedWork.stateNode;
      // 如果在类组件中存在生命周期函数判断条件就会成立
      if (finishedWork.effectTag & Update) {
        // 初始渲染阶段
        if (current === null) {
          // 调用 componentDidMount 生命周期函数
          instance.componentDidMount();
        } else {
          // 更新阶段
          // 获取旧的 props
          const prevProps = finishedWork.elementType === finishedWork.type
              ? current.memoizedProps
              : resolveDefaultProps(finishedWork.type, current.memoizedProps);
          // 获取旧的 state
          const prevState = current.memoizedState;
          // 调用 componentDidUpdate 生命周期函数
          // instance.__reactInternalSnapshotBeforeUpdate 快照 getSnapShotBeforeUpdate 方法的返回值
          instance.componentDidUpdate(prevProps, prevState, instance.__reactInternalSnapshotBeforeUpdate);
        }
      }
      // 获取任务队列
      const updateQueue = finishedWork.updateQueue;
      // 如果任务队列存在
      if (updateQueue !== null) {
        /**
         * 调用 ReactElement 渲染完成之后的回调函数
         * 即 render 方法的第三个参数
         */
        commitUpdateQueue(finishedWork, updateQueue, instance);
      }
      return;
    }
}
```

#### 3、commitUpdateQueue

```js
// src/react/packages/react-reconciler/src/ReactUpdateQueue.js

/**
 * 执行渲染完成之后的回调函数
 */
export function commitUpdateQueue<State>(
  finishedWork: Fiber,
  finishedQueue: UpdateQueue<State>,
  instance: any
): void {
  // effects 为数组, 存储任务对象 (Update 对象)
  // 但前提是在调用 render 方法时传递了回调函数, 就是 render 方法的第三个参数
  const effects = finishedQueue.effects;
  // 重置 finishedQueue.effects 数组
  finishedQueue.effects = null;
  // 如果传递了 render 方法的第三个参数, effect 数组就不会为 null
  if (effects !== null) {
    // 遍历 effect 数组
    for (let i = 0; i < effects.length; i++) {
      // 获取数组中的第 i 个需要执行的 effect
      const effect = effects[i];
      // 获取 callback 回调函数
      const callback = effect.callback;
      // 如果回调函数不为 null
      if (callback !== null) {
        // 清空 effect 中的 callback
        effect.callback = null;
        // 执行回调函数
        callCallback(callback, instance);
      }
    }
  }
}
```

#### 4、commitHookEffectListMount

```js
// src/react/packages/react-reconciler/src/ReactFiberCommitWork.js

/**
 * useEffect 回调函数调用
 */
function commitHookEffectListMount(tag: number, finishedWork: Fiber) {
  // 获取任务队列
  const updateQueue: FunctionComponentUpdateQueue | null =
    (finishedWork.updateQueue: any);
  // 获取 lastEffect
  let lastEffect = updateQueue !== null ? updateQueue.lastEffect : null;
  // 如果 lastEffect 不为 null
  if (lastEffect !== null) {
    // 获取要执行的副作用
    const firstEffect = lastEffect.next;
    let effect = firstEffect;
    // 通过遍历的方式调用 useEffect 中的回调函数
    // 在组件中定义了调用了几次 useEffect 遍历就会执行几次
    do {
      if ((effect.tag & tag) === tag) {
        // Mount
        const create = effect.create;
        // create 就是 useEffect 方法的第一个参数
        // 返回值就是清理函数
        effect.destroy = create();
      }
      // 更新循环条件
      effect = effect.next;
    } while (effect !== firstEffect);
  }
}
```
