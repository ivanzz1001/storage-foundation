# 个人软件RDMA开发环境搭建

> 说明：本文基于`https://zhuanlan.zhihu.com/p/653997181` 及`自己的亲自实操验证` 整理而来

--- 

## 1. 前言

**RDMA**(remote direct memory access)即远端直接内存访问，是一种高性能网络通信技术，具有高带宽、低延迟、无CPU消耗、零拷贝等优点。相比kernel TCP、DPDK等传统通信手段，RDMA在延迟、吞吐和CPU消耗方面均有明显优势。

RDMA需要特定的硬件支持，比如`InfiniBand`网络需要一套独立于以太网的专门的网卡和交换机，`RoCE`是基于以太网的RDMA实现方案，对网卡仍然有一定要求。普通开发者如何搭建RDMA开发和测试环境呢？`Soft-RoCE(RXE)`应运而生。

---

## 2. 什么是RoCE和Soft-RoCE(RXE)

**RDMA**(remote direct memory access)是一种硬件卸载的网络技术，无需操作系统内核参与，没有系统调用和上下文切换，不消耗CPU，也没有用户空间和内核空间的来回内存拷贝（彻底的零拷贝）。

**RoCE**(RDMA over Converged Ethernet)是一种基于以太网的RDMA实现方案，相比于InfiniBand网络RoCE不需要昂贵的专用的网络设备（网卡，交换机），而是基于数据中心已有的以太网架构工作，因此基于RoCE的RDMA技术成为数据中心广泛使用的高性能网络方案（微软、阿里等）。RoCE仍然依赖于数据中心网络，对个人来讲还是不容易接触。
