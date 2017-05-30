在上一节中我们介绍了NodeJS基础的异步回调实现方法，实现了异步的递归函数摆脱了‘回调地狱’。最后引入了Promise 特性将异步回调实现地更加优雅。
然而基于Promise的链式调用方法目前看似只能实现硬编码。基于上一节的问题，我们有一个文件列表，需要顺次读取其中的内容并打印。在这里我们引入es7的 Async 和 Await 关键字。
先实现一个简单的例子，在暂停5秒后在控制台输出`end`
```javascript
const sleep = (time) => {
    return new Promise((resolve, reject) => {
        setTimeout(() => {
            resolve();
        }, time);
    })
};
const start = async () => {
    console.log('start');
    await sleep(5000);
    console.log('end');
};
start();
```
在这里 await 关键字必须修饰 Promise对象，且必须在async 修饰的函数内。这里体现了外异步，内同步的思想。即start函数整体是异步的，但其函数内部的过程是顺序进行的，即内同步。

基于上一节的问题，我们有一个文件列表，需要顺次读取其中的内容并打印。在这里用 async 和 await 来实现
```javascript
import fs from 'fs';
let fsList =['/a.txt','/b.txt','/c.txt','/d.txt','/e.txt'];
const readFile2 = (path) => {
	return new Promise((resolve, reject) => {
		fs.readFile(__dirname + path, 'utf-8', (err, data) => {
			if(!err) resolve(data);
			else reject(err);
		})
	})
}
const readFileList2 = async (fsList) => {
	for(let i = 0; i < fsList.length; ++i){
		console.log(await readFile2(fsList[i]);
	}
	console.log('Done...');
}
console.log('Start reading');
```
输出：
```txt
Start reading
aaa
bbb
ccc
ddd
eee
Done...
```
看到结果，函数整体为异步，因为 Start reading 先输出，
然后内部按照同步的方式依次输出。

由于读取的文件有大有小，我们不希望因为一个文件很大而阻塞整体读取的进程，因此如果不关心文件打印的顺序，而希望其以最快的方式打印出来。假设有 3个文件 (a, b, c)，读取它们的时间分别为 (ta, tb, tc)，则同步读取并打印的事件为 ta + tb + tc, 而异步打印的事件则可接近于 max(ta, tb, tc)。
因此对于IO密集型服务，网络请求数越大，则异步调用整体节省的更多，NodeJS异步回调的优势就体现出来了。

那么如何实现内部的并发调用呢？首先，如果要实现一个对数组的异步操作， Array.forEach和Array.map 方法都是不错的选择。下面的程序将数组 (值为0到10000ms的随机分布) 作为 setTimeOut等待时间的参数，依次打印出10个等待时间, 在打印结束后调用resolve输出Done。

```javascript
let t = [];
for(let i = 0; i < 10; ++i){
	t.push(Math.random(i) * 10000);
}
let start = (callback) => {
	return new Promise((resolve, reject) => {
		let i = 0;
		t.map((time, index, arr) => {
				setTimeout(() => {
				callback(`this is ${index + 1}s value of t, have waited ${time} ms`);
				++i;
				if(i >= 10) resolve('Done');
				if(i >= 11) reject('an error occurs');
			}, time);
		});
	})
}
start(time => console.log(time))
.then(t => console.log(t))
.catch(err => console.log(err));
console.log('Start counting');
```
输出结果为：
```txt
Start counting
858.5494470929777
1482.6545923241151
1699.61125447446
2207.0929478221115
2906.0818746170435
3986.756119433945
4298.932567413627
5385.720057993297
7303.359191124267
8855.286289748638
Done
```
由于其内部的异步方式，运行总时间为 max(t[])  < 10s。
注意到程序终止（到达最大等待时间）时输出Done的判断条件，在内部 `let i = 0` 声明一个计数值，每次+1，判断到达最大值时退出。

Demo试验成功后，回归到原问题，将该模板应用于readFile函数上，就能轻易实现内部的并发调用：

```javascript
import fs from 'fs';
const readFileListAsync = (fsList, callback) => {
	return new Promise((resolve, reject) => {
		let i = 0;
		fsList.map((file, index, fileList) => {
			fs.readFile(__dirname + file, 'utf-8', (err, data) => {
				++i;
				if(err) reject(err);
				if(data) callback(data);
				if(i === fsList.length) {
					resolve('Done');
					return;
				}
				if(i > fsList.length) reject(new Error('an error occurs'));
			})
		})
	})
}
readFileListAsync(fsList, data => console.log(data))
.then(data => console.log(data))
.catch(err => console.log(err));
console.log('Start reading');

```
输出：
```txt
Start reading
aaa
ccc
bbb
eee
ddd
Done
```
由于其异步机制，每次输出结果的顺序将不固定。

到此我们以`setTimeOut` 及  `readFile` 函数为例，实现了对数组元素循环进行异步操作的两种方式，以 async &  await 为关键字实现的有序调用和以 Array.forEach 或 Array.map 实现的并发调用。

那么是否存在更为优雅的，甚至抛弃 `Promise` 以及 `async&await` 关键字来实现这两种调用的方式呢，
在下一节中，笔者将采用then.js这个库来实现。
