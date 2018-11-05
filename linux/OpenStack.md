# OpenStack

![Openstack-知识框架](OpenStack-知识框架.png)

OpenStack 分为:
- 控制节点 -- 包括虚机建立, 迁移, 网络分配, 存储分配 等控制
- 计算节点 -- 虚机运行
- 网络节点 -- 对外网络与对内网络的通信
- 存储节点 -- 额外存储管理

## 控制节点

- 管理支持服务
  - MySQL -- 数据库
  - Qpid -- 消息中间件
- 基础管理服务
  - Keystrone -- 认证管理服务
  - Glance -- 镜像管理服务
  - Nova -- 计算服务
  - Neutron -- 网络管理服务
  - Horizon -- 控制台服务
- 扩展管理服务
  - Cider -- 管理存储节点的 Cider 相关
  - Swift -- 管理存储节点的 Heat 相关
  - Trove -- 管理数据节点的 Trove 相关
  - Heat -- 基于模板实现云环境中资源的初始化
  - Centrmeter -- 对物理资源以及虚拟资源的监控
