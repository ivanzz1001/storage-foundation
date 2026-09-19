# 图文详解io_uring高性能异步IO架构

参考：https://www.51cto.com/article/777885.html

说到高性能网络编程，我们第一时间想到的是epoll机制，epoll很长一段时间统治着整个网络编程江湖，然而io_uring的出现，似乎在撼动epoll的统治地位，今天我们来揭开io_uring的神秘面纱。


