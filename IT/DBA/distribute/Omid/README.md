# Omid

![软件架构](architecture.png)

- Omid 通过记录 key 的 (murmur3)hash 替代 key , 减少存储空间

```java
public long getCellId() {
    return Hashing.murmur3_128().newHasher()
        .putBytes(table.getTableName())
        .putBytes(row)
        .putBytes(family)
        .putBytes(qualifier)
        .hash().asLong();
}
```

# Omid

![软件架构](v2-eeffb6088467586bf75a8157c0f4ead9_hd.jpg)

从架构图获得 部署/组件职责/组件交互 的信息

## 部署

构成 Omid 的三个组件为: Client, TM(Transaction Manager) 和 Persistent Storage
- Client: 发起事务, 读/写私有副本, 提交新版本和更新本地版本
- TM(单点): 处理 Client 请求, 分配时间戳, 冲突检测, 提交事务, 写 Persistent Storage 和高可用
- Persistent Storage: 提供可靠的多版本(MVCC ?)的持久化 key-value 存储, 支持单行原子操作或事务

## 组件交互(data flow/control flow)

![组件交互](v2-25ea22d8dc090589b279838d3cdc0793_hd.jpg)
