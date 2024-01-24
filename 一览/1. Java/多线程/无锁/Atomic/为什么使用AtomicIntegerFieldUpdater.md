之前思考了下明明可以直接

```
1.  AtomicInteger  ai  = new AtomicInteger(22);  

2.  ai.compareAndSet(22,23);  
```



为什么要那么麻烦的使用AtomicIntegerFieldUpdater

```
1.  public volatile int value;  

2.  // 假设value是User类中的变量  

3.  private AtomicIntegerFieldUpdater<User> updater = AtomicIntegerFieldUpdater.newUpdater(User,”value”);  

4.  // 对User中的value递增，也就是22 + 1  

5.  updater.getAndIncrement(new User(22));  
```



后来想了下也挺容易理解的

1）我可以原子性从外部修改某个类中的变量域，当然你也可以说直接将这个变量暴露出来，但这样设计不妥

2）AtomicInteger是自身进行compareAndSet，而AtomicIntegerFieldUpdater更像一个工具帮助你compareAndSet



所以说AtomicIntegerFieldUpdater可以是扩展了AtomicInteger，为程序设计增加了更多的可能性。

另注：在AtomicIXXXFieldUpdater的类使用中，**需要更新的域必须是public volatile修饰的变量**