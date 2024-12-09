---
title: Cloud
date: 2024-03-16 16:00:10
tags:
---

## CAP原则

CAP 原则又称CAP定理，指的是在一个分布式系统中，存在Consistency（一致性）、Availability（可用性）、Partition tolerance（分区容错性），三者不可同时保证，最多只能保证其中的两者。

一致性（C）：在分布式系统中的所有数据备份，在同一时刻都是同样的值（所有的节点无论何时访问都能拿到最新的值）

可用性（A）：系统中非故障节点收到的每个请求都必须得到响应（比如我们之前使用的服务降级和熔断，其实就是一种维持可用性的措施，虽然服务返回的是没有什么意义的数据，但是不至于用户的请求会被服务器忽略）

分区容错性（P）：一个分布式系统里面，节点之间组成的网络本来应该是连通的，然而可能因为一些故障（比如网络丢包等，这是很难避免的），使得有些节点之间不连通了，整个网络就分成了几块区域，数据就散布在了这些不连通的区域中（这样就可能出现某些被分区节点存放的数据访问失败，我们需要来容忍这些不可靠的情况）

### AC 可用性+一致性
要同时保证可用性和一致性，代表着某个节点数据更新之后，需要立即将结果通知给其他节点，并且要尽可能的快，这样才能及时响应保证可用性，这就对网络的稳定性要求非常高，但是实际情况下，网络很容易出现丢包等情况，并不是一个可靠的传输，如果需要避免这种问题，就只能将节点全部放在一起，但是这显然违背了分布式系统的概念，所以对于我们的分布式系统来说，很难接受。

### CP 一致性+分区容错性
为了保证一致性，那么就得将某个节点的最新数据发送给其他节点，并且需要等到所有节点都得到数据才能进行响应，同时有了分区容错性，那么代表我们可以容忍网络的不可靠问题，所以就算网络出现卡顿，那么也必须等待所有节点完成数据同步，才能进行响应，因此就会导致服务在一段时间内完全失效，所以可用性是无法得到保证的。

### AP 可用性+分区容错性
既然CP可能会导致一段时间内服务得不到任何响应，那么要保证可用性，就只能放弃节点之间数据的高度统一，也就是说可以在数据不统一的情况下，进行响应，因此就无法保证一致性了。虽然这样会导致拿不到最新的数据，但是只要数据同步操作在后台继续运行，一定能够在某一时刻完成所有节点数据的同步，那么就能实现**最终一致性**，所以AP实际上是最能接受的一种方案。

## openFeign
### 远程调用含义
`远程调用`和`本地调用`是相对的，那我们先说本地调用更好理解些，本地调用就是同一个 Service 里面的方法 A 调用方法 B。

那远程调用就是不同 Service 之间的方法调用。Service 级的方法调用，我们自己构造请求 URL和请求参数，就可以发起远程调用了。

在服务之间调用的话，我们都是基于 HTTP 协议，一般用到的远程服务框架有 OKHttp3，Netty, HttpURLConnection 等。其调用流程如下：

![](https://cdn.nlark.com/yuque/0/2024/png/34596451/1732590614281-36e2fb2c-c877-4591-8e4d-dadcf48d8288.png)

但是这种虚线方框中的构造请求的过程是很**繁琐**的，有没有更**简便**的方式呢？

**Feign** 就是来简化我们发起远程调用的代码的，那简化到什么程度呢？**简化成就像调用本地方法那样简单。

### OpenFeign工作流程
先看下 OpenFeign 的核心流程图：

![](https://cdn.nlark.com/yuque/0/2024/png/34596451/1732590758302-e6ab47e2-f3da-4a75-ba6b-323e79c1dfbc.png)

+ 1、在 Spring 项目启动阶段，服务 A 的OpenFeign 框架会发起一个主动的扫包流程。
+ 2、从指定的目录下扫描并加载所有被 @FeignClient 注解修饰的接口，然后将这些接口转换成 Bean，统一交给 Spring 来管理。
+ 3、根据这些接口会经过 MVC Contract 协议解析，将方法上的注解都解析出来，放到 MethodMetadata 元数据中。
+ 4、基于上面加载的每一个 FeignClient 接口，会生成一个动态代理对象，指向了一个包含对应方法的 MethodHandler 的 HashMap。MethodHandler 对元数据有引用关系。生成的动态代理对象会被添加到 Spring 容器中，并注入到对应的服务里。
+ 5、服务 A 调用接口，准备发起远程调用。
+ 6、从动态代理对象 Proxy 中找到一个 MethodHandler 实例，生成 Request，包含有服务的请求 URL（不包含服务的 IP）。
+ 7、经过负载均衡算法找到一个服务的 IP 地址，拼接出请求的 URL
+ 8、服务 B 处理服务 A 发起的远程调用请求，执行业务逻辑后，返回响应给服务 A。

**核心思想：**

+ OpenFeign 会扫描带有 @FeignClient 注解的接口，然后为其生成一个动态代理。
+ 动态代理里面包含有接口方法的 MethodHandler，MethodHandler 里面又包含经过 MVC Contract 解析注解后的元数据。
+ 发起请求时，MethodHandler 会生成一个 Request。
+ 负载均衡器 Ribbon 会从服务列表中选取一个 Server，拿到对应的 IP 地址后，拼接成最后的 URL，就可以发起远程服务调用了。

### 使用步骤
利用它实现负载均衡，远程调用

```java
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

接着在启动类添加`@EnableFeignClients`注解：开启远程调用服务。

```java
@SpringBootApplication
@EnableFeignClients
public class BorrowApplication {
    public static void main(String[] args) {
        SpringApplication.run(BorrowApplication.class, args);
    }
}
```

我们可以看到这个 interface 上添加了注解`<font style="color:rgb(44, 62, 80);">@FeignClient</font>`，而且括号里面指定了服务名：book-service。显示声明这个接口用来远程调用 `<font style="color:rgb(44, 62, 80);">book</font>`服务。并且方法上添加了控制层响应地址

```java
@FeignClient(value = "book-service")
public interface BookClient {
    @RequestMapping("/book/{bid}")
    public Book getBookById(@PathVariable("bid") int bid);
}
```

服务层Impl注入接口即可使用

## Alibaba Cloud依赖导入
```java
<dependencyManagement>
    <dependencies>
      <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-dependencies</artifactId>
        <version>2022.0.3</version>
        <type>pom</type>
        <scope>import</scope>
      </dependency>
      <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-alibaba-dependencies</artifactId>
        <version>2022.0.0.0-RC2</version>
        <type>pom</type>
        <scope>import</scope>
      </dependency>
    </dependencies>
  </dependencyManagement>
```

## Nacos
Nacos（Naming Configuration Service）是一款阿里巴巴开源的服务注册与发现、配置管理的组件，相当于是Eureka+Config的组合形态。

**启动命令： **`** ./startup.cmd -m standalone**`

### 服务管理依赖导入
```java
 <dependency>
      <groupId>com.alibaba.cloud</groupId>
      <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
      <version>2022.0.0.0-RC2</version>
    </dependency>
<dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-loadbalancer</artifactId>  // 负载均衡配置
      <version>4.0.3</version>
    </dependency>
```

### 配置管理依赖导入  

```java
 <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-bootstrap</artifactId>
    </dependency>
    <dependency>
      <groupId>com.alibaba.cloud</groupId>
      <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
    </dependency>
```

### yml文件配置
```java
spring:
  application:
    name: borrow-service  // 程序名
  cloud:
    nacos:
      discovery:
        server-addr: localhost:8848	// nacos运行地址
        ephemeral: false	// 是否短暂生效， false为持久化，
        cluster-name: Guangzhou // 集群名字
        weight： 0.2		// 权重大小，越大越优先调用， 默认为1 使用时建议在一个集群底下
    loadbalancer:
      nacos:
        enabled: true	// 开启负载均衡
```

### 配置管理
![](https://cdn.nlark.com/yuque/0/2024/png/34596451/1732512781202-dbd51330-3f89-4bb5-b5ab-614f19301dd0.png)

1. 设置Date ID： 默认为程序名-dev.yml
2. 将需要配置的内容放置下方
3. 删除原程序中yml 的配置内容
4. 导入配置管理依赖
5. 添加bootstrap.yml 添加配置信息

```java
spring:
  application:
    name: user-service
  profiles:
    # 环境与配置文件保持一致
    active: dev
  cloud:
    nacos:
      config:
        # 配置文件的后缀名
        file-extension: yml
        # nacos配置中心服务器地址
        server-addr: localhost:8848
```

程序会实时监听配置类的信息，判断你是否添加新的配置进去。

![](https://cdn.nlark.com/yuque/0/2024/png/34596451/1732513012294-76b6a088-adc1-4a44-b8b1-85d2f18402a3.png)

这里我添加了一个zzh的新值进去，idea程序实时监听

![](https://cdn.nlark.com/yuque/0/2024/png/34596451/1732513054194-c1f7b7c7-64dc-45fc-974b-9da6faf6ffff.png)

但此时你无法读取到配置的改变，如果想实时刷新配置内容并使用需要添加注解实现热更新

![](https://cdn.nlark.com/yuque/0/2024/png/34596451/1732513112573-906217c5-17db-49d1-aef7-5acd35cef806.png)

**<font style="color:rgb(44, 62, 80);">注意：Nacos的配置项优先级高于application.propertite里面的配置。</font>**

### 服务管理  
![](https://cdn.nlark.com/yuque/0/2024/png/34596451/1732512664743-b19eaeea-f1f9-45d0-a84a-418e61a2eada.png)
服务管理分集群使用，如果设置了` ephemeral: false` 临时实例为`false`

**集群分区**通过 `cluster-name: '集群名字'`来实现集群

### 命名空间
通过添加命名空间获取ID

![](https://cdn.nlark.com/yuque/0/2024/png/34596451/1732515676058-66ce7491-3517-4504-a6cb-7a3846dcaaa5.png)

在application.yml添加此id进去

```java
  cloud:
    nacos:
      discovery:
        server-addr: localhost:8848
        cluster-name: Guangzhou
        # 命名空间
        namespace: 31989985-d3f3-473f-bb16-ad36c8624be8
        # 分组
        group: info
```

如果程序之间不在同一个命名空间或者分组下则无法访问，需要两者皆一致

![](https://cdn.nlark.com/yuque/0/2024/png/34596451/1732515580279-0380db69-c029-47ec-8fdc-9d37f6c6667a.png)

### 集群部署
[集群模式部署](https://nacos.io/docs/latest/manual/admin/deployment/deployment-cluster/)

### 工作流程
+ **集群环境**：如果是 Nacos 集群环境，那么拓扑结构是什么样的。
+ **组装请求**：客户端组装注册请求，下一步对 Nacos 服务发起远程调用。
+ **随机节点**：客户端随机选择集群中的一个 Nacos 节点发起注册，实现负载均衡。
+ **路由转发**：Nacos 节点收到注册请求后，看下是不是属于自己的，不是的话，就进行路由转发。
+ **处理请求**：转发给指定的节点后，该节点就会将注册请求中的实例信息解析出来，存到自定义的内存结构中。
+ **最终一致性**：通过 Nacos 自研的 Distro 协议执行`延迟异步任务`，将注册信息同步给集群中的其他节点，保证了数据的最终一致性。
+ **异步重试**：如果注册失败，客户端将会切换 Nacos 节点，再次发起注册请求，保证高可用性。



## GateWay
创建新模块，导入依赖

```java
  <dependency>
      <groupId>com.alibaba.cloud</groupId>
      <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-gateway</artifactId>
    </dependency>

    <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-loadbalancer</artifactId>
    </dependency>
```

注册nacos

```java
spring:
  cloud:
    nacos:  # 注册nacos
      discovery:
        server-addr: localhost:8848
        namespace: 31989985-d3f3-473f-bb16-ad36c8624be8
    gateway:
      # 配置路由，注意這裡是個列表，每一項都很重要
      routes:
        - id: borrow-service # 路由名稱
          uri: lb://borrow-service # 路由地址， lb表示负载均衡到服务器，也可以使用http正常转发
          predicates: # 路由规则 规定什么请求会被路由
            - Path=/borrow/** # 只要访问这个路径下，一律被路由上面指定的服务
        - id: user-service
          uri: lb://user-service
          predicates:
            - Path=/user/**
  application:
    name: gateway-service
```

路由名称是按照当前模块的application.name而定的。

### 过滤器
#### 局部过滤器
<font style="color:rgb(44, 62, 80);">局部过滤器，应用在单个路由或一组路由上的过滤器。标红色表示比较常用的过滤器。</font>

![](https://cdn.nlark.com/yuque/0/2024/png/34596451/1732589561063-f0490508-5fae-4469-92b9-0f286410f3e8.png)

在配置文件中添加filters属性

```java
spring:
  cloud:
    nacos:  # 注册nacos
      discovery:
        server-addr: localhost:8848
        namespace: 31989985-d3f3-473f-bb16-ad36c8624be8
    gateway:
      # 配置路由，注意這裡是個列表，每一項都很重要
      routes:
        - id: borrow-service # 路由名稱
          uri: lb://borrow-service # 路由地址， lb表示负载均衡到服务器，也可以使用http正常转发
          predicates: # 路由规则 规定什么请求会被路由
            - Path=/borrow/** # 只要访问这个路径下，一律被路由上面指定的服务
        - id: user-service
          uri: lb://user-service
          predicates:
            - Path=/user/**
          filters:
            - AddRequestHeader=Test, helloworld # 添加请求头
  application:
    name: gateway-service
```

#### 全局过滤器
<font style="color:rgb(44, 62, 80);">全局过滤器最常见的用法是进行负载均衡，上图的uri:lb。。。就是全局过滤</font>

```java
@Component // 全局过滤
public class TestFilter implements GlobalFilter {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request= exchange.getRequest();
        String token = request.getHeaders().getFirst("token"); // 获取头文件中的token
        if(!StringUtils.isEmpty(token)){ // 判断token是否存在
            if("admin".equals(token)){ // 判断token是否等于admin
                return chain.filter(exchange); // 是就继续执行
            }
        }
        exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED); // 设置状态码
        return exchange.getResponse().setComplete(); // 不往下执行，直接返回
    }
}
```

当然，过滤器肯定是可以存在很多个的，所以我们可以手动指定过滤器之间的顺序：

```java
@Component
public class TestFilter implements GlobalFilter, Ordered {   //实现Ordered接口
  
    @Override
    public int getOrder() {
        return 0;
    }
```

注意Order的值越小优先级越高，并且无论是在配置文件中编写的单个路由过滤器还是全局路由过滤器，都会受到Order值影响（单个路由的过滤器Order值按从上往下的顺序从1开始递增），最终是按照Order值决定哪个过滤器优先执行，当Order值一样时 全局路由过滤器执行 `优于` 单独的路由过滤器执行。

### 设计网关，需要考虑哪些？
如果让你设计一个 API 网关，你会考虑哪些方面？

#### 路由转发
请求先到达 API 网关，然后经过断言，匹配到路由后，由路由将请求转发给真正的业务服务。

#### 注册发现
各个服务实例需要将自己的服务名、IP 地址和 port 注册到注册中心，然后注册中心会存放一份注册表，Gateway 可以从注册中心获取到注册表，然后转发请求时只需要转发到对应的服务名即可。

#### 负载均衡
一个服务可以由多个服务实例组成服务集群，而 Gateway 配置的通常是一个服务名，如 passjava-member 服务，所以需要具备负载均衡功能，将请求分发到不同的服务实例上。

#### 弹力设计
网关还可以把弹力设计中的那些异步、重试、幂等、流控、熔断、监视等都可以实现进去。这样，同样可以像 Service Mesh 那样，让应用服务只关心自己的业务逻辑（或是说数据面上的事）而不是控制逻辑（控制面）。

#### 安全方面
SSL 加密及证书管理、Session 验证、授权、数据校验，以及对请求源进行恶意攻击的防范。错误处理越靠前的位置就是越好，所以，网关可以做到一个全站的接入组件来对后端的服务进行保护。当然，网关还可以做更多更有趣的事情，比如：灰度发布、API聚合、API编排。

##### 灰度发布
网关完全可以做到对相同服务不同版本的实例进行导流，还可以收集相关的数据。这样对于软件质量的提升，甚至产品试错都有非常积极的意义。

##### API 聚合
使用网关可以将多个单独请求聚合成一个请求。在微服务体系的架构中，因为服务变小了，所以一个明显的问题是，客户端可能需要多次请求才能得到所有的数据。这样一来，客户端与后端之间的频繁通信会对应用程序的性能和规模产生非常不利的影响。于是，我们可以让网关来帮客户端请求多个后端的服务（有些场景下完全可以并发请求），然后把后端服务的响应结果拼装起来，回传给客户端（当然，这个过程也可以做成异步的，但这需要客户端的配合）。

##### API 编排
同样在微服务的架构下，要走完一个完整的业务流程，我们需要调用一系列 API，就像一种工作流一样，这个事完全可以通过网页来编排这个业务流程。我们可能通过一个 DSL 来定义和编排不同的 API，也可以通过像 AWS Lambda 服务那样的方式来串联不同的 API。

### 网关设计重点
#### 高性能
在技术设计上，网关不应该也不能成为性能的瓶颈。对于高性能，最好使用高性能的编程语言来实现，如 C、C++、Go 和 Java。网关对后端的请求，以及对前端的请求的服务一定要使用异步非阻塞的 I/O 来确保后端延迟不会导致应用程序中出现性能问题。C 和 C++ 可以参看 Linux 下的 epoll 和 Windows 的 I/O Completion Port 的异步 IO 模型，Java 下如 Netty、Spring Reactor 的 NIO 框架。

#### 高可用
因为所有的流量或调用经过网关，所以网关必须成为一个高可用的技术组件，它的稳定直接关系到了所有服务的稳定。网关如果没有设计，就会成变一个单点故障。因此，一个好的网关至少要做到以下几点。

+ **集群化**。网关要成为一个集群，其最好可以自己组成一个集群，并可以自己同步集群数据，而不需要依赖于一个第三方系统来同步数据。
+ **服务化**。网关还需要做到在不间断的情况下修改配置，一种是像 Nginx reload 配置那样，可以做到不停服务，另一种是最好做到服务化。也就是说，得要有自己的 Admin API 来在运行时修改自己的配置。
+ **持续化**。比如重启，就是像 Nginx 那样优雅地重启。有一个主管请求分发的主进程。当我们需要重启时，新的请求被分配到新的进程中，而老的进程处理完正在处理的请求后就退出。

高可用性涵盖了内部和外部的各种不确定因素，这里讲一下网关系统在高可用性方面做的努力。

![](https://cdn.nlark.com/yuque/0/2024/png/34596451/1732590488499-9411e4b3-e877-4f04-82f4-a4ee3cda2b50.png)

#### 高扩展
因为网关需要承接所有的业务流量和请求，所以一定会有或多或少的业务逻辑。而我们都知道，业务逻辑是多变和不确定的。比如，需要在网关上加入一些和业务相关的东西。因此，一个好的 Gateway 还需要是可以扩展的，并能进行二次开发的。当然，像 Nginx 那样通过 Module 进行二次开发的固然可以。

另外，在**运维方面**，网关应该有以下几个设计原则。

+ **业务松耦合，协议紧耦合**。在业务设计上，网关不应与后面的服务之间形成服务耦合，也不应该有业务逻辑。网关应该是在网络应用层上的组件，不应该处理通讯协议体，只应该解析和处理通讯协议头。另外，除了服务发现外，网关不应该有第三方服务的依赖。
+ **应用监视，提供分析数据**。网关上需要考虑应用性能的监控，除了有相应后端服务的高可用的统计之外，还需要使用 Tracing ID 实施分布式链路跟踪，并统计好一定时间内每个 API 的吞吐量、响应时间和返回码，以便启动弹力设计中的相应策略。
+ **用弹力设计保护后端服务**。网关上一定要实现熔断、限流、重试和超时等弹力设计。如果一个或多个服务调用花费的时间过长，那么可接受超时并返回一部分数据，或是返回一个网关里的缓存的上一次成功请求的数据。你可以考虑一下这样的设计。
+ **DevOps**。因为网关这个组件太关键了，所以需要 DevOps 这样的东西，将其发生故障的概率降到最低。这个软件需要经过精良的测试，包括功能和性能的测试，还有浸泡测试。还需要有一系列自动化运维的管控工具。

### 网关设计注意事项
1. 不要在网关中的代码里内置聚合后端服务的功能，而应考虑将聚合服务放在网关核心代码之外。可以使用 Plugin 的方式，也可以放在网关后面形成一个 Serverless 服务。
2. 网关应该靠近后端服务，并和后端服务使用同一个内网，这样可以保证网关和后端服务调用的低延迟，并可以减少很多网络上的问题。这里多说一句，网关处理的静态内容应该靠近用户（应该放到 CDN 上），而网关和此时的动态服务应该靠近后端服务。
3. 网关也需要做容量扩展，所以需要成为一个集群来分担前端带来的流量。这一点，要么通过 DNS 轮询的方式实现，要么通过 CDN 来做流量调度，或者通过更为底层的性能更高的负载均衡设备。
4. 对于服务发现，可以做一个时间不长的缓存，这样不需要每次请求都去查一下相关的服务所在的地方。当然，如果你的系统不复杂，可以考虑把服务发现的功能直接集成进网关中。
5. 为网关考虑 bulkhead 设计方式。用不同的网关服务不同的后端服务，或是用不同的网关服务前端不同的客户。

另外，因为网关是为用户请求和后端服务的桥接装置，所以需要考虑一些安全方面的事宜。具体如下：

1. **加密数据**。可以把 SSL 相关的证书放到网关上，由网关做统一的 SSL 传输管理。
2. **校验用户的请求**。一些基本的用户验证可以放在网关上来做，比如用户是否已登录，用户请求中的 token 是否合法等。但是，我们需要权衡一下，网关是否需要校验用户的输入。因为这样一来，网关就需要从只关心协议头，到需要关心协议体。而协议体中的东西一方面不像协议头是标准的，另一方面解析协议体还要耗费大量的运行时间，从而降低网关的性能。对此，我想说的是，看具体需求，一方面如果协议体是标准的，那么可以干；另一方面，对于解析协议所带来的性能问题，需要做相应的隔离。
3. **检测异常访问**。网关需要检测一些异常访问，比如，在一段比较短的时间内请求次数超过一定数值；还比如，同一客户端的 4xx 请求出错率太高……对于这样的一些请求访问，网关一方面要把这样的请求屏蔽掉，另一方面需要发出警告，有可能会是一些比较重大的安全问题，如被黑客攻击。

## Sentinel
<font style="color:#000000;background-color:rgba(255, 255, 255, 0);">随着微服务的流行，服务和服务之间的稳定性变得越来越重要。Sentinel 以流量为切入点，从流量控制、熔断降级、系统负载保护等多个维度保护服务的稳定性。</font>

Sentinel 具有以下特征:

+ **丰富的应用场景**<font style="color:#000000;background-color:rgba(255, 255, 255, 0);">：Sentinel 承接了阿里巴巴近 10 年的双十一大促流量的核心场景，例如秒杀（即突发流量控制在系统容量可以承受的范围）、消息削峰填谷、集群流量控制、实时熔断下游不可用应用等。</font>
+ **完备的实时监控**<font style="color:#000000;background-color:rgba(255, 255, 255, 0);">：Sentinel 同时提供实时的监控功能。您可以在控制台中看到接入应用的单台机器秒级数据，甚至 500 台以下规模的集群的汇总运行情况。</font>
+ **广泛的开源生态**<font style="color:#000000;background-color:rgba(255, 255, 255, 0);">：Sentinel 提供开箱即用的与其它开源框架/库的整合模块，例如与 Spring Cloud、Apache Dubbo、gRPC、Quarkus 的整合。您只需要引入相应的依赖并进行简单的配置即可快速地接入 Sentinel。同时 Sentinel 提供 Java/Go/C++ 等多语言的原生实现。</font>
+ **完善的 SPI 扩展机制**<font style="color:#000000;background-color:rgba(255, 255, 255, 0);">：Sentinel 提供简单易用、完善的 SPI 扩展接口。您可以通过实现扩展接口来快速地定制逻辑。例如定制规则管理、适配动态数据源等。</font>

<font style="color:#000000;background-color:rgba(255, 255, 255, 0);"></font>

<font style="color:#000000;background-color:rgba(255, 255, 255, 0);">Sentinel控制台下载：</font>[Releases · alibaba/Sentinel](https://github.com/alibaba/Sentinel/releases)

### 依赖导入
```java
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
</dependency>
```

![](https://cdn.nlark.com/yuque/0/2024/png/34596451/1732530728726-48da0106-dc1e-4d01-8cc2-d6d18a91c552.png)

**yml配置**

```java
spring:
  application:
    name: userservice
  cloud:
    nacos:
      discovery:
        server-addr: localhost:8848
    sentinel:
      transport:
      	# 添加监控页面地址即可
        dashboard: localhost:8858
```

配置完，即可进入控制台。



### 流量控制
#### 限流策略
+ <font style="color:#000000;background-color:rgba(255, 255, 255, 0);">方案一：</font>**<font style="color:#000000;background-color:rgba(255, 255, 255, 0);">快速拒绝</font>**<font style="color:#000000;background-color:rgba(255, 255, 255, 0);">，既然不再接受新的请求，那么我们可以直接返回一个拒绝信息，告诉用户访问频率过高。</font>
+ <font style="color:#000000;background-color:rgba(255, 255, 255, 0);">方案二：</font>**<font style="color:#000000;background-color:rgba(255, 255, 255, 0);">预热</font>**<font style="color:#000000;background-color:rgba(255, 255, 255, 0);">，依然基于方案一，但是由于某些情况下高并发请求是在某一时刻突然到来，我们可以缓慢地将阈值提高到指定阈值，形成一个缓冲保护。</font>
+ <font style="color:#000000;background-color:rgba(255, 255, 255, 0);">方案三：</font>**<font style="color:#000000;background-color:rgba(255, 255, 255, 0);">排队等待</font>**<font style="color:#000000;background-color:rgba(255, 255, 255, 0);">，不接受新的请求，但是也不直接拒绝，而是进队列先等一下，如果规定时间内能够执行，那么就执行，要是超时就算了。</font>

#### <font style="color:#000000;background-color:rgba(255, 255, 255, 0);">限流算法</font>
**漏桶算法**

顾名思义，就像一个桶开了一个小孔，水流进桶中的速度肯定是远大于水流出桶的速度的，这也是最简单的一种限流思路：

+ ![](https://cdn.nlark.com/yuque/0/2024/webp/34596451/1732535005870-30bf134c-5fd7-4bb3-ae40-8fe3af9ec69d.webp)

我们知道，桶是有容量的，所以当桶的容量已满时，就装不下水了，这时就只有丢弃请求了。

利用这种思想，我们就可以写出一个简单的限流算法。

**令牌桶算法**

只能说有点像信号量机制。现在有一个令牌桶，这个桶是专门存放令牌的，每隔一段时间就向桶中丢入一个令牌（速度由我们指定）当新的请求到达时，将从桶中删除令牌，接着请求就可以通过并给到服务，但是如果桶中的令牌数量不足，那么不会删除令牌，而是让此数据包等待。

+ ![](https://cdn.nlark.com/yuque/0/2024/webp/34596451/1732535005889-fde6ba3d-9672-4050-917d-4edc95a7d7fd.webp)

可以试想一下，当流量下降时，令牌桶中的令牌会逐渐积累，这样如果突然出现高并发，那么就能在短时间内拿到大量的令牌。

**固定时间窗口算法**

我们可以对某一个时间段内的请求进行统计和计数，比如在`14:15`到`14:16`这一分钟内，请求量不能超过`100`，也就是一分钟之内不能超过`100`次请求，那么就可以像下面这样进行划分：

+ ![](https://cdn.nlark.com/yuque/0/2024/webp/34596451/1732535005938-8d860c74-522c-4ea5-a6a6-a564ec831200.webp)

虽然这种模式看似比较合理，但是试想一下这种情况：

+ 14:15:59的时候来了100个请求
+ 14:16:01的时候又来了100个请求

出现上面这种情况，符合固定时间窗口算法的规则，所以这200个请求都能正常接受，但是，如果你反应比较快，应该发现了，我们其实希望的是60秒内只有100个请求，但是这种情况却是在3秒内出现了200个请求，很明显已经违背了我们的初衷。

因此，当遇到临界点时，固定时间窗口算法存在安全隐患。

**滑动时间窗口算法**

相对于固定窗口算法，滑动时间窗口算法更加灵活，它会动态移动窗口，重新进行计算：

+ ![](https://cdn.nlark.com/yuque/0/2024/webp/34596451/1732535005932-1223c243-f6b9-4435-9d00-63c2d6df51bb.webp)

虽然这样能够避免固定时间窗口的临界问题，但是这样显然是比固定窗口更加耗时的。

#### 限流模式
+ <font style="color:#000000;">直接：只针对于当前接口。</font>
+ <font style="color:#000000;">关联：当其他接口超过阈值时，会导致当前接口被限流。 绑定某个接口，当其限流时，自身也限流</font>
+ <font style="color:#000000;">链路：更细粒度的限流，能精确到具体的方法。</font>
    - <font style="color:#000000;">需要对特定方法进行注入注解SentinelResource(value = "方法名") ，监控当前方法</font>
    - <font style="color:#000000;">在yml文件关闭Context收敛， 这样可以进行不同链路的单独控制</font>

```java
spring:
  application:
    name: borrowservice
  cloud:
    sentinel:
      transport:
        dashboard: localhost:8858
      # 关闭Context收敛，这样被监控方法可以进行不同链路的单独控制
      web-context-unify: false
```

![](https://cdn.nlark.com/yuque/0/2024/png/34596451/1732535239901-d5e0c042-4ce0-457c-8baa-1209e8115089.png)  
如上图所示，现在只对borrow2/{uid}方法进行限流，borrow/{uid}不受限流影响

除了直接对接口进行限流规则控制之外，我们也可以根据当前系统的资源使用情况，决定是否进行限流：

![](https://cdn.nlark.com/yuque/0/2024/webp/34596451/1732535301833-63c32b2a-5b5d-452d-82d3-fd530d48e885.webp)

系统规则支持以下的模式：

+ **Load 自适应**（仅对 Linux/Unix-like 机器生效）：系统的 load1 作为启发指标，进行自适应系统保护。当系统 load1 超过设定的启发值，且系统当前的并发线程数超过估算的系统容量时才会触发系统保护（BBR 阶段）。系统容量由系统的 `maxQps * minRt` 估算得出。设定参考值一般是 `CPU cores * 2.5`。
+ **CPU usage**（1.5.0+ 版本）：当系统 CPU 使用率超过阈值即触发系统保护（取值范围 0.0-1.0），比较灵敏。
+ **平均 RT**：当单台机器上所有入口流量的平均 RT 达到阈值即触发系统保护，单位是毫秒。
+ **并发线程数**：当单台机器上所有入口流量的并发线程数达到阈值即触发系统保护。
+ **入口 QPS**：当单台机器上所有入口流量的 QPS 达到阈值即触发系统保护。

### 限流异常处理
#### 限流页面跳转
首先需要在控制层添加限流返回的信息

```java
@RequestMapping("/blocked")
    JSONObject blocked(){
        JSONObject jsonObject = new JSONObject();
        jsonObject.put("code",403);
        jsonObject.put("success",false);
        jsonObject.put("message","你的请求频率过快，请稍后再试");
        return jsonObject;
    }
```

<font style="color:rgb(93, 93, 93);">接着我们在配置文件中将此页面设定为限流页面：</font>

```yaml
spring:
  cloud:
    sentinel:
      transport:
        dashboard: localhost:8858
      # 将刚刚编写的请求映射设定为限流页面
      block-page: /blocked
```

<font style="color:rgb(93, 93, 93);">这样，当被限流时，就会被重定向到指定页面：</font>

![](https://cdn.nlark.com/yuque/0/2024/png/34596451/1732536464828-23e60bc7-fd7f-4ace-b47e-3cbde9382c1c.png)

#### 限流方法处理
当某个方法被限流时，会直接在后台抛出异常，这个时候我们需要通过注解，利用替代方法返回。

```java
    @Override
    @SentinelResource(value = "getBorrow", blockHandler = "blocked")
    public UserBorrowDetail getUserBorrowDetailByUid(int uid) {
        List<Borrow> borrow = mapper.getBorrowsByUid(uid);
        User user = userClient.getUserById(uid);
        //获取每一本书的详细信息
        List<Book> bookList = borrow
                .stream()
                .map(b -> bookClient.getBookById(b.getBid()))
                .collect(Collectors.toList());
        return new UserBorrowDetail(user, bookList);
    }
    //替代方案，注意参数和返回值需要保持一致，并且参数最后还需要额外添加一个BlockException
    public UserBorrowDetail blocked(int uid, BlockException e){
        return new UserBorrowDetail(null, Collections.emptyList());
    }

@SentinelResource(value = "test",
        fallback = "except",    //fallback指定出现异常时的替代方案
        blockHandler = "blocked", // 限流出现的情况
        exceptionsToIgnore = IOException.class)  //忽略那些异常，也就是说这些异常出现时不使用替代方案
```

通过blockHandler 绑定 blocked方法，当抛出异常时则进行该方法。需要注意参数需要**携带BlockException！！！ 当出现blockedHandler和fallback都存在时，优先执行blockHandler的方法**

![执行结果](https://cdn.nlark.com/yuque/0/2024/png/34596451/1732536740579-ac861fc2-ba48-409b-b337-c02c10b5f162.png)

### 热点参数限流
创建一个参数多的方法

```java
@RequestMapping("/test")
    @SentinelResource("test")   //注意这里需要添加@SentinelResource才可以，用户资源名称就使用这里定义的资源名称
    String findUserBorrows2(@RequestParam(value = "a", required = false) String a,
        @RequestParam(value = "b", required = false) String b,
        @RequestParam(value = "c",required = false) String c) {
        return "请求成功！a = "+a+", b = "+b+", c = "+c;
    }
```

限流显示  
![](https://cdn.nlark.com/yuque/0/2024/png/34596451/1732538376600-b3b14aa6-69f8-4111-af2c-78073a8bed20.png)

热点规则设置

![](https://cdn.nlark.com/yuque/0/2024/png/34596451/1732538343944-a63f8092-1001-44be-a1d8-7e42761ef941.png)

### 服务熔断和降级
<font style="color:rgb(93, 93, 93);">在整个微服务调用链路出现问题的时候，及时对服务进行降级，以防止问题进一步恶化。</font>

![](https://cdn.nlark.com/yuque/0/2024/webp/34596451/1732541247283-fbc062e9-642e-4e61-9c2b-9c06ca8afa53.webp)

<font style="color:rgb(93, 93, 93);">如果在某一时刻，服务B出现故障（可能就卡在那里了），而这时服务A依然有大量的请求，在调用服务B，那么，由于服务A没办法再短时间内完成处理，新来的请求就会导致线程数不断地增加，这样，CPU的资源很快就会被耗尽。</font>

#### <font style="color:rgb(93, 93, 93);">隔离方式</font>
**<font style="color:rgb(93, 93, 93);">1.线程池隔离</font>**

<font style="color:rgb(93, 93, 93);">线程池隔离实际上就是对每个服务的远程调用单独开放线程池，比如服务A要调用服务B，那么只基于固定数量的线程池，这样即使在短时间内出现大量请求，由于没有线程可以分配，所以就不会导致资源耗尽了。</font>

**<font style="color:rgb(93, 93, 93);">2.信号量隔离</font>**

<font style="color:rgb(93, 93, 93);">信号量隔离是使用</font>`Semaphore`<font style="color:rgb(93, 93, 93);">类实现的（如果不了解，可以观看本系列 并发编程篇 视频教程），思想基本上与上面是相同的，也是限定指定的线程数量能够同时进行服务调用，但是它相对于线程池隔离，开销会更小一些，使用效果同样优秀，也支持超时等。Sentinel也正是采用的这种方案实现隔离的。</font>



因此，<font style="color:rgb(93, 93, 93);">当下游服务因为某种原因变得不可用或响应过慢时，上游服务为了保证自己整体服务的可用性，不再继续调用目标服务而是快速返回或是执行自己的替代方案，这便是服务降级。</font>

#### <font style="color:rgb(93, 93, 93);">服务降级状态</font>
<font style="color:rgb(93, 93, 93);">整个过程分为三个状态：</font>

+ <font style="color:rgb(93, 93, 93);">关闭：熔断器不工作，所有请求全部该干嘛干嘛。</font>
+ <font style="color:rgb(93, 93, 93);">打开：熔断器工作，所有请求一律降级处理。</font>
+ <font style="color:rgb(93, 93, 93);">半开：尝试进行一下下正常流程，要是还不行继续保持打开状态，否则关闭。</font>

![](https://cdn.nlark.com/yuque/0/2024/png/34596451/1732541380241-88eb3345-0685-4f4e-9411-9ec3f8be6042.png)

#### 熔断策略
##### 慢调用比例
<font style="color:rgb(93, 93, 93);"> 如果出现那种半天都处理不完的调用，有可能就是服务出现故障，导致卡顿，这个选项是按照最大响应时间（RT）进行判定，如果一次请求的处理时间超过了指定的RT，那么就被判定为</font>`慢调用`<font style="color:rgb(93, 93, 93);">，在一个统计时长内，如果请求数目大于最小请求数目，并且被判定为</font>`慢调用`<font style="color:rgb(93, 93, 93);">的请求比例已经超过阈值，将触发熔断。经过熔断时长之后，将会进入到半开状态进行试探</font>

```java
@RequestMapping("/borrow2/{uid}")
UserBorrowDetail findUserBorrows2(@PathVariable("uid") int uid) throws InterruptedException {
    Thread.sleep(1000);
    return null;
}
```

![](https://cdn.nlark.com/yuque/0/2024/png/34596451/1732541540825-53ed86b3-dc14-4260-8ee5-bee14655c6c9.png)

<font style="color:rgb(93, 93, 93);">此时超时则会直接触发熔断机制，进入阻止页面。</font>

##### <font style="color:rgb(93, 93, 93);">异常比例</font>
<font style="color:rgb(93, 93, 93);"> 这个与慢调用比例类似，不过这里判断的是出现异常的次数</font>

![](https://cdn.nlark.com/yuque/0/2024/png/34596451/1732541644659-cf4ea51b-b449-414d-9f5b-02b348de6bd9.png)

##### 异常数
<font style="color:rgb(93, 93, 93);">这个和上面的唯一区别就是，只要达到指定的异常数量，就熔断</font>

#### <font style="color:rgb(93, 93, 93);">自定义服务降级</font>
和之前限流，出现异常一样，执行替代方案

 **我们只需要在**`**@SentinelResource**`**中配置**`**blockHandler**`**参数**（那这里跟前面那个方法限流的配置不是一毛一样吗？没错，因为如果添加了`@SentinelResource`注解，那么这里会进行方法级别细粒度的限制，和之前方法级别限流一样，会在降级之后直接抛出异常，如果不添加则返回默认的限流页面，`blockHandler`的目的就是处理这种Sentinel机制上的异常，所以这里其实和之前的限流配置是一个道理，因此下面熔断配置也应该对`value`自定义名称的资源进行配置，才能作用到此方法上）：  


#### 整合Feign
<font style="color:rgb(93, 93, 93);">首先需要在配置文件中开启支持：</font>

```plain
feign:
  sentinel:
    enabled: true
```

然后创建实现类

```java
@Component
public class BookClientImpl implements BookClient{
    @Override
    public Book getBookById(int bid) {
        Book book = new Book();
        book.setTitle("替代");
        book.setDesc("替代啊");
        return book;
    }
}
```

实现接口添加替代方案实现类

```java
@FeignClient(value = "book-service", fallback = BookClientImpl.class)
public interface BookClient {
    @RequestMapping("/book/{bid}")
    public Book getBookById(@PathVariable("bid") int bid);
}
```

然后直接启动，中途下线其他服务，就可以看到正常使用替代方案  
![](https://cdn.nlark.com/yuque/0/2024/png/34596451/1732542014329-6cd1ddc1-4c97-4858-a9fe-ea5d2e2b8abf.png)

## seata
单体应用可以通过@<font style="color:rgb(17, 17, 17);">transactional实现回滚，但分布式需要通过Seata实现</font>

Seata 对分布式事务的协调和控制，主要是通过 XID 和 3 个核心组件实现的。

#### XID
<font style="color:rgb(17, 17, 17);">XID 是全局事务的唯一标识，它可以在服务的调用链路中传递，绑定到服务的事务上下文中。</font>

#### 核心组件
<font style="color:rgb(17, 17, 17);">Seata 定义了 3 个核心组件：</font>

+ <font style="color:rgb(17, 17, 17);">TC（Transaction Coordinator）：事务协调器，它是事务的协调者（这里指的是 Seata 服务器），主要负责维护全局事务和分支事务的状态，驱动全局事务提交或回滚。</font>
+ <font style="color:rgb(17, 17, 17);">TM（Transaction Manager）：事务管理器，它是事务的发起者，负责定义全局事务的范围，并根据 TC 维护的全局事务和分支事务状态，做出开始事务、提交事务、回滚事务的决议。</font>
+ <font style="color:rgb(17, 17, 17);">RM（Resource Manager）：资源管理器，它是资源的管理者（这里可以将其理解为各服务使用的数据库）。它负责管理分支事务上的资源，向 TC 注册分支事务，汇报分支事务状态，驱动分支事务的提交或回滚。</font>

<font style="color:rgb(17, 17, 17);">以上三个组件相互协作，TC 以 Seata 服务器（Server）形式独立部署，TM 和 RM 则是以 Seata Client 的形式集成在微服务中运行，其整体工作流程如下图。</font>

![](https://cdn.nlark.com/yuque/0/2024/png/34596451/1732601959307-9a332c09-ec55-4677-b87e-54991f1af5d9.png)  
图1：Sentinel 的工作流程

Seata 的整体工作流程如下：

1. TM 向 TC 申请开启一个全局事务，全局事务创建成功后，TC 会针对这个全局事务生成一个全局唯一的 XID；
2. XID 通过服务的调用链传递到其他服务;
3. RM 向 TC 注册一个分支事务，并将其纳入 XID 对应全局事务的管辖；
4. TM 根据 TC 收集的各个分支事务的执行结果，向 TC 发起全局事务提交或回滚决议；
5. TC 调度 XID 下管辖的所有分支事务完成提交或回滚操作。

## Seata AT 模式
---

Seata 提供了 AT、TCC、SAGA 和 XA 四种事务模式，可以快速有效地对分布式事务进行控制。

在这四种事务模式中使用最多，最方便的就是 AT 模式。与其他事务模式相比，AT 模式可以应对大多数的业务场景，且基本可以做到无业务入侵，开发人员能够有更多的精力关注于业务逻辑开发。

### AT 模式的前提
任何应用想要使用 Seata 的 AT 模式对分布式事务进行控制，必须满足以下 2 个前提：

+ 必须使用支持本地 ACID 事务特性的关系型数据库，例如 MySQL、Oracle 等；
+ 应用程序必须是使用 JDBC 对数据库进行访问的 JAVA 应用。

此外，我们还需要针对业务中涉及的各个数据库表，分别创建一个 UNDO_LOG（回滚日志）表。不同数据库在创建 UNDO_LOG 表时会略有不同，以 MySQL 为例，其 UNDO_LOG 表的创表语句如下：

```java
CREATE TABLE `undo_log` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `branch_id` bigint(20) NOT NULL,
  `xid` varchar(100) NOT NULL,
  `context` varchar(128) NOT NULL,
  `rollback_info` longblob NOT NULL,
  `log_status` int(11) NOT NULL,
  `log_created` datetime NOT NULL,
  `log_modified` datetime NOT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `ux_undo_log` (`xid`,`branch_id`)
) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8;

```

