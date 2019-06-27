# MySQL Network Protocol

[泮关森](http://www.godpan.me/2017/11/10/mysql-protocol.html)

[胡桃夹子](http://hutaow.com/blog/2013/11/06/mysql-protocol-analysis/)

## String 编码

- `FixedLengthString`(定长方式): 需先知道 String 的长度, MySQL 中的一个例子就是 `ERR_Packet` 包((后续会讲到)就使用了这种编码方式, 因为它的长度固定, 用 5 个字节存储所有数据
- `NullTerminatedString`(Null 结尾方式): 字符串以遇到 Null 作为结束标志, 相应的字节为 00
- `VariableLengthString`(动态计算字符串长度方式): 字符串的长度取决于其他变量计算而定, 比如一个字符串由 `Integer` + `Value` 组成, 我们通过计算 Integer 的值来获取 Value 的具体的长度
- `LengthEncodedString`(指定字符串长度方式): 与 `VariableLengthString` 原理相似, 是它的一种特殊情况, 具体例子就是我上条举的这个例子
- `RestOfPacketString`(包末端字符串方式): 一个包末端的字符串, 可根据包的总长度金和当前位置得到字符串的长度, 实际中并不常用

## 基本数据包格式(LittleEndian)

### Server to Client

- 每个数据包大小不能超过 **2^24** bytes, 否则执行分包
- 每个数据包前都需要加上数据包信息
- (特别的), 当数据等于 2^24-1 bytes, 仍会执行分包, 但是第二个包是一个空包

| Type   | Name           | Description                                                             |
| :-     | :-             | :-                                                                      |
| int<3> | payload_length | 数据包具体长度, 除去首部 4 字节后开始的内容                             |
| int<1> | sequence_id    | 每个包的序列 id , 总数据内容大于 16MB 时使用, 从 0 开始, 新命令重载为 0 |
| string | payload        | 包中除去头部后的具体数据内容                                            |

### Client to Server

| Type   | Name     | Description  |
| int<1> | 执行命令 | 执行的操作码 |
| string | 参数     | 命令相应参数 |

| 类型值 | 	命令                  | 	   功能                |
|     :- | :-                        | :-                         |
|   0x00 | `COM_SLEEP`	           | (内部线程状态)             |
|   0x01 | `COM_QUIT`	            | 关闭连接                   |
|   0x02 | `COM_INIT_DB`	         | 切换数据库                 |
|   0x03 | `COM_QUERY`               | SQL 查询请求               |
|   0x04 | `COM_FIELD_LIST`          | 获取数据表字段信息         |
|   0x05 | `COM_CREATE_DB`           | 创建数据库                 |
|   0x06 | `COM_DROP_DB`             | 删除数据库                 |
|   0x07 | `COM_REFRESH`             | 清除缓存                   |
|   0x08 | `COM_SHUTDOWN`            | 停止服务器                 |
|   0x09 | `COM_STATISTICS`          | 获取服务器统计信息         |
|   0x0A | `COM_PROCESS_INFO`        | 获取当前连接的列表         |
|   0x0B | `COM_CONNECT`             | (内部线程状态)             |
|   0x0C | `COM_PROCESS_KILL`        | 中断某个连接               |
|   0x0D | `COM_DEBUG`               | 保存服务器调试信息         |
|   0x0E | `COM_PING`                | 测试连通性                 |
|   0x0F | `COM_TIME`                | (内部线程状态)             |
|   0x10 | `COM_DELAYED_INSERT`      | (内部线程状态)             |
|   0x11 | `COM_CHANGE_USER`         | 重新登陆(不断连接)         |
|   0x12 | `COM_BINLOG_DUMP`         | 获取二进制日志信息         |
|   0x13 | `COM_TABLE_DUMP`          | 获取数据表结构信息         |
|   0x14 | `COM_CONNECT_OUT`         | (内部线程状态)             |
|   0x15 | `COM_REGISTER_SLAVE`      | 从服务器向主服务器进行注册 |
|   0x16 | `COM_STMT_PREPARE`        | 预处理 SQL 语句            |
|   0x17 | `COM_STMT_EXECUTE`        | 执行预处理语句             |
|   0x18 | `COM_STMT_SEND_LONG_DATA` | 发送 BLOB 类型的数据       |
|   0x19 | `COM_STMT_CLOSE`          | 销毁预处理语句             |
|   0x1A | `COM_STMT_RESET`          | 清除预处理语句参数缓存     |
|   0x1B | `COM_SET_OPTION`          | 设置语句选项               |
|   0x1C | `COM_STMT_FETCH`          | 获取预处理语句的执行结果   |

## 数据库账户验证阶段

1. Client 与 Server 进行连接
2. Server 向 Client 发送 Handshake packet
3. CLient 向 Server 发送 Auth packet
4. Server 向 Client 发送 OK packet 或 Err Packet

### Handshake packet

| 相对包内容的位置 |                    长度/byte | 名称               | 描述                                                             |
| :-               |                           :- | :-                 | :-                                                               |
| 0                |                            1 | 协议版本           | 协议版本的版本号(通常为 10/0X0A )                                |
| 1                | len=strlen(`server_version`)+1 | 数据库版本         | 通过 NullTerminated 格式编码, 长度为 len(`server_version`)+1(0x00) |
| len+1            |                            4 | thread ID          | MySQL Server 上的 thread ID                                      |
| len+5            |          8+1(0x00 表示 结束) | 挑战随机数(Part 1) | 用于后续账户密码验证                                             |
| len+14           |                            2 | 协议协商           | 用于与客户端协商通讯方式                                         |
| len+16           |                            1 | 编码格式           | 标识数据库目前的编码格式                                         |
| len+17           |                            2 | 服务器状态         | 标识服务器状态(如 处于事务模式 / 自动条件事务模式)               |
| len+19           |                           13 | 保留字节           |                                                                  |
| len+32           |              12+1(0x00)      | 挑战随机数(Part 2)      | 用于后续账户密码验证                                                       |

### Auth packet

| 相对包内容的位置 | 长度/byte | 名称         | 描述                                           |
|               :- |        :- | :-           | :-                                             |
|                0 |         4 | 协议协商     | 用于与服务端通讯方式                           |
|                4 |         4 | 消息最长长度 | 客户端(可)发送或接收的最长长度, 0 表示不做限制 |
|                8 |         1 | 字符编码     | 客户端字符编码方式                             |
|                9 |        23 | 保留字节     |                                                |
| 32               | 不定        | 认证字符串        | (用户名/NullTerminatedString 编码)+(加密后密码/LengthEncodedString 编码)+(数据库名称(可选)/NullTerminatedString 编码) |

## 命令执行阶段

即 Server 端接收命令后执行的相应操作, 所发出的数据包:

- Data Field: 包数据的一个基础结构
- OK 包(包括 PREPARE_OK): Server 端发送正确信息的包 | 包头标识 -- 0x00
- Error 包: Server 端发送错误信息的包 | 包头标识 -- 0xFF
- EOF 包: Server 发到 Client 的结束包 | 包头标识 -- 0xFE
- Result Set 包: Server 发到 Client 的查询结果包

### Data Field

查询结果包的数据编码结构, 与 LengthEncodedInteger 和 LengthEncodedString 相似

| 数据长度范围 | 数据长度 | 格式                  |
| :-           | :-       | :-                    |
| [1, 251)     | 1 byte   | 1 byte + Data         |
| [251, 2^16)  | 2 bytes  | 0xFC + 2 bytes + Data |
| [2^16, 2^24) | 4 bytes  | 0xFD + 4 bytes + Data |
| [2^24, 2^64) | 8 bytes  | 0xFE + 8 bytes + Data |
| NULL         | 0 byte   | 0xFB (251)            |

### OK 包

普通的 OK 包:
- `COM_PING`: 连接或者测试数据库
- `COM_QUERY`: 不需要查询结果集的操作(INSERT / UPDATE / ALTER TABLE 等)
- `COM_REFRESH`: 数据刷新
- `COM_REGISTER_SLAVE`: 注册从服务器

| 相对包内容的位置              | 长度/byte  | 名称       | 描述                                                   |
| :-                            | :-         | :-         | :-                                                     |
| 0                             | 1          | 包头标识   | 0x00                                                   |
| 1                             | `rows_len` | 影响行数   | 相应操作影响的行数                                     |
| 1 + `rows_len`                | `id_len`   | 自增 id    | 插入一条记录时, 如果是自增 id 的话, 返回的 id 值       |
| 1 + `rows_len` + `id_len`     | 	2      | 服务器状态 | 用于表示服务器状态, 比如是否是事务模式或者自动提交模式 |
| 3 + `rows_len` + `id_len`	 | 2          | 警告数     | 上次命令引起的警告数                                   |
| 5 + `rows_len` + `id_len`	 | `msg_len`  | 额外信息   | 此次操作的一些额外信息                                 |

### Error 包

错误返回

| 相对包内容的位置 | 长度/byte | 名称     | 描述                                |
|               :- |        :- | :-       | :-                                  |
|                0 |         1 | 包头标识 | 0xFF                                |
|                1 |         2 | 错误代码 | 该错误的相应错误代码                |
|                3 |         1 | 标识位   | SQL 执行状态标识位, 用 '#' 进行标识 |
|                4 |         5 | 执行状态 | SQL 的具体执行状态                  |
|                9 | `msg_len` | 错误信息 | 具体的错误信息                      |

### EOF Packet

EOF 包会在以下的情况产生:
- 结果集中 字段 信息结束的时候
- 结果集中 列 信息结束的视乎
- 服务器确认停止服务的时候
- 客户端发送 `COM_SET_OPTION` and `COM_DEBUG` 命令后, 服务器回应的时候
- 服务器请求使用 MySQL4.1 版本(`textRows`: 3.20 <= Ver < 4.1)之前的认证方式(Auth)的时候

| 相对包内容的位置 | 长度/byte | 名称       | 描述                 |
|               :- |        :- | :-         | :-                   |
|                0 |         1 | 包头标识   | 0xFE                 |
|                1 |         2 | 警告数     | 上次命令引起的警告数 |
|                3 |         2 | 服务器状态 |                      |

### Result Set


| 相对包内容的位置  | 含义                |
| :-                | :-                     |
| Result Set Header | 返回数据的列数量       |
| Field             | 返回数据的列信息(多个) |
| EOF               | 列结束                 |
| Row Data          | 行数据(多个)           |
| EOF               | 数据结束                   |

#### Result Set Header

| 长度      | 含义                                   |
| :-        | :-                                     |
| 1-9 bytes | 数据的列数量/LengthEncodedInteger 编码 |
| 1-9 bytes | (可选)额外信息/LengthEncodedINteger 编码     |

#### Field

| 长度 | 含义                                                         |
| :-   | :-                                                           |
| n    | >=4.1 , 目录名称/"def"(Data Field)                           |
| n    | 数据库名称(Data Field)                                       |
| n    | 假如查询指定了表别名, 就是表别名(Data Field)                 |
| n    | 原始的表名(Data Field)                                       |
| n    | 假如查询指定了列别名, 就是列别名(Data Field)                 |
| n    | 原始的列名(Data Field)                                       |
| 1    | 标识位, 通常为 12, 表示接下去的 12 个字节是具体的 field 内容 |
| 2    | field 列的编码                                               |
| 4    | field 列的长度                                               |
| 1    | field 列的类型(`/include/mysql_com.h` -- `FIELD_TYPE_*`) OR `/libbinlogevents/export/binary_log_types.h` -- `enum enum_field_types`) |
| 2    | field 列的标识                                               |
| 2    | field 值(如果是 数值 )的小数点精度                           |
| 2    | 预留字节                                                     |
| n    | 可选元素, 如果存在, 则表示该 field 的默认值                  |

#### Row Data

- 每个 Row Data 包含多个字段值, 且字段间没有间隔
- 内部的字符串通过 LengthEncodedString 编码

### PREPARE_OK packet

当 Client 向 Server 发送 预处理 SQL 语句 , Server 正确回应时的包

| 长度 | 含义                                                 |
|   :- | :-                                                   |
|    1 | 0x00 (标识是一个 OK 包)                              |
|    4 | `statement_handler_id`(预处理语句 id)                |
|    2 | number of columns in result set (结果集中列的数量)   |
|    2 | number of parameters in query (查询语句中参数的数量) |
|    1 | 0x00 (填充值)                                        |
|    2 | 	警告数                                           |

- 在执行(exec) prepare statement 时, 如果结果集(result set)的 columns 数和 parameters 数都大于 0 , 则会有额外的两个包传输以上两者信息:

| 内容  | 含义            |
| :-    | :-              |
| Field | columns 信息    |
| EOF   | columns 结束    |
| Field | parameters 信息 |
| EOF   | parameters 结束 |

### Row Data Binary

与 [Row Data](#Row Data) 的差异:
- 用不同的方式定义 NULL
- 根据数据类型的不同进行相应的编码

| 相对包内容的位置        | 长度/byte           | 名称                                 | 描述                                                                                                                          |
| :-                      | :-                  | :-                                   |                                                                                                                               |
| 0                       | 1                   | 包头标识                             | 0x00                                                                                                                          |
| 1                       | (`col_count`+7+2)/8 | Null Bit Map                         | 前两位为预留字节, 主要用于区别与其他的几种包(OK, ERROR, EOF), 在 MySQL 5 之后这两个字节都为 0X00, 其中 `col_count` 为列的数量 |
| (`col_count`+7+2)/8 + 1 | n	column values  | 具体的列值, 重复多次, 根据值类型编码 |                                                                                                                               |

#### Null Bit Map

Row Data Binary 中 NULL 的定义((`col_count` +7)/8):
- 参数个数: 1-8
- 长度: 1 byte
- 具体值范围: -1, 2^n 组合
- 描述: 1=2^0 表示第一个参数为 NULL, 3=2^0+2^1 表示第一和第二个参数为 0
- 以 8 个参数为一个周期(1 bytes)

对于 string 之外的具体类型, 编码规则:
- 基本数据类型: TINYINT -- 1 byte; FLOAT -- 4 bytes; DOUBLE -- 8 bytes
- 时间类型(Date/Datetime/Timestamp): 如以下所示

#### DATE/DATETIME/TIMESTAMP

[Binary Protocol Resultset](https://dev.mysql.com/doc/dev/mysql-server/latest/page_protocol_binary_resultset.html)

| Type   | Name        | Description                                           |
|:-      |:-           |:-                                                     |
| int<1> | length      | number of bytes following (valid values: 0, 4, 7, 11) |
| int<2> | year        | year                                                  |
| int<1> | month       | month                                                 |
| int<1> | day         | day                                                   |
| int<1> | hour        | hour                                                  |
| int<1> | minute      | minute                                                |
| int<1> | second      | second                                                |
| int<4> | microsecond | micro seconds                                         |

Example:

```shell
0b da 07 0a 11 13 1b 1e 01 00 00 00 -- datetime 2010-10-17 19:27:30.000 001
04 da 07 0a 11                      -- date = 2010-10-17
0b da 07 0a 11 13 1b 1e 01 00 00 00 -- timestamp
```

### Execute packet

Client 发给 Server, 用于执行 预处理语句

|                长度 | 含义                                                         |
|                  :- | :-                                                           |
|                   1 | `COM_EXECUTE`(标识是一个 Execute 包)                         |
|                   4 | 预处理语句 id                                                |
|                   1 | 游标类型                                                     |
|                   4 | 预留字节                                                     |
|                   0 | 接下去的内容只有在有参数的情况下                             |
| (`param_count`+7)/8 | `null_bit_map`(描述参数中 NULL 的情况)                       |
|                   1 | 参数绑定情况                                                 |
|                 n*2 | 参数类型(依次存储)                                           |
|                   n | 参数具体值(非 NULL)(依次存储, 使用 Row Data Binary 方式编码) |
