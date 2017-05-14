---
title: 简单谈谈Redux在项目中的使用
date: 2016-12-05 18:54:46
tags: Redux
---

# Reducer

`Reducer` :为了描述 `action` 如何改变 `state tree`。 输入 `action` 和 `state`，一般根据 `action` 的 `type` 进行区分，然后处理，改变 `state tree`。<!--more-->

`Reducer` 是纯函数，纯函数就是输入一个值，会输出一个确定的值，也就是 `Reducer` 里面不应该有 `getDate()`, 调用 `API` 等等之类不确定的操作。

e.g :

```javascript
    const store = configStore(reducers)

    KeyboardReducer
    
    export const setKeyboardHeight = createRequest('setKeyboardHeight', (keyboardHeight) => {
    var oldKeyboarHeight = store.getState()['keyboardHeight'];
    if (oldKeyboarHeight == keyboardHeight) {
        return;
    }

    return {keyboardHeight};
}
    export default KeyboardReducer = handleActions({
        [setKeyboardHeight]: acceptSuccessActionReducer,
    },
    {
        keyboardHeight: 0,
    });
});
```
这里有个 `KeyboardReducer` 的一个自定义的 `Reducer` ,用来处理键盘高度的事件，先不看实现，这个 `Reducer` 接受一个 `type`  为 `setKeyboardHeight` 的 `action` ，然后获取原 `store` 中的 `oldState` 中的 `keyboardHeight` ，然后对比，改变 `state tree` 中的 `keyboardHeight` 这个 `state`。 同时可以发现，在创建给某个模块使用的 `store` 的时候，这个 `reducer` 已经对 `state tree` 做了一些改变，设置了 `keyboardHeight : 0` 这个默认值,举例这个我们可以在cs模块的 

```javascript
const store = configStore(reducers);
store.dispatch(getQuestions());

export default store;
```
在处理模块自己的 `action` 之前，断点可以看到

    store.getState()['keyboardHeight'] ：0
也就印证了我们的猜想。

#### 拆分 `Reducer`

`Reducer`接受一个 `state` 和 `action` ，返回一个新的 `State`,如果项目很大，对应的 `state` 也会很大，如果使用一个 `Reducer` 进行整个的 `state` 的处理就会非常的臃肿。这个时候就可以进行拆分 `Reducer` ,每个 `Reducer` 去处理自己职责范围内的 `state` ，类似上面的  `KeyboardReducer` 就只对 `store.getState()['keyboardHeight'];` 这个 `stage` 感兴趣。 因为 `store` 只有一个，当分成多个 `Reducer` 的时候，那就需要一个 `Reducer` 去管理这些 `Reducers` 。可以使用 `combineReducers` 进行 `Reducer` 的合成，

```javascript

function foo(state = {},action){
    switch(action.type){
    ...
    }
    return Object.assign({},state,........);
}
function foo1(state = {},action){
    switch(action.type){
    ...
    }
    return Object.assign({},state,........);
}

const App = combineReducers({
    foo,
    foo1
});

```

在项目中

```javascript
export const reducers = handleActions({
    [setBrandId]: acceptSuccessActionReducer,
    [getBrandCatalogs]: acceptSuccessActionReducer,
    [getBrandInfo]: acceptSuccessActionReducer,
    [setOrderByAndDirect]: acceptSuccessActionReducer,
    [getFilterInfo]: acceptSuccessActionReducer,
    [setFilter]: acceptSuccessActionReducer,
    [followBrand]: acceptSuccessActionReducer,
    [setPriceSelect]: acceptSuccessActionReducer,
    [getCampaign]: acceptSuccessActionReducer,
    [resetgetCampaigned]: acceptSuccessActionReducer,
}
```
这里每个请求都对于同样的 `acceptSuccessActionReducer` 这个 `Reducer`，代码很简单就不贴了，最终这些 `reducer` 的统一处理是在 `createStore` 的时候，下面会说到

项目中管理这些 `Reducers` 是使用了 
``` javascript
reduceReducers(actionStateReducers, KeyboardReducer, reducer)

export default function reduceReducers(...reducers) {
    return (state, action) => {
        //框架的安全检查类型，可暂时忽略。。
        // if (action.type === ActionTypes.INIT) {
        if (action.type === '@@redux/INIT') {
            var state = {};
            // collect all init state.
            reducers.forEach(reduce => {
                state = Object.assign(state, reduce(undefined, action));
            });
            return state;
        }
        else {
            return reducers.reduce(
                //迭代累加调用reducer
                (state, reduce) => reduce(state, action),
                state
            );
        }
    }
}

```
# `Action`
`Action`可以理解为一个动作或者事件。它的作用是将信息传递给某个 `Reducer` 进而去影响 `State tree`,而使用这些 `Action` 只有一种方法就是 `store.dispatch(action)`, 在项目中可以手动去调用，也可以使用 `react-redux` 的 `bindActionCreators`方法自动将多个 `Action` 绑定到 `Dispatch()` 方法上。这个肯定也是调用了 `store.dispatch(action)` ,一般的 `Action` 的结构类似：

```
{
    type: "action-type"
    value: ""
}
```
`action`的定义十分灵活，一般只有 `type`是固定的，其他随便。但是在异步的 `action` 中，通过 `actionCreater`创建的一般都是 `function`，返回值才是起作用的 `action`。

# `Store`
每个 `Redux`应用只有一个 `store`，如果觉得数据很多很臃肿，应该去使用组合 `Reducer` 的方法而不是去创建多个 `store` 解决。 `store`  有下面几个重要方法：

 - `store.getState` 对当前 `store` 进行快照，获取当前 `store` 对应的 `state`.
 - `dispatch(action)` 分发 `action` 给 `reducer`。
 - `subscribe(listener)` 注册监听器，当 `store` 改变时回调监听器。

e.g.

```javascript
store.subscribe(() => {
    //当进入品牌详情页面，设置brandId导致store的改变，进而回调这里，去请求网络。
    var brandId = store.getState()['brandId'];
    if(brandId && brandId !== lastBrandId) {
        lastBrandId = brandId;
        store.dispatch(getBrandInfo());
        store.dispatch(getFilterInfo());
        store.dispatch(getBrandCatalogs(true));
    }
});
```

# `Middleware`
对于中间件，看这个就够了，我也说不了比它更好：

[点击了解Middleware](http://cn.redux.js.org/docs/advanced/Middleware.html)
# 数据流
来简单的看一下项目中的一个请求 `action` 是如果经过一步步到最终 UI 上面展示的。先从创建 `Store` 开始，

```javascript
//参数reducer是我们在模块中自己定义 一般一个请求对应一个reducer 参考actions目录
export default function configStore(reducer) {
    //createStore 第一个参数是reducer
    const store = createStore(
        //array.reducer类似一个迭代累加的效果 这里就是迭代去调用各种reducer,累加对store的影响
        //actionStateReducers:主要是为了异步的action 自动多了ACTION_WILL_START,ACTION_WILL_END这两个action
        //KeyboardReducer :处理键盘的
        //我们在模块中自己定义的各个reducer 详情见模块下的actions/index.js 中的export const reducers = handleActions。。。。。
        //类似注册的作用 预先创建各个 Reducers 到使用的时候一般由下面的中间件中调用。
        reduceReducers(actionStateReducers, KeyboardReducer, reducer),
        
        //Middleware作用在action创建之后,到达reducer之前的阶段
        //actionStateMiddleware:对应上面的actionStateReducers,处理多出来的两个action
        //会去调用上面注册的东西
        applyMiddleware(actionStateMiddleware, errorHandlerMiddleware, promiseMiddleware, createLogger({
            predicate: function (state, action) {
                if (action.type === 'ACTION_WILL_START' || action.type === 'ACTION_WILL_END' || action.type === 'CONCURRENT_EXECUTION_ERROR') {
                    return false;
                }

                return true;
            },
        }))
    );

    global.store = store;

    return store;
}
```
我们去创建一个请求，都是使用 `createRequest` ，其实也就是创建了一个 `action` ，但是这个 `action` 是个 `function`，这个 `function`的返回值是真正的 `action`， 这个先不管，但是起作用的 `action` 结构类似 ：

```javascript
//同步 {
        //   type:setBrandId
        //   payload:object
        // }
//异步{
        //   type:getBrand
        //   payload:promise
        // }
```
对于最常用的异步 `action` ,对应与 `actionStateReducers` 这个 `reducer` ，详情：

```javascript
//每个action都会多创建这两个action，在原始action开始和结束的时候分发。
const actionWillStart = createAction('ACTION_WILL_START');
const actionDidEnd = createAction('ACTION_WILL_END');

export const actionStateReducers = handleActions({
    [actionWillStart]: (state, action) => {
        var actionName = action.payload;//请求回来的数据会在action.payload中 进而去改变store action type也就是请求名

        console.log('调用处理开始的action的地方,action-->', action);

        //state中包含actionstate和我们自己定义的各种state
        var actionState = state['actionState'];
        //actionstate的结构类似这样:
        //{
        // getLiveShow:1   //是action的名称 和 数量 这个数量暂时还不知道是干什么用的  一般就是0或者1
        // }
        var count = actionState[actionName];
        if (count) {
            count += 1;
        }
        else {
            count = 1;
        }
        actionState[actionName] = count;
        //如果是ACTION_WILL_START这个action  就把对应的action中的actionstate +1 表明创建了这个一个action??
        //然后应用修改的state
        return Object.assign({}, state, {actionState});
    },
    //这个是ACTION_WILL_END 同上
    [actionDidEnd]: (state, action) => {//在数据回来的时候调用
        var actionName = action.payload;

        var actionState = state['actionState'];
        var count = actionState[actionName];
        --count;
        actionState[actionName] = count;

        console.log('调用处理结束的action的地方,action-->', action);

        return Object.assign({}, state, {actionState});
    },
}, {
    actionState: {}
});

```
 上面的代码主要是帮我们多创建了两个 `action`，而何时去调用，调用的顺序是在下面的 `actionStateMiddleware` 中定义的，详情：
 

```javascript
 export function actionStateMiddleware({dispatch, getState}) {
    return next => {
        return action => {
            // action throttling
            if (isPromise(action.payload)) {
                if (action.type == 'CONCURRENT_EXECUTION_ERROR') {
                    return next(action);
                }

                console.log('进来中间件,action-->', action);

                if (action.type !== actionWillStart.toString() &&
                    action.type !== actionDidEnd.toString()) {
                    console.log('分发开始的action,action-->', action);
                    dispatch(actionWillStart(action.type));//不是那两种 action, 分发actionWillStart:{type:action.type}
                }
                //这里的payload是个promise
                action.payload.then(
                    result => {
                        if (action.type !== actionWillStart.toString() &&
                            action.type !== actionDidEnd.toString()) {
                            console.log('分发结束的action,action-->', action);
                            dispatch(actionDidEnd(action.type));//数据回来的时候 调用这个
                        }
                    },
                    error => {
                        if (action.type !== actionWillStart.toString() &&
                            action.type !== actionDidEnd.toString()) {
                            dispatch(actionDidEnd(action.type));
                        }
                    }
                );
            }
            else if (isFSA(action) && !action.error && !action.payload) {
                // return action directly, and the payload will not be reduced into store.
                return action;
            }
            return next(action);
        };
    }
}
```
而处理原始的 `action` 则是在 `acceptSuccessActionReducer` 中：

```javascript

export function acceptSuccessActionReducer(state, action) {
    if (!action.error && action.payload) {

        console.log('调用处理原始的action的地方,action-->', action);

        return Object.assign({}, state, action.payload);
    }
    else {
        return state;
    }
};
```

何时去调用则是在 `promiseMiddleware` 这个中间件中：

```
....略
 return isPromise(action.payload) ? action.payload.then(function (result) {
        return dispatch(_extends({}, action, { payload: result }));
....
```


最后以品牌详情切换排序方式为例，看一下完整的 `action` 到 UI 改变的过程。

![](http://7xjlmz.com1.z0.glb.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202016-12-04%20%E4%B8%8B%E5%8D%888.54.40.png) 

无论是开始还是结束的action，都会触发 `state` 中的 `actionState` 的改变，进而触发 `store` 的改变，进而触发页面中 `state` 的更新。所以会出现多次更新 UI的情况。

createRequest
->dispatch(action)
->actionStateMiddleware
->dispatch(actionWillStart(action.type))
->reduceReducers
->迭代执行reducers(1:包括start,end的actionStateReducers，2：KeyboardReducer，3：模块中定义的reducers：handactions(.....))
->调用handleActions中start action对应的reducer逻辑
->修改start action 对应的actionstate
->请求结果返回
->调用handleActions中end action对应的reducer逻辑
->修改end action 对应的actionstate
->调用promiseMiddleware中的promise响应
->修改store
->调用组件中mapStateToProps
->调用componentWillReceiveProps。

## `Why Redux`
扯了半天，为什么要用 `Redux`? 它的定义是 :**Redux 是 JavaScript 状态容器，提供可预测化的状态管理。** 为什么说是可预测的状态，因为一个确定的 `state` 就对应一个确定的 `view`，使用 `Redux` 能感受到它的数据的单向流动性，用户触发某个动作，发出一个 `action` ，经过 `Reducer`,改变 `store` 的数据，进而去控制 UI的效果，仅仅有这一种方式，单向数据流也保证了可预测的状态。如果不这样做，我们的 `View` 可以被各种各样的 `event`,`Model`甚至其他的 `View`控制，一个 `View` 的状态无法预测，当改变因素很多的时候就无法控制。这只是我浅显的理解。
![](http://7xjlmz.com1.z0.glb.clouddn.com/WechatIMG4.jpeg)

## ps
好大老师改了原 `actions/index.js`和 `store/index.js`，结合了这两个，看完了上面的这些东西，再去看改动应该就没什么压力了，也更理解为什么这么改。