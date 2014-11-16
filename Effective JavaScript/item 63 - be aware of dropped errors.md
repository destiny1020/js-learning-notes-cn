异常处理是异步编程的一个难点。在同步的代码中，异常能够很容易地通过try catch语句来完成：

```js
try {
	f();
	g();
	h();
} catch (e) {
	// handle any error that occurred...
}
```

但是在异步代码中，使用一个try代码块将所有可能出现的异常都包括在内是不现实的。实际上，异步API设置不能抛出异常。因为当异常发生时，通常已经没有执行上下文供它抛出了。所有，在异步API中通常会使用特殊的参数或者错误回调函数来表示异常信息。比如，在[Item 61](http://blog.csdn.net/dm_vincent/article/details/41116863)中提到的下载文件的异常处理可以如下进行：

```js
downloadAsync("http://example.com/file.txt", function(text) {
	console.log("File contents: " + text);
}, function(error) {
	console.log("Error: " + error);
});
```

可见`downloadAsync`方法不仅接受了一个作为成功返回响应时的回调函数，同时还接受了一个异常回调函数。那么以此类推，在下载多个文件时的异常处理是这样的：

```js
downloadAsync("a.txt", function(a) {
	downloadAsync("b.txt", function(b) {
		downloadAsync("c.txt", function(c) {
			console.log("Contents: " + a + b + c);
		}, function(error) {
			console.log("Error: " + error);
		});
	}, function(error) { // repeated error-handling logic
		console.log("Error: " + error);
	});
}, function(error) { // repeated error-handling logic
	console.log("Error: " + error);
});
```

第一眼看上去，上述代码有些乱。所以我们可以考虑使用命名回调函数的方式将错误处理抽取出来：

```js
function onError(error) {
	console.log("Error: " + error);
}

downloadAsync("a.txt", function(a) {
	downloadAsync("b.txt", function(b) {
		downloadAsync("c.txt", function(c) {
			console.log("Contents: " + a + b + c);
		}, onError);
	}, onError);
}, onError);
```

在Node.js中，还有一种异常处理API的风格。它会将错误信息作为回调函数的第一个参数，如果没有发生错误，那么该参数的值是null，undefine之类的falsy值。使用这种方式进行异常处理如下所示：

```js
function onError(error) {
	console.log("Error: " + error);
}

downloadAsync("a.txt", function(error, a) {
	if (error) {
		onError(error);
		return;
	}
	downloadAsync("b.txt", function(error, b) {
		// duplicated error-checking logic
		if (error) {
			onError(error);
			return;
		}
		downloadAsync(url3, function(error, c) {
			// duplicated error-checking logic
			if (error) {
				onError(error);
				return;
			}
			console.log("Contents: " + a + b + c);
		});
	});
});
```

为了增加代码的可读性，很多开发人员会省略用于判断是否需要异常处理的if语句块的大括号：

```js
function onError(error) {
	console.log("Error: " + error);
}

downloadAsync("a.txt", function(error, a) {
	if (error) return onError(error);
	downloadAsync("b.txt", function(error, b) {
		if (error) return onError(error);
		downloadAsync(url3, function(error, c) {
			if (error) return onError(error);
			console.log("Contents: " + a + b + c);
		});
	});
});
```

在使用异步API时，忘记对异常进行处理是经常犯的错误。这会让异常信息被忽略掉，从而导致比较糟糕的用户体验。所以在使用异步API时，切记对异常情况进行处理。我想这也是为何在Node.js中，异常信息会被当做回调函数的第一个参数，让开发人员不得不面对它。

### 总结 ###
1. 避免对异常处理代码进行复制粘贴，将它作为共享的函数是更好的选择。
2. 确保对异常情况进行处理，避免对异常的忽略。


