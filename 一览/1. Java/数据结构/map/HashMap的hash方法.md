**为什么HashMap根据 (n - 1) & hash 求出了元素在node数组的下标?**

下面是hash方法的源码:

```
1.  static final int hash(Object key) {  

2.      int h;  

3.      return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);  

4.  }  
```

hash方法的作用是将hashCode进一步的混淆，**增加其“随机度”**，试图减少插入hashmap时的hash冲突，换句更专业的话来说就是提高离散性能。而这个方法知乎上有人回答时称为“**扰动函数**”。

主要分三个阶段: **计算hashcode、高位运算和取模运算**

这里通过key.hashCode()计算出key的哈希值，然后将哈希值h右移16位，再与原来的h做异或^运算——这一步是高位运算。设想一下，如果没有高位运算，那么hash值将是一个int型的32位数。而从2的-31次幂到2的31次幂之间，有将近几十亿的空间，如果我们的每一个HashMap的table都这么长，内存早就爆了，所以这个散列值不能直接用来最终的取模运算，而需要先加入高位运算，将高16位和低16位的信息"融合"到一起，也称为"扰动函数"。这样才能**保证hash值所有位的数值特征都保存下来而没有遗漏，从而使映射结果尽可能的松散**。

最后，根据 n-1做与操作的取模运算。这里也能看出为什么HashMap要限制table的长度为2的n次幂，因为这样，n-1可以保证二进制展示形式是（以16为例）0000 0000 0000 0000 0000 0000 0000 1111。在做"与"操作时，就等同于截取hash二进制值得后四位数据作为下标。这里也可以看出"扰动函数"的重要性了，**如果高位不参与运算，那么高16位的hash特征几乎永远得不到展现，发生hash碰撞的几率就会增大，从而影响性能**。



举例说明流程图如下：

![img](http://pcc.huitogo.club/00df43f7f251cf218f0d4a8bcf134b1a)

