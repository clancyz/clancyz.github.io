---

layout: post

title: 大白话Javascript设计模式之装饰者模式(Decorator)

date: 2015-10-17 23:15:23

summary: 装饰者模式是为已有的功能动态地添加另一种功能，而不必改变原类文件的方式。装饰者模式可以有效地把类的核心职责和装饰功能区分开来，简化原有的类。

categories: blog

---

## 什么是装饰者模式

> 装饰者模式即是动态地给一个对象添加额外的功能。它是用“装饰”来包裹真实的对象。装饰对象和真实对象有同样的接口，客户端对象能以和真实对象相同的方式与其交互。

如果感到有些迷糊，没关系，顾名思义，装饰者模式我们可以把它想象成一个人，可以今天穿裙子，明天穿比基尼，后天内裤外穿扮超人嘛。这无非是各种「装饰」，而且这些装饰是「非常灵活」的。我们结合实际的例子来看。

## QQ秀

好吧，虽然俺早已经过了穿各种QQ秀的年纪，但是QQ秀的确是解释装饰者模式的一个很好的例子，就以此为例。

大家都知道，QQ秀是腾讯的吸金狂魔，如果你不付钱，那么你身上就只有一条内裤（哦不，对MM而言，还有一个Bra）。要穿啥衣服都得买。而且衣服是成千上万的。

首先我们应该想到，这个穿QQ秀的实体是「人」，所以要去模拟实现它，很明显地要把人抽象为一个类。并且，对于每个要穿QQ秀的人都是唯一的，我们就取它的QQ号码作为一个私有属性。最后还得收你钱嘛，每件服饰是要钱的。这里我们把「穿QQ秀的人」用`QQshow`来表示。

{% highlight js %}

var QQShow = function (qq){
  // qq号
  this.qq = qq;
  this.cost = 0;
};

{% endhighlight %}

好，那比如现在我们要穿一件衬衫，一件牛仔裤，一双运动鞋。扩展上面的代码：

{% highlight js %}
 
QQShow.prototype.wearShirt = function () {
  console.log('穿上了衬衫');
  this.cost += 1;
}
QQShow.prototype.wearJeans = function () {
  console.log('穿上了牛仔裤');
  this.cost += 2;
}
QQShow.prototype.wearSneakers = function () {
  console.log('穿上了运动鞋');
  this.cost += 3;
}

//测试
var myQQShow = new QQshow(123456);
myQQShow.wearShirt();
myQQShow.wearJeans();
myQQShow.wearSneakers();
console.log(myQQShow.cost);
// => 穿上了衬衫
// => 穿上了牛仔裤
// => 穿上了运动鞋
// => 6

{% endhighlight %}

上面的代码是可以工作的。当然看到这有些童鞋会想吐槽了。

> 前面提到过，QQ秀是成千上万的。如果我明天又换了，要穿个裙子，穿个背心，穿个皮鞋呢？那难道要把 `QQShow` 这个类无休止地扩大下去吗？这显然是「不优雅」的。

现在，很明显地，我们应该想到把这些具体的服饰类和`QQshow`类分离开来，而不是在`QQShow`类下不断扩大代码。服饰类我们用`Finery`表示, 同时它有一个`wear`方法表示穿上。当穿上时，给qqshow加上对应的价格。

{% highlight js %}

var Finery = function (qqshow){
  this.qqshow = qqshow;
};
Finery.prototype.wear = function (){
  alert('父类的抽象方法');
}

var Shirt =  function (qqshow){
  Shirt.superClass.constructor.call(qqshow);
};
extend(Shirt, Finery);
Shirt.prototype.wear = function () {
  console.log('穿上了衬衫');
  this.qqshow.cost += 1;
}

// ...其余类似，不废话了

// 测试
var myQQShow = new QQShow(123456);
var myShirt = new Shirt(myQQShow);
var myJeans = new Jeans(myQQShow);
var mySneakers = new Sneakers(myQQShow);
myShirt.wear();
myJeans.wear();
mySneakers.wear();
console.log(myQQShow.cost);
{% endhighlight %}

上面的代码这个实现好像解决了问题。在这个例子中，`myQQShow` 是一个`QQShow` 的实例，`Shirt`, `Jeans`, `Sneakers`这几个类保持了对它的引用，且使用`wear`方法将这个实例不断地「装饰」，体现为价格不断地增加。

因为上述例子也实现了「装饰」功能，可以说，这就是一个简单的「装饰者模式」。

## 改进的装饰者模式

但是，上述例子并不是一个严格的装饰者模式。

> 为什么它是一个“伪”装饰者模式? 

因为严格的装饰者模式规定，装饰对象和真实对象具有同样的接口。

实际上，上面的例子中没有已经定义的接口，从myQQShow被处理的过程来看，我们转移了一个对象符合接口要求的职责。

我们可以想象这样的场景：

> 我穿上了一件内裤，一件背心，一件衬衫，一件牛仔裤，一双鞋，一双袜子，一顶帽子......等N个步骤之后，我突然不爽了，我要重头再来，再换一套。我想换成「只穿了内裤和背心」的样子，再从新搭配。
>
> 换言之: 在装饰的过程中我想「撤消装饰」，回到以前的某种状态，怎么办？

显然，前面的这个例子，在你多次将`myQQShow`作为参数传入，执行`wear`方法后，要你回到第2步的状态并计算价格，你肯定是会蒙圈的。为啥？

很明显，在你的装饰过程中，并没有「装饰记录」。

那么，我们想象，是否可以每次装饰后，保留当前装饰的状态呢？

> 实际上我们就可以对此作出改进：每次装饰后，返回一个和原对象相同类的实例。

说得明白点：一个人原来身无寸缕，他是可以穿衣服的，即可以「被装饰」；穿上件衬衫后，他还是一个可以被装饰的对象。每个装饰的对象都在原来的对象上增加某些功能，在我们这个例子里，就是价格的增加。

那么，我们可以使用`Finery`类来继承`QQShow`类，并有一个「装饰」方法：`Decorate` 。为什么？因为这样，经过`Finery`的子类装饰过的`QQShow`, 仍然是一个`QQShow`的实例; 那么，就可以继续地一直装饰下去。

在下面的例子中，我们把`cost`这个属性改为一个接口方法。

在`Finery`这个装饰类中，我们使用一个`Decorate`接口方法，接收传入的`QQShow`实例，这个实例可以是啥也没穿的，也可以是穿了一些衣服的；接收到实例后，把传入的`QQShow`实例保存在自己的属性中。

在`Finery`的子类中，我们可以方便地使用`superClass`（这个属性由extend继承函数产生），来调用父类的价格，加上自己的价格。


那么，这个实现就呼之欲出了：

{% highlight js %}

var QQShow = function (qq) {
  this.qq = qq;
  this.cost = function () {
    return 0;
  }
}

// 服饰类，继承QQShow
var Finery = function (qq){
  Finery.superClass.constructor.call(this, qq);
};
extend(Finery, QQShow);
Finery.prototype = {
  Decorate: function (QQShow) {
    this.QQShow = QQShow;
  },
  cost: function () {
    if (this.QQShow != null) {
      this.QQShow.cost();
    }
  }
}

var Shirt = function (qq) {
  Shirt.superClass.constructor.call(this, qq);
};
extend(Shirt, Finery);
Shirt.prototype = {
  cost: function () {
    console.log('穿上了衬衫');
    // 1为Shirt的价格
    return this.superClass.cost() + 1;
  }
}

// ...其余类似，不废话了

// 测试

var myQQShow = new QQShow(123456);
var decorateShirt = new Shirt();
var decorateJeans = new Jeans();
var decorateSneakers = new Sneakers(); 

decorateShirt.Decorate(myQQShow);
// 要再次装饰，直接装饰decorateShirt即可！
decorateJeans.Decorate(decorateShirt);
decorateSneakers.Decorate(myJeans);

decorateSneakers.cost();

// 要撤消装饰，比如要回到只穿衬衫的状态，直接拿取当时的`decorateShirt`即可！

decorateShirt.cost();

{% endhighlight %}



上述的例子便是一个严格的装饰者模式。和前面的`伪装饰者模式`相比，有以下优点：

> - 装饰后的对象仍可以与客户端交互；
> - 装饰对象和原对象均有一致的接口，方便交互；
> - 可以保持装饰对象的状态；
> - 不用去一直调用wear() 这个改变原对象功能的函数，计算价格`cost`实际都放在了最后一步一次性清算。这也是符合实际逻辑的：我不管你穿了多少次，脱了多少次，在你保存QQ秀形象，就是节算的时候，系统才会根据你身上的衣服，来算你最后需要付多少银子。


## 扩展：装饰者模式 in jQuery

当我们在设计jQuery插件时，通常会写这么一句

{% highlight js %}

var defaultOptions = {
  attr1: true,
  attr2: false
}

// 实际传入的options对defaultOptions作了拓展或修改
var settings = $.extend({},defaultOptions, options);
{% endhighlight %}
这也可以视作装饰者模式。

## 总结

装饰者模式是为已有的功能动态地添加另一种功能，而不必改变原类文件的方式。装饰者模式可以有效地把类的核心职责和装饰功能区分开来，简化原有的类。

通过阅读这篇文章，是否对`装饰者模式`有了了解呢？

以上。

关于extend原型继承函数可以看我的[这篇文章](http://clancyz.github.io/blog/2015/07/27/javascript-prototype-extend/)。







