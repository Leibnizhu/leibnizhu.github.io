---
title: 最近对Java服务框架的思考
date: 2017-10-11 14:19:12
tags:
- Netty
- Spring Boot
- Play
- Vert.X
image: mokou_kaguya.png
---
简单说几句，关于最近对Java服务框架的思考。  
最早我是用`springMVC + Spring`的，因为太臃肿，配置麻烦，很快切换到`SpringBoot`。  
用上`SpringBoot`后，觉得内置`Tomcat/Jetty`性能可能不够好，于是自己写了个[基于Netty的内置Servlet容器](https://github.com/Leibnizhu/spring-boot-starter-netty)，然而简单测试后发现性能与内置的`Tomcat/Jetty`相差不大（也有可能是因为测试用例太简单了，没有把`Netty NIO`在业务阻塞线程时的优势体现出来）。  
期间还考虑过直接用`Netty`原生API来写一个分发请求的简单框架，看了一些别人类似功能的项目，后来不了了之。  
结合这两点，盯上了`Play Framework`，这个在许久前就有关注过，但没深入了解，看了官方文档之后，发现这就是我想要的！开发起来很方便嘛，但是`Session`的实现有点………………建议用`scala`写，我个人是没问题，但不好带人一起写。  
后来在Telegram某群组里被安利了`Vert.X`，看了官方文档，还有详细的官方Demo，以及各种安利文章，发现这玩意真好用诶，跟`Node.Js`有点像诶，也不用跟用`Netty`原生API一样战战兢兢了，配套解决方案也不少，逼格也有，多语言支持（虽然对我而言用处不大），决定就是你了！  
所以最终结论就是：`Vert.X`大法好，退`Spring`保平安～
  
P.S. 在[Github](https://github.com/Leibnizhu/VertxLearn)写了一些简单的`Vert.X`学习例子，另外准备用`Vert.X`写一个微信/支付宝的微服务(2018-08-02更新:2017年年底已经写了,忘了更新这篇文章, 请参阅[基于Vert.X的高性能微信支付宝公众号通用服务](/p/基于Vert.X的高性能微信支付宝公众号通用服务/))。
