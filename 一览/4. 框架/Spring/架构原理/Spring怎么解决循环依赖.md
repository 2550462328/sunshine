默认情况下，Spring不允许循环依赖，如果存在循环依赖，会抛出`BeanCurrentlyInCreationException`异常。这是因为Spring默认使用构造函数注入或者setter注入的方式创建Bean，如果两个Bean之间存在循环依赖，则无法满足其中一个Bean的创建要求。

但是，在某些情况下，循环依赖是必要的。例如，两个Bean需要相互引用对方的属性或方法才能正常工作。这时，可以将`allowCircularReferences`属性设置为true，允许循环依赖的存在。

当`allowCircularReferences`属性设置为true时，Spring会使用一个特殊的方式创建Bean，即使用代理对象来解决循环依赖的问题。这种方式可以满足循环依赖的要求，但同时也会带来一些额外的性能开销和复杂性。

需要注意的是，循环依赖可能导致一些问题，例如**无限递归**、**死锁**等，因此建议在确保必要性的情况下才使用循环依赖。



Spring 循环依赖的**场景**有两种：

1. 构造器的循环依赖。
2. field 属性的循环依赖。

对于构造器的循环依赖，Spring 是无法解决的，只能抛出 `BeanCurrentlyInCreationException` 异常表示循环依赖，**所以下面我们分析的都是基于 field 属性的循环依赖**。

> 注：如果项目中不可避免需要使用循环依赖，则必须使用setter注入替代构造器注入。



另外Spring 只解决 scope 为 singleton 的循环依赖。对于scope 为 prototype 的 bean ，Spring 无法解决，直接抛出 `BeanCurrentlyInCreationException` 异常。

为什么 Spring 不处理 prototype bean 呢？其实如果理解 Spring 是如何解决 singleton bean 的循环依赖就明白了。这里先卖一个关子，我们先来关注 Spring 是如何解决 singleton bean 的循环依赖的。



存在循环依赖的场景下的getBean顺序图：

![img](http://pcc.huitogo.club/258fb9ad86259c92030ee4cbb9f68a44)

总结：**借助三个缓存，第一个缓存存储bean实例，第二个缓存存储bean的半成品，第三个缓存存储bean的工厂（避免多次调用bean的创建工厂）**



源码：

```
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    //先存singletonObjects中获取bean，
    Object singletonObject = this.singletonObjects.get(beanName);
    //如果bean不存在，并且bean正在创建中
    if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
        synchronized (this.singletonObjects) {
            //从earlySingletonObjects中获取
            singletonObject = this.earlySingletonObjects.get(beanName);
            //如果earlySingletonObjects不存在(allowEarlyReference默认为true)
            if (singletonObject == null && allowEarlyReference) {
                //获取singletonFactories
                ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
                if (singletonFactory != null) {
                    //从singletonFactories中获取bean
                    singletonObject = singletonFactory.getObject();
                    //添加到earlySingletonObjects
                    this..put(beanName, singletonObject);
                    this.singletonFactories.remove(beanName);
                }
            }
        }
    }
    return (singletonObject != NULL_OBJECT ? singletonObject : null);
}

public boolean isSingletonCurrentlyInCreation(String beanName) {
    return this.singletonsCurrentlyInCreation.contains(beanName);
}
```

- singletonObjects（成熟体）：缓存key = beanName, value = bean；这里的bean是已经创建完成的，该bean经历过实例化->属性填充->初始化以及各类的后置处理。因此，一旦需要获取bean时，我们第一时间就会寻找一级缓存
- earlySingletonObjects（半成品）：缓存key = beanName, value = bean；这里跟一级缓存的区别在于，该缓存所获取到的bean是提前曝光出来的，是还没创建完成的。也就是说获取到的bean只能确保已经进行了实例化，但是属性填充跟初始化还没有做完(AOP情况后续分析)，因此该bean还没创建完成，仅仅能作为指针提前曝光，被其他bean所引用
- singletonFactories（制品工厂）：该缓存key = beanName, value = beanFactory；在bean实例化完之后，属性填充以及初始化之前，如果允许提前曝光，spring会将实例化后的bean提前曝光，也就是把该bean转换成beanFactory并加入到三级缓存。在需要引用提前曝光对象时再通过singletonFactory.getObject()获取。

> 注：三个缓存查找是从上到下，创建是从下到上



***Q1：为什么不能解决原型模式的循环依赖？***

因为原型模式没有办法提前曝光，初始化A的时候就拿不到指定的B，最后就是死循环



***Q2：能不能把三个缓存缩减到两个缓存？***

这个问题考虑的是能不能去除singletonFactories缓存，只在earlySingletonObjects里面放一个半成品？

答案是不行

我们要知道Spring里面获取Bean并不一定都是原始对象，也有可能是增强代理，即AOP，所以我们需要考虑B生成的方式，即在找不到B的时候我们第一步应该去找它的生成工厂

这个增强获取代理的逻辑见`#getEarlyBeanReference(String beanName, RootBeanDefinition mbd, Object bean)` 方法

```
// AbstractAutowireCapableBeanFactory.java

/**
 * 对创建的早期半成品（未初始化）的 Bean 处理引用
 *
 * 例如说，AOP 就是在这里动态织入，创建其代理 Bean 返回
 *
 * Obtain a reference for early access to the specified bean,
 * typically for the purpose of resolving a circular reference.
 * @param beanName the name of the bean (for error handling purposes)
 * @param mbd the merged bean definition for the bean
 * @param bean the raw bean instance
 * @return the object to expose as bean reference
 */
protected Object getEarlyBeanReference(String beanName, RootBeanDefinition mbd, Object bean) {
	Object exposedObject = bean;
	if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
		for (BeanPostProcessor bp : getBeanPostProcessors()) {
			if (bp instanceof SmartInstantiationAwareBeanPostProcessor) {
				SmartInstantiationAwareBeanPostProcessor ibp = (SmartInstantiationAwareBeanPostProcessor) bp;
				exposedObject = ibp.getEarlyBeanReference(exposedObject, beanName);
			}
		}
	}
	return exposedObject;
}
```
