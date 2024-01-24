HashSet的特性如下：

- **元素不允许重复**
- **元素可以为null**



这些特性我们查看源码不难得出

```
1.  private transient HashMap<E,Object> map;  

2.  public boolean add(E e) {  

3.     return map.put(e, PRESENT)==null;  

4.  }  
```



在HashSet底层维护着一个HashMap，放入HashSet的元素会变成HashMap的key，所以他在查询、修改方面效率比较好，同时由于HashMap的 key不可重复且key可以为null，所以不难得到HashSet的特性。