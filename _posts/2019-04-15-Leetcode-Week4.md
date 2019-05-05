---
layout: post
title: Leetcode刷题--Week 4
subtitle: Improving coding skills in 4th week
date: 2019-04-15T00:00:00.000Z
author: Xu Zhenxue
header-style: text
catalog: true
tags:
  - Leetcode
---
> 滚动阅读全文

#### 前言
本周刷题两道：二叉树的前序遍历，哈希应用求两数之和

#### 二叉树前序遍历
###### 题目  
Given a binary tree, return the preorder traversal of its nodes' values.

For example:
Given binary tree{1,#,2,3},
return[1,2,3].

Note: Recursive solution is trivial, could you do it iteratively?

###### 解题思路
###### 递归方法
递归方法比较简单，把左右子树当成一个完整二叉树递归遍历即可。  

代码：
```java
public ArrayList<Integer> preorderTraversal(TreeNode root) {
		ArrayList<Integer> list = new ArrayList<>();
		traversal(list,root);
		return list;
	}

	public void traversal(ArrayList<Integer> list, TreeNode root){
		if (root == null){
			return;
		}
		list.add(root.val);
		traversal(list,root.left);
		traversal(list,root.right);

	}
```
###### 非递归方法
1. 根节点不为空，则将节点入栈
2. 将栈内节点依次出栈，直至栈空
3. 将栈顶节点出栈后加入List中
4. 若出栈节点有右节点，将右节点入栈
5. 若出栈节点有左节点，将左节点入栈

代码：
```java
public ArrayList<Integer> preOrderTraversal2(TreeNode root){
		ArrayList<Integer> list = new ArrayList<>();
		Stack<TreeNode> stack = new Stack<>();
		if (root != null){
			stack.push(root);
		}
		while (!stack.empty()){
			TreeNode p = stack.pop();
			list.add(p.val);
			if (p.right != null){
				stack.push(p.right);
			}
			if (p.left != null){
				stack.push(p.left);
			}
		}
		return list;
	}
```
##### 哈希
####### 题目
Given an array of integers, find two numbers such that they add up to a specific target number.  
The function twoSum should return indices of the two numbers such that they add up to the target, where index1 must be less than index2. Please note that your returned answers (both index1 and index2) are not zero-based.
You may assume that each input would have exactly one solution.

Input: numbers={2, 7, 11, 15}, target=9  
Output: index1=1, index2=2
###### 解题思路
1. 遍历数组，将数组值作为key，数据下标作为value，放到hashmap中
2. 遍历数组时，计算当前值与target的差值，将差值作为key，冲哈希表中取出下标，若哈希表中有值，则返回这组下标

代码：
```java
public int[] twoSum(int[] numbers, int target) {
		HashMap<Integer, Integer> map = new HashMap<>();
		int[] res = new int[2];
		for (int i = 0; i < numbers.length; i++) {
			int diff =  target - numbers[i];
			Integer value = map.get(diff);
			if (value != null && value != i){
				if (value > i){
					res[0] = i + 1;
					res[1] = value + 1;
				}else {
					res[0] = value + 1;
					res[1] = i + 1;
				}
				return res;
			}
			map.put(numbers[i], i);
		}
		return res;
	}
```