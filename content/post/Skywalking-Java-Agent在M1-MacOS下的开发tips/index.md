---
title: "Skywalking Java Agent在M1芯片MacOS下的开发tips"
date: 2022-06-30T21:10:28+08:00
tags:
- Skywalking
- JavaAgent
- M1
- MacOS
image: platform.jpg
---

Skywalking Java Agent的开发与测试在官网文档已经有详尽的介绍，包括（但不限于）：

- 编译： [Compiling Guidance](https://skywalking.apache.org/docs/skywalking-java/latest/en/contribution/compiling/) ，及 [How to build a project](https://github.com/apache/skywalking/blob/master/docs/en/guides/How-to-build.md)
- 编写testcase并运行： [Plugin automatic test framework](https://skywalking.apache.org/docs/skywalking-java/latest/en/setup/service-agent/java-agent/plugin-test/)
- 发布： [Apache SkyWalking release guide](https://github.com/apache/skywalking/blob/master/docs/en/guides/How-to-release.md)，及 [Java Agent Release Guidance](https://skywalking.apache.org/docs/skywalking-java/latest/en/contribution/release-java-agent/) 

但在M1芯片的MacOS中，还是有一些需要调整的地方，在此简单记录一下。

1. 编译过程需要clone其他仓库等一些网络操作，请先保障网络畅通；若直连网速不佳，请先备好梯子并配置到终端，如：

    ```bash
    export ALL_PROXY=socks5://127.0.0.1:1080
    git config --global http.proxy socks5://127.0.0.1:1080
    git config --global https.proxy socks5://127.0.0.1:1080
    ```

2. clone完 [skywalking-java](https://github.com/apache/skywalking-java) 后，先初始化git子模块，[官方文档](https://github.com/apache/skywalking/blob/master/docs/en/guides/How-to-build.md) 也有说到：

    ```bash
    git submodule init                                                                                                     
    git submodule update
    ```

3. 编译过程会下载 `protobuf`， 而目前没有M1对应的版本，请在 `test/plugin/agent-test-tools/bin/fetch-code.sh` 增加 `-Dos.detected.classifier=osx-x86_64` ，具体位置大约在32行：

    ```bash
    "$ROOT_DIR"/../../../../mvnw -B package -DskipTests -Dos.detected.classifier=osx-x86_64
    ```

4. 启动Docker
5. 测试用到 `docker-maven-plugin` 起docker容器，这插件可能还没支持 macOS aarch64，无法使用 unix socket，导致在 M1 使用 `docker-maven-plugin` 构建镜像报错，如：

    ```log
    could not get native definition for type `POINTER`, original error message follows: java.lang.UnsatisfiedLinkError: Unable to execute or load jffi binary stub from `/var/folders/c0/xxxxxx/T/`. Set `TMPDIR` or Java property `java.io.tmpdir` to a read/write path that is not mounted "noexec".
    [ERROR] /xxxxx/skywalking-java/xxxx.dylib: dlopen(/xxxxx/skywalking-java/xxxxx.dylib, 0x0001): tried: '/xxxxx/skywalking-java/xxxxx.dylib' (fat file, but missing compatible architecture (have 'i386,x86_64', need 'arm64e'))
    ```

    需要通过 `socat` 来桥接，具体操作：

    ```bash
    # 安装socat
    brew install socat 
    # 将 unix socket 代理到 tcp 端口
    nohup socat TCP-LISTEN:2375,range=127.0.0.1/32,reuseaddr,fork UNIX-CLIENT:/var/run/docker.sock &> /dev/null & 
    # 设置环境变量为socat桥接的tcp端口
    export DOCKER_HOST=tcp://127.0.0.1:2375 
    ```

6. 执行测试拉取docker镜像时，tomcat没有m1的版本，参考 [`docker-maven-plugin` 的配置文档](https://dmp.fabric8.io/#build-configuration) 在 `test/plugin/containers/jvm-container/pom.xml` 加入 `createImageOptions` 参数（请注意，`docker-maven-plugin` 的0.39.0版本才有这个参数）：

    ```xml
    <build>
        <plugins>
            <plugin>
                <groupId>io.fabric8</groupId>
                <artifactId>docker-maven-plugin</artifactId>
                <!-- 关键点1: 插件版本 -->
                <version>0.40.0</version>
                <configuration>
                    <images>
                        <image>
                            <name>skywalking/agent-test-jvm:${container_image_version}</name>
                            <build>
                                <!-- 关键点2: 通过 createImageOptions 配置docker镜像选用的平台 -->
                                <createImageOptions>
                                    <platform>linux/x86_64</platform>
                                </createImageOptions>
                                <from>${base_image_java}</from>
                                <workdir>/usr/local/skywalking/scenario</workdir>
                                <assembly>
                                <!-- 以下省略 -->
    ```

到此为止，就可以使用：

```bash
bash ./test/plugin/run.sh -f ${scenario_name}`
```

来执行测试了。
