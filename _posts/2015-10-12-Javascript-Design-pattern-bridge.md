---

layout: post

title: 大白话Javascript设计模式之桥接模式(bridge)

date: 2015-10-12 18:24:50

summary: 形形色色，形状是形状，颜色是颜色。我们要的是「喜形于色」。

categories: blog

---

## 什么是桥接模式

> 抄一个GOF的定义：
>
> - 桥接模式用于弱化它与使用它的类和对象的耦合。
> - 桥接模式作用是将 `抽象` 与其 `实现` 隔离开来，以便二者 `独立变化` 。

这说的是什么鬼，特么的，说了等于没说。

在解释之前，我还是继续 `产品狗与程序猿` 的故事系列吧。相信你看了一定很快懂。

## 那么，需求来了

> 产品狗：哎呀，我们现在要开发个底层canvas图形库。
>
> 程序猿：卧槽， `底层` 听起来挺不错的哟，听着就高端
>
> 产品狗：这个底层最基本的功能是画各种图形，比如矩形，圆形，线条，贝赛尔曲线，正N边形，桃心，内外旋轮，自定义路径...　好了你先实现画矩形和圆形吧，要求程序画完图形之后打印出一句：“我画了个圆！”这样，搞个demo出来，这周周报就有得写了，其他功能再说啦。
>
> 程序猿：好吧

程序猿知道canvas是这么画图形的，以矩形为例：

{% highlight js %}
// 代码1.1
// 先要拿到dom元素
var c = document.getElementById('myCanvas');
var cxt = c.getContext('2d');
// 表示从x为0,y为0的地方，开始画一个150 * 75的矩形
cxt.fillRect(0, 0, 150, 75);
{% endhighlight %}

程序猿沉吟片刻，虎躯一震，俺不是刚用了复杂工厂模式(注1)嘛。于是写出了以下代码：

{% highlight js %}
// 代码1.2
var Shape = function (id) {
  this.id = id;
  this.el = document.getElementById(id);
  this.cxt = this.el.getContext('2d');
};
Shape.prototype.draw = function () {
  throw new Error('俺是父类抽象方法，继承了之后来重载好伐');
}

var Rectangle = function (){};
// 继承父类
extend(Rectangle, Shape);
Rectangle.prototype.draw = function (x, y, width, height){
  this.cxt.fillRect(x, y, width, height);
  console.log('我画了个矩形！');
}

var Circle = function (){};
extend(Circle, Shape);

Circle.prototype.draw = function (x, y, radius) {
  // 这里看不懂没关系，就当作一个画圆的程序就好了
  this.ctx.beginPath();
  this.ctx.arc(x, y, radius, 0, Math.PI * 2, false); 
  this.ctx.stroke();
  console.log('我画了个圆！');
}

// 测试
var myRectangle = new Rectangle('mycanvas');
myRectangle.draw(0, 0, 150, 75);

var myCircle = new Circle('mycanvas');
myCircle.draw(100, 100, 20);

{% endhighlight %}

extend函数为继承函数（注1）。

于是乎，这个程序就跑起来了，跑得还挺好。画了一个矩形和一个圆。

但是程序猿看这着两团黑乎乎的东东（canvas默认颜色为黑），心里头想：卧槽，以那只的操性，后面肯定叫劳资上颜色，tmd。

## “咱都是做互联网的嘛，唯一不变的，只有变化”

> 第二天。
>
> 产品狗：我看了你的demo,　那个...
>
> 程序猿：敠要改需求了是吧。加颜色是吧。
>
> 产品狗：咦，你咋知道我想说啥，不错啊，产品意识加强了嘛！
>
> 程序猿：你大爷的，是个人都知道要加这个吧...
>
> 产品狗：嗯嗯，我们要加红绿两种颜色，然后如果画完了，再打印出来，比如：“我画了个绿色的圆！”
>
> 程序猿（看了看代码）: 卧槽，能打印像这种“我画了个green的圆”吗...
>
> 产品狗：尼煤啊，刚说你有产品意识呢，有这么SB的log吗？按我说的来！
>
> 程序猿：...

在这里，程序猿说这句“我画了个green的圆”话是有一些考虑的：

* canvas设定颜色可以识别 `green`这种单词，和 `css` 里面是类似的。
* 如果可以直接打印出green, 意味着在子类的 `draw` 方法中加一个颜色参数即可。
* 现在问题是，需要打印出来“绿色”，意味着需要有个对应关系，如 {green: '绿色'}这样的东东。
* 还有一个问题，有几种颜色？这是不固定的。理论上讲有N种。

所以，下面的这种代码：

{% highlight js %}
// 代码1.3
var redRectangle = function() {};
extend(redRectangle, Rectangle);
redRectangle.prototype.draw = function (x, y, width, height){
  this.cxt.fillRect(x, y, width, height);
  console.log('我画了个红色的矩形！');
}
{% endhighlight %}

这显然是不行的，有N种图形，还有N种颜色，这样的代码难道要写 `N * N` 次吗？

看来把颜色单独抽离出来形成个 `Color` 类。

程序猿又想到了在 `复杂工厂模式` 里面的 `运行时确定` 的逻辑：

> 实例化推迟到子类，需要使用的时候才执行。

程序猿继续想， `Color` 是父类，然后各种颜色是它的子类，到运行时才执行。
(啪啦啪拉敲键盘声)

{% highlight js %}
//　代码1.4

// ***** start 代码1.3修改的部分******
// 这里加入了Color类作为参数
Rectangle.prototype.draw = function (x, y, width, height, Color){
  this.cxt.strokeStyle = Color.color;
  this.cxt.fillRect(x, y, width, height);
  console.log('我画了个' + Color.text + '色的矩形！');
}

Circle.prototype.draw = function (x, y, radius, Color) {
  this.cxt.strokeStyle = Color.color;
  // 这里看不懂没关系，就当作一个画圆的程序就好了
  this.ctx.beginPath();
  this.ctx.arc(x, y, radius, 0, Math.PI * 2, false); 
  this.ctx.stroke();
  console.log('我画了个' + Color.text + '色的圆！');
}
// ***** end 代码1.3修改的部分******

var Color = function (){};
Color.prototype = {
  text: '',
  color: ''
}

var GreenColor = function (){};
extend(GreenColor, Color);
GreenColor.prototype = {
  text: '绿',
  color: 'green'
};

var RedColor = function (){};
extend(RedColor, Color);
RedColor.prototype = {
  text: '红',
  color: 'red'
}

// 结合代码1.3一起测试之
var red = new RedColor();
var myRectangle = new Rectangle('mycanvas');
myRectangle.draw(0, 0, 150, 75, red);

var green = new GreenColor();
var myCircle = new Circle('mycanvas');
myCircle.draw(100, 100, 20, green);

{% endhighlight %}

> 这样上面的代码就显得很清晰，因为Shape和Color是解耦的。API调用也很方便。
>
> 我们来分析一下，为啥这样的代码会更优雅呢？

## 看下图，秒懂

  原来是这样
  
  {% highlight js %}
                   ----Shape---
                  /            \
         Rectangle               Circle
        /         \             /        \
  GreenRectangle  RedRectangle BlueCircle RedCircle
  {% endhighlight %}

  变成了这样
  {% highlight js %}
          ----Shape---                        Color
         /            \                       /   \
  Rectangle(Color)   Circle(Color)           Blue   Red
  {% endhighlight %}

好了，我们再回来看GOF的定义：

> - 桥接模式用于弱化它与使用它的类和对象的耦合。
> - 桥接模式作用是将 `抽象` 与其 `实现` 隔离开来，以便二者 `独立变化` 。

上面的例子中，`Shape` 和 `Color` 是 `独立变化` 的。这样就减少了耦合，代码可读性更好，API调用也更加优雅。

这便是 `桥接模式` 。是否明白呢亲？

以上。



## 备注

关于 `复杂工厂模式` 可以看我的 [这篇文章](http://clancyz.github.io/blog/2015/09/30/Javascript-Design-pattern-factory/)。

关于 `extend` 函数可以看我的 [这篇文章](http://clancyz.github.io/blog/2015/07/27/javascript-prototype-extend/)。

