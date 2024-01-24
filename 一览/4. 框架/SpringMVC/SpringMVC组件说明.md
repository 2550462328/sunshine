#### 1. DispatcherServlet

**前端控制器DispatcherServlet（不需要工程师开发）,由框架提供（重要）**

作用：Spring MVC 的入口函数。接收请求，响应结果，相当于转发器，中央处理器。

有了 DispatcherServlet 减少了其它组件之间的耦合度。用户请求到达前端控制器，它就相当于mvc模式中的c，DispatcherServlet是整个流程控制的中心，由它调用其它组件处理用户的请求，DispatcherServlet的存在降低了组件之间的耦合性。



部分源码如下：

```
1. package org.springframework.web.servlet; 

2.  

3. @SuppressWarnings("serial") 

4. public class DispatcherServlet extends FrameworkServlet { 

5.  

6.   public static final String MULTIPART_RESOLVER_BEAN_NAME = "multipartResolver"; 

7.   public static final String LOCALE_RESOLVER_BEAN_NAME = "localeResolver"; 

8.   public static final String THEME_RESOLVER_BEAN_NAME = "themeResolver"; 

9.   public static final String HANDLER_MAPPING_BEAN_NAME = "handlerMapping"; 

10.   public static final String HANDLER_ADAPTER_BEAN_NAME = "handlerAdapter"; 

11.   public static final String HANDLER_EXCEPTION_RESOLVER_BEAN_NAME = "handlerExceptionResolver"; 

12.   public static final String REQUEST_TO_VIEW_NAME_TRANSLATOR_BEAN_NAME = "viewNameTranslator"; 

13.   public static final String VIEW_RESOLVER_BEAN_NAME = "viewResolver"; 

14.   public static final String FLASH_MAP_MANAGER_BEAN_NAME = "flashMapManager"; 

15.   public static final String WEB_APPLICATION_CONTEXT_ATTRIBUTE = DispatcherServlet.class.getName() + ".CONTEXT"; 

16.   public static final String LOCALE_RESOLVER_ATTRIBUTE = DispatcherServlet.class.getName() + ".LOCALE_RESOLVER"; 

17.   public static final String THEME_RESOLVER_ATTRIBUTE = DispatcherServlet.class.getName() + ".THEME_RESOLVER"; 

18.   public static final String THEME_SOURCE_ATTRIBUTE = DispatcherServlet.class.getName() + ".THEME_SOURCE"; 

19.   public static final String INPUT_FLASH_MAP_ATTRIBUTE = DispatcherServlet.class.getName() + ".INPUT_FLASH_MAP"; 

20.   public static final String OUTPUT_FLASH_MAP_ATTRIBUTE = DispatcherServlet.class.getName() + ".OUTPUT_FLASH_MAP"; 

21.   public static final String FLASH_MAP_MANAGER_ATTRIBUTE = DispatcherServlet.class.getName() + ".FLASH_MAP_MANAGER"; 

22.   public static final String EXCEPTION_ATTRIBUTE = DispatcherServlet.class.getName() + ".EXCEPTION"; 

23.   public static final String PAGE_NOT_FOUND_LOG_CATEGORY = "org.springframework.web.servlet.PageNotFound"; 

24.   private static final String DEFAULT_STRATEGIES_PATH = "DispatcherServlet.properties"; 

25.   protected static final Log pageNotFoundLogger = LogFactory.getLog(PAGE_NOT_FOUND_LOG_CATEGORY); 

26.   private static final Properties defaultStrategies; 

27.   static { 

28.     try { 

29.       ClassPathResource resource = new ClassPathResource(DEFAULT_STRATEGIES_PATH, DispatcherServlet.class); 

30.       defaultStrategies = PropertiesLoaderUtils.loadProperties(resource); 

31.     } 

32.     catch (IOException ex) { 

33.       throw new IllegalStateException("Could not load 'DispatcherServlet.properties': " + ex.getMessage()); 

34.     } 

35.   } 

36.  

37.   /** Detect all HandlerMappings or just expect "handlerMapping" bean? */ 

38.   private boolean detectAllHandlerMappings = true; 

39.  

40.   /** Detect all HandlerAdapters or just expect "handlerAdapter" bean? */ 

41.   private boolean detectAllHandlerAdapters = true; 

42.  

43.   /** Detect all HandlerExceptionResolvers or just expect "handlerExceptionResolver" bean? */ 

44.   private boolean detectAllHandlerExceptionResolvers = true; 

45.  

46.   /** Detect all ViewResolvers or just expect "viewResolver" bean? */ 

47.   private boolean detectAllViewResolvers = true; 

48.  

49.   /** Throw a NoHandlerFoundException if no Handler was found to process this request? **/ 

50.   private boolean throwExceptionIfNoHandlerFound = false; 

51.  

52.   /** Perform cleanup of request attributes after include request? */ 

53.   private boolean cleanupAfterInclude = true; 

54.  

55.   /** MultipartResolver used by this servlet */ 

56.   private MultipartResolver multipartResolver; 

57.  

58.   /** LocaleResolver used by this servlet */ 

59.   private LocaleResolver localeResolver; 

60.  

61.   /** ThemeResolver used by this servlet */ 

62.   private ThemeResolver themeResolver; 

63.  

64.   /** List of HandlerMappings used by this servlet */ 

65.   private List<HandlerMapping> handlerMappings; 

66.  

67.   /** List of HandlerAdapters used by this servlet */ 

68.   private List<HandlerAdapter> handlerAdapters; 

69.  

70.   /** List of HandlerExceptionResolvers used by this servlet */ 

71.   private List<HandlerExceptionResolver> handlerExceptionResolvers; 

72.  

73.   /** RequestToViewNameTranslator used by this servlet */ 

74.   private RequestToViewNameTranslator viewNameTranslator; 

75.  

76.   private FlashMapManager flashMapManager; 

77.  

78.   /** List of ViewResolvers used by this servlet */ 

79.   private List<ViewResolver> viewResolvers; 

80.  

81.   public DispatcherServlet() { 

82.     super(); 

83.   } 

84.  

85.   public DispatcherServlet(WebApplicationContext webApplicationContext) { 

86.     super(webApplicationContext); 

87.   } 

88.   @Override 

89.   protected void onRefresh(ApplicationContext context) { 

90.     initStrategies(context); 

91.   } 

92.  

93.   protected void initStrategies(ApplicationContext context) { 

94.     initMultipartResolver(context); 

95.     initLocaleResolver(context); 

96.     initThemeResolver(context); 

97.     initHandlerMappings(context); 

98.     initHandlerAdapters(context); 

99.     initHandlerExceptionResolvers(context); 

100.     initRequestToViewNameTranslator(context); 

101.     initViewResolvers(context); 

102.     initFlashMapManager(context); 

103.   } 

104. } 
```



DispatcherServlet类中的属性beans：

- HandlerMapping：用于handlers映射请求和一系列的对于拦截器的前处理和后处理，大部分用@Controller注解。
- HandlerAdapter：帮助DispatcherServlet处理映射请求处理程序的适配器，而不用考虑实际调用的是 哪个处理程序。- - -
- ViewResolver：根据实际配置解析实际的View类型。
- ThemeResolver：解决Web应用程序可以使用的主题，例如提供个性化布局。
- MultipartResolver：解析多部分请求，以支持从HTML表单上传文件。-
- FlashMapManager：存储并检索可用于将一个请求属性传递到另一个请求的input和output的FlashMap，通常用于重定向。



在Web MVC框架中，每个DispatcherServlet都拥自己的WebApplicationContext，它继承了ApplicationContext。WebApplicationContext包含了其上下文和Servlet实例之间共享的所有的基础框架beans。



#### 2. HandlerMapping

**处理器映射器HandlerMapping(不需要工程师开发),由框架提供**



作用：根据请求的url查找Handler。HandlerMapping负责根据用户请求找到Handler即处理器（Controller），SpringMVC提供了不同的映射器实现不同的映射方式，例如：配置文件方式，实现接口方式，注解方式等。

![img](http://pcc.huitogo.club/d99a996a443b1a809d6b59d013baa53f)



HandlerMapping接口处理请求的映射HandlerMapping接口的实现类：

- SimpleUrlHandlerMapping类通过配置文件把URL映射到Controller类。
- DefaultAnnotationHandlerMapping类通过注解把URL映射到Controller类。



#### 3. HandlerAdapter

**处理器适配器HandlerAdapter**



作用：按照特定规则（HandlerAdapter要求的规则）去执行Handler 通过HandlerAdapter对处理器进行执行，这是适配器模式的应用，通过扩展适配器可以对更多类型的处理器进行执行。

![img](http://pcc.huitogo.club/f5f872e391ed050fcf0fc184cddd19b7)



AnnotationMethodHandlerAdapter：通过注解，把请求URL映射到Controller类的方法上。



#### 4. Handler

**处理器Handler(需要工程师开发)**



注意：编写Handler时按照HandlerAdapter的要求去做，这样适配器才可以去正确执行Handler Handler 是继DispatcherServlet前端控制器的后端控制器，在DispatcherServlet的控制下Handler对具体的用户请求进行处理。 

由于Handler涉及到具体的用户业务请求，所以一般情况需要工程师根据业务需求开发Handler。



#### 5. View resolver

**视图解析器View resolver(不需要工程师开发),由框架提供**



作用：进行视图解析，根据逻辑视图名解析成真正的视图（view） View Resolver负责将处理结果生成View视图，View Resolver首先根据逻辑视图名解析成物理视图名即具体的页面地址，再生成View视图对象，最后对View进行渲染将处理结果通过页面展示给用户。 

springmvc框架提供了很多的View视图类型，包括：jstlView、freemarkerView、pdfView等。 一般情况下需要通过页面标签或页面模版技术将模型数据通过页面展示给用户，需要由工程师根据业务需求开发具体的页面。

![img](http://pcc.huitogo.club/1b1a932646d13705655508cd7844ce0b)



UrlBasedViewResolver类 通过配置文件，把一个视图名交给到一个View来处理。



#### 6. View

**视图View(需要工程师开发)**



View是一个接口，实现类支持不同的View类型（jsp、freemarker、pdf...）

注意：处理器Handler（也就是我们平常说的Controller控制器）以及视图层view都是需要我们自己手动开发的

其他的一些组件比如：前端控制器DispatcherServlet、处理器映射器HandlerMapping、处理器适配器HandlerAdapter等等都是框架提供给我们的，不需要自己手动开发。