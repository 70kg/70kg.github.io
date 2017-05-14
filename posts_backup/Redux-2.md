title: 从零撸一个Redux
date: 2016-12-25 21:53:53
tags: JavaScript
---

#### 扯一扯
这段时间在看 `Redux` 的东西，稍微梳理一下整个框架的流程。其实整个 `Redux` 的代码很少，现在我也不能理解很多它的思想，反正先撸出个简单的 `Redux` 吧。目标是照着它的流程写一个最简单的，也要支持异步 `Action`。也算是个笔记性的东西。<!--more-->
#### `Action`
这里把 `Action` 放到了 `Store` 中，这样在 `Component` 中使用的时候只要导入 `store` 就可以，不用导入很多的 `Action`, ~~就是懒~~。当然也可以使用 `React-Redux` 之类的进行绑定，但是这个库主要不是干这个的。。这个后面会提到一些。因为 `Actions` 都是提前定义的，所有写在了一个 `StoreConfig` 中，在创建 `Store` 的时候就可以把 `actions` 注入到 `Store` 中了。当然你也可以不这么做。先贴一下完整的代码吧

``` javascript
import createStore from './createStore'

var store = createStore({
    //把所有的action都放在了store中
    actions: {
        //同步的action
        'printText': createRequest((text)=> {
            return {text: text + "我想修改"};
        }),
        //异步的action，async await可以看上一篇
        'testAsync': createRequest(async()=> {
            return await Longtime();
        })
    },
    initState: {}
});
//测试的代码不用说吧
function Longtime() {
    return new Promise((resolve, reject) => {
        setTimeout(() => {
            resolve({text: 'success'});
            // reject('error');
        }, 2000);
    });
}

//这个是个actionCreater 之前有写过在项目中的使用 这里也不多说
function createRequest(actionCreator) {
    const finalActionCreator = typeof actionCreator === 'function'
        ? actionCreator
        : (t) => t;
    const actionHandler = (...args) => {
        const action = {
            type: actionHandler.toString()
        };
        //真正的action都放在payload中，同步则是数据，异步是Promise。
        const payload = finalActionCreator(...args);
        if (!(payload === null || payload === undefined)) {
            action.payload = payload;
        }
        
        if (action.payload instanceof Error) {
            action.error = true;
        }

        return action;
    };

    return actionHandler;
}

export default store;
```

#### `Reducer`
这里的 `Reducer` 只创建了一个，就是直接把成功的数据更新到 `Store` 中，也没什么好看的，关于异步的 `Promise` 会在结果返回之后再去通知 `Reducer`去更新 `Store`，当然这就不是这个 `Reducer` 的工作了。代码很简单。

``` javascript
export default function acceptSuccessActionReducer(state, action) {
    if (!action.error && action.payload) {
        return Object.assign({}, state, action.payload);
    }
    else {
        return state;
    }
};
```

#### `Store`
我觉得 `Redux` 的最重要的东西就是 `Store` 了。所有的 `Action` 都要经过 `Store` 分发，当结果返回之后也要通知 `Store` 去更新，`Component` 的数据也是从 `Store` 中获取。先贴一下 `CreateStore` 代码

``` javascript
import {handleActions} from 'redux-actions';
import {applyMiddleware} from 'redux';
import promiseMiddleware from 'redux-promise';
import acceptSuccessActionReducer from '../reducers/SuccessReducer'
export default function (config) {

    var actions = config.actions;

    var initState = config.initState;

    var reducersDict = {};

    Object.keys(actions).forEach(key=> {
        var action = actions[key];
        action.toString = ()=>key;
        reducersDict[key] = acceptSuccessActionReducer;//将action与reducer绑定
    });
    //这个reducers应该是个类似arrays.reduce的function   果然--> @see 'reduce-reducers'   what's means??
    var reducers = handleActions(reducersDict, initState);

    var store = generateStore(reducers, applyMiddleware(promiseMiddleware));

    store.actions = {};
    Object.assign(store.actions, actions);

    store.subscribe(()=> {
        console.log('这是修改后的===>', store.getState());
    });
    return store;
}

function generateStore(reducer, preloadedState, enhancer) {

    if (typeof preloadedState === 'function' && typeof enhancer === 'undefined') {
        enhancer = preloadedState;
        preloadedState = undefined;
    }

    if (typeof enhancer !== 'undefined') {
        if (typeof enhancer !== 'function') {
            throw new Error('Expected the enhancer to be a function.');
        }
        return enhancer(generateStore)(reducer, preloadedState);
    }

    let currentState = preloadedState;
    let currentReducer = reducer;
    let listeners = [];
    return {
        getState() {
            return currentState;//返回当前 state
        },
        subscribe(listener) {
            let index = listeners.length;
            listeners.push(listener); //缓存 listener
            return () => listeners.splice(index, 1); //返回删除该 listener 的函数
        },
        dispatch(action) {
            currentState = currentReducer(currentState, action);
            listeners.slice().forEach(listener => listener());
            return action;//返回 action 对象
        }
    }
}
```
这里偷懒用了一些库，但是仅仅是很简单的几个方法，例如 `import {handleActions} from 'redux-actions';` 这个里面的代码我没看的懂，但是知道返回一个类似`arrays.reduce`的`function`，这个可以理解返回一个 `rootReducer` ，也就是会迭代累加调用所有的 `Reducer` 。`generateStore` 也是抄了一部分的源码，大致的原理应该差不多。从这个也会发现一个问题，每当 `dispatch` 一个 `action` 都会去遍历所有的 `listeners` ，我们一般都是在 `listener` 中去做更新 `UI` 的工作，这样就会造成很多次无用的刷新。这个就可以使用上面提到过的 `React-Redux` 的 `mapStateToProps` 在某个 `Component` 中只去接受自己感兴趣的 `State` 的更新，然后再使用类似 `Immutable` 的库进去比较就可以很好的控制更新。当然这里不会多去说这个，因为我也没具体去理解过，以后可以多去学习学习。
这里还偷懒直接用了 `redux` 中的 `applyMiddleware`。我觉得这是我学习 `Redux` 中看到的最精彩的代码，贴一下

``` javascript
function applyMiddleware() {
  for (var _len = arguments.length, middlewares = Array(_len), _key = 0; _key < _len; _key++) {
    middlewares[_key] = arguments[_key];
  }

  return function (createStore) {
    return function (reducer, preloadedState, enhancer) {
      var store = createStore(reducer, preloadedState, enhancer);
      var _dispatch = store.dispatch;
      var chain = [];

      var middlewareAPI = {
        getState: store.getState,
        dispatch: function dispatch(action) {
          return _dispatch(action);
        }
      };
      chain = middlewares.map(function (middleware) {
        return middleware(middlewareAPI);
      });
      _dispatch = _compose2['default'].apply(undefined, chain)(store.dispatch);

      return _extends({}, store, {
        dispatch: _dispatch
      });
    };
  };
}
```
通过和上面的 `generateStore` 一起看，特别是 `applyMiddleware` 返回的匿名`Function`，真是精彩的代码。再去试想一下如果使用 `Java` 这样的强类型语言去实现得是多麻烦的事情。还有就是 `Function` 的一等公民地位的体现。真的值得多去读一读，很赞。不得不去佩服写这段代码的人。甚至有点自举的感觉。在还没创建 `store` 的时候就可以去使用 `Store.dispatch`，(这句话是有问题的，意会即可)。还有就是 `applyMiddleware` 的整个过程其实没有完全的弄明白，这里也不班门弄斧，等明白了会再来更新的。

#### 结束
最后就是使用了，没什么好说的` store.dispatch(store.actions.testAsync());`就可以了。使用手动的 `subscribe` 或者 `React-redux` 的 `connect` 都能实现 `UI` 的更新，当然更推荐后者。后面还会去学习一个 `React-Redux` 这个东西，毕竟又使用 `React` 又使用 `Redux` 肯定也少不了这个东西。
先说到这吧，有什么想说的再来补一补。
