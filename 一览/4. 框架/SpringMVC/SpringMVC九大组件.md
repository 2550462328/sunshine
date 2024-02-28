在Web MVC框架中，每个DispatcherServlet都拥自己的WebApplicationContext，它继承了ApplicationContext。WebApplicationContext包含了其上下文和Servlet实例之间共享的所有的基础框架beans。

DispatcherServlet 会在WebApplicationContext刷新的时候初始化它的九大组件，核心方法#initStrategies(ApplicationContext context)

```
// DispatcherServlet.java

/** MultipartResolver used by this servlet. */
@Nullable
private MultipartResolver multipartResolver;

/** LocaleResolver used by this servlet. */
@Nullable
private LocaleResolver localeResolver;

/** ThemeResolver used by this servlet. */
@Nullable
private ThemeResolver themeResolver;

/** List of HandlerMappings used by this servlet. */
@Nullable
private List<HandlerMapping> handlerMappings;

/** List of HandlerAdapters used by this servlet. */
@Nullable
private List<HandlerAdapter> handlerAdapters;

/** List of HandlerExceptionResolvers used by this servlet. */
@Nullable
private List<HandlerExceptionResolver> handlerExceptionResolvers;

/** RequestToViewNameTranslator used by this servlet. */
@Nullable
private RequestToViewNameTranslator viewNameTranslator;

/** FlashMapManager used by this servlet. */
@Nullable
private FlashMapManager flashMapManager;

/** List of ViewResolvers used by this servlet. */
@Nullable
private List<ViewResolver> viewResolvers;
    
/**
 * This implementation calls {@link #initStrategies}.
 */
@Override
protected void onRefresh(ApplicationContext context) {
	initStrategies(context);
}
    
/**
 * Initialize the strategy objects that this servlet uses.
 * <p>May be overridden in subclasses in order to initialize further strategy objects.
 */
protected void initStrategies(ApplicationContext context) {
    // 初始化 MultipartResolver
	initMultipartResolver(context);
	// 初始化 LocaleResolver
	initLocaleResolver(context);
	// 初始化 ThemeResolver
	initThemeResolver(context);
	// 初始化 HandlerMappings
	initHandlerMappings(context);
	// 初始化 HandlerAdapters
	initHandlerAdapters(context);
	// 初始化 HandlerExceptionResolvers 
	initHandlerExceptionResolvers(context);
	// 初始化 RequestToViewNameTranslator
	initRequestToViewNameTranslator(context);
	// 初始化 ViewResolvers
	initViewResolvers(context);
	// 初始化 FlashMapManager
	initFlashMapManager(context);
}
```



#### 1.MultipartResolver

`org.springframework.web.multipart.MultipartResolver` ，内容类型( `Content-Type` )为 `multipart/*` 的请求的解析器接口。

例如，文件上传请求，MultipartResolver 会将 HttpServletRequest 封装成 MultipartHttpServletRequest ，这样从 MultipartHttpServletRequest 中获得上传的文件。



MultipartResolver 接口，代码如下：

```
// MultipartResolver.java

public interface MultipartResolver {

	/**
     * 是否为 multipart 请求
	 */
	boolean isMultipart(HttpServletRequest request);

	/**
     * 将 HttpServletRequest 请求封装成 MultipartHttpServletRequest 对象
	 */
	MultipartHttpServletRequest resolveMultipart(HttpServletRequest request) throws MultipartException;

	/**
     * 清理处理 multipart 产生的资源，例如临时文件
     *
	 */
	void cleanupMultipart(MultipartHttpServletRequest request);

}
```



#### 2. LocaleResolver

`org.springframework.web.servlet.LocaleResolver` ，本地化( 国际化 )解析器接口。代码如下：

```
// LocaleResolver.java

public interface LocaleResolver {

	/**
     * 从请求中，解析出要使用的语言。例如，请求头的 "Accept-Language"
	 */
	Locale resolveLocale(HttpServletRequest request);

	/**
     * 设置请求所使用的语言
	 */
	void setLocale(HttpServletRequest request, @Nullable HttpServletResponse response, @Nullable Locale locale);

}
```



#### 3. ThemeResolver

`org.springframework.web.servlet.ThemeResolver` ，主题解析器接口。代码如下：

```
// ThemeResolver.java

public interface ThemeResolver {

	/**
     * 从请求中，解析出使用的主题。例如，从请求头 User-Agent ，判断使用 PC 端，还是移动端的主题
	 */
	String resolveThemeName(HttpServletRequest request);

	/**
     * 设置请求，所使用的主题。
	 */
	void setThemeName(HttpServletRequest request, @Nullable HttpServletResponse response, @Nullable String themeName);

}
```

当然，因为现在的前端，基本和后端做了分离，所以这个功能已经越来越少用了。



#### 4. HandlerMapping

`org.springframework.web.servlet.HandlerMapping` ，处理器匹配接口，根据请求( `handler` )获得其的处理器( `handler` )和拦截器们( HandlerInterceptor 数组 )。代码如下：

```
// HandlerMapping.java

public interface HandlerMapping {

	String PATH_WITHIN_HANDLER_MAPPING_ATTRIBUTE = HandlerMapping.class.getName() + ".pathWithinHandlerMapping";
	String BEST_MATCHING_PATTERN_ATTRIBUTE = HandlerMapping.class.getName() + ".bestMatchingPattern";
	String INTROSPECT_TYPE_LEVEL_MAPPING = HandlerMapping.class.getName() + ".introspectTypeLevelMapping";
	String URI_TEMPLATE_VARIABLES_ATTRIBUTE = HandlerMapping.class.getName() + ".uriTemplateVariables";
	String MATRIX_VARIABLES_ATTRIBUTE = HandlerMapping.class.getName() + ".matrixVariables";
	String PRODUCIBLE_MEDIA_TYPES_ATTRIBUTE = HandlerMapping.class.getName() + ".producibleMediaTypes";

	/**
     * 获得请求对应的处理器和拦截器们
	 */
	@Nullable
	HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception;

}
```



返回的对象类型是 HandlerExecutionChain ，它包含处理器( `handler` )和拦截器们( HandlerInterceptor 数组 )，简单代码如下：

```
// HandlerExecutionChain.java

/**
 * 处理器
 */
private final Object handler;
/**
 * 拦截器数组
 */
@Nullable
private HandlerInterceptor[] interceptors;
```

- 注意，处理器的类型可能和我们想的不太一样，是个 **Object** 类型。

  

SpringMVC提供了不同的映射器实现不同的映射方式，例如：配置文件方式，实现接口方式，注解方式等。

![img](http://pcc.huitogo.club/d99a996a443b1a809d6b59d013baa53f)

- SimpleUrlHandlerMapping类通过配置文件把URL映射到Controller类。
- DefaultAnnotationHandlerMapping类通过注解把URL映射到Controller类。



#### 5. HandlerAdapter

`org.springframework.web.servlet.HandlerAdapter` ，处理器适配器接口。代码如下：

```
// HandlerAdapter.java

public interface HandlerAdapter {

	/**
     * 是否支持该处理器
	 */
	boolean supports(Object handler);

	/**
     * 执行处理器，返回 ModelAndView 结果
	 */
	@Nullable
	ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception;

	/**
     * 返回请求的最新更新时间。
     *
     * 如果不支持该操作，则返回 -1 即可
	 */
	long getLastModified(HttpServletRequest request, Object handler);

}
```



通过HandlerAdapter执行处理器是适配器模式的应用，通过扩展适配器可以对更多类型的处理器进行执行。

![img](http://pcc.huitogo.club/f5f872e391ed050fcf0fc184cddd19b7)

- AnnotationMethodHandlerAdapter：通过注解，把请求URL映射到Controller类的方法上。




#### 6. HandlerExceptionResolver

`org.springframework.web.servlet.HandlerExceptionResolver` ，处理器异常解析器接口，将处理器( `handler` )执行时发生的异常，解析( 转换 )成对应的 ModelAndView 结果。代码如下：

```
// HandlerExceptionResolver.java

public interface HandlerExceptionResolver {

	/**
     * 解析异常，转换成对应的 ModelAndView 结果
	 */
	@Nullable
	ModelAndView resolveException(
			HttpServletRequest request, HttpServletResponse response, @Nullable Object handler, Exception ex);

}
```



#### 7. RequestToViewNameTranslator

`org.springframework.web.servlet.RequestToViewNameTranslator` ，请求到视图名的转换器接口。代码如下

```
// RequestToViewNameTranslator.java

public interface RequestToViewNameTranslator {

	/**
     * 根据请求，获得其视图名
	 */
	@Nullable
	String getViewName(HttpServletRequest request) throws Exception;

}
```



#### 8. View resolver

`org.springframework.web.servlet.ViewResolver` ，实体解析器接口，根据视图名和国际化，获得最终的视图 View 对象。代码如下：

```
// ViewResolver.java

public interface ViewResolver {

	/**
     * 根据视图名和国际化，获得最终的 View 对象
	 */
	@Nullable
	View resolveViewName(String viewName, Locale locale) throws Exception;

}
```



springmvc框架提供了很多的View视图类型，包括：jstlView、freemarkerView、pdfView等。 一般情况下需要通过页面标签或页面模版技术将模型数据通过页面展示给用户，需要由工程师根据业务需求开发具体的页面。

![img](http://pcc.huitogo.club/1b1a932646d13705655508cd7844ce0b)

- UrlBasedViewResolver类 通过配置文件，把一个视图名交给到一个View来处理。




#### 9. FlashMapManager

`org.springframework.web.servlet.FlashMapManager` ，FlashMap 管理器接口，负责重定向时，保存参数到临时存储中。代码如下：

```
// FlashMapManager.java

public interface FlashMapManager {

	/**
     * 恢复参数，并将恢复过的和超时的参数从保存介质中删除
	 */
	@Nullable
	FlashMap retrieveAndUpdate(HttpServletRequest request, HttpServletResponse response);

	/**
     * 将参数保存起来
	 */
	void saveOutputFlashMap(FlashMap flashMap, HttpServletRequest request, HttpServletResponse response);

}
```

默认情况下，这个临时存储会是 Session 。也就是说：

- 重定向前，保存参数到 Session 中。
- 重定向后，从 Session 中获得参数，并移除。

当然，实际场景下，使用的非常少，特别是前后端分离之后。