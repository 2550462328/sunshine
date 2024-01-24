LinkedHashSet继承自HashSet，除了拥有HashSet的特性外，它还有一个重要的特性就是**会记住放入元素的顺序**。



查看源码

```
1.  public LinkedHashSet(int initialCapacity, float loadFactor) {  

2.      super(initialCapacity, loadFactor, true);  

3.  }  


4.  // 父类构造  

5.  HashSet(int initialCapacity, float loadFactor, boolean dummy) {  

6.      map = new LinkedHashMap<>(initialCapacity, loadFactor);  

7.  } 
```

可以发现在HashSet中有个实例化LinkedHashMap的构造方法，so它的底层还是依赖LinkedHashMap的特性



**使用LinkedHashSet的场景**
比如有一堆商品，对商品去重，然后按照原先放入的顺序进行输出

在这个场景有个注意事项就是知道HashMap怎么比较key的，换句话说就是怎么认为key是一样的，使用的是**equals方法，所以这个场景中需要覆盖商品类的equals方法**