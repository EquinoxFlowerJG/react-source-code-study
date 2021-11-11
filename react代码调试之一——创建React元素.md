# 一、创建 React 元素

JSX 被 Babel 编译为 React.createElement 方法的调用，createElement 方法在调用后返回的就是 ReactElement，就是 VirtualDOM。

React 的源码全部存放于 src/react/packages 目录下。

## （一）第一步：调试创建 React 元素的方法 createElement

因为 createElement 方法挂载在 React 对象上，所以找到 src/react/packages/react 目录。

找到 src/react/packages/react/package.json 文件，这个文件中的`main`选项，指定了 React 包的入口文件。入口文件为同级目录下的 index.js。

找到并打开 src/react/packages/react/index.js，文件，上面部分的代码是关于 flow 的代码，我们不需要考虑。在下方的 export 导出的对象中，我们可以找到 createElement 方法。在最后一行我们可以发现所有的这些包括 createElement 在内的方法都是从 ./src/React.js 中导出的。

然后找到 src/react/packages/react/src/React.js 文件并打开。在 React.js 文件的上部，我们可以找到以下代码：

```JavaScript
// src/react/packages/react/src/React.js (20~26行）
import {
  createElement as createElementProd,
  createFactory as createFactoryProd,
  cloneElement as cloneElementProd,
  isValidElement,
  jsx as jsxProd,
} from './ReactElement';
```

可以发现，createElement 仍然不是在当前文件中定义的，而是来源自同级目录下的 ReactElement.js 文件中。

打开 src/react/packages/react/src/ReactElement.js 文件，找到 createElement 方法，然后进行解析并添加注释：

```js
// src/react/packages/react/src/ReactElement.js
// createElement 方法

// ⭐️ Line 16~21 ⭐️
const RESERVED_PROPS = {
  key: true,
  ref: true,
  __self: true,
  __source: true,
};

// ⭐️ Line 344~434 ⭐️
/**
 * 创建并返回一个新的给定类型的 ReactElement。 // Create and return a new ReactElement of the given type.
 * type          元素类型
 * config        配置属性
 * children      子元素
 * 1. 分离 props 属性和特殊属性
 * 2. 将子元素挂载到 props.children 中
 * 3. 为 props 属性赋默认值（defaultProps）
 * 4. 创建并返回 reactElement
 */
export function createElement(type, config, children) {
  /**
   * propName -> 属性名称
   * 用于后面 2 次的 for...in 循环
   * 一次声明，两次使用，稍微地提升了些许性能
   */
  let propName;

  /**
   * 存储 React Element 中的普通元素属性，即不包含 key ref self source
   */
  const props = {};

  /**
   * 待提取属性（特殊属性：key、ref、self、source）
   * React 内部为了实现某些功能而存在的属性
   */
  let key = null;
  let ref = null;
  let self = null;
  let source = null;

  // config 不为 null，即当前元素有属性
  if (config != null) {
    // 如果 config 对象中有合法的 ref 属性
    if (hasValidRef(config)) {
      // hasValidRef 方法用于判断传入的对象中是否有 ref 属性，返回值为布尔值
      // 将 config.ref 属性提取到 ref 变量中
      ref = config.ref;

      // 如果 __DEV__ 条件成立，那么所处环境为开发环境
      // 在后续的代码分析中，遇到开发环境的代码我们是不需要分析的，因为它不是主逻辑代码，开发环境中执行的代码一般都是一些警告性代码，他会判断你写的代码是否有问题，如果有毛病的话他会给你一些警告性的提示。
      if (__DEV__) {
        // 如果 ref 属性的值被设置成了字符串形式就报一个提示
        // 说明此用法在将来的版本中会被删除
        warnIfStringRefCannotBeAutoConverted(config);
      }
    }
    // 如果在 config 对象中有合法的 key 属性
    if (hasValidKey(config)) {
      // 将 config.key 属性中的值提取到 key 变量中
      key = "" + config.key; // 将传入的非字符串的 key 值转换为字符串
    }

    // 下面两行代码的意思是：如果 config 对象中有 self 或 source 的属性值，就把值提取到同名变量中，如果没有对应的属性，就将同名变量的值设置为 null 。
    self = config.__self === undefined ? null : config.__self;
    source = config.__source === undefined ? null : config.__source;
    // 遍历 config 对象，将剩余的属性添加到一个新的 props 对象中
    for (propName in config) {
      // 如果当前遍历到的属性是对象自身属性
      // 并且在 RESERVED_PROPS 对象中不存在该属性（ RESERVED_PROPS 对象中存储的属性有 key、ref、__self、__source）
      if (
        hasOwnProperty.call(config, propName) &&
        !RESERVED_PROPS.hasOwnProperty(propName)
      ) {
        // 将满足条件的属性添加到 props 对象中（普通属性）
        props[propName] = config[propName];
      }
    }
  }

  /**
   * 将第三个及之后的参数挂载到 props.children 属性中
   * 如果子元素是多个， props.children 是数组
   * 如果子元素是一个， props.children 是对象
   */

  // 由于从第三个参数开始及以后都表示子元素
  // 所以减去前两个参数的结果就是子元素的数量
  const childrenLength = arguments.length - 2;
  // 如果子元素的数量是 1
  if (childrenLength === 1) {
    // 直接将子元素挂载到 props.children 属性上
    // 此时 children 是对象类型
    props.children = children;
    // 如果子元素的属性大于 1
  } else if (childrenLength > 1) {
    // 创建数组，数组中元素的数量等于子元素的数量
    const childArray = Array(childrenLength);
    // 开启循环，循环次数匹配子元素的数量
    for (let i = 0; i < childrenLength; i++) {
      // 将子元素添加到 childArray 数组中
      // i+ 2 的原因是实参集合的前两个参数不是子元素
      childArray[i] = arguments[i + 2];
    }
    // 如果是开发环境
    if (__DEV__) {
      if (Object.freeze) {
        Object.freeze(childArray);
      }
    }
    // 将子元素数组挂载到 props.children 属性中
    props.children = childArray;
  }

  /**
   * 如果当前处理的是组件
   * 看组件身上是否有 defaultProps 属性
   * 这个属性中存储的是 props 对象中属性的默认值
   * 遍历 defaultProps 对象，查看对应的 props 属性的值是否为 undefined
   * 如果为 undefined 就将默认值赋值给对应的 props 属性值
   */

  // 将 type 属性值视为函数，查看其中是否具有 defaultProps 属性
  if (type && type.defaultProps) {
    // 将 type 函数下的 defaultProps 属性值赋值给 defaultProps 变量
    const defaultProps = type.defaultProps;
    // 遍历 defaultProps 对象中的属性，将属性名称赋值给 propName 变量
    for (propName in defaultProps) {
      // 如果 props 对象中的该属性的值为 undefined
      if (props[propName] === undefined) {
        // 将 defaultProps 对象中的对应属性的值赋值给 props 对象中的对应属性的值
        props[propName] = defaultProps[propName];
      }
    }
  }
  // 如果处于开发环境
  if (__DEV__) {
    if (key || ref) {
      const displayName =
        typeof type === "function"
          ? type.displayName || type.name || "Unknown"
          : type;
      if (key) {
        defineKeyPropWarningGetter(props, displayName);
      }
      if (ref) {
        defineRefPropWarningGetter(props, displayName);
      }
    }
  }
  // 返回 ReactElement
  return ReactElement(
    type,
    key,
    ref,
    self,
    source,
    // 在 Virtual DOM 中用于识别自定义组件
    ReactCurrentOwner.current,
    props
  );
}
```

## （二）调试创建用于创建 React 元素 的直接方法 ReactElement

```js
// src/react/packages/react/src/ReactElement.js
// ReactElement 方法

// ⭐️ Line 126~200 ⭐️
/**
 * 接受参数，返回 ReactElement
 *
 * @param {*} type
 * @param {*} props
 * @param {*} key
 * @param {string|object} ref
 * @param {*} owner
 * @param {*} self A *temporary* helper to detect places where `this` is
 * different from the `owner` when React.createElement is called, so that we
 * can warn. We want to get rid of owner and replace string `ref`s with arrow
 * functions, and as long as `this` and owner are the same, there will be no
 * change in behavior.
 * @param {*} source An annotation object (added by a transpiler or otherwise)
 * indicating filename, line number, and/or other information.
 * @internal
 */
const ReactElement = function (type, key, ref, self, source, owner, props) {
  const element = {
    /**
     * 组件的类型，十六进制数值或者 Symbol 值（具体得看浏览器是否支持 Symbol）（是普通组件、函数组件还是类组件）
     * React 在最终渲染 DOM 的时候，需要确保元素的类型是 REACT_ELEMENT_TYPE（不是则无法渲染）
     * 需要此属性作为判断的依据
     */
    $$typeof: REACT_ELEMENT_TYPE,

    /**
     * 元素具体的类型值，如果是元素节点，type 属性中存储的就是 div 、 span 等等
     * 如果元素是组件，type 属性中存储的就是组件的构造函数
     */
    type: type,
    /**
     * 元素的唯一标识
     * 用作内部 vdom 比对，提升 DOM 操作性能
     */
    key: key,
    /**
     * 存储元素 DOM 对象或者组件实例对象
     */
    ref: ref,
    /**
     * 存储向组件内部传递的数据 */
    props: props,

    /**
     * 记录当前元素所属组件（记录当前元素是哪个组件创建的）
     */
    _owner: owner,
  };

  // 如果处于开发环境
  if (__DEV__) {
    // The validation flag is currently mutative. We put it on
    // an external backing store so that we can freeze the whole object.
    // This can be replaced with a WeakMap once they are implemented in
    // commonly used development environments.
    element._store = {};

    // To make comparing ReactElements easier for testing purposes, we make
    // the validation flag non-enumerable (where possible, which should
    // include every environment we run tests in), so the test framework
    // ignores it.
    Object.defineProperty(element._store, "validated", {
      configurable: false,
      enumerable: false,
      writable: true,
      value: false,
    });
    // self and source are DEV only properties.
    Object.defineProperty(element, "_self", {
      configurable: false,
      enumerable: false,
      writable: false,
      value: self,
    });
    // Two elements created in two different places should be considered
    // equal for testing purposes and therefore we hide it from enumeration.
    Object.defineProperty(element, "_source", {
      configurable: false,
      enumerable: false,
      writable: false,
      value: source,
    });
    if (Object.freeze) {
      Object.freeze(element.props);
      Object.freeze(element);
    }
  }
  // 返回 ReactElement
  return element;
};
```

## （三）React 检测开发者是否错误的使用了 props 属性

在其中一个开发环境的判断里

```js
// src/react/packages/react/src/ReactElement.js
// createElement 方法

// ⭐️ Line 344~434 ⭐️
// 其他部分在前面已经讲过了
export function createElement(type, config, children) {
  let propName;

  const props = {};

  let key = null;
  let ref = null;
  let self = null;
  let source = null;

  if (config != null) {
    if (hasValidRef(config)) {
      ref = config.ref;

      if (__DEV__) {
        warnIfStringRefCannotBeAutoConverted(config);
      }
    }

    if (hasValidKey(config)) {
      key = "" + config.key;
    }

    self = config.__self === undefined ? null : config.__self;
    source = config.__source === undefined ? null : config.__source;

    for (propName in config) {
      if (
        hasOwnProperty.call(config, propName) &&
        !RESERVED_PROPS.hasOwnProperty(propName)
      ) {
        props[propName] = config[propName];
      }
    }
  }

  const childrenLength = arguments.length - 2;
  if (childrenLength === 1) {
    props.children = children;
  } else if (childrenLength > 1) {
    const childArray = Array(childrenLength);
    for (let i = 0; i < childrenLength; i++) {
      childArray[i] = arguments[i + 2];
    }
    if (__DEV__) {
      if (Object.freeze) {
        Object.freeze(childArray);
      }
    }
    props.children = childArray;
  }

  if (type && type.defaultProps) {
    const defaultProps = type.defaultProps;
    for (propName in defaultProps) {
      if (props[propName] === undefined) {
        props[propName] = defaultProps[propName];
      }
    }
  }

  // 如果处于开发环境
  // 如果开发者在组件内通过 props 或 ref 对象去获取相对应的值，则 REACT 会在控制台报错，因为这种用法是不正确的用法。
  if (__DEV__) {
    // 元素具有 key 属性或者 ref 属性
    if (key || ref) {
      // 看一下 type 属性中存储的是否是函数，如果是函数就表示当前元素是组件；
      // 如果元素不是组件，就直接返回元素类型字符串。
      // displayName 用于在报错过程中显示是哪一个组件报错了。
      // 如果开发者显式定义了 displayName 属性，就显示开发者定义的；
      // 否则就显示组件名称；如果组件也没有名称，就显示 “unknown”。
      const displayName =
        typeof type === "function"
          ? type.displayName || type.name || "Unknown"
          : type;
      // 如果 key 属性存在
      if (key) {
        // 为 props 对象添加 key 属性
        // 并指定当通过 props 对象获取 key 属性时报错
        defineKeyPropWarningGetter(props, displayName);
      }
      // 如果 ref 属性存在
      if (ref) {
        // 为 props 对象添加 ref 属性
        // 并指定当通过 props 对象获取 ref 属性时报错
        defineRefPropWarningGetter(props, displayName);
      }
    }
  }

  return ReactElement(
    type,
    key,
    ref,
    self,
    source,
    ReactCurrentOwner.current,
    props
  );
}
```

```js
// src/react/packages/react/src/ReactElement.js
// defineKeyPropWarningGetter 方法

// ⭐️ Line 55~75 ⭐️
/**
 * 指定当通过 props 对象获取 key 属性时报错
 *
 * @param {*} props 组件中的 props 对象
 * @param {string} displayName 组件中名称标识
 */
function defineKeyPropWarningGetter(props, displayName) {
  // 通过 props 对象获取 key 属性报错
  const warnAboutAccessingKey = function () {
    // 在开发环境中
    if (__DEV__) {
      // specialPropKeyWarningShown 控制错误只输出一次的变量
      if (!specialPropKeyWarningShown) {
        // 通过 specialPropKeyWarningShown 变量锁住判断条件
        specialPropKeyWarningShown = true;
        // 指定报错信息和组件名称，%s 用来输出一个字符串，这里 %s 就是 displayName
        console.error(
          "%s: `key` is not a prop. Trying to access it will result " +
            "in `undefined` being returned. If you need to access the same " +
            "value within the child component, you should pass it as a different " +
            "prop. (https://fb.me/react-special-props)",
          displayName
        );
      }
    }
  };
  warnAboutAccessingKey.isReactWarning = true;
  // 为 props 属性添加 key 属性
  Object.defineProperty(props, "key", {
    // 当获取 key 属性时调用 warnAboutAccessingKey 方法进行报错
    get: warnAboutAccessingKey,
    configurable: true,
  });
}
```

```js
// src/react/packages/react/src/ReactElement.js
// defineRefPropWarningGetter 方法

/**
 * 指定当通过 props 对象获取 ref 属性时报错
 *
 * @param {*} props 组件中的 props 对象
 * @param {*} displayName 组件中名称标识
 */
function defineRefPropWarningGetter(props, displayName) {
  // 通过 props 对象获ref 属性报错
  const warnAboutAccessingRef = function () {
    // 在开发环境中
    if (__DEV__) {
      // specialPropKeyWarningShown 控制错误只输出一次的变量
      if (!specialPropRefWarningShown) {
        // 通过 specialPropKeyWarningShown 变量锁住判断条件
        specialPropRefWarningShown = true;
        // 指定报错信息和组件名称，%s 用来输出一个字符串，这里 %s 就是 displayName
        console.error(
          "%s: `ref` is not a prop. Trying to access it will result " +
            "in `undefined` being returned. If you need to access the same " +
            "value within the child component, you should pass it as a different " +
            "prop. (https://fb.me/react-special-props)",
          displayName
        );
      }
    }
  };
  warnAboutAccessingRef.isReactWarning = true;
  // 为 props 属性添加 ref 属性
  Object.defineProperty(props, "ref", {
    // 当获取 ref 属性时调用 warnAboutAccessingRef 方法进行报错
    get: warnAboutAccessingRef,
    configurable: true,
  });
}
```

## （四）isValidElement 方法的内部实现

```js
// src/react/packages/react/src/ReactElement.js
// isValidElement 方法

// ⭐️ Line 10 ⭐️
import { REACT_ELEMENT_TYPE } from "shared/ReactSymbols";

// ⭐️ Line 540~553 ⭐️
/**
 * 验证 object 参数是否是 ReactElement ，返回布尔值
 * 验证成功的条件：
 * object 是对象
 * object 不为 null
 * object 对象中的 $$typeof 属性值为 REACT_ELEMENT_TYPE
 */
export function isValidElement(object) {
  return (
    typeof object === "object" &&
    object !== null &&
    object.$$typeof === REACT_ELEMENT_TYPE
  );
}
```

REACT_ELEMENT_TYPE：

```js
// src/react/packages/shared/ReactSymbols.js
// 对 REACT_ELEMENT_TYPE 属性的定义

// ⭐️ Line 12~16 ⭐️
const hasSymbol = typeof Symbol === "function" && Symbol.for;

export const REACT_ELEMENT_TYPE = hasSymbol
  ? Symbol.for("react.element")
  : 0xeac7;
```
