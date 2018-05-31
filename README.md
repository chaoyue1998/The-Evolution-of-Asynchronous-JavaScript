# 异步javascript 进化史
文章参考：[ECMAScript 6入门](http://es6.ruanyifeng.com/#docs/generator)
## 回调函数

[callback](https://blog.risingstack.com/node-js-best-practices/)
```js
// 惯例
fs.readFile(path, function(err, data) {  
   callback(JSON.parse(data));
});

// cerror in callback
function readJSON(path, callback) {
  fs.readFile(path, function(err, data) {  
    callback(JSON.parse(data));
  });
}

readJSON('./package.json', function (err, pkg) { ... }

// 返回callback，处理错误
function readJSON(filePath, callback) {
  fs.readFile(filePath, function(err, data) {
    if (err) {
      return callback(err);
    }
    
    return callback(null, JSON.parse(data));
  });
}
```
面临的挑战
* 如果使用不当面临大量回调和代码混乱
* 容易忽略错误
* return 不能同步返回值

## Promises
>目前的 JavaScript Promise 规范始于 2012 年，从 ES6 以后提供了支持。然而 Promises 不是 JavaScript 社区发明的，这个术语是 1976 年 Daniel P. Friedman 提出的。

## Generators /yield
### 基本概念
Generator函数形式上是一个普通函数，但是有两个特征：
* function关键字与函数名之间有一个星号
* 函数体内部使用yield表达式，定义不同的内部状态

```js
function* helloWorldGenerator() {
  yield 'hello';
  yield 'world';
  return 'ending';
}
var hw = helloWorldGenerator();
```

执行 Generator 函数会返回一个Iterator遍历器对象，返回的遍历器对象可以依次遍历 Generator 函数内部的每一个状态。

> Generator 函数还是***状态机***，状态机是有序的。

```js
hw.next()
// { value: 'hello', done: false }

hw.next()
// { value: 'world', done: false }

hw.next()
// { value: 'ending', done: true }

hw.next()
// { value: undefined, done: true }
```

> ES6 规定，具备Symbol.iterator方法的对象（例如数组），即不用做任何处理，就可以使用for...of遍历执行，但只有Iterator对象才可以有next方法。
```js
var arr = [1, 2];
var Itr = arr[Symbol.iterator]();
Itr.next(); 
//{value: 1, done: false}

Itr.next(); 
//{value: 2, done: false}

Itr.next(); 
//{value: undefined, done: true}
```
### yield 表达式
由于 Generator 函数返回的遍历器对象，只有调用next方法才会遍历下一个内部状态，所以其实提供了一种可以暂停执行的函数。yield表达式就是暂停标志。

1.  遇到 `yeild` 表达式暂停，返回`yeild`后面的表达式的值。
2.  直到`return`语句为止
3.  如果没有`return`，返回`undefined`

> `yield`表达式只能用在 Generator 函数里面,Generator 函数可以不用`yield`表达式，这时就变成了一个单纯的暂缓执行函数。

### Symbol.iterator
执行 `Symbol.iterator`方法返回时遍历器对象。
```js
function* gen(){}
// 使用Generator 生成的遍历器对象
var g = gen();
// 返回自己
g[Symbol.iterator]() === g
// true
```

### next

`yield` 表达式返回值为undefined。如果next有参数，该参数就会被当作上一个yield表达式的返回值。
```js
function* f() {
  for(var i = 0; true; i++) {
    var reset = yield i;
    if(reset) { i = -1; }
  }
}

var g = f();

g.next() // { value: 0, done: false }
g.next() // { value: 1, done: false }
g.next(true) // { value: 0, done: false }
```
### throw
Generator 函数返回的遍历器对象，都有一个throw方法，可以在函数体外抛出错误，然后在 Generator 函数体内捕获。

1.  使用`catch`捕获时，Generator 函数内部没有部署try...catch代码块，那么throw方法抛出的错误，将被外部try...catch代码块捕获。
2.  如果在Generator 函数内部捕获，必须要执行一次next方法，等同于启动执行 Generator 函数的内部代码。
3.  `throw` 方法被捕获以后，会附带执行下一条yield表达式。

```js 
var g = function* () {
  try {
    yield;
  } catch (e) {
    console.log('内部捕获', e);
  }
};

var i = g();
i.next();

try {
  i.throw('a');
  i.throw('b');
} catch (e) {
  console.log('外部捕获', e);
}
// 内部捕获 a
// 外部捕获 b
```

### \_\_proto\_\_.return
return方法，可以返回给定的值，并且终结遍历 Generator 函数。
```js
function* gen() {
  yield 1;
  yield 2;
  yield 3;
}

var g = gen();

g.next()        // { value: 1, done: false }
g.return('foo') // { value: "foo", done: true }
g.next()        // { value: undefined, done: true }
```
### yield* 表达式
在 Generator 函数内部，调用另一个 Generator 函数。
* 遍历器对象
* 数组
* 字符串
```js
let read = (function* () {
  yield 'hello';
  yield* 'hello';
})();

read.next().value // "hello"
read.next().value // "h"
```
> 实际上，任何数据结构只要有 Iterator 接口，就可以被yield*遍历。
```js
var str = 'abc'
str.__proto__[Symbol.iterator]
```

### 应用
```js
function* asyncJob() {
  // ...其他代码
  var f = yield readFile(fileA);
  // ...其他代码
}
```

## Async / await
ES2017 标准引入了 async 函数，它就是 Generator 函数的语法糖。

```js
async function asyncJob() {
  // ...其他代码
  var f = await readFile(fileA);
  // ...其他代码
}
```
async函数对 Generator 函数的改进，返回值是 `Promise` 对象。
async函数内部return语句返回的值，会成为then方法回调函数的参数。
```js 
async function f() {
  throw new Error('出错了');
}

f().then(
  v => console.log(v),
  e => console.log(e)
)
// Error: 出错了
```
### await

1. await命令后面是一个 `Promise` 状态对象。如果不是，会被转成一个立即 `resolve` 的 `Promise` 对象。
2. await语句后面的 Promise 变为reject，那么整个async函数都会中断执行。
3. await放在try...catch结构里面或者await后面的 Promise 对象再跟一个catch方法，可以捕获错误，避免异步操作失败中断。
```js
async function f() {
  // 1
  try {
    await Promise.reject('出错了');
  } catch(e) {
  }
  // 2
  await Promise.reject('出错了').catch(e => console.log(e));
  return await Promise.resolve('hello world');
}

f()
.then(v => console.log(v))
// hello world
```

### 错误处理
```js
async function main() {
  try {
    const val1 = await firstStep();
    const val2 = await secondStep(val1);
    const val3 = await thirdStep(val1, val2);

    console.log('Final: ', val3);
  }
  catch (err) {
    console.error(err);
  }
}
```
异步互不依赖可以让它们同时触发

```js
async function main() {
  try {
    const first = firstStep();
    const second = secondStep();
    const third = thirdStep();
    
    const val1 = await first;
    const val2 = await second;
    const val3 = await third;
  }
  catch (err) {
    console.error(err);
  }
}
```
> await命令只能用在async函数之中，如果用在普通函数，就会报错。

### 原理
```js
function spawn(genF) {
  return new Promise(function(resolve, reject) {
    const gen = genF();
    function step(nextF) {
      let next;
      try {
        next = nextF();
      } catch(e) {
        return reject(e);
      }
      if(next.done) {
        return resolve(next.value);
      }
      Promise.resolve(next.value).then(function(v) {
        step(function() { return gen.next(v); });
      }, function(e) {
        step(function() { return gen.throw(e); });
      });
    }
    step(function() { return gen.next(undefined); });
  });
}
```

## 异步遍历器 
ES2018 引入了”异步遍历器“（Async Iterator），为异步操作提供原生的遍历器接口，即value和done这两个属性都是异步产生。

> 异步遍历器的最大的语法特点，就是调用遍历器的next方法，返回的是一个 Promise 对象。

```js
asyncIterator
  .next()
  .then(
    ({ value, done }) => /* ... */
  );
```

#### for await...of
用于遍历异步的 Iterator 接口。
```js
async function () {
  try {
    for await (const x of createRejectingIterable()) {
      console.log(x);
    }
  } catch (e) {
    console.error(e);
  }
}
```

### 异步Generator 函数

```js
async function* gen() {
  yield 'hello';
}
const genObj = gen();
genObj.next().then(x => console.log(x));
```
gen是一个异步 Generator 函数，执行后返回一个异步 Iterator 对象。对该对象调用next方法，返回一个 Promise 对象。

```js
function fetchRandom() {
  const url = 'https://www.random.org/decimal-fractions/'
    + '?num=1&dec=10&col=1&format=plain&rnd=new';
  return fetch(url);
}

async function* asyncGenerator() {
  console.log('Start');
  const result = await fetchRandom(); // (A)
  yield 'Result: ' + await result.text(); // (B)
  console.log('Done');
}

const ag = asyncGenerator();
ag.next().then(({value, done}) => {
  console.log(value);
})
```






















