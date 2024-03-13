Spring MVC 的简单原理图如下：

![img](http://pcc.huitogo.club/8b0d7de5b14401d4519e59255479eb19)



详细一点的原理图：

![img](http://pcc.huitogo.club/f54fa4c7d2f3bc2ec085d45b38f4f29c)

其中关键几个步骤有：

 **1）发送请求**

用户向服务器发送 HTTP 请求，请求被 Spring MVC 的调度控制器 DispatcherServlet 捕获。

**2）映射处理器**

DispatcherServlet 根据请求 URL ，调用 HandlerMapping 获得该 Handler 配置的所有相关的对象（包括 **Handler** 对象以及 Handler 对象对应的**拦截器**），最后以 HandlerExecutionChain 对象的形式返回。

- 即 HandlerExecutionChain 中，包含对应的 **Handler** 对象和**拦截器**们。

> 此处，对应的方法如下：
>
> ```
> > // HandlerMapping.java
> > 
> > @Nullable
> > HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception;
> >
> ```

**3）处理器适配**

DispatcherServlet 根据获得的 Handler，选择一个合适的HandlerAdapter 。（附注：如果成功获得 HandlerAdapter 后，此时将开始执行拦截器的 `#preHandler(...)` 方法）。

提取请求 Request 中的模型数据，填充 Handler 入参，开始执行Handler（Controller)。 在填充Handler的入参过程中，根据你的配置，Spring 将帮你做一些额外的工作：

- HttpMessageConverter ：会将请求消息（如 JSON、XML 等数据）转换成一个对象。
- 数据转换：对请求消息进行数据转换。如 String 转换成 Integer、Double 等。
- 数据格式化：对请求消息进行数据格式化。如将字符串转换成格式化数字或格式化日期等。
- 数据验证： 验证数据的有效性（长度、格式等），验证结果存储到 BindingResult 或 Error 中。

Handler(Controller) 执行完成后，向 DispatcherServlet 返回一个 ModelAndView 对象。

> 此处，对应的方法如下：
>
> ```
> > // HandlerAdapter.java
> > 
> > @Nullable
> > ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception;
> >
> ```

 **4）解析视图**

根据返回的 ModelAndView ，选择一个适合的 ViewResolver（必须是已经注册到 Spring 容器中的 ViewResolver)，解析出 View 对象，然后返回给 DispatcherServlet。

> 此处，对应的方法如下：
>
> ```
> > // ViewResolver.java
> > 
> > @Nullable
> > View resolveViewName(String viewName, Locale locale) throws Exception;
> >
> ```

**5）渲染视图** + **响应请求**

ViewResolver 结合 Model 和 View，来渲染视图，并写回给用户( 浏览器 )。

> 此处，对应的方法如下：
>
> ```
> > // View.java
> > 
> > void render(@Nullable Map<String, ?> model, HttpServletRequest request, HttpServletResponse response) throws Exception;
> >
> ```



***Q1：我们生产环境使用SpringMVC流程真的是这样吗？***

既然这么问，答案当然不是。对于目前主流的架构，前后端已经进行分离了，所以 Spring MVC 只负责 **M**odel 和 **C**ontroller 两块，而将 **V**iew 移交给了前端。所以，在上图中的步骤 ⑤ 和 ⑥ 两步，已经不在需要。

那么变成什么样了呢？在步骤 ③ 中，如果 Handler(Controller) 执行完后，如果判断方法有 `@ResponseBody` 注解，则直接将结果写回给用户( 浏览器 )。

但是 HTTP 是不支持返回 Java POJO 对象的，所以需要将结果使用 HttpMessageConverter进行转换后，才能返回。例如说，大家所熟悉的 FastJsonHttpMessageConverter，将 POJO 转换成 JSON 字符串返回，Spring MVC 默认使用 MappingJackson2HttpMessageConverter解析转换。



再来补充一些SpringMVC的工作流程图

![流程示意图](https://pcc.huitogo.club/z0/01.png)



