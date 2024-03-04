下图是tomcat 的启动时序图

![img](http://pcc.huitogo.club/92eeecfa6b461721c5696f00dd7b7971)



由图中我们可以看到从Bootstrap类的main方法开始, tomcat会以链的方式逐级调用各个模块的init()方法进行初始化, 待各个模块都初始化后, 又会逐级调用各个模块的start()方法启动各个模块



下面通过源码的调用层级来看一下：

```
Catilna

		Bootstrap
		
				Server --- StandardServer                tomcat           
				
						Service --- StandardService
						
						Engine --- StandardEngine、EngineConfig          
						
						Container --- ContainerBase          webApps                   
						
										Connector(8080、443)  -> ProtocolHandler -> EndPoint  --- NIOEndPoint、NIO2EndPoint、APREndPoint
										
										Host  --- StandardHost、HosConfig                               
										
													Context --- StandardContext、ContextConfig                        webApp       
													
																Wrapper  ---   StandardWrapper                                 servlet         
```



主要分成两个大接口，Lifeclycle 和 LifecycleListener，Lifeclycle 进行 init 和 start 控制生命周期，变更状态的时候 触发LifecycleListener事件，执行lifecycleEvent

Lifeclycle ---> LifeclycleMBeanBase：ContainerBase（StandardEngine、StandardHost、StandardContext）、StandardServer、StandardService

LifecycleListener：EngineConfig、HosConfig、ContextConfig