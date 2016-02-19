---
layout:     post
title:      单页应用中使用gif作为背景带来的性能问题
date:       2016-02-18 15:31:19
summary:    在单页应用中，HTML全局引入大帧率GIF动画作为背景时，有可能导致性能问题。通过这次优化让CPU占用率降低了50%。
categories: jekyll pixyll
---

### 问题描述 ###

据用户反馈，目前项目的前端存在卡顿现象，具体现象为打开页面后，CPU使用率上升很快，且一直保持在很高的状态，导致页面卡顿。

项目使用了angular, 卡顿页面中使用了echarts。

后端经过排查，HTTP接口返回都在200ms以下，故为前端问题。



### 问题定位 ###

使用devTools timeline定位。 因为CPU始终保持高状态，所以在页面加载完之后再使用timeline record 2s左右。

![image](/assets/images/1.png)

发现有周期性的node paint, 且触发了全页面paint (1751 * 4915). CPU一直保持波动状态。

首先怀疑angular有没有周期性触发echart重新渲染？在$watch方法里面打个断点，很快就被排除，只init了一次。

怀疑是echarts周期性的requestAnimationFrame引起，故对echart demo页面：http://echarts.baidu.com/echarts2/doc/example/pie1.html 进行timeline分析
![image](/assets/images/2.png)



上图证明echart的确会周期性地执行(具体为一个start方法)，但是未引起任何页面重绘。

同样的，使用浏览器打开没有echarts引入的视图页（下图），页面仍有周期性重绘，故可排除echarts原因。

![image](/assets/images/3.png)


根据上图可发现，每次paint都带有对loading.gif的image decode操作。
周期性地带来rasterize paint(GPU渲染)和全页面paint。

想起项目中有一个loading.gif作为视图未加载时的显示。
查找代码发现，

{% highlight ruby %}
  html {
   background: url('/static/common/img/loading.gif');
   background-attachment: fixed;
   background-repeat: no-repeat; 
   background-position: center;
}
{% endhighlight %}
**这个loading.gif直接写在了html上面。**



### 问题解决 ###

注释掉带有loading.gif的这一行css，再次使用timeline记录页面加载后的2s时间：

![image](/assets/images/4.png)



已经没有全页面重绘（node重绘在图中标记应为绿色。）
剩下的周期性的黄色灰色主要是echarts的animationframe，下面的锯齿形状的内存使用也表明了这一点，
因为一个empty requestAnimationFrame动画会固定形成内存占用的锯齿图（**sawtooth graph**）。
让用户打开测试页面，反馈不卡了。CPU占用率在原来的一半以下。
问题解决。



### 问题原因 ###


1.  项目是angular的单页，所以在html上写css, 是影响全局的。当时的想法是在loading的时候有动画看。。。
2.  在html标签上打上loading.gif的话，因为gif需要周期性的paint，当时引入的这个gif体积较大（170*170），一个转圈动画，帧数超过100帧且**始终在不停地paint**，是引起本次性能问题的主要原因。页面看不见的原因是layer问题，实际是一直存在的。



### 结论 ###

在单页应用中，HTML全局引入大帧率GIF动画作为背景时，有可能导致性能问题。
通过这次优化让CPU占用率降低了50%。
