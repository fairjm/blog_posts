---
title: "异步请求中jetty处理ServletRequestListener的坑"
date: 2017/11/19 17:51:00
---
标题起得比较诡异，其实并不是坑，而是jetty似乎压根就没做对异步request的`ServletRequestListener`的特殊处理，如果文中有错误欢迎提出，可能自己有所疏漏了。

之前遇到了一个bug，在Listener中重写`requestDestroyed`清理资源后，这些资源在异步任务中就不可用了。  
这与预期不符，直觉上request应该在任务完成之后才触发`requestDestroyed`，而不应该是开始异步操作返回后就触发。  
正确的触发时机应该是异步任务完成之后。  
后来查阅了下，发现`servlet 3`的规范中只是增加了异步servlet和异步filter的支持，listener如何处理没有定义，也就是各个容器的实现可能会差异（我们自己使用的是jetty）。  
但讲道理的话，合理的实现应该判断当前request是否处于异步处理中，如果是的话将销毁的触发的时机放置在`AsyncContext`完成之后。

自己试了下使用两个容器：`Tomcat 8.5.15`和`Jetty 9.4.7.v20170914`。

测试代码比较简单，为了去除其他框架的影响使用纯的servlet api。
servlet：   
```java
@WebServlet(asyncSupported = true, urlPatterns = { "/test" })
public class TestServlet extends HttpServlet {
    private static final long serialVersionUID = 7395865716615939512L;

    private ExecutorService pool = Executors.newCachedThreadPool();

    @Override
    public void destroy() {
        pool.shutdown();
    }

    @Override
    protected void doGet(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException {
        System.out.println(LocalDateTime.now() + " get start " + request);
        AsyncContext context = request.startAsync();
        context.addListener(new AsyncListener() {
            @Override
            public void onTimeout(AsyncEvent event) throws IOException {
                System.out.println(LocalDateTime.now() + " async onTimeout " + event.getSuppliedRequest());
            }

            @Override
            public void onStartAsync(AsyncEvent event) throws IOException {
                System.out.println(LocalDateTime.now() + " async onStartAsync " + event.getSuppliedRequest());
            }

            @Override
            public void onError(AsyncEvent event) throws IOException {
                System.out.println(LocalDateTime.now() + " async onError " + event.getSuppliedRequest());
            }

            @Override
            public void onComplete(AsyncEvent event) throws IOException {
                System.out.println(LocalDateTime.now() + " async onComplete " + event.getSuppliedRequest());
            }
        }, request, response);
        pool.execute(() -&gt; {
            try {
                Thread.sleep(TimeUnit.SECONDS.toMillis(5));
                System.out.println(LocalDateTime.now() + " job done");
                context.complete();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
        System.out.println(LocalDateTime.now() + " get return");
    }

}
```
listener:  
```java
@WebListener
public class TestListener implements ServletRequestListener {

    @Override
    public void requestDestroyed(ServletRequestEvent event) {
        System.out.println(LocalDateTime.now() + " TestListener requestDestroyed " + event.getServletRequest());
    }

    @Override
    public void requestInitialized(ServletRequestEvent event) {
        System.out.println(LocalDateTime.now() + " TestListener requestInitialized " + event.getServletRequest());
    }

}
```
使用tomcat返回的结果:    
```
2017-11-19T16:42:04.256 TestListener requestInitialized org.apache.catalina.connector.RequestFacade@7c2df671
2017-11-19T16:42:04.257 get start org.apache.catalina.connector.RequestFacade@7c2df671
2017-11-19T16:42:04.261 get return
2017-11-19T16:42:09.261 job done
2017-11-19T16:42:09.261 async onComplete org.apache.catalina.connector.RequestFacade@7c2df671
2017-11-19T16:42:09.261 TestListener requestDestroyed org.apache.catalina.connector.RequestFacade@7c2df671
```
使用jetty返回的结果：    
```
2017-11-19T16:46:35.231 TestListener requestInitialized (GET //localhost:8080/test)@1124787543 org.eclipse.jetty.server.Request@430ae557
2017-11-19T16:46:35.232 get start (GET //localhost:8080/test)@1124787543 org.eclipse.jetty.server.Request@430ae557
2017-11-19T16:46:35.234 get return
2017-11-19T16:46:35.235 TestListener requestDestroyed [GET //localhost:8080/test]@1124787543 org.eclipse.jetty.server.Request@430ae557
2017-11-19T16:46:40.235 job done
2017-11-19T16:46:58.019 async onComplete [GET //localhost:8080/test]@1124787543 org.eclipse.jetty.server.Request@430ae557
```
jetty果然没有讲道理... ...  

先看下讲道理的tomcat，果不其然他做了特殊判断。  
在`StandardHostValve`的`invoke`方法中：  
```java
if (!request.isAsync() &amp;&amp; !asyncAtStart) {
    context.fireRequestDestroyEvent(request.getRequest());
}
```
并且销毁的时机和预想的一样是在`AsyncContext`完成之后：  

![](/images/1244488-20171119174205312-2127854720.png)  
```java
    public void fireOnComplete() {
        List&lt;AsyncListenerWrapper&gt; listenersCopy = new ArrayList&lt;&gt;();
        listenersCopy.addAll(listeners);

        ClassLoader oldCL = context.bind(Globals.IS_SECURITY_ENABLED, null);
        try {
            for (AsyncListenerWrapper listener : listenersCopy) {
                try {
                    listener.fireOnComplete(event);
                } catch (Throwable t) {
                    ExceptionUtils.handleThrowable(t);
                    log.warn("onComplete() failed for listener of type [" +
                            listener.getClass().getName() + "]", t);
                }
            }
        } finally {
            context.fireRequestDestroyEvent(request.getRequest());
            clearServletRequestResponse();
            context.unbind(Globals.IS_SECURITY_ENABLED, oldCL);
        }
    }
```
而jetty没有处理，在`ContextHandler`的`doHandle`中没有异步请求的判断：  
```java
finally
        {
            // Handle more REALLY SILLY request events!
            if (new_context)
            {
                if (!_servletRequestListeners.isEmpty())
                {
                    final ServletRequestEvent sre = new ServletRequestEvent(_scontext,request);
                    for (int i=_servletRequestListeners.size();i--&gt;0;)
                        _servletRequestListeners.get(i).requestDestroyed(sre);
                }

                if (!_servletRequestAttributeListeners.isEmpty())
                {
                    for (int i=_servletRequestAttributeListeners.size();i--&gt;0;)
                        baseRequest.removeEventListener(_servletRequestAttributeListeners.get(i));
                }
            }
        }
```
new_content用来在第一次调用返回true之后是false，和异步处理无关。  

网上查了些也不清楚为什么jetty没有像tomcat那样实现这个，还是jetty希望用户通过`AsyncListener`来做处理呢。
