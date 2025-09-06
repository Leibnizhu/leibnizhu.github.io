---
title: "Undertow下Multipart请求大字段丢失问题的排查"
date: 2025-09-03T17:45:46+08:00
image: thumbnail.png
tags:
- Undertow
- Spring
- Springboot
- Multipart
- multipart/form-data
draft: true
---

## 问题背景

有一个POST接口，部分业务方使用 `multipart/form-data` 格式发送请求。  
最近，我们发现一个奇怪的现象：当请求中某个字段的内容（非文件字段）稍微大一点时，Controller注入的PoJo里面对应的取值就变null。  

**环境信息:**

- **Web 框架:** Spring Boot 2.7.18
- **Web 服务器:** Undertow 2.2.28.Final

经过初步排查，我们发现当一个普通表单字段的大小超过某个阈值时，它在服务端就会“神秘消失”。由于该接口并不接收文件字段，可以使用 `application/x-www-form-urlencoded` 请求，此时则不受影响。

## 初步排查与错误的尝试

1. **现象确认**: 我们稳定复现了问题。一个普通的文本字段，当其内容长度较小时（例如几 KB），后端可以正常接收。一旦超过一个特定值（后来定位到是 16KB），后端获取到的值就变成了 `null`。

2. **猜想**: 我们首先怀疑是 Spring Boot 或 Undertow 的某些配置限制了 multipart 请求的大小。于是，我们尝试调整了几个常见的配置参数：

    ```yaml
    spring:
      servlet:
        multipart:
          file-size-threshold: 2MB # 尝试调大写入临时文件的阈值

    server:
      undertow:
        max-http-post-size: 100MB # 尝试调大整个 POST 请求的大小
    ```

    然而，**这些参数全部无效**。

3. **深入代码**: 我们开始 Debug Spring 和 Undertow 的源码。Spring对PoJo对象的绑定是通过Request的`getParameterValues` 方法获取字段值的，对于 Undertow，实现类是 `io.undertow.servlet.spec.HttpServletRequestImpl`，打断点发现了一个关键线索：

    > multipart的所有字段都是用 `javax.servlet.http.Part` 记录，而在undertow中，Part存储的字段值是 `io.undertow.server.handlers.form.FormData.FormValue`；当普通表段字段大小超过阈值后，Undertow 会将该字段的内容写入一个临时文件，在对应字段值的 `FormValue` 内部被存入 `FileItem fileItem` 属性，而非字符串值的 `String value` 属性。  
    > 而 `getParameterValues`、 `getParameter`、 `getParameterMap` **直接忽略了所有 `FileItem` 类型的 FormValue** ，导致无论这个 `FileItem` 是一个真正的上传文件，还是一个因为内容太大而被临时存储为文件的普通字段，最终都无法通过 `getParameter` 系列方法获取到。

这就是字段变 `null` 的直接原因。  
相关代码节选：

```java
package io.undertow.servlet.spec;
public final class HttpServletRequestImpl implements HttpServletRequest {
    public String getParameter(String name) {
        if (this.queryParameters == null) {
            this.queryParameters = this.exchange.getQueryParameters();
        }

        Deque<String> params = (Deque)this.queryParameters.get(name);
        if (params == null) {
            FormData parsedFormData = this.parseFormData();
            if (parsedFormData != null) {
                FormData.FormValue res = parsedFormData.getFirst(name);
                return res != null && !res.isFileItem() ? res.getValue() : null;
            } else {
                return null;
            }
        } else {
            return (String)params.getFirst();
        }
    }
}

package io.undertow.server.handlers.form;
public final class FormData implements Iterable<String> {
    static class FormValueImpl implements FormValue {
        private final String value;
        private final String fileName;
        private final HttpHeaders headers;
        private final FileItem fileItem;

        public boolean isFileItem() {
            return this.fileItem != null;
        }
    }
}
```

### 根源定位

顺着 `FileItem` 的创建逻辑，一路追溯，最终定位到了问题的根源。

1. undertow使用 `io.undertow.util.MultipartParser` 解析multipart的请求体，大致上就是个有限状态机，当一个字段读取完毕后会调用 `io.undertow.util.MultipartParser.PartHandler#data` 方法进行处理
2. `io.undertow.util.MultipartParser.PartHandler#data` 方法在 `io.undertow.server.handlers.form.MultiPartParserDefinition.MultiPartUploadHandler#data` 中实现；对比 undertow 1.x版本的代码，多出了 `if (file == null && fileSizeThreshold < this.currentFileSize && (fileName != null || this.currentFileSize > fieldSizeThreshold))` 这段判定，如果字段内容大于 `fileSizeThreshold` 阈值，且为文件字段或、或为文本字段但同时超过 `fieldSizeThreshold` 阈值，则会调用 `createFile()` 方法，将字段内容写入临时文件；最后在 `endPart()` 方法中为 `FormData` 增加当前字段对应的 `FormValue`，其中 `fileItem` 为前面写入的临时文件。

```java
package io.undertow.server.handlers.form;
public class MultiPartParserDefinition implements FormParserFactory.ParserDefinition<MultiPartParserDefinition> {
    private final class MultiPartUploadHandler implements FormDataParser, MultipartParser.PartHandler {
        @Override
        public void data(final ByteBuffer buffer) throws IOException {
            this.currentFileSize += buffer.remaining();
            if (this.maxIndividualFileSize > 0 && this.currentFileSize > this.maxIndividualFileSize) {
                throw UndertowMessages.MESSAGES.maxFileSizeExceeded(this.maxIndividualFileSize);
            }
            if (file == null && fileSizeThreshold < this.currentFileSize && (fileName != null || this.currentFileSize > fieldSizeThreshold)) {
                try {
                    createdFiles.add(createFile());

                    FileOutputStream fileOutputStream = new FileOutputStream(file.toFile());
                    contentBytes.writeTo(fileOutputStream);

                    fileChannel = fileOutputStream.getChannel();
                } catch (IOException e) {
                    throw new RuntimeException(e);
                }
            }

            if (file == null) {
                while (buffer.hasRemaining()) {
                    contentBytes.write(buffer.get());
                }
            } else {
                fileChannel.write(buffer);
            }
        }
    }
}
```

3. debug发现，`fileSizeThreshold` 为0， `fieldSizeThreshold` 为16384，所以当文本字段超过16KB的时候，就会写入文件，然后 `getParameterValues` 方法就拿到null，给Pojo对应字段注入null。那么这两个值是哪里配置的呢？
4. 又经过一番debug发现，application.properties的配置其实是有效的，只是我们代码里注册了一个 `MultipartConfigElement` Bean，同时没设置 `FileSizeThreshold`，所以覆盖了配置文件的配置，变为0；只要在Bean定义的地方调用 `setFileSizeThreshold()` 即可：

```java
    @Bean
    public MultipartConfigElement multipartConfigElement() {
        MultipartConfigFactory factory = new MultipartConfigFactory();
        factory.setMaxFileSize(DataSize.of(1024L, DataUnit.MEGABYTES));
        factory.setMaxRequestSize(DataSize.of(1024L, DataUnit.MEGABYTES));
        factory.setFileSizeThreshold(DataSize.of(100L, DataUnit.MEGABYTES));
        return factory.createMultipartConfig();
    }
```

5. 至于 `fieldSizeThreshold` ，是在 `MultiPartParserDefinition` 中定义的。显然，只要在JVM启动命令增加 `-Dio.undertow.multipart.minsize=xxx`、或在启动服务时（比如在main方法中、或spring bean定义中）调用 `System.setProperty("io.undertow.multipart.minsize", "xxx")` 即可。

```java

    /**
     * Proposed default MINSIZE as 16 KB for content in memory before persisting to disk if file content exceeds
     * {@link #fileSizeThreshold} and the <i>filename</i> is not specified in the form.
     */
    private static final long MINSIZE = Long.getLong("io.undertow.multipart.minsize", 0x4000);

    private long fileSizeThreshold;

    /**
     * The threshold of form field size to persist to disk.
     * It takes effect only for the form fields which do not have <i>filename</i> specified.
     */
    private long fieldSizeThreshold = MINSIZE;

    public void setFileSizeThreshold(long fileSizeThreshold) {
        this.fileSizeThreshold = fileSizeThreshold;
    }
```
