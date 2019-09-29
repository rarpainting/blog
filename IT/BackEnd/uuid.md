# UUID

128 位的唯一标识

## Format

16 进制表示, 36 个字符(32 字母数字 + 4 个 "-") , 格式 `8-4-4-4-12`

```shell
xxxxxxxx-xxxx-Mxxx-Nxxx-xxxxxxxxxxxx
```

M: 4 位用于表示 UUID 版本
N: 1-3 位表示不同哦 variant

## Version 1 (Data-time MAC address)

基于 时间戳及 MAC 地址 的 UUID 实现, 包括了 48 位的 MAC 地址和 60 位的时间戳

v1 为了保证唯一性, 当时间精度不够时, 会使用 13~14 位的 clock sequence 来扩展时间戳

Melissa 通过 UUID 提供的 MAC 地址, 反查到对应的主机 MAC

## Version 2 (Date-time MAC address)

DEC Security

## Version 3,5 (namespace name-based)

通过 Hash namespace 的标志符和名称生成

- V3 使用 MD5
- V5 使用 SHA-1
- 如果输入参数一致, 那么得到的 UUID 也会一致

## Version 4(random)

全随机 ??
