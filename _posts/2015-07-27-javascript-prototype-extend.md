---
layout:     post
title:      Javascript：重温原型链继承（extend）
date:       2015-07-27 21:00:22
summary:    2年前学习js的时候写的笔记，重新编排了下，修改了部分内容。这也是我常问候选人的面试题目之一。
categories: blog
---

# 原型链继承

## 最简单的类声明

估计所有学过js的人都会这个吧 :p

{% highlight js %} 
function Person (name) {
  this.name = name;
}

Person.prototype.getName = function () {
  return this.name;
}
{% endhighlight %}


## 原型链继承的标准代码

现在用一个Author类继承Person类。

{% highlight js %} 
function Author (name, book) {
  // 父类使用call到子类的scope上
  Person.call(this, name); 
  // Author的自有属性
  this.book = book; 
}
// 子类的prototype等于父类的新实例
Author.prototype = new Person();
// 子类的原型构造器等于它本身  
Author.prototype.constructor = Author; 
Author.prototype.getBook = function (book) {
  return this.book;
}

{% endhighlight %}
做一个实际的使用：
{% highlight js %} 
var Jim = new Author('Jim', 'Jim\'s Book');
console.log(Jim.getName()); // => Jim
console.log(Jim.getBook()); // => Jim's Book

{% endhighlight %}

> **那么问题来了。为什么类继承要这么写？这里面发生了什么？**

首先看prototype属性的定义：

> - js中任意对象都有prototype属性，记为**\_\_proto\_\_**.
> - 原型链相当于一个链表，链表中的一个位置相当于指针，指向下一个结构体；
> - 当定义一个prototype的时候，相当于把该实例的\_\_proto\_\_指向一个结构体，这个被指向结构体即为该实例的原型。
> - 默认指向: Object.prototype 
> - Object.prototype.\_\_proto\_\_ === null 此为顶端。

如下图的foo对象：

{% highlight js %}
var foo = {
  x: 20,
  y: 30
}
console.log(foo.__proto__) => object{}
{% endhighlight %}


## 原型链如何产生？

看一段代码。
{% highlight js %}
var a = {
  x : 10,
  add: function (z) {
    return this.x + this.y + z;
  }
}

var b = {
  y: 20,
  __proto__: a //把b的__proto__指针指向a
}

console.log(b.add(30));  // => 60


{% endhighlight %}
如上代码所示，以b为例，当b有明确的\_\_proto\_\_属性时，执行b.add(30)的时候，先在自己的属性里面找，没找到，就到上级的\_\_proto\_\_属性中找，如果还没找到就继续往上找......这样就构成了一条**原型链**。

> 回到原来的问题： 为什么要使用Author.prototype = new Person()？

实际做的事情：

{% highlight js %} 
Author.prototype = new Person();
// 执行这一句的时候，实际执行了下面的事情：
Author.__proto__ = Person.prototype
// 验证之
Author.prototype.__proto__ === Person.prototype  // => true
Jim.__proto__ === Author.prototype // => true
// 把上面两行结合在一块，得到：
Jim.__proto__.__proto__ == Person.prototype 
{% endhighlight %}
所以，执行这Author.prototype = new Person()后，相当于形成了一条原型链。

> 为什么要使用Author.prototype.constructor = Author?

因为定一个函数的prototype时，默认情况下prototype属性会默认获得一个constructor属性，是一个指向prototype构造函数的指针。

所以，执行完Author.prototype = new Person()后，隐式声明了

{% highlight js %} 
Author.prototype.constructor === Person
{% endhighlight %}

而在实际的开发过程中，经常要修改子类的prototype, 比如我们又定义了一个Author的子类FictionAuthor：
{% highlight js %}
function FictonAuthor (name,book,fiction) {
  Author.call(this, name,book); 
  this.fiction = fiction;
}
FictonAuthor.prototype = new Author();

// FictionAuthor的构造函数我们希望是Author,而实际上呢？
console.log(FictonAuthor.prototype.constructor) 
// => Person

{% endhighlight %}
所以，在子类继承中我们在执行完Author.prototype = new Person()后，需要对构造函数constructor做一个修正，即：
{% highlight js %} 
Author.prototype.constructor = Author
{% endhighlight %}
这样就达到了我们想要的结果。

综上，可以总结一个extend函数：
{% highlight js %}
function extend (subClass, superClass) {
  var f = function () {}; 
  // 避免创建超类的新实例，因为它可能比较庞大，或者要执行init之类需要大量计算的任务
  f.prototype = superClass.prototype;
  subClass.prototype = new f();
  subClass.prototype.constructor = subClass;
  // 有可能调用父类方法，所以给予一个superClass属性
  subClass.superClass = superClass.prototype;
  // 同样，superClass的constructor需要一个修正。
  if (superClass.prototype.constructor == Object.prototype.constructor) {
      superClass.prototype.constructor=superClass;
    }
}
{% endhighlight %}
