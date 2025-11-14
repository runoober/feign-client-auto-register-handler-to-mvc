# feign-client-auto-register-handler-to-mvc

[![LICENSE](https://img.shields.io/badge/license-Anti%20996-blue.svg)](https://github.com/996icu/996.ICU/blob/master/LICENSE)
[![License](http://img.shields.io/:license-apache-brightgreen.svg)](http://www.apache.org/licenses/LICENSE-2.0.html)

## 前言
目前公司后端服务项目结构为，xxx-api、xxx-service 其中 api模块定义了此微服务模块对外暴露的接口，service模块实现了api模块定义的接口并创建了对应供调用的Controller。
所以实际来说Controller就是一个代理类，用来代理实现Service模块真正对外的HTTP请求响应。那么我们是不是可以通过生成Controller代理类然后注入到容器的方式来代替手写Controller来自动实现这些代理类了？

## 实现思路
能不能实现了？显然是可以的，实现的首要条件是：
1. 请求规则如何定义
2. 如何注入Controller到Spring IOC容器中，下面我们来一一解答。

请求规则如何定义？

我们在使用微服务调用时，实际都是通过Feign来进行实际使用的，所以请求路径实际上来说在基于Feign的接口上都已经通过注解标明了信息。如：
```java
@FeignClient(value = "user", path = "user") //等同于Controller上的 @RequestMapping("user")
public interface UserService{

    @GetMapping("findById") 
    Response<User> findById(@RequestParam("id") Long id);   

}
```

如何注入Controller到Spring IOC容器中？

这里我们首先要了解Spring MVC的运行机制，实际的请求处理是通过DispatcherServlet映射给注册到HandlerMapping中的Handler进行处理的，这个时候我们可以发现通过注册Handler到HandlerMapping 即可实现这个注入逻辑了。
在Spring Boot中默认会注册七个HandlerMapping，我们只需要关注 RequestMappingHandlerMapping 和 SimpleUrlHandlerMapping，更详细的内容可自行查看 WebMvcAutoConfiguration 类。
因为接口定义的关系这里我们只能通过注册 Handler 到 RequestMappingHandlerMapping 来实现注册逻辑。

如何注册 Handler 到 RequestMappingHandlerMapping？

这里一个核心的问题就是Spring MVC默认是通过获取类和方法的注解信息创建 RequestMappingInfo 然后将 RequestMappingInfo、Handler实例以及方法注册到 HandlerMapping 中，他实际的过程是全自动的，通过获取IOC容器中注解了 @Controller 和 @RequestMapping 的Bean，直接解析然后注册进去(AbstractHandlerMethodMaping#detectHandlerMethods)，我们要通过动态代理生成对应Bean，那么就没有了对应的类层级的 @RequestMapping 注解了，所以通过手动创建 RequestMappingInfo 然后手动调用方法注册的方式进行注册了(AbstractHandlerMethodMaping#registerHandlerMethod)。

## 方案
### 方案1，手动生成信息注册的方案：

生成 RequestMappingInfo 信息
获取 @FeignClient 的 path 信息生成 RequestMappingInfo
获取 @FeignClient 注解接口的方法注解信息生成 RequestMappingInfo
合并上列 RequestMappingInfo，生成一个完整的 RequestMappingInfo
通过 @FeignClient 注解的接口和此接口的实现类，生成动态代理类
遍历实现类方法和接口方法将其缓存到映射中，然后生成一个 InvocationHandler 对象
通过 Proxy.newProxyInstance 生成代理类
通过反射调用 RequestMappingHandlerMapping#registerHandlerMethod 注册Handler
### 方案2，通过继承 RequestMappingInfoHandlerMapping 的方式来实现：

覆盖 isHandler 方法，注解了 @FeignClient 接口的实现类即为 Handler
覆盖 getMappingForMethod 方法，通过 @FeignClient 注解信息和接口中方法对应信息生成 RequestMappingInfo
将该 HandlerMapping 注册到IOC容器中，此时就完成了整个功能了。
这两个方案都要注意扫描范围，不然会将引入的模块也生成了Handler。
方案2比较简单，花了一点点时间实现了方案2，目前已应用在项目中，源码见此地址。

应用Spring Cloud微服务之后针对每个feign api模块除了实现其Service外，还需要提供对应的代理Controller以提供HTTP请求响应，所以此库提供了自动的Feign Service注入URL请求映射功能，无需编写代理Controller。

## 如何使用

在 SpringApplication 启动类上配置`@EnableFeignAutoHandlerRegister`注解即可，代码如下

```java
@EnableDiscoveryClient
@EnableFeignClients(basePackages = {"com.liying","com.jq"}, defaultConfiguration = FeignConfig.class)
@SpringBootApplication(scanBasePackages = {"com.liying","com.jq"})
@EnableFeignAutoHandlerRegister({"com.liying.game"})
public class SpringApplication {

	public static void main(String[] args) {
		new SpringApplicationBuilder(SpringApplication.class).run(args);
	}
}
```

## 注意：
1. `@EnableFeignAutoHandlerRegister`注解需提供完整的扫描包路径，否则可能会将其他导入的api模块也注入URL映射。
2. `FeignAutoHandlerRegisterHandlerMapping`优先级低于`RequestMappingHandlerMapping`，如果因为业务问题需自行编写Controller他的执行逻辑是优先于自动注入的映射的。
3. 基于`@FeignClient`自动注入的映射目前无法处理配置的`RequestBodyAdvice`和`ResponseBodyAdvice`。 
4. 如果编写了针对Controller的AOP拦截，这里因为是直接注入的映射所以也不在生效，如果需要拦截请自行编写对应扩展。

## 实现原理

基于Spring `ImportBeanDefinitionRegistrar` 和 Spring MVC `RequestMappingHandlerMapping` 进行实现。

大致流程如下

1. SpringApplication 启动扫描到 `@EnableFeignAutoHandlerRegister` 根据此注解上的 `@Import` 注解执行 `FeignAutoHandlerRegisterConfiguration` 类逻辑。
    1. `FeignAutoHandlerRegisterConfiguration` 继承于 `ImportBeanDefinitionRegistrar` 会执行注册逻辑
2. 注册 `FeignAutoHandlerRegisterHandlerMapping` 和 `FeignReturnValueWebMvcConfigurer`
    1. `FeignReturnValueWebMvcConfigurer` 会注册参数解析器和返回值处理器到`RequestMappingHandlerAdapter`以提供针对`@FeignClient`注解的类所有方法都当作 `@ResponseBody`注解过。因为实现类无法继承接口的注解，所以此处又实现了一遍注解参数解析器。
    2. Spring IOC在初始化时会自动将 `FeignAutoHandlerRegisterHandlerMapping` 注册到 `DispatcherServlet` 的 `handlerMappings` 上。
        1. 实际是 `DispatcherServlet` 主动从 Spring IOC 容器中获取实现了 `HandlerMapping` 接口的类进行注册。

执行逻辑：全部执行逻辑都依托于Spring IOC Bean的初始化过程和 Spring MVC的执行逻辑。    

