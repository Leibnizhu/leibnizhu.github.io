---
title: SpringMVC中的WebSocket开发
date: 2017-07-29 15:26:32
tags:
- Java
- SpringMVC
- WebSocket
image: snow.png
---
# SpringMVC中的WebSocket开发
## WebSocket简介
### WebSocket背景
在WebSocket出现之前，服务器的状态更新想要通知客户端，只能由客户端发起轮询（如Ajax）， 即在特定的的时间间隔（如每1秒），由浏览器对服务器发出HTTP request，然后由服务器返回最新的数据给客服端的浏览器。这种传统的HTTP request 的模式带来很明显的缺点 – 浏览器需要不断的向服务器发出请求，然而HTTP request 的header是非常长的，里面包含的有用数据可能只是一个很小的值，这样会占用很多的带宽。
WebSocket是HTML5开始提供的一种在单个 TCP 连接上进行全双工通讯的协议。WebSocket通讯协议于2011年被IETF定为标准RFC 6455，WebSocketAPI被W3C定为标准。
在WebSocket API中，浏览器和服务器只需要做一个握手的动作，然后，浏览器和服务器之间就形成了一条快速通道。两者之间就直接可以数据互相传送。WebSocket不仅允许服务器和客户端双向通信，而且互相沟通的Header是很小的-大概只有 2 Bytes。

### 支持情况
- Spring： Spring从4.0开始加入了spring-websocket这个模块，并能够全面支持WebSocket，它与Java WebSocket API标准（JSR-356）保持一致，同时提供了额外的服务。
- 浏览器：

| 浏览器 | 支持的版本 |
|:------:|:------:|
| Chrome |  4+ |
| Firefox |  4+|
| Internet Explorer |  10+|
| Opera |  10+|
| Safari |  5+|

- 服务端：

| 服务器 | 支持的版本 |
|:------:|:------:|
| jetty | 7.0.1+ |
| tomcat | 7.0.27+ |
| Nginx | 1.3.13+ |
| resin |  4+|

### Java 实现方法
在 Spring 端可以有以下几种方法使用 WebSocket：
1. 使用 Java EE7 的方式
2. 使用 Spring 提供的接口
3. 使用 STOMP 协议以及 Spring 的 MVC

本文使用Spring提供的接口，实现起来比较简单。

### 适用场景
客户端和服务器需要 **高频率** **低延迟** 交换事件的时候。基本的候选包括但不限于，金融、游戏、合作、以及其他应用。这些应用对时间延迟很敏感，还需要以高频率交换大量的消息。

## Spring MVC的WebSocket开发实战
### Nginx配置
我们知道，WebSocket握手需要在HTTP请求头里增加```Upgrade```和```Connection```字段，以便向服务申请将连接升级为WebSocket。  
但如果tomcat服务器使用了Nginx作为反向代理，那么默认是不会转发这两个请求头的，所以需要手动设置这两个HTTP请求头。  
应在```nginx.conf```对应域名```server```配置里面的```location```配置中增加：  
```bash
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection "upgrade";
```

###
### Maven依赖
在pom.xml文件中增加以下依赖：
```xml
<dependency>
	<groupId>org.springframework</groupId>
	<artifactId>spring-websocket</artifactId>
	<version>${spring-version}</version>
</dependency>
```
### WebSocket相关的类
#### 实现WebSocketConfigurer
对Spring WebSocket进行配置，可以通过xml配置文件的方式，也可以通过实现WebSocketConfigurer接口进行配置：
```java
@Configuration
@EnableWebSocket
public class WebSocketConfig extends WebMvcConfigurerAdapter implements WebSocketConfigurer {
    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        registry.addHandler(SystemWebSocketHandler.getInstance(),"/webSocketServer") //注册WebSocket处理的类的、及监听/映射路径
                .addInterceptors(new WebSocketHandshakeInterceptor()); //注册WebSocket握手的拦截器

        registry.addHandler(SystemWebSocketHandler.getInstance(), "/sockjs/webSocketServer") //注册WebSocket处理的类的、及监听/映射路径
                .addInterceptors(new WebSocketHandshakeInterceptor()) //注册WebSocket握手的拦截器
                .withSockJS(); //设定支持SockJS
    }
}
```
我们设置了两个监听的路径，第一个是传统的WebSocket，第二个是支持SockJS的。SockJS是一个JavaScript库，提供跨浏览器JavaScript的API，创建了一个低延迟、全双工的浏览器和web服务器之间通信通道。SockJS的API的命名方式基本上也和 WebSocket 一样，并且支持自动降级到AJAX轮询（降级顺序依次为：websocket -> html strea m -> long polling -> ajaxjsonp），因此可以很好地跨浏览器工作。
在配置文件里，我们设定了```SystemWebSocketHandler```类（实现```WebSocketHandler```接口，类似Controller）作为WebSocket各种事件的处理器，以及设定```WebSocketHandshakeInterceptor```类（实现```HandshakeInterceptor```接口）作为WebSocket协议握手的拦截器，这两个类时我们自己实现的，将在下文细述。

#### 实现WebSocketHandler接口
WebSocketHandler接口为WebSocket事件处理器接口，有以下方法需要实现：
```java
public interface WebSocketHandler {
	//WebSocket连接建立后的回调方法
	void afterConnectionEstablished(WebSocketSession var1) throws Exception;
	//接收到WebSocket消息后的处理方法
	void handleMessage(WebSocketSession var1, WebSocketMessage<?> var2) throws Exception;
	//WebSocket传输发生错误时的处理方法
	void handleTransportError(WebSocketSession var1, Throwable var2) throws Exception;
	//WebSocket连接关闭后的回调方法
	void afterConnectionClosed(WebSocketSession var1, CloseStatus var2) throws Exception;
	//是否处理WebSocket分段消息
	boolean supportsPartialMessages();
}
```
对于业务逻辑而言，我们主要关注```afterConnectionEstablished()```方法（进行一些初始化工作），以及```handleMessage()```方法（处理页面发出的消息）。其余方法的实现内容相对固定，发生错误和连接关闭应该响应地关闭一些资源，至于分段消息，暂时用不到，可以直接返回```false```。
下面给出一个简单的实现：
```java
/**
 * 实现WebSocketHandler接口,作为WebSocket各种事件的处理器
 *
 * @author leibniz
 */
public class SystemWebSocketHandler implements WebSocketHandler {
    private static Logger LOG;

    /**
     * 维护所有已创建的WebSocket Session，key为用户ID（OpenID或管理员的名字）
     * 考虑到一个用户可能打开多个页面（如管理员可能在手机和PC登录，且多个人用同一个账号），这里使用Guava的Multimap来缓存
     */
    private static Multimap<String, WebSocketSession> WS_SESSION_MAP;

    /**
     * 单例的对象
     */
    private static final SystemWebSocketHandler INSTANCE = new SystemWebSocketHandler();

    public static SystemWebSocketHandler getInstance(){
        return INSTANCE;
    }

    /**
     * WebSocketSession中保存当前用户ID的Attribute key
     */
    static final String WEBSOCKET_USERID = "WS_USERID";

    /**
     * 默认构造器，初始化日志对象和Session缓存Map
     */
    private SystemWebSocketHandler(){
        LOG = LoggerFactory.getLogger(SystemWebSocketHandler.class);
        WS_SESSION_MAP = HashMultimap.create();
    }

    /**
     * 建立连接后，将用户ID和WebSocketSession对象的映射保存到WS_SESSION_MAP
     *
     * @param session WebSocketSession 对象
     * @throws Exception 接口方法声明的异常
     *
     * @author Leibniz
     */
    @Override
    public void afterConnectionEstablished(WebSocketSession session) throws Exception {
        LOG.debug("connect to the websocket success......");
        String userId = (String) session.getAttributes().get(WEBSOCKET_USERID);
        WS_SESSION_MAP.put(userId, session);
    }

    /**
     * 接收到WebSocket消息后的处理方法vvvvv
     * 暂不处理
     *
     * @param webSocketSession WebSocketSession对象
     * @param webSocketMessage 页面发送的WebSocketMessage消息对象
     * @throws Exception 接口方法声明的异常
     * @author Leibniz
     */
    @Override
    public void handleMessage(WebSocketSession webSocketSession, WebSocketMessage<?> webSocketMessage) throws Exception {

    }

    /**
     * WebSocket传输发生错误时，关闭WebSocketSession，并从WS_SESSION_MAP中删除
     *
     * @param session WebSocketSession对象
     * @param exception 异常
     * @throws Exception 接口方法声明的异常
     *
     * @author Leibniz
     */
    @Override
    public void handleTransportError(WebSocketSession session, Throwable exception) throws Exception {
        if(session.isOpen()){
            session.close();
        }
        LOG.debug("websocket connection closed......");
        for(Map.Entry<String, WebSocketSession> entry : WS_SESSION_MAP.entries()){
            if(entry.getValue().equals(session)){
                WS_SESSION_MAP.remove(entry.getKey(), session);
            }
        }
    }

    /**
     * WebSocket连接关闭后，从WS_SESSION_MAP中删除对应WebSocketSession
     *
     * @param session  WebSocketSession对象
     * @param closeStatus 关闭状态
     * @throws Exception 接口方法声明的异常
     *
     * @author Leibniz
     */
    @Override
    public void afterConnectionClosed(WebSocketSession session, CloseStatus closeStatus) throws Exception {
        LOG.debug("websocket connection closed......");
        for(Map.Entry<String, WebSocketSession> entry : WS_SESSION_MAP.entries()){
            if(entry.getValue().equals(session)){
                WS_SESSION_MAP.remove(entry.getKey(), session);
            }
        }
    }

    /**
     * 不支持分段消息
     * @return false
     */
    @Override
    public boolean supportsPartialMessages() {
        return false;
    }

    /**
     * 给所有在线用户发送消息
     *
     * @param message 需要发送的消息对象
     *
     * @author Leibniz
     */
    public void sendMessageToUsers(TextMessage message) {
        for (WebSocketSession user : WS_SESSION_MAP.values()) {
            try {
                if (user.isOpen()) {
                        user.sendMessage(message);
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    /**
     * 给某个用户发送消息
     *
     * @param userId 用户ID（OpenID或管理员的名字）
     * @param message 需要发送的消息对象
     *
     * @author Leibniz
     */
    public void sendMessageToUser(String userId, TextMessage message) {
        Collection<WebSocketSession> users = WS_SESSION_MAP.get(userId);
        try {
            for(WebSocketSession user : users){
                if (user != null && user.isOpen()) {
                    user.sendMessage(message);
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```
值得注意的是，使用```WebSocketSession.sendMessage()```方法可以向指定用户页面发送消息。

#### 实现HandshakeInterceptor接口
HandshakeInterceptor接口为WebSocket握手拦截器接口，包含以下方法：
```java
public interface HandshakeInterceptor {
	//建立WebSocket连接、握手前的处理方法
	boolean beforeHandshake(ServerHttpRequest request, ServerHttpResponse response, WebSocketHandler wsHandler, Map<String, Object> attributes) throws Exception;
	//建立WebSocket连接、握手后的处理方法
	void afterHandshake(ServerHttpRequest request, ServerHttpResponse response, WebSocketHandler wsHandler, Exception exception);
}
```
在前面的WebSocket配置类里面，这个接口的实现类用于拦截WebSocket连接握手，在握手前后都可以拦截。
我们的应用里用到这个握手拦截器，主要是因为在WebSocketHandler接口的方法中，只能拿到WebSocketSession对象，无法直接与用户请求的HttpSession建立关联。
而在握手拦截器中，通过ServerHttpRequest对象可以拿到关于当前用户、当前连接的很多相关信息，包括HttpSession及其属性；同时通过attributes参数可以设置最终生成的WebSocketSession对象的属性；从而WebSocketSession和HttpSession就可以建立起关联。
从一个简单的实现类中就可以清晰看到这一点：
```java
public class WebSocketHandshakeInterceptor implements HandshakeInterceptor {

    private static Logger LOG = LoggerFactory.getLogger(HandshakeInterceptor.class);

    /**
     * 建立WebSocket连接、握手前的处理方
     * 从HttpSession读取当前用户的用户ID（OpenID或管理员的名字），写入attributes
     *
     * @param request Http请求对象
     * @param response Http响应对象
     * @param wsHandler WebSocketHandler实现类的实例，这里是SystemWebSocketHandler类
     * @param attributes 握手生成的WebSocketSession对象的属性
     * @return 是否成功
     * @throws Exception 接口方法声明的异常
     *
     * @author Leibniz
     */
    @Override
    public boolean beforeHandshake(ServerHttpRequest request, ServerHttpResponse response, WebSocketHandler wsHandler, Map<String, Object> attributes) throws Exception {
        if (request instanceof ServletServerHttpRequest) {
            ServletServerHttpRequest servletRequest = (ServletServerHttpRequest) request;
            HttpSession session = servletRequest.getServletRequest().getSession(false);
            if (session != null) {
                //使用userName区分WebSocketHandler，以便定向发送消息
                String userId = (String) session.getAttribute(RegisterLoginController.OPENID_KEY);//普通用户是OpenID
                String adminUserId = (String) session.getAttribute(AdminUserController.USER_NAME);//管理员用户是用户名
                if( null != adminUserId){
                    //优先保存管理员的用户ID1
                    attributes.put(WEBSOCKET_USERID, adminUserId);
                } else {
                    attributes.put(WEBSOCKET_USERID, userId);
                }
            }
        }
        return true;
    }

    /**
     * 建立WebSocket连接、握手后的处理方法
     *
     * @param request Http请求对象
     * @param response Http响应对象
     * @param wsHandler WebSocketHandler实现类的实例，这里是SystemWebSocketHandler类
     * @param exception 抛出的异常
     *
     * @author Leibniz
     */
    @Override
    public void afterHandshake(ServerHttpRequest request, ServerHttpResponse response, WebSocketHandler wsHandler, Exception exception) {
    }
}
```

#### 业务层的封装
实际的使用中，我们封装了一些类，包括WebSocket消息内容的实体类，以及发送消息的Service类（在Controller层触发了相应的事件时进行调用），以下代码仅供参考，请根据实际业务需求进行封装。
```java
/**
 * WebSocket消息的统一封装
 *
 * @author Leibniz
 */
public class WebSocketMessage {
    private ROLE role;//接受消息的角色，枚举，NORMAL=普通用户，ADMIN=客服/管理员
    private String id;//用户ID（OpenID或管理员的名字）
    private String event;//事件，实际上是数字，10001=客服确认借出，10002=客服确认归还，20001=用户申请租赁，20002=用户申请归还
    private String msg;//事件消息，具体的文字描述，英文

    public WebSocketMessage(ROLE role, String id, String event, String msg) {
        this.role = role;
        this.id = id;
        this.event = event;
        this.msg = msg;
    }

    @Override
    public String toString() {
        JSONObject json = new JSONObject();
        json.put("role", this.role.toString());
        json.put("id", this.id);
        json.put("event", this.event);
        json.put("msg", this.msg);
        return json.toString();
    }

    enum ROLE {
        NORMAL {
            @Override
            public String toString() {
                return "normal";
            }
        }, ADMIN {
            @Override
            public String toString() {
                return "admin";
            }
        }
    }
}

/**
 * 发送WebSocket消息的Service类
 *
 * @author Leibniz
 */
@Service
public class WebSocketMessageService {
    @Autowired
    private AdminUserDao adminUserDao;
    private List<AdminUserEntity> adminUserList;

    /**
      * 向普通用户发送客服确认借出消息
     *
      * @param openId 用户OpenID
      * @author Leibniz
      */
    public void sendBorrowConfirm(String openId){
        WebSocketMessage msg = new WebSocketMessage(NORMAL, openId, "10001", "admin_borrow_confirm");
        SystemWebSocketHandler.getInstance().sendMessageToUser(openId, new TextMessage(msg.toString()));
    }

    /**
     * 向普通用户发送客服确认归还消息
     *
     * @param openId 用户OpenID
     * @author Leibniz
     */
    public void sendReturnConfirm(String openId){
        WebSocketMessage msg = new WebSocketMessage(NORMAL, openId, "10002", "admin_return_confirm");
        SystemWebSocketHandler.getInstance().sendMessageToUser(openId, new TextMessage(msg.toString()));
    }

    /**
     * 向超级管理员及指定柜子对应的客服发送用户申请租赁的消息
     *
     * @param boxId 柜子ID
     * @author Leibniz
     */
    public void sendBorrowApply(int boxId){
        //检查管理员列表是否已加载
        if(null == adminUserList){
            adminUserList = adminUserDao.selectAllAdminUser();
        }
        for(AdminUserEntity adminUser : adminUserList) {
            if(adminUser.getBoxId()== null || adminUser.getBoxId().equals(boxId)){
                //当前遍历到的是超级管理员或指定柜子对应的客服
                String adminName = adminUser.getUserName();
                WebSocketMessage msg = new WebSocketMessage(ADMIN, adminName, "20001", "user_borrow_apply");
                SystemWebSocketHandler.getInstance().sendMessageToUser(adminName, new TextMessage(msg.toString()));
            }
        }
    }

    /**
     * 向超级管理员及指定柜子对应的客服发送用户申请归还的消息
     *
     * @param boxId 柜子ID
     * @author Leibniz
     */
    public void sendReturnApply(Integer boxId){
        //检查管理员列表是否已加载
        if(null == adminUserList){
            adminUserList = adminUserDao.selectAllAdminUser();
        }
        for(AdminUserEntity adminUser : adminUserList) {
            if(adminUser.getBoxId() == null || adminUser.getBoxId().equals(boxId)) {
                //当前遍历到的是超级管理员或指定柜子对应的客服
                String adminName = adminUser.getUserName();
                WebSocketMessage msg = new WebSocketMessage(ADMIN, adminName, "20002", "user_return_apply");
                SystemWebSocketHandler.getInstance().sendMessageToUser(adminName, new TextMessage(msg.toString()));
            }
        }
    }
}
```

### 页面&&js
页面需要引入SockJS，js中需要初始化WebSocket并建立链接（前面在WebSocketConfig类中配置的映射路径）。
```html
<!DOCTYPE html>
<html xmlns="http://www.w3.org/1999/xhtml" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="utf-8"/>
</head>
<body>
<div id="msgcount"></div>
<script src="../js/libs/jquery-2.0.2.min.js" th:src="@{/js/libs/jquery-2.0.2.min.js}" type="text/javascript"></script>
<script src="../js/sockjs.min.js" th:src="@{/js/sockjs.min.js}" type="text/javascript"></script>
<script>
    if(typeof $basePath === "undefined"){
        window.$basePath = "/breo/";
    }
    var websocket;
    //根据当前浏览器支持的WebSocket对象类型进行初始化
    if ('WebSocket' in window) {
        //浏览器内置WebSocket API
        websocket = new WebSocket("ws://"+window.location.host+$basePath+"webSocketServer");
    } else if ('MozWebSocket' in window) {
        //Firefox浏览器
        websocket = new MozWebSocket("ws://"+window.location.host+$basePath+"webSocketServer");
    } else {
        //其他浏览器，或不支持WebSocket
        websocket = new SockJS("http://"+window.location.host+$basePath+"sockjs/webSocketServer");
    }
    //WebSocket连接打开的回调方法
    websocket.onopen = function (evnt) {
    };
    //页面接收到WebSocket消息的回调方法
    websocket.onmessage = function (evnt) {
        var msg = JSON.parse(evnt.data);
        console.log(msg);
        if(msg.role === "normal" && msg.event === "10001"){
            $("#msgcount").html("<font color='red'>"+JSON.stringify(msg)+"</font>")
        }
    };
    //WebSocket发生错误的回调方法
    websocket.onerror = function (evnt) {
    };
    //WebSocket连接关闭的回调方法
    websocket.onclose = function (evnt) {
    }
</script>
</body>
</html>
```
