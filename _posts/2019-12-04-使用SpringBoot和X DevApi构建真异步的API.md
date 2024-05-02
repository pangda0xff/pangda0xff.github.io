reactor正在吞噬世界，唯独Java这边就好像什么也没发生一样。仍然有很多Javaer对异步的理解停留在“发起一个http请求然后等服务回调我”，或者“把IO阻塞的操作放到另外一个线程中去”。不仅如此，在Java及其相关技术的roadmap上异步也从来不是一个显要的话题。当然这也不奇怪，首先异步带来的性能提升实质上比较有限，模型却格外复杂，除非业务规模真的上去了否则没有性价比。再者大多数CRUD的性能瓶颈其实在数据库上，一般来说先跪的都是数据库服务器，应用服务器从始至终都是闲得慌的状态。最后相比Java目前关注的高可用一致性等等，性能反而是最好解决的问题——加机器就是了嘛。

当然异步毕竟是趋势，当异步的成本越来越低的时候，再不行动Java的江山也有坐不住的一天。所以Java 9引入了Flow，Spring 5引入了WebFlux。不过和各种纤程相比始终感觉想原始人闯进了21世纪。异步的世界里，我觉得必须要解决的三个问题：

- 统一的异步模型


这大概是Java目前最头疼的地方了。C#有Task，Javascript有Promise，CompletableFuture出来得太晚了，于是Guava，Netty，Apache HttpClient都有自己的Future，再加上RxJava，Reactor，根本无法做到一致的编程体验。要想做到开箱即用困难重重。

- 普遍的异步API


异步是个to be or not to be的问题，要么全异步，要么全同步，不存在渐进式的方案，这就要求所有涉及到IO的API都要异步化。Java目前这方面的进度实在是一言难尽，特别是Jdbc这一块迟迟不见动静，相比之下一堆nosql还走在了前面。

- 统一的线程池


这可能是最容易被忽视的一点。异步的代码依然需要在线程中执行，如果IO结束后的线程和发起IO调用的线程不在一个线程池中，那么仍然可能存在线程的上下文切换，最后的好处可能仅仅是更少的线程数量所节约出来的一点点栈空间。统一的线程池意味着所有涉及到组件都必须统一线程方案，C#的方案是用一个全局的线程池，而将线程调度的细节隐藏起来，Javascript说我只有一个线程，我从来不担心这个问题——当然也就没有真正的并发了。Java在线程池方面优势很大，但是如何将线程池和纤程很好的结合起来目前还没有方案。

Java在异步方面的体验实在不够好，不过放宽到JVM上来说，Kotlin的coroutine倒是让人眼前一亮，最大的亮点是适配了各种异步方案，不仅有纤程，还有统一的语义。Spring也提供了官方的支持，当然Kotlin本身有一定的学习成本，不过相比scala简直不要好太多。

数据库访问方面，也出现过几个异步访问的项目然而都难逃一死，也难怪，Java的同步生态实在太强大带不动带不动。惊喜加意外的是mysql的X DevApi提供了异步访问的接口。当然X DevApi是什么怎么开启就完全是另外一个故事了。我们需要知道的就是这是一个mysql官方的项目，有oracle爸爸撑腰，起码品质和维护还是有保障的。需要注意的是X Protocol依赖于protobufs.

具体的代码参见[Github](https://github.com/Narcissu5/spring-async)，重复进行一个简单的调用，打印一下各个步骤的线程

```
reactor-http-nio-2
Message listeners dispatching thread
Message listeners dispatching thread
Message listeners dispatching thread
reactor-http-nio-2
reactor-http-nio-2
reactor-http-nio-2
reactor-http-nio-2
reactor-http-nio-2
```

可以看出回调的执行线程可能是当前线程也可能是别的线程，suspendCoroutine的备注也是这样描述的。从行为来看应该是原线程的忙就会调度到别的线程，具体的还需要研究。

虽然距离C#的体验还是差着一截，不过终于也像那么回事儿了。