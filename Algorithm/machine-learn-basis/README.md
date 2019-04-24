# 人工智能基础课

## 数学基础 -- 概率论

频率学派:

- 频率学派的概率: 可独立重复的随机试验中单个结果出现频率的 **极限**; 即试验的结果只包含有限个给本事件, 且每个基本事件发生的可能性 **相同**
- 频率学派认为先验分布是固定的, 模型参数依靠 **最大似然估计** 计算
- 假设, 是客观存在且不会改变的, 存在固定的先验分布

贝叶斯学派:

- 贝叶斯学派的概率: 随机事件的可信程度
- 贝叶斯学派认为先验分布是随机的, 模型参数依靠 **后验概率最大化** 计算
- 固定的先验分布是不存在的, 参数本身也是随机数

## 数理统计

- 数理统计, 通过可观察的样本反推断总体的性质
- 推断的工具是统计量, 统计量是样本的函数, 是个随机变量
- 参数估计通过随机抽取的样本来估计总体分布的未知参数, 包括点估计和区间估计
- 假设检验通过随机抽取的样本接受或拒绝关于总体的某个判断, 常用于估计机器学习模型的泛化错误率

## 凸集/凸函数/凸优化

凸集具有的几何性质是凸集中的任意两点都是 **无障碍可见的**