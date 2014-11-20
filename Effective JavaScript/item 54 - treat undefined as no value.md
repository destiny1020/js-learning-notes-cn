# 将undefined视为"没有值" #

JavaScript中的undefined是一个特殊的值：当JavaScript没有提供具体的值时，它就会产生undefined。

比如：

1. 未被赋值的变量的初始值就是undefined
2. 访问对象中不存在的属性会得到undefined
3. 没有返回值的函数，undefined会作为其返回值
4. 函数的参数没有提供时，它的值就是undefined

```js
// 情形1
var x;
x; // undefined

// 情形2
var obj = {};
obj.x; // undefined

// 情形3
function f() {
	return;
}
function g() { }

f(); // undefined
g(); // undefined

// 情形4
function f(x) {
	return x;
}
f(); // undefined
```

将undefined视为任何具体值的缺失是JavaScript语言的一种约定。所以，将它作为其它用途使用就是一种具有风险的行为。比如，一个库中的highlight方法用来改变元素的背景颜色：

```js
element.highlight(); // use the default color
element.highlight("yellow"); // use a custom color
```

如果我们想让highlight方法具备返回随机颜色的功能，我们也许会尝试使用undefined作为这种情况下需要传入的参数来和其他情况区别开：

```js
element.highlight(undefined); // use a random color
```

但是，这样做是有风险的。比如，我们可能会向该方法中传入一个对象的属性，如果该属性没有值时，highlight方法就会返回一个随机的颜色，但是这种情况下，用户期望的结果应该是为该元素使用默认的颜色。

```js
var config = JSON.parse(preferences);
// ...
element.highlight(config.highlightColor); // may be random
```

除了使用undefined之外，有些开发人员可能会选择同样比较特殊的null作为参数传入来进行区分：

```js
element.highlight(null);
```

但是，这样的代码的可读性比较差。用户第一眼看上去会猜想此方法是要移除element的背景颜色，而不是八竿子打不着的返回随机颜色。

一个更好的API应该是这样的，通过传入字符串来表名意图：

```js
element.highlight("random");

// 或者通过配置对象，关于配置对象可以参考Item 55
element.highlight({ random: true });
```

另外一个需要注意undefined的地方是拥有可选参数的函数。虽然可以通过arguments对象(关于此对象，可以参考Item 51)对实际传入的参数进行判断，但是对参数进行undefined判断能够让API更加健壮。比如，一个Server对象或许会接受host名作为参数：

```js
var s1 = new Server(80, "example.com");
var s2 = new Server(80); // defaults to "localhost"

function Server(port, hostname) {
	if (arguments.length < 2) {
		hostname = "localhost";
	}
	hostname = String(hostname);
	// ...
}
```

以上代码使用arguments的length值作为判断依据，来给hostname参数一个默认值。但是，如果hostname被传入了undefined，就会导致默认值不会生效：

```js
// config.hostname为undefined时，就跳过了以上的检查
var s3 = new Server(80, config.hostname);

// 更好的办法是显式地对undefined进行检查
function Server(port, hostname) {
	if (hostname === undefined) {
		hostname = "localhost";
	}
	hostname = String(hostname);
	// ...
}
```

一种替代方案是进行真值判断(参见Item 3)：

```js
function Server(port, hostname) {
	hostname = String(hostname || "localhost");
	// ...
}
```

依据是undefined在做真值判断时会返回false，因此默认值localhost会生效。

但是需要注意在某些情况下使用真值判断也是不安全的。

当一个函数能够接受空的字符串作为合法参数时，进行真值判断就会将传入的空字符串替换为默认值。类似的，如果一个函数能够接受数字0(或者特殊的NaN)作为合法参数，真值判断也会将它替换成默认值。

比如，下面的API用来通过传入元素的宽度和高度进行创建。如果没有传入，则使用默认值：

```js
var c1 = new Element(0, 0); // width: 0, height: 0
var c2 = new Element(); // width: 320, height: 240

function Element(width, height) {
	this.width = width || 320; // wrong test
	this.height = height || 240; // wrong test
	// ...
}

var c1 = new Element(0, 0);

c1.width; // 320
c1.height; // 240
```

当我们传入0时，真值判断会将它替换成默认值。然而这并不是我们想要的行为。更好的方式是显式对undefined进行判断：

```js
function Element(width, height) {
	this.width = width === undefined ? 320 : width;
	this.height = height === undefined ? 240 : height;
	// ...
}

var c1 = new Element(0, 0);

c1.width; // 0
c1.height; // 0

var c2 = new Element();
c2.width; // 320
c2.height; // 240
```

## 总结 ##

1. 不要使用undefined来表达除了缺失特定值外的任何其他意义。
2. 在需要表达特殊情况时，不要使用undefined或者null。而是使用更具表达性的字符串或者对象。
3. 在函数中显式地对参数进行undefined检查，而不要依赖于诸如arguments.length等检查方法。
4. 对于能够接受真值判断返回false的特殊值(如0，NaN，null，"")，不要使用真值判断。






















