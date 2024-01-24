如果说HashSet和LinkedHashSet底层维护的是HashMap和LinkedHashMap，同理，TreeSet底层维护的就是TreeMap


从源码可以看出

```
1.  public TreeSet() {  

2.      this(new TreeMap<E,Object>());  

3.  }  


4.  public boolean add(E e) {  

5.      return m.put(e, PRESENT)==null;  

6.  }  


7.  public  boolean addAll(Collection<? extends E> c) {  

8.      // Use linear-time version if applicable  

9.      if (m.size()==0 && c.size() > 0 &&  c instanceof SortedSet && m instanceof TreeMap) {  

12.         SortedSet<? extends E> set = (SortedSet<? extends E>) c;  

13.         TreeMap<E,Object> map = (TreeMap<E, Object>) m;  

14.         Comparator<?> cc = set.comparator();  

15.         Comparator<? super E> mc = map.comparator();  

16.         if (cc==mc || (cc != null && cc.equals(mc))) {  

17.             map.addAllForTreeSet(set, PRESENT);  

18.             return true;  

19.         }  

20.     }  

21.     return super.addAll(c);  

22. }  
```



有点不同的是TreeMap实现的是NavigableMap，而TreeSet实现的是NavigableSet，所以有些方法不能直接用，不过代码逻辑和TreeMap差不多，比如上面代码中的addAll方法。



对比一下TreeMap和TreeSet的继承结构图

![img](http://pcc.huitogo.club/2b613e59a1654a9f6f071b03d52873f1)

![img](http://pcc.huitogo.club/03385f3be677bf1898d43cf7b3e4a834)