---
title: react-router 背后的故事
date: 2019-04-07 17:58:49
tags:
---

## 底层实现 —— history

<!-- more -->

> 本节内容转载自
> 作者：zhenhua-lee
> 链接：https://juejin.im/post/5b797c52e51d4538db349b05

`history` 是一个独立的第三方js库，可以用来兼容在不同浏览器、不同环境下对历史记录的管理，拥有统一的API。具体来说里面的history分为三类:

* 老浏览器的history: 主要通过hash来实现，对应 createHashHistory
* 高版本浏览器: 通过 html5 里面的 history，对应 createBrowserHistory
* node 环境下: 主要存储在 memeory 里面，对应 createMemoryHistory （木有什么接触，不扩展介绍）


### URL 前进
* createBrowserHistory: pushState、replaceState
* createHashHistory: location.hash=***、location.replace()

### 检测URL回退
* createBrowserHistory: popstate
* createHashHistory: hashchange

### state 值存储在 sessionStorage 里


## rr4 的实现
{% asset_img rrworkflow.png quote from http://www.thisgirlcan.co.uk/activities/rrworkflow.png %}

`Router` 监听 history 的变化，并且通过 `state` 影响子组件的变化。
```
this.unlisten = props.history.listen(location => {
  if (this._isMounted) {
    this.setState({ location });
  } else {
    this._pendingLocation = location;
  }
});

```


## 路由匹配
在之前的版本中，在 Route 中写入的 `path`，在路由匹配时是独一无二的，而 v4 版本则有了一个包含的关系：如匹配 `path="/users"` 的路由会匹配  `path="/"` 的路由，在页面中这两个模块会同时进行渲染。

如果只想匹配一个路由的话，则需要 `<Switch/>` 。v4这个改造与其 **router即组件** 的思路有一定关系， 多个 match 的组件会对一个 url 进行响应，`<Switch/>` 就是个用户定制的条件。

在 `matchPath` 函数中使用了 `"path-to-regexp"` 来匹配 `restful` 接口，写法较为优雅。

```js
// 
const compilePath = (path, options) => {
  const keys = [];
  const regexp = pathToRegexp(path, keys, options);
  return { regexp, keys };
}

const { regexp, keys } = compilePath(path, {
  end: exact,
  strict,
  sensitive
});
const match = regexp.exec(pathname); // 正则匹配

if (!match) return null;

const isExact = pathname === url; // 是否精准匹配
```


## 参考资料
* [React-Router底层原理分析与实现](https://juejin.im/post/5b797c52e51d4538db349b05)
* [react-router的实现原理](http://zhenhua-lee.github.io/react/history.html)
* [react-router 4 升级](https://www.jianshu.com/p/bf6b45ce5bcc)
