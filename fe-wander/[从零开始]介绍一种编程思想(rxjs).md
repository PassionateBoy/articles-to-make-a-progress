> 文章内容由一次内部分享会总结而来. 本文还配套有一个[示例工程](https://github.com/WanderHuang/react-rxjs-demo)，讲解在react中应用rxjs的几个要点。目前包含三个示例。
> * 示例
>   * 异步事件处理，流的合并和拆分
>   * 实现一个dom跟随的示例
>   * 实现兄弟组件通信
> * 作者 wander
> * 时间 2019/10/11
> * 时长 2h

## 介绍一种编程思想：rxjs

## 概要
* [$ 是什么？(思想原理)](#1)
* [$ 为什么？(什么场景)](#2)
* [$ 怎么用？(优雅使用)](#3)
* [$ Q & A](#4)

<h2 id="1">是什么</h2>


### 官方简介
> REACTIVE EXTENSIONS LIBRARY FOR JAVASCRIPT。RxJS is a library for reactive programming using Observables, to make it easier to compose asynchronous or callback-based code.

> 译文：用于 JavaScript 的 ReactiveX 库。RxJS 是使用 Observables 的响应式编程的库，它使编写异步或基于回调的代码更容易。

关键词：`reactive programming` & `Observables`

> Notes: 
> * 函数式：简介内没有提及，但是函数式编码风格确实是与rx不可分离的，利用函数式编码，把很多功能细节都封装了，业务代码更加简洁
> * 响应式：不奢求数据怎么来的，只需要对每次数据的变化作出及时的响应即可
> * 流：把事件和数据流动看作是软件生命周期所发生的，任意事件都可以发生在时间线的某一点

### 观点
从`rx`的角度看问题，数据和事件都是可以`函数式`(就像`lodash`)处理的，数据的产生和使用也是`响应式`(就像`react`，`render`不用关心数据怎么来的，只关心每次拿到值能够更新视图)的。可以视为`函数响应式`的范式。

### 写法

```JavaScript
// 基本范式
// Observable: 流。时间线概念，观察者在各个时刻收到流反馈的数据
// pipe: 管道。所有对流数据的操作都发生在管道内
// operator: 操作。对数据的各种操作，包括但不限于过滤、筛选、合并、延迟等。
// subscribe: 订阅。流的订阅操作，把观察者加到流的订阅清单内，便于推送数据
// observer: 观察者. 可以订阅流的数据，含有三种类型操作next error complete，分别对应流数据操作、流出错、流完结。
// subject: 主体。既是一个流，也是一个观察者。因此可以产生数据，也可以被订阅来推送数据，实际就是多播的概念。
Observable.pipe(operator1, operator2).subscribe(observer/Subject);
Subject.next(data);
Subject.subscribe(observer);

// 创建一个方法：根据给出的url发送ajax请求，并在错误的时候能够自动重试
const retryAjax = (url, payload, times = 3) => (whenTap, whenError) => of(url).pipe(
  // 
  mergeMap(url => ajax(url, payload)),
  // 重试
  retryWhen(err$ =>
    // 是的，在流的世界，异常也是一个流
    err$.pipe(
      // 副作用操作
      tap(val => {
        if (isType(whenTap, 'Function')) {
          whenTap(val.message);
        }
      }),
      // 扫描(累加) 时间线上的reduce
      scan((acc, val) => {
        if (acc > times) {
          throw new Error(`${val}, retry ${times} times, still error.`);
        } else {
          return acc + 1;
        }
      }, 1),
      // 获取未捕获的异常
      catchError(err => {
        if (isType(whenError, 'Function')) {
          whenError(err.message, err);
        }
        return of(err);
      })
    )
  )
)

// 得到一个ajax流
const stream$ = retryAjax('/fetch/something.htm', {value: 1})();
// 订阅一个流，并获得这个订阅本身，后续可以使用subscription.unsubscribe()来取消订阅
const subscription = stream$.subscribe(next => console.log('get values', next));
```

### 范式之争，如何去描述客观问题？

* 过程式编程：程序调用、流程控制。代表：原生`js`，`jquery`
  - 写的时候很清晰，按预定流程编写代码
  - 重复编写业务流程，复用性低
  - 代码量庞大，维护困难
* 函数式编程：函数序列、无状态。代表：`lodash`
  - 声明式函数，见名知义。如map -> 映射，reduce -> 减少，filter -> 过滤
  - 细节封装，流程复用
  - 无副作用，数值可预期、代码可测试
* 面向对象：对象组合、相互作用。代表：封装一个`emitter`
  - 封装数据细节，面向行为
  - 行为组合，代码量庞大
* 响应式编程：数据流、变化传播。代表：`vue`，`react` .etc
  - 关心对数据的操作(业务)，不比关心数据如何产生和流转到终端(细节)
  - 流程更优雅，变化可预期

> 不同的编程思维，代表你看待客观问题的角度；不同问题也可能适用不同的范式，甚至不同的语言，偏向也不一样。

> Notes
> * 函数式编程由数学家们创造出来，已经是好几十年的概念了，在当前的编程世界中，越来越受到重视
> * 响应式编程提倡业务开发者只关心数据本身，而不用去关注数据产生和流动的细节，前端三大框架都是这种范式的实践者。(封装dom操作，开发者更关心UI展示和数据逻辑)
> * RX = 函数式 + 响应式。思想是`响应式`的思维，实现方式是很`函数式`。
> 理解以上三个点，对端正rx的学习很重要。我们并不是为了追求`新技术`才去尝试rx，因为它背后的思想并不是`新的`。之所以采用rx，是为了更好地解决业务开发的那些`痛点`问题。

### rx - 函数响应式编程 - 万物都是流

> 万物视为`流(stream)`，且`流`的相互作用可以生成新的`流`，我们可以从`流`中订阅数据，也可以取消订阅。

```JavaScript
// 如果`b` `c`分别代表了两个不同事件(持续性)的结果，那`a`是否需要每次去计算(过程式)?
let a = b + c
```

* 流的操作，可视化：[弹珠图](https://rxmarbles.com/)

![弹珠图](./fe-wander/assets/marble-merge.png)
> 在时间线上，对数据的组合、拆分、映射等操作都变得更加简单

> Notes:
> 按流的概念来看，`a = b + c`中的`a`，并不需要去关注另外两个业务中的细节，它只关注于自己，因为`b`和`c`的每个变动都会自动`流动`到`a`身上

### 基本概念

* 核心原理
  - 观察者模式(订阅发布)：观察者(observer)订阅(subscribe)到流(Observable)
  - 迭代器模式(next)：迭代(iterator)发布(publish或next)数据
  - 广播(broadcast)和多播(multicast)

* [核心概念](https://cn.rx.js.org/manual/overview.html)
  - Observable (可观察对象): 表示一个概念，这个概念是一个可调用的未来值或事件的集合。
  - Observer (观察者): 一个回调函数的集合，它知道如何去监听由 Observable 提供的值。
  - Subscription (订阅): 表示 Observable 的执行，主要用于取消 Observable 的执行。
  - Operators (操作符): 采用函数式编程风格的纯函数 (pure function)，使用像 map、filter、concat、flatMap 等这样的操作符来处理集合。
  - Subject (主体): 相当于 EventEmitter，并且是将值或事件多路推送给多个 Observer 的唯一方式。
  - Schedulers (调度器): 用来控制并发并且是中央集权的调度员，允许我们在发生计算时进行协调，例如 setTimeout 或 requestAnimationFrame 或其他。

> Notes:
> 核心原理的三点需要各自下来再去仔细学习，本文不囊括这些实现细节。但是可以提一下
> * 观察者模式：核心在于几个概念，数据中心，订阅，发布，观察者
> * 迭代器模式：生成器、迭代推送
> * 多播：向多个对象扩展数据


<h2 id="2">为什么</h2>

### 先说好处

* 函数式的编码风格，结果可预期，代码可测试
* 异步流程控制代码量更少，降低维护成本
* 思维方式：`流`的世界
* 语言扩展好处：多语言实现`RX`，跨语言开发降低难度。已知就有java、python、cpp、dart、ruby、JavaScript

> Notes:
> 多语种支持确实是一种优势，比如你会写`rxjs`，那你要开发app，用`flutter`的时候，`rxdart`可以拿来即用。

### 再说缺点
* 上手难度较高，不是拿来即用，学习曲线比较陡峭
* 操作符较多，难免有抉择之难
* 全新的思维方式

### 异步处理方案

1. 朴素的事件监听、回调方式

```JavaScript
// 回调
ajax(url, callback);
// 回调地狱
function ajax(param, callback, callback2) {
  callback(callback2);
}

asyncCode(() => {
  asyncCode2(() => {
    asyncCode3() => {
      // do something
    }
  })
})
```

> Notes:
> 回调地狱的核心问题在于
> * 全局变量随处引用，数据来源未知性较大
> * 不够清晰，一个功能写几十上百行，维护起来也很不方便，特别是多人维护的时候；调试也很麻烦

2. 基于promise的异步处理

```JavaScript
// promise
ajax(url).then(data => console.log('get', data));
// 链式调用
ajax(url)
  .then()
  .then()
  .then()
  .catch()
```

> Notes:
> `promise`已经能够解决大多数异步场景的问题了，对各个部分业务的划分也比回调的方式好得多。
> * 但是仍然会有一些人不喜欢过长的链式调用，这可能让他难以选择新增功能应该加在哪里

3. 基于生成器的异步事件流处理

```JavaScript
// generator 异步获取多个值
const future = gen();
const a = future.next();
const b = future.next();
// 基于以上语法发展出了async/await语法糖
const a = await ajax();
// async函数内存在同步&异步代码，业务同样复杂
```

> Notes:
> 迭代器解决了对一个事件异步获取多个值的问题，但是需要主动调用。

4. 基于`流`的异步数据流解决方案(`流`本身不关心数据是否异步，因此同步数据流同样可以处理)

```JavaScript
const dom = document.querySelector('.event-element');
// 从事件中建立流
const event$ = from(dom, 'input');
// 管道中处理流
const value$ = event$.pipe(
  pluck('target'),
  map((target) => target.value)
);
// 订阅流 【业务代码】
value$.subscribe((value) => {
  this.setState({ value })
});
```

> Notes:
> 终极方案，解决了一个异步事件推送多值的问题。

### 事件处理方案对比

一个事件有其生产者(Producer)和消费者(Consumer)。
* 一个事件的消费有拉取(Pull)和推送(Push)两种方式，消费者主动获取值为Pull，消费者被动接收生产者提供的值为Push。
* 对于一个事件，待消费数据存在单值和多值之分。
-------------
|      | Single   | Multiple |
| ---- | -------- | -------- |
| Pull | Function | Generator|
| Push | Promise  | `Rx`     |

-------------
> `Rx`的编程方式，解决异步数据多值推送的问题。

### 多任务处理

一个简单的异步组合场景：触发A事件和触发B事件时，同时触发日志事件。

1. 通常，我们会这么写

```JavaScript
/** react内，我们通常这么写 */
async function a(e) {
  // do something...
  // 日志
  dispatch({type: 'LOG_HERE', payload: 'a finished'})
}
async function b(e) {
  // do something...
  // 日志
  dispatch({type: 'LOG_HERE', payload: 'b finished'})
}
```
2. 用流的思维来看 

```JavaScript
// 函数a和函数b代表两条异步数据流，而记录日志的行为，可以用另一个流来表示
const a$ = fromEvent(domA, 'click').pipe( // 管道操作
  pluck('target'),
  map(target => ([target.clientX, target.clientY])),
  // 映射为新的流；点击事件触发另一个流事件，并以该流事件作为返回值
  mergeMap(data => fromPromise(ajaxA(someUrl, data)))
);
const b$ = fromEvent(domB, 'click').pipe(
  pluck('target'),
  map(target => ([target.clientX, target.clientY])),
  mergeMap(data => fromPromise(ajaxB(otherUrl, data)))
);

// 实际的日志处理操作在这里
// 可以看到，`日志流`是不会把自己嵌入a或者b里面的，它只监听上游流的数据推送，然后生成自己的数据并推送给订阅自己的观察者。
// 合流
const log$ = a$.pipe(
  merge(b$)
);
// 订阅会返回一个订阅对象
const subscription = log$.subscribe(data => console.log(data));
// 一分钟后取消订阅
setTimeout(() => { subscription.unsubscribe() }, 1000 * 60)
```
### 其他特性

* 重复 `stream$.repeat(2)`
* 重试 `stream$.retry(3)`
* 累加 `stream$.scan(accumulatorFunction)`
* 取消 `subscription.unsubscribe()`
* 多播 `muticast(new Subject())` `publish`等
* 重播 `stream$.publishReplay(1).refCount()`
* cold & hot 流的冷热特性
* schedule 调度器。比如调度异步操作在`事件队列`的什么时候执行
* 除以上外，还有条件选择、过滤、合流、延迟等操作

> 有人说可以把它看作是处理异步数据流的`lodash`，但实际场景中，它只会表现得更强大

<h2 id="3">怎么用</h2>

### 图片请求

现在图片、视频类应用很多，这种应用往往会向后台发送大量请求。并且拿到二进制数据后往往会做预处理。
```plain
多个请求，分别调用handle、display方法及其后续预处理
A ----- promise ----- handle ----- display -----
B ----- promise ----- handle ----- display -----

------------------|合流|------------------

合流之后只需要从promise$中获取数据，执行handle、display方法即可
promise$ ----- A ----- B ----- C -----
observer ----- data ----- handle ----- display -----
```

> 业务只关心多个异步方法最终数据的，适合采用`流`的方式运行

### 推送消息处理

推送消息：业务方被动接收数据，并且在应用运行中可能持续存在，持续获取值。
消息处理：对接收的消息需要做筛选、过滤、业务分类、预处理等的操作。

前端目前能应用的场景
* websocket
* 视频流
* worker

> 上述示例拿到的数据往往需要较多的预处理，可以利用`流`的特性在管道(`pipe`)中处理，也可以分流为多业务分别处理

### 各子模块存在大量交互(同步、异步交错)

典型案例：游戏开发、富文本工具开发、`excel`等

> 目标业务不需要`命令式`地去调用其他业务，而是从其他业务`订阅`这种变化，并针对得到的数据作出响应的UI变动。

<h2 id="4">Q & A</h2>

* Q 区分生成器和迭代器的概念
* Q 如何理解冷热流(cold & hot)?
  * cold 流在订阅的时候，数据会从一开始生成，新订阅者可以获取到流上的所有数据
  * hot  流在订阅的时候，只是把观察者挂载到了已经存在的流上面，只能获取到订阅之后的数据，订阅之前由流产生的数据就不能得到
* Q 如何理解流的合并，以及高阶流？
  * 通过弹珠图来演示流的合并。
  * 高阶流实际上返回了一个新的流，内部实现是取消上游流的订阅，转而订阅到高阶流返回的流上面。rx内万物都是流。
* Q 如何实现组件内通信、兄弟组件通信、全局状态管理？
  * 是一个复杂的问题，参见工程实现。
  * 全局状态管理是能够实现，并可以无缝替换redux，但是必要性不高，redux已经很优雅了。

