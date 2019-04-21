---
layout: post
title: 基于Semaphore实现QPS，吞吐率限制
subtitle: One QPS limiter
date: 2019-04-14T00:00:00.000Z
author: Xu Zhenxue
header-style: text
catalog: true
tags:
  - java
---
> 滚动阅读全文

#### 业务场景
在一个项目中需要频繁调用一个第三API接口来实现业务功能，但是这个第三方接口有QPS和吞吐率的限制，如果超过限制将请求失败；故我们需要在我们的业务功能中加上吞吐率（吞吐率 < QPS）的限制, 当达到限制时让当前的请求等待到下一个可执行时间段执行。

#### 吞吐率实现 
吞吐率的限制是当前时段内（1S内）最多能处理的任务数量，超过限制必须等待；这个有点类似生产者-消费者模式，请求是生产者，数量没有限制；限制的吞吐量为消费者数量。  
使用Java 并发工具Semaphore 来实现对吞吐量的限制，当请求到达时，若吞吐量未达到限制，判断当前时间与最早请求处理时间是否达到时间间隔，如果达到着执行；未达到则休眠剩余时间后再执行；若吞吐量已达到限制，则等待获得信号量后执行；  

使用数组存储每个请求开始处理的时间，数组大小等于吞吐量限制大小；

```java
package com.anyunbao.common.utils.qps;

import java.util.Arrays;
import java.util.concurrent.Semaphore;
import java.util.concurrent.TimeUnit;

/**
 * @author Think
 * @className ThroughputLimiter
 * @description QPS 限制工具
 * @date 2019/4/13
 **/
public class ThroughputLimiter {
	/**
	 * 接受请求时间窗口
	 */
	private long[] accessTime;
	/**
	 * 限制数量
	 */
	private int limit;
	/**
	 * 指向最早请求时间的位置
	 */
	private int curPosition;
	/**
	 * 时间间隔
	 */
	private long period;
	/**
	 * 限制请求的信号量;
	 */
	private Semaphore semaphore;


	private final String LOCK = "LOCK";

	public ThroughputLimiter(int limit, long period, TimeUnit timeUnit) {
		if (limit < 0) {
			throw new IllegalArgumentException("Illegal Capatity: " + limit);
		}
		this.curPosition = 0;
		this.period = timeUnit.toMillis(period);
		this.limit = limit;
		this.semaphore = new Semaphore(limit);
		this.accessTime = new long[limit];
		Arrays.fill(accessTime, 0);
	}
	

	public ThroughputLimiter(int limit) {
	// 默认间隔为1S
		this(limit, 1, TimeUnit.SECONDS);
	}

	public void acquire() throws InterruptedException {
		synchronized (LOCK) {
			semaphore.acquire();
			long curTime = System.currentTimeMillis();
			if (curTime - accessTime[curPosition] < period) {
			    // 未达到处理间隔， 休眠间隔剩余时间
				Thread.sleep(period - (curTime - accessTime[curPosition]) + 1);
				curTime = System.currentTimeMillis();
			}
			accessTime[curPosition++] = curTime;
			curPosition = curPosition % limit;
		}
	}
	
	
	public void release(){
	    // 任务处理完后释放一个信号量
		semaphore.release();
	}
}


```

#### QPS实现
QPS的思想和吞吐率类似，只是我们不需要考虑当前正在执行的任务数量，故去掉信号量的信息即可

```
	public void limit() throws InterruptedException {
		synchronized (LOCK) {
			long curTime = System.currentTimeMillis();
			if (curTime - accessTime[curPosition] < period) {
			    // 未达到处理间隔， 休眠间隔剩余时间
				Thread.sleep(period - (curTime - accessTime[curPosition]) + 1);
				curTime = System.currentTimeMillis();
			}
			accessTime[curPosition++] = curTime;
			curPosition = curPosition % limit;
		}
	}
```

