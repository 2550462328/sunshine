JDK提供的SPI(Service Provider Interface)机制，可能很多人不太熟悉，因为这个机制是针对厂商或者插件的，也可以在一些框架的扩展中看到。其核心类`java.util.ServiceLoader`可以在[jdk1.8](https://docs.oracle.com/javase/8/docs/api/java/util/ServiceLoader.html)的文档中看到详细的介绍。虽然不太常见，但并不代表它不常用，恰恰相反，你无时无刻不在用它。玄乎了，莫急，思考一下你的项目中是否有用到第三方日志包，是否有用到数据库驱动？其实这些都和SPI有关。



对Java SPI机制可以由下面一段代码进行分析。下面先写了一个接口，然后是这个接口的两个实现类，最后的就是一个JavaSpi的测试类，很简单的一段逻辑，在测试类中首先构建了一个ServiceLoader 对象，然后获取对象中的迭代器，最后判断迭代器中是否有值，有的话就用刚刚创建的接口接受，然后调用对应的方法。这里其实就可以推测iterator.next() 代码中返回的就是接口的实例，也就是上面创建的两个实现类。至于如何做到的，接着看。

```
public interface Robot {
    void sayHello();
}
 
public class Bumblebee implements Robot{
    @Override
    public void sayHello() {
        System.out.println("Hello, I am Bumblebee.");
    }
}
 
public class OptimusPrime implements Robot{
    @Override
    public void sayHello() {
        System.out.println("Hello, I am Optimus Prime.");
    }
}
 
public class JavaSpiTest {
 
    @Test
    public void testSayHello() {
        //创建一个 ServiceLoader对象，
        ServiceLoader<Robot> serviceLoader = ServiceLoader.load(Robot.class);
        //serviceLoader.forEach(Robot::sayHello);
 
 
        //获取服务下的所有实例信息集合
        Iterator<Robot> iterator = serviceLoader.iterator();
 
        /**
         * 循环创建所有服务实例并执行对应方法
         */
        while (iterator.hasNext()) {
            /**
             * 获取一个服务实例
             */
            Robot robot = iterator.next();
            //调用实例方法
            robot.sayHello();
        }
        // serviceLoader.forEach(Robot::sayHello);
    }
}
```



 首先需要知道的是ServiceLoader 对象是什么，它的位置是在Java.util 包下，实现了Iterable 迭代器接口，具体中文意思翻译过来就是：服务加载程序，其实这里我们可以都不用关心，只需要知道这个是有由Java提供可以构建服务加载的一个对象就行了，具体的还是要看下获取到迭代器之后的hasNext 方法和next 方法，下面上代码：

```
private static final String PREFIX = "META-INF/services/";
 
Class<S> service;
ClassLoader loader;
Enumeration<URL> configs = null;
Iterator<String> pending = null;
String nextName = null;
 
public boolean hasNext() {
	if (acc == null) {
		return hasNextService();
	} else {
		PrivilegedAction<Boolean> action = new PrivilegedAction<Boolean>() {
			public Boolean run() { return hasNextService(); }
		};
		return AccessController.doPrivileged(action, acc);
	}
}
 
private boolean hasNextService() {
	if (nextName != null) {
		return true;
	}
	if (configs == null) {
		try {
			String fullName = PREFIX + service.getName();
			if (loader == null)
				configs = ClassLoader.getSystemResources(fullName);
			else
				configs = loader.getResources(fullName);
		} catch (IOException x) {
			fail(service, "Error locating configuration files", x);
		}
	}
	while ((pending == null) || !pending.hasNext()) {
		if (!configs.hasMoreElements()) {
			return false;
		}
		pending = parse(service, configs.nextElement());
	}
	nextName = pending.next();
	return true;
}
 
public S next() {
	if (acc == null) {
		return nextService();
	} else {
		PrivilegedAction<S> action = new PrivilegedAction<S>() {
			public S run() { return nextService(); }
		};
		return AccessController.doPrivileged(action, acc);
	}
}
 
private S nextService() {
	if (!hasNextService())
		throw new NoSuchElementException();
	String cn = nextName;
	nextName = null;
	Class<?> c = null;
	try {
		c = Class.forName(cn, false, loader);
	} catch (ClassNotFoundException x) {
		fail(service,
			 "Provider " + cn + " not found");
	}
	if (!service.isAssignableFrom(c)) {
		fail(service,
			 "Provider " + cn  + " not a subtype");
	}
	try {
		S p = service.cast(c.newInstance());
		providers.put(cn, p);
		return p;
	} catch (Throwable x) {
		fail(service,
			 "Provider " + cn + " could not be instantiated",
			 x);
	}
	throw new Error();          // This cannot happen
}
```

从下面的代码中可以看到，在hasNext 方法真正调用返回的是hasNextService 方法，而hasNextService 方法的代码中可以看到fullName 变量的值为：META-INF/services/接口路径，然后用这个变量值去获取配置文件信息，最后赋值给nextName。后面next 方法其实也就是调用的nextService 方法中，会使用到nextName 这个变量获取具体的实例对象并返回。



那么也就是说只要在META-INF/services/ 这个路径下，编写一个以接口路径为名称的文件，然后在文件中写入对应的实现类路径，就能去加载。所以上面的测试类想要成功就还需要一个名称为：com.spi.jdk.Robot 的文件，其中的应该写入对应两个实现类的路径。

![img](https://img-blog.csdnimg.cn/886c17ef3fd241e3b095cb2863ce03ec.png)



因此Java SPI 实际上是“**基于接口的编程＋策略模式＋配置文件***”组合实现的**动态加载机制**。


