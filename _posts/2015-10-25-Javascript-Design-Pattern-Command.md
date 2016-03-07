---

layout: post

title: 大白话Javascript设计模式之命令模式(Command)

date: 2015-10-25 18:55:20

summary: 这世界上没有什么事是撸上十个串不能解决的；如果有，那你就说：「老板，再来十串」。

categories: blog

---

## 什么是命令模式

> - 命令模式旨在解耦请求的发送者与接收者，发送者与接收者之间没有直接引用关系，发送请求的对象只需要知道如何发送请求，而不必知道如何完成请求。
> - 命令模式将请求封装为对象，从而使我们可用不同的请求对客户进行参数化；对请求排队或者记录请求日志，以及支持可撤销的操作。

这个系列的惯例，还是用具体的例子来解释。今天我们的例子是「撸串」。

## 混乱的小烧烤摊儿

某天晚上，你胃里的馋虫嗷嗷叫，想到撒着孜然淌着油的羊肉串，哈喇子流了一地。

然后你来到个挺热闹的烤串摊儿，烤串的只有一个，自己烤自己收钱。吃烤串的人挺多。

你上去：“老板来十个羊肉串十个肉筋不要辣椒多放孜然”

老板：哦

10分钟过去了...

你：还有几个人到我呐

老板：哦，你前面还5个人

又20分钟过去了...

你：卧槽，你咋回事，我比他先来的，咋先给他？

老板：哦哦这是你的，呐，给你5个串。

你：我要的是10个啊，还要10个肉筋，还有这有辣椒啊我不要辣椒啊啊啊

老板：哦哦不好意思，人太多了记不过来了，下个给你烤

又20分钟过去了...

你：卧槽，怎么还没我的？

老板：哦哦，马上给你烤

你：妈的，劳资不吃了！

你就准备回家了。边走边想，妈的，这个傻逼，不懂拿个小本儿记一下啊？不懂给个号啊？记性还这么差，弄得劳资没得吃！

> 上面的事件中，可以看出请求的发送者和接收者之间是「紧耦合」的；请求一多的情况下，很容易发生混乱。你想到的「拿个小本记一下」，实际上就是「解耦合」的一种方法。

## 正经的烧烤店

你走到半路，还是馋得不行，看来今晚上不吃到嘴里是睡不着了。

路过一家正经的烧烤店，你一看，我去，串一个要十块！不管了，吃吧！

想想一个人吃也没啥意思，得叫个人来一起吹牛逼，于是你打电话叫了另一只单身狗，小明。

小明：尼玛，不是要我来买单的吧。

你：我草，谁让你买单了，今天我请客。

小明：好呀好呀好呀

你嘀咕：真是个屌丝...额看来这么一吃明天开始得啃馒头了...

到了店里坐毕开始点餐了。

你：服务员，来十个串十个肉筋十个鸡翅两个大腰子～

服务员：好的（拿个笔，记上了）

小明：我要和他一样的

服务员：好的（继续记）

你：卧槽，吃得了那么多吗你，我是点我们两个的！服务员，他说的不要了哈

服务员：好的（划掉）

小明：我去，我今天晚上没吃饭，想多吃点儿。那就再来十个串吧

服务员：好的。那就帮您下单了。

（20分钟后）

服务员：您好，这是您的串

额，终于可以大快朵颐了！撸起！

## 效率为什么高？

好，由此，我们可以观察出：

> - 正经烧烤店之所以效率高，首先服务员记录了详细的点菜内容；如果想撤消或修改点单中的内容，告诉服务员即可；有了这个记录就不怕遗忘了；烤肉师傅按每桌客人点单的先后顺序来烤。
> - 点单的客人（请求发送者）只要跟服务员交互即可，实现了和烤肉师傅（请求接收者）的解耦。

## 模拟门店的下单行为

好，我们来模拟门店的下单行为。

首先，既然我们想到了高效的关键是「服务员」，应该想到增加一个服务员类。那么，实现解耦的对象服务员干了什么事呢？

1. 从客户处接收所有的请求（包括修改，撤消等），形成一个点菜单。
2. 把请求告诉烤肉师傅：烤N个串，烤N个鸡翅... 

相当于服务员对烤肉师傅发送了一条条的「命令」，烤肉师傅执行`烤串`, `烤鸡翅`。

换句话说，一条命令对应着烤肉师傅的一种方法。

首先我们来实现烤肉师傅类：

{% highlight js %}
var Barbecuer = function (){};
Barbecuer.prototype.roastMutton = function (num) {
  console.log('烤了' + num + '串羊肉串');
}
Barbecuer.prototype.roastRibs = function (num) {
  console.log('烤了' + num + '串肉筋');
}
// ... 其他的雷同，不废话了

{% endhighlight %}

然后，我们来实现服务员类。

{% highlight js %}
var Waiter = function (){
  // 订单是由一条条的“命令”组成的。写订单就是在写一条条的命令
  // 把订单设为一个数组
  this.orders = [];
};

// 服务员的主要任务之一：写订单

Waiter.prototype.writeOrder = function (command) {
  // 把命令写到订单中
  this.orders.push(command);
}

// 这里，实现一个数组的remove方法, 在取消订单时使用
Array.prototype.remove = function (v) {
  var index = this.indexOf(v);
  if (index > -1) {
    this.splice(index, 1);
  }
}

// 取消掉某一个命令
Waiter.prototype.cancelOrder = function (command) {
  this.orders.remove(command);
}

// 修改某一个命令,即修改某种串类的数量
Waiter.prototype.modifyOrder = function (command, num) {
  var index = this.orders.indexOf(command);
  if (index > -1) {
    this.orders[index].num = num;
  }
}

Waiter.prototype.notify = function () {
  // 把订单中的每一条命令都执行
  this.orders.forEach(function (cmd) {
    cmd.executeCommand();
  })
}
{% endhighlight %}

上面例子中的`notify`方法中，我们给出了一个`cmd.executeCommand`的实现。

如前所述，这个`执行命令`即是对应烤肉师傅烤相应的串，如`roastMutton`, `roastRib`等。

那么，我们剩下来只要实现`Command`类与烤肉师傅的烤肉方法的对应关系即可。

实现`Command`类时，如`烤羊肉串`，`烤肉筋`都是`Command`的子类。

同时，为了实现对应关系，需要将烤肉师傅作为参数传入：

{% highlight js %}
var Command = function (barbecuer, num){
  this.barbecuer = barbecuer;
  this.num = num;
};
// 抽象方法
Command.prototype.executeCommand = function (){
  throw new Error('俺是父类抽象方法');
}

// 烤羊肉串命令，继承Command
var roastMuttonCommand = function (barbecuer, num){
  roastMuttonCommand.superClass.constructor.call(this, barbecuer, num);
}
extend(roastMuttonCommand, Command);
roastMuttonCommand.prototype.executeCommand = function () {
  // 对应上烤肉师傅的roastMutton方法：
  this.barbecuer.roastMutton(this.num);
}

// 烤肉筋命令与上面雷同，不废话了

{% endhighlight %}

ok, 下面我们来看实际的客户端调用。假设我们点单过程是这样的：

1. 点10个串，点10个肉筋
2. 取消掉10个肉筋
3. 修改10个串为5个
4. 下单

{% highlight js %}
// 首先，烤肉师傅和服务员得就位
var myBarbecuer = new Barbecuer();
var myWaiter = new Waiter();
// 点10个串
var command1 = new roastMuttonCommand(myBarbecuer, 10);
// 服务员记下
myWaiter.writeOrder(command1);
// 点10个肉筋
var command2 = new roastRibCommand(myBarbecuer, 10);
// 服务员记下
myWaiter.writerOrder(command2);
// 取消掉10个肉筋，服务员划掉
myWaiter.cancelOrder(command2);
// 修改10个串为5个，服务员改掉
myWaiter.modifyOrder(command1, 5);
// 下单, 师傅烤串去
myWaiter.notify();

{% endhighlight %}

上面的例子很符合实际生活的体验。这便是`命令模式`。毕竟设计模式都是来源于生活的嘛。

## 总结

我们再来看看命令模式的好处：

- 比较容易设计一个命令队列。在我们的例子中order即是命令队列。 
- 对命令有详细的记录。
- 对命令可执行修改和撤销。
- 增加具体的命令类非常容易。
- 关键的一点，实现了请求发送者和接收者的解耦。


是否明白呢亲？

以上。


注：关于extend原型继承函数及超类superClass可以看我的[这篇文章](http://clancyz.github.io/blog/2015/07/27/javascript-prototype-extend/)。
































