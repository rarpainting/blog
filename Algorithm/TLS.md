# TLS 加密

## 密码套件

在 TLS1.0 - TLS 2.0 中
每个密码套件的名称都定义了
密钥交换算法 批量加密算法 讯息鉴别码算法 伪随机算法

以 TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256 为例

1. **密钥交换算法**: ECDHE_RSA 决定客户端与服务端握手时的**身份认证** -- 正向加密部分
2. **批量加密算法**: AES_128_GCM 加密信息流 包括密钥大小及显式/隐式初始向量(密码学伪随机数)
3. **消息认证码算法**: SHA256 散列消息流中的每个数据块 防止消息流被篡改

## session ticket key -- session 重用

## NPL/ALPL

NPL -- 服务端返回上层协议
ALPL -- 客户端发送应用层协议, 通过服务端回显以判断上层协议, 以便优化上层协议

## SETS(signed certificate timestamp support)

第三方审计服务 防止部分 CA 机构恶意签发证书

## Secure-Renegotiation -- 安全重协商

在握手完成后的通讯阶段, 由于上层应用无法判断两次加密环境的数据, 发出再次协商密钥信息的要求

由于协商阶段存在明文传输, 该阶段可能导致非法报文注入

## 压缩算法

在 TLS 使用压缩算法, 可能导致 CRIME , BREACH 等攻击

## OCSP stapling

OCSP: 在线证书验证协议 -- 内包含 OCSP 验证地址
一般而言, 服务器定期执行 OCSP 请求, 缓存结果, 如果客户端支持 OCSP stapling 则发送缓存结果
该结果被 CA 签名 服务器无法伪造该结果

## SNI -- 服务器名称指示

(用于 CDN ??)

