---
layout: post
title: Leetcode刷题--Week 1
subtitle: Improving coding skills in 1st week
date: 2019-03-23T00:00:00.000Z
author: Xu Zhenxue
header-style: text
catalog: true
tags:
  - Leetcode
---
>滚动阅读全文
## 前言
眼瞅今年暑假就要开始找工作了，正好借助Leetcode刷题复习一下相关算法，随便为找工作做些准备。接下来每周至少回顾一个算法同时刷一道题，每周打卡。

## 本周刷题
#### 树
**题目**：
Given a binary tree, find its minimum depth.The minimum depth is the number of nodes along the shortest path from the root node down to the  nearest leaf node.

**解题思路**：
思路一：深度优先遍历，递归遍历左右子树，遇到叶子节点时返回 1，返回过程中，比较左右子树，较小深度逐层加1。

```java
public int run(TreeNode root) {
		if(root==null) {
			return 0;
		}
		if(root.left==null && root.right==null) {
			return 1;
		}
		int l=run(root.left);
		int r=run(root.right);
		if(l==0||r==0){ return 1+r+l;}
		return 1+Math.min(l,r);
	}
```

思路二：广度优先遍历，逐层遍历节点，遇到第一个叶子节点返回其深度；

```
public int runDFS(TreeNode root){
		if (root == null){return 0;}
		LinkedList<TreeNode> linkedList = new LinkedList<>();
		root.val = 1;
		linkedList.add(root);
		while (!linkedList.isEmpty()){
			TreeNode node = linkedList.pollFirst();
			if (node.left == null && node.right == null){return node.val;}
			if (node.left != null){
				node.left.val = node.val + 1;
				linkedList.add(node.left);
			}
			if (node.right != null ){
				node.right.val = node.val + 1;
				linkedList.add(node.right);
			}
		}
		return  0;

	}

```
代码：https://github.com/ZhenxueXu/leetcode/blob/master/src/solution/MinimumDepthOfBinaryTree.java


#### 栈
**题目**：
Evaluate the value of an arithmetic expression in Reverse Polish Notation.
Valid operators are+,-,*,/. Each operand may be an integer or another expression.

Some examples:
```
["2", "1", "+", "3", "*"] -> ((2 + 1) * 3) -> 9
["4", "13", "5", "/", "+"] -> (4 + (13 / 5)) -> 6
```
**解题思路**：
 根据栈的原理，将数字依次入栈，遇到操作符时，从栈顶弹出两个数字，计算后将结果继续入栈；

 ```java
public int evalRPN(String[] tokens) {
		if (tokens == null || tokens.length == 0){
			return 0;
		}
		Stack<Integer> value = new Stack<>();
		int left = 0;
		int right = 0;
		int  result = 0;
		for (String token : tokens) {
			switch (token){
				case "+":
					right = value.pop();
					left = value.pop();
					result = left + right;
					value.push(result);
					break;
				case "-":
					right = value.pop();
					left = value.pop();
					result = left - right;
					value.push(result);
					break;
				case "*":
					right = value.pop();
					left = value.pop();
					result = left * right;
					value.push(result);
					break;
				case "/":
					right = value.pop();
					left = value.pop();
					result = left / right;
					value.push(result);
					break;
					default:
						value.push(Integer.valueOf(token));
			}
		}
		return value.pop();
	}
```
代码：https://github.com/ZhenxueXu/leetcode/blob/master/src/solution/EvaluteReversePolishNotation.java
