---
title: 基于Netty的Spring Boot内置Servlet容器的实现（一）
date: 2017-08-24T14:30:11+08:00
tags:
- Netty
- Spring Boot
- Servlet
image: yuyuko.png
---
# 基于Netty的Spring Boot内置Servlet容器的实现（一）
## 前言
Spring Boot有Tomcat、Jetty和undertow三种内置Servlet容器，默认使用Tomcat。  
一般来说已经够用了，但当Spring Boot用于高并发微服务的时候，可能并不够用，而且tomcat的资源占用在这种情况下说不上轻量化了。于是萌生了自己实现一个Spring Boot的Netty Servlet容器的想法。  
接下来可能会有几篇文章关于这个的，相应的代码也在开发之中，放在[Gitlab](https://gitlab.com/leibnizhu/spring-boot-starter-netty) 和 [GitHub](https://github.com/Leibnizhu/spring-boot-starter-netty)里。

## 需要完成的任务
### 实现Servlet容器
Servlet规范有以下几个核心类(接口)：
- ```ServletContext```：定义了一些可以和Servlet Container交互的方法。
- ```Registration```：实现Filter和Servlet的动态注册。
- ```ServletRequest```(```HttpServletRequest```)：对HTTP请求消息的封装。
- ```ServletResponse```(```HttpServletResponse```)：对HTTP响应消息的封装。
- ```RequestDispatcher```：将当前请求分发给另一个URL，甚至ServletContext以实现进一步的处理。
- ```Servlet```(```HttpServlet```)：所有“服务器小程序”要实现了接口，这些“服务器小程序”重写doGet、doPost、doPut、doHead、doDelete、doOption、doTrace等方法(HttpServlet)以实现响应请求的相关逻辑。
- ```Filter```(```FilterChain```)：在进入Servlet前以及出Servlet以后添加一些用户自定义的逻辑，以实现一些横切面相关的功能，如用户验证、日志打印等功能。
- ```AsyncContext```：实现异步请求处理。

我们想要实现一个Servlet容器，不管是要重头实现一个类似tomcat的容器，还是要实现一个Spring Boot内置Servlet容器，都需要实现以上接口。  
我们的任务就是利用Netty的API实现以上接口。  

### 实现Spring Boot内置Servlet容器接口
具体来说，就是要实现```EmbeddedServletContainer```接口，同时实现一个配置类，配置Spring Boot在哪些情况下启动我们的Netty Servlet容器。  

### 编写测试类/方法
需要测试以下内容:
- 基本的SpringMVC功能，如请求分发、响应是否正常
- 异步请求
- 热交换
- 缓存
- Session
- 在一个现有Spring Boot项目中测试使用
- 与内置Tomcat、Jetty的性能对比
- …………

## 参考
感谢以下项目/博文的作者：
- [SpringBoot源码分析之内置Servlet容器](https://fangjian0423.github.io/2017/05/22/springboot-embedded-servlet-container/)
- [Github DanielThomas/spring-boot-starter-netty](https://github.com/DanielThomas/spring-boot-starter-netty)


## 现在开始
首先创建一个Maven项目。
### Maven依赖
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>io.gitlab.leibnizhu</groupId>
    <artifactId>spring-boot-starter-netty</artifactId>
    <packaging>jar</packaging>
    <version>1.0-SNAPSHOT</version>
    <name>spring-boot-starter-netty</name>
    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
    </properties>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>1.5.2.RELEASE</version>
        <relativePath/>
    </parent>

    <dependencies>
      <!-- Netty及其建议的反射依赖 -->
        <dependency>
            <groupId>io.netty</groupId>
            <artifactId>netty-all</artifactId>
            <version>4.1.2.Final</version>
        </dependency>
        <dependency>
            <groupId>org.javassist</groupId>
            <artifactId>javassist</artifactId>
            <version>3.20.0-GA</version>
        </dependency>
        <!-- Spring Boot基本依赖及测试，排除内置tomcat，我们自己来实现 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
            <exclusions>
                <exclusion>
                    <groupId>org.springframework.boot</groupId>
                    <artifactId>spring-boot-starter-tomcat</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <!-- Servleten基本API -->
        <dependency>
            <groupId>javax.servlet</groupId>
            <artifactId>javax.servlet-api</artifactId>
            <version>3.1.0</version>
        </dependency>
        <dependency>
            <groupId>com.google.guava</groupId>
            <artifactId>guava</artifactId>
            <version>18.0</version>
        </dependency>
    </dependencies>

    <build>
        <!-- 省略 -->
    </build>
</project>
```

### Web应用测试类
我们直接在test包里创建一个SpringBoot应用，暂时先覆盖最基本的SpringMVC使用。  
```java
package io.gitlab.leibnizhu.sbnetty;

@Controller
@EnableAutoConfiguration(exclude = WebMvcAutoConfiguration.class)
@ComponentScan
@EnableWebMvc
public class TestWebApp {
    private static final String MESSAGE = "Hello, World!这是一条测试语句";

    public static void main(String[] args) {
      SpringApplication.run(TestWebApp.class, args);
    }

    @RequestMapping(value = "/plaintext", produces = "text/plain")
    @ResponseBody
    public String plaintext() {
        return MESSAGE;
    }

    @RequestMapping(value = "/async", produces = "text/plain")
    @ResponseBody
    public Callable<String> async() {
        return () -> MESSAGE;
    }

    @RequestMapping(value = "/json", produces = "application/json")
    @ResponseBody
    public Message json() {
        return new Message(MESSAGE);
    }

    @Bean
    public ServletRegistrationBean nullServletRegistration() {
        return new ServletRegistrationBean(new HttpServlet(){
            @Override
            protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
                resp.getOutputStream().print("Null Servlet Test");
            }
        }, "/null");
    }

    private static class Message {
        private final String message;
        public Message(String message) {
            this.message = message;
        }
        public String getMessage() {
            return message;
        }
    }
}

```

### 实现EmbeddedServletContainerFactory接口
直接启动，提示缺少```EmbeddedServletContainerFactory```Bean：
```java
org.springframework.context.ApplicationContextException: Unable to start embedded container; nested exception is org.springframework.context.ApplicationContextException: Unable to start EmbeddedWebApplicationContext due to missing EmbeddedServletContainerFactory bean.
```
Spring Boot会查找EmbeddedServletContainerFactory接口的实现类(工厂类)，调用其getEmbeddedServletContainer()方法，来获取web应用的容器。
所以我们要实现这个接口，这里不直接实现，而是通过继承AbstractEmbeddedServletContainerFactory类来实现。
其中最重要的就是
```java
public EmbeddedServletContainer getEmbeddedServletContainer(ServletContextInitializer... initializers);
```
方法，用于生成```EmbeddedServletContainer```容器实例，顺便可以做一些初始化动作，比如定义监听的端口号，初始化Context，同时调用传入参数的```ServletContextInitializer```（Servlet初始化器）们的```onStartup()```方法以设置ServletContext中的一些配置。  
目前的实现是这样的：
```java
package io.gitlab.leibnizhu.sbnetty.bootstrap;

/**
 * Spring Boot会查找EmbeddedServletContainerFactory接口的实现类(工厂类)，调用其getEmbeddedServletContainer()方法，来获取web应用的容器
 * 所以我们要实现这个接口，这里不直接实现，而是通过继承AbstractEmbeddedServletContainerFactory类来实现
 *
 * @author Leibniz on 2017-08-24.
 */
public class EmbeddedNettyFactory extends AbstractEmbeddedServletContainerFactory implements ResourceLoaderAware {
  private final static Logger LOG = LoggerFactory.getLogger(EmbeddedNettyFactory.class);
  private static final String SERVER_INFO = "Netty@SpringBoot";
  private ResourceLoader resourceLoader;

  @Override
  public EmbeddedServletContainer getEmbeddedServletContainer(ServletContextInitializer... initializers) {
      //Netty启动环境相关信息
      Package nettyPackage = Bootstrap.class.getPackage();
      String title = nettyPackage.getImplementationTitle();
      String version = nettyPackage.getImplementationVersion();
      LOG.info("Running with " + title + " " + version);
      //上下文，暂时为空
      ServletContext context = null;
      if (isRegisterDefaultServlet()) {
          LOG.warn("This container does not support a default servlet");
      }
      for (ServletContextInitializer initializer : initializers) {
          try {
              initializer.onStartup(context);
          } catch (ServletException e) {
              throw new RuntimeException(e);
          }
      }
      //从SpringBoot配置中获取端口，如果没有则随机生成
      int port = getPort() > 0 ? getPort() : new Random().nextInt(65535 - 1024) + 1024;
      InetSocketAddress address = new InetSocketAddress(port);
      LOG.info("Server initialized with port: " + port);
      return null; //初始化容器并返回
  }

  @Override
  public void setResourceLoader(ResourceLoader resourceLoader) {
      this.resourceLoader = resourceLoader;
  }
}
```
现在```ServletContext```和```EmbeddedServletContainer```接口还没实现，先用null代替。  

### 配置Spring Boot启动自定义Servlet容器
就这样直接启动测试Web应用是不行的，因为这个```EmbeddedNettyFactory```并没有被Spring加载。  
想被Spring加载很简单，类加```@Component```之类的注解就行，但这样集成在任何环境中都会加载，可能引起端口冲突。  
所以我们还要写一个配置类，配置Spring什么时候去加载```EmbeddedNettyFactory```，具体如下，注释里写得比较清楚了：  
```java
package io.gitlab.leibnizhu.sbnetty.bootstrap;

/**
 * 配置加载内置Netty容器的工厂类Bean。
 * 最早是直接将EmbeddedNettyFactory加@Component注解，这样集成在任何环境中都会加载，可能引起端口冲突。
 * 所以通过这个配置类，配置在当前上下文缺少EmbeddedServletContainerFactory接口实现类时（即缺少内置Servlet容器），加载EmbeddedNettyFactory
 * 这样SpringBoot项目在引入这个maven依赖，并且排除了内置tomcat依赖、且没引入其他servlet容器（如jetty）时，就可以通过工厂类加载并启动netty容器了。
 *
 * @author Leibniz 2017-08-24
 */
@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE)
@Configuration
@ConditionalOnWebApplication // 在Web环境下才会起作用
public class EmbeddedNettyAutoConfiguration {
    @Configuration
    @ConditionalOnClass({Bootstrap.class}) // Netty的Bootstrap类必须在classloader中存在，才能启动Netty容器
    @ConditionalOnMissingBean(value = EmbeddedServletContainerFactory.class, search = SearchStrategy.CURRENT) //当前Spring容器中不存在EmbeddedServletContainerFactory接口的实例
    public static class EmbeddedNetty {
        //上述条件注解成立的话就会构造EmbeddedNettyFactory这个EmbeddedServletContainerFactory
        @Bean
        public EmbeddedNettyFactory embeddedNettyFactory() {
            return new EmbeddedNettyFactory();
        }
    }
}
```

### 再次启动
这样子是启动不了的，但启动报错信息已经改了，变成：
```java
2017-08-24 14:20:25.660 ERROR 16708 --- [           main] o.s.boot.SpringApplication               : Application startup failed

org.springframework.context.ApplicationContextException: Unable to start embedded container; nested exception is java.lang.NullPointerException
	at org.springframework.boot.context.embedded.EmbeddedWebApplicationContext.onRefresh(EmbeddedWebApplicationContext.java:137) ~[spring-boot-1.5.2.RELEASE.jar:1.5.2.RELEASE]
	at org.springframework.context.support.AbstractApplicationContext.refresh(AbstractApplicationContext.java:536) ~[spring-context-4.3.7.RELEASE.jar:4.3.7.RELEASE]
	at org.springframework.boot.context.embedded.EmbeddedWebApplicationContext.refresh(EmbeddedWebApplicationContext.java:122) ~[spring-boot-1.5.2.RELEASE.jar:1.5.2.RELEASE]
	at org.springframework.boot.SpringApplication.refresh(SpringApplication.java:737) [spring-boot-1.5.2.RELEASE.jar:1.5.2.RELEASE]
	at org.springframework.boot.SpringApplication.refreshContext(SpringApplication.java:370) [spring-boot-1.5.2.RELEASE.jar:1.5.2.RELEASE]
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:314) [spring-boot-1.5.2.RELEASE.jar:1.5.2.RELEASE]
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:1162) [spring-boot-1.5.2.RELEASE.jar:1.5.2.RELEASE]
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:1151) [spring-boot-1.5.2.RELEASE.jar:1.5.2.RELEASE]
	at io.gitlab.leibnizhu.sbnetty.TestWebApp.main(TestWebApp.java:102) [test-classes/:na]
Caused by: java.lang.NullPointerException: null

```
因为SpringBoot在启动的时候，```SpringApplication```会调用```refresh(context)```方法进行初始化动作，而我们的context传入了null，当然报空指针异常了。  
我们将在下一篇文章再讨论怎么实现这个。
