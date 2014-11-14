JavaScript处理并发事件的机制是十分友好和强大的，它结合了事件队列(Event Queue)/事件循环并发(Event-loop Concurrency)和一套异步调用API。这因为这一点，JavaScript不仅可以在浏览器环境中运行，还可以在桌面应用和服务器应用中运行，如Node.js。

令人奇怪的是，ECMAScript标准时至今日对并发这个问题还是只字未提。所以以下提到的各种并发方面的注意事项只是基于JavaScript本身语言特性总结得到的。

JavaScript程序围绕事件进行组织，比如用户点击按钮，输入了文字，触摸了屏幕。也可以是应用本身设置的定时器，异步调用返回了数据等。在其它程序中，我们经常会写下面这种代码：

```js
var text = downloadSync("http://example.com/file.txt");
console.log(text);
```

以上的`downloadSync`是一个同步(Synchronous)的方法，也可以被称为阻塞(Blocking)的方法。在该方法运行期间，程序的其它部分也无法执行。而在进行文件下载的过程中，程序其它部分往往是能够正常工作的，所以一些编程语言中会提供多线程的相关API供开发人员使用。

在JavaScript中，绝大多数和I/O相关的操作都是异步，非阻塞的(Asynchrounous, Non-Blocking)。相比于阻塞当前线程来等待运行结果，JavaScript需要开发人员提供一个回调函数(Callback)来指定应该如何处理在将来返回的结果。可以参考[Item 19](http://blog.csdn.net/dm_vincent/article/details/39288847)了解更多关于回调函数的特点。

```js
downloadAsync("http://example.com/file.txt", function(text) {
	console.log(text);
});
```

在调用了`downloadAsync`方法后程序不会阻塞和等待，而会继续执行后续的代码。传入的回调函数则会在内部被注册。当下载行为结束，得到了相应的数据之后，注册的回调函数会在**合适的时候**被系统所调用，需要的数据会被作为参数传入其中。

为什么说是**合适的时候**？这是因为JavaScript本身提供了一种"运行到完成(Run-to-completion)"的保证：在任意一个时间点，当前正在执行的代码会一直执行直到它结束。实际上，系统维护了一个发生了的事件的队列，然后一个个地执行它们注册的回调函数。

下图是JavaScript分别在客户端应用(a)和服务端应用(b)上，事件队列的示意图：

![](https://github.com/destiny1020/js-learning-notes-cn/blob/master/Effective%20JavaScript/images/61-1.PNG)

最近发生的事件会被放到队列的头部，即图中的顶部。然后JavaScript会按照添加的顺序从队列中拿出对应事件注册的回调函数并执行。

"运行到完成(Run-to-completion)"这种机制的好处在于，当代码运行时，当前代码对整个应用有完全的控制权，不需要担心与此同时有别的线程会改变应用的状态。

但是它的缺点也很明显，正因为当前运行的代码拥有绝对的控制权，如果这段代码是在交互性较强的场合如浏览器中使用的话，就需要注意不要轻易的阻塞。否则会造成浏览器无响应的现象，造成用户体验的降低。

JavaScript并发中最重要的一条规则就是：绝不要在事件队列中间使用任何的阻塞API。

如果应用的主要事件队列不受影响的话，那么使用阻塞操作也不会产生很大的问题。比如，部分Web环境中提供了Worker API，它用来帮助执行并行计算。和传统线程不一样的是，workers运行在一个完全被隔离的上下文环境中，和应用主线程的全局作用域和页面内容之间没有建立连接。因此workers不会对应用的主要事件队列产生影响。

### 总结 ###

1. 使用异步API结合回调函数完成昂贵操作的实现，来避免应用的阻塞。
2. JavaScript能够并行地接受事件，但是会使用一个事件队列来串行地对它们进行处理。
3. 在应用的事件队列中绝不要使用阻塞I/O。





