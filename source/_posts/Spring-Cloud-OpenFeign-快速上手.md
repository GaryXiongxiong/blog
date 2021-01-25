---
title: Spring Cloud OpenFeign 快速上手
date: 2020-09-07 10:20:04
tags:
- Spring Cloud
categories: 
- blog
---

## Feign是什么

Feign是一个声明式的web服务客户端。他允许开发者通过注解与接口实现简单快捷的http客户端创建。Spring Cloud OpenFeign在Feign的基础上加入了对SpringMVC注解的支持

## 快速开始

#### 引入Maven依赖

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

#### 启用Feign的客户端功能

```java
@SpringBootApplication
//在启动类上添加注解@EnableFeignClients
@EnableFeignClients
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

#### 声明需要的客户端接口

例如我们需要一个用于`stores`服务中`/stores`接口的客户端，那我们可以声明如下的接口：

```java
@FeignClient("stores")
public interface StoreClient {
    @RequestMapping(method = RequestMethod.GET, value = "/stores")
    List<Store> getStores();

    @RequestMapping(method = RequestMethod.GET, value = "/stores")
    Page<Store> getStores(Pageable pageable);

    @RequestMapping(method = RequestMethod.POST, value = "/stores/{storeId}", consumes = "application/json")
    Store update(@PathVariable("storeId") Long storeId, Store store);
}
```

之后我们接可以通过简单的自动注入获取到StoreClient的服务，并调用其中的方法：

```java
@Resource
StoreClient storeService;

List<Store> = storeService.getStores();
```

`getStroes()`方法实际上是向`stores/sotres`接口发送GET请求，并将请求结果映射成`List<Store>`返回，正如我们在`StoreClient`接口中声明的。

## 常用注解

#### SpringMVC相关注解

从快速开始的例子中可以看到，OpenFeign支持SpringMVC风格的注解，包括`GetMapping`,`PostMapping`,`RequestMapping`等等。需要注意的是，这里的注解对应的是服务端的接口信息。例如在快速开始中的` @RequestMapping(method = RequestMethod.POST, value = "/stores/{storeId}", consumes = "application/json")` 其中的consumes不是我们这个方法消费参数的格式，而是服务端消费请求的方式，即我们的客户端发送请求的方式。

#### @FeignClient注解

`FeignClient`注解是Feign中最常用的注解，可作用于接口上，用来声明此接口是一个Feign客户端，并声明该客户端调用服务的名称或url。该注解包含以下参数：

| 参数名            | 说明                                                         |
| ----------------- | ------------------------------------------------------------ |
| `value` / `name`  | 客户端的名称。不管是否提供`url`都必须明确该属性。如果项目使用了Ribbon，name属性会作为微服务的名称，用于服务发现。 |
| `qualifier`       | 定义客户端的qualifier值，对应`@Qualifier`注入方法。          |
| `url`             | 配置调用服务的绝对地址                                       |
| `decode404`       | boolean值，当调用请求404的时候，如果该参数为true，就会执行配置的Decoder进行解码。如果该参数为false直接抛出异常。 |
| `configuration`   | 指定Feign的配置类。可参考`org.springframework.cloud.netflix.feign.FeignClientsConfiguration`。 |
| `fallback`        | 定义fallback类，执zhi行熔断或请求失败后的容错处理。这种做法无法知道熔断的异常信息。样例实现见下文。 |
| `fallbackFactory` | 定义fallbackFactor类，执zhi行熔断或请求失败后的容错处理，可以知道熔断的异常信息。样例实现见下文。 |
| `path`            | 客户端访问接口地址的前缀。                                   |
| `primary`         | 对应`Primary`注解，标注该bean为高注入优先级。                |

#### fallback类

该类用于`fallback`参数，需实现对应的feignClient接口，例如对于此client：

```java
@FeignClient(value = "optimization-user", fallback = UserRemoteClientFallback.class)
public interface UserRemoteClient {

    @GetMapping("/user/get")
    public User getUser(@RequestParam("id")int id);
    
}
```

fallback类可写为：

```java
@Component
public class UserRemoteClientFallback implements UserRemoteClient {
    @Override
    public User getUser(int id) {
        return new User(0, "默认fallback");
    }
}
```

#### fallbackFactory类

fallbackFactory不同于fallback，它通过工厂模式生产一个实现了客户端接口的匿名内部类，并通过该工厂将熔断的异常信息传入该匿名内部类中。

```java
@FeignClient(value = "optimization-user", fallbackFactory = UserRemoteClientFallbackFactory.class)
public interface UserRemoteClient {

    @GetMapping("/user/get")
    public User getUser(@RequestParam("id")int id);

}
```

fallbackFactory类定义：

```java
@Component
public class UserRemoteClientFallbackFactory implements FallbackFactory<UserRemoteClient> {
    private Logger logger = LoggerFactory.getLogger(UserRemoteClientFallbackFactory.class);

    @Override
    public UserRemoteClient create(Throwable cause) {
        return new UserRemoteClient() {
            @Override
            public User getUser(int id) {
                logger.error("UserRemoteClient.getUser异常", cause);
                return new User(0, "默认");
            }
        };
    }
}
```

## References:
>
> - [Spring Cloud OpenFeign 官方文档](https://github.com/spring-cloud/spring-cloud-openfeign/blob/master/docs/src/main/asciidoc/spring-cloud-openfeign.adoc)
> - [那天晚上和@FeignClient注解的深度交流 from 猿天地](https://www.cnblogs.com/yinjihuan/p/12159986.html)