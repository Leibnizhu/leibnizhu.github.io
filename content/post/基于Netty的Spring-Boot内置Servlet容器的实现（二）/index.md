---
title: 基于Netty的Spring Boot内置Servlet容器的实现（二）
date: 2017-08-24 17:43:37
tags:
- Netty
- Spring Boot
- Servlet
- Servlet Context
image: sanae.png
---
# 基于Netty的Spring Boot内置Servlet容器的实现（二）
## 实现Servlet Context接口
### Servlet Context接口简介
接口```ServletContext```定义了一系列方法用于与相应的servlet容器通信，比如：获得文件的MIME类型，分派请求，或者是向日志文件写日志等。每一个web-app只能有一个```ServletContext```，webapp可以是一个放置有web application 文件的文件夹，也可以是一个.war的文件。```ServletContext```对象包含在```ServletConfig```对象之中，```ServletConfig```对象在servlet初始化时提供servlet对象。
接口```ServletContext```定义的方法比较多，大致可以分为：添加和配置Servlet、添加和配置Filter、添加和配置Listener、添加Servlet、Filter和Listener的注解处理需求、初始化参数、Context属性、资源获取等几大类方法。  
具体可以参考：
- [Tomcat的JavaDoc](https://tomcat.apache.org/tomcat-8.0-doc/servletapi/index.html)
- [中文翻译的文档 ServletContext 接口介绍](https://waylau.gitbooks.io/servlet-3-1-specification/docs/Servlet%20Context/4.1%20Introduction%20to%20the%20ServletContext%20Interface.html)

### 实现
实现的思想：
- 不处理的：InitParameter相关的方法、Listener相关方法——目前用不到
- 以后处理的：Session Cookie相关的方法等待实现
- Context的Attributes用```Hashtable```实现，主要是考虑到相关的方法需要返回```Enumeration```类型，用```Hashtable```有现成方法可以返回。
- Filter的注册用```HashMap```存储FilterName及对应```Registration```的映射关系，暂时还没处理Filter的URL Pattern（所有注册的Filter对所有请求都会过滤，暂时可以满足需求）
- Servlet的注册也是用```HashMap```存储ServletName及对应```Registration```的映射关系，以及URL Pattern和ServletName的映射关系（相当与```web.xml```里的配置）

这里列出主要的方法，一些没具体实现，或者比较简单的方法就省略了（部分代码参考了Tomcat 8.0.45的源码）：  
```java
package io.gitlab.leibnizhu.sbnetty.core;

/**
 * ServletContext实现
 */
public class NettyContext implements ServletContext {
    private final Logger log = LoggerFactory.getLogger(getClass());

    private final String contextPath; //保证不以“/”结尾
    private final ClassLoader classLoader;
    private final String serverInfo;
    private volatile boolean initialized; //记录是否初始化完毕
    private RequestUrlPatternMapper servletUrlPatternMapper;

    private final Map<String, NettyServletRegistration> servlets = new HashMap<>(); //getServletRegistration()等方法要用，key是ServletName
    private final Map<String, NettyFilterRegistration> filters = new HashMap<>(); //getFilterRegistration()等方法要用，Key是FilterName
    private final Map<String, String> servletMappings = new HashMap<>(); //保存请求路径urlPattern与Servlet名的映射,urlPattern是不带contextPath的
    private final Hashtable<String, Object> attributes = new Hashtable<>();

    /**
     * 默认构造方法
     *
     * @param contextPath contextPath
     * @param classLoader classLoader
     * @param serverInfo  服务器信息，写在响应的server响应头字段
     */
    public NettyContext(String contextPath, ClassLoader classLoader, String serverInfo) {
        if(contextPath.endsWith("/")){
            contextPath = contextPath.substring(0, contextPath.length() -1);
        }
        this.contextPath = contextPath;
        this.classLoader = classLoader;
        this.serverInfo = serverInfo;
        servletUrlPatternMapper = new RequestUrlPatternMapper(servletMappings, contextPath);
    }

    private void checkNotInitialised() {
        checkState(!isInitialised(), "This method can not be called before the context has been initialised");
    }

    public void addServletMapping(String urlPattern, String name, Servlet servlet) {
        checkNotInitialised();
        servletMappings.put(urlPattern, checkNotNull(name));
        servletUrlPatternMapper.addWrapper(urlPattern, servlet, name);
    }

    public void addFilterMapping(EnumSet<DispatcherType> dispatcherTypes, boolean isMatchAfter, String urlPattern) {
        checkNotInitialised();
        //TODO 过滤器的urlPatter解析
    }

    @Override
    public String getMimeType(String file) {
        return MimeTypeUtil.getMimeTypeByFileName(file);
    }

    @Override
    public Set<String> getResourcePaths(String path) {
        Set<String> thePaths = new HashSet<>();
        if (!path.endsWith("/")) {
            path += "/";
        }
        String basePath = getRealPath(path);
        if (basePath == null) {
            return thePaths;
        }
        File theBaseDir = new File(basePath);
        if (!theBaseDir.exists() || !theBaseDir.isDirectory()) {
            return thePaths;
        }
        String theFiles[] = theBaseDir.list();
        if (theFiles == null) {
            return thePaths;
        }
        for (String filename : theFiles) {
            File testFile = new File(basePath + File.separator + filename);
            if (testFile.isFile())
                thePaths.add(path + filename);
            else if (testFile.isDirectory())
                thePaths.add(path + filename + "/");
        }
        return thePaths;
    }

    @Override
    public URL getResource(String path) throws MalformedURLException {
        if (!path.startsWith("/"))
            throw new MalformedURLException("Path '" + path + "' does not start with '/'");
        URL url = new URL(getClassLoader().getResource(""), path.substring(1));
        try {
            url.openStream();
        } catch (Throwable t) {
            log.error("Throwing exception when getting InputStream of " + path, t);
            url = null;
        }
        return url;
    }

    @Override
    public InputStream getResourceAsStream(String path) {
        try {
            return getResource(path).openStream();
        } catch (IOException e) {
            log.error(e.getMessage(), e);
            return null;
        }
    }

    @Override
    public RequestDispatcher getRequestDispatcher(String path) {
        String servletName = servletUrlPatternMapper.getServletNameByRequestURI(path);
        Servlet servlet;
        try {
            servlet = null == servletName ? null : servlets.get(servletName).getServlet();
            if (servlet == null) {
                return null;
            }
            //TODO 过滤器的urlPatter解析
            List<Filter> allNeedFilters = new ArrayList<>();
            for (NettyFilterRegistration registration : this.filters.values()) {
                allNeedFilters.add(registration.getFilter());
            }
            FilterChain filterChain = new SimpleFilterChain(servlet, allNeedFilters);
            return new NettyRequestDispatcher(this, filterChain);
        } catch (ServletException e) {
            log.error("Throwing exception when getting Filter from NettyFilterRegistration of path " + path, e);
            return null;
        }
    }

    @Override
    public String getRealPath(String path) {
        if (!path.startsWith("/"))
            return null;
        try {
            File f = new File(getResource(path).toURI());
            return f.getAbsolutePath();
        } catch (Throwable t) {
            log.error("Throwing exception when getting real path of " + path, t);
            return null;
        }
    }

    @Override
    public String getServerInfo() {
        return serverInfo;
    }

    // InitParameter相关的方法不实现（返回空/空集合）基本用不到

    @Override
    public Object getAttribute(String name) {
        return attributes.get(name);
    }

    @Override
    public Enumeration<String> getAttributeNames() {
        return attributes.keys();
    }

    @Override
    public void setAttribute(String name, Object object) {
         attributes.put(name, object);
    }

    @Override
    public void removeAttribute(String name) {
        attributes.remove(name);
    }

    @Override
    public ServletRegistration.Dynamic addServlet(String servletName, String className) {
        return addServlet(servletName, className, null);
    }

    @Override
    public ServletRegistration.Dynamic addServlet(String servletName, Servlet servlet) {
        return addServlet(servletName, servlet.getClass().getName(), servlet);
    }

    @Override
    public ServletRegistration.Dynamic addServlet(String servletName, Class<? extends Servlet> servletClass) {
        return addServlet(servletName, servletClass.getName());
    }

    private ServletRegistration.Dynamic addServlet(String servletName, String className, Servlet servlet) {
        NettyServletRegistration servletRegistration = new NettyServletRegistration(this, servletName, className, servlet);
        servlets.put(servletName, servletRegistration);
        return servletRegistration;
    }

    @Override
    public javax.servlet.FilterRegistration.Dynamic addFilter(String filterName, String className) {
        return addFilter(filterName, className, null);
    }

    @Override
    public javax.servlet.FilterRegistration.Dynamic addFilter(String filterName, Filter filter) {
        return addFilter(filterName, filter.getClass().getName(), filter);
    }

    private javax.servlet.FilterRegistration.Dynamic addFilter(String filterName, String className, Filter filter) {
        NettyFilterRegistration filterRegistration = new NettyFilterRegistration(this, filterName, className, filter);
        filters.put(filterName, filterRegistration);
        return filterRegistration;
    }

    @Override
    public javax.servlet.FilterRegistration.Dynamic addFilter(String filterName, Class<? extends Filter> filterClass) {
        return addFilter(filterName, filterClass.getName());
    }

    @Override
    public <T extends Filter> T createFilter(Class<T> c) throws ServletException {
        try {
            return c.newInstance();
        } catch (InstantiationException | IllegalAccessException e) {
            e.printStackTrace();
            return null;
        }
    }

    @Override
    public javax.servlet.FilterRegistration getFilterRegistration(String filterName) {
        return filters.get(filterName);
    }

    @Override
    public Map<String, ? extends FilterRegistration> getFilterRegistrations() {
        return ImmutableMap.copyOf(filters);
    }
    //TODO Session Cookie相关的方法等待实现

    //TODO 暂不支持Listener，现在很少用了吧
}
```

### URL Pattrn匹配查找
参考Tomcat源码，设计了一个```RequestUrlPatternMapper```类用于保存，计算URL-pattern与请求路径的匹配关系。在```NettyContext```的```public RequestDispatcher getRequestDispatcher(String path)```方法中可以看到对其的调用，传入请求的路径，返回对应处理的Servlet名称。此外在```NettyContext```的``` public void addServletMapping(String urlPattern, String name, Servlet servlet)```方法中也调用该类，增加新的Servlet映射。  
增加映射的时候，先后判断：
1. 路径匹配
2. 扩展名匹配
3. 默认匹配
4. 精确匹配

用```MappedWrapper```类包装起新的Servlet，根据对应的匹配策略，放加入到```ContextVersion```实例的```wildcardWrappers```、```extensionWrappers```、```defaultWrapper```、```exactWrappers```中进行保存。  
在查询匹配的时候，处理完请求路径后，根据URL Pattern的定义，先后根据以下匹配方法进行匹配：
1. 精确匹配
2. 路径匹配
3. 后缀名匹配
4. Welcome资源匹配
5. 默认Servlet匹配

使用```MappingData```类实例对查询结果进行保存，每一级匹配如果已经找到对应的Servlet，那么下一级的匹配将不会进行，直接返回，此时```MappingData```对象里保存的就是最终匹配到的结果。  
具体的匹配中，精确匹配直接对Map进行查找即可，后缀名匹配类似，根据当前请求的后缀名进行精确匹配；而路径匹配，则是将路径进行降序排序，匹配的时候依次匹配，就能匹配到最长的那一个。  
下面贴上主要的实现代码：  
```java
package io.gitlab.leibnizhu.sbnetty.utils;

/**
 * 保存，计算URL-pattern与请求路径的匹配关系
 *
 * @author Leibniz.Hu
 * Created on 2017-08-25 11:32.
 */
public class RequestUrlPatternMapper {
    private final Logger log = LoggerFactory.getLogger(getClass());

    private UrlPatternContext urlPatternContext;
    private String contextPath;

    public RequestUrlPatternMapper(String contextPath) {
        this.urlPatternContext = new UrlPatternContext();
        this.contextPath = contextPath;
    }

    /**
     * 增加映射关系
     *
     * @param urlPattern  urlPattern
     * @param servlet     servlet对象
     * @param servletName servletName
     * @author Leibniz
     */
    public void addServlet(String urlPattern, Servlet servlet, String servletName) throws ServletException {
        if (urlPattern.endsWith("/*")) {
            // 路径匹配
            String pattern = urlPattern.substring(0, urlPattern.length() - 1);
            for (MappedServlet ms : urlPatternContext.wildcardServlets) {
                if (ms.pattern.equals(pattern)) {
                    throw new ServletException("URL Pattern('" + urlPattern + "') already exists!");
                }
            }
            MappedServlet newServlet = new MappedServlet(pattern, servlet, servletName);
            urlPatternContext.wildcardServlets.add(newServlet);
            urlPatternContext.wildcardServlets.sort((o1, o2) -> o2.pattern.compareTo(o1.pattern));
            log.debug("Curretn Wildcard URL Pattern List = " + Arrays.toString(urlPatternContext.wildcardServlets.toArray()));
        } else if (urlPattern.startsWith("*.")) {
            // 扩展名匹配
            String pattern = urlPattern.substring(2);
            if (urlPatternContext.extensionServlets.get(pattern) != null) {
                throw new ServletException("URL Pattern('" + urlPattern + "') already exists!");
            }
            MappedServlet newServlet = new MappedServlet(pattern, servlet, servletName);
            urlPatternContext.extensionServlets.put(pattern, newServlet);
            log.debug("Curretn Extension URL Pattern List = " + Arrays.toString(urlPatternContext.extensionServlets.keySet().toArray()));
        } else if (urlPattern.equals("/")) {
            // Default资源匹配
            if (urlPatternContext.defaultServlet != null) {
                throw new ServletException("URL Pattern('" + urlPattern + "') already exists!");
            }
            urlPatternContext.defaultServlet = new MappedServlet("", servlet, servletName);
        } else {
            // 精确匹配
            String pattern;
            if (urlPattern.length() == 0) {
                pattern = "/";
            } else {
                pattern = urlPattern;
            }
            if (urlPatternContext.exactServlets.get(pattern) != null) {
                throw new ServletException("URL Pattern('" + urlPattern + "') already exists!");
            }
            MappedServlet newServlet = new MappedServlet(pattern, servlet, servletName);
            urlPatternContext.exactServlets.put(pattern, newServlet);
            log.debug("Curretn Exact URL Pattern List = " + Arrays.toString(urlPatternContext.exactServlets.keySet().toArray()));
        }
    }

    /**
     * 删除映射关系
     *
     * @param urlPattern
     */
    public void removeServlet(String urlPattern) {
        if (urlPattern.endsWith("/*")) {
            //路径匹配
            String pattern = urlPattern.substring(0, urlPattern.length() - 2);
            urlPatternContext.wildcardServlets.removeIf(mappedServlet -> mappedServlet.pattern.equals(pattern));
        } else if (urlPattern.startsWith("*.")) {
            // 扩展名匹配
            String pattern = urlPattern.substring(2);
            urlPatternContext.extensionServlets.remove(pattern);
        } else if (urlPattern.equals("/")) {
            // Default资源匹配
            urlPatternContext.defaultServlet = null;
        } else {
            // 精确匹配
            String pattern;
            if (urlPattern.length() == 0) {
                pattern = "/";
            } else {
                pattern = urlPattern;
            }
            urlPatternContext.exactServlets.remove(pattern);
        }
    }

    public String getServletNameByRequestURI(String absoluteUri) {
        MappingData mappingData = new MappingData();
        try {
            matchRequestPath(absoluteUri, mappingData);
        } catch (IOException e) {
            log.error("Throwing exception when getting Servlet Name by request URI, maybe cause by lacking of buffer size.", e);
        }
        return mappingData.servletName;
    }

    /**
     * Wrapper mapping.
     *
     * @throws IOException buffer大小不足
     */
    private void matchRequestPath(String absolutePath, MappingData mappingData) throws IOException {
        // 处理ContextPath，获取访问的相对URI
        boolean noServletPath = absolutePath.equals(contextPath) || absolutePath.equals(contextPath + "/");
        if (!absolutePath.startsWith(contextPath)) {
            return;
        }
        String path = noServletPath ? "/" : absolutePath.substring(contextPath.length());
        //去掉查询字符串
        int queryInx = path.indexOf('?');
        if(queryInx > -1){
            path = path.substring(0, queryInx);
        }

        // 优先进行精确匹配
        internalMapExactWrapper(urlPatternContext.exactServlets, path, mappingData);

        // 然后进行路径匹配
        if (mappingData.servlet == null) {
            internalMapWildcardWrapper(urlPatternContext.wildcardServlets, path, mappingData);
            //TODO 暂不考虑JSP的处理
        }

        if (mappingData.servlet == null && noServletPath) {
            // 路径为空时，重定向到“/”
            mappingData.servlet = urlPatternContext.defaultServlet.object;
            mappingData.servletName = urlPatternContext.defaultServlet.servletName;
            return;
        }

        // 后缀名匹配
        if (mappingData.servlet == null) {
            internalMapExtensionWrapper(urlPatternContext.extensionServlets, path, mappingData);
        }

        //TODO 暂不考虑Welcome资源

        // Default Servlet
        if (mappingData.servlet == null) {
            if (urlPatternContext.defaultServlet != null) {
                mappingData.servlet = urlPatternContext.defaultServlet.object;
                mappingData.servletName = urlPatternContext.defaultServlet.servletName;
            }
            //TODO 暂不考虑请求静态目录资源
            if (path.charAt(path.length() - 1) != '/') {
            }
        }
    }


    /**
     * 精确匹配
     */
    private void internalMapExactWrapper(Map<String, MappedServlet> servlets, String path, MappingData mappingData) {
        MappedServlet servlet = servlets.get(path);
        if (servlet != null) {
            mappingData.servlet = servlet.object;
            mappingData.servletName = servlet.servletName;
        }
    }

    /**
     * 路径匹配
     */
    private void internalMapWildcardWrapper(List<MappedServlet> servlets, String path, MappingData mappingData) {
        if (!path.endsWith("/")) {
            path = path + "/";
        }
        MappedServlet result = null;
        for (MappedServlet ms : servlets) {
            if (path.startsWith(ms.pattern)) {
                result = ms;
                break;
            }
        }
        if (result != null) {
            mappingData.servlet = result.object;
            mappingData.servletName = result.servletName;
        }
    }

    /**
     * 后缀名匹配
     */
    private void internalMapExtensionWrapper(Map<String, MappedServlet> servlets, String path, MappingData mappingData) {
        int dotInx = path.lastIndexOf('.');
        path = path.substring(dotInx + 1);
        MappedServlet servlet = servlets.get(path);
        if (servlet != null) {
            mappingData.servlet = servlet.object;
            mappingData.servletName = servlet.servletName;
        }
    }

    /*
     * 以下是用到的内部类
     */

    private class UrlPatternContext {
        MappedServlet defaultServlet = null; //默认Servlet
        Map<String, MappedServlet> exactServlets = new HashMap<>(); //精确匹配
        List<MappedServlet> wildcardServlets = new LinkedList<>(); //路径匹配
        Map<String, MappedServlet> extensionServlets = new HashMap<>(); //扩展名匹配
    }

    private class MappedServlet extends MapElement<Servlet> {
        @Override
        public String toString() {
            return pattern;
        }

        String servletName;

        MappedServlet(String name, Servlet servlet, String servletName) {
            super(name, servlet);
            this.servletName = servletName;
        }
    }

    private class MapElement<T> {
        final String pattern;
        final T object;

        MapElement(String pattern, T object) {
            this.pattern = pattern;
            this.object = object;
        }
    }
}

public class MappingData {
    Servlet servlet = null;
    String servletName;
    String redirectPath ;
    public void recycle() {
        servlet = null;
        servletName = null;
        redirectPath = null;
    }
}
```

## 再次启动
现在ServletContext有了，再次启动，不再报错了。  
```java
::: Using Embedded Netty Servlet Container (version:)  :::      ＼(^O^)／      Spring-Boot 1.5.2.RELEASE
2017-08-25 22:08:33.019  INFO 17565 --- [           main] io.gitlab.leibnizhu.sbnetty.TestWebApp   : Starting TestWebApp on XPS13 with PID 17565
………………
………………
………………
2017-08-25 22:08:35.760  INFO 17565 --- [           main] io.gitlab.leibnizhu.sbnetty.TestWebApp   : Started TestWebApp in 3.383 seconds (JVM running for 4.012)
2017-08-25 22:08:35.761  INFO 17565 --- [       Thread-2] ationConfigEmbeddedWebApplicationContext : Closing org.springframework.boot.context.embedded.AnnotationConfigEmbeddedWebApplicationContext@4a07d605: startup date [Fri Aug 25 22:08:33 CST 2017]; root of context hierarchy
2017-08-25 22:08:35.763  INFO 17565 --- [       Thread-2] o.s.j.e.a.AnnotationMBeanExporter        : Unregistering JMX-exposed beans on shutdown
Disconnected from the target VM, address: '127.0.0.1:46101', transport: 'socket'
```
可是…………好像有点不对劲……  
启动之后过一会儿就自动关了。  
原因很简单，在```EmbeddedNettyFactory```类里面，我们还没返回真正的```EmbeddedServletContainer```实现类，而只是返回null，所以Spring没有Servlet容器可用，也就只能关闭啦。  
我们将在下一篇文章里讨论如何实现```EmbeddedServletContainer```——与netty结合最紧密的地方。
