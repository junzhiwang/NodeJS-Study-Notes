在上一节中，我们实现了批量文件读取，即循环异步函数的并发和有序操作。
总结为：
	并发操作可以在Array.forEach 和 Array.map 中进行；
	有序操作则在循环中对Promise加入await关键字，意味着同步等待结果，并用async修饰整个函数；

由于需要实现在批量读取完成后回调的功能，例如提示操作完成，统计结果等，我们需要在每个循环中判断是否读取到了最后一个
文件，采用的方法是计数器。尽管它在NodeJS单线程环境下，不会造成由于多线程造成的变量读取冲突，但这种方法显然是丑陋且繁杂的。

在这里，我们引入[thenjs](https://www.npmjs.com/package/thenjs)库来实现批量异步操作的顺序和无序执行以及回调。

这一节我们将用网络请求代替文件读取来模拟批量的异步操作。

我们用[Rap](http://rapapi.org/org/index.do)来模拟http请求并得到自定义数据：
![利用Rap模拟JSON数据](http://img.blog.csdn.net/20170510114031307?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvd2FuZ2p1bnpoaTkz/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

![JSON数据](http://img.blog.csdn.net/20170510114430667?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvd2FuZ2p1bnpoaTkz/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

对于NodeJS端http请求的发送和获取，常用的库有`request.js` , `node-fetch`  等，笔者都实现了一遍，在这里以[node-fetch](https://www.npmjs.com/package/node-fetch) 为例来实现。

先贴代码再讲解：

thenjs实现事件队列的并发（无序调用）：
```javascript
import fetch from 'node-fetch';
import bodyParser from 'body-parser';
import thenjs from 'thenjs';
import request from "request";
import fs from 'fs';
const api = 'http://rapapi.org/mockjsdata/18728/fetchtest';
let requests = [];
for(let i = 0; i < 10; ++i){
	let obj = {};
	obj.url = api;
	obj.count = i + 1;
	requests.push(obj);
}

const batchRequst = (urls) => {
	thenjs.each(urls, (defer, obj) => {
		fetch(obj.url)
		.then(res => res.json())
		.then(res => {
			console.log(JSON.stringify(res) + ' ' + obj.count);
			defer(null, res);
		})
		.catch(err => defer(err, res));
	}).then((defer, result) => {
		console.log(result);
		console.log('Done...')
	}).fail((defer, err) => {
		console.log(err);
	})
}

batchRequst(requests);
console.log('Start fetching...')
```
输出：
```txt
Start fetching...
{"data":{"name":"Paul  Robinson","id":65133}} 4
{"data":{"name":"Margaret  Lee","id":70397}} 2
{"data":{"name":"Joseph  Davis","id":46583}} 6
{"data":{"name":"Kenneth  Johnson","id":51935}} 5
{"data":{"name":"Timothy  Garcia","id":46554}} 3
{"data":{"name":"Maria  Anderson","id":93948}} 10
{"data":{"name":"Elizabeth  Martin","id":19124}} 8
{"data":{"name":"Betty  Hall","id":14796}} 7
{"data":{"name":"Shirley  Robinson","id":92156}} 9
{"data":{"name":"Michelle  Hernandez","id":48647}} 1
[ { data: { name: 'Michelle  Hernandez', id: 48647 } },
  { data: { name: 'Margaret  Lee', id: 70397 } },
  { data: { name: 'Timothy  Garcia', id: 46554 } },
  { data: { name: 'Paul  Robinson', id: 65133 } },
  { data: { name: 'Kenneth  Johnson', id: 51935 } },
  { data: { name: 'Joseph  Davis', id: 46583 } },
  { data: { name: 'Betty  Hall', id: 14796 } },
  { data: { name: 'Elizabeth  Martin', id: 19124 } },
  { data: { name: 'Shirley  Robinson', id: 92156 } },
  { data: { name: 'Maria  Anderson', id: 93948 } } ]
Done...
Done...
```
将上述代码中的 `each` 改为 `eachSeries`，得到结果为：
```txt
Start fetching...
{"data":{"id":72083,"name":"Linda  Young"}} 1
{"data":{"id":33670,"name":"Nancy  Thomas"}} 2
{"data":{"id":75790,"name":"Jennifer  Moore"}} 3
{"data":{"id":82450,"name":"Carol  Thomas"}} 4
{"data":{"id":45270,"name":"Deborah  Garcia"}} 5
{"data":{"id":33547,"name":"Kevin  Walker"}} 6
{"data":{"id":57576,"name":"William  Walker"}} 7
{"data":{"id":86271,"name":"Richard  Harris"}} 8
{"data":{"id":45700,"name":"Helen  Perez"}} 9
{"data":{"id":59519,"name":"Maria  Davis"}} 10
[ { data: { id: 72083, name: 'Linda  Young' } },
  { data: { id: 33670, name: 'Nancy  Thomas' } },
  { data: { id: 75790, name: 'Jennifer  Moore' } },
  { data: { id: 82450, name: 'Carol  Thomas' } },
  { data: { id: 45270, name: 'Deborah  Garcia' } },
  { data: { id: 33547, name: 'Kevin  Walker' } },
  { data: { id: 57576, name: 'William  Walker' } },
  { data: { id: 86271, name: 'Richard  Harris' } },
  { data: { id: 45700, name: 'Helen  Perez' } },
  { data: { id: 59519, name: 'Maria  Davis' } } ]
Done...
```
如thenjs的官网介绍所说，它能将任何同步或异步回调函数转化为then的链式调用。它的主要特征如下：

 1. 可以像标准的 Promise 那样，把N多异步回调函数写成一个长长的 then 链，并且比 Promise 更简洁自然。因为如果使用标准 Promise 的 then 链，其中的异步函数都必须转换成 Promise，Thenjs 则无需转换，像使用 callback
    一样执行异步函数即可。

 2. 可以像 async那样实现同步或异步队列函数，并且比 async 更方便。因为 async 的队列是一个个独立体，而 Thenjs 的队列在 Thenjs 链上，可形成链式调用。

 3.  强大的 Error 机制，可以捕捉任何同步或异步的异常错误，甚至是位于异步函数中的语法错误。并且捕捉的错误任君处置。

 4.  开启debug模式，可以把每一个then链运行结果输出到debug函数（未定义debug函数则用 console.log），方便调试。

对于第三点和第四点，笔者尚未在程序中体现。因此着重介绍第一点和第二点。对于事件队列的操作，本节主要应用了如下两个Api：

**Thenjs.each(array, iterator, [debug])**
将 array 中的值应用于 iterator 函数（同步或异步），并行执行。返回一个新的 Thenjs 对象。

**Thenjs.eachSeries(array, iterator, [debug])**
将 array 中的值应用于 iterator 函数（同步或异步），串行执行。返回一个新的 Thenjs 对象。

```javascript
thenjs.each(urls, (defer, obj) => {
		fetch(obj.url)
		.then(res => res.json())
		.then(res => {
			console.log(JSON.stringify(res) + ' ' + obj.count);
			defer(null, res);
		})
		.catch(err => defer(err, res));
	}).then((defer, result) => {
		console.log(result);
		console.log('Done...')
	}).fail((defer, err) => {
		console.log(err);
	})
```

一般来说[debug]参数不常用。**iterator**是用户自定义的对每个数组元素执行的函数，一般形式为 `function(defer, value)`。

 在`each`或`eachSeries`函数中使用了`fetch`这一异步获取网络数据的操作，并在**then**函数中捕获http请求的结果，显示，并调用`defer`函数。对于一般形如 `asyncfunction(arg, (err, data) => {})` 的NodeJS异步函数来说，只要在回调函数中调用`defer`函数即可。

  `defer`为一个回调函数，形式一般为 `function(err,  res)` 该函数返回一个thenjs对象供下一步操作，这也是thenjs能实现链式调用的关键步骤。若无错误，则第一个参数为null；`res`则表示批量请求结束后传入回调函数的数据。

 第二个.then函数，传入的是形如`(defer, result) => {}`的函数，result收集了在`defer(err, res)`中传入的结果并组装成数列。无论是同步还是异步(eachSeries or each)的形式，得到数组的排列顺序都和原始数组的请求数据一一对应。读者可观察打印出的`result`来得到上述结论。

归纳起来，thenjs实现批量异步请求的函数模板为：
```javascript
thenjs.[each|eachSeries](array, (defer, element) => {
	// implement async funtion here
	// callback: defer(err||null, res)
}).then((defer, result) => {
	// to do something
}.fail((defer, err) => {
	// catch err
})
```
一般来说，并发比顺序执行的效率高很多，且应用也更广，这也是NodeJS占优势的地方。由于许多服务器的限流，如果发送的http请求并发数量过大，可能会被服务器端禁止访问，这涉及到爬虫和反爬虫的策略，在此无法细数。但一个较为直接的解决方案是采用 `setInterval(callback, millisecond)` 方法向服务器端定时发送请求，在这里也不再赘述。

对于并发执行，也可采用 [Promise.all](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/all)等方法在实现，读者可自行研读。

自此，三节有关于NodeJS异步回调实现的博客已经全部完成。

从单个的异步回调，到批量请求的并发及有序实现，以文件读取和http请求为例进行了实现并讲解，希望我的经验能帮助到大家。

由于笔者理工科出生，且这是我第一次发技术类博客，可能语言文字组织的不够严密，望各位博友多提宝贵意见，对于文章中出现的错误，也请各位斧正。
