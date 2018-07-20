# Golang

## 陷阱

### 返回值

- 有名返回值 -- func f() (v Type) {} -- 在函数声明时已经被定义
- 匿名返回值 -- func f() Type {} -- 在 return 执行时被声明

结论:
defer 语句能访问有名返回值
      不能直接访问匿名返回值

### defer

defer 的 颗粒度 是 函数级 的, 即 defer 会在函数结束时调用, 而不在 代码块
      
### http 响应

resp, err := http.Get("https://api.ipify.io?format=content")

当发生 http 的重定向时, err 和 resp 都**不为空**
因此保险的做法:

```golang
resp, err := http.Get("https://api.ipify.io?format=content")
if resp != nil {
    defer resp.Body.Close()
}
if err != nil {
    fmt.Println("sorry")
    return
}
// do anything...
```

### 失败的类型断言

失败的类型断言 返回断言声明中 **类型** 的 "零" 值

### 阻塞的 goroutine 和资源泄漏

例子:

```golang
func First(query string, replicas ...Search) Result {
  c := make(chan Result)
  searchReplicas := func(i int) {c <- replicas[i](query)}
  for i := range replicas {
    go searchReplicas(i)
  }
  return <- c
}
```

该例子导致 chan 阻塞, 从而泄漏大量的 goroutine

解决方法:

1. 申请足够的 chan , 缓存所有的结果

```golang
func First(query string, replicas ...Search) Result {
  c := make(chan Result, len(replicas))
  //...
}
```

2. 添加一个 default 选项

```golang
func First(query string, replicas ...Search) Result {
  //...
  searchReplicas := func(i int) {
    select {
      case c <- replicas[i](query):
      default:
    }
  }
  //...
}
```

3. 添加另一个特殊的 chan 来终止 goroutine

```golang
func First(query string, replicas ...Search) Result {
  //...
  done := make(chan struct())
  defer close(done)
  searchReplicas := func(i int) {
    select {
      case c <- replicas[i](query):
      case <-done:
    }
  }
  //...
}
```

### 使用指针接受方法

