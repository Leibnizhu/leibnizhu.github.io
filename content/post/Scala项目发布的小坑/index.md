---
title: Scala项目发布的小坑
date: 2018-08-01T15:05:44+08:00
tags:
- Maven
- Scala
image: raindrop.jpg
---
之前写了一篇文章 [发布项目到Maven中央仓库](/p/发布项目到maven中央仓库/), 最近发布一个scala项目的时候, 用原来的配置, deploy到仓库后,提示找不到javadoc.  
折腾了一番, 结论:  
1. ***-javadoc.jar肯定是要有的, 否则Nexus校验不通过
2. 用maven的javadoc插件默认查找`src/main/java`下面的源码进行文档构建, scala项目里这个目录当然是不存在的
3. 可以通过`<sourceDictionary>`配置让javadoc插件去`src/main/scala`目录去找源码, 但是显然没有java文件给他找,也是构建不出文档的
4. 所以要用maven的scala插件进行scaladoc构建文档并打包(默认jar包文件名跟javadoc兼容), 命令是`mvn scala:doc-jar`
5. 为了和gpg签名,以及最后的deploy到远程maven仓库配合, 只要修改scala插件的配置即可

最后配置如下:
**注**: 主要增加了`doc-jar`的`goal`, 其实在profile或直接在build中配置均可  
```xml
  <build>
      <plugins>
        <!--scala编译,scala-doc等-->
          <plugin>
              <groupId>net.alchim31.maven</groupId>
              <artifactId>scala-maven-plugin</artifactId>
              <version>3.2.2</version>
              <configuration>
                  <recompileMode>incremental</recompileMode>
                  <args>
                      <arg>-target:jvm-1.8</arg>
                  </args>
                  <javacArgs>
                      <javacArg>-source</javacArg>
                      <javacArg>1.8</javacArg>
                      <javacArg>-target</javacArg>
                      <javacArg>1.8</javacArg>
                  </javacArgs>
                  <jvmArgs>
                      <jvmArg>-Xms1024m</jvmArg>
                      <jvmArg>-Xmx1024m</jvmArg>
                  </jvmArgs>
              </configuration>
              <executions>
                  <execution>
                      <id>scala-compile-first</id>
                      <phase>process-resources</phase>
                      <goals>
                          <goal>add-source</goal>
                          <goal>compile</goal>
                          <goal>doc-jar</goal> <!--scaladoc打jar包-->
                      </goals>
                  </execution>
                  <execution>
                      <id>scala-test-compile</id>
                      <phase>process-test-resources</phase>
                      <goals>
                          <goal>add-source</goal>
                          <goal>testCompile</goal>
                      </goals>
                  </execution>
              </executions>
          </plugin>
          <!-- 以前的配置 -->
      </plugins>
  </build>
```