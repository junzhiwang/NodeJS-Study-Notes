
众所周知，NodeJS具有的单线程，事件驱动，异步非阻塞IO模型，使得其在IO密集型程序，尤其是大型的Web服务中占有很大的优势。

下面就来谈谈几种NodeJS异步回调的实现。
最常规的一种是：

```javascript
import fs from 'fs';
fs.readFile(__dirname + '/a.txt', 'utf-8', (err, data) => {
	if(!err){
		console.log(data);
		console.log('Finish reading');
	} else console.log(err);
})
console.log('Start reading');
```
输出：
```txt
Start reading
aaa
Finish reading
```

如果用户想自定义回调函数代替 `console.log(data)`
可以再封装一层，将callback作为回调函数传入内部：
```javascript
import fs from 'fs';
const readFile = (path, callback) =>{
	fs.readFile(__dirname + path, 'utf-8', (err, data) => {
		if(!err){
			callback(data);
			console.log('Finish reading');
		} else console.log(err);
	})
}
readFile('/a.txt', (data) => { console.log(`The content of file "a.txt" is: ${data}`)});
console.log('Start reading');
```
输出：
```txt
Start reading
The content of file "a.txt" is aaa
Finish reading
```

NodeJS将几乎所有的库函数实现为如下形式的异步回调机制

``` javascript
asyncFunc(arg0, arg1, ... (err, data) => {
	if(!err) callback(data);
})
```
此时，假设我们以readFile函数为例，它实现了异步读取本地文件并在读取到内容后回调的功能。假如我们已经获取了当前路径某个文件夹下的3个文件[a.txt, b.txt, c.txt]，需将顺次遍历和打印其内容，首先想到的是暴力和丑陋的多层回调方法如下：

```javascript
import fs from 'fs';
const read3Files = (callback) => {
	fs.readFile(__dirname + '/a.txt', 'utf-8', (err, data) => {
	if(!err){
		callback('a.txt', data);
		fs.readFile(__dirname + '/b.txt', 'utf-8', (err, data) => {
			if(!err){
				callback('b.txt', data);
				fs.readFile(__dirname + '/c.txt', 'utf-8', (err, data) => {
					if(!err){					
						callback('c.txt', data);
						console.log(`Done...`);
					} else console.log(err);
				})
			} else console.log(err);
		})
	} else console.log(err);
})
}
read3Files( (file, data) => { console.log(`The content of file ${file} is: ${data}`)});
console.log('Start reading');
```
输出：
```
Start reading
The content of file a.txt is: aaa
The content of file b.txt is: bbb
The content of file c.txt is: ccc
Done...
```
这就是臭名昭著的回调地狱了，回调函数层层嵌套，不仅缺乏美感，而且给Debug造成很大的困难。

而且这么做有个很大的弊端，读取的文件是硬编码的，如果给的文件列表数个很长的数组，就无法这么做了。

观察到上述代码的特性，我们可以将该其写成递归函数的形式如下，函数细节不再赘述，在当前文件夹下有5个txt文件如下，代码采用递归形式依次读取其内容。

```javascript
import fs from 'fs';
let fsList = ['/a.txt','/b.txt','/c.txt','/d.txt','/e.txt'];
const readFileList = (fsList, i, callback, err, data) => {
	if(i > fsList.length) {
		console.log(`Done...`);
		return;
	}
	if(err) {
		console.log(err);
		return;
	} else if(data){
		callback(fsList, i - 1, data);
	}
	fs.readFile(__dirname + fsList[i], 'utf-8', (err, data) => {
		return readFileList(fsList, i + 1, callback, err, data);
	});
}
readFileList(fsList, 0, (fsList, i, data) => {console.log(`The content of file ${fsList[i]} is: ${data}`)}, null, null);
```
输出：
```
Start reading
The content of file a.txt is: aaa
The content of file b.txt is: bbb
The content of file c.txt is: ccc
The content of file /d.txt is: ddd
The content of file /e.txt is: eee
Done...
```

这种方法看似简便了许多，但我们不想将时间浪费在自己实现递归函数这种难以Debug又容易产生各种错误的工作上，那么有没有
更为优雅的实现方式呢，在这里我们引入ES6和ES7的新特性，`Promise`和`Async, Await`关键词。

ES6 原生提供了 Promise 对象。所谓 Promise 对象，就是代表了某个未来才会知道结果的事件（通常是一个异步操作），并且这个事件提供统一的 API，可供进一步处理。

有了 Promise 对象，就可以将异步操作以同步操作的流程表达出来，避免了层层嵌套的回调函数。此外，Promise 对象提供的接口，使得控制异步操作更加容易。

``` javascript
let promise = new Promise((resolve, reject) => {
  if (/* 异步操作成功 */){
    resolve(value);
  } else {
    reject(err);
  }
});

promise.then(value => {
  // success
}.catch(err => {
  // fail
})
```

因此对于readFile这个函数，或是你需要在程序中实现‘整体异步，内部链式调用‘的异步函数，我们可以将其封装为一个Promise对象，并将自定义callback传入then和catch函数：

```javascript
import fs from 'fs';
const readFile2 = (path) => {
	return new Promise((resolve, reject) => {
		fs.readFile(__dirname + path, 'utf-8', (err, data) => {
			if(!err) resolve(data);
			else reject(err);
		})
	})
}
readFile2('/a.txt').then(data => console.log(data)).catch(err => console.log(err));

##使用链式调用读取多个文件：

readFile2('/a.txt').then(data => {
	console.log(data);
	return readFile2('/b.txt');
}).then(data => {
	console.log(data);
	return readFile2('/c.txt');
}).then(data => {
	console.log(data);
	return readFile2('/d.txt');
}).then(data => {
	console.log(data);
	console.log(`Done...`);
}).catch(err => console.log(err));

```
输出：
```txt
Start reading
aaa
bbb
ccc
ddd
Done...
```
封装Promise及链式调用的方法看上去很优雅，但其缺点也是硬编码，无法实现读取一个列表中的文件，
笔者暂时还没想到用递归调用的形式实现上述的链式函数。

在下一节中，笔者将引入 Async 和 Await 关键字来实现顺次以及非顺次读取一个FileList中的文件内容。
