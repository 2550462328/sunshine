B*树 是B+树的变体，在B+树的非根和非叶子结点再增加指向兄弟的指针；B*树定义了非叶子结点关键字个数至少为(2/3)*M，即块的最低使用率为2/3（代替B+树的1/2）；



一个3阶的B*树如下所示：



![img](http://pcc.huitogo.club/8fc8162f5559551c69d3eee4fad6f75f)



和B+树相比B*树最大的差异就是在分裂节点的时候

- B+树的分裂：当一个结点满时，分配一个新的结点，并将原结点中1/2的数据复制到新结点，最后在父结点中增加新结点的指针；B+树的分裂只影响原结点和父结点，而不会影响兄弟结点，所以它不需要指向兄弟的指针；
- B*树的分裂：当一个结点满时，如果它的下一个兄弟结点未满，那么将一部分数据移到兄弟结点中，再在原结点插入关键字，最后修改父结点中兄弟结点的关键字（因为兄弟结点的关键字范围改变了）；如果兄弟也满了，则在原结点与兄弟结点之间增加新结点，并各复制1/3的数据到新结点，最后在父结点增加新结点的指针；

所以，**B\*树分配新结点的概率比B+树要低，空间使用率更高**；



通过以上介绍，大致将B树，B+树，B*树总结如下：

B树：有序数组+平衡多叉树；

B+树：有序数组链表+平衡多叉树；

B*树：一棵丰满的B+树。