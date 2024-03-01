`org.springframework.web.servlet.HandlerInterceptor` ，拦截器接口。代码如下：

```
// HandlerInterceptor.java

/**
 * 拦截处理器，在 {@link HandlerAdapter#handle(HttpServletRequest, HttpServletResponse, Object)} 执行之前
 */
default boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)
		throws Exception {
	return true;
}

/**
 * 拦截处理器，在 {@link HandlerAdapter#handle(HttpServletRequest, HttpServletResponse, Object)} 执行成功之后
 */
default void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler,
		@Nullable ModelAndView modelAndView) throws Exception {
}

/**
 * 拦截处理器，在 {@link HandlerAdapter#handle(HttpServletRequest, HttpServletResponse, Object)} 执行完之后，无论成功还是失败
 *
 * 并且，只有该处理器 {@link #preHandle(HttpServletRequest, HttpServletResponse, Object)} 执行成功之后，才会被执行
 */
default void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler,
		@Nullable Exception ex) throws Exception {
}
```

一共有三个方法，分别为：

- `#preHandle(...)` 方法，调用 Controller 方法之**前**执行。
- `#postHandle(...)` 方法，调用 Controller 方法之**后**执行。
- `#afterCompletion(...)`方法，处理完 Controller 方法返回结果之后执行。
  - 例如，页面渲染后。
  - **当然，要注意，无论调用 Controller 方法是否成功，都会执行**。

- 举个例子：
  - 当俩个拦截器都实现放行操作时，执行顺序为 `preHandle[1] => preHandle[2] => postHandle[2] => postHandle[1] => afterCompletion[2] => afterCompletion[1]` 。
  - 当第一个拦截器 `#preHandle(...)` 方法返回 `false` ，也就是对其进行拦截时，第二个拦截器是完全不执行的，第一个拦截器只执行 `#preHandle(...)` 部分。
  - 当第一个拦截器 `#preHandle(...)` 方法返回 `true` ，第二个拦截器 `#preHandle(...)` 返回 `false` ，执行顺序为 `preHandle[1] => preHandle[2] => afterCompletion[1]` 。
- 总结来说：
  - `#preHandle(...)` 方法，按拦截器定义**顺序**调用。若任一拦截器返回 `false` ，则 Controller 方法不再调用。
  - `#postHandle(...)` 和 `#afterCompletion(...)` 方法，按拦截器定义**逆序**调用。
  - `#postHandler(...)` 方法，在调用 Controller 方法之**后**执行。
  - `#afterCompletion(...)` 方法，只有该拦截器在 `#preHandle(...)` 方法返回 `true` 时，才能够被调用，且一定会被调用。为什么“且一定会被调用”呢？即使 `#afterCompletion(...)` 方法，按拦截器定义**逆序**调用时，前面的拦截器发生异常，后面的拦截器还能够调用，**即无视异常**。