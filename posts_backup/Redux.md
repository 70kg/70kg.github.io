---
title: React Native + Redux + React-Redux
date: 2016-09-30 15:17:53
tags: React Native
---
在`Component`中的`state`相当于组件内的本地变量，用于储存view的本地变化

`props`中存储数据， `props`属于readOnly的常量dic<!--more-->

store相当于全集的数据状态仓库，只有一个，改变它的唯一方式就是store.dispatch(action)，可以通过store.getState对当前的store进行一次快照，store还有store.subscribe方法，也就是在store.dispatch触发了store的数据更新，回调这个subscribe方法。

使用了React-Redux这个库有两个重要的方法：`mapStateToProps`是用来把外部的state,也就是当前store的快照映射到UI组件的参数上(props),也就是输入数据。`mapDispatchToProps`是把对UI组件的操作映射成action，

当dispatch一个action后，reduce会自动生成一个当前store的快照 =>state，调用mapStateToProps，将state映射到props,调用React native的生命周期方法`componentWillReceiveProps`,这里可以再处理一下数据，改变this.state的值。

`mapStateToProps`会订阅 Store，每当state更新的时候，就会自动执行，重新计算 UI 组件的参数，从而触发 UI 组件的重新渲染
