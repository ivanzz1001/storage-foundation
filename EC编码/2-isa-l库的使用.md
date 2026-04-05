# Erasure Code: isa-l库的使用

文章转载自[erasure code: isa-l库使用](https://blog.csdn.net/wsqyz/article/details/142322571), 并对其中的一些内容做了相应的修改。

参看：

- [isa-l库github](https://github.com/intel/isa-l.git)

- [纠删码(Erasure Code)的数学原理竟然这么简单](https://www.miaokee.com/2546675.html)

- [Erasure-Code(纠删码) 最佳实践](https://zhuanlan.zhihu.com/p/106096265)

- [纠删码(erasure code)介绍](https://zhuanlan.zhihu.com/p/554262696)


<br>

## 1. isa-l库的使用

最近在PureFlash存储项目上实现EC功能，用到了intel isa-l库的Erasure code。查看了`isa-l`库关于EC的使用用例，这里记录一下编码以及解码过程中函数的使用。

在EC的实现中，我们想要对一段数据进行编码，假设`chunk size`为128K， ec模式为`8+4`(k=8, p=4, m=12)，因此当凑足1M数据后，我们需要对1M数据进行编码，生成512K的校验数据。

