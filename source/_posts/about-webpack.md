---
title: 写写Webpack插件
date: 2020-08-29 11:51:07
tags:
---

#### Webpack 是什么

本质上，webpack 是一个现代 JavaScript 应用程序的静态模块打包器(module bundler)。当 webpack 处理应用程序时，它会递归地构建一个依赖关系图(dependency graph)，其中包含应用程序需要的每个模块，然后将所有这些模块打包成一个或多个 bundle。

Webpack 有以下四个核心概念：

- 入口(entry)
- 输出(output)
- loader
- 插件(plugins)

我们先简单的看下这四个的具体解释：

<!-- more -->

##### 入口(entry)

入口起点(entry point)指示 webpack 应该使用哪个模块，来作为构建其内部依赖图的开始。进入入口起点后，webpack 会找出有哪些模块和库是入口起点（直接和间接）依赖的。
每个依赖项随即被处理，最后输出到称之为 bundles 的文件中，我们将在下一章节详细讨论这个过程。
可以通过在 webpack 配置中配置 entry 属性，来指定一个入口起点（或多个入口起点）。默认值为 ./src。
接下来我们看一个 entry 配置的最简单例子：
`webpack.config.js;`

```js
module.exports = {
  entry: "./path/to/my/entry/file.js",
};
```

##### 出口(output)

output 属性告诉 webpack 在哪里输出它所创建的 bundles，以及如何命名这些文件，默认值为 ./dist。基本上，整个应用程序结构，都会被编译到你指定的输出路径的文件夹中。你可以通过在配置中指定一个 output 字段，来配置这些处理过程：

```js
const path = require("path");
module.exports = {
  entry: "./path/to/my/entry/file.js",
  output: {
    path: path.resolve(__dirname, "dist"),
    filename: "my-first-webpack.bundle.js",
  },
};
```

在上面的示例中，我们通过 output.filename 和 output.path 属性，来告诉 webpack bundle 的名称，以及我们想要 bundle 生成(emit)到哪里。可能你想要了解在代码最上面导入的 path 模块是什么，它是一个 Node.js 核心模块，用于操作文件路径。

##### loader

loader 让 webpack 能够去处理那些非 JavaScript 文件（webpack 自身只理解 JavaScript）。loader 可以将所有类型的文件转换为 webpack 能够处理的有效模块，然后你就可以利用 webpack 的打包能力，对它们进行处理。

本质上，webpack loader 将所有类型的文件，转换为应用程序的依赖图（和最终的 bundle）可以直接引用的模块。

```js
const config = {
  output: {
    filename: "my-first-webpack.bundle.js",
  },
  module: {
    rules: [{ test: /\.txt$/, use: "raw-loader" }],
  },
};
module.exports = config;
```

以上配置中，对一个单独的 module 对象定义了 rules 属性，里面包含两个必须属性：test 和 use。这告诉 webpack 编译器(compiler) 如下信息：

> “嘿，webpack 编译器，当你碰到「在 require()/import 语句中被解析为 '.txt' 的路径」时，在你对它打包之前，先使用 raw-loader 转换一下。”

##### 插件(plugins)

loader 被用于转换某些类型的模块，而插件则可以用于执行范围更广的任务。插件的范围包括，从打包优化和压缩，一直到重新定义环境中的变量。插件接口功能极其强大，可以用来处理各种各样的任务。

想要使用一个插件，你只需要 require() 它，然后把它添加到 plugins 数组中。多数插件可以通过选项(option)自定义。你也可以在一个配置文件中因为不同目的而多次使用同一个插件，这时需要通过使用 new 操作符来创建它的一个实例。

```js
const HtmlWebpackPlugin = require("html-webpack-plugin"); // 通过 npm 安装
const webpack = require("webpack"); // 用于访问内置插件

const config = {
  module: {
    rules: [{ test: /\.txt$/, use: "raw-loader" }],
  },
  plugins: [new HtmlWebpackPlugin({ template: "./src/index.html" })],
};

module.exports = config;
```

#### 编写一个插件

插件是 webpack 的支柱功能。插件目的在于解决 loader 无法实现的其他事。webpack 插件由以下组成：

- 一个 JavaScript 命名函数或类。
- 在插件函数的 prototype 上定义一个 apply 方法。
- 指定一个绑定到 webpack 自身的事件钩子。
- 处理 webpack 内部实例的特定数据。
- 功能完成后调用 webpack 提供的回调。

```js
class MyExampleWebpackPlugin {
  // 构造函数
  constructor(options) {
    console.log("MyExampleWebpackPlugin", options);
  }
  // 应用函数
  apply(compiler) {
    console.log(compiler);
    // 绑定钩子事件
    compiler.plugin("done", (compilation, callback) => {
      console.log(compilation);
      // 功能完成后调用 webpack 提供的回调。
      callback();
    });
  }
}
```

然后，要安装这个插件，只需要在你的 webpack 配置的 plugin 数组中添加一个实例：

```js
var MyExampleWebpackPlugin = require("./MyExampleWebpackPlugin");
var webpackConfig = {
  // ... 这里是其他配置 ...
  plugins: [new MyExampleWebpackPlugin({ options: true })],
};
```

自己写的插件如下执行：

- webpack 启动后，在读取配置的过程中会先执行 new MyExampleWebpackPlugin() 初始化一个 MyExampleWebpackPlugin 获得其实例。
- 在初始化 compiler 对象后，再调用 MyExampleWebpackPlugin.apply(compiler) 给插件实例传入 compiler 对象。
- 插件实例在获取到 compiler 对象后，就可以通过 compiler.plugin(事件名称, 回调函数) 监听到 Webpack 广播出来的事件。
- 并且可以通过 compiler 对象去操作 webpack。

这时候你会发现，在插件开发过程中最重要的两个资源就是 compiler 和 compilation 对象。所以理解它们的角色是扩展 webpack 引擎重要的第一步。

- **compiler 对象代表了完整的 webpack 环境配置**。这个对象在启动 webpack 时被一次性建立，并配置好所有可操作的设置，包括 options，loader 和 plugin。当在 webpack 环境中应用一个插件时，插件将收到此 compiler 对象的引用。可以使用它来访问 webpack 的主环境。
- **compilation 对象代表了一次资源版本构建**。当运行 webpack 开发环境中间件时，每当检测到一个文件变化，就会创建一个新的 compilation，从而生成一组新的编译资源。一个 compilation 对象表现了当前的模块资源、编译生成资源、变化的文件、以及被跟踪依赖的状态信息。compilation 对象也提供了很多关键时机的回调，以供插件做自定义处理时选择使用。

> Compiler 和 Compilation 的区别在于：Compiler 代表了整个 Webpack 从启动到关闭的生命周期，而 Compilation 只是代表了一次新的编译。

这两个组件是任何 webpack 插件不可或缺的部分（特别是 compilation），因此 webpack 官方非常建议我们去阅读这部分的相关源码：

- [Compiler Source](https://github.com/webpack/webpack/blob/master/lib/Compiler.js)
- [Compilation Source](https://github.com/webpack/webpack/blob/master/lib/Compilation.js)

#### 热更新

前面讲了那么多基础的内容，再额外说说热更新，hot module replacement 简称 hmr，这是个很重要的 dev 环境的插件，可以帮助程序员“自动”的更新代码，而不需要保存完代码之后，再去手动的刷新页面来验证代码是否生效。这个热更新我看的时候有一点比较好奇，就是 webpack 检测到更新的时候，是会主动的打包的，可是打的包放在哪里了呢，查了一下，打包生成的文件，他并不会放在 dist 目录下。而是放到电脑中的内存里面。这样的话，可以有效的提升打包的速度。让我们开发更快，然后客户端 从 websocket 建立连接，然后内存里的内容推送到上面。

```js
const evtSource = new EventSource("//api.example.com/ssedemo.php", {
  withCredentials: true,
});
```

鉴于现在 HotModuleReplacementPlugin 插件已经帮我们做了很多东西，包括 react、vue 等也都内置了很多相关逻辑，所以我们只是简单地引入就可以，但是它背后依然还是有很多方法提供给了我们

如果已经通过 HotModuleReplacementPlugin 启用了模块热替换(Hot Module Replacement)，则它的接口将被暴露在 module.hot 属性下面。通常，用户先要检查这个接口是否可访问，然后再开始使用它。举个例子，你可以这样 accept 一个更新的模块：

```js
if (module.hot) {
  module.hot.accept("./library.js", function () {
    // 使用更新过的 library 模块执行某些操作...
  });
}
```

比如说，我们现在有这样一个场景：

index.js

```js
import counter from "./counter";
import number from "./number";
counter();
number();
```

counter.js

```js
function counter() {
　　 var div = document.createElement('div');
　　 div.setAttribute('id', 'counter');
　　 div.innerHTML = 1;
　　 div.onclick = function() {
　　 div.innerHTML = parseInt(div.innerHTML, 10) + 1;
}
document.body.appendChild(div);

export default counter;
```

number.js

```js
function number() {
  var div = document.createElement("div");
  div.setAttribute("id", "number");
  div.innerHTML = 1000;
  document.body.appendChild(div);
}
export default number;
```

很明显 counter 文件有个计数器，点击后数字会一直+1，而 `number` 文件有个静态的数字 1000。
这个时候运行页面，把 1 增加到 10。这个时候将 number.js 里面写死的 1000 变成 2000。发现，哎呀，我之前点击到的 10，又重新恢复到了 1。说明刷新了页面。之前的一些数据没有保存下来。我希望 `number` 的数据更新，但别去更新我 counter.js 到内容。这时候我们就要改改配置项：

```js
module.exports = {
  devServer: {
    hot: true, // 开启 Hot Module Replacement
    hotOnly: true, // 即便 hmr 的功能没有生效，浏览器也不要自动刷新
  },
  plugins: [new webpack.HotModuleReplacementPlugin()],
};
```

> hot 和 hotOnly 的区别是在某些模块不支持热更新的情况下，前者会自动刷新页面，后者不会刷新页面，而是在控制台输出热更新失败

这时候会发现，页面上的 number 没有改成更新之后的 2000，因为我们还需要再加点代码，就是上面提到的 accept 方法

```js
import counter from "./counter";
import number from "./number";
counter();
number();

if (module.hot) {
  module.hot.accept("./number", () => {
    let removeNode = document.getElementById("number");
    document.body.removeChild(removeNode);
    number();
  });
}
```

当然还有其他方法，以下是些简单的截图，你也可以从官网上自己去查找自己需要的方法，
![img](1.png)
![img](2.png)

#### `Tapable`

插件能够 钩入(hook) 到在每个编译(compilation)中触发的所有关键事件。在编译的每一步，插件都具备完全访问 compiler 对象的能力，如果情况合适，还可以访问当前 compilation 对象。
tapable 这个小型 library 是 webpack 的一个核心工具，但也可用于其他地方，以提供类似的插件接口。webpack 中许多对象扩展自 Tapable 类。这个类暴露 tap, tapAsync 和 tapPromise 方法，可以使用这些方法，注入自定义的构建步骤，这些步骤将在整个编译过程中不同时机触发。

因此，根据你触发到 tap 事件，插件可能会以不同的方式运行。例如，当钩入 compile 阶段时，只能使用同步的 tap 方法：

```js
compiler.hooks.compile.tap("MyPlugin", (params) => {
  console.log("以同步方式触及 compile 钩子。");
});
```

然而，对于能够使用了 AsyncHook(异步钩子) 的 run，我们可以使用 tapAsync 或 tapPromise（以及 tap）：

```js
compiler.hooks.run.tapAsync("MyPlugin", (compiler, callback) => {
  console.log("以异步方式触及 run 钩子。");
  callback();
});

compiler.hooks.run.tapPromise("MyPlugin", (compiler) => {
  return new Promise((resolve) => setTimeout(resolve, 1000)).then(() => {
    console.log("以具有延迟的异步方式触及 run 钩子");
  });
});
```

这些需求(story)的含义在于，可以有多种方式将 hook 钩入到 compiler 中，可以让各种插件都以合适的方式去运行。

#### 总结

相比于上次讲到的 babel 的内部优化代码细节写法，webpack 则是从工程、文件结构、统计的大方向上去优化代码。一样也是一个很强大的工具，充分理解其中的原理知识对我们前端工程师来说，在写业务的时候也会不自觉的向更优的写法靠拢。
最后我想说的是，整个捋完 webpack 的一些方法，就知道了为什么官方非常推荐我们阅读源码来学习使用 webpack，因为他内部使用的就是暴露给我们这些方法，整个生态就是靠着文档里提到的一些基础方法来运作起来的~
![img](3.png)

#### 参考文献

- [Webpack 官方文档](https://www.webpackjs.com)
- [热更新的推送事件流](https://developer.mozilla.org/zh-CN/docs/Server-sent_events/Using_server-sent_events)
- [Compiler Source](https://github.com/webpack/webpack/blob/master/lib/Compiler.js)
- [Compilation Source](https://github.com/webpack/webpack/blob/master/lib/Compilation.js)
