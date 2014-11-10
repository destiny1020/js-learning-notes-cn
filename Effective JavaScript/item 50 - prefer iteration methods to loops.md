## 优先使用遍历方法而非循环 ##

在使用循环的时候，很容易违反DRY(Don't Repeat Yourself)原则。这是因为我们通常会选择复制粘贴的方法来避免手写一段段的循环语句。但是这样做回让代码中出现大量重复代码，开发人员也在没有意义地"重复造轮子"。更重要的是，在复制粘贴的时候很容易忽视循环中的那些细节，比如起始索引值，终止判断条件等。

比如以下的for循环就存在这个问题，假设n是集合对象的长度：

```js
for (var i = 0; i <= n; i++) { ... }
// 终止条件错误，应该是i < n
for (var i = 1; i < n; i++) { ... }
// 起始变量错误，应该是i = 0
for (var i = n; i >= 0; i--) { ... }
// 起始变量错误，应该是i = n - 1
for (var i = n - 1; i > 0; i--) { ... }
// 终止条件错误，应该是i >= 0
```

可见在循环的一些细节处理上很容易出错。而利用JavaScript提供的闭包(参见[Item 11](http://blog.csdn.net/dm_vincent/article/details/39079555))，可以将循环的细节给封装起来供重用。实际上，ES5就提供了一些方法来处理这一问题。其中的`Array.prototype.forEach`是最简单的一个。利用它，我们可以将循环这样写：

```js
// 使用for循环
for (var i = 0, n = players.length; i < n; i++) {
	players[i].score++;
}

// 使用forEach
players.forEach(function(p) {
	p.score++;
});
```

除了对集合对象进行遍历之外，另一种常见的模式是对原集合中的每个元素进行某种操作，然后得到一个新的集合，我们也可以利用forEach方法实现如下：

```js
// 使用for循环
var trimmed = [];
for (var i = 0, n = input.length; i < n; i++) {
	trimmed.push(input[i].trim());
}

// 使用forEach
var trimmed = [];
input.forEach(function(s) {
	trimmed.push(s.trim());
});
```

但是由于这种由将一个集合转换为另一个集合的模式十分常见，ES5也提供了`Array.prototype.map`方法用来让代码更加简单和优雅：

```js
var trimmed = input.map(function(s) {
	return s.trim();
});
```

另外，还有一种常见模式是对集合根据某种条件进行过滤，然后得到一个原集合的子集。ES5中提供了`Array.prototype.filter`来实现这一模式。该方法接受一个Predicate作为参数，它是一个返回true或者false的函数：返回true意味着该元素会被保留在新的集合中；返回false则意味着该元素不会出现在新集合中。比如，我们使用以下代码来对商品的价格进行过滤，仅保留价格在[min, max]区间的商品：

```js
listings.filter(function(listing) {
	return listing.price >= min && listing.price <= max;
});
```

当然，以上的方法是在支持ES5的环境中可用的。在其它环境中，我们有两种选择：
1. 使用第三方库，如underscore或者lodash，它们都提供了相当多的通用方法来操作对象和集合。
2. 根据需要自行定义。

比如，定义如下的方法来根据某个条件取得集合中前面的若干元素：

```js
function takeWhile(a, pred) {
	var result = [];
	for (var i = 0, n = a.length; i < n; i++) {
		if (!pred(a[i], i)) {
			break;
		}
		result[i] = a[i];
	}
	return result;
}

var prefix = takeWhile([1, 2, 4, 8, 16, 32], function(n) {
	return n < 10;
}); // [1, 2, 4, 8]
```

为了更好的重用该方法，我们可以将它定义在`Array.prototype`对象上，具体的影响可以参考Item 42。

```js
Array.prototype.takeWhile = function(pred) {
	var result = [];
	for (var i = 0, n = this.length; i < n; i++) {
		if (!pred(this[i], i)) {
			break;
		}
		result[i] = this[i];
	}
	return result;	
};

var prefix = [1, 2, 4, 8, 16, 32].takeWhile(function(n) {
	return n < 10;
}); // [1, 2, 4, 8]
```

只有一个场合使用循环会比使用遍历函数要好：需要使用break和continue的时候。
比如，当使用forEach来实现上面的takeWhile方法时就会有问题，在不满足predicate的时候应该如何实现呢？

```js
function takeWhile(a, pred) {
	var result = [];
	a.forEach(function(x, i) {
		if (!pred(x)) {
			// ?
		}
		result[i] = x;
	});
	return result;
}
```

我们可以使用一个内部的异常来进行判断，但是它同样有些笨拙和低效：

```js
function takeWhile(a, pred) {
	var result = [];
	var earlyExit = {}; // unique value signaling loop break
	try {
		a.forEach(function(x, i) {
			if (!pred(x)) {
				throw earlyExit;
			}
			result[i] = x;
		});
	} catch (e) {
		if (e !== earlyExit) { // only catch earlyExit
			throw e;
		}
	}
	return result;
}
```

可是使用forEach之后，代码甚至比使用它之前更加冗长。这显然是存在问题的。
对于这个问题，ES5提供了some和every方法用来处理存在提前终止的循环，它们的用法如下所示：

```js
[1, 10, 100].some(function(x) { return x > 5; }); // true
[1, 10, 100].some(function(x) { return x < 0; }); // false

[1, 2, 3, 4, 5].every(function(x) { return x > 0; }); // true
[1, 2, 3, 4, 5].every(function(x) { return x < 3; }); // false
```

这两个方法都是短路方法(Short-circuiting)：只要有任何一个元素在some方法的predicate中返回true，那么some就会返回；只有有任何一个元素在every方法的predicate中返回false，那么every方法也会返回false。

因此，takeWhile就可以实现如下：

```js
function takeWhile(a, pred) {
	var result = [];
	a.every(function(x, i) {
		if (!pred(x)) {
			return false; // break
		}
		result[i] = x;
		return true; // continue
	});
	return result;
}
```

实际上，这就是函数式编程的思想。在函数式编程中，你很少能够看见显式的for循环或者while循环。循环的细节都被很好地封装起来了。

值得一提的是，在Java 8中引入了函数式编程和Lambda表达式的概念。如果有兴趣了解，可以参考[这里](http://blog.csdn.net/dm_vincent/article/category/2648241)。

### 总结 ###
1. 使用遍历方法`Array.prototype.forEach`和`Array.prototype.map`来代替循环，从而让代码更加清晰可读。
2. 对于重复出现的循环，可以考虑将它们进行抽象。通过第三方提供的方法或者自己实现。
3. 显式的循环在一些场合下还是有用武之地的，相应的也可以使用some或者every方法。







