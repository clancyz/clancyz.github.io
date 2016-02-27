---

layout: post

title: 某前端项目重构及性能优化记录

date: 2015-08-20 22:06:21

summary: 我的一次前端“推倒式重构”的经历。由我总结的一篇项目文档修改而来，因为是内部项目，隐去了具体项目内容。经过重构和优化后，项目代码的结构更加清晰，易于部署，可维护性更好；页面普遍的加载速度快了1s以上。

categories: blog

---

## 项目背景

我还是从前端的技术角度来阐述。

首先，在框架选择上，这个项目的使用的主要技术是：

> - fis-plus，用于工程化
> - jQuery + 各种自定义plugin，开源的不太满足需求，自己写
> - D3，需要的图形化结果没有任何一个D3的demo可以满足的，基本是自己做二次开发的

项目在我加入时，已经由一个FE同学开发了2个月。然而，当项目需求井喷的时候，哥们说要创业，走了...

后来我就在 `1个前端，9个后端` 的情况下死撑... 一个月码4w多行js+html+css，最多的一天有checkin近2000行js... 当然，那次2000行主要是用jQuery写业务代码，可能跟大牛比起来微不足道，但是我之前没经历过这样的压力，现在回想起来，也是段难得的经历。

好了有点跑偏，这个项目后来在 `需求相对稳定` 的时候，相对得闲了，就想做些优化了...

## 项目结构的问题

当时的我接手后的项目结构是这样的：

![6.png](/assets/images/6.png)

如上图的项目目录，实际上并没有遵循 `fis-plus` 的 `模块规范`。　带来的问题是：

1.  没有抽离公共模块和页面模块，页面和母版、插件等混杂在一起；

2.  整个项目只有一个fis-conf.js配置文件（用于fis编译），每次改动要发布整个项目；如果项目加入optimize压缩静态资源等策略，则会特别耗时。就当时项目规模而言，采用md5/optimize压缩后，一次编译耗时接近3分钟。

3.  没有使用组件化思想。具体如下：

    * 平台的前端母版页没有进行组件分离，header/footer的行为都集成在一个文件中，且未模块化，代码混乱；
    * 有两个页面使用了左导航栏，但未进行组件化复用，造成效果不一；

4.  未进行模块化改造。具体如下：

    * 前端与后端交互的接口地址列表应该模块化并统一维护；
    * 前端的各页面的预处理中，将产品线等urlParam的处理为一个全局变量window.util且与header/footer的行为混杂，难以解耦和维护；
    * 前端的模块加载器采用fisp默认的mod.js, 不利于amd标准（一种通用的前端模块化标准）的模块化加载（如echart）。
    * 前端现在所有插件基本均为一次加载全部，影响页面性能。

## 具体的代码问题

页面代码问题，主要问题有以下几点：(涉及业务的已经略去)

1.  代码没有结构化，UI、事件绑定和数据请求混杂在一起，代码注释少，维护难度大。
2.  存在部分无用逻辑。
3.  某些页面注册了多个domReady事件。
4.  选择器存在多次重复使用的问题。
5.  函数重复定义，如timeFormat。
6.  某页面的 `树状表格` 重复N次DOM操作，渲染性能低下。如遇到稍大的表格，其渲染时间可超过5s，严重影响体验。

## 当时自己的一个失误

当时自己在写这篇重构文档时提到了一个问题，即页面加载速度缓慢。

当时使用 `DevTool` 来观察页面加载时间时，发现：

> 所有页面的DomContentLoaded在4s左右，Load在5~6s之间。
>
> 卧槽，这也太慢了吧！

当时我经过分析，应该是 `服务端` 的问题，不是前端问题。还给后端同学提了建议让他们去优化。结果证明，完全是我自己的问题，我也感到很羞愧。关于这个问题我也专门总结了一番，可以看我的 [这篇文章](http://clancyz.github.io/blog/2015/09/08/The-reason-of-long-ttfb-spend/)。

## 优化过程

### 组件化：

1.  抽离项目代码为common公共部分和页面部分；
2.  抽离页面header和footer成为组件；
3.  抽离可多页面共用的页面组件，如左导航成为组件；
4.  表格组件优化，暴露出refresh接口，在新的ajax请求完成渲染表格后或翻页后触发refresh事件。


### 模块化：

1.  将预处理的js、需要在多个页面中使用中的函数进行模块化。
如通用功能函数util； 时间函数timeformat；获取urlParam函数getUrlParam等。

2.  页面js处理，将可以按需加载（用户点击时才加载）的js资源（如复制插件，绘图插件等）使用require.async异步加载。


### 工程化：

1. 打包策略：将资源打包，减少http请求。在打包的考虑中要同时考虑 `同类模块` ， `打包后的大小`　， `浏览器并发`　这三要素；目前的css可以全部进行打包；js文件如果全部打包会把不需要的js都加载，造成单页加载速度较慢，故将js打包策略集中于模块化的js。

2. 时间戳缓存策略：页面资源打时间戳（天为单位）；这里不打md5戳的原因主要是上线比较频繁，如果打md5戳会造成源代码量增加过快。调整fis的roadmap.path中的query即可。

3.  压缩策略（可选）：压缩代码，减少下载量。由于在局域网环境下资源加载速度较快，压缩后带来的优化并不大；而压缩混淆代码后如有线上问题，将不方便调试。现在没有线下黑盒环境，后续可考虑此措施。


4.  改造fisp原生mod.js为amd模块加载器：

    * 改造为 `esl.js`　方便使用amd规范的插件如 `echarts`
    * 改造fis.conf中modules.postprocessor / settings.postprocessor
    * 修改fis smarty插件中的FISResource.class.php为支持amd标准的模块加载
  


### 业务代码优化

#### 这里我主要是提出了一些　`代码规范`,　评审通过后要求大家遵循。

1. 首先，对页面代码（每个页面对应的jquery代码，不包含插件组件）做如下规范：
  * 页面代码目录中static/* 中只允许有页面代码，不允许有公共组件。
  * 与后端交互的接口路径统一放到config文件中。
  * 页面代码使用require引入所需资源，页面tpl中不使用模板require引入资源；
  * 页面代码需要使用自执行函数以防变量污染；
  * 页面代码中第一行需要声明严格模式（use strict）；
  * 代码中需要使用的jquery选择器统一定义在页面最上方；
  * 代码中需要使用的页面级全局变量定义在页面最上方，选择器下部；
  * 定义一个页面object变量，定义initUI()和bind()函数，执行加载UI与绑定事件两种功能；
  * 定义一个fn的object用于写页面功能函数；
  * 所有function均需要加入业务功能注释。

2. 对于某页面A：

    * 代码重构，优化代码逻辑，去掉无用逻辑。
    * 页码计算的c方法去掉（可调用组件）。
    * 函数整合。

3. 对于某页面B：

    * 代码重构，优化代码逻辑，去掉无用逻辑。
    * 优化页面表格加载方式，使用模板引擎减少dom操作，加载完后再考虑二次处理。

## 总结

以上便是整个重构优化的过程，可能没有结合具体的例子来讲，有些笼统；但是，经过这次重构和优化后，项目代码的结构更加清晰，易于部署，可维护性更好；页面普遍的加载速度快了1s以上。对自己也算是一次完整的重构经历，也有一定的成长。









