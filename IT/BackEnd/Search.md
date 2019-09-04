# Search --- 搜索引擎基础
## B树
一个[m]阶B树具有以下的特征
- 根节点(**Root Node**)至少(>=)有 ***2*** 个子女(**Children**) ===> 一个元素(**Key**)
- 每个中间节点(**Mid Node**)都包括 ***k-1*** 个元素(Key)和 ***k*** 个孩子(Children), 其中 ([m/2 <= ](??)k <= m)
- 每一个叶子节点(**Leaf Node**)都包含 ***k-1*** 个元素(Key),  其中 ([m/2 <= ](??)k <= m)
- 所有的叶子节点(**Leaf Node**)都位于同一层
- 每个节点(**Node**) 中的元素(**Key**)从小到大排序, 节点当中 ***k-1*** 个元素(**Key**)正好是 ***k*** 个孩子(**Children**)包含的元素(Key)的值域划分

## 文本相关性排序
### 概念
- Term -- 分词后的最小单位
- TF(Term Frequency) -- Term 在**单一文章**中出现的频率 -- Term 次数/总 Term 数
- DF(Document Frequency) -- 文档频率, 某个 Term 在**总文档**中出现的频率, 包含 Term 的文档数/总文档数
- IDF(Inverse Document Frequency) -- 逆文档频率, log(总文档数/包含 Term 的文档数)
### TF-IDF 相关性排序
IDF 的作用
- 去除高频词干扰
- 区分单一文档的重要的词
### 词距
联合查询时, Term 之间的距离
### 位置信息
Term 的位置, 如果在标题, 摘要命中的话明显应该比在正文中命中term的权重高
