![img](http://pcc.huitogo.club/258fb9ad86259c92030ee4cbb9f68a44)



总结：**借助三个缓存，第一个缓存存储bean实例，第二个缓存存储bean的创建工厂，第三个缓存存储bean的半成品（避免多次调用bean的创建工厂）**



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



1. singletonObjects（成熟体）：缓存key = beanName, value = bean；这里的bean是已经创建完成的，该bean经历过实例化->属性填充->初始化以及各类的后置处理。因此，一旦需要获取bean时，我们第一时间就会寻找一级缓存
2. earlySingletonObjects（半成品）：缓存key = beanName, value = bean；这里跟一级缓存的区别在于，该缓存所获取到的bean是提前曝光出来的，是还没创建完成的。也就是说获取到的bean只能确保已经进行了实例化，但是属性填充跟初始化还没有做完(AOP情况后续分析)，因此该bean还没创建完成，仅仅能作为指针提前曝光，被其他bean所引用
3. singletonFactories（制品工厂）：该缓存key = beanName, value = beanFactory；在bean实例化完之后，属性填充以及初始化之前，如果允许提前曝光，spring会将实例化后的bean提前曝光，也就是把该bean转换成beanFactory并加入到三级缓存。在需要引用提前曝光对象时再通过singletonFactory.getObject()获取。



解决方案：如果项目中出现了循环依赖，则使用setter注入替代构造器注入。