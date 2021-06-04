# 1. 介绍下原型链

原型链这东西，基本上是面试必问，而且不是知识点还是基于原型链扩展，所以我们先把原型链整明白。我们看一张网上非常流行的图：

![原型链](https://user-gold-cdn.xitu.io/2020/2/23/1707146321400d31?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

代码举例：

```js
function person() {
  this.name = 10
}

person.prototype.age = 10
const p = new person()
```

## 分析构造函数

我们通过断点看下person这个函数的内容

![](https://user-gold-cdn.xitu.io/2020/2/23/17071456b8ce57da?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

它是一个自定义的函数类型，看关键的两个属性`prototype`和`__proto__`，我们一一分析

1. prototype分析

对 `prototype`展开看，是个自定义的对象，这个对象有三个属性`age, constructor, __proto__`，`age`的值是10，那么可以得出通过`person.prototype`赋值的参数都是在`prototype`这个对象中。

点开 `constructor`，发现这个属性的值就是指向构造器`person`函数，其实就是循环引用，这时候就有点套娃的意思了。

![](https://user-gold-cdn.xitu.io/2020/2/23/1707146b212791f2?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

那么，根据字面意思，`prototype`可以翻译成，原先对象，用于扩展属性和方法。

2. `__proto__` 分析对 `__proto__` 展开看看

![](https://user-gold-cdn.xitu.io/2020/2/23/1707146f25a52c6c?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

`person` 中的 `__proto__` 是一个原始的 `function`对象，在`function`对象中，有看到了`__proto__`这个属性，这个时候它的值是原始的`Object`对象，在`Object`对象中又再次发现了`__proto__`属性，这个时候`__proto__`等于`null`

js 中数据类型分为两种，基本类型和对象类型，所以我们可以这么猜测，person是一个自定义的函数类型，它应该是属于函数这一家族下的，对于函数，我们知道它是属于对象的，那么它们几个是怎么关联起来的呢？

没错，就是通过`__proto__`这个属性，而由这个属性组成的链，就叫做原型链。

根据上面的例子，我们可得出，原型链的最顶端是`null`，往下是`Object`对象，而且只要是对象或函数类型都会有`__proto__`这个属性，毕竟大家都是`js-family`的一员。

