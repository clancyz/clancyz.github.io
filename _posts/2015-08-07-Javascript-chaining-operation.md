---
layout:     post
title:      Javascript链式操作的实现
date:       2015-08-07 15:16:23
summary:    我面试时问过候选人的问题：怎么实现链式操作？
categories: blog
---

这个问题我在面试普通工程师的时候经常会问：


> 问：用过jQuery的链式操作吗？
>
> 答：用过，blablabla...
>
> 问：能设计一个类，支持类似jQuery链式操作吗？
>
> 答：...


其实，很多初级工程师只是 `能这样用` 没有思考 `为啥能这么用`　...

即使没有去想过，当场仔细思考下应该都OK的。

我给个我自己能接受的答案吧...

链式操作在 `jQuery` 里面应该已经很常见，例如：

{% highlight js %} 
$('.red').addClass('green').setStyle('padding', '0').removeClass('red');
{% endhighlight %}

自己实现一个支持链式操作的API也很简单，即：

> 如果一个对象有方法A，B，当一个对象执行完该对象的方法A后，再把该对象返回，就可以继续调用方法B。

链式操作最核心的一句话就是 `return this`。

比如设计一个 `Dog` 类，又能 `eat` 又能 `sleep`。

{% highlight js %} 
(function (){
  //　用于实例化
  function _Dog(){
    //do something to Arguments
  }
  
  _Dog.prototype = {
    eat: function (food){
      console.log('eat ' + food);
      return this;
    },
    sleep: function () {
      console.log('sleeping now');
      return this;
    }
  }
  
  window.Dog = function (){
    return new _Dog(arguments);
  }
})();
{% endhighlight %}

这里使用了　`延迟实例化`　。

调用：

{% highlight js %} 
Dog().eat('meat').sleep();
// => eat meat
// => sleeping now 
{% endhighlight %}

以上。





