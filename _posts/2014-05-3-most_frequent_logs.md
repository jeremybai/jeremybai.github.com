---
layout: post
title: "Most_Frequent_Logs"
description: ""
categories: 
- c
tags: [面试题]
---
{% include JB/setup %}  

**题目如下：**  
　　Time Limit: 10000ms  
　　Case Time Limit: 3000ms  
　　Memory Limit: 256MB     
**Description**  
　　In a running system, there're many logs produced within a short period of time, we'd like to know the count of the most frequent logs.  
　　Logs are produced by a few non-empty format strings, the number of logs is N(1=N=20000), the maximum length of each log is 256.  
　　Here we consider a log same with another when their edit distance (see note) is = 5.  
　　Also we have   
　　a) logs are all the same with each other produced by a certain format string   
　　b) format strings have edit distance  5 of each other.   
　　Your program will be dealing with lots of logs, so please try to keep the time cost close to O(nl), where n is the number of logs, and l is the average log length.  
　　Note edit distance is the minimum number of operations (insert/delete/replace a character) required to transform one string into the other, please refer to httpen.wikipedia.orgwikiEdit_distance for more details.    
**Input**  
　　Multiple lines of non-empty strings.    
**Output**  
　　The count of the most frequent logs.    
**Sample In**  
　　Logging started for id:1  
　　Module ABC has completed its job  
　　Module XYZ has completed its job  
　　Logging started for id:10  
　　Module ? has completed its job  

  
**Sample Out**  
　　3

---
　　题目的起源是在一个运行着的系统中很短时间内会产生许多条日志，我们需要知道最频繁出现的那条日志出现的次数。两条日志相似的条件是他们之间的编辑距离小于等于5。编辑距离的定义在后面也给了出来：编辑距离就是将一个字符串变为另外一个字符串需要的最少的操作数，操作数包括增加/删除/替换一个字符。举个例子："abcd"和"abcdef"编辑距离为2，因为把"abcd"变成"abcdef"，需要在后面做2次增加操作。  
　　首先我们需要考虑的就是计算编辑距离了，编辑距离肯定不超过其中较长的字符串的长度，也就意味着编辑距离是有限的，我们可以这样考虑：假设有两个字符串，字符串A的长度为n，字符串B的长度为m，A和B的编辑距离是如何来的？我们可以分成三种情况来讨论：  
　　（1）当A[1]...A[n-1]与B[1]...B[m-1]的编辑距离已知时，如果A[n]和B[m]相同，那么A[1...n]和B[1...m]的编辑距离等于A[1...n]和B[1...m]的编辑距离；如果A[n]和B[m]不同，那么A[1...n]和B[1...m]的编辑等于A[1...n]和B[1...m]的编辑距离加1(把A[n]变成B[m]或者把B[m]变成A[n]的距离为1)；  
　　（2）当A[1]...A[n]与B[1]...B[m-1]的编辑距离已知时，A[1...n]和B[1...m]的编辑距离等于A[1]...A[n]与B[1]...B[m-1]的编辑距离加1（A中在增加1个字符或者B中减少1个字符）；    
　　（3）当A[1]...A[n-1]与B[1]...B[m]的编辑距离已知时，A[1...n]和B[1...m]的编辑距离等于A[1]...A[n-1]与B[1]...B[m]的编辑距离加1（B中在增加1个字符或者A中减少1个字符）.    
　　A和B的编辑距离是从这3中情况中取最小的那个，所以可以看出这就是一个典型的动态规划问题（最优子结构，重叠子问题），我们可以用一个二维数组ed[i][j]表示A中前i个字符到B中前j个字符的最短编辑距离，首先我们初始化ed[i][0]=i，ed[0][j]=j,因为ed[i][0]代表A中有i个字符，B中有0个字符，那么把A变到B需要做i次删除操作，同理可以得到ed[0][j]=j。由上面的分析可以得到状态迁移公式：  

	ed[i][j] = min (ed[i - 1][j - 1] + (A[i] == B[j] ?  0 : 1) ,  ed[i - 1][j] + 1 ,  ed[i][j - 1] + 1)

　　编辑距离的计算解决之后，就要考虑这个问题的实际实际场景了，首先我想到的最笨的方法就是用两次循环，对于每一条日志求出它和其他所有的日志的编辑距离，用一个值来记录相似日志的条数。可是这种方法效率实在让人捉急。那么能不能将模板记录下来，每次输入日志的时候，让它和模板中的模板进行比较，当和模板距离小于5时，就说明它与模板类似，把模板对应的日志数加1，否则就为它新建一个模板，这样效率会提高很多。思路有了接下来就是如何实现了，模板一开始我采用的是二维数组template[MAX_NUM][256]，MAX_NUM代表日志的条数，256代表日志的最大吃那个度，当需要增加模板时，就将对应的字符串拷贝到这个二维数组中，比如说第一条日志就将它拷贝到template[0]中。后来review的时候发现这种拷贝操作完全不必要，只需要使用指针数组，将模板的指针存放到该指针数组中去，因为比较的时候只需要提供和长度即可，这样又减少了空间消耗，源码如下。	

##源代码
{% highlight c++ %}
/****************************************************************************
 * @file     answer.c
 * @brief    见description
 * @version  V2.00
 * @date     2014年5月3日
 * @note     V1 采用计算编辑距离的方法,模板采用二维数组存放。
			 V2 模板采用指针数组存放。
****************************************************************************/
#include <stdio.h>
#include <malloc.h>
#include <string.h>

#define MAX_NUM 20000

/**		求最小值   
 *     
 *		求三个数中最小的一个。    
 */
int min(int a, int b, int c)
{
    return (a = a < b ? a : b) < c ? a : c;
}


/** 
 * @brief     二维动态规划计算字符串的编辑距离。
 * @param[in] str1      字符串1的起始地址
 * @param[in] str1len   字符串1的长度
 * @param[in] str2      字符串2的起始地址
 * @param[in] str2len   字符串2的长度
 * @retval    成功：两个字符串之间的编辑距离，失败：-1
 * @see       None
 * @note      		  
 */
int CalEditDist(char* str1, int str1len, char* str2, int str2len)
{
	if(NULL == str1 || NULL == str2 || 0 == str1len || 0 == str2len)
	{
		printf("参数有误\n");
		return -1;
	}
	int i,j,re;
	int *array = (int*)malloc(((str1len+1)*(str2len+1))*sizeof(int));
	
	for(i = 0; i <= str1len; i++)
		*(array + i*(str2len + 1)) = i;           //array[i][0]
	for(j = 0; j <= str2len; j++)
	    *(array + j) = j;                         //array[0][j]
	for(i = 1; i <= str1len; i++)
	{
		for(j = 1; j <= str2len; j++)
		{
			*(array+i*(str2len+1)+j) = min(*(array+i*(str2len+1)+j-1)+1,*(array+(i-1)*(str2len+1)+j)+1,*(array+(i-1)*(str2len+1)+j-1) + (str1[i-1] == str2[j-1] ? 0 : 1));
		}
	}
	re =  *(array+(str1len)*(str2len+1)+(str2len));
	free(array);
	array=NULL;
	return re;
}
 
int main()
{
	char *temp = (char*)malloc(256*sizeof(char)),*ptr;
	char *template[MAX_NUM];
	int count[MAX_NUM],i;
	int len[MAX_NUM] = {0},length,flag,max;
	memset(count,-1,sizeof(int)*MAX_NUM);
	while(gets(temp) != NULL)
	{
		if(strlen(temp) == 0)
			break;
		//初始化第一个模板，否则无法进行下一步
		if(count[0] == -1)
		{
			template[0] = temp;
			count[0] = 1;
			len[0] = strlen(temp);
			continue;
		}
		length = strlen(temp);
		flag = 0;
		for(i = 0; i < MAX_NUM && count[i] != -1; i++)
		{
			//如果编辑距离小于5，说明是同一个模板
			if(CalEditDist(temp,length,template[i],len[i]) <= 5)
			{
				count[i]++;
				flag = 1;
				break;
			}
		}
		//如果在模板列表中没有找到对应的模板，则新建一个模板
		if(!flag)
		{
			template[i] = temp;
			count[i] = 1;
			len[i] = length;
		}
	}
	i = 0;
	max = 0;
	while(count[i] != -1)
	{
		if(count[i] > max)
			max = count[i];
		i++;
	}
	printf("%d ",max);
	free(temp);
	temp = NULL;
}
{% endhighlight %}