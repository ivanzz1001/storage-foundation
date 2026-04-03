# 同一文件的不同stripe对应的rados对象名称是否一样，如何进行区？


在Ceph中，每个RADOS对象的名称都是唯一的。Ceph通过一个分层且结构化的命名规则来区分它们，其核心是计算一个唯一的“对象编号”（objnum），然后将它拼接到一个“基础名”（Base Name）上

<br>

## 🎯 第一步：核心公式——计算“对象编号” (objnum)

`objnum`是一个从0开始的递增整数，其计算逻辑由Ceph文件布局（Layout）的四个关键参数决定:

- `stripe_unit`(条带单元): 4MB

- `stripe_count`(条带宽度): 4

- `object_size`(对象大小): 4MB(此例中等于`stripe_unit`)

- `file_offset`(文件内容偏移): 指待读写文件在原始文件中的起始位置

计算`objnum`的核心思路是：逻辑偏移量 → 逻辑条带单元（blockno）→ 逻辑对象号（objnum）。这里我们以`1GB`大小的文件为例，它会生成256个`objnum`从0递增到255的RADOS对象。具体计算步骤如下：

1. **计算逻辑条带单元号(blockno)**

    用文件内偏移除以`stripe_unit`。例如，文件开头(file_offset=0)的数据，对应的`blockno = 0 / 4MB = 0`

1. **将一维偏移量映射到三维坐标**

    - 条带号`(stripeno) = blockno / stripe_count`

    - 条带内位置`(stripeos) = blockno % stripe_count`

    - 对象集号`(objectsetno) = stripeno / (object_size / stripe_unit)`

    - 最终对象号`(objnum) = objectsetno * stripe_count + stripepos`

    >ps: 当`object_size`等于`stripe_unit`时，每个对象集只包含一条带。此时公式可简化为`objnum=blockno`。



<br>

## 🏷️ 第二步：拼接“基础名” (Base Name)

得到`objnum`后，Ceph会将其作为后缀，拼接到由上层应用确定的基础名（Base Name)后面。不同应用场景下，基础名的生成规则也不同：

1. **CephFS(文件存储)**

    基础名由文件的 inode号（**十六进制格式**） 构成。例如，inode号为`1099511627776`的文件，其基础名是`10000000000`，最终对象名形如`10000000000.00000000`、`10000000000.00000001`。

    参看:
    
    - [Cephfs数据池数据对象命名规则解析](https://segmentfault.com/a/1190000041823850)


1. **RBD(块存储)**

    基础名称通常是`rbd_data.[image_id]`。RBD镜像（Image）会被切分为许多RADOS对象，对象名格式为`rbd_data.[image_id].[objnum]`。例如, `rbd_data.123abc.0000000000000400`。


1. **RGW(对象存储)**

    基础名由`[桶名]/[对象名]`构成，但会经过哈希处理，以确保分步均匀。实际的对象名还会包含`_`和`[对象分片编号]`等后缀。

1. **LibRados(直接调用)**

    当通过librados接口直接写入条带化对象时，RADOS对象名会在你提供的`对象ID`后追加`.`和`16字节的十六进制序号`（例如`foo.0000000000000000`），或`###<object number>`来区分。

    参看:

    - [go-ceph/rados/striper/doc.go](https://git.redxen.eu/RepoMirrors/go-ceph/src/commit/2c8601001720606bf444453aa2448065d64ada87/rados/striper/doc.go


总结来说，不同stripe对应的RADOS对象名称各不相同。它们共享同一个由上层应用决定的“基础名”，但通过一个计算出的、唯一的数字后缀 (objnum) 来进行区分。