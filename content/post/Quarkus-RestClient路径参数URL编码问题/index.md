---
title: "Quarkus RestClient路径参数URL编码问题"
date: 2023-03-10T13:17:21+08:00
image: thumbnail.jpg
tags:
- RestClient
- Quarkus
- RestEasy Client
---

问题比较简单，简单记录下。

当我们在Quarkus中使用他的 [RestClient](https://cn.quarkus.io/guides/rest-client)，恩其实主要是用RestEasy client，声明了一个带路径参数的API，如：

```java
@Path("/path")
@RegisterRestClient
public interface SomeApi {
    @GET
    @Path("/to/{id}")
    String getSomething(@PathParam("id") String id;
}
```

如果路径参数的取值带有 `"/"` ，而该服务要求对这种参数进行URL编码，以免被其他endpoint接收，这时候会发现，如果传入未URL编码的id，如 `"aaa/bbb"`，实际发出的请求为：

```bash
GET /path/to/aaa/bbb
```

此时该API可能无法正常工作（比如，有另一个 `"/path/to/{id}/bbb"` 的接口）。如果传入已URL编码的id，如 `"aaa％2Fbbb"`，实际发出的请求为：

```bash
GET /path/to/aaa%252Fbbb
```

这里是RestEasy client将 `%` 字符进行了URL编码变成 `%25`，而 `2F` 当作普通字符未处理。而该API要求传入 `GET /path/to/aaa%2Fbbb` 才能正常工作。

因为在quarkus的github中没搜到相关issue，所以直接看代码。

经过调试，发现 `RestClient` 是在 `org.jboss.resteasy.client.jaxrs.internal.proxy.processors.ProcessorFactory#createProcessor()` 方法对路径参数进行处理的创建 `PathParamProcessor` 实例的，具体过程：

1. `ProcessorFactory#createProcessor` 根据 `@PathParam` 注解创建 `PathParamProcessor` 实例，如果同时还有 `@Encoded` 注解控制是否URL编码
2. `org.jboss.resteasy.client.jaxrs.internal.proxy.processors.webtarget.PathParamProcessor#build` 方法创建 `ResteasyWebTarget` 实例
3. `PathParamProcessor#build` 方法里面会调用 `org.jboss.resteasy.specimpl.ResteasyUriBuilderImpl#resolveTemplate()` 方法创建 `UriBuilder` 实例，这里会调用 `ResteasyUriBuilderImpl#replaceParameter` 方法将实际传入参数构建uri
4. `replaceParameter` 方法先从调用的参数Map获取当前要替换的参数，如果需要对 `'/'` 编码，也就是说有 `@Encoded` 注解，则调用 `org.jboss.resteasy.util.Encode.encodePathSegmentAsIs` 方法进行编码，否则使用 `org.jboss.resteasy.util.Encode#encodePathAsIs` 进行编码。
5. `encodePathSegmentAsIs` 和 `Encode#encodePathAsIs` 方法最终都是调用 `Encode#encodeFromArray`方法，区别在于传入的编码Map，前者传入 `pathEncoding`，后者传入 `pathSegmentEncoding`；
6. 在 `Encode` 的静态代码块里，对 `pathEncoding` 和 `pathSegmentEncoding` 进行初始化，其中 `pathEncoding` 没有 '/' 的映射，而 `pathSegmentEncoding` 特意加入 '/' 的映射，具体如下：

```java
   private static final String[] pathEncoding = new String[128];
   private static final String[] pathSegmentEncoding = new String[128];

   static
   {
      for (int i = 0; i < 128; i++)
      {
         if (i >= 'a' && i <= 'z') continue;
         if (i >= 'A' && i <= 'Z') continue;
         if (i >= '0' && i <= '9') continue;
         switch ((char) i)
         {
            // …………………… 其他跳过的字符省略
            case '/':
               continue;
         }
         pathEncoding[i] = encodeString(String.valueOf((char) i));
      }
      pathEncoding[' '] = "%20";
      // ……………………
      System.arraycopy(pathEncoding, 0, pathSegmentEncoding, 0, pathEncoding.length);
      pathSegmentEncoding['/'] = "%2F";
      // ……………………
   }
```

所以达到了前面说的效果。

根据代码分析，只要给路径参数加上 `@Encoded` 注解即可：

```java
@Path("/path")
@RegisterRestClient
public interface SomeApi {
    @GET
    @Path("/to/{id}")
    String getSomething(@PathParam("id") @Encoded String id;
}
```

翻完代码，再想起来可以看看RestEasy的文档，发现其实有写的：[RestEasy4.7.7.Final UserGuide Chapter 16. @Encoded and encoding](https://docs.jboss.org/resteasy/docs/4.7.7.Final/userguide/html/_Encoded_and_encoding.html) 。
