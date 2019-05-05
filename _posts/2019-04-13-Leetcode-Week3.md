---
layout: post
title: Leetcode刷题--Week 3
subtitle: Improving coding skills in 3rd week
date: 2019-04-08T00:00:00.000Z
author: Xu Zhenxue
header-style: text
catalog: true
tags:
  - Leetcode
---
> 滚动阅读全文
#### 前言
本周练习Leetcode两题，分别是链表的插入排序和二叉树后续遍历  

#### 链表插入排序
###### 题目
Sort a linked list using insertion sort.

###### 解题思路
1. 一个指针指从原始列表头部开始逐步往后移指向插入元素；
2. 一个指针从头遍历已排序链表，找到插入节点，将元素插入：

###### 代码

```java
 if (head == null || head.next == null){
			return head;
		}
		ListNode first = new ListNode(0);
		first.next = head;
		ListNode cur = head.next;
		head.next = null;
		while (cur != null){
			ListNode p = first;
			while(p!=null && p.next != null){
				if (cur.val < p.next.val ){
					break;
				}
				p = p.next;
			}
			ListNode temp = cur.next;
			cur.next = p.next;
			p.next = cur;
			cur = temp;
		}
		return first.next;

```

#### 二叉树后序遍历
###### 题目
Given a binary tree, return the postorder traversal of its nodes' values.  
For example:
Given binary tree{1,#,2,3}, return[3,2,1].  
Note: Recursive solution is trivial, could you do it iteratively?

###### 解法一，递归遍历
递归遍历比较简单，直接上代码
###### 代码

```java
    public ArrayList<Integer> postorderTraversal(TreeNode root) {
		ArrayList<Integer> orderList = new ArrayList<>();
		traversal(orderList,root);
		return orderList;
	}
	
    public void traversal(ArrayList<Integer> list, TreeNode node){
		if (node == null ){
			return;
		}
		traversal(list, node.left);
		traversal(list, node.right);
		list.add(node.val);
	}

```
###### 解法二，非递归遍历
1. 将根节点如栈
2. 循环访问栈顶节点，若栈顶节点是叶子节点或是上次出栈节点的父节点，将其值放入列表中，并将该元素出栈；
3. 若右节点不为空则将右节点入栈，左节点不为空也将左节点入栈

###### 代码

```java
    public ArrayList<Integer> postorderTraversalItera(TreeNode root) {
		ArrayList<Integer> orderList = new ArrayList<>();
		Stack<TreeNode> stack = new Stack<>();
		if (root != null){
			stack.push(root);
		}
		TreeNode head = root;
		while (!stack.empty()){
			TreeNode p = stack.peek();
			if ( (p.left == null && p.right == null) || p.right == head || p.left == head ){
				orderList.add(p.val);
				head = p;
				stack.pop();
			}else {

				if (p.right != null){
					stack.push(p.right);
				}

				if (p.left != null){
					stack.push(p.left);
				}

			}
		}
		return orderList;
	}

```
