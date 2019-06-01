---
layout: post
title: 一次Redis内存飙升的排查
subtitle: A troubleshooting of Redis memory soaring
date: 2019-06-01T00:00:00.000Z
author: Xu Zhenxue
header-style: text
catalog: true
tags:
  - Redis
  - Troubleshooting
---
> 滚动阅读全文


#### 一、现象
在该项目中，多个微服务使用一台Redis虚拟机，项目开发完成后，进入试运行阶段，在项目平稳运行五天后，Redis使用的虚拟机内存在某一时刻突然飙升，很短的时间内内存耗尽，Redis虚拟机宕机，所有微服务连接Redis超时无法使用。
![异常图片](https://img-blog.csdnimg.cn/20190601113129794.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1poZW54dWVfWHU=,size_16,color_FFFFFF,t_70)
#### 二、问题排查
1. **怀疑在某一时刻出现大量写入数据导致内存飙升**。我们的业务存到redis的数据量几乎是可以预估的，应该向之前一样平稳，除非对方公司突然增加很多合作方，但是并没有这样的情况。为了确认不是这个原因所导致的，我们查看了Redis的Info，看到Redis使用的峰值也才5Mb左右，所以应该不是这个原因导致的。
2. **Redis连接未释放，导致大量连接耗尽内存**。我们设置的连接池最大连接数为1000，根据我们的业务场景几乎是不可能耗尽的。第一次排查时，因为对方公司重启了Redis服务器，所以没法看到宕机时的连接数，但是一直也想不到其他原因。第二次出现该问题时通过Redis Manager看了Redis连接数，果然是连接数达到了1000左右的问题。

#### 三、问题产生根本原因
我们定位到了问题产生原因为连接增加耗尽了内存，接下来我们要从代码中分析是什么原因导致了连接数的持续增加。
1. **高并发导致**，我们的Redis操作都是在线程池管理的线程中进行，线程池最大线程数也就20，几个微服务加起来也不超过200，故排除
2. **使用后没有释放连接**，我们使用RedisTemplate进行redis操作，应该用完会释放。唯一需要手动释放的地方，是在一个Redis工具类中使用了Redis的事务支持，所以定位问题应该出现在这里。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190601121821870.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1poZW54dWVfWHU=,size_16,color_FFFFFF,t_70)
由于团队内是共享该工具类的，如果几个微服务同时出现该问题，连接数能够立刻飙升。

#### 四、解决方案
在网上也看到很多人在使用多线程进行Redis事务操作时也出现了该类问题，他们给出的解决方案是增加线程池大小。但是我觉得并不能实质性的解决问题。我觉得如果要使用保证事务性的化可以考虑使用Lua脚本，将事务操作都放在脚本中进行（在看分布式锁实现时学来的）。  
对我们这个项目来说，其实这段代码是没有任何意义的，这段事务中仅有一个操作而已，也不太清楚写这段代码小伙伴的用途。
最后，我们将这段代码的事务支持取消了，Redis就没有再出现该问题。

#### 五、总结
对于我来说，这也是第一次在分布式环境中排查该类问题，也是刚学Redis而已，不过和师兄一起解决这个问题的过程，受益匪浅。
