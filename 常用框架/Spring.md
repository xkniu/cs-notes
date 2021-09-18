# Spring

Spring 核心技术原理：

- IoC (Inversion of Control)：控制反转。通过 IoC 容器来统一的管理容器对象之间的依赖，依赖的关系由对象主动创建，变成由 IoC 容器来进行注入。
- AOP (Aspect Oriented Programming)：面向切面编程。一种程序设计范式，可以无侵入的在原有逻辑上进行切面。

Bean 的生命周期，主要分为四个阶段：

- 实例化 Instantiation：创建 bean 实例。
- 属性赋值 Populate：注入依赖的属性。
- 初始化 Initialization：`InitializingBean.afterPropertiesSet`/`init-method`
- 销毁 Destruction：`DiposibleBean.destory`/`destory-method`

Bean 生命周期中的一些扩展点：

- 影响所有 bean：
    - BeanPostProcessor：在初始化的前后执行：`preProcessBeforeInitialization`、`postProcessAfterInitialization`。
- 影响单个 bean：
    - XxxAware 接口：在 BeanPostProcessor 之前执行，用来获取 Spring 容器里的一些资源。
    - 生命周期接口：初始化和销毁。

Bean 的作用域：

- singleton（默认）：在 Spring 容器中只有一个实例。
- prototype：每次创建一个实例。
- request：每次 HTTP 请求中创建一个实例，Web ApplicationContext 下有效。
- session：一个 HTTP session 中创建一个实例，Web ApplicationContext 下有效。
- global-session：在 Global session 中创建一个实例，porlet 应用中才有用。

注入 Bean 的两种注解：

- `@Autowire`：默认是通过类型注入。
- `@Resource`：默认是通过名称注入，JSR 定义的注解，建议使用。

AOP 核心概念：

- Aspect（切面）：由 PointCut 和 Advice 组成，包含了横切逻辑的定义和切入位置的定义。
- PointCut(切点）：表达式和模式指定要进行切入的位置。
- Advice（通知）：在方法前后要执行的操作。
- JoinPoint（切点）：程序执行过程中明确的点。
- Target（目标对象）：需要被增强的对象。
- Weaving（织入）：将切面和业务逻辑对象连接起来，创建代理对象。编译阶段织入是静态代理，运行阶段织入是动态代理。

两种动态代理方式的区别：

- JDK 动态代理：根据接口创建一个代理类。
- cglib 动态代理：在运行时动态生成一个类的子类，通过继承的方式实现动态代理。因此 final 类无法使用 cglib 动态代理。

事务的传播属性：

- `REQUIRED`：当前有事务则使用，没有则创建一个。
- `REQUIRES_NEW`：创建一个新的事务。
- `SUPPORTS`：当前有事务则使用，没有则以非事务运行。
- `NOT_SUPPORTED`：以非事务运行。
- `MANDATORY`：当前有事务则使用，没有则抛异常。
- `NEVER`：以非事务运行，如果当前有事务则抛异常。
- `NESTED`：使用数据库的 SavePoint 机制，如果当前有事务，则使用嵌套事务运行，没有则创建事务。嵌套事务中，外层事务回滚，内层事务也会回滚；内层事务回滚，不影响外层事务。

## Spring Boot

为了简化 Spring 的使用难度，使用约定优于配置的思想。引入按约定组织的 jar 包后，能自动注册其中定义的 bean 快速启动。

Starter 根据约定的信息来初始化，启动过程如下：

- `@SpringBootApplication = @Configuration + @EnableAutoConfiguration + @ComponentScan`
- AutoConfigurationImportSelector 会在依赖的 jar 包中找 `resources/META-INF/spring.factories` 文件，找到需要加载的配置类列表。
- 加载配置类，根据配置类的 `@Conditional` 注解，将需要的 bean 注册到 ApplicationContext 中。

## Spring Cloud

Spring Cloud 是一个分布式云服务的框架集合，基于 Spring Boot 来构建服务。它包括服务注册与发现、配置中心、全链路监控、网关、负载均衡、熔断器等一整套的解决方案。Spring Cloud 一方面致力于定义协议规范，另一方面也提供了具体的实现。

Spring Cloud 常用的套件：

- Spring Cloud Netfix：Netflix 开源的 Spring Cloud 套件，开始时最流行的套件，随着 Netflix 减少在开源领域的投入，大部分套件进入维护模式。Spring Clould 在 2020.0.0 版本开始逐渐地移除 Netfix 组件（除了 eureka）。
- Spring Cloud 官方：Spring Cloud 官方也提供了一些组件的具体实现。
- Spring Cloud Alibaba：Alibaba 开源的 Spring Cloud 套件，随着 Netfix 进入维护模式，Alibaba 套件是一个不错的选择。

各套件组件对比：

| | Spring Cloud Netflix | Spring Cloud Alibaba | Spring Cloud 官方 |
| - | - | - | - |
| 配置中心 | Archaius | Nacos | Spring Cloud Config |
| 服务注册发现 | Eureka | Nacos | - |
| 服务熔断 | Hystrix | Sentinel | - |
| 服务调用 | Feign | Dubbo RPC | Spring Cloud OpenFeign RestTemplete |
| 负载均衡 | Ribbon | Dubbo LB | Spring Cloud Load Balancer |
| API 网关 | Zuul | - | Spring Cloud Gateway |
| 分布式消息 | - | Spring Cloud Stream RocketMQ | Spring Cloud Stream RabbitMQ, Spring Cloud Stream Kafka |
| 分布式事务 | - | Seata | - |
