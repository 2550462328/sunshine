在 Spring Boot 中，除了我们常用的 application 配置文件之外，还有：

- 系统环境变量
- 命令行参数
- 等等…



我们整理顺序如下：

1. spring-boot-devtools依赖的spring-boot-devtools.properties配置文件。

> 这个灰常小众，建议无视。

2. 单元测试上的 @TestPropertySource和@SpringBootTest注解指定的参数。

> 前者的优先级高于后者。

3. 命令行指定的参数。例如 `java -jar springboot.jar --server.port=9090` 。

4. 命令行中的 `spring.application.json` 指定参数。例如 

   ```
   `java -Dspring.application.json='{"name":"Java"}' -jar springboot.jar
   ```

5. ServletConfig 初始化参数。

6. ServletContext 初始化参数。

7. JNDI 参数。例如 `java:comp/env` 。

8. Java 系统变量，即 `System#getProperties()` 方法对应的。

9. 操作系统环境变量。

10. RandomValuePropertySource 配置的 `random.*` 属性对应的值。

11. Jar **外部**的带指定 profile 的 application 配置文件。例如 `application-{profile}.yaml` 。

12. Jar **内部**的带指定 profile 的 application 配置文件。例如 `application-{profile}.yaml` 。

13. Jar **外部** application 配置文件。例如 `application.yaml` 。

14. Jar **内部** application 配置文件。例如 `application.yaml` 。

15. 在自定义的 `@Configuration` 类中定于的 `@PropertySource` 。

16. 启动的 main 方法中定义的默认配置。即通过`SpringApplication#setDefaultProperties(Map<String, Object> defaultProperties)` 方法进行设置。



***Q1：bootstrap.yml是什么？优先级如何？***

bootstrap.yml是 Spring Cloud 新增的启动配置文件，需要引入 `spring-cloud-context` 依赖后，才会进行加载。它的特点和用途主要是：

- 【特点】因为 bootstrap 由父 ApplicationContext 加载，比 application 优先加载。
- 【特点】因为 bootstrap 优先于 application 加载，所以不会被它覆盖。
- 【用途】使用配置中心 Spring Cloud Config 时，需要在 bootstrap 中配置配置中心的地址，从而实现父 ApplicationContext 加载时，从配置中心拉取相应的配置到应用中。