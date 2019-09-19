# 工具链

整理下这一年半的工作以及学习上的工具链, 顺便整理一下 github 的收藏

绝赞推荐: [后端架构师图谱](https://github.com/xingshaocheng/architect-awesome)

## rpc

- [grpc](https://github.com/grpc/grpc)
  - [grpc-go](https://github.com/grpc/grpc-go)
  - [grpc-web](https://github.com/grpc/grpc-web)
- [rpcx](https://github.com/smallnest/rpcx)

## 消息队列

- [kafka](https://github.com/apache/kafka): zookeeper
- [nsq](https://github.com/nsqio/nsq): zookeeper, golang
- [redis](https://redis.io/)/[disque](https://github.com/antirez/disque)
- [etcd](https://github.com/etcd-io/etcd)

## data engine

- [badger](https://github.com/dgraph-io/badger): LSM
- [bolt](https://github.com/boltdb/bolt): B-Tree

## 搜索引擎

- [elasticsearch](https://www.elastic.co/): [go-mysql-elasticsearch](https://github.com/siddontang/go-mysql-elasticsearch)

## 服务器负荷测试

- http benchmark:
  - [ab](https://httpd.apache.org/docs/2.4/programs/ab.html): apache benchmark
  - [hey](https://github.com/rakyll/hey): golang
  - [bombardier](https://github.com/codesenberg/bombardier): golang
- [fio](https://github.com/axboe/fio): 磁盘 IOPS 测试
  - `fio -filename=$filename -direct=1 -iodepth 1 -thread -rw=randrw -ioengine=psync -bs=16k -size=500M -numjobs=10 -runtime=10 -group_reporting -name=mytest`

## 服务器系统运维

- [saltStack](http://docs.saltstack.cn/)
- [Ansible](http://ansible.com.cn/)

## 服务器监控

- [Grafana](https://grafana.com/): 开箱即用的可视化工具
- [influxdb](https://github.com/influxdata/influxdb): 备份, 存储分析

## 协议转换

- [kcptun](https://github.com/xtaci/kcptun): tcp 转 [kcp](https://github.com/skywind3000/kcp)+udp

## 前端工具

- [banshee](https://github.com/eleme/banshee)/[statsd-benshee](https://github.com/etsy/statsd): golang -- 前端 埋点工具
- [fetch](https://github.com/github/fetch): [fetch API](https://developer.mozilla.org/zh-CN/docs/Web/API/Fetch_API) 的修补包(polyfill)
- [isomorphic-fetch](https://github.com/matthew-andrews): fetch API 的非安全实现(修改 `fetch` 的全局)
- [redux-saga](https://github.com/redux-saga/redux-saga): [redux](https://github.com/reduxjs/redux) 的异步事件处理库
- [react-dnd](https://github.com/react-dnd/react-dnd): 浏览器拖拽 API 的 react 组件库
- [immutability-helper](https://github.com/kolodny/immutability-helper): 类 MongoDB 语法的不可变数据 创建/更新 工具
- 编辑器:
  - [slate](https://github.com/ianstormtaylor/slate): 前端富文本编辑器框架, 与 react 关系更密切
  - [Draft.js](https://github.com/facebook/draft-js): 富文本编辑器
- [react virtualized](https://github.com/bvaughn/react-virtualized): 长~列表方案, 优秀的懒加载实现

## 后端工具

- [fasthttp](https://github.com/valyala/fasthttp): net/http 的快速版本
- [httprouter](https://github.com/julienschmidt/httprouter): 框架 [Gin](https://github.com/gin-gonic/gin) 使用的路由管理工具
- [docopt.go](https://github.com/docopt/docopt.go): 命令行参数解析
- [casbin](https://github.com/casbin/casbin): 权限管理认证, 规范工具
- [casbin-server](https://github.com/casbin/casbin-server): casbin as a server(CAAS), gRPC 交互
- [go-ldap](https://github.com/go-ldap/ldap): golang 的 [ldap](https://www.cnblogs.com/yjd_hycf_space/p/7994597.html) client
- [elastic-go](https://github.com/olivere/elastic): elasticsearch 的 golang client
- [mongo-connector](https://github.com/yougov/mongo-connector): 同步 mongo 和 elasticsearch 数据的工具
- [go-mysql-elasticsearch](https://github.com/siddontang/go-mysql-elasticsearch): 同步 mysql 和 elasticsearch 数据的工具
- [gops](https://github.com/google/gops): 调试正在运行的 golang 程序
- [monkey](https://github.com/bouk/monkey): golang 补丁包
- [localroast](https://github.com/caalberts/localroast): golang 的 Mock 工具, 标准文本是 json
- [gopherjs](https://github.com/gopherjs/gopherjs): 编写能在浏览器运行的 go 代码, wasm ?

## 架构/框架

- [ENode](http://www.cnblogs.com/netfocus/category/496012.html): [领域驱动设计(DDD)](http://www.cnblogs.com/netfocus/archive/2011/10/10/2204949.html), 事件驱动, [CQRS](https://www.cnblogs.com/yangecnu/p/Introduction-CQRS.html)模型, 最终一致性, 事件溯源, 分布式
- [FalconEngine](https://github.com/wyh267/FalconEngine): golang, 搜索引擎
- [language-server-protocol](https://github.com/Microsoft/language-server-protocol): 编程语言服务器协议

## 玩具

- [KeyCastOW](https://github.com/brookhong/KeyCastOW): 键盘按键修改工具
- [Spacemacs](https://github.com/syl20bnr/spacemacs): emacs 开箱工具
- [certbot](https://github.com/certbot/certbot): tls 证书自动更新工具

