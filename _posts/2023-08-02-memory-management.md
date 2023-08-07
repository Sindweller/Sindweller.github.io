---
layout: default
title: 2023/08/02 系统内存管理
author: sindweller <sindweller5530@gmail.com>
tags: [操作系统]
---

# 虚拟内存

虚拟内存的出现是为了隔离程序和物理内存，以防止两个程序同时读写同一块内存。操作系统负责管理虚拟地址和物理地址的映射，程序只操作自己所拥有的那一部分虚拟内存。

硬件上，是通过CPU芯片中的内存管理单元（MMU）存储映射关系。

整个流程： CPU->虚拟地址->MMU->物理地址->物理内存

操作系统管理虚拟地址和物理地址的关系有两种方式：内存分段和内存分页

## 内存分段

按照逻辑分段，分为代码段、数据段、栈、堆。每个分段都有自己的属性。在这种情况下，通过段选择子和段偏移量来记录虚拟地址，同时使用段表来映射物理地址。

- 段选择子：保存在段寄存器。段选择子里包含
  - 段号：段表的索引。
- 段偏移量：位于0到段界限之间，如果合法，则物理内存地址=段基地址+段偏移量
  
虚拟地址分为4个段（代码段、数据段、堆、栈），每个段在段表中是一个项，在这个item中找到段基地址+段偏移量就能得到物理地址。

### 产生的问题

#### 内存碎片

因为内存分段是按照实际需求分配内存的，不会出现内部内存碎片，但是每个段的长度不固定，段之间可能产生不连续的物理内存，出现外部内存碎片。

解决外部内存碎片的方式是内存交换。使用到Swap空间，这块空间是从硬盘划分出来的。

先将256MB文件从内存写到硬盘，然后从硬盘读回来。这样就能跟在已被占用的内存后面