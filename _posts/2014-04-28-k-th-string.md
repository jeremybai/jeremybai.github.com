---
layout: post
title: "K-th-string"
description: ""
categories: 
- c
tags: [面试题]
---

**题目如下：**  
　　Time Limit: 10000ms  
　　Case Time Limit: 1000ms  
　　Memory Limit: 256MB  
**Description**  
　　Consider a string set that each of them consists of {0, 1} only. All strings in the set have the same number of 0s and 1s. Write a program to find and output the K-th string according to the dictionary order. If s​uch a string doesn’t exist, or the input is not valid, please output “Impossible”. For example, if we have two ‘0’s and two ‘1’s, we will have a set with 6 different strings, {0011, 0101, 0110, 1001, 1010, 1100}, and the 4th string is 1001.  
**Input**  
　　The first line of the input file contains a single integer t (1 ≤ t ≤ 10000), the number of test cases, followed by the input data for each test case.  
　　Each test case is 3 integers separated by blank space: N, M(2 <= N + M <= 33 and N , M >= 0), K(1 <= K <= 1000000000). N stands for the number of ‘0’s, M stands for the number of ‘1’s, and K stands for the K-th of string in the set that needs to be printed as output.
**Output**  
　　For each case, print exactly one line. If the string exists, please print it, otherwise print “Impossible”.   
**Sample In**  
　　3  
　　2 2 2  
　　2 2 7  
　　4 7 47  
**Sample Out**  
　　0101  
　　Impossible  
　　01010111011  

---
　　题目的意思就是输入N个0，M个1，然后将它们进行组合排序，然后输出第k个数。当我看到题目最先反应到的就是找出相邻两个数之间的关系，然后从最小的数开始不停的生成下一个数，直到生成k次为止，可惜没找到什么规律，没做的出来。后来看到网上有人使用字典序生成全排列这个算法解了这道题，于是上网搜了搜什么是字典序，以及如何用字典序生成全排列。
##字典序
　　对于集合中所有元素所形成的序列，字典序是比较序列大小的一种排序方式，字典序的前提是集合中的元素值有特定的大小顺序。以A{1,2,3,4,5}为例，元素大小顺序为1<2<3<4<5，其所形成的排列12345<12354，比较的方法是从前到后依次比较两个序列的对应元素，如果当前位置对应元素相同，则继续比较下一个位置，直到第一个元素不同的位置为止，元素值大的元素在字典序中就大于元素值小的元素。上面的两个数前三位都是相同的，但是第四位12354大于12345（5>4），所以12345<12354，同样，字典序也被运用于字符串的处理中。
##字典序生成全排列算法
　　使用字典序输出全排列的思路是，首先输出字典序最小的排列，然后输出字典序次小的排列，……，最后输出字典序最大的排列，生成已知排列的下一个字典序排列的算法如下，证明这里就不介绍了。  
　　1 对于排列a[1...n]，找到所有满足a[k]<a[k+1] (0<k<n-1)的k的最大值，如果这样的k不存在，则说明当前排列已经是a的所有排列中字典序最大者，所有排列输出完毕。  
　　2 在a[k+1...n]中，寻找满足这样条件的元素l，使得在所有a[l]>a[k]的元素中，a[l]取得最小值。也就是说a[l]>a[k]，但是小于所有其他大于a[k]的元素（存在相同元素时，l尽量向后找）。  
　　3 交换a[l]与a[k]。  
　　4 对于a[k+1...n]，反转该区间内元素的顺序。也就是说a[k+1]与a[n]交换，a[k+2]与a[n-1]交换，……，这样就得到了a[1...n]在字典序中的下一个排列。
##算法与本题关联
　　这一题其实就是N个0和M个1的第K个字典序全排列。想清楚这一点就可以开始写代码了。要注意的就是判断k是否超过C(N+M,N)，也就是总共的组合数个数。  
　　**对于元素集合不方便比较的情况**，可以将它们在数组中的索引作为元素，按照字典序生成索引的全排列，然后按照索引输出对应集合元素的排列，**对于包含重复元素的输入集合**，需要先将相同的元素放在一起，以集合{a,b,c,b,c}为例，如果直接对其索引12345进行全排列，将不会得到想要的结果，这里将重复的元素放到相邻的位置，得到排列{a,b,b,c,c}或者{a,c,c,b,b}（不一定需要顺序或者逆序，只需要保证相同的元素相邻即可），然后将不同的元素对应不同的索引值，生成索引排列12233，再执行全排列算法，即可得到最终结果。这种情况适用于对全排列的顺序没有要求而只是需要输出全排列，当需要按序输出全排列时（比如本文所讲的这一道题），只能按部就班的按照元素的大小比较进行生成全排列。  
##源代码
{% highlight c++ %}
/****************************************************************************
 * @file     answer.c
 * @brief    题目参考description
 * @version  V1.00
 * @date     2014.4.28
 * @note     思路1：全排列的字典序生成算法
****************************************************************************/
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stdint.h>

/** 
 * @brief     计算组合数C(n,r)。
 * @param[in] n:总数  r:取出的数
 * @retval    0：出错  其他：组合数
 * @see       无
 * @note      无
 */
long long com(int n,int r)
{
	if(n <= 0 || r < 0)
	{
		perror("function com para error!\n");
		return 0;
	}
	if(r == 0)
		return 1;
	if(n - r < r) 
		r = n-r;  //减少计算量
	long long s = 1;
	int i,j;
	for(i=0,j=1;i < r;++i)
	{
		s*=(n-i);
		for(;j <= r && s%j == 0; ++j) // 这个循环就是除以r!,防止溢出
			s/=j;  
	}	
	return s;
}
/** 
 * @brief     反转字符串。
 * @param[in] begin:字符串起始地址  end:字符串结束地址 len:字符串长度
 * @retval    错误码，0:反转成功 其他:反转失败
 * @see       无
 * @note      无
 */
int reverse(char *begin, char *end,int len)
{
	if(NULL == begin || NULL == end)
	{
		perror("function reverse para error!\n");
		return 1;
	}
	if(len == 1)
		return 0;
	int length = len/2;
	while(length--)
	{
		*begin ^= *end;
		*end ^= *begin;
		*begin++ ^= *end--;
	}
	return 0;
}


int main(void) 
{
	long long num = 0, ordinal = 0;
	int num_zero, num_one,i,j,k,left,right,sum;
	while(scanf("%d", &num) != EOF) {
		while(num--)
		{
			scanf("%d%d%d", &num_zero, &num_one, &ordinal);
			sum = num_zero + num_one;			
			if(num_zero < 0 || num_one < 0 || sum < 2 || sum > 33 || ordinal < 0 || ordinal > 1000000000)
			{
				continue;
			}
			char *p = (char *)malloc(sizeof(char)*(num_zero+num_one)+1);
			//数组从0开始
			i = 0;
			//构造最小的组合数
			j = num_zero;
			while(j--)
			{
				p[i++] = '0';
			}
			j = num_one;
			while(j--)
			{
				p[i++] = '1';
			}
			p[i] = '\0';
			// printf("%s\n",p);
			//判断参数是否越界
			if( (0 == ordinal) || (ordinal > com(sum,num_zero)) ) 
			{
				printf("Impossible\n");
			}
			else
			{
				for(i = ordinal ;i > 1 ;i--)
				{
					for(j = sum-1; j > 0; j--)
					{
						if( (p[j-1] == '0') && (p[j] == '1') )
						{
							left = j-1;
							for(k = j; k < sum; k++)
							{	
								right = k;
								if(p[k] != '1')
								{
									right = k -1;
									break;
								}
							}
							//交换找到的0和1
							p[left] ^= p[right];
							p[right] ^= p[left];
							p[left] ^= p[right];
							//反转left之后字符串
							reverse(p+left+1,p+sum-1,sum-1-left);
							break;					
						}
						
					}
				}
				printf("%s\n",p);
				free(p);
				p = NULL;
			}
		}

	}
	return 0;
}
{% endhighlight %}



