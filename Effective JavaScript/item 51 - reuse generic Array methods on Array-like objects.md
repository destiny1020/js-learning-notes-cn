## 在类数组对象上重用通用的数组方法 ##

`Array.prototype`对象上的标准方法被设计为也可以在其它对象上重用 - 即使不是继承自Array的对象。因此，在JavaScript中存折一些类数组对象(Array-like Objects)。

一个典型的例子是函数的arguments对象，在[Item 22](http://blog.csdn.net/dm_vincent/article/details/39368359)中对它进行过介绍。该对象并不继承自Array.prototype，所以我们不能直接调用`arguments.forEach`来对其中的元素进行遍历。但是，我们可以通过首先得到forEach方法的对象，然后调用call方法(可以参考[Item 20](http://blog.csdn.net/dm_vincent/article/details/39313317))：

```js
function highlight() {
	[].forEach.call(arguments, function(widget) {
		widget.setBackground("yellow");
	});
}
```

forEach方法本身而是一个Function类型的对象，因此它能够继承`Function.prototype`的call方法。

在Web环境中，DOM的NodeList类型的实例也是类数组对象。因此，对于它也可以使用以上的方式借助Array中的方法进行操作。

那么，究竟什么才是"类数组对象"呢？实际上，只要对象满足了以下两个规定，那么它就是一个"类数组对象"：

- 它拥有一个名为length，介于0到2^32-1之间的整型属性。
- length属性的值大于该对象上的最大索引值。索引值的范围在0到2^32-2之间，索引值的字符串表示就是该对象上对应于一个属性值的键。

所以下面的这个对象就是一个"类数组对象"，它能够利用Array.prototype上定义的方法：

```js
var arrayLike = { 0: "a", 1: "b", 2: "c", length: 3 };
var result = Array.prototype.map.call(arrayLike, function(s) {
	return s.toUpperCase();
}); // ["A", "B", "C"]
```

对于字符串类型的实例，也可以将它们看做是一种"类数组对象"。毕竟它们也拥有length属性，也能够通过索引值访问到其中的每一个字符。因此，它们也能够利用Array.prototype上定义的方法：

```js
var result = Array.prototype.map.call("abc", function(s) {
	return s.toUpperCase();
}); // ["A", "B", "C"]
```

只不过，需要注意字符串实际上是一个不可变(Immutable)的"类数组对象"。

对于"类数组对象"，他还具有两个比较特别的行为：

- 将length属性设置的比当前实际的大小要小时，会自动的删除多余的元素。
- 当添加的属性的索引值大于等于当前的length属性时，比如索引值为n，length属性的只会被自动的更新为n + 1。

在所有Array提供的方法中，只有一个是不能够被"类数组对象"使用的：`Array.prototype.concat`方法。它虽然可以被"类数组对象"通过call方法进行调用，但是它还会检查[[class]]的值(实际上就是对象的类型)，关于[[class]]，在[Item 40](http://blog.csdn.net/dm_vincent/article/details/40106917)有提到过。

concat方法会判断传入的对象是否是一个真正的数组对象。如果是数组对象，就会按照期望的方式执行连接操作；如果不是真正的数组对象，那么会直接将参数作为一个整体进行连接，像下面这样：

```js
function namesColumn() {
	return ["Names"].concat(arguments);
}

namesColumn("Alice", "Bob", "Chris");
// ["Names", { 0: "Alice", 1: "Bob", 2: "Chris" }]
```

可见，concat方法将arguments对象作为一个整体进行了连接。

那么，解决方法就是让concat方法将"类数组对象"当做是一个真正的数组对象。一种比较简便和常用的方法是使用slice方法：

```js
function namesColumn() {
	return ["Names"].concat([].slice.call(arguments));
}

namesColumn("Alice", "Bob", "Chris");
// ["Names", "Alice", "Bob", "Chris"]
```

### 总结 ###

1. 通过获取方法的引用结合call方法，对Array上的方法进行重用，使之能够被用在"类数组对象"上。
2. 任何对象都能够利用Array上的方法，只要改方法满足了"类数组对象"的两条规则。





