# ECMAScript规范阅读

## Array.prototype.fill

### 场景描述

`Array.prototype.fill`是ECMAScript 6中新增加进来的，用于填补`类数组对象`。比如：
```JavaScript
var arr = new Array(3); // output: [undefined * 3] 
arr.fill(0); // [0, 0, 0]
// 或
arr = Array(3).fill(0); // output: [0, 0, 0]
```
写图形图像算法或矩阵相关的代码时会很常用这个API，而且很容易使人产生一个幻觉，比如：
```JavaScript
// 初始化3*3矩阵
var arr = Array(3).fill(Array(3).fill(0));
// 感觉很方便的啊，有木有！
```
然而，这样写你一定会得到让你想不到的结果，如果你知道这个结果，并且了解其根本原理，那就把这篇文章做个提示惊醒好了，如果你并不知道，那么我们来一起了解下新的规范中对其如何解释的，然后我们就知道他会产生什么样的后果了~

### 规范

[规范](http://people.mozilla.org/~jorendorff/es6-draft.html#sec-array.prototype.fill)中对`fill`的描述如下：

> #Array.prototype.fill (value [ , start [ , end ] ] )
> The fill method takes up to three arguments value, start and end.
> <br>
> *NOTE The start and end arguments are optional with default values of 0 and the length of the this object. If start is negative, it is treated as length+start where length is the length of the array. If end is negative, it is treated as length+end.*
> <br>
> The following steps are taken:
> 
1. Let O be [ToObject](http://people.mozilla.org/~jorendorff/es6-draft.html#sec-toobject)(this value).
2. ReturnIfAbrupt(O).
3. Let len be ToLength(Get(O, "length")).
4. ReturnIfAbrupt(len).
5. Let relativeStart be ToInteger(start).
6. ReturnIfAbrupt(relativeStart).
7. If relativeStart < 0, let k be max((len + relativeStart),0); else let k be min(relativeStart, len).
8. If end is undefined, let relativeEnd be len; else let relativeEnd be ToInteger(end).
9. ReturnIfAbrupt(relativeEnd).
10. If relativeEnd < 0, let final be max((len + relativeEnd),0); else let final be min(relativeEnd, len).
11. Repeat, while k < final
Let Pk be ToString(k).
Let setStatus be Set(O, Pk, value, true).
ReturnIfAbrupt(setStatus).
Increase k by 1.
12. Return O.
>
The length property of the fill method is 1.
> <br>
>
> *NOTE The fill function is intentionally generic; it does not require that its this value be an Array object. Therefore it can be transferred to other kinds of objects for use as a method.*

`fill`方法的处理过程如下：
> 1~2：将引用***fill的对象***的`this`转为`Object`并赋值给O，若无法转换则会抛出对应错误（如：Undefined、Null），若为Obect则直接返回该Object，其他情况（Boolean、Number、String、Symbol）则调用对应的类方法处理该数值使其成为对象，并将该对象的[[value]]返回。

概括一下，这里相当于执行了`Object(this)`

> 3~4：取O.length，然后执行len = ToLength(O.length)

这里有个问题，规范中[ToLength](http://people.mozilla.org/~jorendorff/es6-draft.html#sec-tointeger)(arguments)内部实现中会调用[ToInteger](http://people.mozilla.org/~jorendorff/es6-draft.html#sec-tointeger)(arguments)处理一些额外情况，比如：`NaN`和`Undefined`时会返回+0，但是实际上这么写fill并不会去处理，相反该报错的还会报错。

> Note：特别注意

ToLength本身还会处理一些情况但是浏览其中并未实现，如：`若length <= +0则返回+0，假如你在chrome中试一下length为负的情况，必会导致浏览器崩溃`，但是同样是对start和end的处理也调用了ToInteger，而start和end却可以按照规范中定义的方式被处理，这里可以推断chrome对ToLenghth()的实现并没有真调用ToNumber()、ToInteger()处理，而只是做了 len = Get(0, "length")，具体是否如此尚在求证，但是可以肯定的是这部分并未按照规范实现，而且有BUG。

> 5~10：处理`fill`的第二个参数和第三个参数，处理方式与`slice`类似：
> 
> + start < 0：k = len - abs(start) > 0 ? len - abs(start) : 0; 从倒数第start个开始处理
> + start >= 0：k = len > start ? start : len；从start开始处理 
> + end === undefined：end = len；从start开始处理到尾
> + end < 0：final = len - abs(end) > 0 ? len - abs(end) : 0；处理到倒数第end个
> + end >= 0：final = end < len ? end : len；处理到end

这里同样使用ToInteger()去处理了start和end，但是显然这里在chrome中的表现符合预期，并未有任何不妥。

> 11：执行真正的fill逻辑，这里包含了本文索要提醒大家需要注意的地方

需要注意，[Set](http://people.mozilla.org/~jorendorff/es6-draft.html#sec-set-o-p-v-throw)(O, Pk, value, true)这个内部处理方法，规范中定义的它的内部实现，并不会对value做甄别，而是直接使用，这相当于使用JS本来的赋值处理逻辑，看到这里我们都能想到一个关键的地方：`如果你传进来的是一个对象，那么它将会使用引用赋值！`，这意味着我们最开始那个二维矩阵的例子中，你修改任何一个`arr内部的任何一个横行中的一个值`，都会导致其他横行也跟随改变....

```JavaScript
var arr = Array(3).fill(Array(3).fill(0)); // [[0, 0, 0], [0, 0, 0], [0, 0, 0]]
arr[1][0] = 1; // [[1, 0, 0], [1, 0, 0], [1, 0, 0]]
```

### 总结

综上所述，如果你是要对矩阵或者数组进行初始化某个值，那么当这个值不会造成引用时，可以放心使用`Array.prototype.fill`这个API，若不是，你还是老老实实的用`while`或者`for`吧。

