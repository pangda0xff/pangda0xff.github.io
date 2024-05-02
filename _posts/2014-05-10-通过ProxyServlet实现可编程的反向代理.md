说道反向代理，可能首先想到的就是nginx。不过在我们的需求中，对于转发过程有更多需求：

*   需要操作session，根据session的取值决定转发行为
*   需要修改Http报文，增加Header或是QueryString

第一点决定了我们的实现必定是基于Servlet的。jetty提供的ProxyServlet就可以满足我们的要求，ProxyServlet直接继承自HttpServlet，采用异步的方式调用内部服务器，因材效率上不会有什么问题，并且各种可重载的函数也提供了比较强大的定制机制。

首先确保当前Servlet版本到达3.1，这个版本才能提供ProxyServlet所要求的异步功能。jetty9搭载的Servlet版本就是3.1,构建的时候引用它就好。其次是在Servlet的配置中打开异步功能：

```xml
<servlet>
    <servlet-name>proxyservlet</servlet-name>
    <servlet-class>net.narcissu5.SimpleProxyServlet</servlet-class>
    <load-on-startup>1</load-on-startup>
    <async-supported>true</async-supported>
</servlet>
```

然后是实现我们的类，从ProxyServlet继承：

```java
package net.narcissu5;

import org.eclipse.jetty.client.api.Request; 
import org.eclipse.jetty.proxy.ProxyServlet; 
import javax.servlet.http.HttpServletRequest; 
import java.net.URI; 
import java.net.URISyntaxException; 

public class SimpleProxyServlet extends ProxyServlet {
    @Override protected Request addViaHeader(Request proxyRequest) {
        proxyRequest.header("Basic","123456"); return super.addViaHeader(proxyRequest);
    }

    @Override protected URI rewriteURI(HttpServletRequest request) {
        URI uri = null; try{ if(request.getQueryString() == null){
                uri = new URI("http://192.168.235.129:8080");
            } else {
                uri = new URI("http://192.168.235.129:8080?" + request.getQueryString());
            }
        } catch (URISyntaxException e){

        } return uri;
    }
}
```

当然rewriteURI中可以有更复杂的逻辑，既然都已经进入到了这里，转发规则就不是什么问题了。最后注意下jetty class load的问题。jetty 9 使用JavaEE规范，先加载用户类再加载服务器类，因此引用的jetty-proxy.jar如果设置为provide（也就是不打包进入war）的话会出现ClassNotFound异常。通常的做法（感谢StackOverflow）是设置为编译时依赖：

```xml
<dependency>
    <groupId>org.eclipse.jetty</groupId>
    <artifactId>jetty-proxy</artifactId>
    <version>9.1.5.v20140505</version>
</dependency>
```

这样它就普通的Servlet没什么区别了，虽然war最后会大那么一点点~~