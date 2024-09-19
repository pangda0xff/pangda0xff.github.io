---
layout: post
title:  "生活在Kubernetes中的Springboot"
date:   2024-09-17 16:00:00 +0800
tags: [kubernetes,springboot]
---

Springboot和Kubernetes中的很多功能都是重叠的，SpringCloud重合的就更多了。不过我还是希望尽可能采用微服务及服务网格这套思路，应用层做轻，SpringCloud就不用了，重合的部分也尽可能用Kubernetes的功能。

## 配置中心

Kubernetes本身提供了对配置中心的支持，不需要再使用Apollo之类的工具。

### 使用ConfigMap来指定Profile

期望应用部署时不用分别设置当前使用的profile，相同集群相同名称空间下的应用自动使用相同的profile。这一功能可以通过ConfigMap来实现。

首先创建一个ConfigMap用存储当前的profile：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: spring-profile
  namespace: default
data:
  SPRING_PROFILES_ACTIVE: prod
```

在`Deployment`部署文件的`containers`节点下增加如下配置：

```yaml
      containers:
        - name: your-app-name
          envFrom:
          - configMapRef:
              name: spring-profile
```

这样配置将拉取spring-profile中的所有配置，并将其作为当前容器的环境变量。Springboot会识别前面设置的`SPRING_PROFILES_ACTIVE`变量并将其作为profile。另外也可以不使用整个Map,Kubernetes提供了[相当灵活的配置方式](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/)。


### 使用Secret来存储数据库密码

数据库密码写在配置文件中是不安全的，配置中心中也不一定安全。Kubernetes提供了专门的Secret来存储密码，存储在Secret的信息都会加密。

可以用和ConfigMap相似的方法创建Secret,或者如果担心文件会泄漏，也可以直接使用命令行：

```sh
kubectl create secret generic mysql-user-psd --from-literal=mysql-user=username --from-literal=mysql-password='password'
```

在`Deployment`部署文件的`containers`节点下增加类似的一行：

```yaml
      containers:
        - name: your-app-name
          envFrom:
          - secretRef:
              name: mysql-user-psd
```

同时配置文件需要做响应的修改，不再直接写明用户名密码等敏感信息，而是使用环境变量：

```yaml
spring:
    datasource:
        url: jdbc:mysql://host.minikube.internal:3306/yourdatabase?useUnicode=true&characterEncoding=utf-8&allowMultiQueries=true
        username: ${mysql-user}
        password: ${mysql-password}
```

## 服务注册与发现

Kubernetes中通过**服务**在集群内部暴露一组服务，首先创建一个服务对象：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: order-system
  labels:
    name: order-system
spec:
  type: NodePort
  selector:
    app: order-system
  ports:
    - port: 80  # 服务暴露端口
      targetPort: 8080 # 目标服务的端口
```

服务创建后Kubernetes会为其分配一个IP地址，通过该地址即可访问服务：

```
docker@minikube:~$ curl http://10.105.200.157/actuator/health
{"status":"DOWN","groups":["liveness","readiness"]}
```

那么如果获得上面这个IP地址呢？有两种方法：

**通过环境变量**

当一个Pod启动，Kubernetes会为每个运行中的服务注入两个环境变量：`{SVCNAME}_SERVICE_HOST`和`{SVCNAME}_SERVICE_PORT`，其中服务名会全部转为大写，并且使用`_`替换`-`。但是注意Pod启动之后出现的服务不会被注入进来。

**通过DNS**

Kubernetes并不原生支持DNS的方式，不过一般集群都会启用DNS插件(如[CoreDNS](https://coredns.io/))，minikube中DNS插件也是默认启用的，所以可以直接使用域名访问服务：

```
root@order-system-7ccc9d54b6-fp9nz:~# curl http://order-system.default/actuator/health
{"status":"DOWN","groups":["liveness","readiness"]}
```

如果是同一名称空间，则域名中表示名称空间的`default`也可以省略。

## 健康检查

Kubernetes会对容易进行检查：

- 存活（Liveness）：如果Pod频繁在存活检查中失败，Kubernetes会尝试重启Pod
- 就绪（Readiness）：Pod启动时检查，在就绪之前Kubernetes不会为其分配网络流量

SpringBoot内置了对这两种检查的支持，需要引入actuator的依赖，当然通常都会依赖这个：

```groovy
dependencies {
	implementation 'org.springframework.boot:spring-boot-starter-validation'
	implementation 'org.springframework.boot:spring-boot-starter-web'
	implementation 'org.springframework.boot:spring-boot-starter-actuator'
  // ... 其它依赖
}
```

然后在配置中打开支持：

```yaml
management:
  endpoint:
    health:
      probes:
        enabled: true
  health:
    livenessstate:
      enabled: true
    readinessstate:
      enabled: true
  endpoints:
    web:
      exposure:
        include:
          - health
```

## 对外网关

Kubernetes本身不提供Zuul这样的网关，需要通过其它技术如Istio来支持。
