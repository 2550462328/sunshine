LRU（Least recently used，最近最少使用）算法根据数据的历史访问记录来进行淘汰数据，其核心思想是“如果数据最近被访问过，那么将来被访问的几率也更高”。

其实是一个缓存淘汰策略，当缓存数量达到一个阈值时，优先淘汰最近最少被访问的数据。当我们使用LinkedHashMap来实现LRU缓存时，可以通过覆写removeEldestEntry方法可以实现自定义策略的LRU缓存。比如我们可以根据节点数量判断是否移除最近最少被访问的节点，或者根据节点的存活时间判断是否移除该节点等。下面例子所实现的缓存是基于判断节点数量是否超限的策略。



继承LinkedHashMap类覆盖removeEldestEntry方法

```
1.  public class MyLinkedHashMap<K, V> extends LinkedHashMap<K, V> {  

2.      private static final long serialVersionUID = 1L;  

3.      private static int MAX_LIMIT = 100;  

4.      private int limit;  

5.      public MyLinkedHashMap() {  

6.          this(MAX_LIMIT);  

7.      }  

8.      public MyLinkedHashMap(int cacheSize) {  

9.          super();  

10.         this.limit = cacheSize;  

11.     }  

12.     public void save(K key, V value) {  

13.         put(key, value);  

14.     }  

15.     public V getValue(K k) {  

16.         return get(k);  

17.     }  

18.     public boolean exists(K k) {  

19.         return containsKey(k);  

20.     }  

21.     /** 

22.      * 判断什么时候开始移除最近最少使用元素，也就是返回true 

23.      */  

24.     @Override  

25.     protected boolean removeEldestEntry(java.util.Map.Entry<K, V> eldest) {  

26.         return size() > limit;  

27.     }  

28.     @Override  

29.     public String toString() {  

30.         StringBuilder sb = new StringBuilder();  

31.         for (Map.Entry<K, V> entry : entrySet()) {  

32.             sb.append(String.format("%s:%s ", entry.getKey(), entry.getValue()));  

33.         }  

34.         return sb.toString();  

35.     }  

36. }
```



进行测试，是否留下来的元素是最近访问过的

```
1.      public static void main(String[] args) {  

2.          MyLinkedHashMap<String, String> map = new MyLinkedHashMap<>(3);  

3.          for(int i = 0; i < 10; i++) {  

4.              map.save("key"+i, "value"+i);  

5.          }  

6.          System.out.println(map);  

7.          map.save("keyx","valuex");  

8.          System.out.println(map);  

9.          map.get("key8");  

10.         System.out.println(map);  

11.         map.remove("keyx");  

12.         System.out.println(map);  

13.     }     
```

