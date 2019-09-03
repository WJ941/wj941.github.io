---
title: React-Router 原理
date: 2019-09-03 12:29:01
tags:
---
# React-Router是怎么工作的？
### Router组件是什么？
`Router`组件是`RouterContext`的`Provider`，他的value是一个包括`history`, `location`, `match`, `staticContext`的对象。`history`就是`props.history`，`location`是`props.history.location`， `match`由`props.history.location.pathname`计算而来，
`staticContext`是`props.staticContext`。
`BrowserRouter`是使用`history`第三方库的`createBrowserHistory`方法生成的`history`对象，传递给`Router`组件。
```
function BrowserRouter() {
  return <Router history={createBrowserHistory} />
}
```
### Route组件是什么？
`Route`组件是`RouterContext`的`Consumer`，计算是否匹配当前路径，如果不匹配返回`null`，不渲染当前组件。
### WithRouter是什么？
放置在`Route`的`component`属性中的组件，属性里会有路由`history`,`match`, `location`和`staticContext`对象；
对于其他组件，如果也想获得路由相关的信息，可以使用`withRouter`高阶函数,
`withRouter(component)`的原理很简单，就是把`component`外面包装一层`ReactContext.Consumer`，把路由传递给`component`;
```
function withRouter(compoent) {
  return props => (
    <RouterContext.Consumer>
      {
       value => React.createElement(component, {...value, ...props}) 
      }
    </RouterContext.Consumer>
  )
}
```