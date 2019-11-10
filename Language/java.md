# Java

## 坑

### `ClassNotFoundException: javax.xml.bind.JAXBContext`

[stackoverflow](https://stackoverflow.com/questions/43574426/how-to-resolve-java-lang-noclassdeffounderror-javax-xml-bind-jaxbexception-in-j)

运行 springboot 项目出现: `Type javax.xml.bind.JAXBContext not present`

原因: java9+ 版本以后, JAXB 默认没有加载(但是在测试中, Java8 也没有启用 jaxb, 为什么 ?)

解决方案:
```xml
<!-- spring-boot 1.5.* -->
<!-- Java 6 = JAX-B Version 2.0   -->
<!-- Java 7 = JAX-B Version 2.2.3 -->
<!-- Java 8 = JAX-B Version 2.2.8 -->
<dependency>
    <groupId>javax.xml.bind</groupId>
    <artifactId>jaxb-api</artifactId>
    <version>2.2.11</version>
</dependency>
<dependency>
    <groupId>com.sun.xml.bind</groupId>
    <artifactId>jaxb-core</artifactId>
    <version>2.2.11</version>
</dependency>
<dependency>
    <groupId>com.sun.xml.bind</groupId>
    <artifactId>jaxb-impl</artifactId>
    <version>2.2.11</version>
</dependency>
<dependency>
    <groupId>javax.activation</groupId>
    <artifactId>activation</artifactId>
    <version>1.1.1</version>
</dependency>
```

```xml
<!-- spring boot 2.0.* -->
<dependency>
   <groupId>javax.xml.bind</groupId>
    <artifactId>jaxb-api</artifactId>
</dependency>
<dependency>
    <groupId>com.sun.xml.bind</groupId>
    <artifactId>jaxb-impl</artifactId>
    <version>2.3.0</version>
</dependency>
<dependency>
    <groupId>org.glassfish.jaxb</groupId>
    <artifactId>jaxb-runtime</artifactId>
    <version>2.3.0</version>
</dependency>
<dependency>
    <groupId>javax.activation</groupId>
    <artifactId>activation</artifactId>
    <version>1.1.1</version>
</dependency>
```

## 库

### ShardingSphere

#### 读写分离

- 提供了一主多从的读写分离配置, 可独立使用, 也可配合分库分表使用
- 同个调用线程, 执行多条语句, 其中一旦发现有非读操作, 后续所有读操作均从主库读取
- Spring 命名空间
- 基于 Hint 的强制主库路由

- 调用 getConnection 方法时, 先检查 ThreadLocal 是否有连接, 没有就尝试获取一个底层连接, 然后存储到 ThreadLocal 变量中去, 下次就直接在中获取了
- sharding-jdbc 在 PreparedStatement (实际上为 ShardingPreparedStatement)的 executeXX 层进行了主从库的连接处理

```java
// 获取连接
public Connection getConnection(final String dataSourceName, final SQLType sqlType) throws SQLException {
        if (getCachedConnections().containsKey(dataSourceName)) {
            return getCachedConnections().get(dataSourceName);
        }
        DataSource dataSource = shardingContext.getShardingRule().getDataSourceRule().getDataSource(dataSourceName);
        Preconditions.checkState(null != dataSource, "Missing the rule of %s in DataSourceRule", dataSourceName);
        String realDataSourceName;
        if (dataSource instanceof MasterSlaveDataSource) {
            NamedDataSource namedDataSource = ((MasterSlaveDataSource) dataSource).getDataSource(sqlType);
            realDataSourceName = namedDataSource.getName();
            if (getCachedConnections().containsKey(realDataSourceName)) {
                return getCachedConnections().get(realDataSourceName);
            }
            dataSource = namedDataSource.getDataSource();
        } else {
            realDataSourceName = dataSourceName;
        }
        Connection result = dataSource.getConnection();
        getCachedConnections().put(realDataSourceName, result);
        replayMethodsInvocation(result);
        return result;
    }

// 如果是 MasterSlaveDataSource 则尝试获取真正的数据源(master/slave)
public NamedDataSource getDataSource(final SQLType sqlType) {
    if (isMasterRoute(sqlType)) {
        DML_FLAG.set(true);
        return new NamedDataSource(masterDataSourceName, masterDataSource);
    }
    String selectedSourceName = masterSlaveLoadBalanceStrategy.getDataSource(name, masterDataSourceName, new ArrayList<>(slaveDataSources.keySet()));
    DataSource selectedSource = selectedSourceName.equals(masterDataSourceName) ? masterDataSource : slaveDataSources.get(selectedSourceName);
    Preconditions.checkNotNull(selectedSource, "");
    return new NamedDataSource(selectedSourceName, selectedSource);
}
private static boolean isMasterRoute(final SQLType sqlType) {
    return SQLType.DQL != sqlType || DML_FLAG.get() || HintManagerHolder.isMasterRouteOnly();
}
```

以下情况下, 会走主库
- DDL: 不是查询类型的语句, 比如更新字段(`isMasterRoute()==true`)
- DML: `DML_FLAG` 变量为 true 的时候
- 强制 Hint 方式走主库

在从库中查询, 会有一个 `getAndIncr%size` 的负载均衡

```java
public final class RoundRobinMasterSlaveLoadBalanceStrategy implements MasterSlaveLoadBalanceStrategy {

    private static final ConcurrentHashMap<String, AtomicInteger> COUNT_MAP = new ConcurrentHashMap<>();

    @Override
    public String getDataSource(final String name, final String masterDataSourceName, final List<String> slaveDataSourceNames) {
        AtomicInteger count = COUNT_MAP.containsKey(name) ? COUNT_MAP.get(name) : new AtomicInteger(0);
        COUNT_MAP.putIfAbsent(name, count);
        count.compareAndSet(slaveDataSourceNames.size(), 0);
        // 负载均衡
        return slaveDataSourceNames.get(count.getAndIncrement() % slaveDataSourceNames.size());
    }
}
```
