# C++

## Lambda

形如

```cpp
[capture list] (params list) mutable exception-> return type { function body }
```

- capture list: 捕获外部变量列表
- params list: 形参列表
- mutable 指示符: 用来说用是否可以修改捕获的变量
- exception: 异常设定
- return type: 返回类型
- function body: 函数体

| 捕捉方式       | 说明                                                                                          |
| :-             | :-                                                                                            |
| `[]`           | 	不捕获任何外部变量                                                                        |
| `[变量名, …]` | 默认以值得形式捕获指定的多个外部变量(用逗号分隔), 如果引用捕获, 需要显示声明(使用&说明符) |
| `[this]`       | 	以值的形式捕获 this 指针                                                                  |
| `[=]`          | 	以值的形式捕获所有外部变量                                                                |
| `[&]`          | 以引用形式捕获所有外部变量                                                                    |
| `[=, &x]`      | 	变量 x 以引用形式捕获, 其余变量以传值形式捕获                                            |
| `[&, x]`       | 	变量 x 以值的形式捕获, 其余变量以引用形式捕获                                            |

Lambda 表达式的参数限制:
- 参数列表中不能有默认参数(可惜了)
- 不支持可变参数
- 所有参数必须有参数名
