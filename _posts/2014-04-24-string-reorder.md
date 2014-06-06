---
layout: post
title: "String reorder（微软2014在线测试第一题）"
description: ""
categories: 
- c
tags: [面试题]
---
{% include JB/setup %}  

**题目如下：**  
　　Time Limit: 10000ms  
　　Case Time Limit: 1000ms   
　　Memory Limit: 256MB   
**Description**  
　　For this question, your program is required to process an input string containing only ASCII characters between ‘0’ and ‘9’, or between ‘a’ and ‘z’ (including ‘0’, ‘9’, ‘a’, ‘z’).  
　　Your program should reorder and split all input string characters into multiple segments, and output all segments as one concatenated string. The following requirements should also be met,  
　　1. Characters in each segment should be in strictly increasing order. For ordering, ‘9’ is larger than ‘0’, ‘a’ is larger than ‘9’, and ‘z’ is larger than ‘a’ (basically following ASCII character order).  
　　2. Characters in the second segment must be the same as or a subset of the first segment; and every following segment must be the same as or a subset of its previous segment.   
　　Your program should output string “&lt;invalid input string&gt;” when the input contains any invalid characters (i.e., outside the '0'-'9' and 'a'-'z' range).  
**Input**  
　　Input consists of multiple cases, one case per line. Each case is one string consisting of ASCII characters.  
**Output**  
　　For each case, print exactly one line with the reordered string based on the criteria above.  
**Sample In**  
　　aabbccdd  
　　007799aabbccddeeff113355zz  
　　1234.89898  
　　abcdefabcdefabcdefaaaaaaaaaaaaaabbbbbbbddddddee  
**Sample Out**  
　　abcdabcd  
　　013579abcdefz013579abcdefz  
　　&lt;invalid input string&gt;  
　　abcdefabcdefabcdefabdeabdeabdabdabdabdabaaaaaaa    

---
　　这一题是四道上机题中的第一道，题目的意思是输入一串字符串，然后按照其中0-9，a-z的顺序输出，输出一遍之后将剩余的字符串再按照这个顺序再次输出，直到字符串中的字符都输出为止。比如输入的字符串为aabbccdd，按照指定的排序第一次输出为abcd，此时字符串中还剩四个字符，再按照这个顺序输出，得到abcd，这个时候字符串中全部字符已经输出，所以最后的输出序列就是abcdabcd。  
　　一般碰到这种涉及到字符出现次数相关的题目时，我都喜欢先考虑**hash表**，因为这个方法屡试不爽，这一题也不例外。这里可以构建一个0-9和a-z的为键的hash表，然后遍历字符串统计每个字符出现的次数生成hash表，接着要做的事情就是按照hash的键的顺序按序输出字符，每次输出完就将hash值减1，直到所有的字符输出完毕也就实现了该功能。除去构造hash表的时间，该方法的时间复杂度是O(N*M)，其中N为字符的个数，M为hash表的大小。空间复杂度为O(1)。
##源代码
{% highlight c++ %}
/****************************************************************************
 * @file     answer.c
 * @brief    题目参考description
 * @version  V1.00
 * @date     2014年4月12日
 * @note     
****************************************************************************/
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define NUM 36

int HashChar[NUM] = {0};   // contains0-9 and a-z character
int Size = 0;
/** 
 * @brief     计算元素在HASH表中的位置。
 * @param[in] c  需要计算位置的元素
 * @retval    元素在HASH表中的位置，如果不在范围之内，返回-1
 * @see       无
 * @note      无
 */
int Hash(char c) {
	if(c >='0' && c <= '9')
	{
		return c-'0';
	}
	else if(c >='a' && c <= 'z')
	{
		return c-'a' + 10;
	}
	else
	{
		return -1;
	}
}
/** 
 * @brief     初始化HASH表。
 * @param[in] c  需要计算位置的元素
 * @retval    元素在HASH表中的位置，如果不在范围之内，返回-1
 * @see       无
 * @note      无
 */
int InitialHash(char *B) {
	int i;
	int pos;
	for(i = 0;B[i];i++) {
		pos = Hash(B[i]);
		if(pos == -1) {
			
			return -1;
		}
		else 
			HashChar[pos] ++;
		Size++;
	}
	return 0;
}
 /**	字符串出现排序   
 *     
 *		将字符串按照指定要求重新排序，并且打印。。    
 */ 
void ReOrder(){
	int i;
	for(i = 0; i < NUM;i++)
	{
		if(HashChar[i])
		{
			if(i <= 9)
			{
				printf("%c",i+'0');
			}
			else
			{
				printf("%c",i - 10 + 'a');
			}
			HashChar[i]--;
			Size--;
		}
		
	}
}

int main(void) {
    char unorder[500];
    while(scanf("%s", unorder) != EOF) {
		if(-1 == InitialHash(unorder))
		{
			printf("<invalid input string>");
			Size = 0;
		}
		while(Size)
		{
			ReOrder();
		}
    	printf("\n");
		Size = 0;
		memset(HashChar,0,36);
    }
    return 0;
}

{% endhighlight %}



