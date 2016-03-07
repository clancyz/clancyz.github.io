---

layout: post

title: 大白话Javascript设计模式之单例模式(Singleton) / 模块模式(Module)

date: 2015-09-27 23:00:09

summary: “你心里如果有人了，只能有一个。”

categories: blog

---

传统上对单例模式的解释，就一句话：

> 一个类只能被实例化一次，且这个实例易于被外界访问。

当然，我想写的这个系列叫 `大白话Javascript设计模式` ，如果像以上这么写，我还不如直接上网上copy一堆来。

假如你是个玩过类似wow (魔兽世界)之类的玩家，下面这个例子就很好理解：

> - 你建了个角色是个侏儒术士，　这个职业有个技能就是召唤一只宠物叫 `恶魔猎手`。
> - 当你想召唤这只宠物出来的时候，你要使用你的技能`召唤恶魔`。第一次使用这个技能时，你得耗费2s的 `施法时间`。
> - 一旦你召唤出来之后，你想: 我要再召唤几只出来！
> 于是你再次使用 `召唤恶魔` 这个技能，此时使用此技能都是`瞬发`的，就是不用再耗费2s啦。但是并没有什么卵用，还是只有原先那一只在那傻楞着看你。
> 很简单，系统还能让你召唤个100只让你毁天毁地毁银河系啊，最多只能让你带一只！
> - 如果敌人把它打死了，然后你再使用 `召唤恶魔`　时，又得耗费2s的施法时间，重新召唤。


## 最简单例模式(最简模块模式)：恶魔猎手

最简单的单例模式代码, 就是一个{}表示的object对象，即**对象字面量**。

例如，你定义了一个`恶魔猎手` 的对象字面量：

{% highlight js %}
// 代码1.1
var DevilHunter = {
  // 这货有100滴血。
  hp: 100,
  // 这货只会一招，咬人
  bite: function () {
  }
}
// 验证之
var nima = DevilHunter;
var nidaye = DevilHunter;
consoel.log(nima === nidaye) // => true
{% endhighlight %}

在这个例子中，比如你有只恶魔猎手宠物，你现在想叫它 `尼玛` ，过了一会你觉得这名字不够霸气，又叫它 `尼大爷`　，这也就是你一时爽，它还是那只傻楞的它...

术语上的解释：

> 非常明显，`尼玛`和`尼大爷`都是DevilHunter的副本，对象字面量为javascript里面的 `引用类型` ，它们都是个指向DevilHunter的指针。所以它们必然全等。
> （是不是感觉枯燥多了。。）

同时，要进一步指出的是：

> 以上例子实际也可以视作Javascript中的一种「模块实现」，即我可以说，我定义了一个DevilHunter模块，随用随取即可。


**好，那上面的DevilHunter例子，有什么问题么？（话外音：同学们，这是送分题啊~）**

很显然，DevilHunter中的所有属性和方法，都能被外界直接访问到。

换句话说，DevilHunter.bite()是调用它的`咬人`方法，那问题是，我也希望它注意素质，不能乱咬人啊，我让它咬时它再咬。

要使用js从根本上解决这个问题，只能拿个东西把这些不想暴露在外面的属性或方法包起来，那么就是使用 `闭包` 啦。

## 模块模式：使用闭包的单例模式

这种方法使用了一个**自执行函数**,在函数体内就可以定义私有变量了。
因为js的执行作用域是**函数级作用域**，说白了就是：

> - 函数内部作用域可以访问外部作用域
> - 函数外部作用域访问不了内部作用域
> - 向上搜索作用域链，直到window

所以，在函数体内定义的变量才是真正的私有变量。
代码如下：

{% highlight js %} 
// 代码1.3
var DevilHunter = (function () {
  var hp = 100;
  var bite = function () {
  };
  return {
    // 只有我下令攻击的时候，再去咬人~
    attackCall: function () {
      bite();
    }
  }
})();
{% endhighlight %}

在这里，DevilHunter是一个立即执行的函数，最终的结果也是返回一个对象字面量。与之前的不同是，可以自由定义私有变量了。

> 实际上，这便是「模块模式」的实现。
>
> 我们相当于声明了一个DevilHunter的全局模块，有一个公开方法`attackCall`, 同时，匿名函数的闭包维持了私有的内部状态。
>
> Javascript中实现模块，除了对象字面量和闭包之外，还包括：
>
> - AMD / CMD / CommonJS
> - ES6 Module
>
> 这些模块的使用方式，在本文中就不做详细介绍了。

**1.3的例子「模块模式」， 其实实用性已经不错了。那，还有什么可以改进的地方？**

## 严格单例模式

代码1.2和1.3的共有问题：

- 有可能引起性能问题。
- 仍然不是严格可实例化的类。

> 为什么可能会有性能问题捏？ 


设想一个这样的场景，需要设计一个术士 `Warlock` 类，它的作用是：

- 可以通过`召唤恶魔` 这个技能，召唤一只 `恶魔猎手`。
- 对外只暴露了一个`attack` 方法，流程是：先召唤出来，再使用恶魔猎手去咬人。

。那么，可以搞出以下一个版本的伪代码：

{% highlight js %} 
// 代码1.4
var Warlock = (function () {
  //　先做一个初始化，即召唤它
  var devilHunter = callDevil();

  function callDevil () {
    // 要2s时间才能召唤出来，这期间都在blablabla
    blablabla();
    return DevilHunter;
  }

  return {
    attack: function () {
      devilHunter.bite();
    }
  }
})();

{% endhighlight %}

可以明显地看出，1.4中的这个Warlock, 做了一些初始化步骤，产生了`devilHunter`这只`恶魔猎手`。

在现实的开发过程中，使用singleton单例模式时，避免不了需要初始化一些步骤。

如果这个初始化步骤耗时比较长（举上面的例子为例，如果blablabla的过程中掉链子花了20秒呢？），就会引起一定性能问题。

> 这个性能问题一般是在定义一个需要 `加载众多配置项` 或者 `读取大量数据` 的Singleton时会出现。
>
> 一般情况下，1.3的例子「模块模式」已经够用拉。


在这种情况下，在脚本加载时，我们其实有其他优先级更高的事情做，,即我们希望叫恶魔猎手攻击的时候，再给我去做那些初始化(花2s时间召唤)的工作, 即所谓的 `lazyload`。

所以我们现在的需求是：

- 保持DevilHunter实例唯一性的情况下，按需加载，使用时才实例化。
- 对外暴露一个获取实例 `getInstance`方法。在我们的例子中，即是召唤一只`恶魔猎手`，所以这个获取实例方法叫 `callDevil`。

{% highlight js %}
var Warlock = (function (){
  // 构造函数，也是闭包中的一个“私有类”
  function DevilHunter(hp) {
    blabla(); //耗费2s
    this.hp = hp;
    this.bite = function (){
    };
  }
 
  var instance;

  return {
    callDevil: function (hp) {
      if( instance === undefined ) {
        instance = new DevilHunter(hp);
      }
      return instance;
    },
    attack: function () {
      var devilHunter = Warlock.callDevil();
      devilHunter.bite();
    }
  };
})();

// 测试之
Warlock.callDevil({hp: 100}) 
// => DevilHunter {hp: 100}
Warlock.getInstance({hp: 200})
// => SingletonConstructor {hp: 100} 未改变

{% endhighlight %}

## 总结

> - 模块模式可以是简单对象自变量，也可以使用闭包创建；
> - 严格单例模式需要可实例化，对于资源密集或者配置开销很大导致初始化消耗性能较多的单例，可以进行惰性实例化。

是否明白呢亲？

以上。












