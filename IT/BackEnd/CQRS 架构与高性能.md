# CQRS 架构与高性能
## 高性能要求
	避开网络开销(IO), 避开海量数据, 避开资源争夺
## CQRS(Command Query Responsibility Segregation) 架构
Command Side -- C端
Query Side -- Q端
系统处理 -- 命令处理(写请求)+ 查询处理(读请求)
## 避开资源争夺
### 工作单元(Unit of Work, 简称UoW)模式
为实现工作单元的流程控制,  要求
### 1.让一个 Command 总是只修改一个聚合根
- 确保一个事务一次只修改一个聚合根
-
### 2.对修改同一个聚合根的 Command 进行排队
