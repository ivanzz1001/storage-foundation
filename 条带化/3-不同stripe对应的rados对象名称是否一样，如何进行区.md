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
  





<br>

## 🏷️ 第二步：拼接“基础名” (Base Name)

