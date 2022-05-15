---
title: Spring Boot快速入门（四）——日志系统
date: 2016-11-26 16:00:08
tags:
- Spring Boot
- JavaWeb
---

# 日志系统
## 简介
Spring Boot默认使用的Apache的Common Logging日志系统，但同时也提供了Java Util Logging, Log4J, Log4J2和Logback等日志系统的支持（无需额外增加依赖）。
## 日志格式
Spring Boot默认输出的日志各列含义如下：
- 日期和时间 - 精确到毫秒，且易于排序。
- 日志级别 - ERROR, WARN, INFO, DEBUG 或 TRACE。
- Process ID。
- 一个用于区分实际日志信息开头的---分隔符。
- 线程名 - 包括在方括号中（控制台输出可能会被截断）。
- 日志名 - 通常是源class的类名（缩写）。
- 日志信息。

如：
```bash
2014-03-05 10:57:51.112  INFO 45469 --- [           main] org.apache.catalina.core.StandardEngine  : Starting Servlet Engine: Apache Tomcat/7.0.52
2014-03-05 10:57:51.253  INFO 45469 --- [ost-startStop-1] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2014-03-05 10:57:51.253  INFO 45469 --- [ost-startStop-1] o.s.web.context.ContextLoader            : Root WebApplicationContext: initialization completed in 1358 ms
2014-03-05 10:57:51.698  INFO 45469 --- [ost-startStop-1] o.s.b.c.e.ServletRegistrationBean        : Mapping servlet: 'dispatcherServlet' to [/]
2014-03-05 10:57:51.702  INFO 45469 --- [ost-startStop-1] o.s.b.c.embedded.FilterRegistrationBean  : Mapping filter: 'hiddenHttpMethodFilter' to: [/*]
```
可以在resources/application.properties中使用logging.pattern.console和logging.pattern.file属性进行配置。

## 日志配置
可以在**resources/application.properties** 中进行配置，如下：
```
###日志配置###
#日志输出级别
logging.level.root=INFO
logging.level.com.turingdi.dmp=DEBUG
#检查终端是否支持ANSI，是的话就采用彩色输出
spring.output.ansi.enabled=DETECT
#设置文件，可以是绝对路径，也可以是相对路径
#logging.file=
#设置目录，会在该目录下创建spring.log文件，并写入日志内容
#logging.path=
#定义输出到控制台的样式（不支持JDK Logger）
logging.pattern.console=%d{yyyy-MM-dd HH:mm:ss} [%-5level] %logger{36}[%line]=> %msg%n
#定义输出到文件的样式（不支持JDK Logger）
#logging.pattern.file=
```

## 日志API调用
类似Log4j：
```java
	import org.apache.commons.logging.Log;
	import org.apache.commons.logging.LogFactory;

	public class TemplateController {
	private static Log log = LogFactory.getLog(TemplateController.class);
	  public String index(ModelMap map) {
	    ……
	    log.info("hahahaha");//还有warn\error\debug等方法
	    ……
	  }
	}
```

# 其他相关网站
Spring Boot相关博客：  
http://blog.didispace.com/categories/Spring-Boot/  
Thymeleaf相关博客：  
http://www.cnblogs.com/vinphy/p/4674247.html  
http://www.jianshu.com/p/ed9d47f92e37  
