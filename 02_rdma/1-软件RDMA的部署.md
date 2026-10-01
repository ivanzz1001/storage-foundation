# 个人软件RDMA开发环境搭建

> 说明：本文基于`https://zhuanlan.zhihu.com/p/653997181` 及`https://zhuanlan.zhihu.com/p/361740115`整理而来

--- 

## 1. 前言

RDMA技术实际应用的话是得依赖网卡来完成大部分工作的，但是好在我们有Soft-RoCE。它通过软件代替硬件来将IB传输层的报文加在普通UDP报文中，从而得以让普通网卡也可以发送RoCE报文，这对于为我们学习IB传输层协议，以及编写调试基于Verbs的RDMA程序提供了一种非常低成本的方案。

本篇文章前半部分是讲解，后半部分是实验。首先我将介绍RoCE是什么、它的由来以及Soft-RoCE的实现原理，最后介绍如何在只有一台没有IB网卡的PC的情况下搭建Soft-RoCE的实验环境，以后可以用来运行RDMA程序并通过抓包学习IB传输层协议。

---


## 2. RoCE是什么
**RDMA**(remote direct memory access)是一种硬件卸载的网络技术，无需操作系统内核参与，没有系统调用和上下文切换，不消耗CPU，也没有用户空间和内核空间的来回内存拷贝（彻底的零拷贝）。

**RoCE**(RDMA over Converged Ethernet)是一种基于以太网的RDMA实现方案。用通俗的话讲，就是基于传统以太网的部分下层协议，在其基础上实现Infiniband的部分上层协议。

RoCE本身分为两个版本，我们先简单讲一下发展历史：

- 1999年，由Compaq, Dell, HP, IBM, Intel, Microsoft和Sun公司组成了IBTA组织。愿景是设计一种更高速的新的互联协议规范标准，来应对传统以太网在面对未来计算机行业的发展时可能遇到的瓶颈。

- 2000年，IBTA组织设计并发布了Infiniband Architecture Specification 1.0（IB规范）。

- 2007年，IETF发布了iWARP（Internet Wide Area RDMA Protocol）的一系列RFC。

- 2010年，IBTA发布了RoCE v1规范。

- 2014年，IBTA发布了RoCE v2规范。

可以看出相比于上个世纪70年代左右诞生的TCP/IP协议族来说，RoCE协议本身还算比较“年轻的”

### 2.1 RoCE的协议层次

下图比较清晰的展现了RDMA几种协议的关系：

![rdma-roce](https://raw.githubusercontent.com/ivanzz1001/storage-foundation/master/02_rdma/assets/rdma_protocol_layer.jpg)

可能还不够直观，我们把RoCE v2的一个报文展开来看（没有画出物理层协议）：





![rdma-roce](https://raw.githubusercontent.com/ivanzz1001/storage-foundation/master/02_rdma/assets/rdma_roce.jpg)



