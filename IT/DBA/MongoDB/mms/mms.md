# MongoDB MMS(MongoDB Monitor Service) 本地部署

1. 从官方安装 MongoDB-MMS 的 deb 包并安装

2. 运行 MongoDB 副本集, 开启 --replSet 选项
```bash
/usr/bin/mongod \
	--port port \
	--dbpath /var/lib/repl1(/2/3) \
	--logpath /var/log/mongodb-repl/repl1(/2/3).log \
	--replSet mms \
	--unixSocketPrefix=/run/mongo-repl \
	--config /etc/mongodb-repl.conf \
	--logappend --fork
```

3. 配置副本集
```json
config = {
    "_id": "mms",
    "members": [{
        "_id": 0,
        "host": "host1:port1",
    }, {
        "_id": 1,
        "host": "host2:port2",
    }, {
        "_id": 2,
        "host": "host3:port3",
    }]
}

rs.initiate(config)
```

4. 修改 /opt/mongodb/mms/conf/conf-mms.properties 并开启 MongoDB-MMS 服务
```bash
mongo.mongoUri=mongodb://host1:port1,host2:port2,host3:port3/?maxPoolSize=150
```

```bash
/opt/mongodb/mms/bin/mongodb-mms start
```

5. 网页登录本地的 MongoDB-MMS , 配置 Agent 的 GroudId , ApiKey 和 BaseUrl 在 /etc/mongodb-mms/automation-agent.config

6. 在本地的 MMS 服务器开启 Agent 服务作为通讯的服务器端

```bash
systemctl start mongodb-mms-automation-agent.service
or
/opt/mongodb-mms-automation/bin/mongodb-mms-automation-agent -f /etc/mongodb-mms/automation-agent.config
```

7. 在需要监控的 MongoDB Server 端也开启 Agent 服务作为客户端(同时修改必须的参数)

注: 需要监控的 Server Agent Host 不能是 127.0.0.1

## 各种疑问

1. 通过 docker 搭建, 但是 MMS Service 没办法和 MongoDB Server 通过 Agent 连接

可能: 因为 docker 里面, 用的是 rootUser, 导致两端认证用的 Agent 的 User 不一致 ?

如果通过 sudoer 修改 权限, 那么 docker volume 卷存储的数据还是 root User 吗?

2. 经常出现 CPU 资源占用率低, 但是机器负载很高的情况

可能: 一开始给 MMS 的句柄限制太低了, 导致需要经常释放无用的句柄给进程, 所以 IO 过于频繁 ?
