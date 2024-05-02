## 问题

因为想在多个应用之间共享用户的登录态，因此实现了自己的`SessionDAO`，使用Kryo把`SimpleSession`序列化然后放到redis之中去，同时也使用了`shiro.userNativeSessionManager: true`来使用shiro自己的存储。然而之后一直出现丢失更新的问题，例如

```java
Session session = SecurityUtils.getSubject().getSession();
User user = (User) session.getAttribute(MembershipConst.SessionKey.USER);
user.setName("newName");  // 名称没有更新
```

## 分析

DEBUG之后发现，从Subject中取到的Session并不是我们在SessionDAO中创建的SimpleSession，而是`DelegatingSubject$StoppingAwareProxiedSession`,这是一个代理类，本身并不做任何事情,而是通过`DelegatingSession`调用真正的方法。而DelegatingSession实则也并没有真正的调用SimpleSession，而是调用的SessionManager中的方法：

```
/**
* @see Session#setAttribute(Object key, Object value)
*/
public void setAttribute(Object attributeKey, Object value) throws InvalidSessionException {
    if (value == null) {
        removeAttribute(attributeKey);
    } else {
        sessionManager.setAttribute(this.key, attributeKey, value);
    }
}
```

**而默认的`DefaultSessionManager`在进行任何写操作之前总是会先通过SessionDAO读一次**,如setAttribute方法

```java
public void setAttribute(SessionKey sessionKey, Object attributeKey, Object value) throws InvalidSessionException {
    if (value == null) {
        removeAttribute(sessionKey, attributeKey);
    } else {
        Session s = lookupRequiredSession(sessionKey);
        s.setAttribute(attributeKey, value);
        onChange(s);
    }
}
```

这就是了，实际上我们并未显式的将Session写回redis，而是更新lastAccessTime的时候一并写回去的，而更新访问时间的时候调用了`touch()`方法，SessionManager又通过SessionDAO读取了一次，**重新读取了redis然后反序列化出一个新的Session**，原来Session的各种改动自然也就丢失了。

## 解决

首先是在SessionDAO上加上缓存，一来避免频繁的redis读取，二来避免出现每次读取返回一个新Session的问题。然后在我们的场景中并不需要最后访问时间，因此重写了`ShiroFilterFactoryBean`，不在更新最后访问时间，当Session需要更新的时候，直接调用SessionDAO写回redis，避免SessionManager做二传手。

当然这不是完美的解决方案，并发场景下依然会有更新问题。调式中可以看出Shiro通过SessionDAO进行的读写操作非常频繁，显然在设计时并未将它当作一个涉及外部IO的类。因此将Session放在redis实则不是一个好注意，应该考虑其它的机制。