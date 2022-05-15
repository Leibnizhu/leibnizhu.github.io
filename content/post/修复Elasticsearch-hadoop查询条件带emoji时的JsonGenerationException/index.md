---
title: 修复Elasticsearch-hadoop查询条件带emoji时的JsonGenerationException
date: 2021-06-20 13:17:54
tags:
- Elasticsearch
- ES
- Spark
- 大数据
- emoji 
categories:
    - Spark
image: bg1.png
---
## 故障背景
通过 [ES-Hadoop](https://www.elastic.co/guide/en/elasticsearch/hadoop/current/spark.html) （亦可参考 [Github](https://github.com/elastic/elasticsearch-hadoop) ） 查询ES时，若查询条件包含emoji，会在json序列化的时候抛出异常（在最新的 `7.13.2` 版本仍存在）：

```shell
Caused by: org.codehaus.jackson.JsonGenerationException: Incomplete surrogate pair: first char 0xde97, second 0x22
	at org.codehaus.jackson.impl.JsonGeneratorBase._reportError(JsonGeneratorBase.java:480)
	at org.codehaus.jackson.impl.Utf8Generator._decodeSurrogate(Utf8Generator.java:1708)
	at org.codehaus.jackson.impl.Utf8Generator._outputSurrogates(Utf8Generator.java:1663)
	at org.codehaus.jackson.impl.Utf8Generator._outputRawMultiByteChar(Utf8Generator.java:1649)
	at org.codehaus.jackson.impl.Utf8Generator.writeRaw(Utf8Generator.java:757)
	at org.codehaus.jackson.impl.Utf8Generator.writeRaw(Utf8Generator.java:697)
	at org.elasticsearch.hadoop.serialization.json.JacksonJsonGenerator.writeRaw(JacksonJsonGenerator.java:252)
	... 21 more
```

## 故障根源分析
根据报错的调用栈，直接原因出在 `org.codehaus.jackson.impl.Utf8Generator#_decodeSurrogate` 方法中，其源码：
```java
protected final int _decodeSurrogate(int surr1, int surr2) throws IOException {
    if (surr2 < 56320 || surr2 > 57343) {
        String msg = "Incomplete surrogate pair: first char 0x" + Integer.toHexString(surr1) + ", second 0x" + Integer.toHexString(surr2);
        this._reportError(msg);
    }

    int c = 65536 + (surr1 - '\ud800' << 10) + (surr2 - '\udc00');
    return c;
}
```
是判断第二个字符在指定范围（`[56320, 57343]` 区间）外就报这个错误。

同时注意，这里用的是 `org.codehaus` 的 `jackson-core-asl` 依赖包，[众所周知](https://github.com/FasterXML/jackson-docs/wiki/Presentation-Jackson-2.0) ，`Jackson` 自2.x版本开始就迁移到 `com.fasterxml` 下了，这个 `org.codehaus` 的 `jackson-core-asl` 是1.x版本的（Es-Spark通过打包时改第三方包名的方法将Jackson 打进其jar包中，具体参见 [build.gradle文件的relocate操作](https://github.com/elastic/elasticsearch-hadoop/blob/master/thirdparty/build.gradle) ，实际版本为 [1.8.8](https://github.com/elastic/elasticsearch-hadoop/blob/master/gradle.properties) ）。

针对这个报错，可以查到是一个已经 [在2.3.0修复的bug](https://github.com/FasterXML/jackson-core/issues/115) ，是旧版本不完全支持UTF-8的 `surrogate pair` （这里又是一个兼容性的大坑，可以参见 [维基百科](https://en.wikipedia.org/wiki/UTF-16#Code_points_from_U+010000_to_U+10FFFF) 的介绍）导致的。

综上所述，Es-Spark 在执行查询的时候，`org.elasticsearch.hadoop.rest.RestClient#searchRequest` 方法构建了 `org.elasticsearch.hadoop.serialization.json.JacksonJsonGenerator` 实例，并将 `QueryBuilder` 写入到 `JacksonJsonGenerator` 中序列化成查询json，在这一步中由于使用了 Jackson 1.x 版本对UTF8的emoji支持不好，导致抛出 `JsonGenerationException` 异常、中断查询。

```java
//RestClient 某查询方法
xxx queryXxx(......) {
    //......
    Response response = execute(POST, uri.toString(), searchRequest(query));
    //......
}

static BytesArray searchRequest(QueryBuilder query) {
    FastByteArrayOutputStream out = new FastByteArrayOutputStream(256);
    JacksonJsonGenerator generator = new JacksonJsonGenerator(out); //注意此处
    try {
        generator.writeBeginObject();
        generator.writeFieldName("query");
        generator.writeBeginObject();
        query.toJson(generator);
        generator.writeEndObject();
        generator.writeEndObject();
    } finally {
        generator.close();
    }
    return out.bytes();
}
```
 
## 解决方案
故障分析到这里，似乎只要升级 `jackson-core` 版本就可以了……  
然而上面提到，在 `jackson-core 2.3.0` 才修复了这个问题，而 Es-Spark 使用的是内置的1.x 版本，前面也有提到 `jackson-core` 自2.x开始迁移到 `com.fasterxml` 。具体到代码，Es-spark 的 `JacksonJsonGenerator` 是这样使用 `jackson` 的：

```java
package org.elasticsearch.hadoop.serialization.json;

import org.codehaus.jackson.JsonEncoding;
import org.codehaus.jackson.JsonFactory;
import org.codehaus.jackson.JsonGenerator;
import org.elasticsearch.hadoop.serialization.Generator;

public class JacksonJsonGenerator implements Generator {
    //省略部分字段
    private static final JsonFactory JSON_FACTORY;
    private final JsonGenerator generator;
    private final OutputStream out;

    static {
        //省略部分代码
        JSON_FACTORY = new JsonFactory();
        JSON_FACTORY.configure(JsonGenerator.Feature.QUOTE_FIELD_NAMES, true);
    }

    //RestClient 就是调用这个构造方法
    public JacksonJsonGenerator(OutputStream out) {
        try {
            this.out = out;
            // use dedicated method to lower Jackson requirement
            this.generator = JSON_FACTORY.createJsonGenerator(out, JsonEncoding.UTF8);
        } catch (IOException ex) {
            throw new EsHadoopSerializationException(ex);
        }
    }
}
```

也就是说，直接升级依赖版本是不行的，maven的GAV都变了，类名也变了，必须改代码。

### 同名类的Patch
可以看到，虽说 `Jackson` 迁移了包名，但如果是通过创建同名类的方式Patch，其实也很简单，只要把 `JSON_FACTORY` 和 `generator` 这个两个属性替换为 `Jackson 2.3+` 版本的类、并微调静态代码块和构造方法里面的代码即可：
```java
package org.elasticsearch.hadoop.serialization.json;

import com.fasterxml.jackson.core.JsonEncoding;
import com.fasterxml.jackson.core.JsonGenerator;
import com.fasterxml.jackson.core.JsonFactory;
import org.elasticsearch.hadoop.serialization.Generator;

public class JacksonJsonGenerator implements Generator {
    //省略部分字段
    private static final JsonFactory JSON_FACTORY;
    private final JsonGenerator generator;
    private final OutputStream out;

    static {
        //省略部分代码
        JSON_FACTORY = new JsonFactory();
        JSON_FACTORY.configure(JsonGenerator.Feature.QUOTE_FIELD_NAMES, true);
    }

    public JacksonJsonGenerator(OutputStream out) {
        try {
            this.out = out;
            // use dedicated method to lower Jackson requirement
            this.generator = JSON_FACTORY.createJsonGenerator(out, JsonEncoding.UTF8);
        } catch (IOException ex) {
            throw new EsHadoopSerializationException(ex);
        }
    }
}
```

### Javassist Patch
与上一篇博客一样，为了可维护性，最后还是选择使用 `Javassist` 进行Patch。但受限于 `Javassist` 的机制，这个Patch起来有点麻烦。  
首先，阅读 `JacksonJsonGenerator` 源码，它其实相当于是在 Es-spark 的 `Generator` 接口与 `jackson 1.8.8` 的 `JsonGenerator` 之间做了Adaptor；因此可以考虑写一个 `Generator` 接口与 `jackson 2.3+` 之间的Adaptor给原调用者使用。  
但阅读 Es-spark 的其他代码可以发现，虽然它定义了 `Generator` 接口，但调用时都是直接面向 `JacksonJsonGenerator` 实现类，如上面给出过的 `RestClient#searchRequest` 的代码：
```java
//RestClient 某查询方法
static BytesArray searchRequest(QueryBuilder query) {
    //......
    JacksonJsonGenerator generator = new JacksonJsonGenerator(out); //注意此处
    //......
}
```
因此首先排除了通过修改 `JacksonJsonGenerator` 调用者来Patch的方向，还是需要从 `JacksonJsonGenerator` 内部入手。

如果用`javassist`修改`JacksonJsonGenerator`，参考上一小节的内容，其实只要改俩成员变量的类型，再改改静态代码块即可。但真的如此吗？并不。写同名类能这样做到是因为会整个类重新编译，`JacksonJsonGenerator`中大量delegate的方法在编译时是用 `Jackson 2.3+` 的类进行连接的；然而`javassist`修改成员变量的时候真的只是修改了成员变量本身，如果只改了这里，对应的delegate方法在运行时会找不到成员变量。

如果是在静态代码块和构造方法新增 `Jackson 2.3+` 对应的类，并给 `writeRaw` 方法增加try-catch，在catch中使用 `Jackson 2.3+` 对应的类进行json序列化呢？也不行。因为序列化是输出到`OutputStream`（构造方法传入的那个），是有状态的，同时给`jackson 1.8.8`和`jackson 2.3+`持有并写入，恐怕会大乱（实测的确如此，不确定是不是没处理好`flush`，但至少这个方案太危险）。

还有一个方案是替换 `JacksonJsonGenerator` 的 `generator` 成员变量，可以做一个 `org.codehaus.jackson.JsonGenerator` 与  `Jackson 2.3+` 的 `com.fasterxml.jackson.core.JsonGenerator` 之间的Adaptor来替换之。

首先是Adaptor：
```java
package xxx.yyy.zzz;

import com.fasterxml.jackson.core.JsonEncoding;
import com.fasterxml.jackson.core.JsonFactory;
import com.fasterxml.jackson.core.JsonGenerator;
import org.apache.commons.logging.LogFactory;
import org.codehaus.jackson.*;
import org.elasticsearch.hadoop.serialization.EsHadoopSerializationException;
import org.elasticsearch.hadoop.util.StringUtils;

import java.io.IOException;
import java.io.OutputStream;
import java.math.BigDecimal;
import java.math.BigInteger;
import java.util.Deque;
import java.util.LinkedList;

public class JacksonJsonGeneratorAdaptor extends org.codehaus.jackson.JsonGenerator {

    private static final boolean HAS_UTF_8;
    private static final JsonFactory JSON_FACTORY;
    private final JsonGenerator generator;
    private final OutputStream out;
    private final Deque<String> currentPath = new LinkedList<>();
    private String currentPathCached;
    private String currentName;
    protected ObjectCodec _objectCodec;

    static {
        boolean hasMethod = false;
        try {
            JsonGenerator.class.getMethod("writeUTF8String", byte[].class, int.class, int.class);
            hasMethod = true;
        } catch (NoSuchMethodException ignored) {
        }
        HAS_UTF_8 = hasMethod;
        if (!HAS_UTF_8) {
            LogFactory.getLog(JacksonJsonGeneratorAdaptor.class).warn("Old Jackson version (pre-1.7) detected; consider upgrading to improve performance");
        }

        JSON_FACTORY = new JsonFactory();
        JSON_FACTORY.configure(JsonGenerator.Feature.QUOTE_FIELD_NAMES, true);
    }

    public JacksonJsonGeneratorAdaptor(OutputStream out) {
        try {
            this.out = out;
            // use dedicated method to lower Jackson requirement
            this.generator = JSON_FACTORY.createGenerator(out, JsonEncoding.UTF8);
        } catch (IOException ex) {
            throw new EsHadoopSerializationException(ex);
        }
    }
    
    //省略大量delegate方法，只列出不是简单delegate的

    @Override
    public void writeStartObject() throws IOException {
        generator.writeStartObject();
        if (currentName != null) {
            currentPath.addLast(currentName);
            currentName = null;
            currentPathCached = null;
        }
    }

    @Override
    public void writeEndObject() throws IOException {
        generator.writeEndObject();
        currentName = currentPath.pollLast();
        currentPathCached = null;
    }

    @Override
    public void writeFieldName(String name) throws IOException {
        generator.writeFieldName(name);
        currentName = name;
    }

    @Override
    public void writeUTF8String(byte[] text, int offset, int len) throws IOException {
        if (HAS_UTF_8) {
            generator.writeUTF8String(text, offset, len);
        } else {
            generator.writeString(new String(text, offset, len, StringUtils.UTF_8));
        }
    }

    @Override
    public void writeBinary(Base64Variant var1, byte[] data, int offset, int len) throws IOException {
        generator.writeBinary(data, offset, len);
    }

    @Override
    public void writeBinary(byte[] data) throws IOException {
        writeBinary(Base64Variants.getDefaultVariant(), data, 0, data.length);
    }

    @Override
    public void copyCurrentEvent(JsonParser jp) throws IOException {
        JsonToken t = jp.getCurrentToken();
        if (t == null) {
            throw new JsonGenerationException("No current event to copy");
        }

        switch(t) {
            case START_OBJECT:
                this.writeStartObject();
                break;
            case END_OBJECT:
                this.writeEndObject();
                break;
            case START_ARRAY:
                this.writeStartArray();
                break;
            case END_ARRAY:
                this.writeEndArray();
                break;
            case FIELD_NAME:
                this.writeFieldName(jp.getCurrentName());
                break;
            case VALUE_STRING:
                if (jp.hasTextCharacters()) {
                    this.writeString(jp.getTextCharacters(), jp.getTextOffset(), jp.getTextLength());
                } else {
                    this.writeString(jp.getText());
                }
                break;
            case VALUE_NUMBER_INT:
                switch(jp.getNumberType()) {
                    case INT:
                        this.writeNumber(jp.getIntValue());
                        return;
                    case BIG_INTEGER:
                        this.writeNumber(jp.getBigIntegerValue());
                        return;
                    default:
                        this.writeNumber(jp.getLongValue());
                        return;
                }
            case VALUE_NUMBER_FLOAT:
                switch(jp.getNumberType()) {
                    case BIG_DECIMAL:
                        this.writeNumber(jp.getDecimalValue());
                        return;
                    case FLOAT:
                        this.writeNumber(jp.getFloatValue());
                        return;
                    default:
                        this.writeNumber(jp.getDoubleValue());
                        return;
                }
            case VALUE_TRUE:
                this.writeBoolean(true);
                break;
            case VALUE_FALSE:
                this.writeBoolean(false);
                break;
            case VALUE_NULL:
                this.writeNull();
                break;
            case VALUE_EMBEDDED_OBJECT:
                this.writeObject(jp.getEmbeddedObject());
                break;
            default:
                throw new RuntimeException("Internal error: should never end up through this code path");
        }

    }

    @Override
    public void copyCurrentStructure(JsonParser jp) throws IOException {
        JsonToken t = jp.getCurrentToken();
        if (t == JsonToken.FIELD_NAME) {
            this.writeFieldName(jp.getCurrentName());
            t = jp.nextToken();
        }

        switch(t) {
            case START_OBJECT:
                this.writeStartObject();

                while(jp.nextToken() != JsonToken.END_OBJECT) {
                    this.copyCurrentStructure(jp);
                }

                this.writeEndObject();
                break;
            case START_ARRAY:
                this.writeStartArray();

                while(jp.nextToken() != JsonToken.END_ARRAY) {
                    this.copyCurrentStructure(jp);
                }

                this.writeEndArray();
                break;
            default:
                this.copyCurrentEvent(jp);
        }
    }

    @Override
    public void close() {
        try {
            generator.close();
        } catch (IOException ex) {
            throw new EsHadoopSerializationException(ex);
        }
    }

    @Override
    public Object getOutputTarget() {
        //return generator.getOutputTarget();
        return out;
    }

    @Override
    public org.codehaus.jackson.JsonGenerator enable(Feature feature) {
        switch (feature) {
            case AUTO_CLOSE_TARGET:
                generator.enable(JsonGenerator.Feature.AUTO_CLOSE_TARGET);
                break;
            case AUTO_CLOSE_JSON_CONTENT:
                generator.enable(JsonGenerator.Feature.AUTO_CLOSE_JSON_CONTENT);
                break;
            case QUOTE_FIELD_NAMES:
                generator.enable(JsonGenerator.Feature.QUOTE_FIELD_NAMES);
                break;
            case QUOTE_NON_NUMERIC_NUMBERS:
                generator.enable(JsonGenerator.Feature.QUOTE_NON_NUMERIC_NUMBERS);
                break;
            case WRITE_NUMBERS_AS_STRINGS:
                generator.enable(JsonGenerator.Feature.WRITE_NUMBERS_AS_STRINGS);
                break;
            case FLUSH_PASSED_TO_STREAM:
                generator.enable(JsonGenerator.Feature.FLUSH_PASSED_TO_STREAM);
                break;
            case ESCAPE_NON_ASCII:
                generator.enable(JsonGenerator.Feature.ESCAPE_NON_ASCII);
                break;
        }
        return this;
    }

    @Override
    public org.codehaus.jackson.JsonGenerator disable(Feature feature) {
        switch (feature) {
            case AUTO_CLOSE_TARGET:
                generator.disable(JsonGenerator.Feature.AUTO_CLOSE_TARGET);
                break;
            case AUTO_CLOSE_JSON_CONTENT:
                generator.disable(JsonGenerator.Feature.AUTO_CLOSE_JSON_CONTENT);
                break;
            case QUOTE_FIELD_NAMES:
                generator.disable(JsonGenerator.Feature.QUOTE_FIELD_NAMES);
                break;
            case QUOTE_NON_NUMERIC_NUMBERS:
                generator.disable(JsonGenerator.Feature.QUOTE_NON_NUMERIC_NUMBERS);
                break;
            case WRITE_NUMBERS_AS_STRINGS:
                generator.disable(JsonGenerator.Feature.WRITE_NUMBERS_AS_STRINGS);
                break;
            case FLUSH_PASSED_TO_STREAM:
                generator.disable(JsonGenerator.Feature.FLUSH_PASSED_TO_STREAM);
                break;
            case ESCAPE_NON_ASCII:
                generator.disable(JsonGenerator.Feature.ESCAPE_NON_ASCII);
                break;
        }
        return this;
    }

    @Override
    public boolean isEnabled(Feature feature) {
        switch (feature) {
            case AUTO_CLOSE_TARGET:
                return generator.isEnabled(JsonGenerator.Feature.AUTO_CLOSE_TARGET);
            case AUTO_CLOSE_JSON_CONTENT:
                return generator.isEnabled(JsonGenerator.Feature.AUTO_CLOSE_JSON_CONTENT);
            case QUOTE_FIELD_NAMES:
                return generator.isEnabled(JsonGenerator.Feature.QUOTE_FIELD_NAMES);
            case QUOTE_NON_NUMERIC_NUMBERS:
                return generator.isEnabled(JsonGenerator.Feature.QUOTE_NON_NUMERIC_NUMBERS);
            case WRITE_NUMBERS_AS_STRINGS:
                return generator.isEnabled(JsonGenerator.Feature.WRITE_NUMBERS_AS_STRINGS);
            case FLUSH_PASSED_TO_STREAM:
                return generator.isEnabled(JsonGenerator.Feature.FLUSH_PASSED_TO_STREAM);
            case ESCAPE_NON_ASCII:
                return generator.isEnabled(JsonGenerator.Feature.ESCAPE_NON_ASCII);
        }
        return false;
    }

    @Override
    public org.codehaus.jackson.JsonGenerator setCodec(ObjectCodec objectCodec) {
        this._objectCodec = objectCodec;
        return this;
    }

    @Override
    public ObjectCodec getCodec() {
        return this._objectCodec;
    }
}
```

然后通过`javassist` 修改`JacksonJsonGenerator` 的 `generator` 成员变量实际取值：
```java
ClassPool pool = ClassPool.getDefault();
try {
    CtClass cc = pool.get("org.elasticsearch.hadoop.serialization.json.JacksonJsonGenerator"); //这里必须用类全限定名

    //这里自己造了一个无参构造器给原构造器调用，否则 JacksonJsonGenerator 的 currentPath 一直是null（字段有初始化值但还是null），原因未知，可能是setBody影响了原构造器的行为
    cc.addConstructor(CtNewConstructor.make("JacksonJsonGenerator(){this.currentPath = new java.util.LinkedList();}", cc));
    //构造函数将 generator 替换成我们的 Adaptor
    CtConstructor jacksonJsonGeneratorConstructor = cc.getDeclaredConstructor(new CtClass[]{pool.get(OutputStream.class.getName())});
    jacksonJsonGeneratorConstructor.setBody("{\n" +
            "        this();\n" + //调用无参构造器，这里用 $0() 是不行的，必须this();
            "        try {\n" +
            "            $0.out = $1;\n" +
            "            $0.generator = new xxx.yyy.zzz.JacksonJsonGeneratorAdaptor($1);\n" +
            "        } catch (java.io.IOException ex) {\n" +
            "            throw new org.elasticsearch.hadoop.serialization.EsHadoopSerializationException(ex);\n" +
            "        }\n" +
            "    }");

    cc.toClass().getConstructor(OutputStream.class).newInstance(System.out);
    log.info("完成对 JacksonJsonGenerator 进行静态代码块和构造方法的pack");
} catch (Exception e) {
    log.error("给 JacksonJsonGenerator 进行静态代码块和构造方法的pack失败:" + e.getMessage(), e);
}
```

完事儿。