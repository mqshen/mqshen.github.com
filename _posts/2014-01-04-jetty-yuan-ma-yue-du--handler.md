---
layout: post
title: "Jetty源码阅读 Handler"
description: ""
category: ""
tags: [Jetty]
---
{% include JB/setup %}

###Server
![Jetty Server]({{ BASE_PATH }}/assets/images/JettyServer.png)

可以配置多个Connector，起多个端口监听请求。  
如jetty-http.xml中

{% highlight xml linenos %}
<Call name="addConnector">
  <Arg>
    <New class="org.eclipse.jetty.server.ServerConnector">
      <Arg name="server"><Ref refid="Server" /></Arg>
      <Arg name="factories">
        <Array type="org.eclipse.jetty.server.ConnectionFactory">
          <Item>
            <New class="org.eclipse.jetty.server.HttpConnectionFactory">
              <Arg name="config"><Ref refid="httpConfig" /></Arg>
            </New>
          </Item>
        </Array>
      </Arg>
      <Set name="host"></Set>
      <Set name="port">8080</Set>
      <Set name="idleTimeout">300000</Set>
    </New>
  </Arg>
</Call>
{% endhighlight %}

Connector负责请求的接受和处理，生成Request交给Server处理。

ThreadPoll负责任务的调度
在jetty.xml中没有指定ThreadPool实例,由下面代码初始化.

{% highlight java linenos %}
public Server(@Name("threadpool") ThreadPool pool) {
    _threadPool=pool!=null?pool:new QueuedThreadPool();
    addBean(_threadPool);
    setServer(this);
}
{% endhighlight %}

再由jetty.xml中的

{% highlight xml linenos %}
<Get name="ThreadPool">
  <Set name="minThreads" type="int"><Property name="threads.min" default="10"/></Set>
  <Set name="maxThreads" type="int"><Property name="threads.max" default="200"/></Set>
  <Set name="idleTimeout" type="int"><Property name="threads.timeout" default="60000"/></Set>
  <Set name="detailedDump">false</Set>
</Get>
{% endhighlight %}

设置具体的参数值.
Server处理请求时调用父类HandlerWrapper的

{% highlight java linenos %}
public void handle(String target, Request baseRequest, HttpServletRequest request, 
        HttpServletResponse response) throws IOException, ServletException {
    if (_handler!=null && isStarted()) {
        _handler.handle(target,baseRequest, request, response);
    }
}
{% endhighlight %}
这里个_handler是由jetty.xml指定的。

{% highlight xml linenos %}
<Set name="handler">
  <New id="Handlers" class="org.eclipse.jetty.server.handler.HandlerCollection">
    <Set name="handlers">
     <Array type="org.eclipse.jetty.server.Handler">
       <Item>
         <New id="Contexts" class="org.eclipse.jetty.server.handler.ContextHandlerCollection"/>
       </Item>
       <Item>
         <New id="DefaultHandler" class="org.eclipse.jetty.server.handler.DefaultHandler"/>
       </Item>
     </Array>
    </Set>
  </New>
</Set>
{% endhighlight %}

然后获取最佳匹配的ContextHandler
{% highlight java linenos %}
if (target.startsWith("/")) {
    int limit = target.length()-1;
    
    while (limit>=0) {
        // Get best match
        ContextHandler[] contexts = _contexts.getBest(target,1,limit);
        if (contexts==null)
            break;
        int l=contexts[0].getContextPath().length();
        if (l==1 || target.length()==l || target.charAt(l)=='/') {
            for (ContextHandler handler : contexts) {
                handler.handle(target,baseRequest, request, response);
                if (baseRequest.isHandled())
                    return;
            }
        }
        limit=l-2;
    }
}
{% endhighlight %}

请求交给不同应用的ContextHandler,而ContextHandler有两种身份：ServletContext和ScopedHandler。    
ScopedHandler组成了Jetty的Handler链结构，进行特殊的处理逻辑。    

###ScopedHandler
官方解释为
{% highlight java %}
ScopedHandler. A ScopedHandler is a HandlerWrapper where the wrapped handlers each define a scope.
When handle(String, Request, HttpServletRequest, HttpServletResponse) is called on the first ScopedHandler in a chain of HandlerWrappers, the doScope(String, Request, HttpServletRequest, HttpServletResponse) method is called on all contained ScopedHandlers, before the doHandle(String, Request, HttpServletRequest, HttpServletResponse) method is called on all contained handlers.

For example if Scoped handlers A, B & C were chained together, then the calling order would be:

 A.handle(...)
   A.doScope(...)
     B.doScope(...)
       C.doScope(...)
         A.doHandle(...)
           B.doHandle(...)
              C.doHandle(...)
 
If non scoped handler X was in the chained A, B, X & C, then the calling order would be:

 A.handle(...)
   A.doScope(...)
     B.doScope(...)
       C.doScope(...)
         A.doHandle(...)
           B.doHandle(...)
             X.handle(...)
               C.handle(...)
                 C.doHandle(...)
{% endhighlight %}

{% highlight java linenos %}
@Override
public final void handle(String target, Request baseRequest, HttpServletRequest request, 
        HttpServletResponse response) throws IOException, ServletException {
    if (isStarted()) {
        if (_outerScope==null)
            doScope(target,baseRequest,request, response);
        else
            doHandle(target,baseRequest,request, response);
    }
}
{% endhighlight %}

+ ScopedHandler从字面上可以理解为：“有上下文环境限定的处理器”
    
+ ScopedHandler定义了两个方法分别为：doScope()，doHandle()；doScope()可理解为设置并维护处理器的上下文环境，doHandle()可理解为在该上下文环境下执行处理操作。
    
+ ScopedHandler首先是一个HanderWrapped，其次拥有两个属性：_nextScope、_outScope，使得其可以定义一个Handle链A->B->C，对于B而言，_nextScope即C，_outScope即A。这时你要问了，既然是HandlerWrapped为什么还需要这俩属性呢，都认为inner Hander为_nextScope不就行了。框架这样做既增加了灵活性，同时inner Hander可不一定是ScopedHandler。
    
+ 通过上面的功能描述你是否还觉得是个责任链模式呢，确实有责任链的影子，如果是纯粹的责任链，我想作者绝对不会起名：ScopedHandler了。而这种设计的应用场景如下：
        

    + Jetty 中的ScopedHandler 链结构如下：Context->session->servlet
    + 请求进入ContextHandler时将设置并维护上下文环境，离开时将恢复上下文环境。这里主要维护的上下文环境如下：

{% highlight java linenos %}
// reset the context and servlet path.
baseRequest.setContext(old_context);
__context.set(old_context);
baseRequest.setContextPath(old_context_path);
baseRequest.setServletPath(old_servlet_path);
baseRequest.setPathInfo(old_path_info);
{% endhighlight %}

请求的这些属性不是原先是空的么，怎么还要设置并维护最后还有恢复呢？这里主要考虑到了跨app的请求，如果请求跨app的话，那么它将携带不正确的path信息运行到新的ContextHandler中，正如下面的调用：

{% highlight java linenos %}
request.getServletContext().getContext("app2")
    .getRequestDispatcher("new.htm").forward(request,response)
{% endhighlight %}

+ &nbsp;
    
    + 同样的道理，所以SessionHandler也需要doScope，跨app自然需要替换sessionManager。

    + 所有的上下文都维护好之后，才可以在各自的上下文方位内handle.

ScopedHandler有三个子类，也是Handler链中的三个主要成员，下面将逐一介绍。

####ContextHandler
Jetty的主要职责:接受并解析请求，管理ServletContext。因此ContextHandler在Jetty中的地位还是很高的。
1.  ContextHandler模型

![Jetty ContextHandler]({{ BASE_PATH }}/assets/images/JettyContextHandler.png)
    
框架把内部类Context作为ServletContext暴露给应用，具体的实现封装在了ContextHandler中；每一个层级需要的属性和方法也很明了，ContextHandler层级主要提供了ServletContext作为Context容器的一些功能性的操作；ServletContextHandler层级主要提供了servlet的管理；最后的WebAppContext层级主要提供了作为一个Web工程的上下文环境的描述和资源的定位加载。

![Jetty ContextHandler]({{ BASE_PATH }}/assets/images/JettyWebAppContext.png)

preConfigure主要负责：初始化Configuration->创建WebAppClassLoader->解压WAR包->解析webdefault.xml->扫描meta的jar包加入classpath中。不妨看下Configuration的顺序：

{% highlight java linenos %}
public static final String[] DEFAULT_CONFIGURATION_CLASSES = {
    "org.eclipse.jetty.webapp.WebInfConfiguration",
    "org.eclipse.jetty.webapp.WebXmlConfiguration",
    "org.eclipse.jetty.webapp.MetaInfConfiguration",
    "org.eclipse.jetty.webapp.FragmentConfiguration",
    "org.eclipse.jetty.webapp.JettyWebXmlConfiguration"
} ;
{% endhighlight %}

####SessionHandler
该handler主要职责是为请求设置当前app的sessionManager，用于管理session。由于请求是可以跨app的，因此对于该handler的doScope操作应类似于ContextHandler具有维护上下文的功能。
{% highlight java linenos %}
old_session_manager = baseRequest.getSessionManager();
old_session = baseRequest.getSession(false);

if (old_session_manager != _sessionManager) {
    // new session context
    baseRequest.setSessionManager(_sessionManager);
    baseRequest.setSession(null);
    checkRequestedSessionId(baseRequest,request);
}
......
finally {
    if (access != null)
        _sessionManager.complete(access);

    HttpSession session = baseRequest.getSession(false);
    if (session != null && old_session == null && session != access)
        _sessionManager.complete(session);

    if (old_session_manager != null && old_session_manager != _sessionManager) {
        baseRequest.setSessionManager(old_session_manager);
        baseRequest.setSession(old_session);
    }
}
{% endhighlight %}

####ServletHandler

#####doScope过程

+ 主要职责有三：
    + 根据target匹配出servlet
    + 根据匹配出的url路径分割出servlet_path和path_info，
    + 处理完全之后要恢复servlet_path和path_info。

这里要注意区分url中的uri、target、context_path、servlet_path和path_info的区别。

#####doHandler过程
+ 主要职责有二
    + 根据target构造FilterChain并缓存起来
    + 执行FilterChain

注意这里的FilterChain可不仅仅是filter，也包括了最后的servlet。

{% highlight java linenos %}
@Override
public void doHandle(String target, Request baseRequest,HttpServletRequest request, 
    HttpServletResponse response) throws IOException, ServletException {
    DispatcherType type = baseRequest.getDispatcherType();

    ServletHolder servlet_holder=(ServletHolder) baseRequest.getUserIdentityScope();
    FilterChain chain=null;

    // find the servlet
    if (target.startsWith("/")) {
        if (servlet_holder!=null && _filterMappings!=null && _filterMappings.length>0)
            chain=getFilterChain(baseRequest, target, servlet_holder);
    }
    else {
        if (servlet_holder!=null) {
            if (_filterMappings!=null && _filterMappings.length>0) {
                chain=getFilterChain(baseRequest, null,servlet_holder);
            }
        }
    }

    LOG.debug("chain={}",chain);
    Throwable th=null;
    try {
        if (servlet_holder==null) {
            if (getHandler()==null)
                notFound(request, response);
            else
                nextHandle(target,baseRequest,request,response);
        }
        else {
            // unwrap any tunnelling of base Servlet request/responses
            ServletRequest req = request;
            if (req instanceof ServletRequestHttpWrapper)
                req = ((ServletRequestHttpWrapper)req).getRequest();
            ServletResponse res = response;
            if (res instanceof ServletResponseHttpWrapper)
                res = ((ServletResponseHttpWrapper)res).getResponse();

            // Do the filter/handling thang
            if (chain!=null)
                chain.doFilter(req, res);
{% endhighlight %}

### 参考资料
>   [Jetty源码学习](http://my.oschina.net/tryUcatchUfinallyU/blog/112972)    
>   [Jetty apidocs](http://download.eclipse.org/jetty/stable-9/apidocs/)
