# Spring Cloud Hystrix

在微服务架构中， 我们将系统拆分成了很多服务单元， 各单元的应用间通过服务注册与订阅的方式互相依赖。 由于每个单元都在不同的进程中运行， 依赖通过远程调用的方式执行， 这样就有可能因为网络原因或是依赖服务自身间题出现调用故障或延迟，而这些问题会直接导致调用方的对外服务也出现延迟， 若此时调用方的请求不断增加，最后就会因等待出现故障的依赖方响应形成任务积压，最终导致自身服务的瘫痪。

## 快速入门



![01](.\images\hystrix-02.png)

### 启动工程如下

eureka-server:服务注册中心，端口11112

hello-server:服务单元，启动两个实例8081/8082

ribbon-consume:使用ribbon实现的消费者，端口9000

未加入断路器之前，关闭8081，发送GET消费，获得如下输出

![01](.\images\hystrix-01.png)

### 下面开始引入hystrix

- **在ribbon-consume的pom.xml加入依赖**

```java
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
</dependency>
```

- **工程主类加入@EnableCircuitBreaker注解开启断路器功能**

注意：这里还可以使用 Spring Cloud 应用中的 @SpringCloudApplication 注解来修饰应用主类， 该注解的具体定义如下所示。 可以看到， 该注解中包含了上述我们所引用的三个注解， 这也意味着—个Spring Cloud 标准应用应包含**服务发现以及断路器**。

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootApplication
@EnableDiscoveryClient
@EnableCircuitBreaker
public @interface SpringCloudApplication {
}
```

- **改造服务消费方式**

**新增 HelloService 类**， 注入 RestTemplate 实例。 然后，将在 ConsumerController 中对 RestTemplate 的使用迁移到 helloService函数中，最后，在 helloService 函数上增加 @HystrixCommand注解来指定回调方法。

```java
@Service
public class helloService {
    @Autowired
    RestTemplate restTemplate;

    @HystrixCommand(fallbackMethod = "helloFaceback")
    public String helloConsumer(){
        return restTemplate.getForEntity("http://HELLOSERVICE/hello",String.class).getBody();
    }

    public String helloFaceback(){
        return "error";
    }
}
```

**修改controller**,注入实例并调用

```java
	@Autowired
    private helloService helloService;

    @RequestMapping(value = "/ribbon-consumer",method = RequestMethod.GET)
    public String helloService(){
        return helloService.helloConsumer();
    }
```

- **验证结果**

我们来验证一下通过断路器实现的服务回调逻辑，重新启动之前关闭的8081端口的Hello-Service, 确保此时服务注册中心、 两个 Hello-Service 以及 RIBBON­CONSUMER 均已启动，访问 http://localhost:9000/ribbon-consumer 可以轮询两个 HELLO-SERV工CE 并返回一些文字信息 此时我们继续断开8081的HELLO-SERVICE,然后访问 http://localhost:9000/ribbon-consumer, 当轮询到 8081 服务端时，输出内容为 error, 不再是之前的错误内容， Hystrix 的服务回调生效。

![01](.\images\hystrix-03.png)

除了通过断开具体的服务实例来模拟某个节点无法访问的情况之外，我们还可以模拟 一 下服务阻塞（长时间未响应）的情况。 我们对 HELLO-SERVICE 的 /hello 接口做 一 些修改，具体如下:

```java
@RequestMapping(value = "/hello",method = RequestMethod.GET)
    public String index() throws InterruptedException {
        for(String s:client.getServices()){
            List<ServiceInstance> instances=client.getInstances(s);
            //让线程等待几秒
            int sleepTime=new Random().nextInt(3000);
            logger.info("sleepTime:" + sleepTime);
            Thread.sleep(sleepTime);
            for(ServiceInstance instance:instances){
                logger.info(
                        "/hello, host:" + instance.getHost() + ", service_id:" + instance.getServiceId());
            }
        }
        return "Hello World";
    }
```

通过 Thread. sleep ()函数可让 /hello 接口的处理线程不是马上返回内容，而是在阻塞几秒之后才返回内容。 

由于 Hystrix 默认超时时间为 1000 毫秒，所以这里采用了 0至3000 的随机数以让处理过程有 一 定概率发生超时来触发断路器。

![1555210847334](D:/%E7%A7%81%E4%BA%BA/markdownFile/hystrix-04.png)

为了更精准地观察断路器的触发，在消费者调用函数中做一些时间记录，具体如下：

```java
@HystrixCommand(fallbackMethod = "helloFaceback",commandKey = "helloKey")
    public String helloConsumer(){
        long start = System.currentTimeMillis();
        String result=restTemplate.getForEntity("http://HELLO-SERVICE/hello",String.class).getBody();
        long end= System.currentTimeMillis();
        logger.info("Spend time : "+ (end - start));
        return result.toString();
    }
```

重新启动HELLO-SERVICE和RIBBON-CONSUMER的实例，连续访问http://localhost:9000/ribbon-consumer几次，我们可以观察到，当RIBBON-CONSUMER的控制台中输出的Spend time大于2000的时候，就会返回error, 即服务消费者因调用的服务超时从而触发熔断请求，并调用回调逻辑返回结果。

**参数配置对应类：HystrixCommandProperties**

```
@HystrixCommand(fallbackMethod = "helloFaceback",commandProperties = {@HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds",value = "2000")})
```

## Hystrix仪表盘

![01](.\images\hystrix-05.png)

- **创建一个标准的 Spring Boot 工程， 命名为 hystrix-dashboard 。**
- **编辑 pom.xml,  具体依赖内容如下所示**：

```java
		<dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-hystrix-dashboard</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
```

- **为应用主类加上 @EnableHystrixDashboard, 启用 Hystrix Dashboard 功能**
- **根据实际情况修改application.properties 配置文件**

```
spring.application.name=hystrix-dashboard
server.port=9001
```

修改ribbon-consumer的pom

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

启动类加上

```
@EnableCircuitBreaker
@EnableHystrix
@EnableHystrixDashboard



@Bean
public ServletRegistrationBean getServlet(){

    HystrixMetricsStreamServlet streamServlet = new HystrixMetricsStreamServlet();

    ServletRegistrationBean registrationBean = new ServletRegistrationBean(streamServlet);

    registrationBean.setLoadOnStartup(1);

    registrationBean.addUrlMappings("/actuator/hystrix.stream");

    registrationBean.setName("HystrixMetricsStreamServlet");


    return registrationBean;
}
```

![01](.\images\hystrix-06.png)

![01](.\images\hystrix-07.png)



