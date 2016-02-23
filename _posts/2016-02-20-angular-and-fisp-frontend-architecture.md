---

layout:     post
title:      使用angular+fis-plus+oclazyload搭建单页应用
date:       2016-02-20 21:36:11
summary:    给去年年底的一个angular项目做个总结，使用angular+fis-plus搭建前端单页应用的一些小心得。
categories: blog

---


## 项目背景

此项目前端为一个单页（Single Page Application），header和footer固定，中间视图页随导航点击而变化。后端为百度内部的PHP框架ODP。
使用的主要框架和类库有：

> - angular 1.4.7
> - angular ui-router
> - angular-bootstrap
> - [fis-plus](http://oak.baidu.com/fis-plus  "fis-plus") （以下简称fisp）
> - [oclazyload](https://github.com/ocombe/ocLazyLoad "oclazyload")
> - echarts
> - d3

## 框架选型的考虑

当时angular的最新版本是1.4.7，俺学习angular的时候版本为1.2。选新不选旧的原则，就上了最新版本。
前端ui框架方面选择了angular-bootstrap, 因为组内对bs还是比较熟的。
工程化方面，后端同学是搞PHP的，以前配合的时候都是使用fis-plus, 所以继续采用。

## 项目架构

![image](/assets/images/5.png)

下面逐项说明。

> - build_offline.sh: 线下部署脚本
> - build_online.sh: 线上部署脚本
> - output: 部署脚本产生的tar包目录，执行脚本即打包dist目录生成tar包并拷贝到ODP目录解压
> - src: 源码目录
> - src/business: 业务目录，目录下是分块的业务
> - common: 公共目录，同时也一是一个fisp模块，存放公共资源
> - common/page: 单页的主页面分解，index.tpl, app.html, aside, header, nav,footer等
> - common/plugin: fisp的php插件
> - common/static: 公共静态资源目录
> - dist: 经过fis-plus编译的产出目录，供打包使用


## 搭建架构和开发的一些心得

### 业务优先，类型次之

首先，刚开始的时候想过用view/controller/directive/service这样的结构的。
（之前做一个backbone项目的时候就是使用view/model/collection/这样的分层，当时没有使用啥工程化手段）

但因为fisp是推崇 `模块机制` 的，即一个业务一个模块。
fisp编译一个模块：

`# fisp release -d dist` 

显然这种做法就被排除了，因为如果后续要修改某个页面模块时，如果是按这样划分，那么要到n个文件夹下去编译，
我们更多需要的是 `只编译修改的部分` 。
所以形成了business目录，下面一个页面一个模块，这样很清晰，也符合fisp的规范。

### 应用程序入口
由于是一个单页应用，服务器配置路由来加载index.tpl首页。
index.tpl除了加载各种公共静态资源外，会进行以下的配置行为：

> - config.js： 进行全局设定
> - config.router.js: 路由设定
> - config.lazyload.js: 按需加载设定 
> - storage.js: 缓存设定

### 使用oclazyload配合ui-router实现按需加载
首先来看[oclazyload](https://github.com/ocombe/ocLazyLoad)的官方key features:

> - Dependencies are automatically loaded
> - Debugger friendly (no eval code)
> - The ability to mix normal boot and load on demand
> - Load via the service or the directive
> - Use the embedded async loader or use your own (requireJS, ...)
> - Load js (angular or not) / css / templates files
> - Compatible with AngularJS 1.2.x/1.3.x/1.4.x

这种按需加载的模式是很有必要的，毕竟没有人想页面一进来把整个项目的所有脚本和静态资源都加载吧。。
而且自带async loader, 我觉得已经没有必要用啥requireJS, 真要用的话fisp有个mod.js也足够使用了，要用它还得改造。

oclazyload配合ui-router的代码如下：

{% highlight ruby %} 
// config.lazyload.js
angular.module('app')
    .config(['$ocLazyLoadProvider', function ($ocLazyLoadProvider) {
        $ocLazyLoadProvider.config({
            debug: true, //是否开启调试
            events: true, // 是否给console.log一个模块加载事件
            modules: [
            // 定义一个模块
            {
                name: 'ui.grid',
                files: [
                    '/static/common/vendor/modules/angular-ui-grid/ui-grid.min.js',
                    '/static/common/vendor/modules/angular-ui-grid/ui-grid.min.css'
                ]
            }
            ]
        });
    }]);
{% endhighlight %}

可以看见上述config已经定义了模块名如 `ui.grid` ，配合ui-router时直接load这个模块名就行了

{% highlight ruby %}
//config.router.js
var app = angular.module('app');
app.config(
    ['$stateProvider', '$urlRouterProvider',
        function ($stateProvider, $urlRouterProvider) {
            $urlRouterProvider
                .otherwise('/app/home');
            $stateProvider
                .state('app', {
                    'abstract': true,
                    'url': '/app',
                    'templateUrl': '/static/common/page/app.html'
                })
                .state('app.home', {
                    url: '/home',
                    templateUrl: '/static/home/page/home.html',
                    resolve: {
                        deps: ['$ocLazyLoad', function ($ocLazyLoad) {
                            return $ocLazyLoad.load(['ui.grid'])
                            .then(function () {
                                      return $ocLazyLoad.load(
                                          '/static/home/homeController.js', {cache: false});
                                  });
                        }]
                    }
                })
{% endhighlight %}

这里要注意：

> - $oclazyload是支持类似promise的链式写法的。
> - 我们在这里一般的做法是，先load依赖，最后再load业务代码 `homeController.js` 。
> - 最后跟了 `{cache:false}` 参数，$oclazyload会生成时间戳，修改上线时用户不会缓存原文件。

### 缓存设计

项目需要缓存一些默认信息，比如产品线信息。
我是使用了localStorage来缓存，在body的controller对应的$scope.setting中存储。
核心代码很简单：

{% highlight ruby %}
if (angular.isDefined($localStorage.settings)) {
    $scope.app.settings = $localStorage.settings;
} else {
    $localStorage.settings = $scope.app.settings;
}
$scope.$watch('app.settings', function () {
    $localStorage.settings = $scope.app.settings;
}, true);
{% endhighlight %}
这样，在页面开发中如果要更改某些信息，修改app.settings即可。

### 静态资源的缓存
如果使用fisp的规范，每个页面是一个tpl, 经过服务器编译；而项目是个单页，一开始只向服务器请求index.tpl模板编译, 后续路由由前端来设定，每个view均是html。
这样带来了一个问题：
> view中的静态资源无法使用fisp的规则来缓存。

但是前面提到了oclazyload是可以设定缓存资源策略的。所以我们最终采用的方式是：

> 入口点加载公共静态资源，并缓存。
> 每个视图加载的静态资源由oclazyload控制；组件类的静态资源缓存（因为基本不修改），业务类的静态资源不缓存（因为经常修改）

### 代码规范

代码规范主要依据的是著名的[angular-styleguide](https://github.com/johnpapa/angular-styleguide/blob/master/a1/i18n/zh-CN.md)。建议在项目开始时组织项目成员都学习一遍，定期code review。

### 工程化
因为项目主要使用fisp, 使用svn做代码管理（度厂估计今年有可能转git），所以主要的部署逻辑是：

1. 执行 `fisp release -pd dist` 编译至dist目录
2. svn add/ci
3. 上服务器，svn up
4. 执行build.sh部署至odp

之前快速迭代没有使用厂内规范的icafe来编译，后续可能采用。

在fisp的配置fis-conf里面，除了遵循一些odp服务器的目录规范外，主要是 `合并零散资源` ，比如一个页面有多个controller, 最终会打包成controller.pack.js. 一些公共组件也做了相应的打包。



## 最后说一些坑。。

### html ng-app结合fisp报错问题

如上所述，我们只有入口点index.tpl去请求服务器编译。
fisp的规范下，原来设想应该这么写：
{% highlight ruby %}
// tag前我没加%, 意会就行了...
{html framework="common:static/script/vendor/libs/mod.js" ng-app="app"}
  {body ng-controller="appCtrl"}
{/html%}
{% endhighlight %}
结果，卧槽，返回的html中格式正确，但是似乎angular没有parse到ng-app,整个页面一片空白（当然了，ng-app都没有你还想干啥...）

最后用了个很土气的方法解决了这个问题：
{% highlight ruby %}
<script>
    document.querySelector('html').setAttribute('data-ng-app', 'app');
    document.body.setAttribute('ng-controller', 'AppCtrl');
</script>
{% endhighlight %}
### 使用包管理工具安装依赖
angular-boostrap是依赖angular主干版本的，所以最好使用包管理工具如 `bower` 之类的来安装。
否则会报这样的问题

`$position is now deprecated. Use $uibPosition instead.`

用好像还是可以用的，但是强迫症，还是升级版本吧。

### 避免头咬尾巴的行为
{% highlight ruby %}
$scope.$watch('users', function(value) {
  $scope.users = [];
});
{% endhighlight %}
有时就会写出这样的循环代码，new Error 

`10 $digest() iterations reached. Aborting!` 

等着你~

### Use angular post in "jQuery way"
angular中的post api乍一看跟jQuery区别不大，实际上post的是json, 不是parameter。
{% highlight ruby %}
$http.post("/foo/bar", {
  param1: value1,
  param2: value2
})
  .success(function(responseData){
    //do something
  }});
{% endhighlight %}
要直接这么用, 骚年，一般来说一个5xx Server Response error等着你～

看看 `dev tools` 中的 `network` tab可以解救你：

> jQuery: 
>
> - Content-Type: x-www-form-urlencoded
> - data: param1=value1&param2=value2
>
> Angular:
>
> - Content-Type: application/json 
> - data: {param1: value1,param2: value2}

一个完整的解决方案如下：
{% highlight ruby %}
var app = angular.module('app');
app.config(
    ['$httpProvider',
        function ($httpProvider) {
            $httpProvider.defaults.headers.post['Content-Type']
                = 'application/x-www-form-urlencoded;charset=utf-8';

            var param = function (obj) {
                var query = '';
                var name;
                var value;
                var fullSubName;
                var subName;
                var subValue;
                var innerObj;
                var i;

                for (name in obj) {
                    value = obj[name];

                    if (value instanceof Array) {
                        for (i = 0; i < value.length; ++i) {
                            subValue = value[i];
                            fullSubName = name + '[' + i + ']';
                            innerObj = {};
                            innerObj[fullSubName] = subValue;
                            query += param(innerObj) + '&';
                        }
                    } else if (value instanceof Object) {
                        for (subName in value) {
                            subValue = value[subName];
                            fullSubName = name + '[' + subName + ']';
                            innerObj = {};
                            innerObj[fullSubName] = subValue;
                            query += param(innerObj) + '&';
                        }
                    } else if (value !== undefined && value !== null) {
                        query += encodeURIComponent(name) + '=' + encodeURIComponent(value) + '&';
                    }
                }

                return query.length ? query.substr(0, query.length - 1) : query;
            };

            $httpProvider.defaults.transformRequest = [function (data) {
                return angular.isObject(data) && String(data) !== '[object File]' ? param(data) :
                    data;
            }];
        }
    ]);
{% endhighlight %}

额，暂时要去搬砖了，以后有想到的再来更。




