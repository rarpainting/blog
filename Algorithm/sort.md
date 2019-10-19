# 排序算法

![](https://zhuanlan.zhihu.com/p/34421623)

排序稳定:
- A1==A2 , 那么在排序前后, 两者 **顺序不变**

## 冒泡算法

- 时间: O(n^2)
- 稳定

## 选择排序

- 时间: O(n^2)
- 不稳定

## 插入排序

- 建立 **新序列**, 从后往前查找第一个 `<=value` 的值, 插入后面
- 时间: O(n^2)
- 稳定

## 快速排序

- 时间: O(nlogn)
- 两个方向, 其中从后往前的排序不稳定

## 归并排序

- 时间: O(nlogn)
- 稳定, 归并的稳定性可以通过 归纳法 确定

## 桶排序 / 基数排序

- 通过 **多种分类规则** 分类, 将一种规则下的各个类别做 compare, 合并, 以及分类
- 稳定

## 希尔排序

- 直接插入排序的改进
- 时间: O(n^(3/2))
- 不同的步长可能有不同的排序结果, 不稳定

## 梳排序

- 冒泡的改进
- 时间: O(nlogn)
- 不同的步长可能有不同的排序结果, 不稳定

## 堆排序

- 构建最小/最大堆(完全二叉) , 将 最小/最大 的值从头到尾构建即可
- 时间: O(nlogn)
- 堆构造后, 最值和尾值交换, 不稳定

## 鸡尾酒排序

- 冒泡的变形
- 先找最大冒泡到后面, 再找最小冒泡到前面, 以此循环
- 时间: O(n^2)
- 稳定