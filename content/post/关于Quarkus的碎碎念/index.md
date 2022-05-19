---
title: 关于Quarkus的碎碎念
date: 2022-05-15T16:03:10+08:00
#draft: true
tags:
- Quarkus
- 云原生
image: thumbnail.jpg
---

最近把一个老项目迁到了Quarkus，想谈谈一些感想和想-法。

## Why Quarkus???

答案很简单，for 云原生。在 [官网](https://quarkus.io/) 也可以看到Quarkus的宣传点也是围绕云原生进行的。  
云原生的实现大概有这些要点：

1. 应用/架构设计遵循 [12-Factor](https://12factor.net/)  
2. 服务容器化，最好框架层面就有支持
3. 启动速度快，资源占用少，便于伸缩。这里又包括镜像大小、layer设计、应用本身启动速度等因素。
4. 配合GitOps持续集成、持续交付

而Quarkus很好解决了这些问题。反观Spring，在启动速度、资源占用这块实在不尽人意，对k8s的支持也是很落后，对graalvm native- image也是起步落后。

## 用不用Native??

谈Quarkus绕不开native（native可执行文件的构建）。没办法，官方先来的，首页就是大大的native/jvm性能对比。  
native实测是真的快（不到0.1s），内存占用也确实小（启动几十MB，跑一段时间稳定大概300MB附近，不过不同压力的不同类型服务也不能说明什么）。但有什么缺点？  

1. **编译速度慢** ：一个比较简单的web服务，在我们CICD的机器上编译大概需要4min，如果是容器内multistage编译则时间更长，但直接编译对环境又有要求，所以还是最好有更强大的编译机器。
2. **编译产物体积大** ：我们编译出来的二进制文件大约130-150MB，`upx` 压缩后40MB以内；单看最后的docker镜像，如果用最小的基础镜像，大约整个镜像可以做到100MB以内，是挺小的；但是每次构建变更的layer，都是整个二进制文件的变更，也就是没次编译多了一个40MB的layer出来，体积相当可观。同样的服务，如果打jar，真正变更的只有不到1MB，pod拉镜像的速度会更快，对docker仓库的压力也更小。
3. **开发多了不少限制** ：你要注意新引入的依赖是不是支持native的；所以可能需要json序列化反序列化的类记得加上 `@RegisterForReflection` 注解；如果是外部依赖的实体类，要编写一个什么json文件声明要处理…………本文不是Quarkus教程，就不详细展开了，总之，如果要native，最好整个服务都只用quarkus生态里面的依赖，否则你不知道什么时候就编译不了了。虽然quarkus的生态已经比较完善，但总有没cover到的地方，比如hadoop生态的sdk。

没错，我们quarkus native上了生产一两个月，运行良好，充分体现了native的优越性；但是，现在要加入hadoop这些，没办法，只能放弃，要么自己实现一整套hdfs client之类的，成本很高。

## 响应式要上吗?

官网说：

> Unifies imperative and reactive
> Combine both the familiar imperative code and the reactive style when developing applications.

真的有那么Unify吗？未必哦。  
这里说的 `Unifies imperative and reactive` 是指：

- Quarkus core基于Vert.X实现，是响应式的，这也是它能unify命令式和响应式的关键原因，也是各个扩展能自行选择命令式实现和响应式实现的原因
- Quarkus在Vert.X基础上做了一个路由层来兼容两种实现，像HTTP请求这种直接通过路由层、如果是响应式代码，直接在I/O线程上执行；命令式代码则丢到Worker线程执行（就要做上下文切换了）；至于Quarkus怎么知道是什么实现，就看方法注解以及签名了，具体不展开
- 也就是说，至少在http接口这一层面，开发者可以选择用命令式还是响应式编程实现，另外定时任务、Eventbus之类也是支持的
- 但到了ORM层只能有一个选择，是的， `Hibernate` 和 `Hibernate-reactive` 显然不能共存。

如果选了 `Hibernate-reactive` 那么意味你的dao都是返回Uni或Multi，这会一直传递出去（除非await结果）。  
另外，像 `resteasy` 用了reative的话，filter这些在I/O线程运行，不能阻塞，如果里面用到redis client之类也要改Reactive RedisClient。  
总而言之，选了Reactive，多少是有 **传递性** 的，很多地方要跟着改。  
在我们的项目里，也开了个分支彻底改了Reactive，发现在一些地方支持还不是很好，比如 ：

- `@Scheduled` 方法里面开Panache的多表事务会报错
- EventBus的 `@ConsumeEvent` 注释的方法，同上
- `Hibernate-reactive` 不支持 OneToMany ManyToOne 等的懒加载，必须在HQL用 `join fetch` 声明拉外表（就不是lazy了）
- ………………

最后，基于以下考虑，放弃了reactive：

1. 老代码迁移，不可能dao全部改reactive，况且dao层还会往外传递reactive
2. quarkus reactive 本身限制，比如上面列的一些点
3. 团队接受、理解Reactive的成本
4. [virtual thread](https://openjdk.java.net/jeps/425) 已经进入 jdk19 preview了，到下个LTS，jdk21，quarkus应该已经跟进支持，到时切过去就是。

从长远来看，趋势应该是开发者只要写同步代码，不用管实际执行，通过虚拟线程压榨线程性能，大部分需求不需要在代码层面通过类似reactive这样的pattern提高性能。  
当然不是说有了Loom、reactive就没用了，只是说在压榨线程性能这一点上，可以交给loom；而我们用reactive也不只是为了性能，具体就不展开了，可以看看 [The Reactive Manifesto](https://www.reactivemanifesto.org/)。  

> **有一说一，Quarkus 的 mutiny对比RxJava之类，好用不是一点两点，接口设计相当优秀，值得学习。**

## 其他

### OpenAPI

回顾下使用OpenAPI的目的：

1. 接口标准
2. 提供给其他业务方/前端访问文档
3. 提供给其他业务方自己按需生成SDK使用

第1点是OpenAPI本身的特性，第2点Quarkus可以集成。  
Quarkus对OpenAPI的支持可以见 [官网Guides](https://quarkus.io/guides/openapi-swaggerui) ，说几个点：

1. 支持按代码生成OpenAPI定义
2. 支持按代码生成Swagger-UI界面（后者默认是dev模式才有，可以通过 `quarkus.swagger-ui.always-include=true` 开启非dev模式下可用）
3. 支持放一个自己写的OpenAPI定义文件，作为上面1、2点生成的一部分；但，Quarkus要求openapi定义文件必须放在固定路径下，如`resources/META-INF/openapi.yml`
等几个指定路径，具体参见 [Using OpenAPI And Swagger UI](https://quarkus.io/guides/openapi-swaggerui#open-document-paths) 。

第3点会带来一些不便，比如，OpenAPI spec文件在另一个项目，deploy到maven仓库，Quarkus的项目去依赖的时候，就要求它按这个目录来，如果OpenAPI spec是外部的项目，就不好控制了。

### RestClient

Quarkus的 [RestClient](https://quarkus.io/guides/rest-client-reactive) 基本功能是完备的，配合 `quarkus-smallrye-fault-tolerance` 可以做一些重试啊、Fallback啊、熔断啊之类的操作。  
但有两个点要吐槽下：

1. RestClient是可以通过 `@RegisterClientHeaders` 注册一个请求header的处理器的；但这个处理器的生命周期是restclient自己管的，给一个输入的header，你可以往里加header之类，实际上能操作的空间的很小，一般就是只能加一些固定的header。所以很多时候只能在接口方法加 `@HeaderParam` 的参数，如果又有一些固定的header处理逻辑，要么就写一层Facade，要么就在接口里做个default方法处理下。
2. 接口的异常处理定义比较弱，基本只能通过Interceptor、或者异常捕获；比如想根据响应http status code有不同处理，获取响应header这种，就比较麻烦。

### Multipart

有一点比较坑的，`@MultipartForm` 注解有俩：  

- org.jboss.resteasy.annotations.providers.multipart.MultipartForm
- org.jboss.resteasy.reactive.MultipartForm

还都是resteasy下的，注意哈，用第一个才行的，resteasy-multipart-provider 里面的。

### RedisClient

好几个方法，如 `io.quarkus.redis.client.RedisClient#del` ，没用变长参数，用的是List接收参数。 算是有点不方便吧。
