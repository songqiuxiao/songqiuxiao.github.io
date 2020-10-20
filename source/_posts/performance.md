---
title: performance - chrome
date: 2020-10-18 14:24:57
tags:
---

### 面板介绍

先简单看下这个面板，他包含了四个大块。

1. `Controls`。开始记录，停止记录和配置记录期间捕获的信息。
2. `Overview`。 页面性能的高级汇总。更多内容请参见下文。
3. `火焰图`。 CPU 堆叠追踪的可视化。
   您可以在火焰图上看到一到三条垂直的虚线。蓝线代表 DOMContentLoaded 事件。 绿线代表首次绘制的时间。 红线代表 load 事件。
4. `Details`。选择事件后，此窗格会显示与该事件有关的更多信息。 未选择事件时，此窗格会显示选定时间范围的相关信息。
<!-- more -->

![img](1.png)

其中 `Overview` 窗格包含以下三个图表：

- `FPS`。每秒帧数。绿色竖线越高，FPS 越高。 FPS 图表上的红色块表示长时间帧，很可能会出现卡顿。
- `CPU`。 CPU 资源。此面积图指示消耗 CPU 资源的事件类型。
- `NET`。每条彩色横杠表示一种资源。横杠越长，检索资源所需的时间越长。 每个横杠的浅色部分表示等待时间（从请求资源到第一个字节下载完成的时间）。

深色部分表示传输时间（下载第一个和最后一个字节之间的时间）。
横杠按照以下方式进行彩色编码：

- HTML 文件为<span style="color:lightblue">蓝色</span>。
- 脚本为<span style="color:gold">黄色</span>。
- 样式表为<span style="color:purple">紫色</span>。
- 媒体文件为<span style="color:green">绿色</span>。
- 其他资源为<span style="color:gray">灰色</span>。

第 4 块 details 面板，也就是我们最关心的，是浏览器对页面的绘制精确到毫秒级的情况，准确的为我们剖析了浏览器加载渲染页面的全过程。
![img](2.png)
先来分析 Summary 面板，和其字面意思一样，这是总结面板。从宏观层面概括了浏览器加载的总时间，包括:

| 颜色   | 英文      | 中文           |
| ------ | --------- | -------------- |
| 蓝色   | Loading   | 记载           |
| 黄色   | Scripting | 脚本           |
| 紫色   | Rendering | 渲染           |
| 绿色   | Painting  | 绘制           |
| 深灰色 | Other     | 其他           |
| 浅灰色 | 其他      | 未熄火（空闲） |

再来分析 Bottom-Up 面板
Self Time 和 Total Time 以及 Activity，其中的 Self Time 代表函数本身执行消耗时间，Total Time 则是函数本身消耗再加上在调用它的函数中消耗的总时间，Activity 不用解释，就是浏览器活动的意思。
![img](3.png)
值得注意的是，这里的 Group 面板非常有用。我们可以很清晰明了得分析按照活动，目录，域，子域，URL 和 Frame 进行分组的前端性能。对于开发非常有帮助。
![img](4.png)
其实 Bottom-Up 和 Call Tree 都有各自的意思，前者是 The Heavy (Bottom Up) view is available in the Bottom-Up tab，后者是 And the Tree (Top Down) view is available in the Call Tree tab。个人理解的话，前者类似事件冒泡，后者类似事件捕获。
![img](5.png)
![img](6.png)
看一下二者的对比图，很明显可以看出，自上而下的 Call-Tree 更符合我们的人类正常思维，可以更直观地分析浏览器对页面的 build 精确到毫秒级的情况。
就以百度首页进行分析。

#### 绘制阶段

综合视窗，绘制
![img](7.png)

#### 加载阶段

解析样式表，解析 HTML（评估脚本，事件）
![img](8.png)

#### 脚本阶段

DOM GC（DOM 垃圾回收），评估脚本，事件，Major GC（清理年老区（Tenured space）），Minor GC（每次 Minor GC 只会清理年轻代），Run Microtasks（运行微服务），Timer Fired（销毁计时器） ，XMR Load（异步加载对象加载）。
![img](9.png)

#### 渲染阶段

视窗，升级视图树，重新计算样式。
![img](10.png)

#### Event Log

最后说一下 Event Log ，顾名思义就是事件日志的意思，可以很方便的选择想查看的某一阶段的日志。
![img](11.png)

### 浏览器如何渲染网页

关于浏览器的整个渲染阶段，这里单独做了一下研究拓展，首先简单写一段 demo，让我们看看浏览器都做了什么

```html
<!DOCTYPE html>
<html>
  <head>
    <title>The "Click the button" page</title>
    <meta charset="UTF-8" />
    <link rel="stylesheet" href="styles.css" />
  </head>

  <body>
    <h1>Click the button.</h1>

    <button type="button">Click me</button>

    <script>
      var button = document.querySelector("button");
      button.style.fontWeight = "bold";
      button.addEventListener("click", function () {
        alert("Well done.");
      });
    </script>
  </body>
</html>
```

1. 使用 HTML 创建文档对象模型（DOM）
2. 使用 CSS 创建 CSS 对象模型（CSSOM）
3. 基于 DOM 和 CSSOM 执行脚本（Scripts）
4. 合并 DOM 和 CSSOM 形成渲染树（Render Tree）
5. 使用渲染树布局（Layout）所有元素
6. 渲染（Paint）所有元素

![img](12.png)

#### HTML

浏览器从上到下读取标签，把他们分解成节点，从而创建 DOM 。
![img](13.png)

##### HTML 加载优化策略

- 样式在顶部，脚本在底部
  总体思路是尽可能早的加载样式，尽可能晚的加载脚本。原因是脚本执行之前，需要 HTML 和 CSS 解析完成，因此，样式尽可能的往顶部放，当底部脚本开始执行之前，样式有足够的时间完成计算。
  进一步讲讲如何优化
- 最小化和压缩
  方法可用于所有内容，包括 HTML，CSS，JavaScript，图片和其它资源。
  最小化是移除所有多余的字符，包括空格，注释，多余的分号，等等。
  压缩比如 GZip，大大压缩下载文件的大小
  两种方法都用的情况下，资源加载量减少了 80% 到 90%。比如：bootstrap 节省了 87% 的流量。
- 无障碍
  不会提升页面的下载速度，但会大大提升残障人士的满意度。给元素加上 aria 标签，图片提供 alt 文本等。

#### CSS

当浏览器发现任何与节点相关的样式时，比如：外部，内部，或行内样式，立即停止渲染 DOM ，并利用这些节点创建 CSSOM。这就是 CSS “渲染阻塞“ 的由来。这里是不同类型样式的优缺点。

```html
//外部样式

<link rel="stylesheet" href="styles.css" />
// 内部样式
<style>
  h1 {
    font-size: 18px;
  }
</style>
// 行内样式
<button style="background-color: blue;">Click me</button>
```

CSSOM 节点创建与 DOM 节点创建类似，随后，两者合并如下：
![img](14.png)

CSSOM 的构建会阻塞页面的渲染，因此我们想尽早加载样式，

##### CSS 加载优化策略

- 使用 media 属性
  media 属性指定加载样式的条件，比如：符合最大或最小分辨率？还是面向屏幕阅读器？
- 延迟加载 CSS
  有些样式，比如：首屏以下的，或者不那么重要的，可以等待首屏最有价值的内容渲染完成再加载，可以使用脚本等待页面加载，然后再插入样式。
  这有两个栗子：The future of loading CSS，Defer load CSS
- 只加载需要的样式
  使用 uncss 类似的工具，尽量移除不需要的样式。

#### JavaScript

浏览器不断构建 DOM / CSSOM 节点，直到发现外部或者行内的脚本。
由于脚本可能需要访问或操作之前的 HTML 或样式，我们必须等待它们构建完成。
因此浏览器必须停止解析节点，完成构建 CSSOM，执行脚本，然后再继续。这就是 JavaScript 被称作“解析器阻塞”的原因。
脚本只能等到先前的 CSS 节点构建完成。
![img](15.png)

##### JavaScript 加载优化策略

- 异步加载脚本 async
  脚本添加 async 属性，可以通知浏览器不要阻塞其余页面的加载，下载脚本处于较低的优先级。一旦下载完成，就可以执行。
  ![img](16.png)

async 适用于不影响 DOM 或 CSSOM 的脚本，对一些跟我们的代码无关的，不影响用户体验的外部脚本尤其适用，比如：分析统计脚本。

- 延迟加载脚本 defer
  defer 跟 async 非常相似，不会阻塞页面加载，但会等到 HTML 完成解析后再执行。
  ![img](17.png)

不过， async 和 defer 对于行内的脚本不起作用，浏览器默认会编译执行它们。

- 操作之前克隆节点（jq 时代。。。）
  多次操作 DOM 时可以尝试，首先克隆整个 DOM 节点更加高效，操作克隆后的节点，然后替换先前的节点，避免了多次重绘，降低了 CPU 和内存消耗，同时也避免了不必要的页面闪烁。
  需要注意，克隆的时候并没有克隆事件监听。
- Preload/Prefetch/Prerender/Preconnect
  这些新属性并不是所有的浏览器都支持。了解详情可以看这里：Prefetching, preloading, prebrowsing

#### 渲染树（Render Tree）

一旦所有节点已被解析，DOM 和 CSSOM 准备合并，浏览器便会构建渲染树。如果我们把节点想象成单词，那么对象模型就是句子，渲染树便是整个页面。
![img](18.png)

#### 布局（Layout）

布局阶段需要确定页面上所有元素的大小和位置。
![img](19.png)

#### 渲染（Paint）

最终的渲染阶段，会真正地光栅化屏幕上的像素，把页面呈现给用户。
![img](20.png)
整个过程耗时 1 秒或十分之一秒，我们的任务是让它更快。
如果 JavaScript 事件改变了页面的某部分，便会引起渲染树的重绘，并且迫使布局（Layout）和渲染（Paint）过程再次进行。

### 浏览器网络优化策略

- 充分利用 Chrome 开发者工具
  network、performance 面板都帮助我们统计了很多页面加载时的有效信息。这篇文章 值得一读，帮你理解网络资源
- 在优质(4g+)的环境里开发，在较差(3g\2g..)的环境里测试
  开发时大可使用 1Tb SSD，32G 内存的 Macbook Pro ，但是性能测试时还是要到 Chrome 的 network 标签下模拟低带宽的情形，从而获取有价值的信息。
- 合并资源/文件
  基本上，每接收到一个外部 CSS 和 JavaScript 文件，浏览器都会构建 CSSOM，执行脚本。尽管几个文件可以在一次往返中传送，但也浪费了浏览器的宝贵时间和资源，最好还是合并文件，减少不必要的加载。但是合并到一个文件之后，又可能单次请求体积较大导致花费时间较长，所以具体情况还是得具体分析的
- 首屏内容使用内部样式
  内部 CSS 和 JavaScript 不需要请求外部资源，相反，外部资源又可以被缓存，并保持 DOM 轻量，两者没有非黑即白。
  但是一个非常好的论点是首屏关键内容使用内部样式，可以避免请求额外的资源，节省时间做最有意义的渲染。
- 最小化/压缩图片
- 延迟加载图片
- 异步记载字体
- 是否真正需要 JavaScript / CSS?
  原生 HTML 元素可以实现的行为是否用了脚本？是否有样式或者图标可以行内创建的，不需要内部/外部资源？比如：行内 SVG。
- CDN
  可以利用 CDN（内容分发网络）存储资源，它会从离用户最近，延迟最低的位置分发到用户设备，加载时间更快。

### 彩蛋之 分析 youtube 的加载情况

。。。记录了这么多内容，就拿着点鸡毛知识去试着分析一下国外的网站的优化思路，把知识转化为手段。

- **http3**
  http/2 相信还有比较多人已经开始尝试使用了，但 quic 在生产环境使用的并不多见。QUIC（Quick UDP Internet Connection）是 google 自 spdy 后推出的新的协议，从名字就能看到它最大的区别是使用 UDP 传输协议，这可以说是对 http/2 是一个关键的补充
  用最直观的感受来解释就是使用 quic 后网站的性能提升 15%，弱网环境提升 20%，如果你的用户会频繁的 wifi 4g 之间切换或者经常乘坐高铁、地铁高速移动，quic 让他们 ip 变更时不要重新建联，这对视频网站来说是非常有意义的提升。

- **首屏样式控制基本使用了内联样式， 所以 css 资源请求顺序非常靠后**
  在首个请求体积可控的情况下内联必须的 JS 和 CSS 会让你获得更快的首屏时间（根据网上参考建议<200k）
- 。。。

### 参考文献

- [Get Started With Analyzing Runtime Performance](https://developers.google.com/web/tools/chrome-devtools/evaluate-performance)
- [High Performance Browser Networking](https://hpbn.co/)
- [core-web-vitals](https://web.dev/vitals/#core-web-vitals)
