# Spring Cloud Feign

基于 Netflix Feign 实现，整合了 Spring Cloud Ribbon 与 Spring Cloud Hystrix, 除了提供这两者的强大功能之外，它还提供了 一 种声明式的 Web 服务客户端定义方式。

创建一个接口并用注解的方式来配置它， 即可完成对服务提供方的接口绑定，简化了在使用 Spring Cloud Ribbon 时自行封装服务调用客户端的开发量。 Spring Cloud Feign 具备可插拔的注解支持，包括 Feign 注解和 JAX-RS 注解。 同时，为了适应 Spring 的广大用户，它在 Netflix Feign的基础上扩展了对 Spring MVC 的注解支待。

## 快速入门

1.创建 一 个 Spring Boot 基础工程， 取名为 feign-consumer, 并在 pom.xml中引入 spring-cloud-starter-eureka和spring-cloud-starter-feign依赖。

2.启动类注解开启

@EnableFeignClients
@EnableDiscoveryClient

3.定义 HelloService 接口， 通过 @FeignClient 注解指定服务名来绑定服务， 然后再使用 Spring MVC 的注解来绑定具体该服务提供的 REST 接口。

```
@FeignClient("HELLO-SERVICE")
public interface HelloService {
    @RequestMapping(value = "/hello")
    String hello();
}
```

注意：这里服务名不区分大小写， 所以使用 hello-service和HELLO-SERVICE 都是可以的。 另外， 在 Brixton.SR5 版本中， 原有的 serviceld 属性已经被废弃，若要写属性名， 可以使用 name或value 。

4.创建一个 ConsumerController 来实现对 Feign 客户端的调用。使用@Autowired直接注入上面定义的 HelloService 实例，并在 helloConsumer函数中调用这个绑定了 hello-service 服务接口的客户端来向该服务发起
/hello 接口的调用。

```java
@RestController
public class ConsumerController {
    @Autowired
    private HelloService helloService;
    @RequestMapping(method = RequestMethod.GET,value = "/feign-consumer")
    public String helloConsumer(){
        return helloService.hello();
    }
}
```

5.修改配置application.properties

```java
spring.application.name=feign-consumer
server.port=9001
eureka.client.serviceUrl.defaultZone = http://localhost:11112/eureka/
```

6.访问http://localhost:9001/feign-consumer

```
@RestController
public class ConsumerController {
    @Autowired
    private HelloService helloService;
    @RequestMapping(method = RequestMethod.GET,value = "/feign-consumer")
    public String helloConsumer(){
        return helloService.hello();
    }
}
```

