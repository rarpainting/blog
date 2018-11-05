# Tomcat

## 运行模式

### bio

默认模式, 未经过优化处理及支持

### nio

基于缓冲区, 提供非阻塞 I/O 操作的 Java API

### apr

从操作系统级别解决 **异步的 I/O 问题**

## 优化方案

- 从 bio 切换到 nio/apr
- 使用线程池来处理请求
- 禁用 AJP(Apache JServer Protocol) 连接器
  - Web 服务器和 Servlet 容器通过 TCP 连接交互
  - 为了节省 socket 创建的昂贵代价, Web Server 会尝试维护一个永久 TCP 连接到 servlet 容器, 并在多个请求和响应周期过程会重用连接
