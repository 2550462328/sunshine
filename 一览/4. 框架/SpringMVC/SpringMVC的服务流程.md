Spring MVC 的简单原理图如下：

![img](http://pcc.huitogo.club/8b0d7de5b14401d4519e59255479eb19)



详细一点的原理图：

![img](http://pcc.huitogo.club/f54fa4c7d2f3bc2ec085d45b38f4f29c)



流程说明：

1. 用户发送请求至前端控制器DispatcherServlet。
2. DispatcherServlet收到请求调用HandlerMapping处理器映射器。
3. 处理器映射器找到具体的处理器(可以根据xml配置、注解进行查找)，生成处理器对象及处理器 拦截器(如果有则生成)一并返回给DispatcherServlet。
4. DispatcherServlet调用HandlerAdapter处理器适配器。
5. HandlerAdapter经过适配调用具体的处理器(Controller，也叫后端控制器)。
6. Controller执行完成返回ModelAndView。
7. HandlerAdapter将controller执行结果ModelAndView返回给DispatcherServlet。
8. DispatcherServlet将ModelAndView传给ViewReslover视图解析器。
9. ViewReslover解析后返回具体View。
10. DispatcherServlet根据View进行渲染视图（即将模型数据填充至视图中）。
11. DispatcherServlet响应用户。



**从SpringMVC的核心组件交互来看流程就是**

1. 首先用户发送请求——>**DispatcherServlet**，前端控制器收到 请求后自己不进行处理，而是委托给其他的解析器进行处理，作为统一访问点，进行全局的流程控 制；
2. DispatcherServlet——>**HandlerMapping**， HandlerMapping 将会把请求映射为 HandlerExecutionChain 对象（包含一个Handler 处理器（页面控制器）对象、多个 HandlerInterceptor 拦截器）对象，通过这种策略模式，很容易添加新的映射策略；
3. DispatcherServlet——>**HandlerAdapter**，HandlerAdapter 将会把处理器包装为适配器，从而支 持多种类型的处理器，即适配器设计模式的应用，从而很容易支持很多类型的处理器；
4. HandlerAdapter——>处理器功能处理方法的调用，HandlerAdapter 将会根据适配的结果调用真 正的处理器的功能处理方法，完成功能处理；并返回一个**ModelAndView** 对象（包含模型数据、逻辑视图名）；
5. ModelAndView的逻辑视图名——>**ViewResolver**， ViewResolver 将把逻辑视图 名解析为具体的View，通过这种策略模式，很容易更换其他视图技术；
6. View——>渲染，**View** 会根据传进来的Model模型数据进行渲染，此处的Model实际是一个Map数据结构，因此很容易支 持其他视图技术；
7. 返回控制权给DispatcherServlet，由DispatcherServlet返回响应给用户，到此一个流程结束。



**简而言之**

1. 前端控制器（DispatcherServlet）：接收请求，响应结果，相当于电脑的CPU。
2. 处理器映射器（HandlerMapping）：根据URL去查找处理器。
3. 处理器（Handler）：需要程序员去写代码处理逻辑的。
4. 处理器适配器（HandlerAdapter）：会把处理器包装成适配器，这样就可以支持多种类型的处理 器，类比笔记本的适配器（适配器模式的应用）。
5. 视图解析器（ViewResovler）：进行视图解析，多返回的字符串，进行处理，可以解析成对应的页 面。