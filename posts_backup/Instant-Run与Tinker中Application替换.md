---
title: Instant-Run与Tinker中Application替换
date: 2017-05-08 22:58:02
tags: Android
---


 - 为什么要替换application
 
因为5.0以下开始只会去加载第一个dex,如果appliaction不在第一个dex,则无法启动。如果把自己的appliaction放在第一个dex中，而自己的application没有使用multidex,则只会去加载原始加载生成的dex,也是会报错。所以办法就是去代理掉原始的application，将app启动的application设置为这个代理的application，在这个代理的application中可以确保使用multidex功能。而且这个代理的application可以轻松的放在第一个patched.dex中，无需去解析依赖关系。<!--more-->

 - 替换操作的差异

Instant run 是在attachBaseContext的时候再去反射创建原始的application，然后再反射替换掉Framework中已经初始化的application(这个时候还是代理的application),还原回去我们原始的application。接下来就是正常的流程，在app中去get application也不会有问题。

Tinker使用的是静态代理，使用代理方案代码会更复杂一些，因为要去模拟出一个applicationLike的接口，在tinker中，我们真正的application只需要实现这个接口，并不需要去继承Android中的application，因为会在代理的application中反射调用我们的application生命周期方法。也就是说tinker中只有一个真正的application。
 
  - 优缺点
  
在instant run中application的替换的透明的，两个application都是真正的Android application。好处就是透明，坏处也是使用了反射替换运行期的application，兼容性不如静态代理好，类似tinker中反射插入dex就分了好多个版本实习，这个会更复杂。tinker更加稳定。
