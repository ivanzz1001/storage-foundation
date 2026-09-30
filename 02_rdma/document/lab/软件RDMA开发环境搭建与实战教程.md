# 软件 RDMA 开发环境搭建与实战教程

基于 Ubuntu 24 04 LTS 和 Soft RoCE

版本 1.0  2026 年 9 月

## 文档目标

本教程帮助没有 RDMA 网卡的开发者搭建一套可重复的软件 RDMA 实验环境，并沿着设备识别、连接建立、数据传输、性能测试和 C 语言编程逐步完成实验。完成全部实验后，读者应能解释 PD、MR、CQ、QP、WR、WC 和 RDMA CM 的作用，能定位常见的链路与权限问题，也能把同一套用户态程序迁移到真实 RoCE 或 InfiniBand 网卡。

主线环境使用两台 Ubuntu 24.04 LTS 节点，每台节点把内核 Soft RoCE 驱动 rdma_rxe 绑定到一块普通以太网接口。两台虚拟机是最容易复现的选择。Soft RoCE 的目标是验证软件栈和编程模型，不适合用来评价真实 RDMA 网卡的吞吐、时延、CPU 占用或零拷贝能力。

## 1 阅读指南

建议严格按以下顺序学习。每一步都有成功判据。某一步未通过时，不要跳到后面的代码实验。

1. 完成实验一到实验三，确认两台节点的 IP 网络和 RXE 设备正常。
2. 完成实验四和实验五，用现成工具验证 RDMA CM、Send Receive、RDMA Write 和 RDMA Read。
3. 完成实验六，抓取 RoCEv2 报文，把用户态操作与网络流量对应起来。
4. 完成实验七和实验八，编译并修改配套 C 程序。
5. 使用故障排查章节定位错误，并根据进阶路线迁移到真实网卡。

配套目录 `lab` 包含以下文件。

| 文件 | 用途 |
| --- | --- |
| `setup-rxe.sh` | 加载 rdma_rxe 并创建软件 RDMA 设备 |
| `teardown-rxe.sh` | 删除软件 RDMA 设备 |
| `verify-rdma.sh` | 汇总内核模块、设备节点和 RDMA 资源 |
| `src/verbs_probe.c` | 枚举设备并查询端口能力 |
| `src/rdma_cm_echo.c` | 最小的 RC Send Receive 回显程序 |
| `Makefile` | 编译两个 C 程序 |

## 2 环境设计

### 2.1 推荐拓扑

准备两台 Linux 节点。可以是两台物理机、两台虚拟机，或一台物理机上的两台虚拟机。两台节点应位于同一可互通的 IPv4 网络。

| 项目 | 节点 A | 节点 B |
| --- | --- | --- |
| 角色 | 服务端 | 客户端 |
| 主机名示例 | `rdma-a` | `rdma-b` |
| 实验 IP | `192.168.56.101/24` | `192.168.56.102/24` |
| 普通网卡示例 | `enp0s8` | `enp0s8` |
| RDMA 设备名 | `rxe0` | `rxe0` |
| 操作系统 | Ubuntu 24.04 LTS | Ubuntu 24.04 LTS |

虚拟化平台可使用 KVM、VMware、VirtualBox 或 Parallels。为两台虚拟机增加一块仅主机网络或内部网络网卡，并给它们配置上表中的静态 IP。NAT 网卡可继续用于访问软件仓库。RDMA 实验流量走实验网卡。

如果使用云主机，先确认云平台没有过滤 UDP 4791，并确认虚拟机允许加载 `rdma_rxe` 内核模块。普通无特权容器通常不能创建 RDMA 设备，因此不推荐作为第一次实验环境。

### 2.2 软件栈

应用程序通过两类用户态库工作。

- `librdmacm` 负责地址解析、路由解析、连接建立和断开。它的概念接近 socket，但连接最终落在 QP 上。
- `libibverbs` 负责创建 PD、MR、CQ、QP，并提交 Send、Receive、RDMA Read、RDMA Write 等工作请求。

内核中的 `ib_uverbs` 提供用户态 verbs 接口，`rdma_rxe` 把 RDMA 操作封装为 RoCEv2 报文并通过普通以太网接口发送。换成真实网卡时，应用层 API 基本不变，但数据路径会由硬件执行。

### 2.3 学习环境的边界

Soft RoCE 可以验证连接状态机、内存注册、队列深度、完成队列、访问权限和错误处理。它不能复现硬件网卡的 DMA、PCIe、NUMA、PFC、ECN、硬件时间戳或拥塞控制行为。文档中的性能数字只用于比较参数变化，不应与产品规格或生产容量直接比较。

## 3 实验一 检查网络与系统

### 3.1 确认内核和架构

在两个节点执行：

```bash
uname -a
uname -m
cat /etc/os-release
```

成功判据是系统为 Ubuntu 24.04 LTS，内核能够正常启动。x86_64 和 arm64 均可使用发行版提供的 rdma-core 软件包。宿主机的 CPU 架构与虚拟机架构可以不同，但两台节点最好使用相同的软件版本。

### 3.2 找到实验网卡

```bash
ip -brief link
ip -brief address
ip route
```

不要直接复制示例中的 `enp0s8`。先找出持有 `192.168.56.0/24` 地址的真实接口名，然后把后续命令中的 `IFACE` 替换为该名称。

如果尚未配置地址，可临时执行以下命令。重启后会失效。

节点 A：

```bash
sudo ip link set enp0s8 up
sudo ip address add 192.168.56.101/24 dev enp0s8
```

节点 B：

```bash
sudo ip link set enp0s8 up
sudo ip address add 192.168.56.102/24 dev enp0s8
```

### 3.3 验证双向连通

节点 A：

```bash
ping -c 3 192.168.56.102
```

节点 B：

```bash
ping -c 3 192.168.56.101
```

两端都应收到回复，并且丢包率为 0。若失败，先检查虚拟交换机、IP 掩码、路由和主机防火墙。普通 IP 网络未连通时，RDMA CM 也无法完成地址和路由解析。

## 4 实验二 安装开发工具

### 4.1 安装软件包

在两个节点执行：

```bash
sudo apt update
sudo apt install -y \
  build-essential pkg-config iproute2 ethtool tcpdump \
  rdma-core ibverbs-providers ibverbs-utils \
  libibverbs-dev librdmacm-dev rdmacm-utils perftest
```

如果 `perftest` 无法找到，启用 Ubuntu Universe 仓库后重试：

```bash
sudo add-apt-repository universe
sudo apt update
sudo apt install -y perftest
```

| 软件包 | 主要内容 |
| --- | --- |
| `rdma-core` | RDMA 用户态基础组件和配置 |
| `ibverbs-providers` | 用户态设备 provider 包括 RXE provider |
| `ibverbs-utils` | `ibv_devices` `ibv_devinfo` `ibv_rc_pingpong` |
| `rdmacm-utils` | `rping` `ucmatose` 等 RDMA CM 示例 |
| `libibverbs-dev` | verbs 头文件和链接库 |
| `librdmacm-dev` | RDMA CM 头文件和链接库 |
| `perftest` | 带宽和时延测试程序 |

### 4.2 检查命令和头文件

```bash
command -v rdma ibv_devices ibv_devinfo rping ib_write_bw
pkg-config --modversion libibverbs librdmacm
test -f /usr/include/infiniband/verbs.h && echo verbs-header-ok
test -f /usr/include/rdma/rdma_cma.h && echo rdmacm-header-ok
```

所有命令应返回路径，两个头文件检查应输出 `ok`。发行版升级可能改变库版本，但不影响本教程使用的基础 API。

## 5 实验三 创建 Soft RoCE 设备

### 5.1 加载内核模块

在两个节点执行：

```bash
sudo modprobe rdma_rxe
lsmod | grep -E 'rdma_rxe|ib_uverbs|rdma_cm'
```

如果 `modprobe` 提示找不到模块，先检查内核配置：

```bash
grep CONFIG_RDMA_RXE /boot/config-$(uname -r)
```

期望值为 `CONFIG_RDMA_RXE=m` 或 `CONFIG_RDMA_RXE=y`。若当前云内核或裁剪内核未启用 RXE，需要换用 Ubuntu 通用内核或重新编译内核。

### 5.2 绑定普通网卡

以下命令假定实验接口为 `enp0s8`。两个节点都执行：

```bash
sudo rdma link add rxe0 type rxe netdev enp0s8
```

也可以使用配套脚本，它会检查接口、加载模块，并在设备已存在时保持幂等：

```bash
cd lab
chmod +x setup-rxe.sh teardown-rxe.sh verify-rdma.sh
./setup-rxe.sh enp0s8 rxe0
```

官方 rdma-core 使用相同的 `rdma link add NAME type rxe netdev DEVICE` 方式创建软件设备。旧资料中的 `rxe_cfg` 已不再是本教程的首选入口。

### 5.3 验证 RDMA 设备

```bash
rdma link show
ibv_devices
ibv_devinfo -d rxe0
ls -l /dev/infiniband
```

典型结果应满足以下条件。

- `rdma link show` 包含 `rxe0/1`，状态通常为 `ACTIVE`，并显示绑定的普通网卡。
- `ibv_devices` 能列出 `rxe0`。
- `ibv_devinfo` 的 `transport` 为 InfiniBand transport，但端口 `link_layer` 为 Ethernet。
- `/dev/infiniband` 中存在 `uverbs0` 和 `rdma_cm` 等设备节点。

运行配套检查脚本可一次收集这些信息：

```bash
./verify-rdma.sh
```

### 5.4 查看 GID

```bash
ibv_devinfo -d rxe0 -v | less
```

在端口信息中找到 GID 表。RXE 通常会为网卡 IP 生成 IPv4 映射 GID。后续如果工具不能自动选择正确 GID，可通过 `-g` 或 `-x` 显式指定索引。索引随地址和环境变化，不要把某个固定数字写进自动化脚本。

### 5.5 删除和重建

```bash
sudo rdma link delete rxe0
sudo rdma link add rxe0 type rxe netdev enp0s8
```

配套脚本的等价用法：

```bash
./teardown-rxe.sh rxe0
./setup-rxe.sh enp0s8 rxe0
```

成功删除设备不会删除普通网卡或其 IP 地址。它只移除绑定在该网卡上的 RXE RDMA 设备。

## 6 实验四 使用 RDMA CM 完成第一次传输

`rping` 使用 librdmacm 建立 RC 连接，并执行 RDMA 数据传输。它比直接操作 QP 更适合作为第一条端到端验证命令。

### 6.1 启动服务端

节点 A：

```bash
rping -s -a 192.168.56.101 -p 7471 -v -V
```

### 6.2 启动客户端

节点 B：

```bash
rping -c -a 192.168.56.101 -p 7471 -C 10 -v -V
```

客户端和服务端都应打印 10 次数据交换，并在校验开启时不报告数据错误。`-C 10` 让客户端在 10 次之后退出；不指定该参数时默认持续运行。

### 6.3 防火墙说明

RoCEv2 数据使用 UDP 4791。实验网卡受防火墙保护时，应只允许对端实验 IP 访问该端口。例如节点 A 使用 UFW 时：

```bash
sudo ufw allow in on enp0s8 from 192.168.56.102 to any port 4791 proto udp
```

节点 B 把源地址改为 `192.168.56.101`。`rping -p 7471` 中的端口是 RDMA CM 服务标识，不能简单等同于一个监听在 UDP 7471 上的普通 socket。抓包时应重点观察 UDP 4791。

### 6.4 观察资源

让 `rping` 保持运行，在任意节点打开另一个终端：

```bash
rdma resource show
rdma resource show qp
rdma resource show cq
rdma resource show mr
```

结束 `rping` 后再次执行。由进程创建的 QP、CQ 和 MR 应被释放。这个实验说明 RDMA 资源由内核按用户态上下文跟踪，进程退出时内核会清理残留资源。

## 7 实验五 验证 verbs 与性能工具

### 7.1 RC Send Receive 测试

节点 A：

```bash
ibv_rc_pingpong -d rxe0 -n 1000 -s 4096
```

节点 B：

```bash
ibv_rc_pingpong -d rxe0 -n 1000 -s 4096 192.168.56.101
```

工具使用 TCP 18515 交换连接参数，实际数据通过 RDMA QP 传输。如果地址解析不正确，先从 `ibv_devinfo -v` 找到对应 IPv4 地址的 GID 索引，然后在两端增加 `-g 索引`。防火墙还需允许对端访问 TCP 18515。

### 7.2 RDMA Write 带宽

节点 A：

```bash
ib_write_bw -d rxe0 -R -D 10 --report_gbits
```

节点 B：

```bash
ib_write_bw -d rxe0 -R -D 10 --report_gbits 192.168.56.101
```

两端参数必须一致。`-R` 让 perftest 通过 RDMA CM 创建 QP，`-D 10` 表示运行 10 秒。成功时输出应包含设备名、连接类型、链路类型、消息大小、平均带宽和消息速率。

### 7.3 RDMA Read 时延

节点 A：

```bash
ib_read_lat -d rxe0 -R -s 8 -n 5000
```

节点 B：

```bash
ib_read_lat -d rxe0 -R -s 8 -n 5000 192.168.56.101
```

这个实验让客户端从服务端暴露的远端内存读取 8 字节。程序需要先交换远端虚拟地址和 rkey，然后才能发起 RDMA Read。

### 7.4 参数实验

依次测试 64 B、4 KiB、64 KiB 和 1 MiB：

```bash
for size in 64 4096 65536 1048576; do
  echo "size=$size"
  ib_write_bw -d rxe0 -R -D 5 -s "$size" --report_gbits 192.168.56.101
done
```

服务端需针对每个大小运行相同参数，或分别执行四次。记录平均带宽、消息速率和 CPU 占用。小消息通常受每秒操作次数和软件处理开销限制，大消息更容易接近底层普通网卡的可用吞吐。这里观察的是 RXE 软件路径，不是硬件 RDMA 的性能。

### 7.5 建议的实验记录

| 测试 | 大小 | 迭代或时长 | 平均值 | CPU | 备注 |
| --- | --- | --- | --- | --- | --- |
| `ib_write_bw` | 64 B | 5 s | 记录 | 记录 | 小消息 |
| `ib_write_bw` | 4 KiB | 5 s | 记录 | 记录 | 常用页大小 |
| `ib_write_bw` | 64 KiB | 5 s | 记录 | 记录 | 中等消息 |
| `ib_write_bw` | 1 MiB | 5 s | 记录 | 记录 | 大消息 |
| `ib_read_lat` | 8 B | 5000 | 记录 | 记录 | 单向读取 |

## 8 实验六 抓取并识别 RoCEv2 报文

### 8.1 抓包

节点 A 在测试前启动：

```bash
sudo tcpdump -ni enp0s8 udp port 4791 -vv
```

另一个终端运行 `rping` 服务端，节点 B 运行客户端。应看到源和目的 IP 之间的 UDP 4791 流量。

保存为 pcap：

```bash
sudo tcpdump -ni enp0s8 udp port 4791 -w rocev2.pcap
```

完成一次短测试后按 Ctrl C 停止。可把 pcap 复制到安装 Wireshark 的桌面环境，使用过滤器 `udp.port == 4791` 查看。

### 8.2 对照用户态动作

按时间观察以下阶段。

1. RDMA CM 完成地址和路由解析，并建立连接。
2. 接收方预先提交 Receive WR。
3. 发送方提交 Send WR，RXE 生成 RoCEv2 报文。
4. 对端处理数据并在 CQ 中产生完成项。
5. 应用通过 `ibv_poll_cq` 或完成事件发现 WC。

抓包只能看到网络报文，不能直接看到 PD、MR、CQ 或 QP 对象。使用 `rdma resource show` 补充观察内核资源。

## 9 RDMA 编程模型

### 9.1 核心对象

| 对象 | 全称 | 作用 | 常见错误 |
| --- | --- | --- | --- |
| Context | Device Context | 打开一个 RDMA 设备 | 设备名错误或权限不足 |
| PD | Protection Domain | 隔离一组 QP 和 MR | QP 与 MR 不属于同一 PD |
| MR | Memory Region | 注册可被 RDMA 访问的内存 | access flags 或 memlock 不足 |
| CQ | Completion Queue | 保存完成项 WC | 不轮询导致队列积压 |
| QP | Queue Pair | 包含发送队列和接收队列 | 状态转换或对端信息错误 |
| WR | Work Request | 描述一次发送、接收或读写 | SGE 地址、长度或 lkey 错误 |
| WC | Work Completion | 描述 WR 的完成结果 | 忽略 status 和 vendor_err |
| CM ID | Connection Manager ID | 表示地址、路由和连接状态 | 忘记确认 CM event |

### 9.2 Send Receive 与单边操作

Send Receive 是双边操作。接收方必须提前发布 Receive WR，发送方才能安全发送。它适合交换控制消息、元数据和小型通知。

RDMA Write 和 RDMA Read 是单边操作。发起方使用对端提供的虚拟地址和 rkey，直接访问对端已注册的 MR。对端 CPU 不需要为每次数据移动发布匹配的 Receive WR，但应用仍需设计权限交换、生命周期和完成通知。

### 9.3 典型调用顺序

客户端连接路径：

```text
rdma_create_event_channel
rdma_create_id
rdma_resolve_addr
RDMA_CM_EVENT_ADDR_RESOLVED
rdma_resolve_route
RDMA_CM_EVENT_ROUTE_RESOLVED
创建 PD CQ QP 和 MR
预先提交 Receive WR
rdma_connect
RDMA_CM_EVENT_ESTABLISHED
提交 Send 或 RDMA WR
轮询 CQ 并检查 WC
rdma_disconnect
销毁资源
```

服务端路径：

```text
rdma_create_event_channel
rdma_create_id
rdma_bind_addr
rdma_listen
RDMA_CM_EVENT_CONNECT_REQUEST
为新连接创建 PD CQ QP 和 MR
预先提交 Receive WR
rdma_accept
RDMA_CM_EVENT_ESTABLISHED
处理 WR 和 WC
RDMA_CM_EVENT_DISCONNECTED
销毁连接和监听资源
```

## 10 实验七 编译设备探测程序

### 10.1 编译全部示例

把配套 `lab` 目录复制到两个节点，然后执行：

```bash
cd lab
make clean
make
```

成功后生成 `verbs_probe` 和 `rdma_cm_echo`。Makefile 的核心链接方式如下：

```make
cc -O2 -g -Wall -Wextra -Wpedantic \
  -o verbs_probe src/verbs_probe.c -libverbs

cc -O2 -g -Wall -Wextra -Wpedantic \
  -o rdma_cm_echo src/rdma_cm_echo.c -lrdmacm -libverbs
```

### 10.2 运行设备探测

```bash
./verbs_probe
```

预期至少发现一个设备，并显示类似信息：

```text
found 1 RDMA device(s)

device 0: rxe0
  firmware: 0.0.0
  max_qp: ...
  max_cq: ...
  max_mr_size: ...
  port 1: state=PORT_ACTIVE link_layer=Ethernet active_mtu=...
```

程序展示了最短的 verbs 设备发现路径：调用 `ibv_get_device_list`，通过 `ibv_open_device` 打开设备，再使用 `ibv_query_device` 和 `ibv_query_port` 查询能力。它没有创建 QP，也不产生网络流量。

### 10.3 阅读代码的顺序

1. 找到 `ibv_get_device_list`，确认设备列表必须由 `ibv_free_device_list` 释放。
2. 找到 `ibv_open_device` 和 `ibv_close_device`，确认 context 的生命周期。
3. 查看 `phys_port_cnt`，理解端口编号从 1 开始。
4. 查看 `link_layer`，确认 RXE 的链路层为 Ethernet。

## 11 实验八 运行并修改 RDMA CM 回显程序

### 11.1 运行服务端

节点 A：

```bash
cd lab
./rdma_cm_echo server 192.168.56.101 7472
```

预期输出：

```text
server listening on 192.168.56.101:7472
```

### 11.2 运行客户端

节点 B：

```bash
cd lab
./rdma_cm_echo client 192.168.56.101 7472
```

客户端预期输出：

```text
client received: hello from RDMA server
```

服务端预期随后输出：

```text
server received: hello from RDMA client
```

如果连接超时，先重新运行 `rping`。若 `rping` 也失败，问题在环境或网络，而不是示例程序。

### 11.3 代码中的关键设计

程序为发送和接收分别分配并注册缓冲区。这样避免同一块内存在发送尚未完成时被对端回复覆盖。接收 WR 在连接建立前发布，防止对端建立连接后立即发送而本端没有接收缓冲区。

每个 Send WR 都设置 `IBV_SEND_SIGNALED`，因此 CQ 会产生发送完成项。示例使用 `ibv_poll_cq` 忙轮询，并设置 10 秒超时。生产程序通常会批量提交 WR、减少完成通知、使用完成事件或自适应轮询，并为每个连接维护更完整的状态机。

程序使用 Send Receive，不交换 rkey，因此还不是单边 RDMA Write 示例。先掌握连接、MR 和 CQ，再扩展到单边操作更容易排错。

### 11.4 动手修改

完成以下练习，每次修改后都重新编译并同时观察 `rdma resource show` 与抓包结果。

1. 把 `BUFFER_SIZE` 从 256 改为 4096，并在消息中发送更长字符串。
2. 把客户端消息改为命令行参数，验证长度边界。
3. 连续发送 1000 条消息。接收方每消费一个 WC 后立即补发 Receive WR。
4. 把忙轮询改为 completion channel 和 `ibv_get_cq_event`。
5. 为每个 WR 使用唯一的 `wr_id`，在 WC 中恢复请求上下文。
6. 增加消息序号和校验值，检测丢失、乱序或缓冲区复用错误。

### 11.5 扩展为 RDMA Write

单边 Write 至少需要增加以下协议。

1. 服务端注册目标 MR，access flags 包含 `IBV_ACCESS_REMOTE_WRITE`。
2. 服务端通过 Send Receive 把目标地址、长度和 rkey 发给客户端。
3. 客户端构造 `IBV_WR_RDMA_WRITE`，填写 `wr.rdma.remote_addr` 和 `wr.rdma.rkey`。
4. 客户端等待本地 WC，并通过额外的 Send 或 Write with Immediate 通知服务端数据已经可用。
5. 服务端在撤销 MR 或释放内存前，确认对端不会再使用旧 rkey。

这个扩展的重点不是复制几行代码，而是正确处理 rkey 权限、端序、资源生命周期和完成通知。

## 12 调试方法

### 12.1 分层排查顺序

按以下顺序排查，可以避免在应用代码中寻找网络问题。

1. `ip -brief address` 和双向 `ping`。
2. `lsmod` 确认 `rdma_rxe`，`rdma link show` 确认 ACTIVE。
3. `ibv_devices` 和 `ibv_devinfo -d rxe0`。
4. `rping`。
5. `ibv_rc_pingpong` 或 `perftest`。
6. 自己编写的程序。

### 12.2 常见问题矩阵

| 现象 | 最可能原因 | 检查 | 处理 |
| --- | --- | --- | --- |
| `No IB devices found` | RXE 未创建或 provider 缺失 | `rdma link show` `dpkg -l ibverbs-providers` | 重建 RXE 并安装 provider |
| `rdma link add` 报设备已存在 | 同名 RXE 已创建 | `rdma link show` | 复用现有设备或先删除 |
| `RDMA_CM_EVENT_ADDR_ERROR` | IP 路由或 GID 不匹配 | `ip route get 对端IP` `ibv_devinfo -v` | 修正接口地址和路由 |
| `RDMA_CM_EVENT_ROUTE_ERROR` | 对端不可达或防火墙过滤 | 双向 ping 和 `tcpdump` | 放行 UDP 4791 |
| `Cannot allocate memory` | memlock 限制过低 | `ulimit -l` | 减小 MR 或调整 limits |
| `IBV_WC_RNR_RETRY_EXC_ERR` | 对端未提前发布 Receive | 检查 `ibv_post_recv` 顺序 | 连接前发布并持续补充 Receive |
| `IBV_WC_REM_ACCESS_ERR` | rkey 地址或远端权限错误 | 检查 MR access flags 和交换值 | 重新注册并控制生命周期 |
| perftest 两端参数不一致 | 大小 模式或 QP 参数不同 | 比较完整命令 | 两端使用相同选项 |
| TCP 18515 连接失败 | OOB 通道被过滤 | `ss -lntp` 防火墙规则 | 放行对端 TCP 18515 |
| Soft RoCE 性能很低 | 软件处理和虚拟化开销 | `top` `mpstat` `ethtool -k` | 仅作功能学习 不作硬件结论 |

### 12.3 检查日志和计数器

```bash
dmesg --level=err,warn | tail -n 100
journalctl -k -b | grep -Ei 'rdma|rxe|infiniband'
rdma statistic show
ip -s link show dev enp0s8
```

如果出现工作完成错误，始终打印 `ibv_wc_status_str(wc.status)` 和 `vendor_err`。只输出“传输失败”会丢失最有价值的定位信息。

### 12.4 memlock

注册内存可能受到 `RLIMIT_MEMLOCK` 限制：

```bash
ulimit -l
```

本教程的小缓冲区通常不会触发限制。大规模测试若失败，可以先临时在当前 shell 中提高限制，前提是系统策略允许：

```bash
ulimit -l unlimited
```

长期配置应由管理员根据用户和服务范围写入 `/etc/security/limits.d` 或 systemd unit 的 `LimitMEMLOCK`。不要为了省事给所有用户无条件放开无限锁页内存。

### 12.5 卸载和恢复

删除 RXE：

```bash
sudo rdma link delete rxe0
```

确认没有 RXE 设备和使用者后，可卸载模块：

```bash
sudo modprobe -r rdma_rxe
```

普通网卡与 IP 配置保持不变。若模块忙，使用 `rdma resource show` 和 `lsof /dev/infiniband/*` 查找仍在运行的进程。

## 13 自动化与持续验证

### 13.1 每次启动后的检查

建议把以下检查放进实验记录或 CI 日志，而不是只依赖一次人工观察。

```bash
rdma link show
ibv_devices
ibv_devinfo -d rxe0
rping -c -a 192.168.56.101 -p 7471 -C 3 -V
```

CI 中需要提前启动服务端，并为每个命令设置超时。任何失败都保存 `dmesg`、`rdma resource show` 和 `ip -details address`。

### 13.2 持久化 RXE

学习阶段建议手动创建，便于理解和清理。确认接口名稳定后，可建立 systemd oneshot unit，在网络上线后执行：

```ini
[Unit]
Description=Create Soft RoCE device rxe0
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/sbin/rdma link add rxe0 type rxe netdev enp0s8
ExecStop=/usr/sbin/rdma link delete rxe0

[Install]
WantedBy=multi-user.target
```

先用 `command -v rdma` 确认绝对路径，并把 `enp0s8` 改为真实接口。启用前必须删除手工创建的同名设备，否则 ExecStart 会因为设备已存在而失败。生产环境还应增加幂等包装、接口存在性检查和日志。

## 14 迁移到真实 RDMA 网卡

### 14.1 不变的部分

`librdmacm` 和 `libibverbs` 的核心对象、连接流程、MR 访问控制、WR 和 WC 处理方式保持不变。配套程序通常只需选择实际设备和地址即可运行。

### 14.2 新增检查

| 范围 | 真实环境需要增加的检查 |
| --- | --- |
| 固件与驱动 | 网卡固件 驱动和 rdma-core 兼容性 |
| 交换网络 | VLAN MTU PFC ECN DSCP 和无损队列规划 |
| 地址 | RoCE GID 类型 GID index 和路由一致性 |
| 性能 | NUMA 绑定 CPU 亲和 中断 PCIe 带宽和内存带宽 |
| 可靠性 | 丢包 拥塞 恢复 超时 重试和队列耗尽 |
| 安全 | rkey 生命周期 租户隔离 防火墙和资源限额 |

在真实 RoCE 网络中，不应在不理解影响的情况下直接打开全局 PFC。先确认厂商设计、交换机缓冲和 ECN 配置，再用分层测试验证。InfiniBand 环境还需要 Subnet Manager，地址和链路诊断工具也与以太网 RoCE 不同。

### 14.3 性能测试纪律

1. 两端使用相同版本的 perftest 和相同参数。
2. 同时记录消息大小、队列深度、QP 数、MTU、双向模式和测试时长。
3. 记录 CPU、NUMA、频率调节和虚拟化条件。
4. 先测单流，再测多 QP 和双向流量。
5. 把功能正确性、峰值吞吐、尾时延和故障恢复分别测试。

## 15 学习路线

### 第一阶段 设备与连接

完成本教程实验一到实验四，能够从 `rdma link`、`ibv_devinfo` 和 `rping` 输出判断链路状态，理解 CM 事件顺序。

### 第二阶段 verbs 数据路径

阅读 `verbs_probe.c` 和 `rdma_cm_echo.c`，能够解释每个资源的创建与销毁顺序，能够从 WC status 定位错误。

### 第三阶段 单边读写

实现 MR 元数据交换、RDMA Write、RDMA Read 和 Write with Immediate。为 rkey 设计短生命周期，不允许对端在 MR 释放后继续访问。

### 第四阶段 并发与性能

实现多个 outstanding WR、批量 doorbell、CQ moderation、多 QP 和多线程。测量吞吐、平均时延、P99 和 CPU 使用率，并区分轮询与事件模式。

### 第五阶段 生产化

加入断线重连、超时、资源上限、协议版本、端序、指标、日志、模糊测试和故障注入。最后迁移到真实网卡和受控交换网络。

## 16 完成检查表

- [ ] 两台节点可以双向 ping。
- [ ] 两台节点均能看到 ACTIVE 的 `rxe0`。
- [ ] `ibv_devices` 和 `ibv_devinfo` 能识别设备。
- [ ] `rping` 完成 10 次校验传输。
- [ ] `ibv_rc_pingpong` 完成 1000 次交换。
- [ ] `ib_write_bw` 和 `ib_read_lat` 输出有效结果。
- [ ] tcpdump 能捕获 UDP 4791 报文。
- [ ] `verbs_probe` 能打印设备和端口能力。
- [ ] `rdma_cm_echo` 能完成双向消息。
- [ ] 能解释 PD MR CQ QP WR WC 和 CM ID。
- [ ] 能按分层顺序定位一个故意制造的错误。
- [ ] 能说明 Soft RoCE 结果为什么不能代表硬件 RDMA 性能。

## 17 参考资料

1. linux-rdma rdma-core 项目。软件 RDMA 的创建命令和用户态库源码。https://github.com/linux-rdma/rdma-core
2. Linux rdma-link 手册。`rdma link add NAME type rxe netdev NETDEV` 的语法和示例。https://man7.org/linux/man-pages/man8/rdma-link.8.html
3. Linux 内核文档 Userspace verbs access。用户态 verbs、设备节点、内存锁页和内核资源管理。https://docs.kernel.org/infiniband/user_verbs.html
4. librdmacm rdma_cm 手册。连接管理 API、客户端和服务端事件顺序。https://man7.org/linux/man-pages/man7/rdma_cm.7.html
5. Ubuntu 24.04 rping 手册。命令行选项和数据校验方式。https://manpages.ubuntu.com/manpages/noble/man1/rping.1.html
6. linux-rdma perftest 项目。测试类型、服务端客户端命令结构和参数要求。https://github.com/linux-rdma/perftest
7. libibverbs ibv_rc_pingpong 手册和示例源码。https://github.com/linux-rdma/rdma-core/blob/master/libibverbs/man/ibv_rc_pingpong.1

参考资料用于核对接口和命令。发行版包版本、内核模块和硬件能力可能随系统更新而变化，实际部署应同时查阅当前发行版和网卡厂商文档。
