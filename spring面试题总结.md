# Spring框架是什么

> Spring is one of the most widely used Java EE framework. Spring framework core concepts are “Dependency Injection” and “Aspect Oriented Programming”.
>
> Spring framework can be used in normal java applications also to achieve loose coupling between different components by implementing dependency injection and we can perform cross-cutting tasks such as logging and authentication using spring support for aspect-oriented programming.
>
> I like spring because it provides a lot of features and different modules for specific tasks such as Spring MVC and Spring JDBC. Since it’s an open source framework with a lot of online resources and active community members, working with the Spring framework is easy and fun at the same time.

Spring是在java领域中最为广泛使用的开源框架。spring的核心就是`IOC`和`AOP`。

spring框架能够通过`IOC`的方式实现松耦合，能够通过`AOP`的方式进行横切业务，就比如日志或者权限认证。

# Spring的亮点是什么

> Spring Framework is built on top of two design concepts – Dependency Injection and Aspect Oriented Programming.
>
> Some of the features of spring framework are:
>
> - Lightweight and very little overhead of using framework for our development.
> - Dependency Injection or Inversion of Control to write components that are independent of each other, spring container takes care of wiring them together to achieve our work.
> - Spring IoC container manages Spring Bean life cycle and project specific configurations such as JNDI lookup.
> - Spring MVC framework can be used to create web applications as well as restful web services capable of returning XML as well as JSON response.
> - Support for transaction management, JDBC operations, File uploading, Exception Handling etc with very little configurations, either by using annotations or by spring bean configuration file.
>
> Some of the advantages of using Spring Framework are:
>
> - Reducing direct dependencies between different components of the application, usually Spring IoC container is responsible for initializing resources or beans and inject them as dependencies.
> - Writing unit test cases are easy in Spring framework because our business logic doesn’t have direct dependencies with actual resource implementation classes. We can easily write a test configuration and inject our mock beans for testing purposes.
> - Reduces the amount of boiler-plate code, such as initializing objects, open/close resources. I like JdbcTemplate class a lot because it helps us in removing all the boiler-plate code that comes with JDBC programming.
> - Spring framework is divided into several modules, it helps us in keeping our application lightweight. For example, if we don’t need Spring transaction management features, we don’t need to add that dependency on our project.
> - Spring framework support most of the Java EE features and even much more. It’s always on top of the new technologies, for example, there is a Spring project for Android to help us write better code for native [Android](https://www.journaldev.com/android) applications. This makes spring framework a complete package and we don’t need to look after the different framework for different requirements.

Spring是在`IOC`和`AOP`的基础上设计的，亮点如下：

1. 轻量级的开源框架，有高拓展性
2. `IOC`是为了解决相互依赖，spring的IOC容器能够帮助我们实现依赖注入解决相互依赖的关系
3. Spring ioc管理bean的生命周期还有项目的一些特殊配置，就比如**JNDI**
4. Spring MVC 框架用于构建Web应用，也能够用于restful的web服务，能够返回XML响应或者JSON响应
5. 能够通过一些简单的配置就能够支持事务管理，JDBC操作，文件上传，异常处理等功能

优点如下：

1. 减少应用程序中两个组件的依赖关系，也就是常规说的解耦，通常IOC容器是用来初始化资源