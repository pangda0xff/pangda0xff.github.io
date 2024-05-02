Spring中websocket相关的Bean有一个专门的Scope——`websocket`，因此这在这些Bean当中是无法注入Scope为`request`的各种Bean的。这也挺正常的，一个websocket可能会持续很长时间，request的各种Bean仅仅在握手的时候有用，一直不释放也不是个办法。

STOMP中用户身份认证主要来自于握手的Http请求，具体来说来源于方法`DefaultHandshakeHandler#getUser()`。各种Web的机制在这里也基本都适用。但当客户端请求连接各种TOPIC，特别是各种动态的TOPIC时，鉴权的问题就有点不一样了。Spring文档中有这一句：

> When a WebSocket handshake is made and a new WebSocket session is created, Spring’s WebSocket support automatically propagates the java.security.Principal from the HTTP request to the WebSocket session.

也就是说request中的Principal会自动转到websocket中去，很好。然后我们可以利用`ChannelInterceptor`来拦截订阅请求。但这时候发现，Principal居然是`null`：

```java
@Override
public Message<?> preSend(Message<?> message, MessageChannel channel) {
    StompHeaderAccessor headerAccessor = StompHeaderAccessor.wrap(message);
    if (StompCommand.SUBSCRIBE.equals(headerAccessor.getCommand())) {
        Principal principal = headerAccessor.getUser();
        // principal is null here
    }
    return message;
}
```

原因倒是很简单，SpringBoot的starter少了一个依赖：

```groovy
compile 'org.springframework.security:spring-security-messaging'
```

缺少这个库不会引发任何异常也没有任何Log，但是会使得上面的代码拿不到Principal。可以说各种AutoConfiguration还是挺容易留下这种坑的。SpringBoot的各种Starter用来很爽，就是很容易让人知其然不知其所以然，太多的细节被隐藏了，如果程序工作得好倒还OK，一旦出现问题，连从哪里开始排查都不知道。像上面的情况，引入的相关的Starter，但是需要的特性还要格外的依赖，文档中又没有说明，是最容易踩到的大坑。