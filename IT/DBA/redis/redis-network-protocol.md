# Redis Network Protocol

## 介绍

Redis 网络协议 -- RESP(REdis Serialization Protocol)

Client 和 Server 基于以下 5 种基本类型:
- 简单字符串: 服务器用来返回简单的结果( "OK" 或者 "PONG" )
- bulk string: 大部分单值命令的返回结果(GET, LPOP, HGET 等)
- 整数: 查询长度的命令的返回结果
- 数组: 可以包含其他 RESP 对象, 设置数组, 用来发送命令给服务器, 也用于返回多个值的命令
- Error: 返回错误信息

RESP 的第一个字节表示数据的类型:
- 简单字符串: 第一个字节是 `+`, `+OK\r\n`
- bulk string: `$`, `$6\r\nfoobar\r\n`
- 整数: `:`, `:1000\r\n`
- 数组: `*`, `*2\r\n$3\r\nfoo\r\n$3\r\nbar\r\n`
- Error: `-`, `-Error message\r\n`
