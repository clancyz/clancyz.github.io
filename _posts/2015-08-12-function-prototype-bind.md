---

layout: post

title: 简明深入Function.prototype.bind

date: 2015-08-12 22:25:11

summary: 深入理解了Function.prototype.bind，你的js就不再是「小白」水平了。

categories: blog

---

## 面试场景

对于这个话题，我面试时常会按照以下的顺序来对候选人进行提问：

> - 说一下对Javascript中的 `this` 关键字的理解。-- 答对，30分
> - 如果能说出来 `this` 的作用，那么，手写两段程序，说明 `this` 啥时候指向原对象，啥时候指向 `window` 。-- 答对，50分
> - 有什么方式可以改变 `this` 的指向？ --　答对，70分
> - 引出 `Function.prototype.bind` , 是否有用过，怎么用　--　答对，80分
> - 写一个 `Function.prototype.bind` 的 `polyfill` (浏览器兼容方案)。 -- 写得基本正确能运行，100分
> - 能考虑到constructor模式调用 -- 120分!

下面是我的一些理解。

## this关键字

`this` 关键字表示当前调用函数或方法的所有者。this的值取决于它的 `调用模式` 。

> - 对于一个全局函数，表示的就是 `window` 对象；
> - 对于一个对象的方法，表示的是该对象的实例；
> - 在一个事件句柄中，表示的是接收到该事件的元素。
> - 对于一个类的实例，表示的是该类的对象。


举例说明。

### 函数调用模式

{% highlight js %}
function cat () {
  console.log(this);
  this.name = 'miaomiao';
}

// => window
{% endhighlight %}
即上面的第1条规则。

### 方法调用模式
{% highlight js %}
var cat = {
  name: 'miaomiao',
  sleep: function () {
    console.log(this);
    return this.name + 'is sleeping';
  }
}

cat.sleep();
// => cat
{% endhighlight %}
即上面的第2条规则。

### 事件句柄模式

{% highlight html %}
<a id="lk" href="https://www.baidu.com">click me</a>
{% endhighlight %}

{% highlight js %}
var link = document.getElementById('lk');
        link.addEventListener('click', function (e) {
            e.preventDefault();
            console.log(this);
            console.log(this.href);
        })
//=> <a id="lk" href="https://www.baidu.com">click me</a>
//=> https://www.baidu.com
{% endhighlight %}

即上面的第3条规则。

### 构造器调用模式

{% highlight js %}
function cat () {
  this.name = 'miaomiao';
}

var myCat = new cat();
console.log(myCat.name);
console.log(window.name);
// => miaomiao, 这里this指向了cat
// => undefined, 说明没指向window.
{% endhighlight %}

即上面的第4条规则。

### 改变this的指向：apply/call

{% highlight js %}
function getName (name) {
  console.log(this);
  this.name = name;
　 console.log(this);
}
var foo = {name: 'hehe'}; 
getName('xixi');
// => Window
// => Window

getName.call(foo, 'xixi');

// => Object {name: "hehe"}
// => Object {name: "xixi"}
{% endhighlight %}

上面这个例子中，调用 `getName.call(foo, 'xixi')` 时，在getName函数中的`this`就指向了`foo`; 所以再执行`this.name = name` 后，`foo` 对象的 `name` 相应地做出改变。

### 改变this的指向：bind

{% highlight js %}
function getName (name) {
  console.log(this);
  this.name = name;
  console.log(this);
}

var foo = {name: 'hehe'};

var bar = getName.bind(foo);
bar('xixi');

// => Object {name: "hehe"}
// => Object {name: "xixi"}
{% endhighlight %}

上面这个例子中，调用 `getName.bind` 后，就把`this` 绑定到了 `foo` 上。产生的结果和apply/call是一样的。

当然，上面这个例子是在 最新版的`chrome`下测试的。IE9以下的浏览器是不兼容的。

`Function.prototype.bind` 的具体浏览器兼容性请见[这里](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/bind)。

## Polyfill: Function.prototype.bind

你要写一个兼容旧浏览器的Function.prototype.bind，总得知道它怎么调用吧！

调用如下：

> fun.bind(thisArg[, arg1[, arg2[, ...]]])

`thisArg` 表示要把`this`绑定到 `thisArg`；第二个以及以后的参数`arg1, arg2, ... `加上绑定函数运行时本身的参数按照顺序作为原函数的参数来调用原函数。

那么这个顺序具体是什么样的呢？

> arg1, arg2...argN 在前，绑定函数运行时本身的参数在后。

看一个具体例子来验证：


{% highlight js %}
function getName (name) {
  console.log(this);
  console.log(arguments);
  this.name = name;
  console.log(this);
}

var foo = {name: 'hehe'};

var bar = getName.bind(foo, 1, 2, 3);
bar('xixi');

// => Object {name: "hehe"}
// => [1, 2, 3, "xixi"]
// => Object {name: 1}
{% endhighlight %}

输出了`[1, 2, 3, "xixi"]`。验证了这个结论。

### MDN的polyfill解释

下面是MDN的polyfill，咱们有了上面的例子，就比较好理解了：

{% highlight js %}
if (!Function.prototype.bind) {
  Function.prototype.bind = function (oThis) {
    if (typeof this !== "function") {
      // closest thing possible to the ECMAScript 5
      // internal IsCallable function
      throw new TypeError("Function.prototype.bind - what is trying to be bound is not callable");
    }

    var aArgs = Array.prototype.slice.call(arguments, 1), 
        fToBind = this, 
        fNOP = function () {},
        fBound = function () {
          return fToBind.apply(this instanceof fNOP
                                 ? this
                                 : oThis || this,
                               aArgs.concat(Array.prototype.slice.call(arguments)));
        };

    fNOP.prototype = this.prototype;
    fBound.prototype = new fNOP();

    return fBound;
  };
}
{% endhighlight %}


里面有几个重要的点，我逐条说明：


1. 首先判断是否原生支持，判断调用 `bind` 的是不是一个`function`类型等，略
2. `Array.prototype.slice.call(arguments,1)` 这一句：
   - 把 `arguments` 这个伪数组，除了第1个参数后的所有参数取出为一个数组。
  - `aArgs`有啥用，留着后面合并啊，看上面的[1,2,3,"xixi"]的例子~
3. `fToBind = this` : 当前这个待bind的对象；
4. `fNOP` 我不知道它为啥要起这个名，是啥的缩写？这个东东要和下面的两句 `fNOP = prototype = this.prototype` , `fBound.prototype = new fNOP()` 一起看，这一看就明白，`fNOP`是当前待bind函数的一个原型副本， `fBound` 继承了 `fNOP`。　最后把 `fBound` 返回。`fBound`是个函数，因为`bind`的结果还是个函数嘛！
5. 好了，中间这一句，总体看是这样：`return fToBind.apply(this, arguments)`; 这个arguments就是用`aArgs`和绑定函数运行时的函数给`concat`组合起来了。

看了上面的分析可能有些童鞋会有疑问：

>　为啥要搞个`fNOP`出来，然后 `fBound`去继承它？
>
>　为什么不能直接 `fBound.prototype = this.prototype`　， 然后返回`fBound`？

哎，童鞋，你还是太单纯呐。

要是按你那么搞，因为`prototype`是个`引用类型`，`fBound`又能被外界访问到的话，如果我来个`fBound.prototype = shit` 之类的，你让 `this.prototype` 情何以堪呐。

> 这里`fNOP`作为私有变量，就杜绝了改变`this.prototype`的可能性。

#### polyfill中的主要难点

好，那这里的一个___主要的难点___是：

{% highlight js %}
this instanceof fNOP
    ? this
    : oThis || this
{% endhighlight %}

此时`fNOP` 它是一个`空函数`。

  -  解读上面这一句的用意：

1. 如果 `this`　是 `fNOP` 的实例，那么就是`this`
2. 否则就是 `oThis` ,如果`oThis`为`null`或者`undefined`,　那么就是`this`

那么问题来了：

> `this instanceof fNOP`  这是在干毛？
>
> 为什么我要去判断`this`是不是一个`空函数`的实例？
>
> 这判断有啥用？
>
> 我为啥不直接写成oThis \|\| this ?

好，那我们就把MDN上polyfill的这段去掉。为了便于测试，我们把`bind` 改成 `myBind`　：

{% highlight js %}
Function.prototype.myBind = function (oThis) {
    if (typeof this !== "function") {
      throw new TypeError("Function.prototype.bind - what is trying to be bound is not callable");
    }

    var aArgs = Array.prototype.slice.call(arguments, 1), 
        fToBind = this, 
        fNOP = function () {},
        fBound = function () {
          //　把那个instanceof判断去掉
          return fToBind.apply(oThis || this,
            aArgs.concat(Array.prototype.slice.call(arguments)));
        };

    fNOP.prototype = this.prototype;
    fBound.prototype = new fNOP();

    return fBound;
  };
{% endhighlight %}

好了看下面的例子：

{% highlight js %}
function getName (name) {
  this.name = name;
}
var foo = {};

var bar = getName.myBind(foo);

bar('hehe');
console.log(foo.name);
// => hehe

{% endhighlight %}

这是显然的嘛。`myBind`函数也工作得很好。

> 那么，我们使用`new` 构造器来调用绑定后的函数`bar`，会发生啥呢？

在上面的程序下面继续：

{% highlight js %}
var xixi = new bar('xixi');
console.log(foo.name);
// => xixi, foo的name变成了xixi!
console.log(xixi.name);
// => undefined
{% endhighlight %}

上面的例子显而易见，我们当然希望`foo.name`仍然是`hehe`, 而`xixi.name`是`xixi`。
说明我们的`myBind`函数在这种情况下不好使了。

> 这说明啥？说明咱们用`new`构造器来调用`bar`时，绑定的`this`还是`foo`。

我们在`chrome`下用原生的`bind`函数试一次：

{% highlight js %}
function getName (name) {
  this.name = name;
}
var foo = {};

var bar = getName.bind(foo);

bar('hehe');
console.log(foo.name);
// => hehe

var xixi = new bar('xixi');
console.log(foo.name);
// => hehe   还是hehe
console.log(xixi.name);
// => xixi

{% endhighlight %}

可见，`bind` 函数正确绑定在了`xixi`上。

这样，我们再回头来看`this instanceof fNOP` 在对绑定函数执行`new`下调用会是啥结果：

那就相当于执行了这么一句：

{% highlight js %}
new function(oThis){
  //...
  return fBound;
}();

{% endhighlight %}

显然，这时得到的是`fBound`。`fBound`是个啥呢？是个function(){...}。

> 那么执行`this instanceof fNOP` 相当于`fBound instaceof fNOP`
>
> 是`true` ! 

如果为`true` 那么就绑定在`oThis`或`this`上，优先`oThis`;

即传进的参数，在我们的例子里面，就是`xixi`。

所以可以得出结论：`this instanceof fNOP` 这个判断分支是为了在用`new`调用绑定函数的情况下而生的。

## 总结

`Function.prototype.bind` 可以引申出很多的知识点， 甚至可以认为：

> 熟悉了这些知识点， 你的js就不再是 `小白` 水平了。

写这篇文章前专门网上走了一圈，发现没有对MDN的polyfill的详细解释，所以才打算写~

希望俺这篇文章能让你有所收获~ 

以上。







  
  

 









