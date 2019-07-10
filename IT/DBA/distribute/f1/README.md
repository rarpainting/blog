# Google F1

## 异步 schema 变更

## F1 简介

F1 设计目标:
- 可扩展性: F1 的用户经常有复杂查询和 join, F1 提供了增加机器就能自动 resharding 和负载均衡
- 高可用性
- 事务: 提供包含 Spanner 的外部一致性的 ACID 事务
- 易用性:友好的 SQL 接口、索引支持、即查询

![F1 架构](16020-48e9028a2cfd789f.png)

## F1 数据模型

![RDBMS 和 F1/Spanner 的层级结构区别](16020-afed4ae956645e2e.png)

层级存储:
- 可以并行化处理
- 可以加速 update 操作

## 索引

索引在 F1 中通过单个表存储起来; 同时支持对 常见类型的字段 和 Protocol Buffer 里的字段 索引

索引有两种:
- Local: 包含 root row 主键的索引为 local 索引
  - 索引和 root row 在同一个 directory
  - 索引文件和被索引的 row 放在同一个 spanserver
- Global: 不包含 root row 主键的索引为 local 索引
  - 索引被 shared 在多个 spanserver

## 无阻塞 schema 变更

## ACID 事务

F1 数据库层面的事务一致性:
- 快照事务: 只读事务
- 悲观事务: Spanner 标准事务
- 乐观事务: 两阶段: 无锁读, 而后短暂写
  - 写阶段: 尝试 CAS 修改该行的 lock 列(包含最后修改的 timestamp); 如果成功则表示事务成功 commit, 否则表示写重读

## 灵活的锁颗粒度

- 默认使用行级锁, 且只会锁这行的一个默认列
- 可以通过客户端锁定其他栏
- 层级锁: 锁定父表的列, 字表对应的列也会相应锁上

## 变更历史记录

- 每个事务变更都会通过 Protobuf 记录(包括变更前后的列值, 对应主键和事务提交的时间戳), 而且 root 表下会产生单独的 ChangeBatch 子表存放这些子表
- 一个事务同时变更多个 root 表, 每个 root 表的变更依然写入它的子表的对应变更值, 但是同时多了 **连接到其他 root 表的指针**

## F1 client

## 分布式 SQL

F1 支持集中式 SQL (在一个 F1 server 运行) 和 分布式 SQL

```sql
SELECT agcr.CampaignId, click.Region,cr.Language, SUM(click.Clicks) FROM
 AdClick clickJOIN AdGroupCreative agcrUSING (AdGroupId, CreativeId)
 JOIN Creative crUSING (CustomerId, CreativeId) WHERE click.Date ='2013-03-23'
 GROUP BY agcr.CampaignId, click.Region,cr.Language
```

可能的执行计划

![执行计划](16020-89b4dc180a2f5450.png)

- 一个 SQL 生成数十个更小的执行计划, 每个计划通过 有向无环图(DAG) 合并到最顶层
- hash partition 可以通过时刻调整, 而让 F1 的 partition 更高效; 而 range patition 需要相关统计
- F1 的运算都在内存中执行, 再加上管道的运用, 没有中间数据会存放在磁盘上
- 为了减少网络延迟, F1 使用 批处理和管道技术

## 层级表间 Join

## 客户端的并行

## Protobuf 提供嵌套表支持

