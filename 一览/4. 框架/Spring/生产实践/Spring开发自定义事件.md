Spring 的 ApplicationContext 提供了支持事件和代码中监听器的功能。

我们可以创建 Bean 用来监听在 ApplicationContext 中发布的事件。如果一个 Bean 实现了 ApplicationListener 接口，当一个ApplicationEvent 被发布以后，Bean 会自动被通知。示例代码如下：

```
public class AllApplicationEventListener implements ApplicationListener<ApplicationEvent> {  
    
    @Override  
    public void onApplicationEvent(ApplicationEvent applicationEvent) {  
        // process event  
    }
    
}
```



Spring 提供了以下五种标准的事件：

1. 上下文更新事件（ContextRefreshedEvent）：该事件会在ApplicationContext 被初始化或者更新时发布。也可以在调用ConfigurableApplicationContext 接口中的 `#refresh()` 方法时被触发。
2. 上下文开始事件（ContextStartedEvent）：当容器调用ConfigurableApplicationContext 的 `#start()` 方法开始/重新开始容器时触发该事件。
3. 上下文停止事件（ContextStoppedEvent）：当容器调用 ConfigurableApplicationContext 的 `#stop()` 方法停止容器时触发该事件。
4. 上下文关闭事件（ContextClosedEvent）：当ApplicationContext 被关闭时触发该事件。容器被关闭时，其管理的所有单例 Bean 都被销毁。
5. 请求处理事件（RequestHandledEvent）：在 Web应用中，当一个HTTP 请求（request）结束触发该事件。





除了上面介绍的事件以外，还可以通过扩展 ApplicationEvent 类来开发**自定义**的事件。

① 示例自定义的事件的类，代码如下：

```
public class CustomApplicationEvent extends ApplicationEvent{  

    public CustomApplicationEvent(Object source, final String msg) {  
        super(source);
    }  

}
```



② 为了监听这个事件，还需要创建一个监听器。示例代码如下：

```
public class CustomEventListener implements ApplicationListener<CustomApplicationEvent> {

    @Override  
    public void onApplicationEvent(CustomApplicationEvent applicationEvent) {  
        // handle event  
    }
    
}
```



③ 之后通过 ApplicationContext 接口的 `#publishEvent(Object event)` 方法，来发布自定义事件。示例代码如下：

```
// 创建 CustomApplicationEvent 事件
CustomApplicationEvent customEvent = new CustomApplicationEvent(applicationContext, "Test message");
// 发布事件
applicationContext.publishEvent(customEvent);
```