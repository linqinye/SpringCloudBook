# Spring Cloud Eureka

### 产生背景



服务治理可以说是微服务架构中最为核心和基础的模块， 它主要用来实现各个微服务实例的自动化注册与发现。随着业务的发展， 系统功能越来越复杂， 相应的微服务应用也不断增加， 我们的静态配置就会变得越来越难以维护。 并且面对不断发展的业务， 我们的集群规模、 服务的位置 、 服务的命名等都有可能发生变化， 如果还是通过手工维护的方式， 那么极易发生错误或是命名冲突等问题。 同时， 对于这类静态内容的维护也必将消耗大量的人力。为了解决微服务架构中的服务实例维护问题， 产生了大量的服务治理框架和产品



## 服务治理的基本概念



**服务注册：**在服务治理框架中， 通常都会构建 一 个注册中心， 每个服务单元向注册中心登记自己提供的服务， 将主机与端口号、 版本号、 通信协议等一些附加信息告知注册中心， 注册中心按服务名分类组织服务清单。

![00](.\images\eureka1.png)

**服务发现：**由于在服务治理框架下运作， 服务间的调用不再通过指定具体的实例地址来实现， 而是通过向服务名发起请求调用实现。 所以， 服务调用方在调用服务提供方接口的时候， 并不知道具体的服务实例位置。 因此， 调用方需要向服务注册中心咨询服务， 并获取所有服务的实例清单， 以实现对具体服务实例的访问。



### Netflix Eureka

**Spring Cloud Eureka**, 使用Netflix Eureka来实现服务注册与发现， 它既包含了服务端组件，也包含了客户端组件，并且服务端与客户端均采用Java编写，所以Eureka主要适用于通过Java实现的分布式系统，或是与JVM兼容语言构建的系统。但是， 由于Eureka服务端的服务治理机制提供了完备的RESTful API，所以它也支持将非Java语言构建的微服务应用纳入Eureka的服务治理体系中来。只是在使用其他语言平台的时候，需要自己来实现Eureka的客户端程序。不过庆幸的是，在目前几个较为流行的开发平台上，都已经有了一 些针对Eureka 注册中心的客户端实现框架.

**Eureka服务端**，我们也称为服务注册中心。它同其他服务注册中心 一 样，支持高可用配置。它依托于强 一 致性提供良好的服务实例可用性，可以应对多种不同的故障场景。如果Eureka以集群模式部署，当集群中有分片出现故障时，那么Eureka就转入自我保护模式。它允许在分片故障期间继续提供服务的发现和注册，当故障分片恢复运行时，集群中的其他分片会把它们的状态再次同步回来。以在AWS 上的实践为例，Netflix推荐每个可用的区域运行 一 个Eureka服务端，通过它来形成集群。不同可用区域的服务注册中心通过异步模式互相复制各自的状态，这意味着在任意给定的时间点每个实例关于所有服务的状态是有细微差别的。

**Eureka客户端**，主要处理服务的注册与发现。客户端服务通过注解和参数配置的方式，嵌入在客户端应用程序的代码中，在应用程序运行时，Eureka客户端向注册中心注册自身提供的服务并周期性地发送心跳来更新它的服务租约。同时，它也能从服务端查询当前注册的服务信息并把它们缓存到本地并周期性地刷新服务状态。



### 搭建服务注册中心

```java
<parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.1.3.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
</parent>

<dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>

```

通过 @EnableEurekaServer 注解启动 一 个服务注册中心提供给其他应用进行对话

```java
@EnableEurekaServer
@SpringBootApplication
public class EurekaServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}
```

在默认设置下， 该服务注册中心也会将自己作为客户端来尝试注册自己，所以我们需要禁用它的客户端注册行为

```java
server.port=8080
spring.application.name=eureka-server
eureka.instance.hostname=localhost
eureka.client.register-with-eureka=false
eureka.client.fetch-registry=false
eureka.client.serviceUrl.defaultZone=http://${eureka.instance.hostname}:${server.port}/eureka/
```

` eureka.client.register-with-eureka: 由于该应用为注册中心，所以设置为 false, 代表不向注册中心注册自己。`

`eureka.client.fetch-registry: 由于注册中心的职责就是维护服务实例，它并不需要去检索服务， 所以也设置为 false `

访问 http://localhost: 8080/ 



### 注册服务提供者

```java
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

在主类中通过加上 @EnableEurekaClient 注解， 激活 Eureka 中的DiscoveryClient 实现（自动化配置， 创建 DiscoveryClient 接口针对 Eureka 客户端的 EurekaDiscoveryClient 实例）

```java
@EnableEurekaClient
@SpringBootApplication
@RestController
public class HelloServerApplication {

    @RequestMapping("/")
    public String home() {
        return "Hello world";
    }

    public static void main(String[] args) {
        SpringApplication.run(HelloServerApplication.class, args);
    }

}
```

在 配置文件中， 通过 spring.application.name属性来为服务命名， 比如命名为 hello-service。 再通过eureka.client.serviceUrl.defaultZone属性来指定服务注册中心的地址， 这里我们指定为之前构建的服务注册中心地址

```java
server.port=9090
spring.application.name=hello-service
eureka.client.serviceUrl.defaultZone=http://127.0.0.1:8080/eureka/
```

![02](.\images\eureka2.png)

在服务注册中心的控制台中，可以看到类似下面的输出，名为hello-service的服务被注册成功了

![03](.\images\eureka3.png)



### 高可用注册中心

Eurek a Server的高可用实际上就是将自己作为服务向其他服务注册中心注册自己， 这样就可以形成 一 组互相注册的服务注册中心， 以实现服务清单的互相同步， 达到高可用的效果。

application-peer1.properties

```java
server.port=8080
spring.application.name=eureka-server
eureka.instance.hostname=peer1
eureka.client.serviceUrl.defaultZone=http://peer2:8081/eureka/
```

application-peer2.properties

```java
server.port=8081
spring.application.name=eureka-server
eureka.instance.hostname=peer2
eureka.client.serviceUrl.defaultZone=http://peer1:8080/eureka/
```

在etc/hosts文件中添加对peer1 和 peer2的转换， 让上面配置的host形式的serviceUrl能在本地正确访间到； Windows系统路径为C:\Windows\System32\drivers\etc\hosts。

```java
127.0.0.1	 	peer1
127.0.0.1		peer2
```

通过spring.profiles.active属性来分别启动peer1和peer2

```java
java -jar eureka-server-0.0.1-SNAPSHOT.jar --spring.profiles.active = peerl
java -jar eureka-server-0.0.1-SNAPSHOT.jar --spring.profiles.active = peer2
```



修改hello-service

```java
eureka.client.serviceUrl.defaultZone=http://127.0.0.1:11112/eureka/,http://127.0.0.1:11111/eureka/
```

如我们不想使用主机名来定义注册中心的地址，也可以使用IP地址的形式，但是需要在配置文件中增加配置参数eureka.instance.prefer-ip-address = true, 该值默认为false



### 服务发现与消费

服务消费者主要完成两个目标，发现服务以及消费服务 。

其中，服务发现的任务由Eureka的客户端完成，而服务消费的任务由Ribbon完成 。

Ribbon是一个基于HTTP和TCP的客户端负载均衡器，它可以在通过客户端中配置的 ribbonServerList服务端列表去轮询访问以达到均衡负载的作用。 当Ribbon与Eureka联合使用时，Ribbon的服务实例清单RibbonServerList会被DiscoveryEnabledNIWSServerList重写， 扩展成从Eureka注册中心中获取服务端列表。 同时它也会用 NIWSDiscoveryPing来取代IPing, 它将职责委托给Eureka 来确定服务端是否已经启动。它在Eureka服务发现的基础上，实现了一套对服务实例的选择策略，从而实现对服务的消费。

**1.为了实验Ribbon的客户端负载均衡功能， 我们通过 java-jar命令行的方式来启动两个不同端口的hello-service**

**2.创建 一 个 Spring Boot 的基础工程来实现服务消费者， 取名为 ribbon-consumer，导入 spring-cloud-starter-ribbon依赖**

**3.创建应用主类 ConsumerApplication, 通过 @EnableDiscoveryClient注解让该应用注册为 Eureka 客户端应用， 以获得服务发现的能力。 同时， 在该主类中创建 RestTemplate 的 Spring Bean 实例，并通过 @LoadBalanced 注解开启客户端负载均衡**

**4.创建ConsumerController类并实现／ribbon-consumer接口。 在该接口中，通过在上面创建的RestTemplate 来实现对 HELLO -SERVICE服务提供的/hello接口进行调用。 可以看到这里访问的地址是服务名HELLO-SERVICE, 而不是一个具体的地址。在服务治理框架中，这是一个非常重要的特性。**

**5.在application.properties中配置Eureka服务注册中心的位置, 需要与之前的HELLO-SERVICE一样，不然是发现不了该服务的，同时设置该消费者的端口不能与之前启动的应用端口冲突.**

**6.通过向 http://localhost:9000/ribbon-consumer 发起 GET 请求， 成功返回了 "Hello World" 。 此时， 我们可以在 ribbon-consumer 应用的控制台中看到Ribbon输出了当前客户端维护的 HELLO-SERVICE 的服务列表情况。其中包含了各个实例的位置,Ribbon就是按照此信息进行轮询访问，以实现基于客户端的负载均衡。 另外还输出了 一 些其他非常有用的信息， 如对各个实例的请求总数量、第一 次连接信息、上一次连接信息、总的请求失败数量等**

**7.再尝试发送几次请求， 并观察启动的两个 HELLO-SERVICE 的控制台， 可以看到两个控制台会交替打印日志， 这是我们之前在 HelloController 中实现的对服务信息的输出， 可以用来判断当前Ribbon-consumer 对 HELLO-SERVICE 的调用是否是负载均衡的。**



## Eureka详解



### 基础架构

Eureka服务治理体系的三个核心角色： 

**服务注册中心**： Eureka提供的服务端， 提供服务注册与发现的功能。

**服务提供者**：提供服务的应用， 可以是 Spring Boot 应用， 也可以是其他技术平台且遵循 Eureka通信机制的应用。它将自己提供的服务注册到 Eureka, 以供其他应用发现。

**服务消费者**：消费者应用从服务注册中心获取服务列表， 从而使消费者可以知道去何处调用其所需要的服务，包括Feign的消费。

**很多时候， 客户端既是服务提供者也是服务消费者。**



### 服务治理机制

- "服务注册中心-1" 和“ 服务注册中心-2", 它们互相注册组成了高可用集群。
- "服务提供者”启动了两个实例，一个注册到 “ 服务注册中心-1"上，另外一个注册到“ 服务注册中心-2" 上。
- 还有两个“ 服务消费者 “ ，它们也都分别只指向了一个注册中心。

![01](.\images\cloud-01.png)

- **服务提供者**

  **服务注册**：“服务提供者”在启动的时候会通过发送REST请求的方式将自己注册到EurekaServer上， 同时带上了自身服务的 一 些元数据信息。Eureka Server接收到这个REST请求之后，将元数据信息存储在一个双层结构Map中， 其中第一层的key是服务名， 第二层的key是具体服务的实例名。

  在服务注册时， 需要确认 一 下 eureka.client.register-with-eureka = true参数是否正确， 该值默认为true。 若设置为false将不会启动注册操作。

  **服务同步**：如架构图中所示， 这里的两个服务提供者分别注册到了两个不同的服务注册中心上，也就是说，它们的信息分别被两个服务注册中心所维护。 此时， 由于服务注册中心之间因互相注册为服务， 当服务提供者发送注册请求到 一 个服务注册中心时， 它会将该请求转发给集群中相连的其他注册中心，从而实现注册中心之间的服务同步 。 通过服务同步，两个服务提供者的服务信息就可以通过这两台服务注册中心中的任意 一 台获取到。

  **服务续约**：在注册完服务之后，服务提供者会维护一个心跳用来持续告诉EurekaSe1-ver: "我还活着” ， 以防Eureka Server的 “剔除任务”将该服务实例从服务列表中排除出去，我们称该操作为服务续约(Renew)

服务续约有两个重要属性：

```java
eureka.instance.lease-renewal-interval-in-seconds=30
```

定义服务续约任务的调用间隔时间，默认为30秒.

```java
eureka.instance.lease-expiration-duration-in-seconds=90
```

定义服务失效的时间，默认为90秒.

- **服务消费者**

  **获取服务**：当我们启动服务消费者的时候，它会发送一个REST请求给服务注册中心，来获取上面注册的服务清单 。为了性能考虑，EurekaServer会维护 一 份只读的服务清单来返回给客户端，同时该缓存清单会每隔30秒更新一 次。

  获取服务是服务消费者的基础

  ```java
  eureka.client.fetch-registry=true
  ```

    该值默认为true。

    若希望修改缓存清单的更新时间

  ```java
  eureka.client.registry-fetch-interval-seconds = 30
  ```

 	 该参数默认值为30, 单位为秒.

​	**服务调用：**服务消费者在获取服务清单后，通过服务名可以获得具体提供服务的实例名和该实例的元数据信息。 因为有这些服务实例的详细信息， 所以客户端可以根据自己的需要决定具体调用哪个实例，在Ribbon中会默认采用轮询的方式进行调用，从而实现客户端的负载均衡.

​	对于访问实例的选择，Eureka中有Region和Zone的概念，一个Region中可以包含多个Zone, 每个服务客户端需要被注册到一个Zone中， 所以每个客户端对应一个Region和一个Zone。 在进行服务调用的时候，优先访问同处一个Zone中的服务提供方， 若访问不到，就访问其他的Zone。

​	**服务下线**：在系统运行过程中必然会面临关闭或重启服务的某个实例的情况， 在服务关闭期间，我们自然不希望客户端会继续调用关闭了的实例。 所以在客户端程序中， 当服务实例进行正常的关闭操作时， 它会触发一个服务下线的REST请求给Eureka Server, 告诉服务注册中心：“我要下线了” 。 服务端在接收到请求之后， 将该服务状态置为下线(DOWN), 并把该下线事件传播出去。

- **服务注册中心**

  **失效剔除**：有些时候， 我们的服务实例并不一定会正常下线， 可能由于内存溢出、 网络故障等原因使得服务不能正常工作， 而服务注册中心并未收到“ 服务下线 ”的请求。 为了从服务列表中将这些无法提供服务的实例剔除， EurekaServer在启动的时候会创建 一 个定时任务，默认每隔一 段时间（默认为60秒）将当前清单中超时（默认为90秒）没有续约的服务剔除出去。

  **自我保护**:当我们在本地调试基于Eureka的程序时，基本上都会碰到这样一个问题， 在服务注册中心的信息面板中出现类似下面的红色警告信息:

  ![04](.\images\eureka4.png)

  实际上， 该警告就是触发了EurekaServer的自我保护机制。 之前我们介绍过， 服务注册到EurekaServer之后，会维护一个心跳连接，告诉EurekaServer自己还活着。EurekaServer在运行期间，会统计心跳失败的比例在15分钟之内是否低于85%, 如果出现低于的情况（在单机调试的时候很容易满足， 实际在生产环境上通常是由于网络不稳定导致）， EurekaServer会将当前的实例注册信息保护起来， 让这些实例不会过期， 尽可能保护这些注册信息。 但是， 在这段保护期间内实例若出现问题， 那么客户端很容易拿到实际已经不存在的服务实例， 会出现调用失败的清况， 所以客户端必须要有容错机制， 比如可以使用请求重试、 断路器等机制。
  由于本地调试很容易触发注册中心的保护机制， 这会使得注册中心维护的服务实例不那么准确。 所以， 我们在本地进行开发的时候，可以使用eureka.server.enable­self-preservation= false参数来关闭保护机制， 以确保注册中心可以将不可用的实例正确剔除。



## 配置详解

在Eureka的服务治理体系中， 主要分为服务端与客户端两个不同的角色， 服务端为服务注册中心， 而客户端为各个提供接口的微服务应用。 当我们构建了高可用的注册中心之后， 该集群中所有的微服务应用和基础类应用（如配置中心、 API网关等）都可以视作该体系下的一个微服务(Eureka客户端）。服务注册中心也 一 样， 只是高可用环境下的服务注册中心除了作为客户端之外， 还为集群中的其他客户端提供了服务注册的特殊功能。

Eureka客户端的配置主要分为以下两个方面：

- 服务注册相关的配置信息， 包括服务注册中心的地址、 服务获取的间隔时间、 可用区域等。
- 服务实例相关的配置信息， 包括服务实例的名称、IP地址、 端口号、 健康检查路径等。



### 服务注册类配置

配置信息都以 eureka.client 为前缀

- 指定注册中心

  为了服务注册中心的安全考虑， 很多时候我们都会为服务注册中心加入安全校验。 这个时候， 在配置 serviceUrl 时， 需要在 value 值的 URL 中加入相应的安全校验信息， 比如 http://<username>:<password>@localhost:1111/eureka 。 其中，<username> 为安全校验信息的用户名， <password> 为该用户的密码.

- 其他配置

  | 参数名                                        | 说明                                                         | 默认值 |
  | --------------------------------------------- | ------------------------------------------------------------ | ------ |
  | enabled                                       | 启用Eureka客户端                                             | true   |
  | registryFetchIntervalSeconds                  | 从Eureka服务端获取注册信息的间隔时间，单位为秒               | 30     |
  | instancelnfoReplicationlntervalSeconds        | 更新实例信息的变化到E田eka服务端的间隔时间， 单位为秒        | 30     |
  | initialInstanceInfoReplicationIntervalSeconds | 初始化 实例信息到Eureka服务端的间隔时 间，单位为秒           | 40     |
  | eurekaServiceUrlPollIntervalSeconds           | 轮询Eureka服务端地址更改的间隔时间， 单位为秒。 当我们与Spring Cloud Config配合，动态刷新Eureka的serv1ceURL地址时需要关注该参数 | 300    |
  | eurekaServerReadTimeoutSeconds                | 读取Eureka Se1-ver信息的超时时间， 单位为秒                  | 8      |
  | eurekaServerConnectTimeoutSeconds             | 连接 Eureka Server的超时时间， 单位为秒                      | 5      |

### 服务实例类配置

- **元数据**

  Eureka客户端在向服务注册中心发送注册请求时,用来描述自身服务信息的对象，其中包含了一些标准化的元数据，比如服务名称、实例名称、实例IP、实例端口等用于服务治理的重要信息；以及一些用于负载均衡策略或是其他特殊用途的自定义元数据信息。

  在使用 Spring Cloud Eureka的时候，所有的配置信息都通过org.springframework.cloud.netflix.eureka.EurekalnstanceConfigBean进行加载，但在真正进行服务注册的时候，还是会包装成com.netflix.appinfo.Instancelnfo 对象发送给Eureka服务端 。我们可以直接查看 com.netflix.appinfo.Instancelnfo类中的详细定义来了解原生Eureka对元数据的定义。 其中，
  Map<String, String> metadata= new ConcurrentHashMap<String, String> ()是自定义的元数据信息， 而其他成员变量则是标准化的元数据信息。

  我们可以通过 eureka.instance.<properties>=<value>的格式对标准化 元数据直接进行配置,其中<properties>就是 EurekainstanceConfigBean对象 中的成员变量名 。而对于自定义元数据，可以通过 eureka.instance.metadataMap.<key>=<value>的格式来进行配置，比如：

  ```java
  eureka.instance.metadataMap.zone=shanghai
  ```

- **实例名配置**

  实例名，即InstanceInfo 中的instanceId参数， 它是区分同一服务中不同实例的唯一标识。在NetflixEureka的原生实现中，实例名采用主机名作为默认值， 这样的设置使得在同一主机上无法启动多个相同的服务实例。所以， 在Spring Cloud Eureka的配置中，针对同一主机中启动多实例的情况，对实例名的默认命名做了更为合理的扩展， 它采用了如下默认规则：

  ```java
  ${spring.cloud.client.hostname}:${spring.application.name}:${spring.application
  .instance—id:$ {server.port}}
  ```

  对千实例名的命名规则，我们可以通过eureka.instance.instanceId参数来进行配置。 比如， 在本地进行客户端负载均衡调试时，需要启动同一服务的多个实例，如果我们直接启动同一个应用必然会产生端口冲突。 虽然可以在命令行中指定不同的server.port 来启动， 但是这样还是略显麻烦。 实际上，我们可以直接通过设置
  server.part=0 或者使用随机数 server.port=${randorn.int[10000,19999]} 来让Tomcat 启动的时候采用随机端口。但是这个时候我们会发现注册到 Eureka Server 的实例名都是相同的， 这会使得只有一个服务实例能够正常提供服务。 对于这个问题， 我们就可以通过设置实例名规则来轻松解决：

  ```java
  eureka.instance.instanceid=${spring.application.name}:${random.int}}
  ```

  利用应用名加随机数的方式来区分不同的实例，从而实现在同一主机上，不指定端口就能轻松启动多个实例的效果。

- **端点配置**

  在 Instanceinfo 中， 我们可以看到 一 些 URL 的配置信息， 比如 homePageUrl、statusPageUrl、healthCheckUrl, 它们分别代表了应用主页的 URL、状态页的 URL、健康检查的 URL 。

  其中，状态页和健康检查的URL在Spring Cloud Eureka 中默认使用了spring-boot-actuator 模块提供的 /info 端点和 /health 端点。

  为了服务的正常运作， 我们必须确保Eureka 客户端的 /health 端点在发送元数据的时候， 是一个能够被注册中心访问到的地址， 否则服务注册中心不会根据应用的健康检查来更改状态（仅当开启了 healthcheck 功能时， 以该端点信息作为健康检查标准）。 而/info 端点如果不正确的话，会导致在 Eureka 面板中单击服务实例时，无法访问到服务实例提供的信息接口。

  - 大多数情况下，我们并不需要修改这几个 URL 的配置，但是在 一 些特殊情况下，比如，为应用设置了 context-path, 这时， 所有 spring-boot-actuator 模块的监控端点都会增加一个前缀。

  ```java
  management.context-path=/hello
  eureka.instance.statusPageUrlPath=${management.context-path}/info
  eureka.instance.healthCheckUrlPath=${management.context-path}/health
  ```

  - 为了安全考虑， 也有可能会修改 /info 和 /health 端点的原始路径

    ```java
    endpoints.info.path=/appinfo
    endpoints.health.path=/checkHealth
    eureka.instance.statusPageOrlPath=/${endpoints.info.path}
    eureka.instance.healthCheckOrlPath=/${endpoints.health.path}
    ```

  - 上述两个配置值有一个共同特点，它们都使用相对路径来进行配置 。由千Eureka的服务注册中心默认会以HTTP的方式来访问和暴露这些端点，因此当客户端应用以HTTPS的方式来 暴露服务和监控端点时，相对路径的配置方式就无法满足需求了。所以，Spring Cloud Eureka还提供了绝对路径的配置参数

    ```java
    eureka.instance.statusPageUrl=https://${eureka.instance.hostname}/info
    eureka.instance.healthCheckUrl=https://${eureka.instance.hostname}/health
    eureka.instance.homePageUrl=https://${eureka.instance.hostname}/
    ```

- **健康检测**

  默认情况下，Eureka中各个服务实例的健康检测并不是通过spring-boot-actuator模块的/health 端点来实现的， 而是依靠客户端心跳的方式来保持服务实例的存活。 在Eureka 的服务续约与剔除机制下， 客户端的健康状态从注册到注册中心开始都会处于UP状态， 除非心跳终止 一 段时间之后，服务注册中心将其剔除。 默认的心跳实现方式可以有效检查客户端进程是否正常运作,但却无法保证客户端应用能够正常提供服务 。由于大多数微服务应用都会有 一 些其他的外部资源依赖， 比如数据库、 缓存、 消息代理等，如果我们的应用与这些外部资源无法联通的时候， 实际上已经不能提供正常的对外服务了， 但是因为客户端心跳依然在运行，所以它还是会被服务消费者调用， 而这样的调用实际上并不能获得预期的结果.

  通过简单的配置， 把Eureka客户端的健康检测交给spring-boot-actuator模块的/health端点， 以实现更加全面的健康状态维护。

  - 在pom.xml中引入spring-boot-starter-actuator模块的依赖。
  - 在application.properties中增加参数配置 eureka.client.healthcheck.enabled=true。
  - 如果客户端的/health端点路径做了一些特殊处理，请参考前文介绍端点配置时的方法进行配置， 让服务注册中心可以正确访问到健康检测端点。

- **其他配置**

  参数均以 eureka.instance为前缀

  | 参数名                           | 说明                                                         | 默认值 |
  | -------------------------------- | ------------------------------------------------------------ | ------ |
  | preferIpAddress                  | 是否优先使用IP地址作为主机名的标识                           | false  |
  | leaseRenewalIntervalInSeconds    | Eureka客户端向服务端发送心跳的时间间隔， 单位为秒            | 30     |
  | leaseExpirationDurationInSeconds | Eureka服务端在收到最后一 次心跳之后等待的时间上限，单位为秒。 超过该时间之后服务端会将该服务实例从服务消单中剔除， 从而禁止服务调用请求被发送到该实例上 | 90     |
  | nonSecurePort                    | 非安全的通信端口号                                           | 80     |
  | securePort                       | 安全的通信端口号                                             | 443    |
  | nonSecurePortEnabled             | 是否启用非安全的通信端口号                                   | true   |
  | securePortEnabled                | 是否启用安全的通信端口号                                     |        |
  | hostname                         | 主机名， 不配置的时候将根据操作系统的主机名来获取            |        |
  | appname                          | 服务名，默认取spring.application.name的配置值，如果没有则为unknown |        |

## 跨平台支持