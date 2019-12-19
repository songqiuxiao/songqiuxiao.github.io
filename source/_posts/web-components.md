---
title: web-components
date: 2019-10-11 23:56:26
tags:
---
组件是前端的发展方向，现在流行的 React 和 Vue 都是组件框架。

谷歌公司由于掌握了 Chrome 浏览器，一直在推动浏览器的原生组件，即 Web Components API。相比第三方框架，原生组件简单直接，符合直觉，不用加载任何外部模块，代码量小。目前，它还在不断发展，已经可用于生产环境。

Web Components API 内容很多，本篇博客不是全面的教程，只是一个简单演示，大家可以看一下怎么用它开发组件。
<!-- more -->
✨ 自定义元素
下图是一个用户卡片。
img
本文演示如何把这个卡片，写成 Web Components 组件。戳这里查看最后的完整代码。

网页只要插入下面的代码，就会显示用户卡片。

<user-card></user-card>
这种自定义的 HTML 标签，称为自定义元素（custom element）。根据规范，自定义元素的名称必须包含连词线，用与区别原生的 HTML 元素。所以，<user-card>不能写成<usercard>。

✨ customElements.define()
自定义元素需要使用 JavaScript 定义一个类，所有<user-card>都会是这个类的实例。

class UserCard extends HTMLElement {
  constructor() {
    super();
  }
}
上面代码中，UserCard就是自定义元素的类。注意，这个类的父类是HTMLElement，因此继承了 HTML 元素的特性。

接着，使用浏览器原生的customElements.define()方法，告诉浏览器元素与这个类关联。

window.customElements.define('user-card', UserCard);
✨ 自定义元素的内容
自定义元素目前还是空的，下面在类里面给出这个元素的内容。

class UserCard extends HTMLElement {
  constructor() {
    super();

    var image = document.createElement('img');
    image.src = 'http://b-ssl.duitang.com/uploads/item/201709/18/20170918154609_MZz3U.jpeg';
    image.classList.add('image');

    var container = document.createElement('div');
    container.classList.add('container');

    var name = document.createElement('p');
    name.classList.add('name');
    name.innerText = 'User Name';

    var email = document.createElement('p');
    email.classList.add('email');
    email.innerText = 'yourmail@email.com';

    var button = document.createElement('button');
    button.classList.add('button');
    button.innerText = 'Follow';

    container.append(name, email, button);
    this.append(image, container);
  }
}
上面代码最后一行，this.append()的this表示自定义元素实例。

完成这一步以后，自定义元素内部的 DOM 结构就已经生成了。

✨ <template>标签
使用 JavaScript 写上面那一段的 DOM 结构很麻烦，Web Components API 提供了
<template id="userCardTemplate">
  <img src="http://b-ssl.duitang.com/uploads/item/201709/18/20170918154609_MZz3U.jpeg" class="image">
  <div class="container">
    <p class="name">User Name</p>
    <p class="email">yourmail@email.com</p>
    <button class="button">Follow</button>
  </div>
</template>
然后，改写一下自定义元素的类，为自定义元素加载
class UserCard extends HTMLElement {
  constructor() {
    super();

    var templateElem = document.getElementById('userCardTemplate');
    var content = templateElem.content.cloneNode(true);
    this.appendChild(content);
  }
}
上面代码中，获取<template>节点以后，克隆了它的所有子元素，这是因为可能有多个自定义元素的实例，这个模板还要留给其他实例使用，所以不能直接移动它的子元素。

到这一步为止，完整的代码如下。

<body>
  <user-card></user-card>
  <template>...</template>

  <script>
    class UserCard extends HTMLElement {
      constructor() {
        super();

        var templateElem = document.getElementById('userCardTemplate');
        var content = templateElem.content.cloneNode(true);
        this.appendChild(content);
      }
    }
    window.customElements.define('user-card', UserCard);    
  </script>
</body>
✨ 添加样式
自定义元素还没有样式，可以给它指定全局样式，比如下面这样。

user-card {
  /* ... */
}
但是，组件的样式应该与代码封装在一起，只对自定义元素生效，不影响外部的全局样式。所以，可以把样式写在
<template id="userCardTemplate">
  <style>
   :host {
     display: flex;
     align-items: center;
     width: 450px;
     height: 180px;
     background-color: #f1d2d7;
     border: 1px solid #f1d2d7;
     box-shadow: 1px 1px 10px rgba(0, 0, 0, 0.3);
     border-radius: 6px;
     overflow: hidden;
     padding: 10px;
     box-sizing: border-box;
     font-family: 'Poppins', sans-serif;
   }
   .image {
     flex: 0 0 auto;
     width: 160px;
     height: 160px;
     vertical-align: middle;
     border-radius: 5px;
   }
   .container {
     box-sizing: border-box;
     padding: 20px;
     height: 160px;
   }
   .container > .name {
     font-size: 20px;
     font-weight: 600;
     line-height: 1;
     margin: 0;
     margin-bottom: 5px;
   }
   .container > .email {
     font-size: 12px;
     opacity: 0.75;
     line-height: 1;
     margin: 15px 0 37px;
   }
   .container > .button {
     border: none;
     padding: 10px 25px;
     font-size: 12px;
     border-radius: 6px;
     text-transform: uppercase;
     background: rgba(255, 255, 255, 0.6);
     transition: ease .2s;
   }
   .container > .button:hover{
     cursor: pointer;
     background: rgba(255, 255, 255, .9);
   }
  </style>
  
  <img class="image">
  <div class="container">
    <p class="name"></p>
    <p class="email"></p>
    <button class="button">Follow</button>
  </div>
</template>
上面代码中，<template>样式里面的:host伪类，指代自定义元素本身。

✨ 自定义元素的参数
<user-card>内容现在是在<template>里面设定的，为了方便使用，把它改成参数。
改造成下面这样使用：

<user-card
  image="http://b-ssl.duitang.com/uploads/item/201709/18/20170918154609_MZz3U.jpeg"
  name="User Name"
  email="yourmail@email.com"
></user-card>
<template>代码也相应改造。

<template id="userCardTemplate">
  <style>...</style>

  <img class="image">
  <div class="container">
    <p class="name"></p>
    <p class="email"></p>
    <button class="button">Follow</button>
  </div>
</template>
最后，改一下类的代码，把参数加到自定义元素里面。

class UserCard extends HTMLElement {
  constructor() {
    super();

    var templateElem = document.getElementById('userCardTemplate');
    var content = templateElem.content.cloneNode(true);
    content.querySelector('img').setAttribute('src', this.getAttribute('image'));
    content.querySelector('.container>.name').innerText = this.getAttribute('name');
    content.querySelector('.container>.email').innerText = this.getAttribute('email');
    this.appendChild(content);
  }
}
window.customElements.define('user-card', UserCard);
✨ Shadow DOM
我们不希望用户能够看到<user-card>的内部代码，Web Component 允许内部代码隐藏起来，这叫做 Shadow DOM，即这部分 DOM 默认与外部 DOM 隔离，内部任何代码都无法影响外部。

自定义元素的this.attachShadow()方法开启 Shadow DOM，详见下面的代码。

class UserCard extends HTMLElement {
  constructor() {
    super();
    var shadow = this.attachShadow( { mode: 'closed' } );

    var templateElem = document.getElementById('userCardTemplate');
    var content = templateElem.content.cloneNode(true);
    content.querySelector('img').setAttribute('src', this.getAttribute('image'));
    content.querySelector('.container>.name').innerText = this.getAttribute('name');
    content.querySelector('.container>.email').innerText = this.getAttribute('email');

    shadow.appendChild(content);
  }
}
window.customElements.define('user-card', UserCard);
上面代码中，this.attachShadow()方法的参数{ mode: 'closed' }，表示 Shadow DOM 是封闭的，不允许外部访问。
如图，Shadow DOM会在自定义标签解析时，加载到普通的DOM上。内部可以通过this.attachShadow()来获取shadow root。它有一个mode属性，值可以是open或者closed,表示能否在外部获取Shadow DOM对象，一般而言应当为closed，内部实现不应该对外可见。
img

✨ 组件的扩展
至此，这个 Web Component 组件就完成了，完整代码戳这里访问。
可以看到，整个过程还是很简单的，不像第三方框架那样有复杂的 API。
那在前面的基础上，可以对组件进行扩展。

与用户互动
上面展示的用户卡片是一个静态组件，如果要与用户互动，也很简单，就是在类里面监听各种事件，比如像下面这样：

this.$button = shadow.querySelector('button');
this.$button.addEventListener('click', () => {
  // do something
});
组件的封装
上面的例子中，<template>与网页代码放在一起，其实可以用脚本把<template>注入网页。这样的话，JavaScript 脚本跟<template>就能封装成一个 JS 文件，成为独立的组件文件。网页只要加载这个脚本，就能使用<user-card>组件。

✨ 总结
任何UI框架或库最期望目标之一是帮助我们建立通用的模式或约定。这些约定使UI代码易于共享并为其提供理论基础。很长一段时间，每个框架或库都有自己的实现或UI组件版本。仅在该组件的生态中，这些组件可以实现代码复用。如果给定的UI组件/插件需要在不同的技术依赖中使用，往往由于特定的生态系统限制而成为局限。 例如，如果我编写一个Angular库并想在我的Vue应用程序中使用我的Angular下拉列表，目前还无法直接做到。如果有一个基于任意JavaScript库或框架编写的通用标准组件可以在任何浏览器中用，那就很好了！这也是Web Components的愿景。期待后面Web Components给前端带来更多的便捷~

更多 Web Components 的高级用法就不展开详细讲了，可以接着学习下面两篇文章。

Web Components Tutorial for Beginners
Custom Elements v1: Reusable Web Components