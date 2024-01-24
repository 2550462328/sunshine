### 1. TreeMap特性

1. 键值不允许重复，键不能为null
2. 默认会对键进行排序，所以键必须实现Comparable接口或者使用外部比较器
3. 查找、移除、添加操作的时间复杂度为log(n)
4. 底层使用的数据结构是红黑树



下面是TreeMap的常用操作示例：

```
1.  public static void main(String[] args){  

2.      TreeMap<String, Integer> grades = new TreeMap<>();  

3.      grades.put("Frank", 100);  

4.      grades.put("Alice", 95);  

5.      grades.put("Mary", 90);  

6.      grades.put("Bob", 85);  

7.      grades.put("Jack", 90);  

8.      System.out.println(grades);  

9.      // 获取从"Bob"到"Jack"的子map，不包括头尾节点  

10.     System.out.println(grades.subMap("Bob", "Jack"));  

11.     // 获取从"Bob"到"Jack"的子map，包括头尾节点      

12.     System.out.println(grades.subMap("Bob", true, "Jack", true));  

13.     // 返回与大于或等于"Bob"的键值映射，如果没有此键，则 null   

14.     System.out.println(grades.ceilingEntry("Bob"));  

15.     System.out.println(grades.ceilingKey("Bob"));  

16.     // 返回与大Bob的键值映射，如果没有此键，则 null   

17.     System.out.println(grades.higherEntry("Bob"));  

18.     System.out.println(grades.higherKey("Bob"));  

19.     // 返回起点是map的最小值，终点是"Bob"的子map,不包括"Bob"  

20.     System.out.println(grades.headMap("Bob"));  

21.     // 返回起点是map的最小值，终点是"Bob"的子map,包括"Bob"  

22.     System.out.println(grades.headMap("Bob", true));  

23.     // 返回起点是"Bob"，终点是map的“最大值”,不包括"Bob"  

24.     System.out.println(grades.tailMap("Bob"));  

25.     // 返回起点是"Bob"，终点是map的“最大值”, 包括"Bob"  

26.     System.out.println(grades.tailMap("Bob", true));  

27.     System.out.println(grades.containsKey("Bob"));  

28.     System.out.println(grades.containsValue(90));  

29.     // 返回当前map的反向排序视图，该视图的操作会影响原视图  

30.     System.out.println(grades.descendingMap());  

31.     System.out.println(grades.descendingKeySet());  

32. }
```



注意：

- 原Map的subMap(或者其他切割map的操作)以及descendingMap的操作（增删改）操作都会影响原视图
- 对上述操作分割出来的map进行增删改的时候，会验证参数key值，如果key比原map最小值还小或者最大值还大将会报错。



### 2. TreeMap继承结构

![img](http://pcc.huitogo.club/3e765bb091a4fee36ee75b039c953b7d)

除了基本的接口外TreeMap主要继承的是NavigableMap接口，而NavigableMap接口继承的是SortedMap接口

**SortedMap接口就是一个具有排序功能的接口**

NavigableMap接口在SortedMap接口的基础上增加了导航（查询定位分割）功能，比如查询指定key的最近的一个比它大的节点是谁，在指定key和key之间的keyMap是什么，以及返回结果的时候包不包括参数key本身之类的等等，这些方法在TreeMap中都有实现。



TreeMap的数据结构也就是红黑树由Entry组成

```
1.  static final class Entry<K,V> implements Map.Entry<K,V> {  

2.      K key;  

3.      V value;  

4.      Entry<K,V> left;  

5.      Entry<K,V> right;  

6.      Entry<K,V> parent;  

7.      boolean color = BLACK;  


8.      /** 

9.       * 构造函数
      
10.      */  

10.     Entry(K key, V value, Entry<K,V> parent) {  

11.         this.key = key;  

12.         this.value = value;  

13.         this.parent = parent;  

14.     }  

15.     ...  

16.   } 
```



TreeMap的变量如下

```
1.  /** 

2.   * 外部比较器 

3.   */  

4.  private final Comparator<? super K> comparator;  

5.  private transient Entry<K,V> root;  

6.  /** 

7.   * 键值对数量 

8.   */  

9.  private transient int size = 0;  

10. private transient int modCount = 0;  

11. private static final boolean RED   = false;  

12. private static final boolean BLACK = true;  

13. /** 

14.  * 键值对集合 

15.  */  

16. private transient EntrySet entrySet;  

17. /** 

18.  * 键的集合 

19.  */  

20. private transient KeySet<K> navigableKeySet;  

21. /** 

22.  * 倒序Map 

23.  */  

24. private transient NavigableMap<K,V> descendingMap;
```



### 3. TreeMap是怎么实现排序的

关于排序想到的是Comparable接口和Comparator比较器，所以对于put进TreeMap的key值如果是自定义类的话需要：

- 自定义类实现**Comparable**
- 自定义**Comrator**比较器，在实例化TreeMap时指定比较器



由于TreeMap底层是**红黑树**的，所以需要根据**比较key**来决定存放元素的位置，在读取元素的时候通过比较key使用**二分法**很快比较出来



相关代码类似以下（先查看有没有自定义比较器）：

```
1.  Comparator<? super K> cpr = comparator;  

2.  if (cpr != null) {  

3.      do {  

4.          parent = t;  

5.         cmp = cpr.compare(key, t.key); //比较一 

6.          ... // 根据比较结果存取值或者覆盖值  

7.      } while (t != null);  

8.  }  

9.  else {  

10.     if (key == null)  

11.         throw new NullPointerException();  

12.      @SuppressWarnings("unchecked")  

13.         Comparable<? super K> k = (Comparable<? super K>) key;  

14.     do {  

15.         parent = t;  

16.         cmp = k.compareTo(t.key);  // 比较二

17.         ... // 根据比较结果存取值或者覆盖值  

18.     } while (t != null);  

19. }  
```



### 4. TreeMap的主要方法

TreeMap底层维护的一个数据结构就是红黑树，知道这个就很好理解TreeMap在增删改查时需要考虑的问题了，其中最重要的就是红黑树的平衡了，使用变色和旋转进行解决，关于这部分代码跟HashMap中同出一辙，详细可以看HashMap红黑树的部分。



#### 4.1 put

```
1.  // 插入元素  

2.  public V put(K key, V value) {  

3.      TreeMap.Entry<K,V> t = root;  

4.      if (t == null) {  

5.           // 这里主要对key进行空值检测和类型检测  

6.           compare(key, key); // type (and possibly null) check  

7.          // 如果根节点不存在，则用传入的键值对信息生成一个根节点  

8.          root = new TreeMap.Entry<>(key, value, null);  

9.          size = 1;  

10.         modCount++;  

11.         return null;  

12.     }  

13.     int cmp;  

14.     TreeMap.Entry<K,V> parent;  

15.     Comparator<? super K> cpr = comparator;  

16.     if (cpr != null) {  

17.         do {  

18.             // 如果外部比较器不为空，则依次与各节点进行比较  

19.             parent = t;  

20.             cmp = cpr.compare(key, t.key);  

21.             if (cmp < 0)   

22.                 // 小于则与左孩子比较  

23.                 t = t.left;  

24.             else if (cmp > 0)  

25.                 // 大于则与右孩子比较  

26.                 t = t.right;  

27.             else  

28.                 // 找到相等的key则替换其value  

29.                 return t.setValue(value);  

30.             // 一直循环，直到待比较的节点为null  

31.         } while (t != null);  

32.     }  

33.     else {  

34.         // 如果外部比较器为null  

35.         // 如果key为null则抛出空指针  

36.         if (key == null)  

37.             throw new NullPointerException();  

38.         // 如果key未实现comparable接口则会抛出异常  

39.         @SuppressWarnings("unchecked")  

40.         Comparable<? super K> k = (Comparable<? super K>) key;  

41.         do {  

42.             // 跟上面逻辑类似，只是用key的compareTo方法进行比较，而不是用外部比较器的compare方法  

43.             parent = t;  

44.             cmp = k.compareTo(t.key);  

45.             if (cmp < 0)  

46.                 t = t.left;  

47.             else if (cmp > 0)  

48.                 t = t.right;  

49.             else  

50.                 return t.setValue(value);  

51.         } while (t != null);  

52.     }  

53.     // 生成键值对  

54.     TreeMap.Entry<K,V> e = new TreeMap.Entry<>(key, value, parent);  

55.     // 连接到当前map的左孩子位置或者右孩子位置  

56.     if (cmp < 0)  

57.         parent.left = e;  

58.     else  

59.         parent.right = e;  

60.     // 插入后的调整  

61.     fixAfterInsertion(e); 

62.     size++;  

63.     modCount++;  

64.     return null;  

65. }  
```



**fixAfterInsertion(e)代码**

```
1.  private void fixAfterInsertion(TreeMap.Entry<K,V> x) {  

2.       // 将插入的节点初始化为红色节点  

3.       x.color = RED;  

4.       // 如果x不为null且x不是根节点，且x的父节点是红色，此时祖父节点一定为黑色  

5.       while (x != null && x != root && x.parent.color == RED) {  

6.           // 如果x的父节点为祖父节点的左孩子  

7.           if (parentOf(x) == leftOf(parentOf(parentOf(x)))) {  

8.               // y指向x的叔叔节点  

9.               TreeMap.Entry<K,V> y = rightOf(parentOf(parentOf(x)));  

10.              // 如果叔叔节点也是红色，则进行变色处理  

11.              if (colorOf(y) == RED) {  

12.                  // 父节点变成黑色  

13.                  setColor(parentOf(x), BLACK);  

14.                  // 叔叔节点变成黑色  

15.                  setColor(y, BLACK);  

16.                  // 祖父节点变成黑色  

17.                  setColor(parentOf(parentOf(x)), RED);  

18.                  // 将x指向祖父节点，继续往上调整  

19.                  x = parentOf(parentOf(x));  

20.              } else {  

21.                  // 如果叔叔节点是黑色节点  

22.                  // 如果x是父节点的右孩子  

23.                  if (x == rightOf(parentOf(x))) {  

24.                      // 将x指向其父节点  

25.                      x = parentOf(x);  

26.                      // 左旋  

27.                      rotateLeft(x);  

28.                  }  

29.                  // 将x的父节点置为黑色  

30.                  setColor(parentOf(x), BLACK);  

31.                  // 将x的祖父节点置为红色  

32.                  setColor(parentOf(parentOf(x)), RED);  

33.                  // 将祖父节点右旋  

34.                  rotateRight(parentOf(parentOf(x)));  

35.              }  

36.          } else {  

37.              // 这里类似操作  

38.              TreeMap.Entry<K,V> y = leftOf(parentOf(parentOf(x)));  

39.              if (colorOf(y) == RED) {  

40.                  setColor(parentOf(x), BLACK);  

41.                  setColor(y, BLACK);  

42.                  setColor(parentOf(parentOf(x)), RED);  

43.                  x = parentOf(parentOf(x));  

44.              } else {  

45.                  if (x == leftOf(parentOf(x))) {  

46.                      x = parentOf(x);  

47.                      rotateRight(x);  

48.                  }  

49.                  setColor(parentOf(x), BLACK);  

50.                  setColor(parentOf(parentOf(x)), RED);  

51.                  rotateLeft(parentOf(parentOf(x)));  

52.              }  

53.          }  

54.      }  

55.      root.color = BLACK;  

56.  }  
```

这里的逻辑跟HashMap中TreeNode的插入逻辑十分类似，也是先找到要插入的位置，然后再进行结构调整。这里的结构调整即红黑树的结构调整



#### 4.2 remove

```
1.  // 删除节点  

2.  public V remove(Object key) {  

3.      // 先找到该key对应的键值对  

4.      TreeMap.Entry<K,V> p = getEntry(key);  

5.      if (p == null)  

6.          // 如果未找到则返回null  

7.          return null;  

8.      V oldValue = p.value;  

9.      // 找到后删除该键值对  

10.     deleteEntry(p);  

11.     return oldValue;  

12. }
```



**getEntry(key)：根据key获取Entry**

```
1.  final TreeMap.Entry<K,V> getEntry(Object key) {  

2.      if (comparator != null)  

3.          // 如果有自定义的比较器  代码和下面key实现了Comparator接口一样

4.          return getEntryUsingComparator(key);  

5.      if (key == null)  

6.          throw new NullPointerException();  

7.      @SuppressWarnings("unchecked")  

8.      Comparable<? super K> k = (Comparable<? super K>) key;  

9.      TreeMap.Entry<K,V> p = root;  

10.     // 使用compareTo方法进行查找  

11.     while (p != null) {  

12.         int cmp = k.compareTo(p.key);  

13.         if (cmp < 0)  

14.             p = p.left;  

15.         else if (cmp > 0)  

16.             p = p.right;  

17.         else  

18.             return p;  

19.     }  

20.     return null;  

21. }  
```



**deleteEntry(p)：删除指定Entry，并调整红黑树以保持它的平衡**

```
1.  private void deleteEntry(TreeMap.Entry<K,V> p) {  

2.      modCount++;  

3.      size--;  

4.      // 如果p的左右孩子均不为空，则找到p的后继节点，并且将p指向该后继节点  

5.      if (p.left != null && p.right != null) {  

6.          TreeMap.Entry<K,V> s = successor(p);  

7.          p.key = s.key;  

8.          p.value = s.value;  

9.          p = s;  

10.     }  // p has 2 children  

11.     // 修复替补节点  

12.     // 用替补节点替换待删除的节点后，需要对其原来所在位置结构进行修复  

13.     TreeMap.Entry<K,V> replacement = (p.left != null ? p.left : p.right);  

14.     if (replacement != null) {  

15.         replacement.parent = p.parent;  

16.         if (p.parent == null)  

17.             root = replacement;  

18.         else if (p == p.parent.left)  

19.             p.parent.left  = replacement;  

20.         else  

21.             p.parent.right = replacement;  

22.         p.left = p.right = p.parent = null;  

23.         // 如果p的颜色是黑色，则进行删除后的修复  

24.         if (p.color == BLACK)  

25.             fixAfterDeletion(replacement);  

26.     } else if (p.parent == null) {  

27.         root = null;  

28.     } else {  

29.         if (p.color == BLACK)  

30.             fixAfterDeletion(p); 

31.         if (p.parent != null) {  

32.             if (p == p.parent.left)  

33.                 p.parent.left = null;  

34.             else if (p == p.parent.right)  

35.                 p.parent.right = null;  

36.             p.parent = null;  

37.         }  

38.     }  

39. }  
```



**successor(p)：找到指定节点的后继节点，这个原理和predecessor一样，这里是在右子树中找最左的，如果没有右子树就在叔叔节点中去找最左的。**

```
1.  static <K,V> TreeMap.Entry<K,V> successor(TreeMap.Entry<K,V> t) {  

2.      if (t == null)  

3.          return null;  

4.      else if (t.right != null) {  

5.          TreeMap.Entry<K,V> p = t.right;  

6.          // 如果右子树不为空，则找到右子树的最左节点作为后继节点  

7.          while (p.left != null)  

8.              p = p.left;  

9.          return p;  

10.     } else {  

11.         TreeMap.Entry<K,V> p = t.parent;  

12.         TreeMap.Entry<K,V> ch = t;  

13.         // 如果右子树为空且当前节点为其父节点的左孩子，则直接返回  

14.         // 如果为其父节点的右孩子，则一直往上找，直到找到根节点或者当前节点为其父节点的左孩子时，用其做为后继节点  

15.         while (p != null && ch == p.right) {  

16.             ch = p;  

17.             p = p.parent;  

18.         }  

19.         return p;  

20.     }  

21. }  
```



**fixAfterDeletion(replacement)：进行删除后的结构修复**

```
1.  private void fixAfterDeletion(TreeMap.Entry<K,V> x) {  

2.      while (x != root && colorOf(x) == BLACK) {  

3.          // 如果x是父节点的左孩子  

4.          if (x == leftOf(parentOf(x))) {  

5.              // sib指向x的兄弟节点  

6.              TreeMap.Entry<K,V> sib = rightOf(parentOf(x));  

7.              // 如果sib是红色，则进行变色处理  

8.              if (colorOf(sib) == RED) {  

9.                  // 兄弟节点改为黑色  

10.                 setColor(sib, BLACK);  

11.                 // 父节点改为红色  

12.                 setColor(parentOf(x), RED);  

13.                 // 父节点左旋  

14.                 rotateLeft(parentOf(x));  

15.                 // sib指向x的父节点的右孩子  

16.                 sib = rightOf(parentOf(x));  

17.             }  

18.             // 如果sib的左孩子和右孩子都是黑色，则进行变色处理  

19.             if (colorOf(leftOf(sib))  == BLACK &&  

20.                     colorOf(rightOf(sib)) == BLACK) {  

21.                 // 将sib置为红色  

22.                 setColor(sib, RED);  

23.                 // x指向其父节点  

24.                 x = parentOf(x);  

25.             } else {  

26.                 // 如果sib的右孩子是黑色而左孩子是红色，则变色右旋  

27.                 if (colorOf(rightOf(sib)) == BLACK) {  

28.                     setColor(leftOf(sib), BLACK);  

29.                     setColor(sib, RED);  

30.                     rotateRight(sib);  

31.                     sib = rightOf(parentOf(x));  

32.                 }  

33.                 // 变色左旋  

34.                 setColor(sib, colorOf(parentOf(x)));  

35.                 setColor(parentOf(x), BLACK);  

36.                 setColor(rightOf(sib), BLACK);  

37.                 rotateLeft(parentOf(x));  

38.                 x = root;  

39.             }  

40.         } else { // symmetric  

41.             // 跟上面操作类似  

42.             TreeMap.Entry<K,V> sib = leftOf(parentOf(x));  

43.             if (colorOf(sib) == RED) {  

44.                 setColor(sib, BLACK);  

45.                 setColor(parentOf(x), RED);  

46.                 rotateRight(parentOf(x));  

47.                 sib = leftOf(parentOf(x));  

48.             }  

49.             if (colorOf(rightOf(sib)) == BLACK &&  

50.                     colorOf(leftOf(sib)) == BLACK) {  

51.                 setColor(sib, RED);  

52.                 x = parentOf(x);  

53.             } else {  

54.                 if (colorOf(leftOf(sib)) == BLACK) {  

55.                     setColor(rightOf(sib), BLACK);  

56.                     setColor(sib, RED);  

57.                     rotateLeft(sib);  

58.                     sib = leftOf(parentOf(x));  

59.                 }  

60.                 setColor(sib, colorOf(parentOf(x)));  

61.                 setColor(parentOf(x), BLACK);  

62.                 setColor(leftOf(sib), BLACK);  

63.                 rotateRight(parentOf(x));  

64.                 x = root;  

65.             }  

66.         }  

67.     }  

68.     setColor(x, BLACK);  

69. }  
```



#### 4.3 buildFromSorted

这个方法可以将一个sortedMap转换成红黑树



在TreeMap的putAll()和带Map的构造方法中都有使用，如下

```
1.  public TreeMap(SortedMap<K, ? extends V> m) {  

2.      comparator = m.comparator();  

3.      try {  

4.          buildFromSorted(m.size(), m.entrySet().iterator(), null, null);  

5.      } catch (java.io.IOException cannotHappen) {  

6.      } catch (ClassNotFoundException cannotHappen) {  

7.      }  

8.  }  
```



**buildFromSorted(m.size(), m.entrySet().iterator(), null, null)**

第一个参数是sortedMap的size

第二个参数是sortedMap集合的迭代器，这里是entrySet的

第三个参数是字符流，当迭代器为空的时候会从字符流中读取数据作为key

第四个参数是默认值，当这个值不为空时，这个默认值作为所有entry的value

```
1.  private void buildFromSorted(int size, Iterator<?> it,  java.io.ObjectInputStream str, V defaultVal)  

4.      throws  java.io.IOException, ClassNotFoundException {  

5.      this.size = size;  

6.      root = buildFromSorted(0, 0, size-1, computeRedLevel(size),  it, str, defaultVal); 

8.  }  
```



**buildFromSorted(0,  0,  size-1,  computeRedLevel(size),  it,  str, defaultVal)**

第一参数是当前的level，可以想象成遍历sortedMap时的树的高度

第二个参数是元素下标的最小值，就是start

第三个参数是元素下标的最大值，就是end

第四个参数是根据sortedMap的size计算出来的树的最大高度，对于通过sortedMap转过来的红黑树，这里原则是叶子节点全部必须是红色的，所以这里会在遍历中判断第一个参数是否等于第四个参数，也就是当前高度是否是最大高度

后面的参数意义同上面



这个方法返回的转换成红黑树的中间节点，所以如果需要全部转换成红黑树的话，这里就要用到递归了



**思路**：如有序列 1,2,3,4,5,6,7,8 ,9,10,以最中间的数作为根结点，然后将序列分成两组，(1,2,3,4) (6,7,8,9,10)，以同理的方法

在第一组序列中找出最中间的树作为根结点，建立一个子树，该子树作为整个树的左子树

在第二个序列中找出最中间的树作为根结点，孩子子树作为整个树的右树

以此递归下去最终形成的树是在叶子结点以上是一个满二叉树，所以满足红黑树的性质，叶子结点不满足，所以把叶子结点都染成红色

```
1.  private final Entry<K,V> buildFromSorted(int level, int lo, int hi,   int redLevel,  Iterator<?> it,  java.io.ObjectInputStream str,  V defaultVal)  

6.      throws  java.io.IOException, ClassNotFoundException {  

7.      if (hi < lo) return null;  

8.      int mid = (lo + hi) >>> 1;  

9.      Entry<K,V> left  = null;  

10.     if (lo < mid)  

11.         left = buildFromSorted(level+1, lo, mid - 1, redLevel,  it, str, defaultVal);  

13.     // extract key and/or value from iterator or stream  

14.     K key;  

15.     V value;  

16.     if (it != null) {  

17.         if (defaultVal==null) {  

18.             Map.Entry<?,?> entry = (Map.Entry<?,?>)it.next();  

19.             key = (K)entry.getKey();  

20.             value = (V)entry.getValue();  

21.         } else {  

22.             key = (K)it.next();  

23.             value = defaultVal;  

24.         }  

25.     } else { // use stream  

26.         key = (K) str.readObject();  

27.         value = (defaultVal != null ? defaultVal : (V) str.readObject());  

28.     }  

29.     Entry<K,V> middle =  new Entry<>(key, value, null); // 创建以MId根的结点  

30.     // color nodes in non-full bottommost level red  

31.     if (level == redLevel)  

32.         middle.color = RED;  

33.     if (left != null) {  

34.         middle.left = left;  

35.         left.parent = middle;  

36.     }  

37.     if (mid < hi) {  

38.         Entry<K,V> right = buildFromSorted(level+1, mid+1, hi, redLevel,  it, str, defaultVal); 

40.         middle.right = right;  

41.         right.parent = middle;  

42.     }  

43.     return middle;  

44. }  
```



**computeRedLevel(size)：计算sortedMap转换成红黑树后的最大高度**

```
1.  private static int computeRedLevel(int sz) {  

2.      int level = 0;  

3.      for (int m = sz - 1; m >= 0; m = m / 2 - 1)  

4.          level++;  

5.      return level;  

6.  }  
```