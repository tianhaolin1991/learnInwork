### Spring IOC
### Spring AOP
### Spring 使用
### Spring 事务
- 1.Spring事务的传播方式
```text

```
### SpringMVC
- 1.描述一下一个HTTP请求在SpringMVC中的过程
```text
1.request到达前端控制器(DispatcherServlet)
2.前端控制器根据url去处理器映射器中查找到执行链HandlerExecutionChain
3.前端控制器请求处理器适配器去执行Handler,执行完成后返回ModelAndView
4.前端控制器请求视图解析器去解析视图,之后将渲染之后的视图返回给用户
```
- 2.SpringMVC为什么要使用适配器模式
```text
spring中有多种可以声名接口的方法
```
### SpringBoot
- 说说SpringBoot的自动配置是怎么实现的
```text
0.@SpringBootApplication注解是一个组合注解,其中包含了@EnableAutoConfiguration注解,其中会使用@Import注解来引入AutoConfigurationImportSelector
1.SpringBoot所有自动装配的包都在resource下面存放了一个spring.factories的文件夹,里面定义了EnableAutoConfiguration所需要加载的类,这些类一般是一个配置类,会声名一些约定的配置
2.SpringBoot容器在初始化时,会根据SpringFactoriesLoader类,通过ClassLoader来获取依赖包中的所有spring.factories对自动化配置做解析
3.在ApplicationContext初始化时,会
```