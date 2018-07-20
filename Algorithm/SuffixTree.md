# 后缀树 Suffix Tree

## 性质

- 存储所有的 n(n-1)/2 个后缀需要 O(n) 的空间, n 为文本(Text)长度
- 构建后缀树需要 O(dn) 的时间, d 为字符集的长度(Alphabet)
- 对模式(Pattern)的查询需要 O(dm) 时间, m 为 Pattern 的长度

### 各字符串匹配算法

|             | 模式预处理 Preprocess Pattern | 文本预处理 Preprocess Text | 空间 Space | 查询 Search Time |
| 暴力查询    |                               |                            | O(1)       | O(mn)            |
| Boyer Moore | O(m+d)                        |                            | O(d)       | O(n)*            |
| Suffix Tree |                               | O(n)                       | O(n)       | O(m)             |
| KMP         | O(n)                          |                            |            |                  |

## 构建 Suffix Tree

- 根据原始文本 Text 生成所有后缀的集合
- 将每个后缀作为一个单独的关键词, 构建一棵 Compressed Trie

