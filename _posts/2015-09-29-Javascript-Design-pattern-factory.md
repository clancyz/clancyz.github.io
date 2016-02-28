---

layout: post

title: 大白话Javascript设计模式之工厂模式(factory)

date: 2015-09-29 23:14:21

summary: 大白话Javascript设计模式系列之-- 工厂模式。 工厂模式是js开发中使用最广泛的模式之一。

categories: blog

---

## 什么是工厂模式

> - 简单工厂模式：就是用一个单例模式来生成实例
> - 复杂工厂模式：使用子类来决定一个成员变量是哪个具体类的实例

你看了可能不太明白在bb什么，是的，上面这个我是抄书的，当初我学习的时候，也不知道它在bb什么。

举具体的例子来说明吧：

## 那么，需求来了

> 产品狗：给我实现一个功能，左边是一个select选择框，右边是一个按钮。
> select选择框的内容有三种：img(图片), link(链接), text(文本)。
> 选择相应的选项后，点击按钮，就往body里面插入该标签。
> 最后alert一下：`我插入了!`
> 
> 程序猿：这有蛋难度啊，秒出一个

{% highlight js %}
// 代码1.1
var Tag = function (){};
Tag.prototype = {
  insert: function (tagName){
    var tag = document.createElement(tagName);
    document.body.appendChild(tag);
    alert('我插入了!')
  }
}

var myAppendTag = new Tag();
myAppendTag.insert('img');

{% endhighlight %}

## 产品狗为啥会被叫做产品狗

> 需求变了：
>
> 产品狗：这怎么行啊，你这空标签插进去啥谁知道啊，我要的是这样的：
>
> 加一个input输入框：
>
> - 对于img标签，输入图片url;
>
> - 对于link标签，输入链接；
>
> - 对于文本，输入文本；
>
> 程序猿：好吧，那只好修一下了：

{% highlight js %}
// 代码1.2
var Tag = function (){};
Tag.prototype = {
  insert: function (tagName, content){
    var tag = document.createElement(tagName);
    switch (tagName) {
      case 'img':
        tag.src = content;
        break;
      case 'a':
        tag.href = content;
        break;
      case 'text':
        tag.value = content;
        break;
    }
    document.body.appendChild(tag);
    alert('我插入了!')
  }
}

var myAppendTag = new Tag();
myAppendTag.insert('img', '/assets/img/1.png');

{% endhighlight %}

上面这个例子类似的功能相信很多FE同学都接触过。这就是 `简单工厂模式` 。

## 简单工厂模式

> 把成员对象的创建工作交给一个外部对象，这个对象集成了所有实例的创建逻辑。

程序猿认为，产品狗肯定后续又改需求，比如要我 `加个span标签` 啥的。在代码1.2中，如果需要加一个标签，我多写一个case就行了嘛。easy。

## 产品狗为啥会被叫做产品狗之系列2

> 需求变了：
>
> 产品狗：那个，我现在要加入一个css link,要加到head里面，不是body。
>
> 程序猿：卧槽...

{% highlight js %}
// 代码1.3
var Tag = function (){};
Tag.prototype = {
  insert: function (tagName, content){
    var tag = document.createElement(tagName);
    switch (tagName) {
      case 'img':
        tag.src = content;
        document.body.appendChild(tag);
        break;
      case 'a':
        tag.href = content;
        document.body.appendChild(tag);
        break;
      case 'text':
        tag.value = content;
        document.body.appendChild(tag);
        break;
      case 'link':
        tag.type = 'text/css';
        tag.rel = 'stylesheet';
        tag.href = content;
        document.head.appendChild(tag);
        break;
    }
    alert('我插入了!')
  }
}

var myAppendTag = new Tag();
myAppendTag.insert('img', '/assets/img/1.png');

{% endhighlight %}

写出上面这段代码之后，程序猿已经隐隐觉得有些 `不优雅` 了。

## 产品狗为啥会被叫做产品狗之系列3

> 需求叕变了...
>
> 产品狗：哎呀，这个链接a的原生效果太不炫了，我们要当客户选择 `链接a` 这个选项时，再出现个下拉框，让用户选颜色的。
>
> 程序猿：卧槽槽...

程序猿开始咬牙切齿了，这要在 `insert` 方法加个新的参数啊。
加个参数也就罢了，但是这个碧池，不知道哪天说不定又说 `img` 要加 `width` 和 `height` ，`table` 还要顺带插入 `thead` `tbody` `tr` `td`  呢。这要是参数都加上去不就累积N个参数了？这特么的，没法搞啊！

> 这要怎么搞？

程序猿开始分析了一下，发现了以下的规律：

> - 需求无非是要 `create` 一个标签然后 `appendchild` ，但是标签是不固定的，碧池不知道又会让加什么
> - 不同的标签，要求的行为方式有可能不一样，不能把它都写到insert方法上
> - 把所有的方法都写在一个对象里面比较臃肿，也违反了 `高内聚` 原则；所创建的类只能是 `事先考虑到` 的，如果需要添加新的类，就需要改变这个工厂类了。 

于是，一个思路慢慢形成了：

> - appendTag是父类，要继承这个父类，在子类里面去实现相应的逻辑
> - 父类应该是 `固定` 的。
> - 最后都要alert('我插入了!')，这是公共的，可以提升到父类

{% highlight js %}

// 代码1.4
// 父类
var appendTag = function (){};
appendTag.prototype = {
  insert: function (tag) {
    this.createAndAppend(tag);
    alert('我插入了!');
  },
  createAndAppend: function (tag) {
    throw new Error('俺是父类的一个抽象方法，在子类里面再重载俺吧好伐');
  }
}

// 子类img
var ImgTag = function (){};
// 继承父类
extend(ImgTag, Tag); 
// overload掉父类的方法
ImgTag.prototype = {
  createAndAppend: function (imgUrl){
    var me = document.createElement('img');
    me.src = imgUrl;
    document.body.appendChild(me);
  }
};

// 调用
var imgTag = new ImgTag();
imgTag.insert('/assets/img/1.png');


// 子类链接a 
var ATag = function () {};
extend(ATag, Tag);
ATag.prototype = {
  createAndAppend: function (linkUrl, color){
    var me = document.createElement('a');
    me.href = linkUrl;
    me.style.color = color;
    document.body.appendChild(me);
  }
}

// 调用
var aTag = new ATag();
aTag.insert('https://www.github.com', 'red');

{% endhighlight %}

关于extend函数可以看我的[这篇文章](http://clancyz.github.io/blog/2015/07/27/javascript-prototype-extend/)。

上面的代码1.4即是 `复杂工厂模式`。

## 复杂工厂模式

> 从代码1.4可以看出，工厂模式的具体实例化方法是 `动态实现` 的，可以随着需求的调整增加相应的逻辑代码。
> 实现了 `对修改关闭，对拓展开放` 这一原则。
> 同时，这也 `节省了性能开销` 。如果像代码1.2那样还好，只是个 `switch` 判断；但是如果有复杂的配置或初始化或判断分支逻辑，显然代码1.4把这些逻辑分散到了各个子类中，需要使用的时候才会执行，明显是优于简单工厂模式的。 
> 当然，多数情况下，简单工厂模式就足以胜任了。

以上。









