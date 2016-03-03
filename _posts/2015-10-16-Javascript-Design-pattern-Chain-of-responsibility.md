---

layout: post

title: 大白话Javascript设计模式之职责链模式(Chain of Responsibility)

date: 2015-10-16 22:25:13

summary: 职责链模式可以让对象得到灵活、高效的处理，减轻请求对象和处理对象的耦合。

categories: blog

---

## 什么是职责链模式
> 当一个对象A要办个事儿，就向一个处理对象B发起请求，如果B不处理，可以把请求转给C，如果C不处理，又可以把请求转给D。一直到有一个对象愿意处理这个请求为止。
>
> B -> C -> D 这就是一条`职责链`。
>
> 如果请求一直没人处理，那么这事儿就黄了，搞不定。
>
> A在一开始是不知道自己会被谁处理的。

## 实例：俺是个鉴黄的！

下面举具体的例子来说明。

俺在路上看到一则新闻说关于「鉴黄师」的，心向往之，不禁开了个脑洞。。

现在，假设你是一名拥有强大技能的「鉴黄师」。

当然，因为你是鉴黄师，不是一个只是爱看黄的普通宅男，那么你肯定是要比一般人牛逼的。

那牛逼在哪呢，就在于，你有一套极其牛逼的`自动化工具链`。

恩，提到这个闪瞎狗眼的神器，必须得先说下工作内容，额，就是鉴黄嘛。比如，把一个文件夹拖进去，自动鉴定里面的内容是不是黄色内容。如果是黄色内容就嗷嗷叫报警。你们还真一个个点开看啊，营养你跟得上啊，敢不敢更low一点。

那么这工具是怎么工作的呢？

> - 首先先拿到文件夹下所有文件。废话吗这不。
> - 然后工具中的「番号模块」开始工作，具有黄色特征的文件名，比如`ebody`, `snis`,  `SOE` 之类，如果碰上，就是黄的，恩那么就嗷嗷报警。
> - 没碰上怎么办呢，「番号模块」就表示无能为力了，把它交给一个很强大的模块「md5模块」。这个`md5`是啥玩意儿呢，就是一个文件有唯一的标识符，把这标识符和国内最大的影片存储中心`百度云`数据库中的标识符做对比。说白了很简单，这部影片标识符如果和百度云禁片的标识符一样，那么肯定是黄的嘛。牛逼吧。
> - 最后一种情况，某片连百度云上都没有，那肿么办，「md5模块」只好把这个文件夹交给最终的大招「视频图像识别模块」，自动识别视频中的图像，如果符合xx特征的，那么就是黄的。

恩，所以这个工具就是这么工作的：

> 番号模块 -> md5模块 -> 视频图像识别模块

这就形成了一条`职责链` ，各个模块各司其职，逐级传递。

## Javascript实现

那么我们怎样用js把它抽象出来呢?

- 首先，这仨东西都是模块，我们应该能想到建立一个「模块类」。
- 每个模块都有处理功能，那么模块类应该有一个抽象的方法`handle`。
- 每个模块都可以设置「下级模块」，如果自己处理不了，就给下级模块处理（当然，在例子中，视频图像识别模块是没有下级的，它处理不了的，整个工具就都处理不了了）。

那么，先建立一个`模块类`：

{% highlight js %}
var Handler = function (){};
// 模块的处理方法(抽象方法)
Handler.prototype.handle = function (files) {
  throw new Error('俺是父类的一个抽象方法，在子类里面再重载俺吧好伐')
}
// 模块的下级
Handler.prototype.next = function (nextHandler) {
  if (nextHandler instanceof Handler) return;
  this.next = nextHandler;
}

{% endhighlight %}

接着来实现番号模块、md5模块、视频图像处理模块。逻辑基本都是一样的：

{% highlight js %}
// 番号模块
var identifierHandler = function (){};
extend(identifierHandler, Handler);
identifierHandler.prototype.handle = function (files) {
  // 按番号来处理, 如果处理不了，给它的下级模块
  try {
    handleByIdentifier();
    function handleByIdentifier(){
      //...
    }
  } catch(e){
    if (this.next) {
      this.next.handle(files);
    } else {
      throw new Error('处理不了');
    }
  }
}

// md5模块
var md5Handler = function (){};
extend(md5Handler, Handler);
md5Handler.prototype.handle = function (files){
  try {
    handleBymd5(files);
    function handleBymd5(files){
      //...
    }
  }
  catch(e){
    if (this.next) {
      this.next.handle(files);
    } else {
      throw new Error('处理不了');
    }
  }
}

// 视频图像处理模块
var videoHandler = function (){};
extend(videoHandler, Handler);
videoHandler.prototype.handle = function (files){
  try {
    handleByVideo(files);
    function handleByVideo(files){
      //...
    }
  }
  catch(e){
    if (this.next) {
      this.next.handle(files);
    } else {
      throw new Error('处理不了');
    }
  }
}
{% endhighlight %}

关于extend原型继承函数可以看我的[这篇文章](http://clancyz.github.io/blog/2015/07/27/javascript-prototype-extend/)。

现在已经基本实现好了。那么做一个具体的调用，这里我们假设取得了要鉴黄的文件，参数`files`是个数组：

{% highlight js %}
var files = ['soe408.mkv', 'snis-242.avi', '1024.rmvb'];

var myIdentifierHandler = new identifierHandler();

var myMd5Handler = new md5Handler();

var myVideoHandler = new videoHandler();


// 设置下级
myIdentifierHandler.next = myMd5Handler;
myMd5Handler.next = myVideoHandler;

// 执行鉴黄
myIdentifierHandler.handle(files);
{% endhighlight %}

由上例可见，我们只要将`files`传给职责链中的第一环`myIdentifierHandler`就可以了，如果它处理不了，就会将请求传给下一环节。

同时我们可以看出一个重点问题：

> 我们可以自由设定一个handler的下级。

比如我们现在的职责链是：

番号模块 -> md5模块 -> 视频图像识别模块

来了一个新任务，你一瞅，我去，今天是1月2号，来了个`1月1日 新作34连发`，这么新的片子，百度云上肯定没有啊，那md5模块检测就基本没有用了，而且这个模块检测还非常耗时，那么你完全可以跳过md5模块：

{% highlight js %}
myIdentifierHandler.next = myVideoHandler;
myIdentifierHandler.handle(files);
{% endhighlight %}

这样，可以根据实际的情况，自由地选择下级职责链中的下一环，在特定的场景中就可以`节约性能开销`。

## 多扯两句：事件委托 / 原型链

学过BOM知识的童鞋肯定都有了解，浏览器的「事件冒泡」、「事件委托」or「事件代理」的过程，实际也可以用`职责链`来解释：一个事件在某个节点上被触发，然后向根节点传递， 直到被节点捕获。

Javascript的`原型链`查找方法也有`职责链`的影子：在子类上调用一个方法，如果自身找不到，就沿着原型链一级级向上查找。

## 总结

职责链模式可以让对象得到灵活、高效的处理，减轻请求对象和处理对象的耦合。

是否明白呢亲？

以上。









