---

layout: post

title: 大白话Javascript设计模式之观察者模式(Observer)

date: 2015-11-15 22:31:50

summary: 小明，你妈喊你回家吃饭了。

categories: blog

---

## 什么是观察者模式

> 观察者模式(Observer Pattern)：定义对象间的一种一对多依赖关系，使得每当一个对象状态发生改变时，其相关依赖对象皆得到通知并被自动更新。

## 日常的使用：EventHandler vs EventListener

我们先来复习一下基础知识。

大家都知道，大多数时间里面，FE都是面向浏览器编程的，即基于DOM脚本的编程环境来工作。

在开发过程中，用户与浏览器的交互是少不了的，所以我们必须基于「事件驱动」去做一些事情。

最简单的例子，绑定一个`button`元素的点击事件：

{% highlight html %}
<button id="btn" onclick="alert('被猛击了！')">猛击我吧</button>
{% endhighlight %}
当然，我们更多希望点击事件是在js代码中方便维护。so:

{% highlight js %}
var btn = document.getElementById('btn');
btn.onclick = fn1;
function fn1 (){
  alert('被猛击了！')
}
{% endhighlight %}

好，那上面我们做的事情，就是给`button`附加了一个「事件处理器」（Eventhandler）来监听点击事件。

那么，如果我再往上加一个事件处理函数呢？

{% highlight js %}
btn.onclick = fn2;
function fn2 (){
  alert('被暴击了！');
}
{% endhighlight %}

那么最后我点击按钮，执行了什么？

> 被暴击了！

为啥没有执行"被猛击了”呢？因为事件处理器对于一种特定的事件类型，最多只能注册一个程序。上面的例子中，fn2把fn1覆盖掉了。

那么我希望顺序执行fn1和fn2怎么办？当然就是使用addEventListener这种「事件监听器」（EventListener）方法拉。

{% highlight js %}
var btn = document.getElementById('btn');
// 兼容性啥的我这就不写了 
btn.addEventListener('click', fn1);
function fn1(){
  alert('被猛击了！')
}

btn.addEventListener('click', fn2);
function fn2(){
  alert('被暴击了！');
}
{% endhighlight %}

点击之：

> 被猛击了！
> 
> 被暴击了！

好，那我们来看，在这个过程中，`EventListener`这个事件监听器做了哪些事情：

> - 监听了`click`事件，并附加了2个处理函数。说明一个事件可对应多个监听函数。
> - 当用户点击时，顺序执行这2个函数。
> - fn1和fn2之间是没有联系的。我也可以很方便地再加个fn3。

实际上，EventListener就是一个浏览器内部实现的`观察者模式`：

> - 一个对象发生改变（用户点击）时，通知其他对象（执行监听函数）。
> - 发生改变的对象称为观察目标，而被通知的对象称为观察者，一个观察目标可以对应多个观察者。
> - 这些观察者之间没有相互联系，可以根据需要增加和删除观察者（removeEventListener），使得系统更易于扩展。

OK，我们已经看到了`观察者模式`的使用，那么下面我们来实现观察者模式。


## 观察者模式的实现：PM2.5

最近帝都的PM2.5几乎天天爆表，在各大应用市场中，PM2.5实时指数的APP下载量也一路飙升。

如果PM2.5指数高于200就算是重度污染，所以，产品狗打算开发个功能，用户可以通过APP的某个设置开关来订阅是否发送空气污染的预警提醒。

那么空气污染指数多高算高呢？有些用户认为300以上是高，有些用户认为100就很高了。

那么，这个值交由用户来设置。

如果用户订阅之后，用户希望当高于用户的设置值时，用户手机就会触发一个预警提醒，提醒出门好戴口罩。

下面我们来做一个具体的实现。

首先是我们的观察者，这里就可以认为是`服务器`; 服务器上存储了订阅用户的信息，并有通知用户的功能。当然，还具有获取并设置PM2.5的功能。

{% highlight js %}
var Server = function () {
  // 订阅的用户名单
  this.list = [];
  // 初始的PM2.5值
  this.airCondition = 0;
};

// 如果有新用户订阅，就在名单上添加
Server.prototype.observe = function (user) {
  this.list.push(user);
}

// 如果用户退订，就从名单上删除
Server.prototype.unobserve = function (user) {
  for( var i = 0, len = this._list.length; i < len; i++ ) {
      if( this.list[i] === user ) {
        this.list.splice(i, 1);
        console.log( '取消订阅了' );
        return true;
      }
    }
  return false;
}

// 通过同步等手段，获取PM2.5并设置（这里我们略去了获取PM2.5的过程）
// 并通知用户

Server.prototype.setAirCondition = function (num) {
  this.airCondition = num;
  this.notify(num);
}

// 发送通知后，接收通知的用户app来判断是否大于设定值，这里是analyze方法

Server.prototype.notify = function () {
  var args = Array.prototype.slice.call( arguments, 0 );
  for( var i = 0, len = this.list.length; i < len; i++ ) {
    this.list[i].analyze.apply( null, args );
  }
};

{% endhighlight %}

好，下面我们来实现用户类`User`。用户的关键方法是`analyze`, 即比对当前空气污染指数和设定值。同时用户有个唯一标识符`id`。

{% highlight js %}
var User = function (id, setNum) {
  this.id = id;
  this.setNum = setNum;
  this.analyze = function (num) {
    if (num > setNum) {
      alert('用户' + id + '：当前空气污染指数高于您的设定值！')
    }
  }
};

{% endhighlight %}

好，现在我们来做一个具体的调用：

{% highlight js %}
// 服务器准备就绪
var server = new Server();
//id为1的用户，设定的预警值为100
var user1 = new User(1, 100);
// 服务器将此人加入订阅列表
server.observe(user1);
// id为2的用户，设定的预警值为200
var user2 = new User(2, 200);
// 服务器将此人加入订阅列表
server.observe(user2);

// 服务器获取到当前的空气污染指数为150，并设置
server.setAirCondition(150);

// 设置的同时，通知用户，用户执行analyze方法

// => 用户1：当前空气污染指数高于您的设定值！

// 用户1退订，服务器将此人从订阅列表删除
server.unobserve(user1);

// 服务器获取到当前的空气污染指数为250，并设置
server.setAirCondition(250);

// => 用户2：当前空气污染指数高于您的设定值！


{% endhighlight %}

例子可以运行得很好。

上面的例子中可以看出：

> - 用户在例子中即是「观察者」，服务器为「观察对象」；
> - 「服务器」和「用户」是一个一对多的关系；可以根据需要增加或删除用户
> - 当服务器的空气污染指数改变时，会实时通知用户，用户去执行自身的更新方法（例子中为`analyze`，大多数写法会写成`update`）。
> - 用户彼此之间是没有关系的。

## 回到事件监听器

现在让我们回到事件监听器。

再次分析，我们可以认为，fn1和fn2都是观察者，而dom的点击事件是被观察对象。

如果点击了，那么就执行fn1和fn2。这里的部分伪代码如下：

{% highlight js %}
Browser.prototype.notify = function (thisObj) {
  var scope = thisObj || window;
  var args = Array.prototype.slice.call( arguments, 1 );
  for( var i = 0, len = this.list.length; i < len; i++ ) {
    // 这里list[i]即为某个事件函数，执行即可。
    this.list[i].call( scope, args );
  }
};
{% endhighlight %}

## 总结

观察者模式的优点如下：

> - 观察者模式可以实现表示层和数据逻辑层的分离，并定义了稳定的消息更新传递机制，抽象了更新接口，使得可以有各种各样不同的表示层作为具体观察者角色。
> - 观察者模式在观察目标和观察者之间建立一个抽象的耦合。
> - 观察者模式支持「事件广播」。
> - 观察者模式符合“开闭原则”的要求。


是否明白呢亲？

以上。








