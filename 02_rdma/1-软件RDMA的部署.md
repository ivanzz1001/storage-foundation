# 个人软件RDMA开发环境搭建

> 说明：本文基于`https://zhuanlan.zhihu.com/p/653997181` 及`自己的亲自实操验证` 整理而来


**RDMA**(remote direct memory access)即远端直接内存访问，是一种高性能网络通信技术，具有高带宽、低延迟、无CPU消耗、零拷贝等优点。相比kernel TCP、DPDK等传统通信手段，RDMA在延迟、吞吐和CPU消耗方面均有明显优势。

RDMA需要特定的硬件支持，比如`InfiniBand`网络需要一套独立于以太网的专门的网卡和交换机，`RoCE`是基于以太网的RDMA实现方案，对网卡仍然有一定要求。普通开发者如何搭建RDMA开发和测试环境呢？`Soft-RoCE(RXE)`应运而生。
