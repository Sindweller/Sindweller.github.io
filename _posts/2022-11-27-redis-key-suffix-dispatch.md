---
layout: default
title: 2022/11/27 在redis集群中给key赋递增后缀以均匀分配到节点上 【阅读笔记】
author: sindweller <sindweller5530@gmail.com>
tags: [存储]
---

## 概述
原文来自字节跳动技术团队：https://mp.weixin.qq.com/s/iZ9BX6cCCp_TB-SC3knuew

关于redis添加递增后缀从而均匀分配节点的技巧： 【【大厂文章速读】字节跳动-在REDIS集群中给key后加递增数字后缀，能否将key均匀分布在各节点上】 https://www.bilibili.com/video/BV1jG4y147SJ/?share_source=copy_web&vd_source=348d1bb980ad52e976e68ab6b081048c

本文是学习笔记。

对于一个key（例如id）来说，对其增加递增后缀，然后使用crc16算法来确定这个key+后缀应当被分配在哪个slot上。

验证方法：例如，可以指定递增20次，然后查看采样结果（应当分配到哪个slot），来模拟实际消费后的分配结果。

> 为什么是20次？是因为这个需求的实际场景是消费券模板，假设每个分片上券模版的库存为20.

原文提供的流程图：

![](https://mmbiz.qpic.cn/mmbiz_png/5EcwYhllQOgM9JC70CE93Gv7jUibxuaxW3R9hicndCnibiaib20sgictR0jf2g9KSPXxEoXUxRIsV7wo9NO7egP3TMLw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

### CRC16 算法
redis 使用 CRC16 来计算【key+后缀】应当被分配到哪个slot，来尽可能均匀分配。

CRC是一种差错校验码，其信息字段和校验字段的长度可以任意。

任意二进制位串都可以对应一个系数为0或1的多项式，称为生成多项式。如CRC16的生成多项式为： `x16+x15+x2+1`。

校验内容组成：信息字段+校验字段。

### redis具体实现

每个分片负责一定范围内的slot。通过查看key->slot的分布情况，可以查看是否均匀分配到多个分片上。

## redis 分布式存储

redis cluster是redis的分布式架构
- 将数据分片，每个master上存放一部分数据
- 提供高可用
- 开放两个端口，一是6379，二是16379（用于集群节点之间通信，故障检测、配置更新、故障转移等）
  
### 集群和哨兵模式的区别

哨兵模式主要是为了多个redis节点高可用，哨兵作为一个监控进程来检测各个redis节点的状态，当一个主节点挂掉的时候，哨兵会监控到这个信息，并且把其中一台redis从节点升级为主节点，然后将这个信息发送给各个从节点，使他们的主从同步更换主服务器ip。

redis集群也是高可用，但它还解决了哨兵模式没有的**负载均衡**问题；集群=高可用+缓存分片。但根据分片算法也有可能多个key落入同一个Redis服务器中，所以我们还希望使用一致性哈希算法。

当超过半数的主服务器投票认为一台主服务器不在线之后，就会从集群中剔除该节点，使用其从节点代替。

**Redis集群不保证数据一致性**，所以更多的是用来做缓存而不是做存储。

#### 哈希槽slot概念

哈希槽中的数字决定了数据会发送到哪台Redis服务器进行存储，也就是说数据先放到slot上，然后根据slot对应哪台服务器发到服务器上进行同步。

redis集群的哈希槽大小为16384（取值[0,16384])。Redis集群会采用上小节所述的CRC16算法来计算key的哈希值，然后对16384取模即可得到结果n，用n来寻找哈希槽，进一步根据哈希槽来寻找服务器进行实际的存储。

每个哈希槽对应的服务器也有主从之分。

#### 一致性哈希算法

服务器数量n会有变化，像数组增删一样，变化后需要将其编号之后的服务器编号-1，此时每个key需要按照（n-1）重新取模来确定其位置。大量key会重定向到其他服务器中，就如同哈希表的扩缩容。这种情况在分布式系统中是非常糟糕的。因此，需要对分布式哈希方案进行设计，使得服务器节点变更不会造成过多的哈希重定位。

一致性哈希是一种特殊的哈希算法，他保证了slot的改变平均只需要对K/n个关键字重更新映射（K是关键字的数量，n是slot数量）。传统哈希表中，slot的变更几乎需要对所有已存在关键字进行重新映射。

抽象来看，一致性哈希算法是将整个哈希值空间组成一个虚拟圆环，0与2^32-1重合。

将服务器按照某个属性（例如ip）作为关键字进行哈希，确定其在哈希环的唯一位置。定位好服务器之后，就可以同样计算key的数据了。当key对应的环内位置确定，从该位置顺时针遇到的第一台服务器就是其应当定位到的服务器。

当一个服务器宕机，该服务器上（前面）所定位的数据将会顺延至下一个服务器，而也只有这些数据需要重定位，不会干扰其他数据。

因此一致性哈希算法的优势就体现在每次服务器节点的增减都只需要变动一小部分数据。

#### 虚拟节点解决节点分布不均匀的问题

一致性哈希算法引入虚拟节点机制，对每个节点计算多个哈希值，每个计算结果都定位到哈希环中，称为虚拟节点。例如可以放server1#1,server1#2...

hash算法不变，增加了虚拟节点到实际节点的映射，这样如果数据定位后落入虚拟节点，实际上保存的是正式节点中。解决了数据分布到服务节点时不平均的问题。