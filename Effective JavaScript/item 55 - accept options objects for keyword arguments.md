# 接受配置对象作为函数参数 #

虽然保持函数接受的参数的顺序很重要，但是当函数能够接受的参数达到一定数量时，也会让用户很头疼：

```js
var alert = new Alert(100, 75, 300, 200,
	"Error", message,
	"blue", "white", "black",
	"error", true);
```

随着函数的不断重构和进化，它能够接受的参数也许会越来越多，最终就像上面的例子那样。

对于这种情况，JavaScript可以使用一个配置对象来替代以上的所有参数：

```js
var alert = new Alert({
	x: 100, y: 75,
	width: 300, height: 200,
	title: "Error", message: message,
	titleColor: "blue", bgColor: "white", textColor: "black",
	icon: "error", modal: true
});
```

这样做虽然会让代码稍微冗长一些，但是毫无疑问它的好处是很明显的：配置对象中的每个属性的名字就好比是一份文档，来告诉用户这个属性是干什么用的。特别是对于布尔值，单独的传入true和false是很难判断它的真实意图的。

使用这种方式的另外一个好处是，所有的属性都是可选的。如果配置对象中没有出现某个属性，那么就是用默认值来代替它。

```js
var alert = new Alert(); // use all default parameter values
```

如果函数需要接受必须的参数，那么最好还是将它放在配置对象的外面，就像下面这样：

```js
var alert = new Alert(app, message, {
	width: 150, height: 100,
	title: "Error",
	titleColor: "blue", bgColor: "white", textColor: "black",
	icon: "error", modal: true
});
```

配置对象中的所有属性都是函数能够接受的可选参数，而app和message则是必须要传入的参数。

对于配置对象的处理，可以像下面这样：

```js
function Alert(parent, message, opts) {
	opts = opts || {}; // default to an empty options object
	this.width = opts.width === undefined ? 320 : opts.width;
	this.height = opts.height === undefined
				? 240
				: opts.height;
	this.x = opts.x === undefined
			? (parent.width / 2) - (this.width / 2)
			: opts.x;
	this.y = opts.y === undefined
			? (parent.height / 2) - (this.height / 2)
			: opts.y;
	this.title = opts.title || "Alert";
	this.titleColor = opts.titleColor || "gray";
	this.bgColor = opts.bgColor || "white";
	this.textColor = opts.textColor || "black";
	this.icon = opts.icon || "info";
	this.modal = !!opts.modal;
	this.message = message;
}
```

对于可选的配置对象，首先使用Item 54中介绍的方法当它在真值判断中返回false时，使用空对象替换它。

上述的代码还有进一步优化的空间：通过使用对象扩展或者合并的函数。在很多JavaScript的库和框架中都有一个extend函数，它接受一个目标对象和一个源对象，然后将源对象中的属性拷贝到目标对象中：

```js
function Alert(parent, message, opts) {
	opts = extend({
		width: 320,
		height: 240
	});
	opts = extend({
		x: (parent.width / 2) - (opts.width / 2),
		y: (parent.height / 2) - (opts.height / 2),
		title: "Alert",
		titleColor: "gray",
		bgColor: "white",
		textColor: "black",
		icon: "info",
		modal: false
	}, opts);

	this.width = opts.width;
	this.height = opts.height;
	this.x = opts.x;
	this.y = opts.y;
	this.title = opts.title;
	this.titleColor = opts.titleColor;
	this.bgColor = opts.bgColor;
	this.textColor = opts.textColor;
	this.icon = opts.icon;
	this.modal = opts.modal;
}
```

通过extend函数，不再需要时常对每个属性进行判断。上述代码中的第一个extend函数用来在width和height属性没有被设置使设置默认值，因为在第二个extend函数中会根据它们进行计算。

如果所有的属性最终会被赋值到this对象上，那么以上代码可以简化成下面这样：

```js
function Alert(parent, message, opts) {
	opts = extend({
		width: 320,
		height: 240
	});
	opts = extend({
		x: (parent.width / 2) - (opts.width / 2),
		y: (parent.height / 2) - (opts.height / 2),
		title: "Alert",
		titleColor: "gray",
		bgColor: "white",
		textColor: "black",
		icon: "info",
		modal: false
	}, opts);
	extend(this, opts);
}
```

extend函数的实现通常都会遍历源对象的属性，然后如果该属性的值不是undefined时，会将它拷贝到目标对象上：

```js
function extend(target, source) {
	if (source) {
		for (var key in source) {
			var val = source[key];
			if (typeof val !== "undefined") {
				target[key] = val;
			}
		}
	}
	return target;
}
```

## 总结 ##

1. 使用可选的配置对象来让API更具可读性。
2. 配置参数中的属性都应该是函数的可选参数。
3. 使用extend工具函数来简化使用了配置对象的代码。








