---
layout: post
title: Leetcode刷题--Week 4
subtitle: Improving coding skills in 4th week
date: 2019-04-25T00:00:00.000Z
author: Xu Zhenxue
header-style: text
catalog: true
tags:
  - Leetcode
---
> 滚动阅读全文

#### 前言
本周刷题未链表排序相关知识点

#### 题目
Given a singly linked list L: L 0→L 1→…→L n-1→L n,
reorder it to: L 0→L n →L 1→L n-1→L 2→L n-2→…  
You must do this in-place without altering the nodes' values.  
For example,  
Given{1,2,3,4}, reorder it to{1,4,2,3}.

#### 解题思路
1. 首先利用快慢指针找到中间节点
2. 从中间节点截断链表，并将中间节点之后得链表进行反转操作
3. 同时遍历前半段链表和后半段链表，将后半段插入前半段

#### 代码
```java
public void reorderList(ListNode head) {
		if (head == null){
			return;
		}
		ListNode fast = head;
		ListNode slow = head;
		while (fast.next !=null && fast.next.next != null){
			fast = fast.next.next;
			slow = slow.next;
		}
		if (fast == head){
			return;
		}
		ListNode h = new ListNode(0);
		h.next = slow.next;
		slow.next = null;
		ListNode cur = h.next;
		ListNode newHead = new ListNode(0);
		while (cur != null){
			ListNode temp = cur;
			cur = cur.next;
			temp.next = newHead.next;
			newHead.next = temp;
		}
		ListNode preInsert = head;
		cur = newHead.next;
		while (cur != null ){
			ListNode temp = cur;
			cur = cur.next;
			temp.next = preInsert.next;
			preInsert.next = temp;
			preInsert = preInsert.next.next;
		}
	}
```