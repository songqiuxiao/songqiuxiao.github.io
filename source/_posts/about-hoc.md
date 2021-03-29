---
title: 高阶组件
date: 2021-03-28 11:34:41
tags:
---

不知道你在工作中有没有接触过高阶组件，很多时候我是没什么机会去写一个高阶组件，所以这个东西在我这里就是一个理论知识储备，或许只会在面试或者是看其他人代码的时候才会去看看。前两天查了一个高阶组件内的 bug，查着查着我就突然在思考，为什么一个很简单的功能，要用高阶组件来实现，高阶组件能为我们带来什么呢？

<!-- more -->

事情是这样的，前两天在做一个紧急需求，需求本身不复杂，但是设计用到了【新手指引】这个交互组件，这个组件的场景相信大家都见过，比如你打开一个新的游戏，肯定有很多 npc 来指引你做一些内容，抓住你眼球的同时，又会强制让你学习某个功能的使用。我们这个组件是一个简化版本，大概就是当你作为一个首次使用某个 亮眼/难以操作 的功能的用户时，系统会高亮出该功能区域，其他区域用一个蒙层遮盖，且在高亮附近有一个友好的提示文案，就像下面这个图一样：
![img](1.jpg)

~~说到这个紧急需求就郁闷，怎么也没想到 315 平台点名的几个投简历的网站，被 cue 说泄露客户隐私的这个事情，居然会波及到我的工作，不得不为了这个隐私而加班补充了一系列功能。。TAT~~

嗯，思考一下，如果让你实现一个这样的功能，你会怎么做？

🙋🏻‍♀️ 我先来，这个功能在移动端并没有实现过，所以我自己写了一个移动端的逻辑，我的思路就是快而简的显示，先写一个 `<Guide/>` 组件，利用 `redux` 存储要展示的指引 `id`，然后在对应的要展示界面上根据一系列判断，然后调用这个公共的 `<Guide/>` 组件展示就好。

大概是下面这段代码逻辑：

```js
//action.js
 function fetchGuideId() {
  return (dispatch) =>
    request
    .get('/api/getGuideId')
    .then(guideIds) =>
      dispatch({
        type: 'REPLACE_GUIDE_IDS',
        payload: { guideIds }
      })
}

// reducer.js
function guides(state = {}, action) {
  switch (action.type) {
    case "REPLACE_GUIDE_IDS": {
      const { guideIds } = action.payload;
      return { ...state, guideIds };
    }

    default:
      return state;
  }
}
```

```js
// App.js
import React from "react";
import Guide from "./Guide";

class App extends React.Component {
  constructor(props) {
    super(props);
  }

  registerGuide = () => {
    const position = this.guideRef.getBoundingClientRect();
    // 根据 position 的一系列信息展示guide的位置
    return <Guide position={position} />;
  };

  render() {
    const someGuideId = "someGuideId";
    // 判断是首次使用此功能的用户
    const showGuide = this.props.guideIds.includes(someGuideId);
    return (
      <div>
        <div ref={(ref) => (this.guideRef = ref)}>
          {/* 要绑定高亮显示guide的 */}
        </div>
        {showGuide && this.registerGuide()}
        <div>{/* 其他的页面圆度 */}</div>
      </div>
    );
  }
}

export default App;
```

嗯。大概就是上面那个样子，基本上就已经满足的需求场景了。

当然啦，看到这个标题你就知道，这个组件最终在我们的系统 pc 端里被设计成了一个高阶组件，一个比较大(大概 329 行，还行)且复杂的组件，还有点点 bug 🤦🏻‍♀️

![img](2.jpg)

然后我一边改着 bug，一边思考，高阶组件是什么，设计之初到底是为了解决什么痛点，看了下官方文档是这么定义的：

> 高阶组件（HOC）是 React 中用于复用组件逻辑的一种高级技巧。HOC 自身不是 React API 的一部分，它是一种基于 React 的组合特性而形成的设计模式
> 具体而言，**高阶组件是参数为组件，返回值为新组件的函数**。

```js
const EnhancedComponent = higherOrderComponent(WrappedComponent);
```

> 组件是将 `props` 转换为 UI，而高阶组件是将组件转换为另一个组件。
> HOC 在 React 的第三方库中很常见，例如 Redux 的 `connect` 和 `Relay` 的`createFragmentContainer`。

诶？说到这里的话，那就会发现，其实你已经悄无声息的使用过很多的高阶组件了，比如说 `connect`。相信大家只要是用了 React 的基本上都有用过这个 HOC。那么仔细看文档就会理解高阶组件的意义，很多时候我们会遇到一些公共的 UI+逻辑，比如说上面提到的新手指引，当然对于界面，我们可以封装出一个公共的 `Guide` 组件，但是在使用的时候，你还是需要写很大一段公共的逻辑，去判断什么时候注册、什么时候渲染、渲染到界面的什么节点，这时候，把这一整套的 UI+逻辑抽离出来做一个高阶组件，然后在使用的时候去传入一些约定好的参数就可以节省很多的时间了。

那还是说回这个新手指引组件，这时候就很适合抽离出一个高阶组件，来统一判断什么时候显示这个 guide，talk is less ，show me the code。

```js
// withGuides.js
import React from "react";
import Guide from "./Guide";

/**
 * Guide HOC
 * @return {React.ComponentClass<T>}
 */
export function withGuides(WrappedComponent, guides) {
  class EnhanceWithGuides extends React.Component {
    constructor(props) {
      super(props);

      this.mountNode = null;
    }

    /**
     * 判断wrapper的组件guides中是否有guideId可以替换当前guideId,
     * 如果有,则返回对应Id,没有则返回null
     */
    getCanLoadGuideId(availableGuideIdSet, currentGuideId) {
      return availableGuideIdSet.has(currentGuideId) ? currentGuideId : null;
    }

    renderGuide = () => {
      if (!this.mountNode) {
        return;
      }

      const { currentGuideId, availableGuideIdSet } = this.props;

      const canLoadGuideId = this.getCanLoadGuideId(
        availableGuideIdSet,
        currentGuideId
      );

      // 判断一些情况
      if (currentGuide && currentStep && dataReady && targetDOMReady) {
        // 当前guide的某个step就是要被展示的step，则render guide
        ReactDOM.render(<Guide {...this.props} />, this.mountNode);
      } else {
        // 否则撤掉guide
        ReactDOM.unmountComponentAtNode(this.mountNode);
      }
    };

    componentDidMount() {
      if (!this.mountNode) {
        this.mountNode = document.createElement("div");
      }
      document.body.appendChild(this.mountNode);
      this.renderGuide();
    }

    componentDidUpdate() {
      this.renderGuide();
    }

    componentWillUnmount() {
      if (!this.mountNode) {
        return;
      }
      try {
        // 不要忘记卸载
        ReactDOM.unmountComponentAtNode(this.mountNode);
        document.body.removeChild(this.mountNode);
      } catch (e) {
        // do nothing.
      }
    }

    render() {
      /**
       * 这里将 renderGuide 方法传给子组件 是为了解决显示新手指引的地方并非子组件 didMount 就出现
       * 可以在需要渲染指引的时机 再次调用 renderGuide 方法
       */
      return (
        <WrappedComponent {...this.props} renderGuide={this.renderGuide} />
      );
    }
  }

  return EnhanceWithGuides;
}
```

有了这个高阶组件之后，你之后就可以直接传递一点基础参数给这个 HOC，剩下的交给它就行：

```js
// App.js
import withGuides from "./withGuides";

class App extends React.Component {
  constructor(props) {
    super(props);
  }

  render() {
    return (
      <div>
        {/* 其他页面元素*/}
        <div id="guide">{/* 需要指引的内容*/}</div>
      </div>
    );
  }
}

const guides = [
  {
    id: "only-guide-id",
    steps: [
      {
        name: "文案标题",
        title: "文案详情内容~~~~",
        selector: "#guide",
        // ...其他配置参数
      },
    ],
  },
];

export default withGuides(App, guides);
```

希望你看完这篇文章之后能重新认识一点 HOC 到底是什么，什么时候会用到，要怎么使用。它不再是一个枯燥的面试知识点，因为很多库已经证明了它的优秀。

### 参考文献

- [Higher-Order Components](https://reactjs.org/docs/higher-order-components.html)

### 推荐阅读

- [深入理解 React 高阶组件](https://zhuanlan.zhihu.com/p/24776678)
- [React Higher Order Components in 3 minutes](https://codeburst.io/higher-order-components-in-3-minutes-93173b2ebe52)
