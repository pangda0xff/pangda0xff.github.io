# 极其弱逼的网络

网络的承载能力极其低下，非常容易就能触碰到上限。新服务启动分配不到IP是常有的事情，新版本好一点。经常是同一主机上的服务能ping通，不同主机上的服务ping不通，重启网络服务之后恢复正常。LoadBalance性能低到不能用，流量稍高就无法连接，LB本身都会进入僵死的状态。

# 极其傻逼的调度

非常喜欢于把同一个服务的多个实例放到同一台机器上，本来是为了高可用才启动了多个实例，结果主机一宕机还是被一锅端了。完全无视机器的负载情况，拼命把服务朝第一台机器上调度，即使前面的机器已经要爆掉了，后面的机器还非常非常空。设置了资源保留也并没有什么卵用，很多时候不得不用标签绑死在特定机器上，完全不像自动化运维。

# 极其傻逼的FailOver

一台机器已经被标记为DISCONNECTED，上面的服务依然没有进行任何转移，就这么一直挂着。如果无法在一台机器上启动服务，不会换一台机器重试，会在这台机器上试到地老天荒。哪怕这台服务器已经只剩一点点内存了。