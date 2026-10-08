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
# rping -s -a 192.168.180.131 -p 7471 -v -V
```
### 1.2 启动客户端

在主机`192.168.180.132`上启动客户端:

```
# rping -c -a 192.168.180.131 -p 7471 -C 10 -v -V
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
# rdma resource show
0: rxe0: pd 2 cq 1 qp 1 cm_id 1 mr 0 ctx 1 srq 0 

# rdma resource show qp
link rxe0/1 lqpn 1 type GSI state RTS sq-psn 2 comm [ib_core] 

# rdma resource show cq
dev rxe0 cqn 0 cqe 1023 users 2 poll-ctx WORKQUEUE adaptive-moderation off comm [ib_core] 

# rdma resource show mr
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
# ibv_rc_pingpong -d rxe0 -g 1 -n 1000 -s 4096 192.168.180.131
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
