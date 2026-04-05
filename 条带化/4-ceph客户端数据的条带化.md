# ceph客户端数据的条带化

本文参考：

- [ceph客户端数据的条带化](https://ivanzz1001.github.io/records/post/ceph/2019/01/05/ceph-src-code-part5_1)

- [Ceph 分布式存储架构解析与工作原理](https://www.cnblogs.com/jmilkfan-fanguiju/p/11825073.html#_71)

- [Ceph 的物理和逻辑结构(Ceph Architecture)](https://www.cnblogs.com/sammyliu/p/4836014.html)

当用户使用RBD、RGW、CephFS类型客户端接口来存储数据时，会经历一个透明的、将数据转化为RADOS统一处理对象的过程，这个过程就称之为数据条带化或分片处理。

熟悉存储系统的你不会对条带化感到陌生，它是一种提升存储性能和吞吐能力的手段，通过将有序的数据分割成多个区段并分别存储到多个存储设备上，最常见的条带化就是RAID， 而Ceph的条带化处理就类似于RAID。如果想发挥ceph的并行IO处理能力，就应该充分利用客户端的条带化功能。需要注意的是，LibRados原生接口并不具有条带化功能，比如：使用LibRados接口上传1GB的文件，那么落到存储磁盘上的就是1GB大小的文件。存储集群中的Objects也同样不具备条带化功能，实际上是由上述三种类型的客户端将数据条带化之后再存储到集群Objects之中的。

## 1. 条带化处理过程

1）将数据切分为条带单元块；

2）将条带单元块写入到Object中，直到Object达到最大容量(默认是4MB)；

3）再为额外的条带单元块创建新的Object并继续写入数据；

4）循环上述步骤，直到数据完全写入

假设Object存储上限为4MB，每一个条带单元块占1MB。此时，我们存储一个8MB大小的文件，那么前4MB就存储在`Object0`中，后4MB就创建`Object1`来继续存储。

>ps: 上面“前4MB就存储在Object0中”是基于stripe_count为1的情况下来说的，参看下图的stripe_unit编号也可以看得出来这一点。

![ceph-stripe](https://raw.githubusercontent.com/ivanzz1001/storage-foundation/master/%E6%9D%A1%E5%B8%A6%E5%8C%96/image/ceph_stripe_1.jpg)
