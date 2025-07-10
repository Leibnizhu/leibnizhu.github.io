---
title: "Quarkus 快速接入 knife4j 文档"
date: 2025-07-09T18:43:20+08:00
image: thumbnail-min.png
tags:
- OpenAPI
- Quarkus
- Knife
---

[Knife4j](https://github.com/xiaoymin/knife4j) 比起 [Swagger-UI](https://swagger.io/tools/swagger-ui/) 功能更齐全，比如有搜索、全局变量、导出文档（Word、PDF、Markdown、OpenAPI 等）、多语言 UI 等等功能。Quarkus 支持 OpenAPI 及 SwaggerUI，但没有对 Kinfe4j 的支持。

最开始的想法是直接做一个 Kinfe4j 的 Quarkus Extension，后来看了下 Knife4j 已经提供了标准静态资源 jar（Servlet 3.0 标准，`META-INF/resources` 下存放了静态资源）了，即 `knife4j-openapi3-ui` 依赖，只要稍微调整下就可以直接使用，就没必要再套娃做个 Quarkus Extension。以下就快速介绍下使用方法。

1. 增加 `knife4j-openapi3-ui` 依赖：

```xml
<dependency>
    <groupId>com.github.xiaoymin</groupId>
    <artifactId>knife4j-openapi3-ui</artifactId>
    <!-- 请根据 Github 使用最新版本 -->
    <version>4.5.0</version>
</dependency>
```

2. 由于 `knife4j-openapi3-ui` 是标准静态资源 jar，Quarkus支持，所以此时打开 `http://localhost:8080/doc.htm` 已经能看到 Knife4j 页面，但是获取不到 OpenAPI 定义，所以还要做个接口
3. 定义 `/v3/api-docs/swagger-config` 接口：

```java
import io.quarkus.runtime.configuration.ConfigUtils;

import java.io.IOException;
import java.io.InputStream;
import java.io.InputStreamReader;
import java.util.List;
import java.util.Map;

import jakarta.ws.rs.GET;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.core.Response;

import lombok.extern.slf4j.Slf4j;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.dataformat.yaml.YAMLFactory;

/**
 * @author Leibniz on 2025/7/2.
 */
@Slf4j
@Path("/v3/api-docs")
public class ApiDocResource {

    @GET
    @Path("/swagger-config")
    public Response swaggerConfig() {
        // 此处可以通过 读取配置、或读取profile、或注入LaunchMode，根据实际的运行环境和配置决定是否允许展示OpenAPI定义
        // 此处示例代码是控制只有 quarkus.profile=dev或local 时才允许展示OpenAPI定义
        if (ConfigUtils.isProfileActive("local") || ConfigUtils.isProfileActive("dev")) {
            return Response.ok(Map.ofEntries(
                // 定义所有可以获取到 OpenAPI 定义的接口
                Map.entry("urls", List.of(
                    // 如果配置了Quarkus OpenAPI 扩展，那么已经有 /q/openapi 接口
                    Map.of("url", "/q/openapi", "name", "aaa API"),
                    // 还可以按需增加其他OpenAPI 定义接口，比如后面代码了另一个OpenAPI定义的接口
                    Map.of("url", "/v3/api-docs/xxx-api", "name", "xxx API")))))
                .build();
        } else {
            return Response.ok(Map.ofEntries(
                Map.entry("urls", List.of()))).build();
        }
    }

    @GET
    @Path("/xxx-api")
    public Response xxxApi() {
        ClassLoader classLoader = Thread.currentThread().getContextClassLoader();
        try (InputStream yamlStream = classLoader.getResourceAsStream("META-INF/xxx-openapi.yaml")) {
            if (yamlStream != null) {
                Object yamlObj =
                    new ObjectMapper(new YAMLFactory()).readValue(new InputStreamReader(yamlStream), Object.class);
                return Response.ok(yamlObj).build();
            }
        } catch (IOException e) {
            log.error("读取xxx OpenAPI定义文件失败: {}", e.getMessage(), e);
        }
        return Response.ok().build();
    }
}
```

4. 重启服务，此时 `http://localhost:8080/doc.htm` Knife4j 页面已经正常可用，左上角的下拉菜单对应我们在 `swaggerConfig()` 返回的 `urls` 字段定义的多个 OpenAPI 定义接口地址。
