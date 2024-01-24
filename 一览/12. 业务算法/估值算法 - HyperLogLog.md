#### 1. HyperLogLog 简介

HyperLogLog 是最早由 [Flajolet](http://algo.inria.fr/flajolet/Publications/FlFuGaMe07.pdf) 及其同事在 2007 年提出的一种 估算基数的近似最优算法。但跟原版论文不同的是，好像很多书包括 Redis 作者都把它称为一种 新的数据结构(new datastruct) (算法实现确实需要一种特定的数据结构来实现)。



**什么是基数统计？**

基数统计(Cardinality Counting) 通常是用来统计一个集合中不重复的元素个数。

思考这样的一个场景： 如果你负责开发维护一个大型的网站，有一天老板找产品经理要网站上每个网页的 UV(独立访客，每个用户每天只记录一次)，然后让你来开发这个统计模块，你会如何实现？

如果统计 **PV(浏览量，用户每点一次记录一次)**，那非常好办，给每个页面配置一个独立的 Redis 计数器就可以了，把这个计数器的 key 后缀加上当天的日期。这样每来一个请求，就执行 INCRBY 指令一次，最终就可以统计出所有的 PV 数据了。

但是UV 不同，它要去重，同一个用户一天之内的多次访问请求只能计数一次。这就要求了每一个网页请求都需要带上用户的 ID，无论是登录用户还是未登录的用户，都需要一个唯一 ID 来标识。



你也许马上就想到了一个 简单的解决方案：那就是 为每一个页面设置一个独立的 set 集合 来存储所有当天访问过此页面的用户 ID。但这样的问题 就是：

1. 存储空间巨大： 如果网站访问量一大，你需要用来存储的 set 集合就会非常大，如果页面再一多.. 为了一个去重功能耗费的资源就可以直接让你 老板打死你；
2. 统计复杂： 这么多 set 集合如果要聚合统计一下，又是一个复杂的事情；



**有哪些用于基数统计的常用方法？**

**第一种：B 树**

**B 树最大的优势就是插入和查找效率很高**，如果用 B 树存储要统计的数据，可以快速判断新来的数据是否存在，并快速将元素插入 B 树。要计算基础值，只需要计算 B 树的节点个数就行了。



不过将 B 树结构维护到内存中，能够解决统计和计算的问题，但是 **并没有节省内存**。



**第二种：bitmap**

**bitmap** 可以理解为通过一个 bit 数组来存储特定数据的一种数据结构，**每一个 bit 位都能独立包含信息**，bit 是数据的最小存储单位，因此能大量节省空间，也可以将整个 bit 数据一次性 load 到内存计算。如果定义一个很大的 bit 数组，基础统计中 **每一个元素对应到 bit 数组中的一位**，例如：

![img](http://pcc.huitogo.club/c61ad3e90cd6ae02c0948481ae670b06)



bitmap 还有一个明显的优势是 可以轻松合并多个统计结果，只需要对多个结果求异或就可以了，也可以大大减少存储内存。可以简单做一个计算，如果要统计 1 亿 个数据的基数值，大约需要的内存：100_000_000/ 8/ 1024/ 1024 ≈ 12 M，如果用 32 bit 的 int 代表 每一个 统计的数据，大约需要内存：32 * 100_000_000/ 8/ 1024/ 1024 ≈ 381 M



可以看到 bitmap 对于内存的节省显而易见，但仍然不够。统计一个对象的基数值就需要 12 M，如果统计 1 万个对象，就需要接近 120 G，对于大数据的场景仍然不适用。



**第三种：概率算法**

实际上目前还没有发现更好的在 大数据场景 中 准确计算 基数的高效算法，因此在不追求绝对精确的情况下，使用概率算法算是一个不错的解决方案。



概率算法 **不直接存储** 数据集合本身，通过一定的 **概率统计方法预估基数值**，这种方法可以大大节省内存，同时保证误差控制在一定范围内。



目前用于基数计数的概率算法包括:

- **Linear Counting**(LC)：早期的基数估计算法，LC 在空间复杂度方面并不算优秀，实际上 LC 的空间复杂度与上文中简单 bitmap 方法是一样的（但是有个常数项级别的降低），都是 O(Nmax)
- **LogLog Counting**(LLC)：LogLog Counting 相比于 LC 更加节省内存，空间复杂度只有 O(log2(log2(Nmax)))
- **HyperLogLog Counting**(HLL)：HyperLogLog Counting 是基于 LLC 的优化和改进，在同样空间复杂度情况下，能够比 LLC 的基数估计误差更小



其中，HyperLogLog 的表现是惊人的，上面我们简单计算过用 bitmap 存储 1 个亿 统计数据大概需要 12 M 内存，而在 HyperLoglog 中，只需要不到 1 K 内存就能够做到！在 Redis 中实现的 HyperLoglog 也只需要 12 K 内存，在 标准误差 0.81% 的前提下，能够统计 264 个数据！



#### 2. HyperLogLog 原理

我们来思考一个抛硬币的游戏：你连续掷 n 次硬币，然后说出其中**连续掷为正面的最大次数，我来猜你一共抛了多少次。**



根据概率学计算方式是如下的：

![img](http://pcc.huitogo.club/a895bf9287ae345ca2bb8e2e5113ece9)



以抛硬币序列"1110100110"为例，其中最长的反面序列是"00"，我们顺手把后面那个1也给带上，也就是"001"，因为它包括了序列中最长的一串0，所以在序列中肯定只出现过一次，而它在任意序列出现出现且仅出现一次的概率显然是上图所示的三个二分之一相乘，也就是八分之一，所以我可以给出一个估计值，你大概总共抛了8次硬币。



很显然，上面这种做法虽然能够估计抛硬币的总数，但是显然误差是比较大的，很容易受到突发事件（比如突然连续抛出好多0）的影响，**HyperLogLog算法研究的就是如何减小这个误差。**



之前说过，HyperLogLog算法是用来计算基数的，这个抛硬币的序列和基数有什么关系呢？比如在数据库中，我只要在每次插入一条新的记录时，计算这条记录的hash，并且转换成二进制，就可以将其看成一个硬币序列了，如下(0b前缀表示二进制数)：

![img](http://pcc.huitogo.club/dc21f27522a4524094bfad6656bb83f1)



##### 2.1 最简单的想法

根据上面抛硬币的启发我可以想到如下的估计基数的算法（这里先给出伪代码）

```
1. 输入：一个集合 

2. 输出：集合的基数 

3. 算法： 

4.   max = 0 

5.   对于集合中的每个元素： 

6.        hashCode = hash(元素) 

7.        num = hashCode二进制表示中最前面连续的0的数量 

8.        if num > max: 

9.          max = num 

10.   最后的结果是2的(max + 1)次幂  
```



举个例子，对于集合{ele1, ele2}，先求hash(ele1)=0b00110111，它最前面的连续的0的数量为2（又称为前导0），然后求hash(ele2)=0b10010000111，它的前导0数量为0，我们始终只保存前导零数量的最大值，所以最后max是2，我们估计的基数就是2的(2+1)次幂，即8。


为什么最后的max要加1呢？这是一个数学细节，具体要看论文，简单的理解的话，可以像之前抛硬币的例子那样理解，把最长的一串零的后面的一个1或者前面的一个1"顺手"带上进行概率估计。

显然这个算法是非常不准确的，但是这个想法还是很有启发性的，从这个简单的想法跟随下文一步一步优化即可得到最终的比较高精度的HyperLogLog算法。



##### 2.2 分桶

最简单的一种优化方法显然就是**把数据分成m个均等的部分**，分别估计其总数求平均后再乘以m，称之为分桶。对应到前面抛硬币的例子，其实就是把硬币序列分成m个均等的部分，分别用之前提到的那个方法估计总数求平均后再乘以m，这样就能一定程度上避免单一突发事件造成的误差。



具体要怎么分桶呢？我们可以将每个元素的hash值的二进制表示的前几位用来指示数据属于哪个桶，然后把剩下的部分再按照之前最简单的想法处理。



还是以刚刚的那个集合{ele1,ele2}为例，假设我要分2个桶，那么我只要去ele1的hash值的第一位来确定其分桶即可，之后用剩下的部分进行前导零的计算，如下图：

![img](http://pcc.huitogo.club/fc4a13d0fdf8f486ec1c8923ba6fcc31)



到这里，你大概已经理解了LogLog算法的基本思想，LogLog算法是在HyperLogLog算法之前提出的一个基数估计算法，HyperLogLog算法其实就是LogLog算法的一个改进版。



LogLog算法完整的基数计算公式如下：

![img](http://pcc.huitogo.club/044b93ed15bda5261d6d2ca06f7610e6)



其中m代表分桶数，R头上一道横杠的记号就代表每个桶的结果（其实就是桶中数据的最长前导零+1）的均值，相比我之前举的简单的例子，LogLog算法还乘了一个常数constant进行修正。



##### 2.3 调和平均数

前面的LogLog算法中我们是使用的是平均数来将每个桶的结果汇总起来，但是平均数有一个广为人知的缺点，就是容易受到大的数值的影响，一个常见的例子是，假如我的工资是1000元一个月，我老板的工资是100000元一个月，那么我和老板的平均工资就是(100000 + 1000)/2，即50500元，显然这离我的工资相差甚远，我肯定不服这个平均工资。



用调和平均数就可以解决这一问题，调和平均数的结果会倾向于集合中比较小的数，x1到xn的调和平均数的公式如下：

![img](http://pcc.huitogo.club/fe22644c5bc98603b0fe713160a2f543)



再用这个公式算一下我和老板的平均工资：

![img](http://pcc.huitogo.club/c913a26f14581dcb4dc8cdaf22ffc1de)



最后的结果是1980元，这和我的工资水平还比较接近，这样的平均工资水平我才比较信服。

再回到前面的LogLog算法，从前面的举的例子可以看出，影响LogLog算法精度的一个重要因素就是，hash值的前导零的数量显然是有很大的偶然性的，经常会出现一两数据前导零的数目比较多的情况，所以HyperLogLog算法相比LogLog算法一个重要的改进就是使用调和平均数而不是平均数来聚合每个桶中的结果，HyperLogLog算法的公式如下：

![img](http://pcc.huitogo.club/3f13df0895ad6a3ad868983a18342884)



##### 2.4 细节微调

算法中还有一些细微的校正，在数据总量比较小的时候，很容易就预测偏大，所以我们做如下校正：

![img](http://pcc.huitogo.club/e720dca1c0892b71cf9300f8d0dc8c43)



DV代表估计的基数值，m代表桶的数量，V代表结果为0的桶的数目，log表示自然对数



我再详细解释一下V的含义，假设我分配了64个桶（即m=64），当数据量很小时（比方说只有两三个），那肯定有大量桶中没有数据，也就说他们的估计值是0，V就代表这样的桶的数目。



事实证明，这个校正的效果是非常好，在数据量小的时，估计得非常准确，有兴趣可以去玩一下外国大佬制作的一个HyperLogLog算法的仿真：

> [http://content.research.neustar.biz/blog/hll.html](https://links.jianshu.com/go?to=http://content.research.neustar.biz/blog/hll.html)



##### 2.5 参数选择

- **m桶数**

很显然分桶数只能是2的整数次幂。



如果分桶越多，那么估计的精度就会越高，统计学上用来衡量估计精度的一个指标是“相对标准误差”(relative standard deviation，简称RSD)，RSD的计算公式这里就不给出了，百科上一搜就可以知道，从直观上理解，RSD的值其实就是（(每次估计的值）在（估计均值）上下的波动）占（估计均值）的比例（这句话加那么多括号是为了方便大家断句）。RSD的值与分桶数m存在如下的计算关系：

![img](http://pcc.huitogo.club/01877e17e2639783beca1684abeb8ea6)



有了这个公式，你可以先确定你想要达到的RSD的值，然后再推出分桶的数目m。



- **constant常数**

constant常数的选择与分桶的数目有关，具体的数学证明请看论文，这里就直接给出结论： 假设：m为分桶数，p是m的以2为底的对数

![img](http://pcc.huitogo.club/9a9a141aad272fbf4b0f764c6895882e)



则按如下的规则计算constant

```
1. switch (p) { 

2.  case 4: 

3.    constant = 0.673 * m * m; 

4.  case 5: 

5.    constant = 0.697 * m * m; 

6.  case 6: 

7.    constant = 0.709 * m * m; 

8.  default: 

9.    constant = (0.7213 / (1 + 1.079 / m)) * m * m; 

10. } 
```



##### 2.6 合并

假设有两个数据流，分别构建了两个HyperLogLog结构，称为a和b，他们的桶数是一样的，为n，现在要计算两个数据流总体的基数。

```
1. 数据流a："a" "b" "c" "d" 基数：4 

2. 数据流b："b" "c" "d" "e" 基数：4 

3. 两个数据流的总体基数：5 
```



从前文我们可以知道，HyperLogLog算法在内存中的结构其实就是一个桶数组，需要先用下面的算法从a和我b的桶数组中构建出新的桶数组c，其实就是从a,b的对应位置取最大的：

```
1. 输入：桶数组a，b。它们的长度都是n 

2. 输出：新的桶数组c 

3. 算法： 

4.   c = c[n]; 

5.   for (i=0; i<n; i++){ 

6.     c[i]=max(a[i], b[i]); 

7.   } 

8.   return c; 
```



之后用桶数组c代入前面的算法即可得到合并的总体基数。



#### 4. Java实现HyperLogLog

stream-lib是一个开源的Java流式计算库，里面有很多大数据估值算法的实现，其中当然包括HyperLogLog算法，HyperLogLog实现类的代码地址如下： [https://github.com/addthis/stream-lib/blob/master/src/main/java/com/clearspring/analytics/stream/cardinality/HyperLogLog.java](https://links.jianshu.com/go?to=https://github.com/addthis/stream-lib/blob/master/src/main/java/com/clearspring/analytics/stream/cardinality/HyperLogLog.java)



其中提取出来可以分成三部分



**1）HyperLogLog.java**

```
1. public class HyperLogLog { 

2.  

3.   private final RegisterSet registerSet; 

4.   private final int log2m;  //log(m) 

5.   private final double alphaMM; 

6.  

7.  

8.   /** 

9.   * 

10.   * rsd = 1.04/sqrt(m) 

11.   * @param rsd 相对标准偏差 

12.   */ 

13.   public HyperLogLog(double rsd) { 

14.     this(log2m(rsd)); 

15.   } 

16.  

17.   /** 

18.   * rsd = 1.04/sqrt(m) 

19.   * m = (1.04 / rsd)^2 

20.   * @param rsd 相对标准偏差 

21.   * @return 

22.   */ 

23.   private static int log2m(double rsd) { 

24.     return (int) (Math.log((1.106 / rsd) * (1.106 / rsd)) / Math.log(2)); 

25.   } 

26.  

27.   private static double rsd(int log2m) { 

28.     return 1.106 / Math.sqrt(Math.exp(log2m * Math.log(2))); 

29.   } 

30.  

31.  

32.   /** 

33.   * accuracy = 1.04/sqrt(2^log2m) 

34.   * 

35.   * @param log2m 

36.   */ 

37.   public HyperLogLog(int log2m) { 

38.     this(log2m, new RegisterSet(1 << log2m)); 

39.   } 

40.  

41.   /** 

42.   * 

43.   * @param registerSet 

44.   */ 

45.   public HyperLogLog(int log2m, RegisterSet registerSet) { 

46.     this.registerSet = registerSet; 

47.     this.log2m = log2m; 

48.     int m = 1 << this.log2m; //从log2m中算出m 

49.  

50.     alphaMM = getAlphaMM(log2m, m); 

51.   } 

52.  

53.  

54.   public boolean offerHashed(int hashedValue) { 

55.     // j 代表第几个桶,取hashedValue的前log2m位即可 

56.     // j 介于 0 到 m 

57.     final int j = hashedValue >>> (Integer.SIZE - log2m); 

58.     // r代表 除去前log2m位剩下部分的前导零 + 1 

59.     final int r = Integer.numberOfLeadingZeros((hashedValue << this.log2m) | (1 << (this.log2m - 1)) + 1) + 1; 

60.     return registerSet.updateIfGreater(j, r); 

61.   } 

62.  

63.   /** 

64.   * 添加元素 

65.   * @param o 要被添加的元素 

66.   * @return 

67.   */ 

68.   public boolean offer(Object o) { 

69.     final int x = MurmurHash.hash(o); 

70.     return offerHashed(x); 

71.   } 

72.  

73.  

74.   public long cardinality() { 

75.     double registerSum = 0; 

76.     int count = registerSet.count; 

77.     double zeros = 0.0; 

78.     //count是桶的数量 

79.     for (int j = 0; j < registerSet.count; j++) { 

80.       int val = registerSet.get(j); 

81.       registerSum += 1.0 / (1 << val); 

82.       if (val == 0) { 

83.         zeros++; 

84.       } 

85.     } 

86.  

87.     double estimate = alphaMM * (1 / registerSum); 

88.  

89.     if (estimate <= (5.0 / 2.0) * count) { //小数据量修正 

90.       return Math.round(linearCounting(count, zeros)); 

91.     } else { 

92.       return Math.round(estimate); 

93.     } 

94.   } 

95.  

96.  

97.   /** 

98.   * 计算constant常数的取值 

99.   * @param p  log2m 

100.   * @param m  m 

101.   * @return 

102.   */ 

103.   protected static double getAlphaMM(final int p, final int m) { 

104.     // See the paper. 

105.     switch (p) { 

106.       case 4: 

107.         return 0.673 * m * m; 

108.       case 5: 

109.         return 0.697 * m * m; 

110.       case 6: 

111.         return 0.709 * m * m; 

112.       default: 

113.         return (0.7213 / (1 + 1.079 / m)) * m * m; 

114.     } 

115.   } 

116.  

117.   /** 

118.   * 

119.   * @param m  桶的数目 

120.   * @param V  桶中0的数目 

121.   * @return 

122.   */ 

123.   protected static double linearCounting(int m, double V) { 

124.     return m * Math.log(m / V); 

125.   } 

126.  

127.   public static void main(String[] args) { 

128.     HyperLogLog hyperLogLog = new HyperLogLog(0.1325);//64个桶 

129.     //集合中只有下面这些元素 

130.     hyperLogLog.offer("hhh"); 

131.     hyperLogLog.offer("mmm"); 

132.     hyperLogLog.offer("ccc"); 

133.     //估算基数 

134.     System.out.println(hyperLogLog.cardinality()); 

135.   } 

136. } 
```



**2）MurmurHash.java**

```
1. /** 

2. * 一种快速的非加密hash 

3. * 适用于对保密性要求不高以及不在意hash碰撞攻击的场合 

4. */ 

5. public class MurmurHash { 

6.  

7.   public static int hash(Object o) { 

8.     if (o == null) { 

9.       return 0; 

10.     } 

11.     if (o instanceof Long) { 

12.       return hashLong((Long) o); 

13.     } 

14.     if (o instanceof Integer) { 

15.       return hashLong((Integer) o); 

16.     } 

17.     if (o instanceof Double) { 

18.       return hashLong(Double.doubleToRawLongBits((Double) o)); 

19.     } 

20.     if (o instanceof Float) { 

21.       return hashLong(Float.floatToRawIntBits((Float) o)); 

22.     } 

23.     if (o instanceof String) { 

24.       return hash(((String) o).getBytes()); 

25.     } 

26.     if (o instanceof byte[]) { 

27.       return hash((byte[]) o); 

28.     } 

29.     return hash(o.toString()); 

30.   } 

31.  

32.   public static int hash(byte[] data) { 

33.     return hash(data, data.length, -1); 

34.   } 

35.  

36.   public static int hash(byte[] data, int seed) { 

37.     return hash(data, data.length, seed); 

38.   } 

39.  

40.   public static int hash(byte[] data, int length, int seed) { 

41.     int m = 0x5bd1e995; 

42.     int r = 24; 

43.  

44.     int h = seed ^ length; 

45.  

46.     int len_4 = length >> 2; 

47.  

48.     for (int i = 0; i < len_4; i++) { 

49.       int i_4 = i << 2; 

50.       int k = data[i_4 + 3]; 

51.       k = k << 8; 

52.       k = k | (data[i_4 + 2] & 0xff); 

53.       k = k << 8; 

54.       k = k | (data[i_4 + 1] & 0xff); 

55.       k = k << 8; 

56.       k = k | (data[i_4 + 0] & 0xff); 

57.       k *= m; 

58.       k ^= k >>> r; 

59.       k *= m; 

60.       h *= m; 

61.       h ^= k; 

62.     } 

63.  

64.     // avoid calculating modulo 

65.     int len_m = len_4 << 2; 

66.     int left = length - len_m; 

67.  

68.     if (left != 0) { 

69.       if (left >= 3) { 

70.         h ^= (int) data[length - 3] << 16; 

71.       } 

72.       if (left >= 2) { 

73.         h ^= (int) data[length - 2] << 8; 

74.       } 

75.       if (left >= 1) { 

76.         h ^= (int) data[length - 1]; 

77.       } 

78.  

79.       h *= m; 

80.     } 

81.  

82.     h ^= h >>> 13; 

83.     h *= m; 

84.     h ^= h >>> 15; 

85.  

86.     return h; 

87.   } 

88.  

89.   public static int hashLong(long data) { 

90.     int m = 0x5bd1e995; 

91.     int r = 24; 

92.  

93.     int h = 0; 

94.  

95.     int k = (int) data * m; 

96.     k ^= k >>> r; 

97.     h ^= k * m; 

98.  

99.     k = (int) (data >> 32) * m; 

100.     k ^= k >>> r; 

101.     h *= m; 

102.     h ^= k * m; 

103.  

104.     h ^= h >>> 13; 

105.     h *= m; 

106.     h ^= h >>> 15; 

107.  

108.     return h; 

109.   } 

110.  

111.   public static long hash64(Object o) { 

112.     if (o == null) { 

113.       return 0l; 

114.     } else if (o instanceof String) { 

115.       final byte[] bytes = ((String) o).getBytes(); 

116.       return hash64(bytes, bytes.length); 

117.     } else if (o instanceof byte[]) { 

118.       final byte[] bytes = (byte[]) o; 

119.       return hash64(bytes, bytes.length); 

120.     } 

121.     return hash64(o.toString()); 

122.   } 

123.  

124.   // 64 bit implementation copied from here: https://github.com/tnm/murmurhash-java 

125.  

126.   /** 

127.   * Generates 64 bit hash from byte array with default seed value. 

128.   * 

129.   * @param data  byte array to hash 

130.   * @param length length of the array to hash 

131.   * @return 64 bit hash of the given string 

132.   */ 

133.   public static long hash64(final byte[] data, int length) { 

134.     return hash64(data, length, 0xe17a1465); 

135.   } 

136.  

137.  

138.   /** 

139.   * Generates 64 bit hash from byte array of the given length and seed. 

140.   * 

141.   * @param data  byte array to hash 

142.   * @param length length of the array to hash 

143.   * @param seed  initial seed value 

144.   * @return 64 bit hash of the given array 

145.   */ 

146.   public static long hash64(final byte[] data, int length, int seed) { 

147.     final long m = 0xc6a4a7935bd1e995L; 

148.     final int r = 47; 

149.  

150.     long h = (seed & 0xffffffffl) ^ (length * m); 

151.  

152.     int length8 = length / 8; 

153.  

154.     for (int i = 0; i < length8; i++) { 

155.       final int i8 = i * 8; 

156.       long k = ((long) data[i8 + 0] & 0xff) + (((long) data[i8 + 1] & 0xff) << 8) 

157.           + (((long) data[i8 + 2] & 0xff) << 16) + (((long) data[i8 + 3] & 0xff) << 24) 

158.           + (((long) data[i8 + 4] & 0xff) << 32) + (((long) data[i8 + 5] & 0xff) << 40) 

159.           + (((long) data[i8 + 6] & 0xff) << 48) + (((long) data[i8 + 7] & 0xff) << 56); 

160.  

161.       k *= m; 

162.       k ^= k >>> r; 

163.       k *= m; 

164.  

165.       h ^= k; 

166.       h *= m; 

167.     } 

168.  

169.     switch (length % 8) { 

170.       case 7: 

171.         h ^= (long) (data[(length & ~7) + 6] & 0xff) << 48; 

172.       case 6: 

173.         h ^= (long) (data[(length & ~7) + 5] & 0xff) << 40; 

174.       case 5: 

175.         h ^= (long) (data[(length & ~7) + 4] & 0xff) << 32; 

176.       case 4: 

177.         h ^= (long) (data[(length & ~7) + 3] & 0xff) << 24; 

178.       case 3: 

179.         h ^= (long) (data[(length & ~7) + 2] & 0xff) << 16; 

180.       case 2: 

181.         h ^= (long) (data[(length & ~7) + 1] & 0xff) << 8; 

182.       case 1: 

183.         h ^= (long) (data[length & ~7] & 0xff); 

184.         h *= m; 

185.     } 

186.     ; 

187.  

188.     h ^= h >>> r; 

189.     h *= m; 

190.     h ^= h >>> r; 

191.  

192.     return h; 

193.   } 

194. } 
```



**3）RegisterSet.java**

```
1. public class RegisterSet { 

2.  

3.   public final static int LOG2_BITS_PER_WORD = 6; //2的6次方是64 

4.   public final static int REGISTER_SIZE = 5;   //每个register占5位,代码里有一些细节涉及到这个5位，所以仅仅改这个参数是会报错的 

5.  

6.   public final int count; 

7.   public final int size; 

8.  

9.   private final int[] M; 

10.  

11.   //传入m 

12.   public RegisterSet(int count) { 

13.     this(count, null); 

14.   } 

15.  

16.   public RegisterSet(int count, int[] initialValues) { 

17.     this.count = count; 

18.  

19.     if (initialValues == null) { 

20.       /** 

21.       * 分配(m / 6)个int给M 

22.       * 

23.       * 因为一个register占五位，所以每个int（32位）有6个register 

24.       */ 

25.       this.M = new int[getSizeForCount(count)]; 

26.     } else { 

27.       this.M = initialValues; 

28.     } 

29.     //size代表RegisterSet所占字的大小 

30.     this.size = this.M.length; 

31.   } 

32.  

33.   public static int getBits(int count) { 

34.     return count / LOG2_BITS_PER_WORD; 

35.   } 

36.  

37.   public static int getSizeForCount(int count) { 

38.     int bits = getBits(count); 

39.     if (bits == 0) { 

40.       return 1; 

41.     } else if (bits % Integer.SIZE == 0) { 

42.       return bits; 

43.     } else { 

44.       return bits + 1; 

45.     } 

46.   } 

47.  

48.   public void set(int position, int value) { 

49.     int bucketPos = position / LOG2_BITS_PER_WORD; 

50.     int shift = REGISTER_SIZE * (position - (bucketPos * LOG2_BITS_PER_WORD)); 

51.     this.M[bucketPos] = (this.M[bucketPos] & ~(0x1f << shift)) | (value << shift); 

52.   } 

53.  

54.   public int get(int position) { 

55.     int bucketPos = position / LOG2_BITS_PER_WORD; 

56.     int shift = REGISTER_SIZE * (position - (bucketPos * LOG2_BITS_PER_WORD)); 

57.     return (this.M[bucketPos] & (0x1f << shift)) >>> shift; 

58.   } 

59.  

60.   public boolean updateIfGreater(int position, int value) { 

61.     int bucket = position / LOG2_BITS_PER_WORD;  //M下标 

62.     int shift = REGISTER_SIZE * (position - (bucket * LOG2_BITS_PER_WORD)); //M偏移 

63.     int mask = 0x1f << shift;   //register大小为5位 

64.  

65.     // 这里使用long是为了避免int的符号位的干扰 

66.     long curVal = this.M[bucket] & mask; 

67.     long newVal = value << shift; 

68.     if (curVal < newVal) { 

69.       //将M的相应位置为新的值 

70.       this.M[bucket] = (int) ((this.M[bucket] & ~mask) | newVal); 

71.       return true; 

72.     } else { 

73.       return false; 

74.     } 

75.   } 

76.  

77.   public void merge(RegisterSet that) { 

78.     for (int bucket = 0; bucket < M.length; bucket++) { 

79.       int word = 0; 

80.       for (int j = 0; j < LOG2_BITS_PER_WORD; j++) { 

81.         int mask = 0x1f << (REGISTER_SIZE * j); 

82.  

83.         int thisVal = (this.M[bucket] & mask); 

84.         int thatVal = (that.M[bucket] & mask); 

85.         word |= (thisVal < thatVal) ? thatVal : thisVal; 

86.       } 

87.       this.M[bucket] = word; 

88.     } 

89.   } 

90.  

91.   int[] readOnlyBits() { 

92.     return M; 

93.   } 

94.  

95.   public int[] bits() { 

96.     int[] copy = new int[size]; 

97.     System.arraycopy(M, 0, copy, 0, M.length); 

98.     return copy; 

99.   } 

100. } 
```



这里hash算法使用的是MurmurHash算法，可能很多人没听说过，其实在开源项目中使用的非常广泛，这个算法在只追求速度和hash的随机性，而不在意安全性和保密性的时候非常有效，我们不去深究这个算法的原理了，这个类的代码也不必仔细看，就把它看成一个hash函数就好了。



还有需要稍微注意一下这里的RegisterSet类，我们把存放一个桶的结果的地方叫做一个register，类中M数组就是存放这些register内容的地方，在这里我们设置一个register占5位，所以每个int(32位)总共可以存放6个register。



**核心内容在HyperLogLog类**



#### **4. Redis中的HyperLogLog**

Redis中和HyperLogLog相关的命令有三个：



1）PFADD hll ele：将ele添加进hll的基数计算中。

流程：

1. 先对ele求hash（使用的是一种叫做MurMurHash的算法）
2. 将hash的低14位(因为总共有2的14次方个桶)作为桶的编号，选桶，记桶中当前的值为count
3. 从hash的第15位开始数0，假设从第15位开始有n个连续的0（即前导0）
4. 如果n大于count，则把选中的桶的值置为n，否则不变



2）PFCOUNT hll：计算hll的基数。

就是使用上面给出的DV公式根据桶中的数值，计算基数



3）PFMERGE hll3 hll1 hll2：将hll1和hll2合并成hll3。

用的就是上面说的合并算法。



Redis的所有HyperLogLog结构都是固定的16384个桶（2的14次方），并且有两种存储格式：

1. **稀疏存储结构**：HyperLogLog算法在刚开始的时候，大多数桶其实都是0，稀疏格式通过存储连续的0的数目，而不是每个0存一遍，大大减小了HyperLogLog刚开始时需要占用的内存
2. **密集型存储结构**：用6个bit表示一个桶，需要占用12KB内存