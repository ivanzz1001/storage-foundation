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

随着存储文件size的增加，可以通过将客户端数据条带化分割存储到多个Objects中，同时由于Object映射到不同的PG上进而会映射到不同的OSD上，这样就能够充分利用每个OSD对应的物理磁盘设备的IO性能，以此实现每个并行的写操作都以最大化的速率进行。随着条带数的增加对写性能的提升也是相当可观的。如下图，数据被分割存储到两个Object Set中，条带单元块存储的顺序为`stripe unit 0~31`:

![ceph-stripe](https://raw.githubusercontent.com/ivanzz1001/storage-foundation/master/%E6%9D%A1%E5%B8%A6%E5%8C%96/image/ceph_stripe_2.jpg)


ceph有3个重要参数会对条带化产生影响：

- **order**: 表示object size。例如order=22，则2^22即为4MB大小。object的大小应该足够大以便与条带单元块相匹配，而且object的大小应该是条带单元块大小的数倍。RedHat建议的Object大小是16MB。

- **stripe_unit**: 表示条带单元块的宽度。客户端写入Object的数据切分为宽度相同的条带单元块（最后一块的宽度可以小于stripe_unit)。条带宽度应该是object大小的一个分数。比如：Object size为4MB，单元块大小为1MB，那么一个Object就能包含4个单元块，以便充分利用object的空间。

- **stripe_count**: 条带宽度，也就是一个strip跨多少个对象，也就是一个objectset中对象的个数

>Note: 由于客户端会指定单个pool进行写入，所以条带化到objects中的所有数据都会被映射在同一个pool包含的PG内

假设上图为RBD Image的写入过程，则：

1.  RBD image会被保存在总共8个RADOS object中（计算方式： client_data_size除以`2^[order]`)

1. stripe_unit为object size的四分之一，也就是说每个object包含4个stripe

1. stripe_count为4，即每个object set包含4个object。这样，client以4为一个循环，向一个object set中的每个object依次写入stripe，写到第16个stripe后，按照同样的方式写第二个object set。

## 2.  地址空间转换

通常，如果我们要读取一个文件的某一段数据，我们只需要(`object-name`, `offset`, `length`)这三个参数就可以了。这个读取过程其实是有一个前提：文件数据存在于一个线性的一维地址空间。如下图所示:

![ceph-stripe](https://raw.githubusercontent.com/ivanzz1001/storage-foundation/master/%E6%9D%A1%E5%B8%A6%E5%8C%96/image/ceph_stripe_3.jpg)

但现在文件数据经过条带化后，其就变成了一个三维地址空间(`objectset`, `object`, `stripe`)。函数file_to_extents()用于实现此功能:

```C
void Striper::file_to_extents(
  CephContext *cct, const char *object_format,
  const file_layout_t *layout,
  uint64_t offset, uint64_t len,
  uint64_t trunc_size,
  map<object_t,vector<ObjectExtent> >& object_extents,
  uint64_t buffer_offset)
{
  ldout(cct, 10) << "file_to_extents " << offset << "~" << len
		 << " format " << object_format
		 << dendl;
  assert(len > 0);

  /*
   * we want only one extent per object!  this means that each extent
   * we read may map into different bits of the final read
   * buffer.. hence ObjectExtent.buffer_extents
   */

  __u32 object_size = layout->object_size;
  __u32 su = layout->stripe_unit;
  __u32 stripe_count = layout->stripe_count;
  assert(object_size >= su);
  if (stripe_count == 1) {
    ldout(cct, 20) << " sc is one, reset su to os" << dendl;
    su = object_size;
  }
  uint64_t stripes_per_object = object_size / su;
  ldout(cct, 20) << " su " << su << " sc " << stripe_count << " os "
		 << object_size << " stripes_per_object " << stripes_per_object
		 << dendl;

  uint64_t cur = offset;
  uint64_t left = len;
  while (left > 0) {
    // layout into objects
    uint64_t blockno = cur / su; // which block
    // which horizontal stripe (Y)
    uint64_t stripeno = blockno / stripe_count;
    // which object in the object set (X)
    uint64_t stripepos = blockno % stripe_count;
    // which object set
    uint64_t objectsetno = stripeno / stripes_per_object;
    // object id
    uint64_t objectno = objectsetno * stripe_count + stripepos;

    // find oid, extent
    char buf[strlen(object_format) + 32];
    snprintf(buf, sizeof(buf), object_format, (long long unsigned)objectno);
    object_t oid = buf;

    // map range into object
    uint64_t block_start = (stripeno % stripes_per_object) * su;
    uint64_t block_off = cur % su;
    uint64_t max = su - block_off;

    uint64_t x_offset = block_start + block_off;
    uint64_t x_len;
    if (left > max)
      x_len = max;
    else
      x_len = left;

    ldout(cct, 20) << " off " << cur << " blockno " << blockno << " stripeno "
		   << stripeno << " stripepos " << stripepos << " objectsetno "
		   << objectsetno << " objectno " << objectno
		   << " block_start " << block_start << " block_off "
		   << block_off << " " << x_offset << "~" << x_len
		   << dendl;

    ObjectExtent *ex = 0;
    vector<ObjectExtent>& exv = object_extents[oid];
    if (exv.empty() || exv.back().offset + exv.back().length != x_offset) {
      exv.resize(exv.size() + 1);
      ex = &exv.back();
      ex->oid = oid;
      ex->objectno = objectno;
      ex->oloc = OSDMap::file_to_object_locator(*layout);

      ex->offset = x_offset;
      ex->length = x_len;
      ex->truncate_size = object_truncate_size(cct, layout, objectno,
					       trunc_size);

      ldout(cct, 20) << " added new " << *ex << dendl;
    } else {
      // add to extent
      ex = &exv.back();
      ldout(cct, 20) << " adding in to " << *ex << dendl;
      ex->length += x_len;
    }
    ex->buffer_extents.push_back(make_pair(cur - offset + buffer_offset,
					   x_len));

    ldout(cct, 15) << "file_to_extents  " << *ex << " in " << ex->oloc
		   << dendl;
    // ldout(cct, 0) << "map: ino " << ino << " oid " << ex.oid << " osd "
    //		  << ex.osd << " offset " << ex.offset << " len " << ex.len
    //		  << " ... left " << left << dendl;

    left -= x_len;
    cur += x_len;
  }
}
```

这个过程其实还算比较简单，如下图所示：


![ceph-stripe](https://raw.githubusercontent.com/ivanzz1001/storage-foundation/master/%E6%9D%A1%E5%B8%A6%E5%8C%96/image/ceph_stripe_4.jpg)