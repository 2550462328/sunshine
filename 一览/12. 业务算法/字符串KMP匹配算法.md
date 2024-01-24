业务场景：判断字符串S1中是否包含格式串S2



对于传统解决方案我们采用的是暴力回溯的方法，对于S1的字符串从0~n在格式串S2中进行判断，如果出现不匹配的情况，i+1。



传统暴力回溯的示例代码如下：

```
1. /** 

2. * 传统暴力回溯求解 

3. * @param pattern 字符串格式 

4. * @param target 待匹配字符串 

5. * @return int 当匹配的时候返回匹配点 否则返回 -1 

6. */ 

7. public static int violent_compute(String pattern, String target){ 

8.  

9.   int pLen = pattern.length(); 

10.   int tLen = target.length(); 

11.  

12.   if(tLen < pLen){ 

13.     return -1; 

14.   } 

15.  

16.   // i是target上的游标 j是pattern上的游标 k是匹配进度条 

17.   int i = 0, j = 0, k = 0; 

18.  

19.   while(k <= (tLen - pLen)){ 

20.     if(target.charAt(i) == pattern.charAt(j)){ 

21.       if(++j == pLen){ 

22.         return k; 

23.       } 

24.       i++; 

25.     }else { 

26.       i = ++k; 

27.       j = 0; 

28.     } 

29.   } 

30.   return -1; 

31. } 
```



使用这种方案的判断过程如下



![img](http://pcc.huitogo.club/8f467910e459716898ed3d61c3e729bd)



对于目标串（上方）中每一个值都需要在格式串（下方）中进行逐一比较，如果匹配成功，两个字符串上的比较下标 + 1，反之，目标串的比较下标 + 1（也就是从下一个字符开始比较），格式串的比较下标清零。



从上述过程中，我们会发现，在比较过程中任一字符匹配失败都会导致从目标串的下一个字符开始重新匹配，我们觉得这样效率是非常低下的，因为当我们匹配到字符n的时候，我们会认为字符0~n-1都是已经匹配好的，那这就是规律



也就是 我们不希望每一次失配后都从目标串的下一个字符开始



所以KMP算法油然而生

![img](http://pcc.huitogo.club/28830b55f39d5d78cdf637f96875c331)



一个基本事实是，当空格与D不匹配时，你其实知道前面六个字符是"ABCDAB"。KMP算法的想法是，设法利用这个已知信息，不要把"搜索位置"移回已经比较过的位置，继续把它向后移，这样就提高了效率。



怎么做到这一点呢？可以针对搜索词，算出一张《部分匹配表》

![img](http://pcc.huitogo.club/17a07e18b724379a54636412a1009299)



已知空格与D不匹配时，前面六个字符"ABCDAB"是匹配的。查表可知，最后一个匹配字符B对应的"部分匹配值"为2，因此按照下面的公式算出向后移动的位数：

![img](http://pcc.huitogo.club/e29fa08ed8e2cbf130ebcf057fcc76ed)



因为 6 - 2 等于4，所以将搜索词向后移动4位。

![img](http://pcc.huitogo.club/8444f5fafa13fe5da1eb767196ae4ac2)



那现在这个匹配表是怎么算出来的呢？



要求next[k+1] 其中k+1=17

![img](http://pcc.huitogo.club/b8e8a06dc3e39e0af68dbc29384e12af)



已知next[16]=8，则元素有以下关系：

![img](http://pcc.huitogo.club/194a7a8d1ff6e24834c7e15fa95593dc)



如果P8=P16，则明显next[17]=8+1=9



如果不相等，又若next[8]=4，则有以下关系

![img](http://pcc.huitogo.club/8089aa1875f06968bd0bf1f6660b7a90)



又加上2的条件知

![img](http://pcc.huitogo.club/74203dad1bedc4d48bae0cd728f366f5)



主要是为了证明

![img](http://pcc.huitogo.club/b4fc4b5443fd574bdd897214e41250bf)



现在在判断，如果P16=P4则next[17]=4+1=5，否则，在继续递推



若next[4]=2，则有以下关系

![img](http://pcc.huitogo.club/42919eadd419a9a66a29ba7dba28dab3)



若P16=P2，则next[17]=2+1=3;否则继续取next[2]=1、next[1]=0；遇到0时还没出结果，则递推结束，此时next[17]=1。



针对以上逻辑，得出最终得到匹配表的算法

```
1. /** 

2. * 获取next数组 

3. * @param pattern 

4. * @return int[] 

5. */ 

6. private static int[] get_nextIndex(String pattern){ 

7.   char[] pChars = pattern.toCharArray(); 

8.  

9.   int i = 1, j = 0,pLen = pChars.length; 

10.  

11.   int[] nextIndexs = new int[pLen]; 

12.  

13.   nextIndexs[1] = 0; 

14.  

15.   while(i + 1 < pLen){ 

16.     if(j ==0 || pChars[i] == pChars[j]) { 

17.       if(pChars[i+1] == pChars[j+1]){ 

18.         nextIndexs[++i] = nextIndexs[++j]; 

19.       } 

20.       nextIndexs[++i] = ++j; 

21.     }else{ 

22.       j = nextIndexs[j]; 

23.     } 

24.   } 

25.   return nextIndexs; 

26. } 
```



这种实现逻辑是动规的一种了，求解next[n]需要依赖next[0]~next[n-1]的值，并且给出了边界值next[1] = 0；



有了匹配表后再实现KMP算法就比较简单了，在失配的时候我们不是让目标串的游标单纯的+1了，这个时候要计算失配时的上一个字符对应的匹配表的值。



示例代码

```
1. /** 

2. * 使用KMP算法求解 

3. * @param pattern 字符串格式 

4. * @param target 待匹配字符串 

5. * @return int 当匹配的时候返回匹配点 否则返回 -1 

6. */ 

7. public static int kmp_compute(String pattern, String target){ 

8.   int pLen = pattern.length(); 

9.   int tLen = target.length(); 

10.  

11.   if(tLen < pLen){ 

12.     return -1; 

13.   } 

14.  

15.   // i是target上的游标 j是pattern上的游标 k是匹配进度条 

16.   int i = 0, j = 0, k = 0; 

17.  

18.   int[] nextIndexs = get_nextIndex(pattern); 

19.  

20.   while(k <= (tLen - pLen)){ 

21.     if(target.charAt(i) == pattern.charAt(j)){ 

22.       if(++j == pLen){ 

23.         return k; 

24.       } 

25.       i++; 

26.     }else { 

27.       if(j != 0){ 

28.         k = k + (j - nextIndexs[j]); 

29.       }else{ 

30.         k = k +1; 

31.       } 

32.       i = k; 

33.       j = 0; 

34.     } 

35.   } 

36.   return -1; 

37. } 
```



**总结**

- 对于使用KMP算法的核心是计算出next数组，也就是匹配表；
- 计算next数组的核心是使用递归，合理利用next[0]~next[n-1]的值，找到其中的关系。