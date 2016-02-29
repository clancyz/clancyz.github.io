---

layout: post

title: 大白话Javascript设计模式之门面模式(facade)

date: 2015-10-14 22:45:50

summary: 使用门面模式设计出简洁、易于调用、兼容性良好的API。

categories: blog

---

## 什么是门面模式

惯例，抄定义：

> 门面模式，是指提供一个统一的接口去访问多个子系统的多个不同的接口，它为子系统中的一组接口提供一个统一的高层接口。使得子系统更容易使用。

我的理解：门面模式对外暴露了一个易于调用的方法，调用的一方不用关注里面有多复杂的逻辑、流程，直接调就可以了。

可能表述得还不是很清楚，那么看一个实际例子：

## 举例说明

比如我们设计了一个厨师 `Cook` 类，厨师能干啥？当然是做菜了。

做一道菜，当然要先 `买菜` , `切菜` , `洗菜` ，最后 `炒菜` 。一共有四个步骤。且这四个步骤是有先后顺序的。

用代码来实现一下：

{% highlight js %}
var Cook = function () {
  this.status = '';
  this.dish = '';
};

Cook.prototype = {
  buy: function (name){
    // 买回来了
    this.dish = name;
    this.status = 'bought';
    console.log(this.status);
  },
  slice: function (name) {
    // 如果买回来，就切了
    if (!this.status === 'bought' || this.dish !== name) return;
    this.status = 'sliced';
    console.log(this.status);
  },
  wash: function (name) {
    // 如果切好，就洗了
    if (!this.status === 'sliced' || this.dish !== name) return;
    this.status = 'washed';
    console.log(this.status);
  },
  make: function (name) {
    // 如果洗好，就做了
    if (!this.status === 'washed' || this.dish !== name) return;
    this.status = 'done';
    console.log(this.status);
  }
}
{% endhighlight %}

OK，很简单，已经实现好了。那么我要调用它做某道菜时，这么调用即可：

{% highlight js %}
var myCook = new Cook();
myCook.buy('tomato');
myCook.slice('tomato');
myCook.wash('tomato');
myCook.make('tomato');
// => bought
// => sliced
// => washed
// => done

{% endhighlight %}

这个运行起来是没问题的。看到这我知道你肯定想吐槽，恩。

如果我现在又想做个`potato`的话，我又得调`buy`, `slice`, `wash`, `make` 共4次。

> 烦不烦啊。

so, 我们自然而然会想到增加一个原型方法 `cook` ，把这4个方法整合到一起；

我才不管你丫买菜切菜洗菜呢，劳资只想吃菜，你给我去弄出来就好了，爱咋咋弄（换句话说，我不关心你怎么做的）。

{% highlight js %}
Cook.prototype.cook = function (name) {
  this.buy(name);
  this.wash(name);
  this.slice(name);
  this.make(name);
}
{% endhighlight %}

这样，调用就变成了：

{% highlight js %}
var myCook = new Cook();
myCook.cook('tomato');
{% endhighlight %}

简洁明快多了嘛。这就是 `门面模式` 的实现。

## 实际例子：浏览器的事件绑定兼容(polyfill)

{% highlight js %}
function addEvent (el, type, fn) {
  if (window.addEventListener) {
    el.addEventListener(type, fn, false);
  }else if (window.attachEvent) {
    el.attachEvent('on' + type, fn)
  }else{
    el['on' + type] = fn;
  }
  
}
{% endhighlight %}

这个例子也是同样的，你总不希望每次想进行事件绑定时都要写这么一大坨吧。

直接调用`addEvent`即可。

其实门面模式也可以看做是 `Don't repeat yourself` 这一精神的体现。

## 总结

> 门面模式对外暴露了一个易于调用的方法，调用的一方不用关注里面有多复杂的逻辑、流程, 直接调用该方法即可。门面模式可以隐藏/封装内部的实现。使用门面模式可以设计出简洁、易于调用、兼容性良好的API。

以上。
