# NVMe（Non-Volatile Memory Express）详解

>转载自: https://zhuanlan.zhihu.com/p/32725156182

## 1. NVMe的定义与核心特性

NVMe(非易失性内存主机控制器接口规范）是一种基于`PCIe总线`的高性能存储协议，专为固态硬盘（SSD）设计，旨在替代传统的AHCI协议（如SATA）。其核心特性包括：

- **低延迟**: 命令队列深度提升至`64K`（AHCI仅32），减少I/O等待时间（典型延迟<100μs);

- **高吞吐量**：支持PCIe 4.0 x4带宽（8GB/s），PCIe 5.0 x4可达16GB/s;

- **多队列并行**: 支持多核CPU并行访问，优化多线程性能;

- **扩展性**: 支持NVMe over Fabrics（NVMe-oF），实现远程存储访问;

