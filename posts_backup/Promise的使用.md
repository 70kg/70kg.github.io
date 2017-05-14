layout: asncy
title: Asncy await Promise的使用
date: 2016-12-15 11:53:56
tags: JavaScript
---

```javascript
const fetch = () => {
    return new Promise((resolve, reject) => {
        setTimeout(() => {
            // resolve('success');
            reject('error');
        }, 2000);
    });
};

const testAsync = async() => {
    try {
         return await fetch();
    } catch (error) {
        return Promise.reject(error);
    }
};

testAsync().then((result)=> {
    console.log('这是成功的--->' + result);
}).catch((error)=> {
    console.log('这是失败的--->' + error);
});;
```
`async`函数返回的一个 `Promise`， 调用耗时函数前面加 `await` 关键字，返回成功的值，可以使用 `try..catch` 来进行捕获错误。上面的代码可以模拟一次请求的过程，最基本用法差不多都包括了<!--more-->

[点我运行][1]

#### `Promise.then`是异步的

```
var promise = new Promise(function (resolve){
    console.log("inner promise"); // 1
    resolve(42);
});
promise.then(function(value){
    console.log(value); // 3
});
console.log("outer promise"); // 2


----------
inner promise // 1
outer promise // 2
42            // 3

```
[为啥这样][2]

更多的关于 `Promise` 看下面的这本电子书，就不扯了，啦啦啦。

参考:
[JavaScript Promise迷你书（中文版）][3]


  [1]: http://babeljs.io/repl/#?babili=false&evaluate=true&lineWrap=true&presets=latest,react,stage-2&code=const%20fetch%20=%20%28%29%20=%3E%20%7B%0A%20%20%20%20return%20new%20Promise%28%28resolve,%20reject%29%20=%3E%20%7B%0A%20%20%20%20%20%20%20%20setTimeout%28%28%29%20=%3E%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20//%20resolve%28%27success%27%29;%0A%20%20%20%20%20%20%20%20%20%20%20%20reject%28%27error%27%29;%0A%20%20%20%20%20%20%20%20%7D,%202000%29;%0A%20%20%20%20%7D%29;%0A%7D;%0A%0Aconst%20testAsync%20=%20async%28%29%20=%3E%20%7B%0A%20%20%20%20try%20%7B%0A%20%20%20%20%20%20%20%20%20return%20await%20fetch%28%29;%0A%20%20%20%20%7D%20catch%20%28error%29%20%7B%0A%20%20%20%20%20%20%20%20return%20Promise.reject%28error%29;%0A%20%20%20%20%7D%0A%7D;%0A%0AtestAsync%28%29.then%28%28result%29=%3E%20%7B%0A%20%20%20%20console.log%28%27%E8%BF%99%E6%98%AF%E6%88%90%E5%8A%9F%E7%9A%84---%3E%27%20%2b%20result%29;%0A%7D%29.catch%28%28error%29=%3E%20%7B%0A%20%20%20%20console.log%28%27%E8%BF%99%E6%98%AF%E5%A4%B1%E8%B4%A5%E7%9A%84---%3E%27%20%2b%20error%29;%0A%7D%29;;&experimental=false&loose=false&spec=false&playground=true
  [2]: https://wohugb.gitbooks.io/promise/content/usage/async.html
  [3]: https://wohugb.gitbooks.io/promise/content/index.html