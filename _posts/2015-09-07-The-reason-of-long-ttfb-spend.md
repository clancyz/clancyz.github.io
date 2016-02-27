---

layout: post

title: TTFB响应时间缓慢，到底是谁的问题？

date: 2015-09-07 23:06:21

summary: 这次给我的教训就是，在你认为某问题的原因不是你导致的而需要沟通时，一定要把当前这个问题定位清楚，因为，这个问题往往不是“你认为的”　那样。

categories: blog

---

### 问题发生的背景

这篇文章部分来源于我做的一次性能优化，具体的背景可看我的[这篇文章](http://clancyz.github.io/blog/2015/08/21/A-Project-refactor-notes/)。

当时，我在我自己的机器（thinkpad x240）上测试，发现页面的平均加载速度为 `3.92s`　。

这个显然是不能忍受的，因为我们这个项目是局域网项目，加载的资源也没有多少个，怎么可能会这么慢？

于是我进行了分析。

### Devtool大法好~

我用Devtools里面的 `timeline` 工具查看了耗时分布的情况，并截了以下的图：

图1: 当时某页面的所有资源请求耗时

![7.png](/assets/images/7.png)


图2: 对于某个资源的具体耗时

![8.png](/assets/images/8.png)

图3　当时另外一个局域网平台的耗时

![9.png](/assets/images/9.png)



当时我的分析如下：

> - 下载的资源都较小，最大不过36kb。ContentDownload时间几乎可忽略。
> - 页面只有33个request，除去base64编码的图片（基本是0ms）和ajax请求（DomContentLoaded不计入）只有26个，这个请求数量并不算多。
> - 从图2可看出，主要资源加载的时间处于Waiting(TTFB)这一栏，即浏览器发出请求后，1.22s之后才收到服务器响应（注：TTFB即Time To First Byte，即从request sent至收到服务器发出的首字节的时间）。
> - 计算了所有静态资源的TTFB时间，得出页面平均的TTFB时间为：883ms。 
> - 对比同类型的内部（局域网）平台的资源加载时间），可见，平台加载一个资源的服务器响应时间大大超出了正常时间，主要为TTFB时间加载到了百倍级别。而正常的公网网站，TTFB时间也应控制在100ms以下。

然后我得出了结论：

> 这特么绝壁是后端的问题。为啥别人的平台返回那么快，咱们平台返回这么慢？是不是服务器设定的问题？

## 撕逼开始

于是我带着我的电脑去找了一个后端的同学，又像上面一样给他展示了一遍。

他看了也皱了眉头，这咋回事？

我说：这个很明显的，是你后端的问题，尽快解决吧，现在这慢得一b，很影响用户体验的。

他说：OK，我去看看服务器的相关设定。

## 被打脸

过了一会哥们蹭蹭跑过来：你的测试方法是不是有问题，我用我的机器看了一下，没你这么慢啊，这TTFB时间很短的。

我一看傻了，他的机器上，同样打开timeline工具，几乎是没有TTFB时间的。资源加载的速度比我的机器快多了。

> 卧槽，这咋回事？难道是我的人品有问题？

## 开始YY

首先我先又在自己机器上看了一遍，TTFB时间还是很长，跟原来测试结果没啥区别。

我先YY，难道是我网的问题？大家都连着wifi的啊。难道是我网卡的问题？难道是我的机器性能不行？

这个首先被排除了，那哥们就离我两三米远，为啥人家没问题。而且我连其他的局域网网站，TTFB的时间也几乎没有。我电脑是X240，他的还是早一年的X230呢。

那么，核心问题就集中在一点：

> 为啥我连接其他局域网网站TTFB时间都很短，就连自己的平台TTFB时间这么长？

## 找出罪魁祸首

我想了半天越想越迷糊，始终找不到这两者的必然联系。

然后我就去 `蹲坑` 了。

事实证明 **蹲坑大法好** 啊，我蹲着蹲着，突然想到：

> 我TM的肯定人为地干了什么事！

那我人为地干预了啥呢？干预了我和这个平台的连接？有什么东西能干预我和这个平台的连接？

> 卧槽。。。。 肯定是这玩意，`Fiddler` !

`Fiddler` 相信很多FE同学都用过，是一个 `代理软件` ，我们主要用来调试线上问题的。Fiddler的主要作用有：

> - 作为一个代理，可以替换浏览器加载的文件，比如线上有个文件是a.js, 你发现出了bug, 可以用你本机修改过的a1.js来替代,　那么你的浏览器加载的是你本机的这个a1.js，就可以看你修改的内容是否符合要求。
> - 可以设定过滤器，设定A网站使用fiddler代理，B网站直接连接

我当时设定的过滤器规则是：对于 `xxx.baidu.com` (我们平台的域名)，都使用fiddler来代理，方便调试线上问题。在开发过程中，我陆续也设定了好几个js使用本地js替代加载。

那么问题就很清晰了：

> - 我连接项目平台时，相当于中间多了个代理，做了替换资源等操作，所以TTFB时间明显变慢。
> - 我对当前平台做了fiddler代理，但对其他局域网平台没做，所以其他局域网平台的TTFB时间就很短。

这样结论就是，完全是我自己的问题。只是没有注意到 `fiddler` 这货的存在。

想起之前还blabla写了一堆关于这TTFB的时间给大家看说后端性能差，就觉得很丢脸...

## 总结

这次给我的教训就是，在你认为某问题的原因不是你导致的而需要沟通时，一定要把当前这个问题定位清楚，因为，这个问题往往不是“你认为的”　那样。

进行测试的时候要充分，找不同环境来复现（例如，找个别人的机器来试试是否有同样的问题）。






