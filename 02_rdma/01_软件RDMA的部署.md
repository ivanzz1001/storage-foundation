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

可能还不够直观，我们把RoCE v2的一个报文展开来看：
```text
RoCE v2 = 标准以太网帧 + IP + UDP(4791) + IB 传输层
┌──────────────────────────────────────────────────────────────────────┐
│ Ethernet Header (14B)                                                │
│ ┌──────────────┬──────────────┬──────────────────────┐               │
│ │ Dst MAC (6B) │ Src MAC (6B) │ EtherType (2B)       │               │
│ └──────────────┴──────────────┴──────────────────────┘               │
├──────────────────────────────────────────────────────────────────────┤
│ IP Header (IPv4 20B / IPv6 40B)                                      │
│ ┌──────┬──────┬────────────┬────────────┬───────────────┐            │
│ │ Ver  │ TOS  │ Protocol=17│ Src IP     │ Dst IP        │            │
│ └──────┴──────┴────────────┴────────────┴───────────────┘            │
├──────────────────────────────────────────────────────────────────────┤
│ UDP Header (8B)                                                      │
│ ┌──────────────┬──────────────────┬──────────┬──────────┐            │
│ │ Src Port     │ Dst Port = 4791  │ Length   │ Checksum │            │
│ └──────────────┴──────────────────┴──────────┴──────────┘            │
├──────────────────────────────────────────────────────────────────────┤
│ IB BTH (12B)                                                         │
│ ┌────────┬────────┬────────┬───────┬──────────┬───────┬─────────┐    │
│ │ OpCode │ Flags  │ P_Key  │ Rsvd  │ Dest QP  │ A/Rsvd│ PSN     │    │
│ │ 8b     │ 8b     │ 16b    │ 8b    │ 24b      │ 8b    │ 24b     │    │
│ └────────┴────────┴────────┴───────┴──────────┴───────┴─────────┘    │
├──────────────────────────────────────────────────────────────────────┤
│ IB ExtHDR (可选)  RETH / AETH / ImmDt / DETH ...                     │
├──────────────────────────────────────────────────────────────────────┤
│ IB Payload (RDMA 数据)                                               │
├──────────────────────────────────────────────────────────────────────┤
│ ICRC (4B)  IB 层端到端校验，覆盖 BTH~Payload                          │
├──────────────────────────────────────────────────────────────────────┤
│ Ethernet FCS (4B)  链路层校验，通常网卡硬件处理                      │
└──────────────────────────────────────────────────────────────────────┘
```

### 2.2 RoCE的优势

为什么我们有了Infiniband协议之后，还要设计RoCE协议呢？最主要的原因还是成本问题：由于Infiniband协议本身定义了一套全新的层次架构，从链路层到传输层，都无法与现有的以太网设备兼容。也就是说，如果某个数据中心因为性能瓶颈，想要把数据交换方式从以太网切换到Infiniband技术，那么需要购买全套的Infiniband设备，包括网卡、线缆、交换机和路由器等等。商用级设备由于对可靠性有比较高的要求，所以这一套下来是非常昂贵的。

而RoCE协议的出现解决了这一问题，如果用户想要从以太网切换到RoCE，那么只需要购买支持RoCE的网卡就可以了，线缆、交换机和路由器（RoCE v1不支持以太网路由器）等网络设备都是兼容的——因为我们只是在以太网传输层基础上又定义了一套协议而已。

所以RoCE相比于Infiniband，主要还是省钱，当然性能上相比Infiniband还是有一些损失，毕竟人家是全套重新设计的。

至于iWARP，相比于RoCE协议栈更复杂，并且由于TCP的限制，只能支持可靠传输，即无法支持UD等传输类型。所以目前iWARP的发展并不如RoCE和Infiniband。


### 2.3 Soft-RoCE
虽然RoCE相比Infiniband具有兼容性优势，价格也便宜，但是**实际应用**的时候**依然需要专用的网卡支持**。有的读者可能会问，TCP/IP协议栈不是由软件实现的吗，只是在UDP层基础上加了层内容，为什么会对硬件有依赖？

RoCE本身确实可以由软件实现，也就是本节即将介绍的**Soft-RoCE**，但是商用的时候，几乎不会有人用软件实现的RoCE。RDMA技术本身的一大特点就是`“硬件卸载”`，即把本来软件（CPU）做的事情放到硬件中实现以达到加速的目的。CPU主要是用来计算的，让它去处理协议封包和解析以及搬运数据，这是对计算资源的浪费。所以RoCE网卡会把TCP/IP协议栈放到硬件中实现以解放CPU，让它去做更重要的事。

我们说回Soft-RoCE，它由IBM和Mellanox牵头的IBTA RoCE工作组实现。本身的设计初衷有几点：

- ##### 降低RoCE部署成本

  Soft-RoCE可以使不具备RoCE能力的硬件和支持RoCE的硬件间进行基于IB语义的交流，这样可以免于替换网络中的一些非关键节点的旧型号网卡
  
- ##### 相比TCP提升性能？

  虽然软件实现IB传输层带来了一定的开销，但是相比基于Socket-TCP/IP的传统通信方式，Soft-RoCE因为减少了系统调用（只在软件通知硬件下发了新SQ WQE时才会使用系统调用），发送端的零拷贝以及接收端的只需要单次拷贝等原因，仍然带来了性能上的提升？

  > **注意：根据网友们的反馈以及我自己实测，Soft-RoCE的性能不及TCP。**
  > 
  > 主要是几个原因：
  > 1) IB传输层MTU最大为4096，256 Bytes的Header + 4096 Bytes的Payload，Header所占比例较高；而TCP的MTU可以很大，相当于提高了有效载荷。
  > 2) 网卡往往可以为TCP提供硬件加速功能。
  > 3) Soft-RoCE用CPU去计算CRC，这是一件很慢的事情。
  >  
  > 这里提供我能想到的提升性能的思路：
  > 1) 在编程时做多线程，每个线程绑定一个核，并且每个线程间不要共享使用QP，因为会出现抢锁。
  > 2) 使用WR List代替单个WR，即每次Post Send时下发多个WR组成的WR链表，减少敲Doorbell时的系统调用开销。
  > 3) 将网卡的MTU值设置为大于4096 + 256，可以避免链路层切包的开销。
  > 4) 如果是RXE对接测试，可以通过修改RXE驱动关闭CRC校验，并提高RoCE的MTU值，但是这样违反了协议，貌似没什么意义。
  >  
  > 附社区讨论该问题的链接：[Soft-RoCE performance - Christian Blume (kernel.org)](https://lore.kernel.org/linux-rdma/CAGP7Hd6PAYcX_gMMh8jbpezeSSWQxqDrYwxEq1N-zjgT7563+g@mail.gmail.com/)

- ##### 便于开发和测试RDMA程序

  有了Soft-RoCE，我们基于**Verbs API**编写的程序，就可以不依赖于硬件执行起来，也可以很方便的跑在虚拟机里。


### 2.4 实现原理

Soft-RoCE就是把本来应该卸载到硬件的封包和解析工作，又拿到软件来做。其本身是基于Linux内核的TCP/IP协议栈实现的，网卡本身并不感知收发的数据包是RoCE报文，其驱动程序按照IB规范中的报文格式将用户数据封装成IB传输层报文，然后把报文整体当做数据填入Socket Buffer当中，由网卡进行下一步收发包处理。

> 下面这张图取自IBTA对于Soft-RoCE的介绍文章[1]，左边是需要硬件的普通RoCE，右边是Soft-RoCE。可以看出普通RoCE是把协议栈卸载到RoCE NIC网卡实现的，而Soft-RoCE则是在软件协议栈中实现的。

![rdma-roce](https://raw.githubusercontent.com/ivanzz1001/storage-foundation/master/02_rdma/assets/rdma_roce.jpg)

---

## 3. 实验环境部署

下面开始实操部分。当前笔者使用的是`Ubuntu 22.04`, 该版本只需要很简单的配置就可以跑RDMA的程序了，并且比较新.

```bash
# cat /etc/lsb-release 
DISTRIB_ID=Ubuntu
DISTRIB_RELEASE=22.04
DISTRIB_CODENAME=jammy
DISTRIB_DESCRIPTION="Ubuntu 22.04.4 LTS"
```

### 3.1 准备环境

我们测试的网络拓扑很简单，一台PC，以及其上运行的两台Ubuntu虚拟机都连接到一个虚拟子网上，两台虚拟机上将运行Soft-RoCE，我们在宿主机上通过Wireshark抓取数据包。

![rdma-roce](https://raw.githubusercontent.com/ivanzz1001/storage-foundation/master/02_rdma/assets/rdma_roce_deploy.jpg)

确保上面三个主机之间都可以相互ping通。

### 3.2 确认当前内核是否支持RXE

执行如下命令查询：
```bash
# cat /boot/config-$(uname -r) | grep RXE      
CONFIG_RDMA_RXE=m
```

1）如果CONFIG_RDMA_RXE的值为y或者m，表示当前的操作系统可以使用RXE

  > **CONFIG_RDMA_RXE**的具体含义是Linux内核中控制Soft-RoCE软件实现的编译开关，当配置值为y时，该驱动会直接编译进内核；配置值为m时，则会以独立内核模块rdma_rxe的形式存在，需要手动加载。

2）如果该选项值为n或者搜索不到RXE，那么很遗憾你可能需要重新编译内核

  编译内核时需要使能如下几个选项：
  ```text
  CONFIG_INET
  CONFIG_PCI
  CONFIG_INFINIBAND
  CONFIG_INFINIBAND_VIRT_DMA
  ```
  至于具体的重新编译内核的方法，可以先自行查找。

### 3.3 安装开发工具

#### 3.3.1 安装软件包
在两个节点执行：
```bash
# sudo apt update
# sudo apt install -y \
  build-essential pkg-config iproute2 ethtool tcpdump \
  rdma-core ibverbs-providers ibverbs-utils \
  libibverbs-dev librdmacm-dev rdmacm-utils perftest
```

如果 perftest 无法找到，启用 Ubuntu Universe 仓库后重试：

```bash
sudo add-apt-repository universe
sudo apt update
sudo apt install -y perftest
```
下面介绍一下上面安装的各组件:

1）**build-essential**

  build-essential 是 Debian/Ubuntu 下的一个元包，本身不包含具体程序，而是依赖一组常用编译工具：
  
  - gcc
  - g++
  - make
  - libc6-dev
  - dpkg-dev

  它常用于从源码编译软件。可以使用`dpkg -s build-essential`来检查是否已经安装:

  ```bash
  # dpkg -s build-essential
  Package: build-essential
  Status: install ok installed
  ```

2）**iproute2（可选）**

  iproute2 是 Linux 下最核心的网络管理工具集，用于配置网络接口、IP 地址、路由、隧道、流量控制、套接字统计等。它是现代 Linux 中替代老式 net-tools（ifconfig、route、netstat、arp）的标准工具。
  主要包含如下：
  
  - `ip`: 网络接口、地址、路由、邻居、规则、命名空间等
  - `ss`: 查看套接字连接，替代 netstat
  - `tc`: 流量控制、QoS、限速、队列规则
  - `bridge`: 网桥管理
  - `rdma`: RDMA 设备管理
  - `devlink`: 设备驱动参数、固件、端口管理
  - `genl`: 通用 netlink 操作
  - `nstat / lnstat`: 网络统计
  - `rtacct / rtmon`: 路由统计与监控

3）**rdma-core**

  `rdma-core`是RDMA的“用户态基础环境包”， 含有RDMA 用户态基础组件和配置。

  执行`apt install rdma-core`是安装的内容主要包括：

  - **自动安装依赖库**

    - libibverbs1：libibverbs 运行时库
    - librdmacm1：RDMA CM 运行时库
    - libibumad3：用户态 MAD 库
    - libibmad5：MAD 库
    - libnl-3-200、libnl-route-3-200 等网络库

  - **核心工具和守护进程**

    - rdma 命令：查看/管理 RDMA 设备，如 rdma link show、rdma dev show
    - rdma-ndd：RDMA 网络设备命名守护进程
    - 相关 udev 规则和 systemd 服务

  - **默认可能安装的推荐工具**

    apt 默认会安装推荐包，通常还包括：
    - ibverbs-utils：提供 ibv_devices、ibv_devinfo 等
    - rdmacm-utils：提供 rping、ucmatose 等
    - infiniband-diags：可能提供 ibstat 等

4）**ibverbs-providers**

  ibverbs-providers 是 rdma-core 项目下的一个包，专门包含 libibverbs 的用户态硬件驱动（provider drivers）。它是让上层应用能够通过 libibverbs 与你具体的 RDMA 网卡（HCA）通信的关键组件。
  >ps: 一般它会被 apt install rdma-core 命令自动作为依赖安装

   - **核心作用：连接通用库与具体硬件**
   
     要理解它的作用，需要先厘清 RDMA 用户态的几个层次：

     - 应用程序：如 MPI、NVMe-oF 等，它们调用通用的 RDMA 接口。
     - libibverbs 库：提供统一的 API（即“verbs”），应用程序通过它来请求 RDMA 操作。
     - ibverbs-providers：包含针对不同品牌和型号网卡的驱动程序。当应用程序通过 libibverbs 发起请求时，libibverbs 会加载对应的 provider 驱动，由它来直接操作硬件。
     
     简单说，libibverbs 是“插座”，而 ibverbs-providers 是各种“插头”，确保你的应用能连接到特定的 RDMA 硬件。

  - **包含哪些驱动**

    该包内含大量主流 RDMA 网卡的驱动，常见的有：
    - Mellanox (NVIDIA): mlx4 (ConnectX-3), mlx5 (Connect-IB/ConnectX-4及以上)
    - Broadcom: bnxt_re (NetXtreme-E RoCE)
    - Intel: hfi1verbs (Omni-Path), i40iw (X722 RDMA)
    - Amazon: efa (Elastic Fabric Adapter)
    - Chelsio: cxgb4 (T4 iWARP)
    
    软件实现: rxe (Soft-RoCE), siw (Soft-iWARP)，用于在没有硬件的情况下测试 RDMA 功能

5）**ibverbs-utils**

  `ibverbs-utils` 是 rdma-core 项目提供的一个用户态 RDMA 诊断与测试工具集，基于 libibverbs 库，用于查看 RDMA 设备信息、测试基本连通性和性能。它不包含驱动，也不包含开发库，只是一组命令行实用程序

  - **包含哪些工具**

    在 Debian/Ubuntu 中，ibverbs-utils 通常包含以下命令：

    - ibv_devices: 列出系统中所有可用的 RDMA 设备
    - ibv_devinfo: 显示指定 RDMA 设备的详细信息（端口、GID、MTU、状态等）
    - ibv_rc_pingpong: 测试 RC（可靠连接）模式的 RDMA 通信
    - ibv_uc_pingpong: 测试 UC（不可靠连接）模式
    - ibv_ud_pingpong: 测试 UD（不可靠数据报）模式
    - ibv_srq_pingpong: 测试共享接收队列（SRQ）
    - ibv_asyncwatch: 监控 RDMA 异步事件
    - ibv_odp_test: 测试按需分页（ODP）功能


6） **libibverbs-dev**

  `libibverbs-dev`是 `Debian/Ubuntu` 下 `libibverbs` 的开发包，用于编译和链接使用 RDMA verbs API 的程序。它提供头文件、静态库、符号链接和 pkg-config 文件；而运行时共享库由 libibverbs1 提供.

7） **librdmacm-dev**

  `librdmacm-dev` 是` Debian/Ubuntu` 下 `librdmacm` 的开发包，用于编译和链接使用 RDMA 连接管理（RDMA CM） API 的程序。它提供头文件、开发用符号链接、静态库和 pkg-config 文件；运行时共享库由 librdmacm1 提供

  - **什么是 librdmacm**

    librdmacm 是 RDMA 用户态栈中的连接管理库，提供一套类似 socket 的 API（rdma_cm），用于：
    
    - 建立、维护和拆除 RDMA 连接
    - 解析地址、解析路由
    - 管理事件（连接请求、建立、断开等）
    - 支持 InfiniBand、RoCE、iWARP 等多种 RDMA 传输
    
    它位于 libibverbs 之上：libibverbs 负责底层 verbs 操作（QP、CQ、MR 等），librdmacm 负责连接的生命周期管理。

8）**rdmacm-utils**

  `rdmacm-utils` 是 `rdma-core` 项目下的一个工具包，它提供了一组基于 librdmacm 库的示例程序和诊断测试工具，用于验证 RDMA 连接管理（RDMA CM）的功能是否正常工作

  - **包含哪些工具**

    该包包含多个用于测试和演示的实用程序，主要工具如下:

    - rping: 最常用的工具，用于测试 RDMA CM 的连接建立和 ping-pong 通信
    - ucmatose: 测试不可靠连接（UC）模式的 RDMA 通信
    - udaddy: 测试不可靠数据报（UD）模式的 RDMA 通信
    - mckey: 测试 RDMA CM 的多播（Multicast）设置和简单数据传输
    - rdma_client / rdma_server: 简单的 RDMA 客户端/服务器示例
    - rdma_xclient / rdma_xserver: 基于扩展 API 的 RDMA 客户端/服务器示例
    - riostream / rstream: 测试 RDMA 流式通信
    - rcopy: 使用 RDMA 进行文件复制测试
    - cmtime: 测量 RDMA CM 事件的时间

9）**perftest**

  `perftest` 是一套基于 libibverbs 接口开发的RDMA 性能基准测试工具集，专用于评估 RDMA 网络（InfiniBand、RoCE、iWARP）的带宽和延迟性能。它采用客户端-服务器（Client-Server）架构，通过网络连接在两台机器间进行点对点性能测试

  - **包含哪些工具**

    perftest 提供了覆盖不同 RDMA 操作类型的带宽和延迟测试程序:

    - ib_send_bw / ib_send_lat: 带宽 / 延迟	SEND 操作
    - ib_write_bw / ib_write_lat: 带宽 / 延迟	RDMA WRITE 操作
    - ib_read_bw / ib_read_lat: 带宽 / 延迟	RDMA READ 操作
    - ib_atomic_bw / ib_atomic_lat: 带宽 / 延迟	Atomic 原子操作
    - raw_ethernet_bw / raw_ethernet_lat 等: 带宽 / 延迟	原始以太网（Raw Ethernet）
   
    此外还包括辅助脚本 run_perftest_loopback（回环测试）和 run_perftest_multi_devices（多设备测试）

#### 3.3.2 检查命令和头文件

```bash
# command -v rdma ibv_devices ibv_devinfo rping ib_write_bw
/usr/bin/rdma
/usr/bin/ibv_devices
/usr/bin/ibv_devinfo
/usr/bin/rping
/usr/bin/ib_write_bw
# pkg-config --modversion libibverbs librdmacm
1.14.39.0
1.3.39.0

# test -f /usr/include/infiniband/verbs.h && echo verbs-header-ok
verbs-header-ok
# test -f /usr/include/rdma/rdma_cma.h && echo rdmacm-header-ok
rdmacm-header-ok
```
所有命令应返回路径，两个头文件检查应输出 ok。

### 3.4 实验一 创建 Soft RoCE 设备

#### 3.4.1 加载内核模块
在两个节点执行：
```
# sudo modprobe rdma_rxe
# lsmod | grep -E 'rdma_rxe|ib_uverbs|rdma_cm'
rdma_rxe              196608  0
ib_uverbs             192512  1 rdma_rxe
ip6_udp_tunnel         16384  1 rdma_rxe
udp_tunnel             32768  1 rdma_rxe
ib_core               507904  2 rdma_rxe,ib_uverbs
```
如果 `modprobe` 提示找不到模块，先检查内核配置：
```bash
# grep CONFIG_RDMA_RXE /boot/config-$(uname -r)
```
期望值为 `CONFIG_RDMA_RXE=m` 或 `CONFIG_RDMA_RXE=y`。若当前云内核或裁剪内核未启用 RXE，需要换用 Ubuntu 通用内核或重新编译内核。

#### 3.4.2 绑定普通网卡

当前我们两台机器上的网卡均为`ens33`:
```bash
# ifconfig
ens33: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.180.131  netmask 255.255.255.0  broadcast 192.168.180.255
        ether 00:0c:29:80:ff:ae  txqueuelen 1000  (Ethernet)
        RX packets 90866  bytes 117373439 (117.3 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 25581  bytes 2329638 (2.3 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```
在两个节点分别执行如下命令绑定网卡:

```bash
# sudo rdma link add rxe0 type rxe netdev ens33
```
官方 `rdma-core` 使用相同的 `rdma link add NAME type rxe netdev DEVICE` 方式创建软件设备

#### 3.4.3 验证 RDMA 设备

分别在`192.168.180.131`和`192.168.180.132`这两个节点执行:
```bash
# rdma link show
link rxe0/1 state ACTIVE physical_state LINK_UP netdev ens33

# ibv_devices
    device                 node GUID
    ------              ----------------
    rxe0                020c29fffe80ffae

# ibv_devinfo -d rxe0
hca_id: rxe0
        transport:                      InfiniBand (0)
        fw_ver:                         0.0.0
        node_guid:                      020c:29ff:fe80:ffae
        sys_image_guid:                 020c:29ff:fe80:ffae
        vendor_id:                      0xffffff
        vendor_part_id:                 0
        hw_ver:                         0x0
        phys_port_cnt:                  1
                port:   1
                        state:                  PORT_ACTIVE (4)
                        max_mtu:                4096 (5)
                        active_mtu:             1024 (3)
                        sm_lid:                 0
                        port_lid:               0
                        port_lmc:               0x00
                        link_layer:             Ethernet
# ls -l /dev/infiniband
total 0
crw-rw-rw- 1 root root  10, 263 10月  8 18:03 rdma_cm
crw-rw-rw- 1 root root 231, 192 10月  8 18:03 uverbs0
```

典型结果应满足以下条件:

- rdma link show 包含 rxe0/1，状态通常为 ACTIVE，并显示绑定的普通网卡。
- ibv_devices 能列出 rxe0。
- ibv_devinfo 的 transport 为 InfiniBand transport，但端口 link_layer 为 Ethernet。
- /dev/infiniband 中存在 uverbs0 和 rdma_cm 等设备节点。

#### 3.4.4 查看 GID
```bash
# ibv_devinfo -d rxe0 -v | less
hca_id: rxe0
        transport:                      InfiniBand (0)
        fw_ver:                         0.0.0
        node_guid:                      020c:29ff:fe80:ffae
        sys_image_guid:                 020c:29ff:fe80:ffae
        vendor_id:                      0xffffff
        vendor_part_id:                 0
        hw_ver:                         0x0
        phys_port_cnt:                  1
        max_mr_size:                    0xffffffffffffffff
        page_size_cap:                  0xfffff000
        max_qp:                         1048560
        max_qp_wr:                      1048576
        device_cap_flags:               0x01223c76
                                        BAD_PKEY_CNTR
                                        BAD_QKEY_CNTR
                                        AUTO_PATH_MIG
                                        CHANGE_PHY_PORT
                                        UD_AV_PORT_ENFORCE
                                        PORT_ACTIVE_EVENT
                                        SYS_IMAGE_GUID
                                        RC_RNR_NAK_GEN
                                        SRQ_RESIZE
                                        MEM_WINDOW
                                        MEM_MGT_EXTENSIONS
                                        MEM_WINDOW_TYPE_2B
        max_sge:                        32
        max_sge_rd:                     32
        max_cq:                         1048576
        max_cqe:                        32767
        max_mr:                         524287
        max_pd:                         1048576
        max_qp_rd_atom:                 128
        max_ee_rd_atom:                 0
        max_res_rd_atom:                258048
        max_qp_init_rd_atom:            128
        max_ee_init_rd_atom:            0
        atomic_cap:                     ATOMIC_HCA (1)
        max_ee:                         0
        max_rdd:                        0
        max_mw:                         524287
        max_raw_ipv6_qp:                0
        max_raw_ethy_qp:                0
        max_mcast_grp:                  8192
        max_mcast_qp_attach:            56
        max_total_mcast_qp_attach:      458752
        max_ah:                         32767
        max_fmr:                        0
        max_srq:                        917503
        max_srq_wr:                     1048576
        max_srq_sge:                    27
        max_pkeys:                      64
        local_ca_ack_delay:             15
        general_odp_caps:
        rc_odp_caps:
                                        NO SUPPORT
        uc_odp_caps:
                                        NO SUPPORT
        ud_odp_caps:
                                        NO SUPPORT
        xrc_odp_caps:
                                        NO SUPPORT
        completion_timestamp_mask not supported
        core clock not supported
        device_cap_flags_ex:            0x1C001223C76
                                        Unknown flags: 0x1C000000000
        tso_caps:
                max_tso:                        0
        rss_caps:
                max_rwq_indirection_tables:                     0
                max_rwq_indirection_table_size:                 0
                rx_hash_function:                               0x0
                rx_hash_fields_mask:                            0x0
        max_wq_type_rq:                 0
        packet_pacing_caps:
                qp_rate_limit_min:      0kbps
                qp_rate_limit_max:      0kbps
        tag matching not supported
        num_comp_vectors:               128
                port:   1
                        state:                  PORT_ACTIVE (4)
                        max_mtu:                4096 (5)
                        active_mtu:             1024 (3)
                        sm_lid:                 0
                        port_lid:               0
                        port_lmc:               0x00
                        link_layer:             Ethernet
                        max_msg_sz:             0x80000000
                        port_cap_flags:         0x00010000
                        port_cap_flags2:        0x0000
                        max_vl_num:             1 (1)
                        bad_pkey_cntr:          0x0
                        qkey_viol_cntr:         0x0
                        sm_sl:                  0
                        pkey_tbl_len:           1
                        gid_tbl_len:            1024
                        subnet_timeout:         0
                        init_type_reply:        0
                        active_width:           1X (1)
                        active_speed:           2.5 Gbps (1)
                        phys_state:             LINK_UP (5)
                        GID[  0]:               fe80::20c:29ff:fe80:ffae, RoCE v2
                        GID[  1]:               ::ffff:192.168.180.131, RoCE v2
```
在端口信息中找到 `GID` 表。RXE 通常会为网卡 IP 生成 IPv4 映射 GID。后续如果工具不能自动选择正确 GID，可通过 `-g`或 `-x` 显式指定索引。

#### 3.4.5 删除和重建
```bash
# sudo rdma link delete rxe0
# sudo rdma link add rxe0 type rxe netdev ens33
```
成功删除设备不会删除普通网卡或其 IP 地址。它只移除绑定在该网卡上的 RXE RDMA 设备.
