# RDMA相关实验

本文接着“软件RDMA的部署”一文中的环境来做相关的实验。

1）**实验拓扑**

| 项目 | 节点 A | 节点 B |
| --- | --- | --- |
| 角色 | 服务端 | 客户端 |
| 主机名示例 | `rdma-a` | `rdma-b` |
| 实验 IP | `192.168.180.131/24` | `192.168.180.132/24` |
| 普通网卡示例 | `ens33` | `ens33` |
| RDMA 设备名 | `rxe0` | `rxe0` |
| 操作系统 | Ubuntu 22.04 LTS | Ubuntu 22.04 LTS |

2）**软件栈**

应用程序通过两类用户态库工作:
- librdmacm 负责地址解析、路由解析、连接建立和断开。它的概念接近 socket，但连接最终落在 QP 上;
- libibverbs 负责创建 PD、MR、CQ、QP，并提交 Send、Receive、RDMA Read、RDMA Write 等工作请求;
  
内核中的 ib_uverbs 提供用户态 verbs 接口，rdma_rxe 把 RDMA 操作封装为 RoCEv2 报文并通过普通以太网接口发送。换成真实网卡时，应用层 API 基本不变，但数据路径会由硬件执行。

>说明：关于PD、MR、CQ、QP等的说明，请参考《2.2-RDMA相关实验.md》

---

## 1. 实验一 使用 RDMA CM 完成第一次传输
`rping` 使用 librdmacm 建立 RC 连接，并执行 RDMA 数据传输。它比直接操作 QP 更适合作为第一条端到端验证命令.

### 1.1 启动服务端
在主机`192.168.180.131`启动服务端：
```bash
$ rping -s -a 192.168.180.131 -p 7471 -v -V
```
### 1.2 启动客户端

在主机`192.168.180.132`上启动客户端:

```
$ rping -c -a 192.168.180.131 -p 7471 -C 10 -v -V
ping data: rdma-ping-0: ABCDEFGHIJKLMNOPQRSTUVWXYZ[\]^_`abcdefghijklmnopqr
ping data: rdma-ping-1: BCDEFGHIJKLMNOPQRSTUVWXYZ[\]^_`abcdefghijklmnopqrs
ping data: rdma-ping-2: CDEFGHIJKLMNOPQRSTUVWXYZ[\]^_`abcdefghijklmnopqrst
ping data: rdma-ping-3: DEFGHIJKLMNOPQRSTUVWXYZ[\]^_`abcdefghijklmnopqrstu
ping data: rdma-ping-4: EFGHIJKLMNOPQRSTUVWXYZ[\]^_`abcdefghijklmnopqrstuv
ping data: rdma-ping-5: FGHIJKLMNOPQRSTUVWXYZ[\]^_`abcdefghijklmnopqrstuvw
ping data: rdma-ping-6: GHIJKLMNOPQRSTUVWXYZ[\]^_`abcdefghijklmnopqrstuvwx
ping data: rdma-ping-7: HIJKLMNOPQRSTUVWXYZ[\]^_`abcdefghijklmnopqrstuvwxy
ping data: rdma-ping-8: IJKLMNOPQRSTUVWXYZ[\]^_`abcdefghijklmnopqrstuvwxyz
ping data: rdma-ping-9: JKLMNOPQRSTUVWXYZ[\]^_`abcdefghijklmnopqrstuvwxyzA
client DISCONNECT EVENT...
```

客户端和服务端都应打印 10 次数据交换，并在校验开启时不报告数据错误。-C 10 让客户端在 10 次之后退出；不指定该参数时默认持续运行。

### 1.3 防火墙说明
RoCEv2 数据使用 UDP 4791。实验网卡受防火墙保护时，应只允许对端实验 IP 访问该端口。例如节点 A 使用 UFW 时：
```bash
sudo ufw allow in on ens33 from 192.168.180.132 to any port 4791 proto udp
```
节点 B 把源地址改为 192.168.180.131。`rping -p 7471` 中的端口是 RDMA CM 服务标识，不能简单等同于一个监听在 UDP 7471 上的普通 socket。抓包时应重点观察 UDP 4791。

### 1.4 观察资源
让 rping 保持运行，在任意节点打开另一个终端：
```bash
$ rdma resource show
0: rxe0: pd 2 cq 1 qp 1 cm_id 1 mr 0 ctx 1 srq 0 

$ rdma resource show qp
link rxe0/1 lqpn 1 type GSI state RTS sq-psn 2 comm [ib_core] 

$ rdma resource show cq
dev rxe0 cqn 0 cqe 1023 users 2 poll-ctx WORKQUEUE adaptive-moderation off comm [ib_core] 

$ rdma resource show mr
```
结束 rping 后再次执行。由进程创建的 QP、CQ 和 MR 应被释放。这个实验说明 RDMA 资源由内核按用户态上下文跟踪，进程退出时内核会清理残留资源。

---

## 2. 实验二 验证 verbs 与性能工具

### 2.1 RC Send Receive 测试

节点A(192.168.180.131):
```bash
# ibv_rc_pingpong -d rxe0 -g 1 -n 1000 -s 4096
```

节点B(192.168.180.132):
```bash
$ ibv_rc_pingpong -d rxe0 -g 1 -n 1000 -s 4096 192.168.180.131
  local address:  LID 0x0000, QPN 0x000014, PSN 0xbfc017, GID ::ffff:192.168.180.132
  remote address: LID 0x0000, QPN 0x000014, PSN 0x3df778, GID ::ffff:192.168.180.131
8192000 bytes in 1.30 seconds = 50.38 Mbit/sec
1000 iters in 1.30 seconds = 1300.73 usec/iter
```
工具使用 TCP 18515 交换连接参数，实际数据通过 RDMA QP 传输。如果地址解析不正确，先从 ibv_devinfo -v 找到对应 IPv4 地址的 GID 索引，然后在两端增加 `-g` 索引。防火墙还需允许对端访问 TCP 18515。

>说明：实际运行时当未使用`-g`指定索引时，客户端报告如下错误
>
> $ ibv_rc_pingpong -d rxe0 -n 1000 -s 4096 192.168.180.131
>
>  local address:  LID 0x0000, QPN 0x000013, PSN 0x44aac0, GID ::
>
> client read/write: No space left on device
>
> Couldn't read/write remote address


### 2.2 RDMA Write 带宽

- **节点A(192.168.180.131)**
  ```bash
  $ ib_write_bw -d rxe0 -R -D 10 --report_gbits
  
  ************************************
  * Waiting for client to connect... *
  ************************************
  ```

- **节点B(192.168.180.132)**
  ```bash
  $ ib_write_bw -d rxe0 -R -D 10 --report_gbits 192.168.180.131
  ---------------------------------------------------------------------------------------
                      RDMA_Write BW Test
   Dual-port       : OFF          Device         : rxe0
   Number of qps   : 1            Transport type : IB
   Connection type : RC           Using SRQ      : OFF
   PCIe relax order: ON
   ibv_wr* API     : OFF
   TX depth        : 128
   CQ Moderation   : 1
   Mtu             : 1024[B]
   Link type       : Ethernet
   GID index       : 1
   Max inline data : 0[B]
   rdma_cm QPs     : ON
   Data ex. method : rdma_cm
  ---------------------------------------------------------------------------------------
   local address: LID 0000 QPN 0x0016 PSN 0xa3d5f5
   GID: 00:00:00:00:00:00:00:00:00:00:255:255:192:168:180:132
   remote address: LID 0000 QPN 0x0016 PSN 0x52f407
   GID: 00:00:00:00:00:00:00:00:00:00:255:255:192:168:180:131
  ---------------------------------------------------------------------------------------
   #bytes     #iterations    BW peak[Gb/sec]    BW average[Gb/sec]   MsgRate[Mpps]
   65536      4819             0.00               0.42               0.000804
  ---------------------------------------------------------------------------------------
  ```

>说明：两端参数必须一致。`-R` 让 perftest 通过 RDMA CM 创建 QP，`-D 10` 表示运行 10 秒。成功时输出应包含设备名、连接类型、链路类型、消息大小、平均带宽和消息速率。

### 2.3 RDMA Read 时延

- **节点A(192.168.180.131)**
  ```bash
  $ ib_read_lat -d rxe0 -R -s 8 -n 5000
  ---------------------------------------------------------------------------------------
  Device not recognized to implement inline feature. Disabling it
  
  ************************************
  * Waiting for client to connect... *
  ************************************
  ```

- **节点B(192.168.180.132)**
  ```bash
  $ ib_read_lat -d rxe0 -R -s 8 -n 5000 192.168.180.131
  ---------------------------------------------------------------------------------------
  Device not recognized to implement inline feature. Disabling it
  ---------------------------------------------------------------------------------------
                      RDMA_Read Latency Test
   Dual-port       : OFF          Device         : rxe0
   Number of qps   : 1            Transport type : IB
   Connection type : RC           Using SRQ      : OFF
   PCIe relax order: ON
   ibv_wr* API     : OFF
   TX depth        : 1
   Mtu             : 1024[B]
   Link type       : Ethernet
   GID index       : 1
   Outstand reads  : 128
   rdma_cm QPs     : ON
   Data ex. method : rdma_cm
  ---------------------------------------------------------------------------------------
   local address: LID 0000 QPN 0x0018 PSN 0x4f70ec
   GID: 00:00:00:00:00:00:00:00:00:00:255:255:192:168:180:132
   remote address: LID 0000 QPN 0x0018 PSN 0x73c994
   GID: 00:00:00:00:00:00:00:00:00:00:255:255:192:168:180:131
  ---------------------------------------------------------------------------------------
   #bytes #iterations    t_min[usec]    t_max[usec]  t_typical[usec]    t_avg[usec]    t_stdev[usec]   99% percentile[usec]   99.9% percentile[usec] 
   8       5000          95.60          7993.00      213.33              490.49           1052.79         5958.70                 7579.10
  ---------------------------------------------------------------------------------------
  ```

这个实验让客户端从服务端暴露的远端内存读取 8 字节。程序需要先交换远端虚拟地址和 rkey，然后才能发起 RDMA Read。

### 2.4 参数实验

依次测试 64 B、4 KiB、64 KiB 和 1 MiB：
```bash
for size in 64 4096 65536 1048576; do
  echo "size=$size"
  ib_write_bw -d rxe0 -R -D 5 -s "$size" --report_gbits 192.168.56.101
done
```
服务端需针对每个大小运行相同参数，或分别执行四次。记录平均带宽、消息速率和 CPU 占用。小消息通常受每秒操作次数和软件处理开销限制，大消息更容易接近底层普通网卡的可用吞吐。这里观察的是 RXE 软件路径，不是硬件 RDMA 的性能。

### 2.5 建议的实验记录

|     测试	|   大小	|迭代或时长	|平均值	|CPU	|备注|
| --------- | ------- | -------- | ------ | ----- | -------- |
|ib_write_bw|	64 B   |	5 s      |	记录	|记录	  |小消息    |
|ib_write_bw|	4 KiB  |	5 s      |	记录	|记录	  |常用页大小 |
|ib_write_bw|	64 KiB |	5 s	     |记录	  |记录	  |中等消息   |
|ib_write_bw|	1 MiB  |	5 s	     |记录	  |记录	  |大消息     |
|ib_read_lat|	8 B    |	5000	   |记录	  |记录	  |单向读取   |

---

## 3. 实验三 抓取并识别 RoCEv2 报文

### 3.1 抓包

1）**tcpdump直接抓包并输出**

  首先，`节点 A`(192.168.180.131) 在测试前启动抓包命令：
  ```bash
  sudo tcpdump -ni ens33 udp port 4791 -vv
  ```
  
  接着，`节点A`(192.168.180.131)在另一个终端启动`rping`服务端:
  ```bash
  rping -s -a 192.168.180.131 -p 7471 -v -V
  ```

  之后，`节点B`(192.168.180.132)启动`rping`客户端:
  ```bash
  $ rping -c -a 192.168.180.131 -p 7471 -C 10 -v -V
  ping data: rdma-ping-0: ABCDEFGHIJKLMNOPQRSTUVWXYZ[\]^_`abcdefghijklmnopqr
  ping data: rdma-ping-1: BCDEFGHIJKLMNOPQRSTUVWXYZ[\]^_`abcdefghijklmnopqrs
  ping data: rdma-ping-2: CDEFGHIJKLMNOPQRSTUVWXYZ[\]^_`abcdefghijklmnopqrst
  ping data: rdma-ping-3: DEFGHIJKLMNOPQRSTUVWXYZ[\]^_`abcdefghijklmnopqrstu
  ping data: rdma-ping-4: EFGHIJKLMNOPQRSTUVWXYZ[\]^_`abcdefghijklmnopqrstuv
  ping data: rdma-ping-5: FGHIJKLMNOPQRSTUVWXYZ[\]^_`abcdefghijklmnopqrstuvw
  ping data: rdma-ping-6: GHIJKLMNOPQRSTUVWXYZ[\]^_`abcdefghijklmnopqrstuvwx
  ping data: rdma-ping-7: HIJKLMNOPQRSTUVWXYZ[\]^_`abcdefghijklmnopqrstuvwxy
  ping data: rdma-ping-8: IJKLMNOPQRSTUVWXYZ[\]^_`abcdefghijklmnopqrstuvwxyz
  ping data: rdma-ping-9: JKLMNOPQRSTUVWXYZ[\]^_`abcdefghijklmnopqrstuvwxyzA
  ```

  我们可以看到tcpdump的输出如下:
  ```text
  # sudo tcpdump -ni ens33 udp port 4791 -vv
  tcpdump: listening on ens33, link-type EN10MB (Ethernet), snapshot length 262144 bytes
  23:55:06.732122 IP (tos 0x0, ttl 64, id 35041, offset 0, flags [DF], proto UDP (17), length 308)
      192.168.180.132.55410 > 192.168.180.131.4791: [no cksum] UDP, length 280
  23:55:06.733211 IP (tos 0x0, ttl 64, id 29560, offset 0, flags [DF], proto UDP (17), length 308)
      192.168.180.131.55410 > 192.168.180.132.4791: [no cksum] UDP, length 280
  23:55:06.735869 IP (tos 0x0, ttl 64, id 35042, offset 0, flags [DF], proto UDP (17), length 308)
      192.168.180.132.55410 > 192.168.180.131.4791: [no cksum] UDP, length 280
  23:55:06.735871 IP (tos 0x0, ttl 64, id 35043, offset 0, flags [DF], proto UDP (17), length 60)
      192.168.180.132.61931 > 192.168.180.131.4791: [no cksum] UDP, length 32
  23:55:06.735937 IP (tos 0x0, ttl 64, id 29563, offset 0, flags [DF], proto UDP (17), length 48)
      192.168.180.131.61931 > 192.168.180.132.4791: [no cksum] UDP, length 20
  23:55:06.736150 IP (tos 0x0, ttl 64, id 29564, offset 0, flags [DF], proto UDP (17), length 60)
      192.168.180.131.61931 > 192.168.180.132.4791: [no cksum] UDP, length 32
  23:55:06.736524 IP (tos 0x0, ttl 64, id 35044, offset 0, flags [DF], proto UDP (17), length 112)
      192.168.180.132.61931 > 192.168.180.131.4791: [no cksum] UDP, length 84
  23:55:06.736641 IP (tos 0x0, ttl 64, id 29565, offset 0, flags [DF], proto UDP (17), length 60)
      192.168.180.131.61931 > 192.168.180.132.4791: [no cksum] UDP, length 32
  23:55:06.736868 IP (tos 0x0, ttl 64, id 35045, offset 0, flags [DF], proto UDP (17), length 48)
      192.168.180.132.61931 > 192.168.180.131.4791: [no cksum] UDP, length 20
  23:55:06.737051 IP (tos 0x0, ttl 64, id 35046, offset 0, flags [DF], proto UDP (17), length 60)
      192.168.180.132.61931 > 192.168.180.131.4791: [no cksum] UDP, length 32
  23:55:06.737072 IP (tos 0x0, ttl 64, id 29566, offset 0, flags [DF], proto UDP (17), length 48)
      192.168.180.131.61931 > 192.168.180.132.4791: [no cksum] UDP, length 20
  23:55:06.737187 IP (tos 0x0, ttl 64, id 29567, offset 0, flags [DF], proto UDP (17), length 124)
  ...
  ```

2）**tcpdump抓包并保存，使用wireshark进行分析**

  首先，`节点 A`(192.168.180.131) 在测试前启动抓包命令：
  ```bash
  sudo tcpdump -ni ens33 udp port 4791 -w rocev2.pcap
  ```
  接着分别在`节点A`启动rping服务端，`节点B`启动rping客户端。

  完成一次短测试后按 Ctrl C 停止。

  之后，使用wireshark进行分析:
  
  ![rdma-roce](https://raw.githubusercontent.com/ivanzz1001/storage-foundation/master/02_rdma/assets/rdma_roce_wireshark.jpg)

### 3.2 对照用户态动作
按时间观察以下阶段：

1. `RDMA CM`完成地址和路由解析，并建立连接。
1. 接收方预先提交`Receive WR`。
1. 发送方提交`Send WR`，`RXE` 生成 `RoCEv2` 报文。
1. 对端处理数据并在 `CQ` 中产生完成项。
1. 应用通过 `ibv_poll_cq` 或完成事件发现 `WC`。

抓包只能看到网络报文，不能直接看到 `PD`、`MR`、`CQ` 或 `QP` 对象。使用 `rdma resource show` 补充观察内核资源。

下面结合前面的抓包及wireshark截图来分析一下，从抓包可以确认：
- 192.168.180.132：rping client
- 192.168.180.131：rping server
- 使用 RoCEv2、RC（Reliable Connected）QP
- 握手完成后，每次 rping 循环都执行：`client公布源内存` → `server RDMA Read` → `server通知client` → `client公布目标内存` → `server RDMA Write回显` → `server通知完成`

这与`rdma-core`的`rping` 实现一致：客户端通过 SEND 发送 `远程地址 + rkey + 长度`，服务端读取客户端内存，再把数据写回客户端的另一个内存区


#### 3.2.1 RDMA CM 建连(1~3)

|包号	|    方向	    |  含义  |
| --- | ----------- | ------|
|1    |	.132 → .131	|CM: ConnectRequest。客户端请求建立 RDMA RC 连接，携带本端 QP、初始 PSN、重试参数、读资源能力、Service ID 等。|
|2    |	.131 → .132	|CM: ConnectReply。服务端接受连接并返回自己的 QP、PSN、Responder Resources 等参数。|
|3    |	.132 → .131	|CM: ReadyToUse。客户端确认双方参数已经准备好，QP 可以进入 RTS，开始传输。|


这相当于 `RDMA CM` 的 `REQ → REP → RTU`，不是 TCP 三次握手。

截图中的 `CM` 帧都是 322 字节；数据阶段则通过 UDP 4791 承载 RoCEv2。rping 默认应用端口通常是 7174，但它属于 CM Service ID 语义，不能把 Wireshark 中的 UDP 4791 当成 rping 应用监听端口。


#### 3.2.2 第一次 rping(4～15)
| 包号 | 方向 | Wireshark 类型 | rping 中的含义 |
|---|---|---|---|
| 4 | client → server | `RC Send Only` | 客户端发送源内存描述符：`start_buf虚拟地址 + rkey + size`。不是直接发送 ping 数据，载荷只有 16 字节。 |
| 5 | server → client | `RC Acknowledge` | RC 传输层确认已经可靠接收第 4 包。 |
| 6 | server → client | `RC RDMA Read Request` | 服务端使用第 4 包提供的地址和 rkey，发起单边 RDMA Read，读取客户端的 `start_buf`。 |
| 7 | client → server | `RC RDMA Read Response Only` | 客户端 RNIC 返回被读取的数据。截图中可见 `rdma-ping-0: ABCDEFGHI...`，共约 64 字节。 |
| 8 | server → client | `RC Send Only` | 服务端确认 RDMA Read 已完成，应用层含义是“go ahead，可以继续”。 |
| 9 | client → server | `RC Acknowledge` | 对第 8 包 SEND 的 RC 可靠确认。 |
| 10 | client → server | `RC Send Only` | 客户端发送用于接收回显的目标内存描述符：`rdma_buf地址 + rkey + size`。 |
| 11 | server → client | `RC Acknowledge` | 确认收到第 10 包的目标内存信息。 |
| 12 | server → client | `RC RDMA Write Only` | 服务端把刚才 Read 到的数据，单边写入客户端第 10 包指定的 `rdma_buf`。这就是 pong/echo。 |
| 13 | client → server | `RC Acknowledge` | 客户端 RNIC 确认 RDMA Write 已成功落入内存。 |
| 14 | server → client | `RC Send Only` | 服务端发送第二个“go ahead”，通知客户端：本轮 RDMA Write 已完成，可以校验数据或开始下一轮。 |
| 15 | client → server | `RC Acknowledge` | 对第 14 包的可靠确认。第一轮结束。 |

这里要区分两种“确认”：

- RC Acknowledge 是 RNIC/RC 传输层 ACK，由网卡协议栈产生。
- 第 8、14 包的 Send Only 是 rping 程序自己设计的应用层状态通知。

#### 3.2.3 第二次 rping(16～27)
流程与第一轮完全相同，只是测试数据变成下一组，例如 `rdma-ping-1: ...`

| 包号 | 含义 |
|---|---|
| 16 | client 发送第二轮源内存地址、rkey、长度 |
| 17 | server 对该 SEND 返回 RC ACK |
| 18 | server 发起第二轮 RDMA Read Request |
| 19 | client 返回 RDMA Read Response，携带第二轮 ping 数据 |
| 20 | server SEND“Read 完成，可以继续” |
| 21 | client 对该 SEND 返回 ACK |
| 22 | client 发送用于接收回显的目标内存地址、rkey、长度 |
| 23 | server ACK |
| 24 | server RDMA Write，把数据回写到 client 目标内存 |
| 25 | client ACK，表示 Write 已被可靠接收 |
| 26 | server SEND“Write 完成，本轮结束” |
| 27 | client ACK |

#### 3.2.4 第三次 rping(28～截图末尾)
| 包号 | 含义 |
|---|---|
| 28 | client 发送第三轮源内存描述符 |
| 29 | server ACK |
| 30 | server 发起 RDMA Read Request |
| 31 | client 返回 RDMA Read Response，携带第三轮数据 |
| 32 | server SEND“Read 完成”通知 |
| 33 | client ACK |
| 34 | 截图只显示了一部分；从方向和 74 字节长度判断，是 client 发送第三轮目标内存描述符。其后正常应出现 ACK、RDMA Write、ACK、完成通知。 |

#### 2.2.5 为什么长度正好是这些数值
这些长度也能验证上述判断：
- `RC Acknowledge = 62 B`
  
  Ethernet 14 + IPv4 20 + UDP 8 + BTH 12 + AETH 4 + ICRC 4
  
- `RC Send Only = 74 B`
  
  基础头部和 ICRC 58 + rping 内存描述符 16
  该描述符结构为：8 字节地址 + 4 字节 rkey + 4 字节长度。
  
- `RDMA Read Request = 74 B`
  
  基础 58 + RETH 16；RETH 携带远端地址、rkey 和读取长度。
  
- `RDMA Read Response Only = 126 B`
  
  Ethernet/IP/UDP/BTH/AETH + 64 字节数据 + ICRC。
  
- `RDMA Write Only = 138 B`
  
  Ethernet/IP/UDP/BTH + RETH 16 + 64 字节数据 + ICRC。
  
其中 `Only` 表示这个 `RDMA` 消息只有一个分片，即同时是 `First` 和 `Last`，并不是“只有该操作”或“不需要确认”。



--- 

## 参考文档

- ibta-specification： https://infinibandta.org/ibta-specification/

- RoCE v2 Considerations： https://enterprise-support.nvidia.com/s/article/roce-v2-considerations
