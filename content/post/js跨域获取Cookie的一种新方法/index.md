---
title: js跨域获取Cookie的一种新方法
date: 2017-01-12 14:01:25
tags:
- Netty
- Cookie
- 跨域
---

# 背景
同一个项目分配了多个域名，在其中一个域名（下称域名A）的一级域名上放了Cookie。  
使用其他域名(下面统称域名B\*)去访问某些页面时，需要使用js读取域名A下的那个Cookie。

# 已有的解决方案
网上已经有一些解决方案，如：[JS跨域（ajax跨域、iframe跨域）解决方法及原理详解（jsonp）](https://m.th7.cn/show/22/201503/88209.htm)、[JS 获取跨域的cookie](http://www.cnblogs.com/chris-shao/archive/2012/12/27/2835986.html)。  
其中document.domain的方法已确认不可用于我的需求，iframe跨域可以实现，但是跨域后的页面在iframe中，此时js变量的作用域只在iframe中，需要通过一个中间元素的值或者属性来写入需要传递的值，比较麻烦；document.name的方法虽然可以跨域，但同时也跨页面了，处理起来要小心点。  

# 本文的解决方案
## 原理
在域名B\*的页面上，js的确不能直接获取到域名A的cookie，但显然，域名B\*的页面如果发请求到域名A，会带上域名A的Cookie。  
针对这一点，只要我们在域名A上面部署一个微服务，域名B\*的页面发AJAX请求到这个服务，返回域名A的Cookie中我们感兴趣字段的值。域名B\*的页面就能接收到域名A的Cookie，可以各种利用了。

## Netty实现
域名A的服务器端选用Netty实现。实现代码很简单，这里只给出核心Handler部分代码：
```java
public class RPCMissionHandler extends SimpleChannelInboundHandler<FullHttpRequest> {
    private static Logger LOG = Logger.getLogger(RPCMissionHandler.class);

    @Override
    public void channelRead0(ChannelHandlerContext ctx, FullHttpRequest req) throws Exception {
        if(!req.decoderResult().isSuccess()){
            sendError(ctx, BAD_REQUEST);
            return;
        }
        if(req.method() != GET){
            sendError(ctx, METHOD_NOT_ALLOWED);
            return;
        }
        String uri = req.uri();
        String paramUri = uri.substring(uri.indexOf("?") + 1);
        List<NameValuePair> params = URLEncodedUtils.parse(paramUri, Charset.forName("UTF-8"));
       if(uri.startsWith("/getid")){
            String refererDomain = null;
            for(NameValuePair pair : params){
                if("domain".equals(pair.getName())){
                    refererDomain = pair.getValue();
                }
            }
            if(refererDomain != null){
                if(req.headers().get("Cookie") !=  null) {
                    Set<Cookie> cookies = CookieDecoder.decode(req.headers().get("Cookie"));
                    for (Cookie cookie : cookies) {
                        if ("ID".equals(cookie.name())) {
                            LOG.info("Request ID = " + cookie.value());
                            responseWithString(ctx, cookie.value(), "http://" + refererDomain);
                            return;
                        }
                    }
                } else {
                    responseWithString(ctx, "", "http://" + refererDomain);
                    return;
                }
            }
        }
        responseWithImage(ctx);
    }

    private void responseWithString(ChannelHandlerContext ctx, String data, String referer) {
        FullHttpResponse resp = new DefaultFullHttpResponse(HttpVersion.HTTP_1_1, OK);
        /*配置不缓存*/
        resp.headers().set("Cache-Control", "no-cache");
        resp.headers().set("Pragma", "no-cache");
        resp.headers().set("Expires", "Wed, 31 Dec 1969 23:59:59 GMT");
        /*配置跨域允许*/
        resp.headers().set("Access-Control-Allow-origin", referer);
        resp.headers().set("Access-Control-Allow-Methods", "GET, POST");
        resp.headers().set("Access-Control-Allow-Credentials", "true");
        /*响应类型*/
        resp.headers().set("Content-Type", "text/html;charset=UTF-8");
        ByteBuf buf = Unpooled.copiedBuffer(new StringBuffer(data), CharsetUtil.UTF_8);
        resp.content().writeBytes(buf);
        buf.release();
        ctx.writeAndFlush(resp).addListener(ChannelFutureListener.CLOSE);
    }

    private void sendError(ChannelHandlerContext ctx, HttpResponseStatus status) {
        FullHttpResponse resp = new DefaultFullHttpResponse(HttpVersion.HTTP_1_1, status, Unpooled.copiedBuffer("Failure:" + status, CharsetUtil.UTF_8));
        resp.headers().set(CONTENT_TYPE, "text/plain; charset=UTF-8");
        ctx.writeAndFlush(resp).addListener(ChannelFutureListener.CLOSE);
    }

    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
        ctx.writeAndFlush(Unpooled.EMPTY_BUFFER);
        super.channelReadComplete(ctx);
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        LOG.error("抛出异常", cause);
        super.exceptionCaught(ctx, cause);
    }
}
```

唯一需要注意的是，在请求的时候，通过`domain`参数带上了域名B\*，然后在响应的时候，配置跨域相关的一些HTTP响应头：
```java
resp.headers().set("Access-Control-Allow-origin", referer);
resp.headers().set("Access-Control-Allow-Methods", "GET, POST");
resp.headers().set("Access-Control-Allow-Credentials", "true");
```
其中referer就是域名B\*。

## 页面js调用
js发AJAX请求这块就更简单了。唯一注意的是，我们这个Cookie字段可能在页面加载后一段时间才能获取到，所以这里设置了重试机制，获取不到ID的时候等待1秒后重新发送一次请求，一共最多请求5次。  
这里发送AJAX请求也涉及到一些跨域的配置，详见注释
```javascript
$(window).load(function(){
    setTimeout(function(){getID(5);}, 1000);
    function getID(times){
        $.ajax({
            url: 'http://A.com/getid?domain='+document.domain,
            type: 'get',
            crossDomain: true, /*允许跨域*/
            xhrFields: { withCredentials: true }, /*允许跨域*/
            error: function(data){alert(data);},
            success: function (data) {
                if(data != ""){
                    ID = data;
                } else {
                    if(--times > 0)
                        setTimeout(function(){getID(times);}, 1000);
                }
            }
        });
    }
});
```
