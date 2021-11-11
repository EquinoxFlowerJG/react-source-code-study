# React 源码本地调试环境搭建流程及遇到的问题

weixin_39532019 2021-01-26 17:46:21  
文章标签： build 怎么调试 react

## 版权

版权声明：本文为 CSDN 博主「weixin_39532019」原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接和本声明。  
本文链接：https://blog.csdn.net/weixin_39532019/article/details/113376833  
原文标题：《build 怎么调试 react_GitHub - bozhouyongqi/debug-react: 本地调试 react 源码环境》此处进行了修改

[TOC]

## 正文

工欲善其事，必先利其器。

在学习 react 源码时，如果能够在浏览器中单步调试，势必会加深理解。其实可以借助 webpack 的 resolve.alias 将 react 等指向本地的目录，这样就不会使用 node_modules 中的 react 包。从而可以在本地调试 react 源码。

### 相关的模块版本

- create-react-app：v3.0.1
- react：v16.13.1

_注意：使用不同的版本可能会存在不同的问题，调试时请注意这一点。_

### 一、本地调试 react 源码步骤

#### （一）生成 react 项目

两种方案：

1. 使用 `create-react-app` 脚手架创建 react 项目：

```powershell
npx create-react-app debug-react
```

2. 或者，可以本地安装 react 脚手架（这样可以安装指定版本的脚手架工具），然后再创建项目：

```powershell
# 安装指定的 3.0.1 版本的 create-react-app
yarn (global )add create-react-app@3.0.1

# 下面是使用本地的 create-react-app 创建 react 项目的两种方式，任选一个即可：
npm init react-app debug-react
# 或
yarn create react-app debug-react
```

#### （二）暴露出 webpack 配置

1. 使用 `yarn (run) eject` / `npm run eject` 命令暴露 webpack 配置。

webpack 配置暴露出来后，可以看到会生成 config 目录，里面包括 webpack 的配置文件和一些环境定义。

2. 但是，直接运行 `yarn run start` 会编译失败。出现下述错误。

```powershell
Failed to compile.

./src/index.js

Error: [BABEL] /src/index.js: Cannot find module '@babel/plugin-syntax-jsx' (While processing: "xxx/debug-react/node_modules/babel-preset-react-app/index.js")
```

- （1）报错原因：缺少 `@babel/plugin-syntax-jsx` 这个模块

- （2）解决方法：安装这个模块：`yarn add @babel/plugin-syntax-jsx -D`

3. 之后再运行，就可以顺利启动项目了。

```powershell
Compiled successfully!

You can now view debug-react in the browser.

Local: http://localhost:3000

On Your Network: http://192.168.0.103:3000

Note that the development build is not optimized.

To create a production build, use yarn build.
```

#### （三）清理 src 目录

删除测试以及其他文件，只保留 App 等业务组件。并去除相关引用。

**当然，你也可以不删除任何文件。**

#### （四）clone react 源码至 src 目录下

1. 使用 `git submodule` 命令引入 react 源码作为子模块，以方便后续代码单独管理。

具体实现步骤：

- 进入 src 目录，
- 执行 `git submodule add git@github.com:facebook/react.git`命令。

2. 克隆之后，就可以在 src/react 中正常切换分支，而不影响主工程。我这里使用 `v16.13.1` 版本，因此可以切换至具体的 tag 。

现在我们将 react 版本切换到 v16.13.1 ：

```powershell
git checkout tags/v16.13.1 -b v16.13.1
```

#### （五）修改 webpack.config.js 添加 alias 配置

1. 打开 config/webpack.config.js 文件，注释掉或者删除掉原先的 alias 配置，添加下面的新 alias ，将 react 等指向本地：

```js
alias: {
  'react': path.resolve(\_\_dirname, '../src/react/packages/react'),

  'react-dom': path.resolve(\_\_dirname, '../src/react/packages/react-dom'),

  'shared': path.resolve(\_\_dirname, '../src/react/packages/shared'),

  'react-reconciler': path.resolve(\_\_dirname, '../src/react/packages/react-reconciler'),

  "legacy-events": path.resolve(\_\_dirname, "../src/react/packages/legacy-events"),

  // 'react-events': path.resolve(\_\_dirname, '../src/react/packages/events'),

  // scheduler: path.resolve(\_\_dirname, "../src/react/packages/scheduler"),

},
```

#### （六）修改 ReactFiberHostConfig 文件

1. 新的报错：

修改完 webpack.config.js 的 alias 配置后，重新运行项目，发现会出现如下错误：

```powershell
Failed to compile.

./src/packages/react-reconciler/src/ReactFiberDeprecatedEvents.js

Attempted import error: 'DEPRECATED_mountResponderInstance' is not exported from './ReactFiberHostConfig'.
```

2. 解决报错：修改 ReactFiberHostConfig 文件

打开 `src/packages/react-reconciler/src/ReactFiberHostConfig.js` 文件，修改成如下：

```js
// We expect that our Rollup, Jest, and Flow configurations

// always shim this module with the corresponding host config

// (either provided by a renderer, or a generic shim for npm).

//

// We should never resolve to this file, but it exists to make

// sure that if we _do_ accidentally break the configuration,

// the failure isn't silent.

// invariant(false, 'This module must be shimmed by a specific renderer.');

export * from "./forks/ReactFiberHostConfig.dom";
```

从注释中可以看出，这个文件实际上 react 并不会直接引入，会被打包工具依赖相应的宿主环境替换掉。

#### （七）修改 react 引用方式

1. 新的报错：

修改完 ReactFiberHostConfig.js 文件后，重新启动项目，发现会报如下错误：

```powershell
Failed to compile.

./src/index.js

Attempted import error: 'react' does not contain a default export (imported as 'React').
```

2. 解决报错：修改 react 引用方式

出现了上述错误，我们到源码中查看源码，发现 /debug-react/src/packages/react/index.js 中确实没有默认导出。

但是必须保证业务组件中要引入 React ，因为组件需要用 babel-jsx 插件进行转换(即使用 React.createElement 方法)。

因此可以将全部导出导入到 index.js 和 App.js 文件中，并重命名为需要的名称：

```js
// src/index.js 文件导入部分修改为：
import * as React from "react";
import * as ReactDOM from "react-dom";
```

```js
// src/App.js 文件导入部分修改为或添加以下代码：
import * as React from "react"; // App.js 中如果没有这行代码，后续可能会报错
```

**_或者_**，可以添加一个中间模块文件，来适配该问题：

```js
// adaptation.js

import * as React from "react";

import * as ReactDOM from "react-dom";

export { React, ReactDOM };
```

之后在业务组件中从 adaptation 引入 React 和 ReactDOM。

#### （八）关闭 ESlint 对 `fbjs, prettier` 插件的扩展

1. 新的报错：

修改完 react 的引用方式后，重新启动项目，还是会出现新错误：

```powershell
Failed to compile.

Failed to load config "fbjs" to extend from.

Referenced from: /xxx/baidu/learn/debug-react/src/react/.eslintrc.js
```

2. 解决报错：关闭 ESlint 对 `fbjs, prettier` 插件的扩展

清空 react 源码项目下.eslintrc 文件中的 extends 选项:

```
// .eslintrc

extends: [

'fbjs',

'prettier'

],
```

改为

```
// .eslintrc

extends: [],
```

#### （九）安装 eslint-plugin-no-for-of-loops 插件

1. 新的报错：

解决完 ESLint 的报错后，依然会继续报错:

```powershell
Failed to compile.

Failed to load plugin 'no-for-of-loops' declared in 'src/react/.eslintrc.js': Cannot find module 'eslint-plugin-no-for-of-loops'

Require stack:

- /config/**placeholder**.js
```

2. 解决报错：安装 eslint-plugin-no-for-of-loops 插件

`yarn add eslint-plugin-no-for-of-loops -D`

#### （十）修改 scheduler/index.js 文件

1. 新的报错：

依旧继续报错：

```powershell
Failed to compile.

./src/react/packages/react-reconciler/src/ReactFiberWorkLoop.js

Attempted import error: 'unstable_flushAllWithoutAsserting' is not exported from 'scheduler' (imported as 'Scheduler').
```

2. 解决报错：修改 scheduler/index.js 文件

```js
// /src/react/packages/scheduler/index.js 修改为

"use strict";

export * from "./src/Scheduler";

//添加以下

export {
  unstable_flushAllWithoutAsserting,
  unstable_flushNumberOfYields,
  unstable_flushExpired,
  unstable_clearYields,
  unstable_flushUntilNextPaint,
  unstable_flushAll,
  unstable_yieldValue,
  unstable_advanceTime,
} from "./src/SchedulerHostConfig.js";
```

```js
// react/packages/scheduler/src/SchedulerHostConfig.js 修改为

// throw new Error('This module must be shimmed by a specific build.');

// 添加以下

export {
  unstable_flushAllWithoutAsserting,
  unstable_flushNumberOfYields,
  unstable_flushExpired,
  unstable_clearYields,
  unstable_flushUntilNextPaint,
  unstable_flushAll,
  unstable_yieldValue,
  unstable_advanceTime,
} from "./forks/SchedulerHostConfig.mock.js";

export {
  requestHostCallback,
  requestHostTimeout,
  cancelHostTimeout,
  shouldYieldToHost,
  getCurrentTime,
  forceFrameRate,
  requestPaint,
} from "./forks/SchedulerHostConfig.default.js";
```

#### （十一）解决 react-internal 错误

1. 新的报错：

继续出现以下错误：

```powershell
Failed to compile.

Failed to load plugin 'react-internal' declared in 'src/react/.eslintrc.js': Cannot find module 'eslint-plugin-react-internal'

Require stack:

- /config/**placeholder**.js
```

2. 解决报错：

这是 react 在本地安装的，在源码的 package 中可以看到。但是安装之后依然会报错，这里决定删除这个插件，不进行安装。在 `debug-react/src/react/.eslintrc.js` 中的 plugins 中将其删除。

下面是安装本地 react-internals。笔者这里没有安装，直接不使用该插件。

`yarn add link:./src/react/scripts/eslint-rules -D`

> **_备注：8-11 中的错误都是 react eslint 中的错误，可以试试在 webpack.config.js 中删除 eslint 插件。_**

#### （十二）修复 react/jsx-dev-runtime 报错

1. 新的报错：

删除 react-internals 插件之后，还是报错：

```powershell
Failed to compile.

./src/index.js

Module not found: Can't resolve 'react/jsx-dev-runtime' in '/src'
```

2. 解决报错：

在 webpack-config.js 中可以看到 hasJsxRuntime 变量的取值过程，直接在函数中返回 false.

修改完之后，会报一些 react-internal 有关的错误。

```powershell
Line 1:1: Definition for rule 'react-internal/no-to-warn-dev-within-to-throw' was not found react-internal/no-to-warn-dev-within-to-throw

Line 1:1: Definition for rule 'react-internal/invariant-args' was not found react-internal/invariant-args

Line 1:1: Definition for rule 'react-internal/warning-args' was not found react-internal/warning-args

Line 1:1: Definition for rule 'react-internal/no-production-logging' was not found react-internal/no-production-logging

src/react/packages/react-dom/src/shared/validAriaProperties.js

Line 1:1: Definition for rule 'react-internal/no-primitive-constructors' was not found react-internal/no-primitive-constructors

Line 1:1: Definition for rule 'react-internal/no-to-warn-dev-within-to-throw' was not found react-internal/no-to-warn-dev-within-to-throw

Line 1:1:
```

可以到 /src/react/.eslintrc.js 中将 react-internal 相关的规则都注释掉。

可以看到会继续报如下错误：

```powershell
Failed to compile.

src/react/packages/react-dom/src/client/ReactDOM.js

Line 241:9: Definition for rule 'react-internal/no-production-logging' was not found react-internal/no-production-logging

src/react/packages/react-reconciler/src/ReactFiberHostConfig.js

Line 10:1: Definition for rule 'react-internal/invariant-args' was not found react-internal/invariant-args

Line 12:8: 'invariant' is defined but never used no-unused-vars

src/react/packages/react-reconciler/src/ReactFiberReconciler.js

Line 584:7: Definition for rule 'react-internal/no-production-logging' was not found react-internal/no-production-logging
```

可以到对应的文件中将 eslint 注释删除掉。

_至此，命令行中不会报错误了_。

#### （十三）设置 DefinePlugin 插件

1. 新的问题：

虽然命令行中不会再报错了，但是浏览器中会报错误，提示 **DEV** 没有定义。

2. 解决报错：这个简单了，在 DefinePlugin 中定义就行。

在/config/env.js 中添加:

```js
const stringified = {
  "process.env": Object.keys(raw).reduce((env, key) => {
    env[key] = JSON.stringify(raw[key]);

    return env;
  }, {}),

  "**DEV**": true,

  "**PROFILE**": true,

  "**UMD**": true,

  "**EXPERIMENTAL**": true,
};
```

#### （十四）修改 invariant.js

1. 新的报错：

浏览器中仍然会报错：

```
Uncaught Error: Internal React error: invariant() is meant to be replaced at compile time. There is no runtime version.
```

2. 解决报错：

```js
// /src/react/packages/shared/invariant.js

export default function invariant(condition, format, a, b, c, d, e, f) {
  if (condition) {
    return;
  }

  throw new Error(
    "Internal React error: invariant() is meant to be replaced at compile " +
      "time. There is no runtime version."
  );
}
```

至此，大功告成，终于没有错误了。后面就可以针对源码进行断点调试或者打日志了。
