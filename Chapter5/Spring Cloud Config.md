# Spring Cloud Config(分布式配置中心)

为分布式系统中的基础设施和微服务应用提供集中化的外部配置支持， 它分为服务端与客户端两个部分。 

- 服务端也称为分布式配置中心， 它是一个独立的微服务应用， 用来连接配置仓库并为客户端提供获取配置信息、加密／解密信息等访问接口；
- 客户端则是微服务架构中的各个微服务应用或基础设施， 它们通过指定的配置中心来管理应用资源与业务相关的配置内容，并在启动的时候从配置中心获取和加载配置信息。

Spring Cloud Config 实现了对服务端和客户端中环境变量和属性配置的抽象映射， 所以它除了适用于 Spring 构建的应用程序之外，也可以在任何其他语言运行的应用程序中使用。

Spring Cloud Config 实现的配置中心默认采用 Git 来存储配置信息， 所以使用 Spring Cloud Config 构建的配置服务器， 天然就支持对微服务应用配置信息的版本管理， 并且可以通过 Git 客户端工具来方便地管理和访问配置内容。 当然它也提供了对其他存储方式的支持。

## 快速入门

```java
	<dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-config-server</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-config</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
```

```java
@SpringBootApplication
@EnableConfigServer
public class ConfigServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApplication.class, args);
    }

}
```

```java
spring.application.name = config-server
server.port=7001

spring.cloud.config.server.git.uri = https://github.com/linqinye/cloud-config.git
spring.cloud.config.server.git.searchPaths = config-repo
spring.cloud.config.server.git.username = linqinye
spring.cloud.config.server.git.password = lin201542
```

其中 Git 的配置信息分别表示如下内容。
• spring.cloud.config.server.git.uri: 配置 Git 仓库位置。
• spring.cloud.config.server.git.searchPaths: 配置仓库路径下的相对搜索位置， 可以配置多个。
• spring.cloud.config.server.git.username: 访问 Git 仓库的用户名。
• spring.cloud.config.server.git.password: 访问 Git 仓库的用户密码。



## 配置规则详解

访问配置信息的URL与配置文件的映射关系如下所示：
• /{application}/{profile} [/{label}]
• /{application}-{profile}. yml
• /{label}/{application}-{profile}.yml
• /{application}-{profile}.properties
• /{label}/{application}-{profile}.properties

url会映射 {application}-{profile} .properties 对应的配置文件，其中 {label} 对应Git上不同的分支，默认master 。我们可以尝试构造不同的 url 来访问不同的配置内容， 比如， 要访问 config-label-test分支，didispace应用的 prod环境， 就可以访问这个 url: http://localhost:7001/didispace/prod/config-label-test

```java
{
    "name": "didispace",
    "profiles": [
        "prod"
    ],
    "label": "config-label-test",
    "version": "b2369e3cf49036896d15be8292764a508b7eb322",
    "state": null,
    "propertySources": [
        {
            "name": "https://github.com/linqinye/cloud-config.git/config-repo/didispace-prod.properties",
            "source": {
                "from": "git-prod-2.0"
            }
        },
        {
            "name": "https://github.com/linqinye/cloud-config.git/config-repo/didispace.properties",
            "source": {
                "from": "git-default-2.0"
            }
        }
    ]
}
```

该JSON中返回了应用名 didispace, 环境名 prod, 分支名 config-label-test,以及 default 环境和 prod 环境的配置内容。 另外， 之前没有提到过的 version, 从下图我们可以观察到， 它对应的是在 Git 上的 commit 号。

![01](.\images\config-01.png)

同时， 我们可以看到 config-server 的控制台中还输出了下面的内容，配置服务器在从 Git 中获取配置信息后， 会存储一份在 config-server 的文件系统中， 实质上config-server 是通过 git clone 命令将配置内容复制了一份在本地存储， 然后读取这些内容并返回给微服务应用进行加载。

![02](.\images\config-02.png)

config-server 通过 Git 在本地仓库暂存，可以有效防止当 Git 仓库出现故障而引起无法加载配置信息的情况。 我们可以通过断开网络， 再次发起http://localhost:7001/didispace/prod/config-label-test请求，在控制台中可输出 config-server 提示无法从远程获取该分支内容的报错信息： Could not pull remote for config-label-test, 但是它依然会为该请求返回配置内容， 这些内容源于之前访问时存于 config-server 本地文件系统中的配置内容。

## 客户端配置映射

创建项目

创建 bootstrap.properties 配置， 来指定获取配置文件的 config-server位置

```java
spring.application.name = didispace
spring.cloud.config.profile=dev
spring.cloud.config.label=master
spring.cloud.config.uri=http://localhost:7001/
server.port=7002
```

上述配置参数与 Git 中存储的配置文件中各个部分的对应关系如下所示。
• spring.application.name: 对应配置文件规则中的 {application} 部分。
• spring.cloud.config.profile:  对应配置文件规则中的 {profile} 部分。
• spring.cloud.config.label: 对应配置文件规则中的 {label} 部分。
• spring.cloud.config.uri:  配置中心 config-server 的地址。

**注意**

上面这些属性必须配置在 bootstrap.properties 中， 这样config-server 中的配置信息才能被正确加载。在第 2 章中，我们详细说明了 Spring Boot对配置文件的加载顺序， 对于本应用 jar 包之外的配置文件加载会优先于应用 jar 包内的配置内容， 而通过 bootstrap.properties 对 config-server 的配置， 使得该应用会从 config-server 中获取 一 些外部配置信息， 这些信息的优先级比本地的内容要高， 从而实现了外部化配置。

**千万不要引入**

```java
		<dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-config-server</artifactId>
        </dependency>
```

只需要引入

```java
		<dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-config</artifactId>
        </dependency>
```



- 创建一个 RESTful 接口来返回配置中心的 from 属性， 通过 @Value("${from}")绑定配置服务中配置的 from 属性

  ```java
  @RefreshScope
  @RestController
  public class TestController {
  
      @Value("${from}")
      private String from;
  
      @RequestMapping("/from")
      public String from(){
          return this.from;
      }
  }
  ```

- 除了通过@Value注解绑定注入之外， 也可以通过Environment对象来获取配置属性

  ```java
  @RefreshScope
  @RestController
  public class TestController {
      
      @Autowired
      private Environment environment;
  
      @RequestMapping("/from")
      public String from(){
          return environment.getProperty("from","underfined");
      }
  }
  ```

  启动config-client应用， 并访问http://localhost:7002/from, 我们就可以根据配置内容输出对应环境的from内容了。根据当前配置，我们可以获得如下返回内容：git-dev-1.0。 可以继续通过修改bootstrap.properties中的配置内容获取不同的配置信息来熟悉配置服务中的配置规则。

## 服务端详解

### 占位符配置URI

{application}、{profile}、{label}这些占位符除了用于标识配置文件的规则之外， 还可以用于ConfigServer 中对 Git 仓库地址的URI配置。 比如， 我们可以通过{application}占位符来实现 一 个应用对应一个Git仓库目录的配置效果.

```java
spring.cloud.config.server.git.uri = https://github.com/linqinye/cloud-config.git/{application}
spring.cloud.config.server.git.username = linqinye
spring.cloud.config.server.git.password = lin201542
```

- {application}代表了应用名， 所以当客户端应用向ConfigServer发起获取配置的请求时，ConfigServer会根据客户端的spring.application.name 信息来填充{application}占位符以定位配置资源的存储位置， 从而实现根据微服务应用的属性动态获取不同位置的配置。 
- 在这些占位符中， {label}参数较为特别 ， 如果Git的分支和标签名包含"/", 那么{label}参数在HTTP的URL中应该使用 "(_)" 来替代， 以避免改变了URI含义， 指向到其他的URI资源

### 本地仓库

文件都会在ConfigServer的本地文件系统中存储一份，这些文件默认会被存储于以config-repo为前缀的临时目录中， 比如名为/tmp/config-repo-<随机数＞的目录。 由于其随机性以及临时目录的特性， 可能会有一些不可预知的后果， 为了避免将来可能会出现的问题， 最好的办法就是指定一个固定的位置来存储这些重要信息。我们只需要通过spring.cloud.cofig.server.git.basedir或 spring.cloud.config.server.svn.basedir来配置一个我们准备好的目录即可。

### 健康监测

```java
spring.cloud.config.server.git.uri=http://git.oschina.net/didispace/{applicatio
n}-config
spring.cloud.config.server.git.username = username
spring.cloud.config.server.git.username = password
spring.cloud.config.server.health.repositories.check.name = check-repo
spring.cloud.config.server.health.repositories.check.label = master
spring.cloud.config.server.health.repositories.check.profiles = default
```

由千健康检测的 repositories 是个 Map 对象，所以实际使用时我们可以配置多个。
而每个配置中包含了与定位仓库地址时类似的三个元素。
• name:  应用名。
• label:  分支名。
• profiles:  环境名

通过使用 spring.cloud.config.server.health.enabled=false 参数设置来关闭



### 属性覆盖

只需要通过 spring.cloud.config.server.overrides 属性来设置键值对的参数，这些参数会以 Map 的方式加载到客户端的配置中

```java
spring.cloud.config.server.overrides.name = didi
spring.cloud.config.server.overrides.from = shanghai
```

通过该属性配置的参数，不会被 Spring Cloud 的客户端修改，并且 Spring Cloud 客户端从 Config Server 中获取配置信息时，都会取得这些配置信息。 利用该特性可以方便地为Spring Cloud 应用配置 一 些共同属性或是默认属性

### 安全保护

基于Spring boot,结合Spring Security

pom.xml添加依赖

```java
<dependency>
	<groupid>org.springframework.boot</groupid>
	<artifactid>spring-boot-starter-security</artifac七Id>
</dependency>
```

默认情况下，我们可以获得 一 个名为 user 的用户，并且在配置中心启动的时候，在日志中打印出该用户的随机密码.

大多数情况下，我们并不会使用随机生成密码的机制。 我们可以在配置文件中指定用户和密码

```java
security.user.name=user
security.user.password=37cc5635-559b-4e6f-b633-7e932b813f73
```

由于我们已经为 config-server 设置了安全保护，如果这时候连接到配置中心的客户端中**没有设置对应的安全信息**，在获取配置信息时会返回401错误。 所以，需要通过配置的方式在客户端中加入安全信息来通过校验

```java
spring.cloud.config.username=user
spring.cloud.config.password=37cc5635-559b-4e6f-b633-7e932b813f73
```

### 加密解密

如果我们直接将敏感信息以明文的方式存储于微服务应用的配置文件中是非常危险的。针对这个问题， Spring Cloud Config 提供了对属性进行加密解密的功能，以保护配置文件中的信息安全。

```java
spring.datasource.username = didi
spring.datasource.password={cipher}dba6505baa8ld78bd08799d8d4429de499bd4c2053c0
5f029e7cfbf143695f5b
```

在 Spring Cloud Config 中通过在属性值前使用 {cipher} 前缀来标注该内容是一个加密值，当微服务客户端加载配置时，配置中心会自动为带有 {cipher} 前缀的值进行解密。通过该机制的实现，运维团队就可以放心地将线上信息的加密资源给到微服务团队，而不用担心这些敏感信息遭到泄露了。

#### 前提

在使用 Spring Cloud Config 的加密解密功能时，有 一 个必要的前提需要我们注意。 为了启用该功能， 我们需要在配置中心的运行环境中安装不限长度的 JCE 版本 (UnlimitedStrength Java Cryptography Extension) 。 虽然， JCE 功能在JRE中自带，但是默认使用的是有长度限制的版本。 我们可以从Oracle的官方网站下载到它，它是 一 个压缩包，解压后可以看到下面三个文件

```java
README. txt
local_policy.jar
US_export_policy.jar
```

将 local_policy. jar 和 US _ export-policy. jar 两个文件复制到$JAVA HOME/jre/lib/security 目录下， 覆盖原来的默认内容。 到这里， 加密解密的准备工作就完成了。

## 服务端详解

### 服务化配置中心

#### 服务端配置

```java
		<dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
        </dependency>
```

```java
application.properties


spring.application.name = config-server
server.port=7001
eureka.client.serviceUrl.defaultZone=http://127.0.0.1:11112/eureka/
spring.cloud.config.server.git.uri = https://github.com/linqinye/cloud-config.git
spring.cloud.config.server.git.searchPaths = config-repo
spring.cloud.config.server.git.username = linqinye
spring.cloud.config.server.git.password = lin201542
```

```
@EnableDiscoveryClient
```

#### 客户端配置

```
	<dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
    </dependency>
```

```java
bootstrap.properties


spring.application.name=didispace
server.port=7002
eureka.client.serviceUrl.defaultZone=http://127.0.0.1:11112/eureka/
spring.cloud.config.profile=dev
spring.cloud.config.discovery.enabled=true
spring.cloud.config.discovery.serviceId=config-server
```

```
@EnableDiscoveryClient
```

#### 失败快速响应与重试

##### 快速响应

要实现客户端优先判断ConfigServer获取是否正常， 并快速响应失败内容， 只需在bootstrap.properties中配置参数spring.cloud.config.failFast= true即可

##### 自动重试

- 上述基础上进行,在客户端的 pom.xml 中增加 spring-retry 和 spring-boot-starter-aop依赖

```java
<dependency>
	<groupid>org.springframework.retry</groupid>
	<artifactid>spring-retry</artifactid>
</dependency>
<dependency>
	<groupid>org.springframework.boot</groupid>
	<artifactid>spring-boot-starter-aop</ar七ifactid>
</dependency>
```

- 不需要再做其他任何配置， 启动客户端应用， 在控制台中可以看到如下内容。 客户端在连接 Config Server 失败之后， 会继续尝试， 直到第 6 次失败后， 才返回错误信息。 通过这样的重试机制， 可以避免 一 些间歇性问题引起的失败导致客户端应用无法启动的情况。
- 若对默认的最大重试次数和重试间隔等设置不满意，还可以通过下面的参数进行调整
  - spring.cloud.config.retry.multiplier: 初始重试间隔时间（单位为毫秒），默认为 1000 毫秒。
  - spring.cloud.config.retry.initial-interval: 下 一 间隔的乘数，默认为 1.1, 所以当最初间隔是 1000 毫秒时， 下 一 次失败后的间隔为 1100 毫秒。
  - spring.cloud.config.retry.max-interval: 最大间隔时间，默认为 2000毫秒。
  - spring.cloud.config.retry.max-attempts: 最大重试次数，默认为 6 次。

##### 动态刷新配置

- 在config-client的pom.xml中新增spring-boot-starter-actuator监控模块。 其中包含了/refresh端点的实现，该端点将用于实现客户端应用配置信息的重新获取与刷新

```java
<dependency>
	<groupid>org.springframework.boot</groupid>
	<artifactid>spring-boot-starter-actuator</artifactid>
</dependency>

需要在配置的页面加上，就是说附带@Value的页面加上@RefreshScope


注意：config-server和config-client的配置都得加上
management.endpoints.web.exposure.include=bus-refresh（只暴露bus-refresh节点）
```

- 重新启动config-client ， 访问一次http://localhost:7002/frorn, 可以看到当前的配置值。
- 修改Git仓库config-repo/didispace-dev.properties文件中from的值。
- 再访问一次http://localhost7002/frorn, 可以看到配置值没有改变。
- 通过POST请求发送到请求刷新的页面http://localhost:7002/actuator/bus-refresh刷新配置
- 再访问一次http://localhost7002/from, 可以看到配置值已经是更新后的值