# Spring Cloud Zuul(API网关)

## 快速入门

### 1.api-gateway

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-zuul</artifactId>
</dependency>
```

zuul包含下列依赖：

- spring-cloud-starter-hystrix： 该依赖用来在网关服务中实现对微服务转发时候的保护机制，通过线程隔离和断路器，防止微服务的故障引发API网关资源无法释放， 从而影响其他应用的对外服务。
- spring-cloud-starter-Ribbon: 该依赖用来实现在网关服务进行路由转发时候的客户端负载均衡以及请求重试。
- spring-boot-starter-actuator: 该依赖用来提供常规的微服务管理端点。 另外， 在Spring Cloud Zuul中还特别提供了/routes端点来返回当前的所有路由规则。

### 2.@EnableZuulProxy注解开启Zuul的API网关服务功能

### 3.application.properties中配置Zuul应用的基础信息

```
spring.application.name = api-gateway
server.port = 5555
```

### 请求路由

- 为了与Eureka整合， 我们需要在 api-gateway 的 pom.xml 中引入spring­cloud-starter-eureka依赖
- 在api-gateway的application.properties 配置文件中指定Eureka注册中心的位置， 并且配置服务路由.

```
zuul.routes.api-a.path=/api-a/**
zuul.routes.api-a.serviceId=hello-service
zuul.routes.api-b.path=/api-b/**
zuul.routes.api-b.serviceId=feign-consumer
eureka.client.serviceUrl.defaultZone=http://localhost:11112/eureka/
```

通过上面的搭建工作， 我们已经可以通过服务网关来访问 hello-service 和feign-consumer这两个服务 了。 根据配置的映射关系， 分别向网关发 起下面这些请求。
• http://localhost:5555/api-a/hello: 该 url符合/api-a/**规则， 由api-a路由负责转发，该路由映射的serviceId为hello-service, 所以最终/hello请求会被发送到hello-service服务的 某个 实例上去。
• http://localhost:5555/api-b/feign-consumer: 该url符合/api-b/**规则，由 api-b路由负责转发，该路由映射的serviceId为feign-consumer,所以最终/feign-consumer请求会被发送到feign-consumer服务的某个实例上去.

### 请求过滤

- 剥离出校验逻辑，构建独立的鉴权服务
- 前置网关完成非业务性质的校验
- 在请求到达的时候就完成校验和过滤，而不是转发后再过滤而导致更长的请求延迟
- 通过在网关中完成校验和过滤， 微服务应用端就可以去除各种复杂的过滤器和拦截器了

Zuul 允许开发者在API网关上通过定义过滤器来实现对请求的拦截与过滤，实现的方法非常简单，我们只需要继承 ZuulFilter抽象类并实现它定义的4个抽象函数就可以完成对请求的拦截和过滤了.

**实现了在请求被路由之前检查HttpServletRequet中是否有 accessToken 参数， 若有就进行路由， 若没有就拒绝访问， 返回 401 Unauthorized 错误**

```
@Slf4j
public class AccessFilter extends ZuulFilter {
    @Override
    public String filterType() {
        return "pre";
    }

    @Override
    public int filterOrder() {
        return 0;
    }

    @Override
    public boolean shouldFilter() {
        return true;
    }

    @Override
    public Object run() throws ZuulException {
        RequestContext ctx = RequestContext.getCurrentContext();
        HttpServletRequest request = ctx.getRequest();
        log.info("send {} request to{}", request.getMethod () ,
                request.getRequestURL().toString());
        Object accessToken = request.getParameter("accessToken");
        if(accessToken ==  null) {
            log.warn("access token is empty");
            ctx.setSendZuulResponse(false);
            ctx.setResponseStatusCode(401);
            return null;
        }
            log.info("access token ok");
            return null;
    }
}
```

在上面实现的过滤器代码中，我们通过继承ZuulFilter抽象类并重写下面4个方法来实现自定义的过滤器。 这4个方法分别定义了如下内容。

- **FilterType: 过滤器的类型**， 它决定过滤器在请求的哪个生命周期中执行。 这里定义为pre, 代表会在请求被路由之前执行。
- **FilterOrder: 过滤器的执行顺序**。 当请求在一个阶段中存在多个过滤器时，需要根据该方法返回的值来依次执行。
- **shouldFilter: 判断该过滤器是否需要被执行**。 这里我们直接返回了true, 因此该过滤器对所有请求都会生效。 实际运用中我们可以利用该函数来指定过滤器的有效范围。
- **run: 过滤器的具体逻辑**。 这里我们通过ctx.setSendZuulResponse(false)令zuul过滤该请求， 不对其进行路由， 然后通过 ctx.setResponseStatus­Code (401)设置了其返回的错误码，当然也可以进一步优化我们的返回，比如，通过ctx.setResponseBody(body)对返回的body内容进行编辑等.

实现了自定义过滤器之后， 它并不会直接生效， 我们还需要为其创建具体的Bean才能启动该过滤器， 比如， 在应用主类中增加如下内容

```
@SpringBootApplication
@EnableZuulProxy
@EnableDiscoveryClient
public class ApiGatewayApplication {

    public static void main(String[] args) {
        SpringApplication.run(ApiGatewayApplication.class, args);
    }

    @Bean
    public AccessFilter accessFilter(){
        return new AccessFilter();
    }

}
```

在对api-gateway服务完成了上面的改造之后， 我们可以重新启动它， 并发起下面的请求， 对上面定义的过滤器做 一 个验证。
• http://localhost:5555/api-a/hello: 返回401错误。
• http://localhost:5555/api-a/hello&accessToken = token: 正确路由到hello-service的/hello接口， 并返回Hello World。



**总结**

- 它作为系统的统一入口，屏蔽了系统内部各个微服务的细节。
- 它可以与服务治理框架结合，实现自动化的服务实例维护以及负载均衡的路由转发。
- 它可以实现接口权限校验与微服务业务逻辑的解耦。
- 通过服务网关中的过炖器， 在各生命周期中去校验请求的内容， 将原本在对外服务层做的校验前移， 保证了微服务的无状态性， 同时降低了微服务的测试难度， 让服务本身更集中关注业务逻辑的处理

## 路由详解

### 传统路由方式

#### 单实例配置

```java
zuul.routes.api-a-url.path=/api-a-url/**
zuul.routes.api-a-url.url=http://localhost:8080/
```

该配置定义了发往API网关服务的请求中， 所有符合/api-a-url/**规则的访问都将被路由转发到http://localhost:8080/地址上，也就是 说，当我们访问http://localhost:5555/api-a-url/hello的时候，API网关服务会将该请求路由到http://localhost:8080/hello提供的微服务接口上。 其中 ，配置属性zuul.routes.api-a-url.path 中的api-a-url部分为路由的名字，可以任意定义，但是`一组path和url映射关系的路由名要相同`，面向服务的映射方式也是如此。

#### 多实例配置

```java
zuul.routes.user-service.path = /user-service/**
zuul.routes.user-service.serviceid=user-service
ribbon.eureka.enabled = false
user-service.ribbon.listOfServers = http://localhost:8080/,http://localhost:8081/
```

该 配 置 实 现了对 符 合 /user-service/** 规 则 的 请 求 路 径 转 发 到http://localhost:8080/和http://localhost:8081/两个实例地址的路由规则。 它的配置方式与服务路由的配置 方式一 样，都采用了zuul.routes.
< route>.path与zuul.routes. < route>.serviceId参数对的映射方式，只是这里的 serviceid 是由用户手工命名 的服务名称 ，配 合ribbon.listOfServers 参数实现服务与实例的维护。由于存在多个实例，API网关在进行路由转发时需要实现负载均衡策略，于是这里还需要Spring Cloud Ribbon的配合。由千在Spring Cloud Zuul中自带了对Ribbon的依赖， 所以我们只需做 一 些配置即可。

- ribbon.eureka.enabled:由于zuul.routes. < route>.serviceid指定的是服务名称，默认清况下Ribbon会根据服务发现机制来获取配置服务名对应的实例清单。 但是，该示例并没有整合类似Eureka之类的服务治理框架，所以需要将该参数设置为false, 否则配置的serviceId获取不到对应实例的清单.
- user-service.ribbon.listOfServers: 该 参数内容与 zuul.routes.<route>.serviceid 的配置相对 应， 开头的 user-service 对应了serviceId的值， 这两个参数的配置相当于在该应用内部手工维护了服务与实例的对应关系.

**不论是单实例还是多实例的配置方式，我们都需要为每一对映射关系指定一个名称，也就是上面配置中的 <route>, 每一个<route>对应了一条路由规则。 每条路由规则都需要通过 path 属性来定义一个用来匹配客户端请求的路径表达式， 并通过 url 或serviceId属性来指定请求表达式映射具体实例地址或服务名。**

### 面向服务的路由

传统路由运维修改配置麻烦，可以让路由的path不是映射具体的url, 而是让它映射到某个具体的服务 ，而具体的url则交给Eureka的服务发现机制去自动维护，我们称这类路由为面向服务的路由。

**zuul.routes.<route>.path与zuul.routes.<route>.serviceid参数对的方式进行配置, 其中<route>可以指定为任意的路由名称**

```java
zuul.routes.user-service.path=/user-service/**
zuul.routes.user-service.serviceid=user-service
```

对千面向服务的路由配置，更简洁的配置方式：zuul.routes.<serviceid>=<path>, 其中<serviceid>用来指定路由的具体服务名，<path>用来配置匹配的请求表达式

```java
zuul.routes.user-service=/user-service/**
```

当采用path与serviceId以服务路由的方式实现时，在没有配置任何实例地址的情况下，外部请求经过API网关的时候， 它是如何被解析并转发到服务具体实例的呢？

***答***

Zuul巧妙地整合了Eureka 来实现面向服务的路由。实际上，我们可以直接将API网关也看作Eureka服务治理下的
一个普通微服务应用。 它除了会将自己注册到Eureka服务注册中心上之外，也会从注册中心获取所有服务以及它们的实例清单。所以，在Eureka的帮助下，API网关服务本身就已经维护了系统中所有 serviceId与实例地址的映射关系。当有外部请求到达 API 网关的时候，根据请求的URL路径找到最佳匹配的path规则， API 网关就可以知道要将该请求路由到哪个具体的serviceId上去。由于在 API 网关中已经知道serviceid对应服务实例的地址清单，那么只需要通过Ribbon的负载均衡策略，直接在这些清单中选择一个具体的实例进行转发就能完成路由工作了.

### 服务路由的默认规则

对于具有规则性的配置内容， 我们总是希望可以自动化地完成。 非常庆幸，Zuul默认实现了这样的贴心功能， 当我们为Spring Cloud Zuul构建的 API 网关服务引入Spring Cloud Eureka之后， 它为Eureka中的每个服务都自动创建一个默认路由规则， 这些默认规则的path会使用serviceId配置的服务名作为请求前缀。

由千默认情况下所有Eureka上的服务都会**被Zuul自动地创建映射关系**来进行路由，这会使得 一 些我们不希望对外开放的服务也可能被外部访问到。 这个时候， 我们可以使用**zuul.ignored-services参数来设置**一个服务名匹配表达式来定义**不自动创建路由的规则**。 Zuul在自动创建服务路由的时候会根据该表达式来进行判断， 如果服务名匹配表达式 ， 那 么 Zuul 将跳过该服 务 ，不为其创建路由规则 。比如，设 置 为zuul.ignored-services=*的时候，Zuul将对所有的服务都不自动创建路由规则。 在这种情况下，我们就要在配置文件中逐个为需要路由的服务添加映射规则（可以使用path与 serviceId组合的配置方式， 也可使用更简洁的 zuul.routes.<serviceid>=<path>配置方式），只有在配置文件中出现的映射规则会被创建路由，而从Eureka中获取的其他服务，Zuul将不会再为它们创建路由规则。

### 自定义路由映射规则

我们在构建微服务系统进行业务逻辑开发的时候， 为了兼容外部不同版本的客户端程序（尽量不强迫用户升级客户端），一 般都会采用开闭原则来进行设计与开发。这使得系统在迭代过程中，有时候会需要我们为一组互相配合的微服务定义一个版本标识来方便管理它们的版本关系，根据这个标识我们可以很容易地知道这些服务需要一起启动并配合使用。比如可以采 用 类 似这 样 的 命 名：userservice-vl 、userservice-v2、orderservice-vl、orderservice-v2。默认情况下，Zuul自动为服务创建的路由表达式会采用服务名作为前缀， 比如针对上面的userservice-vl 和userservice-v2,它会产生/userservice-vl 和/userservice-v2两个路径表达式来映射，但是这样生
成出来的表达式规则较为单 一 ，不利于通过路径规则来进行管理。 通常的做法是为这些不同版 本的 微 服 务 应 用 生 成以版 本代号 作 为 路 由 前 缀 定 义 的 路 由 规 则 ， 比如/vl/userservice/ 。 这时候， 通过这样具有版本号前缀的 URL 路径，我们就可以很容易地通过路径表达式来归类和管理这些具有版本信息的微服务了。

对上面所述的需求，如果我们的各个微服务应用都遵循了类似userservice-vl这样的命名规则，通过－分隔的规范来定义服务名和服务版本标识的话，那么，我们可以使用Zuul中自定义服务与路由映射 关系的功能，来实现为符合上述规则的微服务自动化地创建类似/vl/userservice/** 的路由匹配规则。 实现步骤非常简单， 只需在 API 网关程序中， 增加如下Bean的创建即可：

```java
@Bean
public PatternServiceRouteMapper serviceRouteMapper () {
	return new PatternServiceRouteMapper(
		"(?<name>^.+)-(?<version>v.+$)",
		"${version}/${name}");
}

```

PatternServiceRouteMapper对象可以让开发者**通过正则表达式来自定义服务与路由映射的生成关系**。 其中构造函数的第一个参数是用来匹配服务名称是否符合该自定义规则的正则表达式， 而第二个参数则是定义根据服务名中定义的内容转换出的路径表达式规则。当开发者在 API 网关中定义了PatternServiceRouteMapper 实现之后，只要符合第一个参数定义规则的服务名， 都会优先使用该实现构建出的路径表达式，如果没有匹配上的服务则还是会使用默认的路由映射规则，即采用完整服务名作为前缀的路径表达式。

### 路径匹配

在Zuul中，路由匹配的路径表达式采用了Ant风格定义。

Ant风格的路径表达式使用起来非常简单，它 一 共有下面这三种通配符。

| ？   | 匹配任意单 个字符                 |
| ---- | --------------------------------- |
| *    | 匹配任意数朵的字符                |
| **   | 匹配任意数址的字符， 支待多级目录 |

当我们使用通配符的时候，经常会碰到这样的问题：一个URL 路径可能会被多个不同路由的表达式匹配上。 比如，有这样一个场景，我们在系统建设的一开始实现了user-service 服务

```java
zuul.routes.user-service.path=/user-service/**
zuul.routes.user-service.serviceid=user-service
```

随着版本的迭代， 我们对 user-service 服务做了一些功能拆分， 将原属于user-service 服务的某些功能拆分到了另外 一 个全新的服务 user-service-ext 中去，而这些拆分的外部调用 URL 路径希望能够符合规则 /userservice/ext/**, 这个时候我们需要就在配置文件中增加一个路由规则

```java
zuul.routes.user-service.path=/user-service/**
zuul.routes.user-service.serviceid=user-service
zuul.routes.user-service-ex七.path=/user-service/ext/**
zuul.routes.user-service-ext.serviceid=user-service-ext
```

此时， 调用 user-service-ext服务的 URL 路径实际上会同时被 /user­service/** 和 /user-service/ext/** 两个表达式所匹配。 在逻辑上， API 网关服务需要优先选择 /user-service/ext/** 路由，然后再匹配 /user-service/** 路由才能实现上述需求。但是如果使用上面的配置方式，实际上是无法保证这样的路由优先顺序的。

下面的路由匹配算法中， 我们可以看到它在使用路由规则匹配请求路径的时候是通过线性遍历的方式，在请求路径获取到第 一 个匹配的路由规则之后就返回并结束匹配过程。所以当存在多个匹配的路由规则时， 匹配结果完全取决于路由规则的保存顺序。

```java
@Override
public Route getMatchingRoute{final String path) {
	ZuulRoute route= null;
	if (!matchesignoredPatterns(adjustedPath)) {
		for (Entry<String, ZuulRoute> entry : this. routes. get () . entrySet ()) {
			String pattern= entry. getKey () ;
			log.debug("Matching pat七ern:" + pattern);
				if (this.pathMatcher.match(pattern, adjustedPa七h)) {
					route = entry.getValue();
					break;
                }
        }
		log.debug("route matched=" + route);
		return getRoute(route, adjustedPath);
}
```

下面所示的代码是基础的路由规则加载算法， 我们可以看到这些路由规则是通过LinkedHashMap保存的， 也就是说， 路由规则的保存是有序的， 而内容的加载是通过遍历配置文件中路由规则依次加入的，所以导致问题的根本原因是对配置文件中内容的读取。

```java
protected Map<String, ZuulRoute> locateRoutes() {
	LinkedHashMap<String, ZuulRoute> routesMap = new LinkedHashMap<String,ZuulRoute>();
	for (ZuulRoute route : this.properties.getRoutes() .values()) {
		routesMap.put(route.getPath(), route);
    }
    return routesMap;
}
```

由千properties的配置内容无法保证有序，所以当出现这种情况的时候， 为了保证路由的优先顺序， 我们需要使用YAML文件来配置， 以实现有序的路由规则， 比如使用下面的定义

```yaml
zuul:
	routes:
		user-service-ext:
			path: /user-service/ext/**
			serviceld: user-service-ext
		user-service:
			path: /user-service/**
			serviceld: user-service
```

***忽略表达式***

通过 path 参数定义的 Ant 表达式已经能够完成 API 网关上的路由规则配置功能，但是为了更细粒度和更为灵活地配置路由规则， Zuul 还提供了一个忽略表达式参数zuul.ignored-patterns 。 该参数可以用来设置不希望被 API 网关进行路由的 URL 表达式。该参数在使用时还需要注意它的范围并不是对某个路由， 而是对所有路由。 所
以在设置的时候需要全面考虑 URL 规则， 防止忽略了不该被忽略的 URL 路径

不希望 /hello 接口被路由

```java
zuul.ignored-patterns = /**/hello/**
zuul.routes.api-a.path=/api-a/**
zuul.routes.api-a.serviceid=hello-service
```

通过网关来访间 hello-service 的 /hello 接口 http://localhost:5555/api-a/hello 。 虽然该访问路径完全符合 path 参数定义的/api-a/** 规则，但是由于该路径符合 zuul.ignored-patterns 参数定义的规则，所以不会被正确路由。 同时，我们在控制台或日志中还能看到没有匹配路由的输出信息

```java
o.s.c.n.z.f.pre.PreDecorationFilter  : No route found for uri: /api-a/hello\
```

### 路由前缀

方便全局地为路由规则增加前缀信息， Zuul 提供了 zuul.prefix 参数来进行设置。比如，希望为网关上的路由规则都增加 /api 前缀，那么我们可以在配置文件中增加配置： zuul.prefix = /api 。 另外， 对于代理前缀会默认从路径中移除，我们可以通过设置zuul.stripPrefix = false 来关闭该移除代理前缀的动作，也可以通过 zuul.routes.
<route>.strip-prefix = true 来对指定路由关闭移除代理前缀的动作。

***注意， 在使用 zuul.prefix 参数的时候， 目前版本的实现还存在一些 Bug, 所以请谨慎使用， 或是避开会引发 Bug 的配置规则。 具体会引发 Bug 的规则如下***

- 假设我们设置 zuul.prefix = /api, 当路由规则的 path 表达式以 /api 开头的时候，将会产生错误的映射关系.务必避免让路由表达式的起始字符串与 zuul.prefix 参数相同

### 本地跳转

在 Zuul 实现的 API 网关路由功能中， 还支持 forward 形式的服务端跳转配置。 实现方式非常简单，只需通过使用 path 与 url 的配置方式就能完成，通过 url 中使用 forward来指定需要跳转的服务器资源路径.

```java
zuul.routes.api-b.path=/api-b/**
zuul.routes.api-b.url=forward:/local
```

 当 API 网关接收到请求 /api-b/hello, 它符合 api-b 的路由规则，所以该请求会被 API 网关转发到网关的 /local/hello 请求上进行本地处理。

**注意**， 由于需要在 API 网关上实现本地跳转，所以相应的我们也需要为本地跳转实现对应的请求接口。 按照上面的例子， 在 API 网关上还需要增加 一 个/ local/hello的接口实现才能让 api-b 路由规则生效， 比如下面的实现。否则 Zuul 在进行 forward 转发的时候会因为找不到该请求而返回404错误。

```java
@RestController
public class HelloController {
	@RequestMapping ("/local /hello")
	public String hello() {
		return "Hello World Local";
	}
}
```

### Cookie和头信息

默认情况下， SpringCloud Zuul在请求路由时， 会过滤掉HTTP请求头信息中的一 些敏感 信 息， 防止它们被传 递到下游的外部服 务器。 默 认的 敏感 头 信 息 通 过zuul.sensitiveHeaders参数定义，包括Cookie、Set-Cookie、Authorization三个属性。

通过指定路由的参数来配置， 方法有下面两种。

```java
＃方法一：对指定路由开启自定义敏感头
zuul.routes.<router>.customSensitiveHeaders = true
＃方法二：将指定路由的敏感头设置为空
zuul.routes.<router>.sensitiveHeaders =
```

仅对指定的Web应用开启对敏感信息的传递，影响范围小， 不至于引起其他服务的信息泄露间题。

***重定向问题***

虽然可以通过网关访问登录页面并发起登录请求， 但是登录成功之后， 我们跳转到的页面URL却是具体Web应用实例的地址， 而不是通过网关的路由地址。 这个问题非常严重， 因为使用API网关的一个重要原因就是要将网关作为统一入口，从而不暴露所有的内部服务细节。

引起问题的大致原因是由于SpringSecurity或Shiro在登录完成之后，通过重定向的方式跳转到登录后的页面，此时登录后的请求结果状态码为302, 请求响应头信息中的 Location指向了具体的服务实例地址， 而请求头信息中的Host也指向了具体的服务实例 IP地址和端口。 所以，该问题的根本原因在于Spring Cloud Zuul在路由请求时，并没有将最初的Host信息设置正确。

## 过滤器详解

### 过滤器

Zuul包含了对请求的路由和过滤两个功能，其中路由功能负责将外部请求转发到具体的微服务实例上，是实现外
部访问统一入口的基础；而过滤器功能则负责对请求的处理过程进行干预，是实现请求校验、 服务聚合等功能的基础。 然而实际上，路由功能在真正运行时，它的路由映射和请求转发都是由几个不同的过滤器完成的。 其中，路由映射主要通过pre类型的过滤器完成，它将请求路径与配置的路由规则进行匹配，以找到需要转发的目标地址；而请求转发的部分则是由route类型的过滤器来完成，对pre类型过滤器获得的路由地址进行转发。 所以，过滤器可以说是Zuul实现API网关功能最为核心的部件，每 一 个进入Zuul的HTTP请求都会经过 一 系列的过滤器处理链得到请求响应并返回给客户端。

在Spring Cloud Zuul中实现的过滤器必须包含***4个基本特征： 过滤类型、 执行顺序、执行条件、 具体操作***

### 请求生命周期

![01](.\images\zuul-01.png)

### 核心过滤器

![02](.\images\zuul-02.png)

### 异常处理

- **SendErrorFilter 是用来处理异常信息**

error.status_code 参数就是 SendErrorFilter 过滤器用来判断是否需要执行的重要参数

- **对于意外抛出的异常又会导致没有控制台输出也没有任何响应信息的情况出现**,**可以通过创建 一 个 error 类型的过滤器来捕获这些异常信息，并根据这些异常信息在请求上下文中注入需要返回给客户端的错误描述**

### 禁用过滤器

不论是核心过滤器还是自定义过滤器， 只要在API网关应用中为它们创建了实例， 那么默认情况下， 它们都是启用状态的。 那么如果有些过滤器我们不想使用了， 如何禁用它们呢？大多情况下初识Zuul的使用者第 一 反应就是通过重写shouldFilter逻辑， 让它返回false, 这样该过滤器对于任何请求都不会被执行， 基本实现了对过滤器的禁用。 但是， 对于自定义过滤器来说似乎是实现了过滤器不生效的功能， 但是这样的做法缺乏灵活性。 由于直接要修改过旃器逻辑， 我们不得不重新编译程序， 并且如果该过滤器在未来一段时间还有可能被启用的时候， 那么就又得修改代码并编译程序。 同时， 对于核心过滤器来说， 就更为麻烦， 我们不得不获取源码来进行修改和编译。

Zuul中特别提供了一个参数来禁用指定的过滤器

```java
zuul.<SimpleClassName>.<filterType>.disable = true
```

 <SimpleClassName>代表过滤器的类名， 比如快速入门示例中的AccessFilter;＜filterType>代表过滤器类型， 比如快速入门示例中 AccessFilter 的过滤器类型pre。 所以， 如果我们想要禁用快速入门示例中的 AccessFilter 过滤器， 只需要在application.properties 配置文件中增加如下配置即可：

```java
zuul.AccessFilter.pre.disable = true
```

该参数配置除了可以对自定义的过滤器进行禁用配置之外， 很多时候可以用它来禁用Spring Cloud Zuul中默认定义的核心过滤器。 这样我们就可以抛开Spring Cloud Zuul自带的那套核心过滤器， 实现 一 套更符合我们实际需求的处理机制。

## 动态加载

在微服务架构中， 由于API网关服务担负着外部访问统一入口的重任， 它同其他应用不同， 任何关闭应用和重启应用的操作都会使系统对外服务停止， 对于很多7 X  24小时服务的系统来说， 这样的情况是绝对不被允许的。 所以， 作为最外部的网关， 它必须具备动态更新内部逻辑的能力， 比如动态修改路由规则、 动态添加／删除过滤器等。

我们可以在不重启 API网关服务的前提下， 为其动态修改路由规则和添加或删除过滤器。 下面我们分别来看看如何通过Zuul来实现动态API网关服务。

### 动态路由

只需将API网关服务的配置文件通过Spring Cloud Config连接的Git仓库存储和管理， 我们就能轻松实现动态刷新路由规则的功能。

### 动态过滤器