在一开始，理解异步程序的调用顺序会有些困难。比如，下面的程序中，starting会先被打印出来，然后才是finished：

```js
downloadAsync("file.txt", function(file) {
	console.log("finished");
});
console.log("starting");
```

`downloadAsync`方法在执行之后会立即返回，它只是为下载这个行为注册了一个回调函数而已。
由于JavaScript"运行到完成(Run-to-completion)"的特点，当前执行的代码不会被中断，因此会先打印出starting。直到下载完成之后，JavaScript从事件队列中拿到下载对应的回调函数并执行后，finished才会被输出。

一个便于理解调用顺序的思考方式是：将异步API想成是初始化(Initializing)，而不是执行(Performing)了某种行为。用这种思考方式，上面的代码就容易理解了，`downloadAsync`方法仅仅是在初始化下载这一行为，实际上没有做出实际的下载动作。

那么对于一些依赖于执行顺序的行为，比如在下载发生前，我们首先需要查询下载的目标URL。按照下面的实现方式是不行的：

```js
db.lookupAsync("url", function(url) {
	// ?
});
downloadAsync(url, function(text) { // error: url is not bound
	console.log("contents of " + url + ": " + text);
});
```

在调用`downloadAsync`方法的时候，url指向的实际上是undefined。这也很容易理解，`lookupAsync`方法只是初始化了查询这一行为，实际的动作还没发生，因此作为查询结果的url是不可用的。

最直接的解决方案是使用嵌套(Nesting)，通过闭包的特性(关于闭包，请参考[Item 11](http://blog.csdn.net/dm_vincent/article/details/39079555))：

```js
db.lookupAsync("url", function(url) {
	downloadAsync(url, function(text) {
		console.log("contents of " + url + ": " + text);
	});
});
```

现在我们把下载行为放到了查询行为注册的回调函数中，通过闭包的性质，在下载方法中就可以访问到查询方法得到的结果url。

使用这种嵌套的方式来规定异步调用的顺序是很简单的，可是随着调用的数量增加，代码的也会变得难以阅读，就像下面这样：

```js
db.lookupAsync("url", function(url) {
	downloadAsync(url, function(file) {
		downloadAsync("a.txt", function(a) {
			downloadAsync("b.txt", function(b) {
				downloadAsync("c.txt", function(c) {
					// ...
				});
			});
		});
	});
});
```

一种防止过度嵌套的方法是将嵌套的回调函数声明为命名函数，然后将需要的数据作为参数传入：

```js
db.lookupAsync("url", downloadURL);

function downloadURL(url) {
	downloadAsync(url, function(text) { // still nested
		showContents(url, text);
	});
}

function showContents(url, text) {
	console.log("contents of " + url + ": " + text);
}
```

为了让`downloadAsync`方法的回调函数能够利用url，因此上述代码中还是出现了嵌套的现象。这一嵌套可以借助bind方法进一步消除(关于bind方法，可以参考[Item 25](http://blog.csdn.net/dm_vincent/article/details/39472817)):

```js
db.lookupAsync("url", downloadURL);

function downloadURL(url) {
	downloadAsync(url, showContents.bind(null, url));
}

function showContents(url, text) {
	console.log("contents of " + url + ": " + text);
}
```

当然使用bind方法确实能够消除过多的嵌套，可是它的问题就是需要声明一些命名函数。当这些函数的数量过多时，也会带来不小的干扰。所以，在使用嵌套和使用bind方法之间通常需要谋求一种平衡，可以将重要的步骤使用命名函数的方式，而其他的步骤还是使用嵌套：

```js
db.lookupAsync("url", function(url) {
	downloadURLAndFiles(url);
});

function downloadURLAndFiles(url) {
	downloadAsync(url, downloadFiles.bind(null, url));
}

function downloadFiles(url, file) {
	downloadAsync("a.txt", function(a) {
		downloadAsync("b.txt", function(b) {
			downloadAsync("c.txt", function(c) {
				// ...
			});
		});
	});
}
```

对于`downloadFiles`方法，可以使用抽象程度更高的方法(该方法的实现会在Item 66中进行介绍)。将下载的文件保存到一个数组中：

```js
function downloadFiles(url, file) {
	downloadAllAsync(["a.txt", "b.txt", "c.txt"], function(all) {
		var a = all[0], b = all[1], c = all[2];
		// ...
	});
}
```

`downloadAllAsync`方法能够并行地下载多个文件。同时适当的利用嵌套也保证了程序的执行顺序。
在Item 68中，会介绍如何封装程序的执行流程，让流程控制更加简单。

### 总结 ###
1. 使用嵌套或者命名回调函数的方式来控制异步行为的执行顺序。
2. 在嵌套和命名回调函数这两种方式中谋求一种平衡。
3. 能够并行处理的任务，就不要将它们串行化。
















