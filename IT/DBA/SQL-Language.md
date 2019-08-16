# SQL Language

## 关系代数 和 SQL 语言

基础运算符:
- 选择(σ , selection) -- where
- 投影(π , projection) -- select...distinct
- 叉乘(x , cross-product) -- from
- 差(- , set-difference) -- R-S === 在 R 中不在 S 中的元组
- 并(υ , union) -- RuS === 在 R 和 S 中的所有元组

合成运算符:
- 交(∩ , intersection) -- R∩S === 在 R 中也在 S 中的元组
- 除(÷ , division) -- R÷S === R 中包含与 S 共有列相同, 其他列不同的关系实例 (??)
- 自然连接(⋈, natural join) -- 先做叉乘, 再选择 *公共属性一样*(??) 的关系实例

![](5741745-a9cb16a85e234f99.png)
![](5741745-4006f30276138875.png)

## 外键约束

- CASCADE: Delete row from parent and automatically delete matching rows in child, and so on in cascading manner
- SET NULL: Delete row from parent and set FK column(s) in child to NULL. Only valid if FK columns are NOT NULL
- SET DEFAULT: Delete row from parent and set each component of FK in child to specified default. Only valid if DEFAULT specified for FK columns
- NO ACTION: Reject delete from parent. This is the default setting if ON DELETE rule is omitted

