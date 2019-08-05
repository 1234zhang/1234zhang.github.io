---
layout: '[layout]'
title: Median of Two Sorted Arrays（中位数问题）
date: 2019-07-27 16:52:56
categories: 
	- Java
	- 算法
tags: 
	- leetcode
---

# *Median of Two Sorted Arrays*

```text
中位数用于：将两个集合分为长度相同的子集合，其中一个子集合中的元素总是小于另一个集合的元素。
```

### 首先将A集合划分

A集合根据随机的i值分为两个部分，其中，左边最大的总是小于右边最小的。因为A有m个元素，所以可以有m种分法。   

- len(left_A) = i, len(right_A)  = m - i；

### 再将B集合划分

同样的方式，根据一个随机的J值，将B集合分成两个部分。满组A集合划分的两个条件。因为B有n个元素，所以有n种分法

### 处理A，B集合划分的两个部分

将A集合与B集合的左边部分置于同一个集合，右边部分置于同一个集合。将他们命名为left_part, right_part

我们可以保证的是：

```
len(left_part) == len(right_part)
max(left_part) < min(right_part)
```

所以最后我们将A，B两个集合中的所有元素划分为两个长度相同的集合。一个集合的元素总是小于另一个集合。

median = $(max(left_p) + min(right_p))/2$

为了保证上面的两个条件，我们只需要确保

- $i+j = m - i + n - j (or: m - i + n - j + 1)$

​       if n >= m, we just need to set : i = 0 ~ m, j = (m+n+1)/2 - i​

- B[j - 1] <=  A[i] and A[i - 1] <= B[j] 

```
1. 为了方便考虑，我们假设A[i - 1], B[j - 1], A[i],B[j] 总是有效的，甚至是 i= 0， i=m， j=0,j=n
2. 为什么要保证n>= m？ 这是因为我们要保证j是非负的，从0 <= i <= m 以及 j=(m+n+1)/2 -i可以看出，如果j是负数，就会导致结果错误。
```



## 算法具体过程

- 从[0,m] 中找到一个对象i， 满足：

  - ​	B[j - 1]<= A[i] and A[i -1] <= B[j] (j = (m+n+1)/2 - i)

- 根据以下步骤，进行二分搜索

  1、设置iMIn = 0, iMax = m, 接着从[iMin,IMax]中间搜索

  2、 令 i = （iMin + iMax)/2,  j = (m+n+1)/2  - i 

  - 这一步的意思是：找到A集合的中间值，再找到B集合的中间值，因为两个集合都是已经排序了的

   3、我们已知的条件len(left_part) = len(right_part). 在寻找 中位数的时候，我们将遇到下面三种情况：

  - B[j - 1] <= A[i] and A[i - 1] <= B[j] 如果出现这种情况，说明我们已经找到了 此时的i 值，就是我们要找的中位数的下标
  - B[j - 1] > A[i] 说明A[i] 值小了，我们只需要调整i 使得B[j - 1] <= A[i] 满足
    - 通过增加 i 的值，如果增加了 i 的值，那么 j 的值就要减小，所以B[j - 1] 的值就要减小，A[i] 的值就要增加，那么就有可能满足B[j - 1 ] <= A[i] 这个条件
    - 如果我们减小了 i 的值，那么B[j - 1] 的值就会变大，A[i] 的值就会变小，那么永远都无法满足条件。
    - 所以我们只能增加 i 的值，再跳到第二步，更新 j 的值
  - A[i - 1] > B[j] 说明A[ i - 1] 的值太大，我们必须减小 i 的值，以满足A[i - 1] < B[ j ]

- 当我们找到 i 值的时候，那么中位数的求解过程：

  - median = max(A[i - 1] , B[j - 1]) 当总的元素个数是奇数的时候
  - median = (max(A[i - 1], B[ j - 1] ) + min(A[ i ] , B[ j ]))/ 2 当总的元素个数是偶数的时候

## 现在我们来讨论一下边界问题

```
当 i= 0，i = m, j = 0, j = n的时候，A[i - 1], B[j - 1], A[i], B[j]的值可能不存在的情况。
```

我们只需要确保max(left_part) <= min(right_part)。如果i 和j 不是边界值，意味着 A[i - 1] , B[j - 1],和A[i]，B[j] 都存在， 那么我们就要检查B[j - 1] <= A[i] 以及 A[i - 1] <= B[j]. 但是如果A[i - 1], B[ j - 1], A[i] , B[j] 他们之中某一个不存在，那么我就不需要再检查其他的条件。例如 如果 i = 0 那么 A[i - 1]就不存在，我们就不需要再检查A[i - 1] <= B[j]。综上所述，我们就可以得到:

```
在[0, m]之中找到合适的i值 只需要满足：
	(j = 0 || i = m || B[j - 1] <= A[i]) && (i = 0 || j = n || A[i - 1] <= B[j]) j = (m + n + 1)/2 - i
```

## 代码

```java
/*
 * @lc app=leetcode id=4 lang=java
 *
 * [4] Median of Two Sorted Arrays
 */
class Solution {
    public double findMedianSortedArrays(int[] nums1, int[] nums2) {
        int m = nums1.length;
        int n = nums2.length;
        if(m > n){
            int temp = n; n = m; m = temp;
            int[] temps = nums1; nums1 = nums2; nums2 = temps;
        }
        int iMin = 0;
        int iMax = m;
        int halfLen = (m + n + 1)/2;
        while(iMin <= iMax){
            int i = (iMin + iMax)/2;
            int j = halfLen - i;
            if(i < iMax && nums2[j - 1] > nums1[i]){
                iMin = i + 1;
            }else if(i > iMin && nums1[i - 1] > nums2[j]){
                iMax = i - 1;
            }else{
                int maxLeft = 0;
                if(i == 0){
                    maxLeft = nums2[j - 1];
                }else if ( j == 0){
                    maxLeft = nums1[i - 1];
                }else{
                    maxLeft = Math.max(nums1[i - 1], nums2[j - 1]);
                }
                if((m + n)% 2 != 0){return maxLeft;}

                int minRight = 0;
                if(i == m){
                    minRight = nums2[j];
                }else if(j == n){
                    minRight = nums1[i];
                }else{
                    minRight = Math.min(nums1[i], nums2[j]);
                }
                return (minRight + maxLeft)/2;
            }
        }
        return 0;
    }
}


```

