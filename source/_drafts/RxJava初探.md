---
title: RxJava初探
tags:
  - Android
  - RxJava
categories:
  - Android
date: 2017-12-24 19:39:45
description:
---


# 介绍

[http://reactivex.io](http://reactivex.io)

[https://github.com/ReactiveX/RxJava](https://github.com/ReactiveX/RxJava)

[https://github.com/ReactiveX/RxAndroid](https://github.com/ReactiveX/RxAndroid)

3个最主要的官方网站就是上面3个，可以看到3个链接中同时出现了**ReactiveX**这个字样，用官方的话来说就是`ReactiveX is a library for composing asynchronous and event-based programs by using observable sequences.`它是一种新的变成思想，将异步任务，通过事件序列发送，接受端用事件流的方式处理，所以也叫流式编程。

ReactiveX不仅支持Java还支持其他一些语言，RxJava只是其中常用的一种，RxJava现在大体上分为两个版本1.xx和2.xx，两个版本的差别还是比较大的，相对独立各成一套，所以直接学2.xx也是没有压力的。RxAndroid又是RxJava下的一个分支，专门为Android开发提供的一套流式API，它依赖与RxJava，这是三者的关系。

<!-- more -->

Rx的最大好处就是避免了异步代码中复杂的逻辑，回调等，可以很顺畅地编写异步代码，并且提供了一系列操作符和轻松的线程切换功能，让代码灰常优雅。下面几个是我学习Rx参考的一些文章，讲得很棒，也推荐给大家。最好还是多看官方文档，优秀blog作为借鉴。

[给 Android 开发者的 RxJava 详解](http://gank.io/post/560e15be2dca930e00da1083)

[大头鬼Bruce blog](http://blog.csdn.net/lzyzsd)

[这可能是最好的RxJava 2.x 教程](https://www.jianshu.com/p/0cd258eecf60)



# RxJava使用介绍



# RxAndroid使用介绍