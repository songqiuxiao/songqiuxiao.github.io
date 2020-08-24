---
title: 写写Babel插件
date: 2020-08-22 15:38:53
tags:
---

#### Babel 是什么

Babel 是一个工具链，主要用于将 ECMAScript 2015+ 版本的代码转换为向后兼容的 JavaScript 语法，以便能够运行在当前和旧版本的浏览器或其他环境中。比如：

- 语法转换
- 通过 Polyfill 方式在目标环境中添加缺失的特性 (通过 @babel/polyfill 模块)
- 源码转换 (codemods)
- ...

然而作为一款功能强大的 JavaScript 编译器，Babel 值得我们探索的远不止于此。这篇博客就以 Babel 的核心之一插件为切入点，探寻在开发实践中除了转译 ES6，Babel 插件到底有什么用？

<!-- more -->

#### 关于 Babel 插件

Babel 编译代码的过程可分为三个阶段：解析（parsing）、转换（transforming）、生成（generating），这三个阶段分别由 `@babel/parser`、`@babel/core`、`@babel/generator` 执行。
所以说 Babel 本质上只是一个代码的搬运工，如果不给 Babel 装上插件，它将会把输入的代码原封不动地输出。正是因为有插件的存在， Babel 才能将输入的代码进行转变，从而生成新的代码。

Babel 插件大致分为两种：语法插件（syntax plugin）和 转换插件（transform plugin）：语法插件作用于 `@babel/parser`，负责将代码解析为抽象语法树；转换插件作用于 `@babel/core`，负责转换 AST 的形态。

语法插件虽然名字叫插件，但是其实不具备功能性。语法插件所对应的语法功能其实都已在 `@babel/parser` 里实现，插件的作用只是将对应语法的解析功能打开。所以下文提及的 Babel 插件将专指转换插件。
![img](1.png)
想象我们现在需要借助 Babel 来帮我们转换一些代码写法，那么我们第一件事是什么？对的，是需要先拿到整个 AST，然后对 AST 内满足一系列条件的 node 来做相应的逻辑转换，[https://astexplorer.net](https://astexplorer.net) 这个网站可以帮我们在线生成出 AST，当然你也可以使用 `@babel/core` 来生成语法树 得到的结果是一致的。
既然我们要先得到 AST 才能在此基础上进行插件开发，那么 Babel 插件就负责编译过程中的一个核心任务：转换 AST。AST 是一个有着多层嵌套的树状结构，理论上讲写一个插件改变 AST 并不是什么难事。但想要快速便捷得完成插件的开发，则需要借助以下几样工具。

##### traverse

`@babel/traverse` 是一款用来自动遍历抽象语法树的工具，它会访问树中的所有节点，在进入每个节点时触发 enter 钩子函数，退出每个节点时触发 exit 钩子函数。开发者可在钩子函数中对 AST 进行修改。

```js
import traverse from "@babel/traverse";

traverse(ast, {
  enter(path) {
    // 进入 path 后触发
  },
  exit(path) {
    // 退出 path 前触发
  },
});
```

下面这个图就会比较直观的反应出 `enter` 和 `exit` 的时机
![img](2.png)

##### types

`@babel/types` 是一款作用于 AST 的类 Lodash 库，其封装了大量与 AST 有关的方法，大大降低了转换 AST 的成本。`@babel/types` 的功能主要有两种：一方面可以用它验证 AST 节点的类型，例如使用 isClassMethod 或 assertClassMethod 方法可以判断 AST 节点是否为 class 中的一个 method；另一方面可以用它构建 AST 节点，例如调用 classMethod 方法，可生成一个新的 classMethod 类型 AST 节点；另外还可以用于变换 AST 节点。

```js
import traverse from "@babel/traverse";
import * as t from "@babe/types";

traverse(ast, {
  enter(path) {
    // 进入 path 后触发
    if (t.isIdentifier(path.node, { name: "n" })) {
      path.node.name = "x";
    }
  },
});
```

##### template

`@babel/template` 是另一个虽然很小但却非常有用的模块。它能让你编写字符串形式且带有占位符的代码来代替手动编码， 尤其是生成的大规模 AST 的时候。 实现了计算机科学中一种被称为准引用（quasiquotes）的概念。说白了，它能直接将字符串代码片段转换为 AST 节点。例如下面的例子中，`@babel/template` 可以将一段引入 axios 的声明直接转变为 AST 节点。

```js
import template from "@babel/template";
import generate from "@babel/generator";
import * as t from "@babel/types";

const buildRequire = template(`
  var IMPORT_NAME = require(SOURCE);
`);
const ast = buildRequire({
  IMPORT_NAME: t.identifier("myModule"),
  SOURCE: t.stringLiteral("my-module"),
});
console.log(generate(ast).code); // var myModule = require("my-module");
```

#### Babel 的用处

既然 Babel 插件有着如此丰富的功能，那我们当然不能只满足于用 Babel 转译 ES6 啦~ 其实在开发实践的许多场景中，借助 Babel 插件能够自由转换代码的优势，我们可以在编译代码后大大优化代码的质量，并提高开发效率。

接下来，我们就分别从扩展既有方法、提前执行运行时代码、提高代码性能等三个角度来探索如何在实践中高效利用 Babel 插件。

##### 扩展既有方法

在开发过程中时，我们经常需要用 `console.log` 打印出各种各样的文案。打印文案会更加便于监测程序的执行，但当整个程序中 console.log 较多且散落在各个文件中时，开发者可能很难快速找出屏幕上的文案是由哪个文件里的那一行代码打印的。 想要快速定位到 console.log 被调用的位置，较为粗暴的方式是使用 `console.trace`，console.trace 会把 trace 路径在屏幕上一并打印出来。但 console.trace 显然不适合在生产环境使用，在生产环境使用之将极大地损伤打印内容的可读性。要想让开发环境的 log 显示出 trace 信息而生产环境的不显示，只要在开发环境代码的编译过程中用 Babel 插件为 console.log 添加 trace 功能即可。

```js
// test.js
console.log("something need to log");
```

执行上面的 test.js 文件后，毫无疑问，屏幕上只会出现一句孤零零的文案。要想加上 trace 信息，我们得先把这句代码进行解剖分析，看看如何才能将其改头换面

首先使用 `@babel/parser` 将代码解析成如下 AST
![img](3.png)
从 AST 中我们可以看出，console.log(‘something need to log’) 这行代码是一个 MemberExpression 节点，它由 object、property、arguments 等三个子节点组成。展开 arguments，可以发现它包含着一个 value 为 「something need to log?」 的 Literal。所以只需将位置信息插入到 value 当中即可在 log 的时候显示出 trace 信息。

```js
// plugin.js
import traverse from "@babel/traverse";
import t from "@babel/types";

traverse(ast, {
  enter(path) {
    // 你还可以添加满足console.log参数节点的其他条件
    if (t.isLiteral(path.node)) {
      const location = `line ${path.node.loc.start.line}, column ${path.node.loc.start.column}`;
      path.node.value = `${path.node.value} (trace: ${location})`;
    }
  },
});
```

上面的代码对 AST 进行了遍历，当访问到 console.log 参数所在的 Literal 节点时，先将该节点的位置信息取出，然后将位置信息插入到参数的 value 当中去。用此插件对代码进行编译后，console.log 的功能将得到扩展：不仅能够输出 log 方法的参数值，且能将 console.log 参数在源文件中的位置一并输出。

##### 提前执行运行时代码

```js
let results = [
  { country: "China", capital: "Beijing" },
  { country: "Japan", capital: "Tokyo" },
  { country: "Russia", capital: "Moscow" },
  { country: "France", capital: "Paris" },
].map((res) => "The capital of" + res.country + "is" + res.capital);
```

上面这段代码通过 map 方法处理了一个由对象元素组成的静态数组，生成了一个由字符串元素组成的数组。由于这段代码中没有动态变量，所以放到任何一个用户的浏览器里去执行，都会生成同样的结果。在浏览器或其他客户端的运行时环境里执行这段代码，无疑是一种不必要的消耗。但如果开发者在代码中直接将变量 result 写成一个由字符串组成的数组，会大大降低开发的便捷性。既不想在运行时执行，又不愿意在开发时写死，那只有借助 Babel 在编译时去执行这段代码了。

为了让赋值语句的右值能够在编译时被预处理，我们可以在 Array 的 map 方法外面套一个用来标记用的 calc 方法，以此来告知 Babel 需要在编译时执行这段代码。

```js
let results = calc(`[
  { country: "China", capital: "Beijing" },
  { country: "Japan", capital: "Tokyo" },
  { country: "Russia", capital: "Moscow" },
  { country: "France", capital: "Paris" },
].map((res) => "The capital of" + res.country + "is" + res.capital)`);
```

使用 @babel/parser 对上述方法进行处理，会得到如下 AST
![img](4.png)
从 AST 中可以看出，整段赋值的代码是一个 VariableDeclarator，等号的左侧是一个 name 为 result 的 Identifier，右侧是一个 CallExpression。
观察 CallExpression 的 arguments，可以发现 value 里以字符串的形式完整记录了对数组进行 map 操作的代码。顿时局势变得明朗起来：只需要计算出 map 方法的结果，并用该结果替换等号右侧的 CallExpression 即可。

```js
// plugin.js
import traverse from "@babel/traverse";
import t from "@babel/types";
import template from "@babel/template";

let rawCode;

traverse(ast, {
  enter(path) {
    if (t.isTemplateElement(path.node)) {
      rawCode = path.node.value.raw;
    }
    // 这里只对当前条件做了判断，实际场景需要替换条件
    if (
      t.isVariableDeclarator(path.node) &&
      t.isIdentifier(path.node, { name: "result" })
    ) {
      path.node.init.needReplaced = true;
    }
    if (path.node.needReplaced) {
      const buildRequire = template(`RAW_CODE`);
      const builtAST = buildRequire({
        RAW_CODE: eval(rawCode),
      });
      path.replaceWith(builtAST);
    }
  },
});
```

上面的插件代码中，先通过遍历 AST 找到 `TemplateElement` 节点，从 `TemplateElement` 节点中取出字符串格式的 `map` 方法代码。接下来在访问到 `VariableDeclarator` 节点的时候，使用 `eval` 方法计算出字符串代码的结果，最后用 `@babel/template` 将计算出的数组转为 AST 节点，替换赋值语句等号右侧的 `CallExpression`。至此，一个原本需要在运行时执行的 `map` 方法已在编译时提前计算出了结果。

##### 提高代码性能

在程序的开发过程中，代码的高性能和开发的便捷性一直是一对难以共存的矛盾体。例如要对一个数组进行遍历，有 `for`、`forEach`、`map` 等许多方式可供选择。若选择了 `for` 循环，将无法体验 `forEach`、`map` 等 Array 方法的便捷功能；若选择了 Array 方法，将面临更高的性能开支（因为 Array 方法除了循环以外还需要执行其他许多任务，如考虑上下文、考虑稀疏数组、生成新数组等，其性能注定无法超越 for 循环）。

```js
arr.forEach((e) => console.log(e));
```

上面是一个简单的 forEach 方法，想要提高性能，我们必然会想到将代码写成这样：

```js
for (let i = 0; i < arr.length; i++) {
  console.log(arr[i]);
}
```

为了既保证代码的高性能，又保留开发的便捷性，可以在编译时用 Babel 插件将 `forEach` 转换为 `for` 循环。由于 `forEach` 箭头函数 body 中的内容与 for 循环 body 中的内容大致相同，所以在转换 AST 时，只需将 `forEach` 箭头函数的 body 节点移植到 `for` 循环的 body 节点并修改一些变量名即可。鉴于 `forEach` 转换成 `for` 循环的过程中，需要考虑的特殊情况较多，在此就不详细描述转换过程了。如果想在开发实践中将代码中的 Array 方法全部替换成 `for` 循环以提高性能，可以使用现成的 Babel 插件 `faster.js`。

#### 总结

前面总结了一系列的 Babel 应用，可见使用场景是非常非常多的，纵观整个 Babel 生态，当然还有非常多的事情可做，例如修改 `@babel/parser` 可以给 JavaScript 添加自定义语法，换一个 `generator` 可以将 JavaScript 编译成另外某种语言等等。
大家不妨自己动手试一下，对于理解整个 Babel 是很有帮助的。
不过 Babel 插件也不是越多越好的，因为插件之间会有互相依赖关系，为了保证性能，整个解析 AST 的过程只会有一次，所以当出现了 A 插件依赖 B 插件，B 依赖 C，而 C 又依赖回 A 插件的情况时，会导致后执行的插件拿不到想要的解析结果，从而导致转码失败（我司最近就有个 `lingui` 的国际化插件出现了这个问题）。所以要擦亮眼睛选择最合适的就好~

#### 参考文献

- [Babel 官方文档](https://babeljs.io/)
- https://github.com/jamiebuilds/babel-handbook/blob/master/translations/en/user-handbook.md
- https://github.com/jamiebuilds/babel-handbook/blob/master/translations/en/plugin-handbook.md
- [Babel 的 plugins 和 preset 的区别](https://juejin.im/entry/6844903616554221576)
