---
title: "Quarkus服务接入Skywalking链路跟踪"
date: 2022-07-18T12:58:35+08:00
tags:
- Skywalking
- Quarkus
image: bus.jpg
---

## Quarkus对可观测性的支持情况

Quarkus在可观测性方面支持了 [OpenTracing](https://opentracing.io/) 和 [OpenTelemetry](https://opentelemetry.io/) 。如果要对使用Quarkus开发的服务接入Skywalking做链路跟踪：

- Skywalking支持OpenTelemetry，但只是在 [metric部分](https://skywalking.apache.org/docs/main/latest/en/setup/backend/opentelemetry-receiver/) 支持了。至于为什么，可以看看PR [What does opentelemetry mean?](https://github.com/apache/skywalking/issues/7374) 里面的讨(nu)论(chi)。
- Skywalking兼容Opentracing协议。但Quarkus目前对OpenTracing的支持是针对Jaeger的，而Skywalking是用GRpc通信的，不能直接对接。

所以摆在面前的路有两条：

1. 模仿 [quarkus-smallrye-opentracing](https://github.com/smallrye/smallrye-opentracing) （已deprecated），基于Provider，写一套用于拦截Quarkus请求打span等的插件。
2. 使用Skywalking Java Agent 传统姿势，但要注意各个组件的支持情况。

第一条路工作量是未知的，可行性也未知。还是选简单点的第二条路吧。

## Skywalking对Quarkus常用组件的支持情况

先简单盘点一下Skywalking Java Agent对Quarkus做web服务常用组件的支持情况：

- core: 使用了 `Vert.X-core`， 支持3.x 和 4.x。
- Http server: 使用了 `RestEasy`， 已支持 3.x。Quarkus用的是 4.x
- Http client : RestClient，最终使用的是 `Apache httpcomponent HttpClient` ，已支持。
- ORM:
  - `Hibernate`：不支持。
  - `MyBatis`: 支持 3.4.x -> 3.5.x
- Redis: 使用了 `vertx-redis`，Vertx是自己用NetSocket实现了redis的协议，目前还不支持。

可以看到，目前redis client、http Server、Hibernate还不支持。

注：详细请参考官方的 [支持列表](https://skywalking.apache.org/docs/skywalking-java/latest/en/setup/service-agent/java-agent/supported-list/) 。

## 解决方案

### RestEasy 4.x

这个我提了 [PR](https://github.com/apache/skywalking-java/pull/265)，已 [合入main分支](https://github.com/apache/skywalking-java/commit/a60a61b83de7d7daed1a4bb1d2953ce1bc3f4fa4)，等8.12.0版本发布就行，或者先下载 [main分支源码](https://github.com/apache/skywalking-java/commits/main)、自行打包用着。

**2022-09-10更新** 8.12.0 版本已经release：[Releases v8.12.0](https://github.com/apache/skywalking-java/releases/tag/v8.12.0)，可直接使用

在实操中，请移除掉 RestEasy 3.x 的插件，以免冲突。如在sidecar中执行：

```bash
rm /skywalking/agent/plugins/resteasy-server-3.x*
```

### Hibernate

ORM虽然不支持 `Hibernate`，但支持具体的jdbc驱动（如Mysql、PostgreSql等），其实是可以追踪到数据库读写的，只是少了ORM这一层而已，关系不大。

### Vert.X相关

什么？Vert.X-core不是支持了吗？

正因为支持了才有问题！

#### Skywalking如何支持Vert.X的

我们可以先看看Skywalking的 [`vertx-core-4.x-plugin`](https://github.com/apache/skywalking-java/tree/main/apm-sniffer/apm-sdk-plugin/vertx-plugins/vertx-core-4.x-plugin) 源码，跟踪原理很简单，它对 `VertxBuilder` 进行了增强，在VertxImpl实例化时注入了 `SWVertxTracer` 作为Vertx的tracer，也就是说利用了vertx的tracer机制。

这样做的原因很简单，Vertx大量用到了EventLoop线程和Worker线程的切换，如果直接像其他的Skywalking插件一样，直接增强处理的方法，那么调用 `ContextManager.getOrCreate()` 的时候，由于线程切换，是拿不到上一个span使用的 `TracerContext` （保存在ThreadLocal里）的，也就是每个操作都成了独立的TracerContext，那么就无法将span串起来，也就无法实现追踪。所以直接利用Vert.X自己提供的tracer机制，Vert.X各个组件在触发一些事件（如Http服务接受到请求，Http客户端接受到响应）时，就会调用 `VertxTracer` 的对应方法，记录事件，在这个过程中，Vertx是允许带一个Payload的，对于 `vertx-core-4.x-plugin` 插件，这个Payload就是 Skywalking的 `AbstractSpan` ，这样就可以在Vert.X内部实现绕开ThreadLocal的线程限制、进行追踪了。

#### Skywalking对Vert.X的支持、与Quarkus的关系

那么问题在哪呢？

问题就在于，这样的机制决定了，只有在Vert.X体系里的才会被追踪到，看看前面列的那些组件，RestEasy、HttpClient、JDBC Driver这些都不在Vert.X控制下的，他们产生的Span和Vert.X是无法关联起来的。

我们再看看目前 `vertx-core-4.x-plugin` 实际支持的Vert.X组件，从 [源码](https://github.com/apache/skywalking-java/blob/main/apm-sniffer/apm-sdk-plugin/vertx-plugins/vertx-core-4.x-plugin/src/main/java/org/apache/skywalking/apm/plugin/vertx4/SWVertxTracer.java) 来看，实际支持了HTTP服务、HTTP客户端，以及EventBus（包括本地的和分布式的）。

一个个看：

- HTTP服务，Quarkus在Vert.X-core之上用了 `RestEasy` 做路由，其实是无需再在Vert.X里面追踪的。
- HTTP客户端，前面说了，目前用的是 `Apache httpcomponent HttpClient` ，也是不用管的。
- Eventbus，Quarkus确实用了Vert.X-core的EventBus，这个好像没啥办法
- Redis Client，目前 `vertx-core-4.x-plugin` 不支持，Quarkus用到，有兴趣的朋友可以考虑实现一个、提个PR

问题捋完了，方案就是个取舍的问题了。

- 艰难的道路——全面支持Vert.X
  - 应用里的RestClient改成Vert.X的Http client
  - 扩展 `vertx-core-4.x-plugin` ，增加Redis请求响应的处理
  - 考虑如何支持 JDBC Driver 的追踪
  - 对于Quarkus其他组件，随缘了，或者想办法把VertxTracer和正常的基于TreadLocal的TracerContext关联起来？
- 世上无难事、只要肯放弃之路
  - 排除掉 `vertx-core-4.x-plugin` ，放弃EventBus和Redis Client的追踪

我选择放弃，在sidecar里面执行：

```bash
rm /skywalking/agent/plugins/apm-vertx-core-*
```

## 总结

1. RestEasy 4.x 将在Skywalking 8.12.0里面支持，如有需要可提前自行编译（**2022-09-10更新** 2022-09-03已release）
2. 放弃EventBus和Redis Client的追踪，下次一定.jpg
3. sidecar里面记得排除不必要的plugin：

    ```bash
    rm /skywalking/agent/plugins/apm-vertx-core-* /skywalking/agent/plugins/resteasy-server-3.x*
    ```
