---
title: 前端生成图片
date: 2021-04-25 11:33:55
tags:
---

前几天在面试的时候，和一个候选人聊到了生成海报的一些方案。想到我们的业务里也有类似的场景，就想来看看这种把 dom 生成图片都有哪些个方案，优缺点都是啥。能不能自己实现一个简单的方案，毕竟引入三方库可能会多一些可能会无用的代码，自己造轮子才是最精准解决需求的。

<!-- more -->

# html2canvas

首先是 [`html2canvas`](https://github.com/niklasvh/html2canvas)，我们项目里是有用过这个的，具体可以点击[这里](https://html2canvas.hertzen.com/)来体验。
**👍🏻 它的优点是：**

1. 兼容性好，这是官网给出的 supported
   ![img](1.jpg)
2. star 多、且一直有人在维护
   ![img](2.jpg)

**👎🏻 缺点是：**

1. 生成的图片样式基本上无法调整，有点类似截屏，不能添加一些额外的样式进去，比如边框等。
   具体可以参考官方给出的[feature 文档](https://html2canvas.hertzen.com/features)。
   ![img](3.jpg)
2. 没办法渲染跨域资源

# dom-to-img

再搜搜看的话，还有[`dom-to-img`](https://github.com/tsayen/dom-to-image)，它的 star 数会稍微少一些。
**👍🏻 优点是：**

1. 支持生成多种图片格式，png、base64、SVG 等格式
2. 支持很多样式，方便高还原设计稿的一些细节
   ![img](6.jpg)

**👎🏻 缺点是：**

1. 兼容性较差、只在高版本浏览器里支持。
   主要是由于 `<foreignObject>` 支持度的问题。感觉作者也并没有打算优化这一点的意思。
   ![img](4.png)
2. 维护的频率已经很低了
   ![img](5.jpg)
3. 翻看 issue 的话，会有提到某些机型不支持的情况。<small>（我这边就没做具体测试，可以根据 issue 来解决）</small>
4. 没办法渲染跨域资源

# puppeteer - screenshot

[puppeteer](https://github.com/puppeteer/puppeteer)也是我们项目里用到的。它不只是一个把 html 转成图片的工具，它是 Chrome 开发团队在 2017 年发布的一个 Node.js 的包，可以用来模拟 Chrome 浏览器的运行环境。

> Puppeteer is a Node library which provides a high-level API to control Chrome or Chromium over the DevTools Protocol. Puppeteer runs headless by default, but can be configured to run full (non-headless) Chrome or Chromium.

![img](7.jpg)

既然是在模拟 Chrome 浏览器的运行环境，那么当然能用它提供的打开页面 url + 截图的方式，这里给出示例代码：

```js
const puppeteer = require("puppeteer");
// steam 启动！（不是
let browser = puppeteer.launch(option); // option 内有一些浏览器的配置参数
// 创建一个tab页面
const page = browser.newPage();
// 模拟 iPhone X 设备
page.emulate(devices["iPhone X"]);
// 打开你需要截屏的链接
page.goto(url);
// 页面回传信息
page.evaluate((data) => {
  // 在这里你可以模拟操作 dom
  // 比如选中什么元素加点边框、背景色啥的
});
//塞点样式
page.addStyleTag(options);
// 截个屏
const file = page.screenshot({
  type: "jpeg",
  quality: 1000,
});
// 别忘了关掉页面
page.close();
```

当然 puppeteer 还支持其他 api 使用，这一就不一一列举，感兴趣的可以去[仓库](https://github.com/puppeteer/puppeteer/blob/v9.0.0/docs/api.md)里看一下。
介绍完使用方法，还是要总结下优缺点的。
**👍🏻 优点是：**

依托于 chrome 环境，你能想到的、浏览器支持的、几乎任何操作都能实现。

1. 兼容性好
2. 支持各种生成方式，图片、pdf、任何形式
3. 允许操作 dom，可以任性修改具体的展示，想改点样式，想改下排版。通通 OK！满足设计师和产品的所有任性要求

**👎🏻 缺点是：**

1. 大。我试了下，大概 112.4 Mb ，压缩之后可能会小一些，但估计没什么项目会允许你因为一个 html 转图片功能，让你装这么大的一个库
   ![img](8.jpg)
2. 使用复杂。
   需要先看明白 puppeteer 的作用。杀鸡用牛刀的感觉，比如看 demo，进行了很多操作，最后才是：`page.screenshot`

# 实现原理

看了这么多，各种优缺点也都分析到位了，那么它的实现原理是什么呢？我们能不能自己去实现一个类似的东西。毕竟三方库里可能会提供很多无用工具，自己写的话，可以精简很多内容（emm...）。
刨除去 puppeteer 的实现，简单看了下另外两个库的源码，其实也就是 canvas 和 svg 两种实现方案，原理大概是差不多的，就是把 dom 转成图片，不过实现上是完全不一样的，下面就分开来说。

## canvas 方案

嗯，怎么把 dom 放到 canvas 里呢？肯定是要一点点画到画布上的，光想一想就觉得挺麻烦的，<small><del>此时想起了我上大学的课设，canvas 画小飞机= =</del></small>
大致的思路就是：

1. 递归出需要转成图片的所有 dom 节点，然后放到相应的 `renderList`，dom 里放一些类似 `isRoot`、`style`、`nodeType` 等一系列需要用到的绘制信息
2. 根据记录的 `position`、`z-index`、`float` 等能脱离文档流的 css 属性，把刚刚的 `renderList`，根据前面的层级信息，生成一个 `renderOrderList`
3. 遍历 `renderOrderList`，把 css 样式转换成 `setFillStyle` 可识别的参数，根据 `nodeType` 调用相对应 canvas 方法，如文本则调用 `fillText`，图片 `drawImage`，设置背景色的 div 调用 `fillRect` 等。canvas 具体的 api 可以参考[这里](https://developer.mozilla.org/zh-CN/docs/Web/API/Canvas_API)
4. 创建一个画布，根据刚刚转换好的 `renderOrderList` 画图

看着这几个整体思路，无疑第 2、3 步会是非常麻烦的工作，尤其在实际业务场景下，dom 的排版会非常非常的复杂，html2canvas 的大部分逻辑其实也是在做这个事情。这也是为什么我说它灵活性不够的原因，canvas 的样式不像 css 那么灵活，你想加的样式，并不一定能够支持；但是用 canvas 的兼容性又足够好，只要保证浏览器支持 canvas 就行；由于实际场景下 dom 的复杂度，而整体逻辑又使用了递归，你懂的，长页面几千个 node 节点根本拦不住，所以性能很容易跟不上，html2canvas 内使用了大量的 Promise 来解决这个问题，提升了绘制速度。

另外相比于 css，canvas 对文字排版的支持很弱，在 css 中天然支持的文本自动换行，其他 `letter-spacing` 字间距，`writing-mode` 竖排等都是一个 css 属性就可以实现。但是在 canvas 中，全部都不支持。要实现这种换行效果，就只能对着文本去逐个计算，然后一行一行甚至一个字一个字渲染。。。。。

不过尽管 html2canvas 主页表示它还处于实验室环境，不推荐大家在生产使用，但自 14 年起便已经被 Twitter 等用在了生产环境，所以虽然有诸多限制，稳定性应该还是保障的。
![img](9.jpg)
canvas 的方案看起来如此复杂，根本不想自己手撸一套啊喂！那么有没有一种更简单的方法呢？铛铛铛铛~~那就是 svg 啦

## svg 方案

首先 svg 是矢量图，并且大家也都知道，它是可以用 xml 来描述的。

比如说下面这个可爱的胡萝北~就是用 svg 画的
<svg t="1619407493821" class="icon" viewBox="0 0 1024 1024" version="1.1" xmlns="http://www.w3.org/2000/svg" p-id="16419" width="200" height="200"><path d="M600.8 129.1c34.4-1.9 63.6 24.5 65.5 58.8l8.8 165.7c1.9 34.4-24.5 63.6-58.8 65.5-34.4 1.9-63.6-24.5-65.5-58.8L542 194.6c-1.7-34.4 24.6-63.8 58.8-65.5z" fill="#91CE55" p-id="16420"></path><path d="M616.9 288.3l110.9-123.5c23-25.5 62.3-27.7 87.8-4.7s27.7 62.3 4.7 87.8l-111 123.5c-23 25.5-62.3 27.7-87.8 4.7s-27.6-62.3-4.6-87.8z" fill="#A4CE4E" p-id="16421"></path><path d="M626.3 304.2l165.7-8.8c34.4-1.9 63.6 24.5 65.5 58.8 1.9 34.4-24.5 63.6-58.8 65.5L633 428.5c-34.4 1.9-63.6-24.5-65.5-58.8-1.8-34.4 24.5-63.6 58.8-65.5z" fill="#A1D72B" p-id="16422"></path><path d="M697.1 279.9c79.7 71.6 89.6 193 22.6 276.7-2.3 3.1-4.8 6-7.4 8.8-5.5 6.3-11.5 12.2-17.9 17.5-104.8 107.2-224.2 198.9-354.6 272.5l-5.3 2.9c-46.1 24.5-102.7 17.6-141.6-17.2s-51.9-90.4-32.3-138.9c-0.1-1.9 1.6-3.9 2.4-5.6 59.4-137.6 137.9-266.2 233-382 4.7-6.8 9.9-13.4 15.5-19.5 2.7-2.9 5.3-5.6 8.2-8.2 75.9-75.5 197.7-78.5 277.4-7z" fill="#FF4601" p-id="16423"></path><path d="M598.3 233c90.6 81 102.3 218.7 26.5 313.6-2.8 3.6-5.6 6.8-8.3 9.9-6.5 7.2-13.6 13.9-21.3 19.9a1811.3 1811.3 0 0 1-365 287.3c-30.7-12.2-55.3-36.1-68.3-66.4-13.1-30.3-13.5-64.7-1.2-95.3-0.1-1.9 1.6-3.9 2.4-5.6 10.3-23.4 20.9-46.8 32.3-69.9 9.9-20.6 20.5-41 31.5-61.4 11.1-20.3 23.5-41.7 37.2-64.3 11.8-19.2 23.9-38.2 36.6-56.9 13.9-20.9 28.1-41.2 42.2-61 13.1-18 26.9-35.8 40.9-53.1 3.9-5.2 8-10.3 12.2-15.2 4.7-6.8 9.9-13.4 15.5-19.5 2.7-2.9 5.3-5.6 8.2-8.2 46.1-46.2 112-67.2 176.4-56.1l2.2 2.2z" fill="#F86F1F" p-id="16424"></path><path d="M226.3 564.4l64.4 57.9c11.2 7.1 17.1 20.2 15.1 33.3s-11.6 23.8-24.5 27.1c-12.8 3.5-26.5-1.1-34.8-11.4l-52-45.7c10.3-20.4 20.7-40.8 31.8-61.2zM299.8 447.5l107.3 96.4c10.3 12.8 9.8 31-1.2 43.3-11 12.2-29.1 14.7-42.9 5.9l-99.8-88.5c11.6-19.4 23.9-38.4 36.6-57.1zM382.3 333.2l83 74.5c10.3 12.8 9.8 31-1.2 43.3-11 12.2-29.1 14.7-42.9 5.9L342 385.7c12.6-17.3 26.1-35.3 40.3-52.5z" fill="#F4C1A8" p-id="16425"></path><path d="M573.3 240.2c-21.9-2-44 2-63.8 11.8-6.7 3.3-2.1 14 4.7 10.7 18-8.8 37.8-12.7 57.9-10.8 7.5 0.6 8.7-11.1 1.2-11.7zM487.3 266.1c-4.7-5.9-13.9 1.5-9.1 7.4 4.7 5.8 13.8-1.5 9.1-7.4z" fill="#FFFFFF" p-id="16426"></path></svg>
代码是下面酱紫的：

```html
<svg
  t="1619407493821"
  class="icon"
  viewBox="0 0 1024 1024"
  version="1.1"
  xmlns="http://www.w3.org/2000/svg"
  p-id="16419"
  width="200"
  height="200"
>
  <path
    d="M600.8 129.1c34.4-1.9 63.6 24.5 65.5 58.8l8.8 165.7c1.9 34.4-24.5 63.6-58.8 65.5-34.4 1.9-63.6-24.5-65.5-58.8L542 194.6c-1.7-34.4 24.6-63.8 58.8-65.5z"
    fill="#91CE55"
    p-id="16420"
  ></path>
  <path
    d="M616.9 288.3l110.9-123.5c23-25.5 62.3-27.7 87.8-4.7s27.7 62.3 4.7 87.8l-111 123.5c-23 25.5-62.3 27.7-87.8 4.7s-27.6-62.3-4.6-87.8z"
    fill="#A4CE4E"
    p-id="16421"
  ></path>
  <path
    d="M626.3 304.2l165.7-8.8c34.4-1.9 63.6 24.5 65.5 58.8 1.9 34.4-24.5 63.6-58.8 65.5L633 428.5c-34.4 1.9-63.6-24.5-65.5-58.8-1.8-34.4 24.5-63.6 58.8-65.5z"
    fill="#A1D72B"
    p-id="16422"
  ></path>
  <path
    d="M697.1 279.9c79.7 71.6 89.6 193 22.6 276.7-2.3 3.1-4.8 6-7.4 8.8-5.5 6.3-11.5 12.2-17.9 17.5-104.8 107.2-224.2 198.9-354.6 272.5l-5.3 2.9c-46.1 24.5-102.7 17.6-141.6-17.2s-51.9-90.4-32.3-138.9c-0.1-1.9 1.6-3.9 2.4-5.6 59.4-137.6 137.9-266.2 233-382 4.7-6.8 9.9-13.4 15.5-19.5 2.7-2.9 5.3-5.6 8.2-8.2 75.9-75.5 197.7-78.5 277.4-7z"
    fill="#FF4601"
    p-id="16423"
  ></path>
  <path
    d="M598.3 233c90.6 81 102.3 218.7 26.5 313.6-2.8 3.6-5.6 6.8-8.3 9.9-6.5 7.2-13.6 13.9-21.3 19.9a1811.3 1811.3 0 0 1-365 287.3c-30.7-12.2-55.3-36.1-68.3-66.4-13.1-30.3-13.5-64.7-1.2-95.3-0.1-1.9 1.6-3.9 2.4-5.6 10.3-23.4 20.9-46.8 32.3-69.9 9.9-20.6 20.5-41 31.5-61.4 11.1-20.3 23.5-41.7 37.2-64.3 11.8-19.2 23.9-38.2 36.6-56.9 13.9-20.9 28.1-41.2 42.2-61 13.1-18 26.9-35.8 40.9-53.1 3.9-5.2 8-10.3 12.2-15.2 4.7-6.8 9.9-13.4 15.5-19.5 2.7-2.9 5.3-5.6 8.2-8.2 46.1-46.2 112-67.2 176.4-56.1l2.2 2.2z"
    fill="#F86F1F"
    p-id="16424"
  ></path>
  <path
    d="M226.3 564.4l64.4 57.9c11.2 7.1 17.1 20.2 15.1 33.3s-11.6 23.8-24.5 27.1c-12.8 3.5-26.5-1.1-34.8-11.4l-52-45.7c10.3-20.4 20.7-40.8 31.8-61.2zM299.8 447.5l107.3 96.4c10.3 12.8 9.8 31-1.2 43.3-11 12.2-29.1 14.7-42.9 5.9l-99.8-88.5c11.6-19.4 23.9-38.4 36.6-57.1zM382.3 333.2l83 74.5c10.3 12.8 9.8 31-1.2 43.3-11 12.2-29.1 14.7-42.9 5.9L342 385.7c12.6-17.3 26.1-35.3 40.3-52.5z"
    fill="#F4C1A8"
    p-id="16425"
  ></path>
  <path
    d="M573.3 240.2c-21.9-2-44 2-63.8 11.8-6.7 3.3-2.1 14 4.7 10.7 18-8.8 37.8-12.7 57.9-10.8 7.5 0.6 8.7-11.1 1.2-11.7zM487.3 266.1c-4.7-5.9-13.9 1.5-9.1 7.4 4.7 5.8 13.8-1.5 9.1-7.4z"
    fill="#FFFFFF"
    p-id="16426"
  ></path>
</svg>
```

前面说 canvas 对文本支持弱爆了，但其实，svg 也是半斤八两的，如果 svg 中的文字可以跟 css 中的表现一样就好了。诶嘿还真的有，它里面其实有个 `<foreignObject/>` 标签，也就是说，如果使用 svg 的话，我们不再需要一点点的遍历，转换节点；不用再计算复杂的元素优先级，只需要一股脑的将要渲染的 DOM 扔进 `<foreignObject/>` 就好了，剩下的就交给浏览器去渲染。<del>不要 998，不要 98，通通 9 块 9！</del>

### foreignObject 简介

> SVG 中的 `<foreignObject>` 元素允许包含来自不同的 XML 命名空间的元素。在浏览器的上下文中，很可能是 XHTML / HTML。

比如举个例子：

```html
<svg viewBox="0 0 200 200" xmlns="http://www.w3.org/2000/svg">
  <style>
    polygon {
      fill: black;
    }

    div {
      color: white;
      font: 18px serif;
      height: 100%;
      overflow: auto;
    }
  </style>

  <polygon points="5,5 195,10 185,185 10,195" />

  <!-- Common use case: embed HTML text into SVG -->
  <foreignObject x="20" y="20" width="160" height="160">
    <!--
      In the context of SVG embeded into HTML, the XHTML namespace could
      be avoided, but it is mandatory in the context of an SVG document
    -->
    <div xmlns="http://www.w3.org/1999/xhtml">
      Lorem ipsum dolor sit amet, consectetur adipiscing elit. Sed mollis mollis
      mi ut ultricies. Nullam magna ipsum, porta vel dui convallis, rutrum
      imperdiet eros. Aliquam erat volutpat.
    </div>
  </foreignObject>
</svg>
```

具体其他属性可以[看这里](https://developer.mozilla.org/zh-CN/docs/Web/SVG/Element/foreignObject)

那我们再回到主题上，有了 `foreignObject` 这个利器，我们的实现思路就简单很多了：

1. 首先获取到你需要的 dom，把它给转成需要的 xml 格式
2. 声明一个基础的 svg 模版，这个模版需要一些基础的描述信息，最重要的，它要有 `<foreignObject/>` 这对标签
3. 将要转换好的 dom 模版模版嵌入 `foreignObject` 标签内
4. 利用 `Blob` 构建 svg 图像，取出 `URL`，赋值给 img

```html
<div>
  <h1 style="background: pink;">Hello World</h1>
  <p style="background: orange;">
    break word break word break word break word break word
  </p>
</div>
```

```js
function html2Svg(domStr) {
  let svgXML = `<svg xmlns="http://www.w3.org/2000/svg" width="200" height="200">
                <foreignObject width="100%" height="100%">${generateXML(
                  domStr
                )}</foreignObject>
             </svg>`;
  const svg = new Blob([svgXML], { type: "image/svg+xml" });
  let url = window.URL.createObjectURL(svg);
  let img = new Image();
  img.src = url;
  return img;
}

// 由于 `foreignObject`只能引用XML文档，
// 所以我们需要对DOM进行格式化
function generateXML(domStr) {
  let doc = document.implementation.createHTMLDocument("");
  doc.write(domStr);
  doc.documentElement.setAttribute("xmlns", doc.documentElement.namespaceURI);
  doc = parseStyle(doc);
  let html = new XMLSerializer()
    .serializeToString(doc)
    .replace("<!DOCTYPE html>", "");
  return html;
}
```

可以看到这个实现的思路还是比较简单的，因为省略了解析 dom 的部分，所以就不需要复杂的计算和递归，渲染速度自然要优于前者。巴特，一个最为严肃的问题在于：svg 无法加载外部资源，也就是说，在 svg 里面，无论是图片还是 css 中的背景图，这些资源都是无法加载的。在使用 canvas 实现时，因为我们是一个 node 一个 node 去画，所以不存在资源引用的问题。但使用 svg 实现，相当于我们把文档交给 svg 来渲染一遍，这对于我们来说是其实是无法控制的黑盒操作，是受 svg 控制的。
那有无办法来解决捏，[rasterizeHTML.js](https://github.com/cburgmer/rasterizeHTML.js)这个库里，作者用一系列的 hack 技巧绕过来许多限制，比如：将`<img/>`的 url 转为 `dataURI`；将 `background` 从 `style` 中取出，修改 `url` 后重新插入样式表；将 `link` 的的样式通过 `ajax` down 下来然后注入`<style/>`等等等等...感兴趣可以去库里看看实现。不过 hack 技巧也不是万能的，作者在 readme 里给出了详细的 [Full list of limitations](https://github.com/cburgmer/rasterizeHTML.js/wiki/Limitations)

![img](10.jpg)
即便是刨除前述提到的问题，svg 方案还有个缺点，就是我们觉得方便的 `foreignObject`，它完全不支持 ie 浏览器，下面是官方给出的支持度
![img](11.jpg)
另外 safari 浏览器也出于安全策略的原因支持度较差（[详细戳这里](https://github.com/tsayen/dom-to-image/issues/27)），包括 `dom-to-image` 作者也是推荐在服务端渲染好图片

# 总结

写到这里的话，前端生成海报的大致方案就基本上涵盖全了，需要哪种靠谱的方案相信大家心里已经有数了。
如果是想做一个前端生成固定的海报的话，我是推荐直接 canvas 把那些简单的元素画出来，感觉这种方案目前的兼容性最高，代码也相对简洁，并且网上已经有对文本换行等一系列细节的处理方案了，基本上没什么踩坑的点（<small><del>悄悄看了下，我们的友商实现方案也是这个</del></small>）。但是如果是要实现一个类似前端截屏的生成图片的话，那么还是看你的侧重点，是看重兼容性还是看重还原度，然后按需选择，本文就不做安利了。

希望大家以后有类似的需求场景时，本文能对你们有些帮助。就酱，掰掰~
