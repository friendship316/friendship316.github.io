---
title: 求一个数组中最大的子数组之和以及起终点
date: 2017-05-07 17:30:32
---
* 在LeetCode上做的一道题目，想了很久没有想出比较简便的方法，借鉴了网上的方法之后做了然后做个总结。

#### 题目描述
 Find the contiguous subarray within an array (containing at least one number) which has the largest sum.
For example, given the array [-2,1,-3,4,-1,2,1,-5,4],
the contiguous subarray [4,-1,2,1] has the largest sum = 6. 
简单而言就是找出一个正负数都有的数组中最大的子数组之和。

#### 分析解答
1. 看到这个题，首先想到的一个最简单但是时间复杂度最高的方法就是，把这个数组的所有子数组的和都求出来，一个一个比较，取最大值。
假设数组的长度为n，那么以它的每个元素的下标i（i从0开始）为子数组起点，都有n-i个子数组，所以所有的子数组个数是
n+n-1+n-2+n-3+…+4+3+2+1
=(n+1)+(n-1+2)+(n-2+3)+...
=(n/2)(n+1) 
所以时间复杂度为O（n²）；
2. 动态规划（复杂度为O(n)）。对于数组中第i个元素来说，设curSum为nums[i]之前的累加和最大值，如果这个最大值<0，那么加上nums[i]之后必然会使累加和变小。即curSum+nums[i]<nums[i]，所以此时应该丢掉之前的累加和，从nums[i]重新开始累加。反之，curSum+nums[i]>nums[i]，必定会使和变大（即便nums[i]为负数）。这样，每循环一个元素，都能保证去掉拖后腿的累加和。只要不拖后腿，能多加一个正数，肯定是有利于累加和的增大。从这些累加中比较出值大的累加和即可。
```
public static int maxSubArray(int[] nums){
		int curSum = nums[0];//curSum为nums[i]之前的累加和最大值
		int maxSum = nums[0];//最大累加和
		int curBegin = 0;//当前累加和起点
		int maxBegin = 0;//最大累加和起点
		int maxEnd = 0;//最大累加和终点
		for(int i=1;i<nums.length;i++){
			if(curSum>0){
				//如果之前的累加和>0，加上nums[i]元素
				curSum += nums[i];
			}else{
				//否则，以nums[i]为起点重新开始
				curSum = nums[i];
				curBegin = i;
			}
			//如果当前累加和>记录的累加和，赋值。并将当前累加和的起终点记录下来
			if(curSum > maxSum){
				maxSum = curSum;
				maxBegin = curBegin;
				maxEnd = i;
			}
		}
		System.out.println("起点:"+maxBegin+",终点:"+maxEnd);
		return maxSum;
	}
```