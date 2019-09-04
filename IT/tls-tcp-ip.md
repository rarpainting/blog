# TLS-TCP-IP

<!-- TOC -->

- [TLS-TCP-IP](#tls-tcp-ip)
	- [以太帧](#以太帧)
	- [IPV4](#ipv4)
	- [TCP](#tcp)
		- [Golang-TCPConn 相关设置](#golang-tcpconn-相关设置)
	- [UDP](#udp)
	- [TLS1.2](#tls12)
		- [1. Client-Hello 阶段: (TLS handshake)](#1-client-hello-阶段-tls-handshake)
		- [2. Server-Hello 阶段: (TLS handshake)](#2-server-hello-阶段-tls-handshake)
		- [3. Cipher-spec: (TLS handshake) 检测签名的有效性:](#3-cipher-spec-tls-handshake-检测签名的有效性)
		- [其余的握手报文](#其余的握手报文)
		- [通讯阶段](#通讯阶段)
		- [吊销检查](#吊销检查)
		- [如何加速 TLS 过程](#如何加速-tls-过程)
	- [TLS1.3](#tls13)
		- [术语](#术语)
			- [PSK(`pre_shared_key`) -- 密钥交换 + 身份认证 机制](#pskpre_shared_key----密钥交换--身份认证-机制)
			- [HKDF(`HMAC_based_key_derivation_function`) -- 密钥导出函数](#hkdfhmac_based_key_derivation_function----密钥导出函数)
			- [AEAD(`Authenticated_Encrypted_associated_data`) -- 唯一保留的加密方式](#aeadauthenticated_encrypted_associated_data----唯一保留的加密方式)
		- [handshake 子协议](#handshake-子协议)
			- [加密套件](#加密套件)
			- [DHE 密钥协商机制](#dhe-密钥协商机制)
			- [post-authentication 机制](#post-authentication-机制)
		- [record 子协议](#record-子协议)
		- [alert 子协议](#alert-子协议)

<!-- /TOC -->

## 以太帧

![以太帧](20160428103116706.jpeg)

`Preamble`: 前导码, 7 bytes, 用于数据传输过程中的双方发送、接收的速率的同步

`SFD`: 帧开始符, 1 bytes, 表明下一个字节开始是真实数据(目的MAC地址)

`dst MAC`: 目的MAC地址, 6 bytes, 指明帧的接受者

`src MAC`: 源 MAC 地址, 6 bytes, 指明帧的发送者

`Length`: 长度, 2 bytes, 指明该帧数据字段(`Data and Pad`)的长度, 但不代表数据字段长度能够达到(2^16)bytes

`Type`: 类型, 2 bytes, 指明帧中数据的协议类型, 比如常见的 IPv4 中 ip 协议采用 0x0800

`Data and Pad`: 数据与填充, 46~1500 bytes, 包含了上层协议传递下来的数据, 如果加入数据字段后帧长度不够 64 bytes, 会在数据字段加入 "填充" 至达到 64 bytes

`FCS`: 帧校验序列, 4 bytes, 对接收网卡(主要是检测 Data and Pad 字段)提供判断是否传输错误的一种方法, 如果发现错误, 丢弃此帧; 目前最为流行的用于 FCS 的算法是循环冗余校验(cyclic redundancy check –CRC)

## IPV4

![ipv4](v2-99894d16100918082dff4edb03a079b1_r.jpg)

`Version`: 版本, 4 bits, 用来表明 IP 协议实现的版本号, 当前一般为 IPv4, 即 0100 , IPv6 的为 0110 ; 这个字段确保可能运行不同IP版本的的设备之间的兼容性

`IHL`: 报头长度, 4 bits, 以 `1 = 32 bits` 的字来定义 IP 首部的长度, 包括可选项; 若该字段的值是 5 , 即 `5*32=160 比特=20 字节`; 此字段最大值为 `60(15*32/8=60)bytes`

`TOS`: 服务类型, 8 bits, 用于携带提供服务质量特征信息的字段, 服务类型字段声明了数据报被网络系统传输时可以被怎样处理; 其中 前 3 bit 为优先权子字段(Precedence, 现已被忽略, 各种终端都不采用); 第 8 bit 保留未用; 第 4 至第 7 bit 分别代表延迟、吞吐量、可靠性和花费; 当它们取值为 1 时分别代表要求最小时延、最大吞吐量、最高可靠性和最小费用; 这 4 bit 的服务类型中只能置其中 1 比特为 1 ; 可以全为 0, 若全为 0 则表示一般服务; 大多数情况下该 TOS 会被忽略

`Total Length`: 总长度, 16 bits, 指明整个数据报的长度, 按字节为计算; 最大长度为65535(2^16=65536)字节

`Identification`: 标识, 16 bits, 用来唯一地标识主机发送的每一份数据报; IP 软件会在存储器中维持一个计数器, 每产生一个数据段, 计数器就加 1, 并将此值赋给标识字段; 但这个 "标识" 并不是序号, 因为 IP 是无连接服务, 数据报不存在按序接收的问题; 如数据报由于超过网络的 MTU 而必须分片时, 这个标识字段的值就被复制到所有的数据报的标识字段中; 相同的标识字段的值使分片后各数据报片最后能正确的重装成为原来的数据报

`Flags`: 标志, 3 bits, 分别是(RF, DF, MF), 目前只有DF, MF有效; DF(don't fragment), 置为 0 时表示可以分段, 置为 1 是不能被分段; MF(more fragment), 置为 0 时表示该数据段为最后一个数据段, 置为 1 时表示后面还有被分割分段

`Fragment offset`: 段偏移量, 13 bits, 指出较长的分组在分段后, 某段在原分组的相对位置; 也就是说相对用户字段的起点, 该片从何处开始; 段偏移以 8 bytes(有 3 位被 flags 占据)为偏移单位; 这就是, 每个分片的长度一定是 8 bytes(64 bits) 的整数倍

`Time to live`: 生存期(TTL), 8 bits, 用来设置数据报最多可以经过的路由器数; 由发送数据的源主机设置, 通常为 32、64、128 等; 每经过一个路由器, 其值减 1, 直到 0 时该数据报被丢弃

`Protcol`: 协议, 8 bits, 指明ip数据字段中的数据采用上层什么协议封装的; 常见的有ICMP(1)、IGMP(2)、TCP(6)、UDP(17)

`Header Checksum`: 头部校验和, 16 bits, 填充根据IP头部计算得到的校验和码; 计算方法是: 对头部中每个 16 bits 进行二进制反码求和, 但不和涉及头部后的数据字段

`Source Address`: 源ip地址, 32 bits, 如(192.168.1.2)

`Destination Address`: 目的ip地址, 32 bits, 如(192.168.1.3)

`Option`: 选项, `n*32 bits`; 用来定义一些可选项: 如记录路径、时间戳等; 但这些选项很少被使用, 同时并不是所有主机和路由器都支持这些选项; 可选项字段的长度必须是 32bits 的整数倍, 如果不足, 必须填充0以达到此长度要求; 根据IHL可以得到option的长度

`Data`: 数据, 不定长度, 但受限于数据报的最大长度(65536/2^16 bytes); 这是在数据报中要传输的数据; 它是一个完整的较高层报文或报文的一个分片

## TCP

![tcp](574e9258d109b3de49101ee8cebf6c81810a4c87.jpg)

`src port`: 源端口, 2 bytes, 是一个大于 1023 的 16 位数字, 由基于TCP应用程序的用户进程随机选择

`dst port`: 目的端口, 2 bytes, 指明接收者所用的端口号, 一般由应用程序来指定

`Sequence number`: 顺序号, 4 bytes, 用来标识从 TCP 源端向 TCP 目的端发送的数据字节流, 它表示在这个报文段中的第一个数据字节的顺序号; 如果将字节流看作在两个应用程序间的单向流动, 则 TCP 用顺序号对每个字节进行计数; 序号是 32bit 的无符号数, 序号到达 (2^32) － 1 后又从 0 开始; 当建立一个新的连接时,  SYN 标志变 1 , 顺序号字段包含由这个主机选择的该连接的初始顺序号 ISN (Initial Sequence Number)

`Acknowledgement number`: 确认号, 4 bytes, 包含发送确认的一端所期望收到的下一个顺序号; 因此, 确认序号应当是上次已成功收到数据字节顺序号加 1 ; 只有 ACK 标志为 1 时确认序号字段才有效

`Offset`: 报头长度, 4 bits, 给出报头中 32bit 字的数目 n (**data 部之前的报文长度(32 * n)**); 需要这个值是因为任选字段的长度是可变的; 这个字段占 4bits ,  即 TCP 最多有 60(15*4)bytes 的首部

`Resrvd`: 保留区域, 6 bits, 保留给将来使用, 目前必须置为 0

`Control Flags`(6 bits)控制位包括
- `URG`: 为 1 表示紧急指针有效, 为 0 则忽略紧急指针值
- `ACK`: 为 1 表示确认号有效, 为 0 表示报文中不包含确认信息, 忽略确认号字段; **作为数据帧**
- `PSH`: 为 1 表示是带有 PUSH 标志的数据, 指示接收方应该尽快将这个报文段交给应用层而不用等待缓冲区装满(一次作为完整通讯中的最后一帧)
- `RST`: 用于复位由于主机崩溃或其他原因而出现错误(error)的连接; 它还可以用于拒绝非法的报文段和拒绝连接请求; 一般情况下, 如果收到一个 RST 为 1 的报文, 那么一定发生了某些问题
- `SYN`: 同步序号, 为 1 表示连接请求, 用于建立连接和使顺序号同步(synchronize)
- `FIN`: 用于释放连接, 为 1 表示发送方已经没有数据发送了, 即关闭本方数据流

`Window Size`: 窗口大小, 2 bytes, 表示从确认号开始, 本报文的源方可以接收的字节数, 即源方接收窗口大小; 窗口大小是一个 16bit 字段, 因而窗口大小最大为 65535(2^16 - 1)

`Checksum`: 校验和, 2 bytes, 对整个的 TCP 报文段(包括 TCP 头部和 TCP 数据), 以 16 位字进行计算所得; 这是一个强制性的字段, 要求由发送端计算和存储, 并由接收端进行验证

`Urgent Pointer`: 紧急指针, 2 bytes, 是一个正的偏移量, 和顺序号字段中的值相加表示紧急数据最后一个字节的序号; TCP 的紧急方式是发送端向另一端发送紧急数据的一种方式; 只有当 URG 标志置 1 时紧急指针才有效

`Option and Pad`: 选项和填充, n*4 bytes, 常见的可选字段是最长报文大小 MSS(Maximum Segment Size) ; 每个连接方通常都在通信的第一个报文段(为建立连接而设置 SYN 标志的那个段)中指明这个选项, 它指明本端所能接收的最大长度的报文段; 选项长度不一定是 32 bits 字的整数倍, 所以要加 *填充位* , 使得报头长度成为整字数

`Data`: 数据, 不定长度, 为上层协议封装好的数据

### Golang-TCPConn 相关设置

设置后的相关错误返回:
- SetDeadline                     -- io time out
- SetKeepAlive/SetKeepAlivePeriod -- read: connection timed out

## UDP

![udp](b2de9c82d158ccbf4049fb531bd8bc3eb03541e5.jpg)

`src port`: 源端口, 2 bytes, 是一个大于 1023 的 16 bits 数字, 由基于UDP应用程序的用户进程随机选择;

`dst port`: 目的端口, 2 bytes, 指明接收者所用的端口号, 一般由应用程序来指定

`Length`: 数据总长度, 2 bytes, 指明了包括首部在内的 UDP 报文段长度

`Checksum`: 检验和, 2 bytes, 指整个 UDP 报文头和 UDP 所带的数据的校验和(也包括伪报文头); 伪报文头不包括在真正的 UDP 报文头中, 但是它可以保证 UDP 数据被正确的主机收到了

`Data`: 数据字段, 不定长度, 为上层协议封装好的数据

## TLS1.2

RTT: 请求往返一次为 1 RTT

![TLS1.2 通信概述](v2-9f717c2d57cc29e7f473a500e01f9f6e_r.jpg)

![RFC 5246 的 TLS1.2 握手报文](v2-4da93c38f9b07d63413a2d05dbb49d43_r.jpg)

### 1. Client-Hello 阶段: (TLS handshake)

如果 Client 有证书, 则使用证书私钥(进行 HMAC)签名

**支持的加密套件**
**尝试连接倒 Host 头**

等信息发送给 Server, 附上当前时间戳和随机生成的 Session ticket 1(Random Bytes)

相关报文:
- Client-Hello(TLS handshake): 供选择的加密套件, 时间戳, Session Ticket 1

### 2. Server-Hello 阶段: (TLS handshake)

使用 Server 的私钥签名

Server 支持的加密套件, 随机生成的 Session ticket 2(Random Bytes)

相关报文:
- Server-Hello(TLS handshake): 供选择的加密套件, 时间戳, Session Ticket 2
- Certificate(TLS handshake): 数字证书
- Server Key Exchange(TLS handshake): 可选, 与证书公钥有关的参数, 例如如果使用 ECDHE , 那么该包中包含: 椭圆曲线域参数, 公钥的值
- Server Hello Done(TLS handshake): 表示 Server-Hello 阶段的 报文已结束

### 3. Cipher-spec: (TLS handshake) 检测签名的有效性:

检验步骤:
1. 证书的有效期
2. 证书域名
3. 证书吊销情况(CRL+OSCP), 需要第三方权威认证 ?
4. 验证证书颁发机构, 如果颁发机构是中间证书, 则逐层向上检验(有效期/颁发机构/吊销状态)

以上需要全部通过, 否则 **警告**

若检查通过, 随机生成一份新的 Session Ticket 3, 配合之前的 Session Ticket 1 和 Session Ticket 2, 组合成 Session Key(C)

使用 Server 证书公钥, 通过协商的密钥交换算法 **加密**, 返回给 Server

Server 通过私钥解密, 得到 Session Ticket 3 , 也组合成 Session Key(S)

以上可知 Session Ticket 1 和 2 可以通过抓包获取, 但是 Session Ticket 3 在传输中是被加密的

相关报文:
- Client Key Exchange: Client 验证通过了 Server 证书, 向 Server 发送 Client 的公钥参数
- Certificate(TLS handshake): 可选, 数字证书

### 其余的握手报文

- Change Cipher Spec: TLS handshake ; 紧跟着 Key Exchange ; 用于通知对方已生成新的密钥, 请求更换到新的密钥

- Excrypted Handshake Message: TLS handshake ; 紧跟着 Key Exchange; 测试加密; 加密一段双方协定的内容, 用于测试加密效果

- New Session Ticket: TLS handshake ; Server 发送给 Client 一个 Session, 在 Session 有效时间内, 以最新的 密钥 进行通信
  - **至此, handshake 阶段结束, TLS 连接建立完成**

### 通讯阶段

- Application Data: 正常通讯数据

- Encrypted Alert: 意味着加密通信因为某些原因需要中断, 警告对方不要再发送敏感的数据

### 吊销检查

写进国际标准的吊销状态检查协议有两种: CRL / OCSP

**CRL 是一份全量的文件**

**OCSP 是 TCP 服务**


只内置 CRL 的证书是早年的证书, 不被新型浏览器信任

### 如何加速 TLS 过程

由于现有的成熟的工具, CS 通信不再成为瓶颈, 主要还是受 **网络延迟影响**:
- 使用常用的 CA 证书: CRL 证书的吊销状态, 是操作系统复用的, 在 TTL 时间内, 操作系统检查过对应 CA 的 CRL, 那么 CRL 均可避免二次下载; OCSP 也可受益于 DNS 解析
- 购买国内的 CA 证书, 注意不是中间证书分销商

## TLS1.3

![TLS1.3 的完整握手工作流](v2-dae1d935c4b2e18a354b0163eca91912_hd.png)

- `+`: 报文中值得注意的 extension
- `*`: 该内容可选
- `{}`: 该内容使用 `handshake_key` 加密
- `[]`: 该内容使用 `application_key` 加密

![TLS1.3 定义的 12 种握手报文](v2-aeeaa342c5062271b59457866a366d38_r.jpg)

TLS1.3 包含 3 个子协议:
- alert: 处理以下两个子协议的异常情况
- handshake: 握手, 协商使用的 TLS 版本 / 加密算法 / 数字认证的哈希 / 密钥, 双方的身份认证 / 完整性校验(避免中间人攻击)
- record: 一次通信 Session 中处理数据报文

### 术语

#### PSK(`pre_shared_key`) -- 密钥交换 + 身份认证 机制

- 0-RTT: 通过复用曾经的加密协商结果, 避免 RTT 握手; 同时, 允许在刚开始 TLS 密钥协商时, 就附上部分加密数据
- 为了实现 0-RTT, 双方在协商时就拥有一个对称密钥, 称为 PSK
- TLS 握手结束后, 服务器可以发送一个 NST(`new_session_ticket`) 的报文给客户端, 该报文中记录 PSK 的值、名字和有效期等信息, 双方下一次建立连接可以 **直接使用该 PSK 值作为初始密钥材料**
- **注**: 使用 PSK 密钥协商, 已经对双方的身份做了一定的认证, 不得再使用公钥证书的认证方式, 即 **CV / CR / CT 报文都不会再发送**

#### HKDF(`HMAC_based_key_derivation_function`) -- 密钥导出函数

- 初始密钥材料(PSK 或者 DH 密钥协商出的 Key) 由于随机性低, 需要 密钥导出函数从中获得 安全性更强 的密钥
- HKDF: 输入初始密钥材料, handshake 后的 hash , 输出安全性更强的 密钥
- 新密钥需要 `extract_then_expand` 的两个过程:
  - extract: 增加密钥材料的 **随机性**
  - expand: 通过随机性更强的密钥材料生成新密钥

#### AEAD(`Authenticated_Encrypted_associated_data`) -- 唯一保留的加密方式

- AEAD 集中完成 完整性校验 和 数据加密 两种功能
  - TLS 1.2 还支持 流加密 和 CBC 分组模式的块加密方法, 使用 MAC 来进行完整性校验数据, 这两种方式(流加密 和 CBC)均被证明有一定的安全缺陷
- AEAD 的局限性: 使用同一密钥加密的明文达到一定长度后, 就不能再保证密文的安全性; 为此, TLS1.3 引入 密钥更新机制(???)

### handshake 子协议

#### 加密套件

![TLS1.3 支持的加密套件](v2-d316bb33fcb276de3b33310704416458_r.jpg)

**Golang 12 支持的 TLS1.3 加密套件:**
- TLS_AES_128_GCM_SHA256 (0x1301) / 128 bits / FS
- TLS_CHACHA20_POLY1305_SHA256 (0x1303) / 256 bits / FS
- TLS_AES_256_GCM_SHA384 (0x1302) / 256 bits / FS

#### DHE 密钥协商机制

DH 参数分为:
- 有限域: g / p 的值
- 椭圆 DH: 椭圆曲线 和 基点 的值

#### post-authentication 机制

**TLS 1.3支持服务端在握手结束之后再对客户端发起身份的校验**

CH 报文中有一个扩展字段 post_handshake_auth , 如果 Client 发送了此字段, 则允许 Server 在握手阶段结束后再发起 Certificate Request; Client 会在收到 CR 之后再发送 CT、 CV 报文给 Server 进行身份认证

### record 子协议

![record 层的处理](v2-c5c420d6a87bc1bf08be84b64b86b727_r.jpg)

- 分片: TLS 拆分报文为多个 record , 对每个 record 进行单独的加密后转发, 规定了:
  - 单个 record 的长度
  - 使用不同密钥的消息不能在一个 record
  - 对不同的报文有不同的处理:
    - handshake message(HM): 在传递顺序的 HM record 时, 不得夹杂其他类型
	- alert: 每个alert报文都必须完整发送, 不得分片
	- application data: 无
- 加解密: AEAD 加密机制协商出的密钥加密 record, 伪代码:
  - AEADEncrypted = AEAD-Encrypt(write_key(即加密密钥), nonce, plaintext)
  - 加密后, record 层会在密文头部附加一小段明文信息, 来标识解密后明文长度
- TLS 1.3 允许 TLS 对消息报文填充来阻止攻击者获知传送消息的长度等信息

> 关于随机数的获取(如何保证随机数的唯一性):
>
> 双方维护一个 64 位的 sequence number, 连接建立和每一次密钥变更时, sequence number 置 0, 此后随传递的字节数增加
>
> 加密时将 sequence 扩充到 write_iv(也是由 write_key 导出的一个初始向量)的长度, 再与 write_iv 做异或计算得到 nonce

### alert 子协议

alert 层负责处理 TLS 连接过程中的各种异常情况, 对每种情况发送一个 alert 报文, 报文中附加一些错误处理需要的必要信息, TLS 1.3 中定义了 30 种 alert 报文
