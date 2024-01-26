#### 1. 红黑树构建

当HashMap桶中的元素个数超过一定数量时，就会树化，也就是将链表转化为红黑树的结构。



下面是树化的核心代码：

```
  final void treeifyBin(Node<K,V>[] tab, int hash) {  

      int n, index; Node<K,V> e;  

      if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)  

          resize();  

      else if ((e = tab[index = (n - 1) & hash]) != null) {  

          TreeNode<K,V> hd = null, tl = null;  

          do {  

              //将节点替换为TreeNode  

              TreeNode<K,V> p = replacementTreeNode(e, null);  

             if (tl == null)  　　　　　

                 hd = p;   //hd指向头结点  

             else {  

                 //这里其实是将单链表转化成了双向链表，tl是p的前驱，每次循环更新指向双链表的最后一个元素，用来和p相连，p是当前节点  

                 p.prev = tl;  

                 tl.next = p;  

             }  

             tl = p;  

         } while ((e = e.next) != null);  

         if ((tab[index] = hd) != null)  

            //将链表进行树化  

            hd.treeify(tab);  

     }  

 } 
```

在treeifyBin函数中，先将所有节点替换为TreeNode，然后再将单链表转为双链表，方便之后的遍历和移动操作。而最终的操作，实际上是调用TreeNode的方法treeify进行的。



**treeify(tab)：将链表进行树化**

```
1.  final void treeify(Node<K,V>[] tab) {  

2.              //树的根节点  

3.              TreeNode<K,V> root = null;  

4.              //x是当前节点，next是后继  

5.              for (TreeNode<K,V> x = this, next; x != null; x = next) {  

6.                  next = (TreeNode<K,V>)x.next;  

7.                  x.left = x.right = null;  

8.                  //如果根节点为null，把当前节点设置为根节点  

9.                  if (root == null) {  

10.                     x.parent = null;  

11.                     x.red = false;  

12.                     root = x;  

13.                 }  

14.                 else {  

15.                     K k = x.key;  

16.                     int h = x.hash;  

17.                     Class<?> kc = null;  

18.                     //这里循环遍历，进行二叉搜索树的插入  

19.                     for (TreeNode<K,V> p = root;;) {  //p指向遍历中的当前节点，x为待插入节点，k是x的key，h是x的hash值，ph是p的hash值，dir用来指示x节点与p的比较，-1表示比p小，1表示比p大，不存在相等情况，因为HashMap中是不存在两个key完全一致的情况。  

21.                         int dir, ph;  

22.                         K pk = p.key;  

23.                         if ((ph = p.hash) > h)  

24.                             dir = -1;  

25.                         else if (ph < h)  

26.                             dir = 1;  

27.                         //如果hash值相等，那么判断k是否实现了comparable接口，如果实现了comparable接口就使用compareTo进行进行比较，如果仍旧相等或者没有实现comparable接口，则在tieBreakOrder中比较  

28.                         else if ((kc == null &&   (kc = comparableClassFor(k)) == null) ||   (dir = compareComparables(kc, k, pk)) == 0)  

31.                             dir = tieBreakOrder(k, pk);   //***

32.                         TreeNode<K,V> xp = p;  

33.                         if ((p = (dir <= 0) ? p.left : p.right) == null) {  

34.                             x.parent = xp;  

35.                             if (dir <= 0)  

36.                                 xp.left = x;  

37.                             else  

38.                                 xp.right = x;  

39. 　　　　　  //进行插入平衡处理  

40.                             root = balanceInsertion(root, x);  

41.                             break;  

42.                         }  

43.                     }  

44.                 }  

45.             }  

46. 　　　//确保给定节点是桶中的第一个元素  

47.             moveRootToFront(tab, root);  

48.         }      


49.      //这里不是为了整体排序，而是为了在插入中保持一致的顺序  

50.      static int tieBreakOrder(Object a, Object b) {  

51.             int d;  

52.             //用两者的类名进行比较，如果相同则使用对象默认的hashcode进行比较  

53.             if (a == null || b == null || (d = a.getClass().getName().compareTo(b.getClass().getName())) == 0)  

56.                 d = (System.identityHashCode(a) <= System.identityHashCode(b) ?  -1 : 1);  

58.             return d;  

59.         }    
```

上述代码主要是循环遍历当前树，然后找到可以该节点可以插入的位置，依次和遍历节点比较，比它大则跟其右孩子比较，小则与其左孩子比较，依次遍历，直到找到左孩子或者右孩子为null的位置进行插入。



**moveRootToFront(tab, root)：将root节点移动到桶中的第一个元素**，也就是链表的首节点

```
1.  static <K,V> void moveRootToFront(Node<K,V>[] tab, TreeNode<K,V> root) {  

2.           int n;  

3.           if (root != null && tab != null && (n = tab.length) > 0) {  

4.               int index = (n - 1) & root.hash;  

5.               //first指向链表第一个节点  

6.               TreeNode<K,V> first = (TreeNode<K,V>)tab[index];  

7.               if (root != first) {   //如果root不是第一个节点，则将root放到第一个首节点位置  

9.                    Node<K,V> rn;  

10.                  tab[index] = root;  

11.                  TreeNode<K,V> rp = root.prev;  

12.                  if ((rn = root.next) != null)  

13.                      ((TreeNode<K,V>)rn).prev = rp;  

14.                  if (rp != null)  

15.                      rp.next = rn;  

16.                  if (first != null)  

17.                      first.prev = root;  

18.                  root.next = first;  

19.                  root.prev = null;  

20.              }  

21.              //这里是防御性编程，校验更改后的结构是否满足红黑树和双链表的特性  

22.              //因为HashMap并没有做并发安全处理，可能在并发场景中意外破坏了结构  

23.              assert checkInvariants(root);  

24.          }  

25.      }  
```



**balanceInsertion(root, x)：将红黑树进行插入平衡处理**，保证插入节点后仍保持红黑树性质在讲这个方法之前，红黑树平衡有两种方法，变色和旋转，旋转分为左旋和右旋。

左旋和右旋，就是将节点以某个节点为中心向左或者向右进行旋转操作以保持二叉树的平衡

![img](http://pcc.huitogo.club/d30a91c01b50a3e20621e8cc3b2b0c2d)



![img](http://pcc.huitogo.club/9755eb94e1fd25bdc5f92a174c213a42)



左旋相当于以要旋转的节点为中心，将子树整体向左旋转，该节点变成子树的根节点，原来的父节点A变成了左孩子，如果右孩子C有左孩子D，则将其变为A的右孩子。可以抽象的理解，当节点A向左旋转之后，C的左孩子D可以理解为因为重力作用掉到A的右孩子位置。右旋类似理解。



在HashMap中将上述过程用代码实现就是

```
1.  /** 
2.   * 左旋 
3.   */  

4.  static <K,V> TreeNode<K,V> rotateLeft(TreeNode<K,V> root,  TreeNode<K,V> p) {  

6.      //这里的p即上图的A节点，r指向右孩子即C，rl指向右孩子的左孩子即D，pp为p的父节点  

7.      TreeNode<K,V> r, pp, rl;  

8.      if (p != null && (r = p.right) != null) {  

9.          if ((rl = p.right = r.left) != null)  

10.             rl.parent = p;  

11.         //将p的父节点的孩子节点指向r  

12.         if ((pp = r.parent = p.parent) == null)  

13.             (root = r).red = false;  

14.         else if (pp.left == p)  

15.             pp.left = r;  

16.         else  

17.             pp.right = r;  

18.         //将p置为r的左节点  

19.         r.left = p;  

20.         p.parent = r;  

21.     }  

22.     return root;  

23. }  


24. /** 
25.  * 右旋 
26.  */  

27. static <K,V> TreeNode<K,V> rotateRight(TreeNode<K,V> root,   TreeNode<K,V> p) {  

29.      //这里的p即上图的A节点，l指向左孩子即C，lr指向左孩子的右孩子即E，pp为p的父节点  

30.     TreeNode<K,V> l, pp, lr;  

31.     if (p != null && (l = p.left) != null) {  

32.         if ((lr = p.left = l.right) != null)  

33.             lr.parent = p;  

34.         if ((pp = l.parent = p.parent) == null)  

35.             (root = l).red = false;  

36.         else if (pp.right == p)  

37.             pp.right = l;  

38.         else  

39.             pp.left = l;  

40.         l.right = p;  

41.         p.parent = l;  

42.     }  

43.     return root;  

44. }  
```



现在再来看一下红黑树插入的过程，对于插进来的一个TreeNode，可以分以下情况进行考虑

1）插入的为根节点，则直接把颜色改成黑色即可。

2）插入的节点的父节点是黑色节点，则不需要调整，因为插入的节点会初始化为红色节点，红色节点是不会影响树的平衡的。

3）插入的节点的祖父节点为null，即插入的节点的父节点是根节点，直接插入即可（因为根节点肯定是黑色）。

4）插入的节点父节点和祖父节点都存在，并且其父节点是祖父节点的左节点。这种情况稍微麻烦一点，又分两种子情况：

　A. 插入节点的叔叔节点是红色，则将父亲节点和叔叔节点都改成黑色，然后祖父节点改成红色即可。

　B. 插入节点的叔叔节点是黑色或不存在：

　　a.若插入节点是其父节点的右孩子，则将其父节点左旋，

　　b.若为左孩子，则将其父节点变成黑色节点，将其祖父节点变成红色节点，然后将其祖父节点右旋。

5）插入的节点父节点和祖父节点都存在，并且其父节点是祖父节点的右节点。这种情况跟上面是类似的，分两种子情况：

　A. 插入节点的叔叔节点是红色，则将父亲节点和叔叔节点都改成黑色，然后祖父节点改成红色即可。

　B. 插入节点的叔叔节点是黑色或不存在：

　　a.若插入节点是其父节点的左孩子，则将其父节点右旋

　　b.若为右孩子，则将其父节点变成黑色节点，将其祖父节点变成红色节点，然后将其祖父节点左旋。



然后重复进行上述操作，直到变成1或2情况时则结束变换

举例说明，向红黑树中插入数据的顺序是：10，5，9，3，6，7，19，32，24

对应的过程如下：

![img](http://pcc.huitogo.club/3a2b85157e50acfd114d5d0598b8b243)

![img](http://pcc.huitogo.club/c66eb2eb8283b00d6d6892b7fb84138f)



以上情况的出发都是为了满足红黑树的特性，特别是“**从一个节点到该节点的子孙节点的所有路径上包含相同数目的黑节点**”，下面对于红黑树的删除也是，为了“**平衡**”



根据以上红黑树的插入TreeNode的过程，在HashMap中实现如下：

```
1.  static <K,V> TreeNode<K,V> balanceInsertion(TreeNode<K,V> root,  TreeNode<K,V> x) {  

3.         x.red = true;  

4.         for (TreeNode<K,V> xp, xpp, xppl, xppr;;) {  

5.             //情景1：父节点为null  

6.             if ((xp = x.parent) == null) {  

7.                 x.red = false;  

8.                 return x;  

9.             }  

10. 　　//情景2，3：父节点是黑色节点或者祖父节点为null  

11.            else if (!xp.red || (xpp = xp.parent) == null)  

12.                return root;  


13. 　　//情景4：插入的节点父节点和祖父节点都存在，并且其父节点是祖父节点的左节点  

14.            if (xp == (xppl = xpp.left)) {  

15. 　　　//情景4A：插入节点的叔叔节点是红色  

16.                if ((xppr = xpp.right) != null && xppr.red) {  

17.                    xppr.red = false;  

18.                    xp.red = false;  

19.                    xpp.red = true;  

20.                    x = xpp;  

21.                }  

22. 　　　//情景4B：插入节点的叔叔节点是黑色或不存在  

23.                else {  

24. 　　　　//情景4Ba：插入节点是其父节点的右孩子  

25.                    if (x == xp.right) {  

26.                        root = rotateLeft(root, x = xp); //*** 

27.                        xpp = (xp = x.parent) == null ? null : xp.parent;  

28.                    }  

29. 　　　　//情景4Bb：插入节点是其父节点的左孩子  

30.                    if (xp != null) {  

31.                        xp.red = false;  

32.                        if (xpp != null) {  

33.                            xpp.red = true;  

34.                            root = rotateRight(root, xpp);   //***

35.                        }  

36.                    }  

37.                }  

38.            }  


39. 　　 //情景5：插入的节点父节点和祖父节点都存在，并且其父节点是祖父节点的右节点  

40.            else {  

41. 　　　//情景5A：插入节点的叔叔节点是红色  

42.                if (xppl != null && xppl.red) {  

43.                    xppl.red = false;  

44.                    xp.red = false;  

45.                    xpp.red = true;  

46.                    x = xpp;  

47.                }  

48. 　　　//情景5B：插入节点的叔叔节点是黑色或不存在  

49.                else {  

50. 　　　　//情景5Ba：插入节点是其父节点的左孩子　  

51.                    if (x == xp.left) {  

52.                        root = rotateRight(root, x = xp);  //***

53.                        xpp = (xp = x.parent) == null ? null : xp.parent;  

54.                    }  

55. 　　　　//情景5Bb：插入节点是其父节点的右孩子  

56.                    if (xp != null) {  

57.                        xp.red = false;  

58.                        if (xpp != null) {  

59.                            xpp.red = true;  

60.                            root = rotateLeft(root, xpp);  //***

61.                        }  

62.                    }  

63.                }  

64.            }  

65.        }  

66.    }  
```



#### 2. 红黑树删除

红黑树是一颗特殊的二叉搜索树，所以进行删除操作时，其实是先进行二叉搜索树的删除，然后再进行调整。所以，其实这里分为两部分内容：1.二叉搜索树的删除，2.红黑树的删除调整。

二叉搜索树的删除主要有这么几种情景：

情景1：待删除的节点无左右孩子。

情景2：待删除的节点只有左孩子或者右孩子。

情景3：待删除的节点既有左孩子又有右孩子。

对于情景1，直接删除即可

对于情景2，则直接把该节点的父节点指向它的左孩子或者右孩子即可

对于情景3，需要先找到其右子树的最左孩子（或者左子树的最右孩子），即左（右）子树中序遍历时的第一个节点，然后将其与待删除的节点互换，最后再删除该节点（如果有右子树，则右子树上位）。**总之，就是先找到它的替代者，找到之后替换这个要删除的节点**，然后再把这个节点真正删除掉。**这个替代者就是最接近被删除节点的子节点，即左子树的最大值或者右子树的最小值**。



对于情景3，删除完节点后可能会导致**左子树和右子树路径中黑色节点数量不一致**，所以需要进行**红黑树的调整**。

1）只有右孩子且为红色，直接用右孩子替换该节点然后变成黑色即可

2） 只有右孩子且为黑色，那么删除该节点会导致父节点的左子树路径上黑色节点减一，此时只能去借助右子树，从右子树中借一个红色节点过来即可，具体取决于右子树的情况，这里又分成两种：

　A. 兄弟节点是红色，则此时父节点是黑色，且兄弟节点肯定有两个孩子，且兄弟节点的左右子树路径上均有两个黑色节点，此时只需将兄弟节点与父节点颜色互换，然后将父节点左旋，左旋后，兄弟节点的左子树SL挂到了父节点p的右孩子位置，这时会导致p的右子树路径上的黑色节点比左子树多一，此时再SL置为红色即可。

![img](http://pcc.huitogo.club/8f8c0712bb035744f213fd33f2ef0d70)

　B. 兄弟节点是黑色，那么就只能打它孩子的主意了，这里主要关注远侄子（兄弟节点的右孩子，即SR）的颜色情况，这里分成两种情况：

　　a. 远侄子SR是黑色，近侄子任意（白色代表颜色可为任意颜色），则先将S转为红色，然后右旋，再将SL换成P节点颜色，P涂成黑色，S也涂成黑色，再进行左旋即可。其实简单说就是SL上位，替换父节点位置。

![img](http://pcc.huitogo.club/47ab557884f54901ebf059b3204fdd2a)

　　b. 远侄子SR为红色，近侄子任意（该子树路径中有且仅有一个黑色节点），则先将兄弟节点与父节点颜色互换，将SR涂成黑色，再将父节点左旋即可

![img](http://pcc.huitogo.club/32e815db637bcd8643621a302fcd4ce6)



在讲解完红黑树的节点删除后看一下HashMap中对于TreeNode的删除

```
1.  final void removeTreeNode(HashMap<K,V> map, Node<K,V>[] tab, boolean movable) {  

2.       ......  

3.       //p是待删除节点，replacement用于后续的红黑树调整，指向的是p或者p的继承者。  

4.       //如果p是叶子节点，p==replacement，否则replacement为p的右子树中最左节点  

5.       if (replacement != p) {  

6.          //若p不是叶子节点，则让replacement的父节点指向p的父节点  

7.          TreeNode<K,V> pp = replacement.parent = p.parent;  

8.          if (pp == null)  

9.              root = replacement;  

10.         else if (p == pp.left)  

11.             pp.left = replacement;  

12.         else  

13.             pp.right = replacement;  

14.         p.left = p.right = p.parent = null;  

15.     }  

16.     //若待删除的节点p时红色的，则树平衡未被破坏，无需进行调整。  

17.     //否则删除节点后需要进行调整  

18.     TreeNode<K,V> r = p.red ? root : balanceDeletion(root, replacement);   //***

19.     //p为叶子节点，则直接将p从树中清除  

20.     if (replacement == p) {  // detach  

21.         TreeNode<K,V> pp = p.parent;  

22.         p.parent = null;  

23.         if (pp != null) {  

24.             if (p == pp.left)  

25.                 pp.left = null;  

26.             else if (p == pp.right)  

27.                 pp.right = null;  

28.         }  

29.     }  

30. }  
```



**balanceDeletion(root, replacement)：删除节点后的调整**，也就是在红黑树中讲解的维持红黑树平衡的情形，两个参数分别表示根节点和删除节点的继承者

```
1.  static <K,V> TreeNode<K,V> balanceDeletion(TreeNode<K,V> root, TreeNode<K,V> x) {  

2.      for (TreeNode<K,V> xp, xpl, xpr;;)  {  

3.          //x为空或x为根节点，直接返回  

4.          if (x == null || x == root)  

5.              return root;   

6.          //x为根节点，染成黑色，直接返回（因为调整过后，root并不一定指向删除操作过后的根节点，如果之前删除的是root节点，则x将成为新的根节点）  

7.          else if ((xp = x.parent) == null) {  

8.              x.red = false;   

9.              return x;  

10.         }  

11.         //如果x为红色，则无需调整，返回  

12.         else if (x.red) {  

13.             x.red = false;  

14.             return root;   

15.         }  


16.         //x为其父节点的左孩子  

17.         else if ((xpl = xp.left) == x) {  

18.             //如果它有红色的兄弟节点xpr，那么它的父亲节点xp一定是黑色节点  

19.             if ((xpr = xp.right) != null && xpr.red) {   

20.                 xpr.red = false;  

21.                 xp.red = true;   

22.                 //对父节点xp做左旋转  

23.                 root = rotateLeft(root, xp);   

24.                 //重新将xp指向x的父节点，xpr指向xp新的右孩子  

25.                 xpr = (xp = x.parent) == null ? null : xp.right;   

26.             }  

27.             //如果xpr为空，则向上继续调整，将x的父节点xp作为新的x继续循环  

28.             if (xpr == null)  

29.                 x = xp;   

30.             else {  

31.                 //sl和sr分别为其近侄子和远侄子  

32.                 TreeNode<K,V> sl = xpr.left, sr = xpr.right;  

33.                 if ((sr == null || !sr.red) &&  

34.                     (sl == null || !sl.red)) {  

35.                     xpr.red = true; //若sl和sr都为黑色或者不存在，即xpr没有红色孩子，则将xpr染红  

36.                     x = xp; //本轮结束，继续向上循环  

37.                 }  

38.                 else {  

39.                     //否则的话，就需要进一步调整  

40.                     if (sr == null || !sr.red) {   

41.                         if (sl != null) //若左孩子为红，右孩子不存在或为黑  

42.                             sl.red = false; //左孩子染黑  

43.                         xpr.red = true; //将xpr染红  

44.                         root = rotateRight(root, xpr); //右旋  

45.                         xpr = (xp = x.parent) == null ?  null : xp.right;  //右旋后，xpr指向xp的新右孩子，即上一步中的sl  

47.                     }  

48.                     if (xpr != null) {  

49.                         xpr.red = (xp == null) ? false : xp.red; //xpr染成跟父节点一致的颜色，为后面父节点xp的左旋做准备  

50.                         if ((sr = xpr.right) != null)  

51.                             sr.red = false; //xpr新的右孩子染黑，防止出现两个红色相连  

52.                     }  

53.                     if (xp != null) {  

54.                         xp.red = false; //将xp染黑，并对其左旋，这样就能保证被删除的X所在的路径又多了一个黑色节点，从而达到恢复平衡的目的  

55.                         root = rotateLeft(root, xp);  

56.                     }  

57.                     //到此调整已经完毕，进入下一次循环后将直接退出  

58.                     x = root;  

59.                 }  

60.             }  

61.         }  


62.         //x为其父节点的右孩子，跟上面类似  

63.         else { 

64.             if (xpl != null && xpl.red) {  

65.                 xpl.red = false;  

66.                 xp.red = true;  

67.                 root = rotateRight(root, xp);  

68.                 xpl = (xp = x.parent) == null ? null : xp.left;  

69.             }  

70.             if (xpl == null)  

71.                 x = xp;  

72.             else {  

73.                 TreeNode<K,V> sl = xpl.left, sr = xpl.right;  

74.                 if ((sl == null || !sl.red) &&  

75.                     (sr == null || !sr.red)) {  

76.                     xpl.red = true;  

77.                     x = xp;  

78.                 }  

79.                 else {  

80.                     if (sl == null || !sl.red) {  

81.                         if (sr != null)  

82.                             sr.red = false;  

83.                         xpl.red = true;  

84.                         root = rotateLeft(root, xpl);  

85.                         xpl = (xp = x.parent) == null ?  null : xp.left;  

87.                     }  

88.                     if (xpl != null) {  

89.                         xpl.red = (xp == null) ? false : xp.red;  

90.                         if ((sl = xpl.left) != null)  

91.                             sl.red = false;  

92.                     }  

93.                     if (xp != null) {  

94.                         xp.red = false;  

95.                         root = rotateRight(root, xp);  

96.                     }  

97.                     x = root;  

98.                 }  

99.             }  

100.         }  

101.     }  

102. }  
```