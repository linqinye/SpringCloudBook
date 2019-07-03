# Spring cloud Ribbon

Spring Cloud Ribbon 是一个基于HTTP和TCP的客户端负载均衡工具,它基于 Netflix Ribbon实现。通过 Spring Cloud 的封装，可以让我们轻松地将面向服务的 REST 模板请求自动转换成客户端负载均衡的服务调用。Spring Cloud Ribbon 虽然只是一个工具类框架，它不像服务注册中心、配置中心、API 网关那样需要独立部署，但是它几乎存在于每一个Spring Cloud构建的微服务和基础设施中。因为微服务间的调用，API 网关的请求转发等内容实际上都是通过Ribbon 来实现的，包括后续我们将要介绍的 Feign, 它也是基于 Ribbon实现的工具。 所以，对Spring Cloud Ribbon的理解和使用,对于我们使用 Spring Cloud 来构建微服务非常重要。

## 客户端负载均衡

客户端负载均衡和服务端负载均衡最大的不同点在于上面所提到的服务清单所存储的位置。 在客户端负载均衡中， 所有客户端节点都维护着自己要访问的服务端清单， 而这些服务端的清单来自于服务注册中心。

同服务端负载均衡的架构类似， 在客户端负载均衡中也需要心跳去维护服务端清单的健康性， 只是这个步骤
需要与服务注册中心配合完成。



## RestTemplate 详解

RestTemplat对象会使用 Ribbon 的自动化配置， 同时通过配置 @LoadBalanced 还能够开启客户端负载均衡。

### Get请求

#### 一：getForEntity 函数

该方法返回的是 ResponseEntity, 该对象是 Spring对 HTTP 请求响应的封装， 其中主要存储了 HTTP 的几个重要元素， 比如 HTTP 请求状态码的枚举对象 HttpStatus (也就是我们常说的 404 、500 这些错误码）、 在它的父类
HttpEntity 中还存储着 HTTP 请求的头信息对象 HttpHeaders 以及泛型类型的请求体对象。

```java
RestTemplate restTemplate=new RestTemplate();
ResponseEntity<String> responseEntity=restTemplate.getForEntity("http://USER-SERVICE/user?name={1}", String.class, "didi");
String body = responseEntity.getBody();
```

getForEntity函数实际上提供了以下三种不同的重载实现:

- getForEntity(String url, Class responseType, Object ... urlVariables)

  该方法提供了三个参数， 其中 url 为请求的地址， responseType 为请求响应体body 的包装类型，urlVariables 为 url 中的参数绑定。 

- getForEntity(String url, Class responseType, Map urlVariables)

  这里使用了 Map 类型， 所以使用该方法进行参数绑定时需要在占位符中指定 Map 中参数的 key 值

  ```java
  RestTemplate restTemplate = new RestTemplate();
  Map<String, String> params = new HashMap<>();
  params.put("name", "dada");
  ResponseEntity<String> responseEntity = restTemplate.getForEntity("http://USER-SERVICE/user?name={name}", String.class, params);
  ```

- getForEntity(URI url, Class responseType)

  该方法使用 URI 对象来替代之前的 url 和 urlVariables 参数来指定访问地址和参数绑定。 URI 是 JDKjava.net 包下的一个类，它表示一个统一资源标识符 (Uniform Resource Identifier)引用。

```java
RestTemplate restTemplate = new RestTemplate();
UriComponents uriComponents = UriComponentsBuilder.fromUriString(
"http://USER-SERVICE/user?name={name}")
.build()
.expand ("dodo")
.encode();
URI uri=uriComponents.toUri();
ResponseEntity<String> responseEntity = restTemplate.getForEntity(uri,String.class).getBody();
```

#### 二：getForObject 函数

该方法可以理解为对 getForEntity 的进 一 步封装，它通过 HttpMessageConverterExtractor 对 HTTP 的请求响应体 body 内容进行对象转换， 实现请求直接返回包装好的对象内容。

```java
RestTemplate restTemplate=new RestTemplate();
User result=restTemplate.getForObject(uri,User.class);
```

当不需要关注请求响应除 body 外的其他内容时， 该函数就非常好用， 可以少一个从Response 中获取 body 的步骤。 它与 getForEntity 函数类似， 也提供了三种不同的重载实现。

- getForObject (String url, Class responseType, Object. .. urlVariables)

  与 getForEntity 的方法类似， url 参数指定访问的地址， responseType 参数定义该方法的返回类型， urlVariables 参数为 url 中占位符对应的参数。

- getForObject(String url, Class responseType, Map urlVariables)

  在该函数中，使用 Map 类型的 urlVariables 替代上面数组形式的 urlVariables,因此使用时在 url 中需要将占位符的名称与 Map 类型中的 key一一对应设置。

- getForObject(URI url, Class responseType)

  该方法使用 URI 对象来替代之前的 url和urlVariables 参数使用。

### POST请求

#### 一：postForEntity 函数

```java
RestTemplate restTemplate = new RestTemplate();
User user=new User("didi",30);
ResponseEntity<String> responseEntity=restTemplate.postForEntity("http://USER-SERVICE/user", user, String.class);
String body=responseEntity.getBody();
```

- postForEntity(String url, Objec七 request, Class responseType,Object ... uri Variables)
- postForEntity(String url, Object request, Class responseType,Map uri Variables)
- postForEntity(URI url, Object request,Class responseType)

这里需要注意的是新增加的request参数， 该参数可以是一个普通对象， 也可以是一个HttpEntity对象。 如果是
一个普通对象， 而非HttpEntity对象的时候， RestTempla七e会将请求对象转换为一个HttpEntity对象来处理， 其中Object就是request的类型， request内容会被视作完整的body来处理；而如果request是一个HttpEntity对象， 那么就会被当作一个完成的HTTP请求对象来处理， 这个request中不仅包含了body的内容， 也包含header的内容。

#### 二：postForObject函数

作用是简化postForEntity的后续处理。 通过直接将请求响应的body内容包装成对象来返回使用

```java
RestTemplate rest Template=new RestTemplate();
User user= new User("didi", 20);
String postResult = restTemplate.postForObject("http://USER-SERVICE/user", user,String.class);
```

- postForObject(String url, Object request, Class responseType,Object ... uri Variables)
- postForObject(String url, Object request, Class responseType,Map uriVariables)
- postForObject(URI url, Object reques七， Class responseType)

#### 三：postForLocation函数

 该方法实现了以POST请求提交资源， 并返回新资源的URI。

```java
User user= new User("didi", 40);
URI responseURI = restTemplate.postForLocation("http:/ /USER-SERVICE/user", user);
```

- postForLocation (String url, Object request, Object ... url Variables)

- postForLocation(String url, Object request， Map urlVariables)

- postForLocation(URI url, Object request)

  由于 postForLocation函数会返回新资源的URI, 该URI就相当于指定了返回类型，所以此方法实现的POST请求不需要像postForEntity和postForObject那样指定responseType。

### PUT请求

```java
RestTemplate restTemplate=new RestTemplate();
Long id=100011;
User user=new User("didi", 40);
restTemplate.put("http://USER-SERVICE/user/{l}", user, id);
```

- put(String url, Object request, Object ... urlVariables)
- put(String url, Object request, Map urlVariables)
- put(URI url, Object request)

### DELETE请求

```java
RestTemplate restTemplate = new RestTemplate();
Long id= 10001L;
restTemplate.delete("http://USER-SERVICE/user/{1)", id);
```

- delete(String url, Object ... urlVariables)
- delete(String url, Map urlVariables)
- delete(URI url)

DELETE请求也不需要request的body信息,url指定DELETE请求的位置， urlVariables绑定url中的参数