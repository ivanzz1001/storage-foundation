# 同一文件的不同stripe对应的rados对象名称是否一样，如何进行区？


在Ceph中，每个RADOS对象的名称都是唯一的。Ceph通过一个分层且结构化的命名规则来区分它们，其核心是计算一个唯一的“对象编号”（objnum），然后将它拼接到一个“基础名”（Base Name）上

<br>

## 🎯 第一步：核心公式——计算“对象编号” (objnum)

`objnum`是一个从0开始的递增整数，其计算逻辑由Ceph文件布局（Layout）的四个关键参数决定:

- `stripe_unit`(条带单元): 4MB

- `stripe_count`(条带宽度): 4

- `object_size`(对象大小): 4MB(此例中等于stripe_unit)

- `file_offset`(文件内容偏移): 指待读写文件在原始文件中的起始位置








<br>

## 🏷️ 第二步：拼接“基础名” (Base Name)

