# HTTP Header

## Request

### Accept

指定 **客户端** 能接受的内容类型

- Accept: text/plain,text/html

### Accept-Charset

指定 **客户端** 能接受的字符编码集

- Accept-Charset: iso-8859-5

### Accept-Encoding

指定浏览器可以支持的 Web 服务器返回的内容 **压缩编码** 类型

- Accept-Encoding: compress,gzip

### Accept-Language

指定浏览器可接受的本地化语言

- Accept-Language: en, zh

### Accept-Ranges

可以请求网页实体的一个或者多个子范围字段

- Accept-Ranges: bytes

### Authorization

http 授权的授权证书

- Autorization: Basic

QZ...==

### Cache-Control

指定请求和响应遵循的缓存机制

- Cache-Control: no-cache

### Connection

指定是否需要持久连接(HTTP1.1 默认进行持久连接)

- Connection: close

### Cookie

随请求一同发送的 Cookie 字段

- Cookie: $version=1;Skin=new;

### Content-Length

请求的内容长度

- Content-Length: 345

### Content-Type

请求的相对应的 MIME 信息

- Content-Type: application/json

### Date

请求发送的日期和时间

- Date: Tue, 15 Nov 2018 08:12:33 GMT

### Expect

请求的特定服务器行为

- Expect: 100-continue

### From

发出请求的用户的 Email

- From: user@email.com

### Host

指定主机的域名和端口

Host: www.domain.com

### If-Match

只有请求内容与实体相匹配才有效

- If-Match: “737060cd8c284d8af7ad3082f209582d”

### If-Modified-Since

如果请求的部分在指定时间之后被修改则请求成功，未被修改则返回304代码

- If-Modified-Since: Sat, 29 Oct 2010 19:43:31 GMT

### If-None-Match

如果内容未改变返回304代码，参数为服务器先前发送的Etag，与服务器回应的Etag比较判断是否改变

- If-None-Match: “737060cd8c284d8af7ad3082f209582d”

### If-Range

如果实体未改变，服务器发送客户端丢失的部分，否则发送整个实体。参数也为Etag

- If-Range: “737060cd8c284d8af7ad3082f209582d”

### If-Unmodified-Since

只在实体在指定时间之后未被修改才请求成功

- If-Unmodified-Since: Sat, 29 Oct 2010 19:43:31 GMT

### Max-Forwards

限制信息通过代理和网关传送的时间

- Max-Forwards: 10

### Pragma

用来包含实现特定的指令

- Pragma: no-cache

### Proxy-Authorization

连接到代理的授权证书

- Proxy-Authorization: Basic QWxhZGRpbjpvcGVuIHNlc2FtZQ==

### Range

只请求实体的一部分, 指定范围

- Range: bytes=500-999

### Referer

先前网页的地址, 当前请求网页紧随其后, 即来路

- Referer: http://www.zcmhi.com/archives/71.html

### TE

客户端愿意接受的传输编码, 并通知服务器接受接受尾加头信息

- TE: trailers,deflate;q=0.5

### Upgrade

向服务器指定某种传输协议以便服务器进行转换(如果支持)

- Upgrade: HTTP/2.0, SHTTP/1.3, IRC/6.9, RTA/x11

### User-Agent

User-Agent的内容包含发出请求的用户信息

- User-Agent: Mozilla/5.0 (Linux; X11)

### Via

通知中间网关或代理服务器地址, 通信协议

- Via: 1.0 fred, 1.1 nowhere.com (Apache/1.1)

### Warning

关于消息实体的警告信息

- Warn: 199 Miscellaneous warning

## Responses

### Accept-Ranges

表明服务器是否支持指定范围请求及哪种类型的分段请求

- Accept-Ranges: bytes

### Age

从原始服务器到代理缓存形成的估算时间(以秒计, 非负)

- Age: 12

### Allow

对某网络资源的有效的请求行为, 不允许则返回 405

- Allow: GET, HEAD

### Cache-Control

告诉所有的缓存机制是否可以缓存及哪种类型

- Cache-Control: no-cache

### Content-Encoding

web 服务器支持的返回内容压缩编码类型

- Content-Encoding: gzip

### Content-Language

响应体的语言

- Content-Language: en,zh

### Content-Length

响应体的长度

- Content-Length: 348

### Content-Location

请求资源可替代的备用的另一地址	

- Content-Location: /index.htm

### Content-MD5

返回资源的 MD5 校验值

- Content-MD5: Q2hlY2sgSW50ZWdyaXR5IQ==

### Content-Range

在整个返回体中本部分的字节位置	

- Content-Range: bytes 21010-47021/47022

### Content-Type
返回内容的MIME类型

- Content-Type: text/html; charset=utf-8

### Date

原始服务器消息发出的时间

- Date: Tue, 15 Nov 2010 08:12:31 GMT

### ETag

请求变量的实体标签的当前值

- ETag: “737060cd8c284d8af7ad3082f209582d”

### Expires

响应过期的日期和时间

- Expires: Thu, 01 Dec 2010 16:00:00 GMT

### Last-Modified

请求资源的最后修改时间

- Last-Modified: Tue, 15 Nov 2010 12:45:26 GMT

### Location

用来重定向接收方到非请求 URL 的位置来完成请求或标识新的资源

- Location: http://www.zcmhi.com/archives/94.html

### Pragma

包括实现特定的指令, 它可应用到响应链上的任何接收方

- Pragma: no-cache

### Proxy-Authenticate

它指出认证方案和可应用到代理的该 URL 上的参数

- Proxy-Authenticate: Basic

### refresh

应用于重定向, 在 5 秒之后重定向(由网景提出，被大部分浏览器支持)	

- Refresh: 5; url=http://www.zcmhi.com/archives/94.html

### Retry-After

如果实体暂时不可取，通知客户端在指定时间之后再次尝试

- Retry-After: 120

### Server

web 服务器软件名称	

Server: Apache/1.3.27 (Unix) (Red-Hat/Linux)

### Set-Cookie

设置Http Cookie

- Set-Cookie: UserID=JohnDoe; Max-Age=3600; Version=1

### Trailer

指出头域在分块传输编码的尾部存在

- Trailer: Max-Forwards

### Transfer-Encoding

文件传输编码	

- Transfer-Encoding:chunked

### Vary

告诉下游代理是使用缓存响应还是从原始服务器请求

- Vary: *

### Via

告知代理客户端响应是通过哪里发送的

- Via: 1.0 fred, 1.1 nowhere.com (Apache/1.1)

### Warning

警告实体可能存在的问题

- Warning: 199 Miscellaneous warning

### WWW-Authenticate

表明客户端请求实体应该使用的授权方案

- WWW-Authenticate: Basic
