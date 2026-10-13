# RDMA编程

## 1. RDMA 编程模型

### 1.1 核心对象

|对象	  |           全称	     |         作用	         |              常见错误       |
| ----- | -------------------- | --------------------- | ---------------------------- |
|Context|	Device Context       |打开一个 RDMA 设备      |	设备名错误或权限不足         |
|PD	    |Protection Domain     |隔离一组 QP 和 MR       |	QP 与 MR 不属于同一 PD       |
|MR	    |Memory Region	       |注册可被 RDMA 访问的内存 |	access flags 或 memlock 不足 |
|CQ	    |Completion Queue	     |保存完成项 WC	           |不轮询导致队列积压           |
|QP	    |Queue Pair	           |包含发送队列和接收队列    |	状态转换或对端信息错误     |
|WR	    |Work Request	         |描述一次发送、接收或读写  |	SGE 地址、长度或 lkey 错误 |
|WC	    |Work Completion	     |描述 WR 的完成结果        |	忽略 status 和 vendor_err  |
|CM ID	|Connection Manager ID |	表示地址、路由和连接状态 |	忘记确认 CM event          |

### 1.2 Send Receive 与单边操作

- `Send Receive` 是双边操作。接收方必须提前发布 Receive WR，发送方才能安全发送。它适合交换控制消息、元数据和小型通知。

- `RDMA Write` 和 `RDMA Read` 是单边操作。发起方使用对端提供的`虚拟地址`和`rkey`，直接访问对端已注册的`MR`。对端 CPU 不需要为每次数据移动发布匹配的`Receive WR`，但应用仍需设计权限交换、生命周期和完成通知。

### 1.3 典型调用顺序

**客户端连接路径**：
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

**服务端路径：**

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

