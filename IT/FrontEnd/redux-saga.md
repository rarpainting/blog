# redux-saga

## Effect 创建器

- 例如 { take put call apply cps } 称为 Effect 创建器

- 所有 Effect 创建器都返回一个 **纯文本 JS 对象** 但不执行任何其他操作, 执行由 middleware 在所有已注册的迭代过程中执行

### take(pattern)

创建一条 Effect 描述信息, 指示 middleware 等待 Store 上指定的 Action
此时 Generator 会暂停, 直到一个与 pattern 匹配的 Action 被发起

middleware 会**拉取(pull)** 对应的 action

pattern 规则:

- 参数为空, 或传进 '*' , 则匹配所有被发起的 action
- 参数为函数, 则当 pattern(action) 返回 true 时匹配当前 action
- 参数为字符串, 则当 action.type === pattern 时被匹配
- 参数为数组, 则对 数组的所有项, 都进行 action.type === pattern 的操作

### put(action)

创建一条 Effect 描述信息, 指示 middleware 发起一个 action

- put 是异步执行的, 即该任务不会立即执行

### call(fn, ...arg)

创建一条 Effect 描述信息, 指示 middleware 调用 fn 函数并以 args 为参数

- fn: Fuction -- 一个 **Generator** 函数, 或者返回 **Promise** 的普通函数
- args: Array -- 数组, 作为 fn 的参数

**注意:**
- 若返回一个 Generator 对象, middleware 会执行它; 若存在子 Generator, 则执行子 Generator, 暂停父 Generator 直到子 Generator 正常结束, 若子 Generator 抛出错误, 则被父 Generator 捕捉

- 若返回一个 Promise, middleware 会暂停直到该 Promise 被 resolve; 若 Promise 被 reject ,则在 Generator 抛出一个错误

### call([context, fn], ...args)

通过 context 为 fn 制定 this 上下文

### apply(context, fn, args)

类似于 call([context, fn], ...args)

### cps(fn, ...args)

创建一条 Effect 描述信息, 指示 middleware 以 Node 风格调用 fn 函数

- fn: Function -- Node 风格函数. fn 结束后调用附加的回调函数, 该回调接受两个参数, callback(报错信息, 成功的结果)
- args: Array -- fn 的参数

注意:

### cps([context, fn], ...args)

指定上下文

### fork(fn, ...args)

创建一条 Effect 描述信息, 指示 middleware 以**无阻塞调用**方式执行 fn

**注意:**
- fork 类似于 call , 能用来调用 普通函数和 Generator 函数. 但 fork 是无阻塞调用, 在等待 fn 结果时, middleware 不会暂停 Generator
- fork 类似于 race 是一个中心化的 Effect, 管理 Sagas 间的并发
- yield fork([context, fn], ...args) -- 返回一个 Task 对象, 内含某些有用的方法和属性

### fork([context, fn], ...args)

指定上下文

### join(task)

创建一条 Effect 描述方法, 指示 middleware 等待之前的 fork 任务返回结果

- task: Task -- fork 指令返回的 Task 对象

### cancel(task)

创建一条 Effect 描述信息, 指示 middleware 取消执行中的 fork 任务

- task: Task -- fork 指令返回的 Task 对象

**注意:**

- 取消执行中的 Generator , 会抛出 SagaCancellactionException
- 取消会向下传播, 但如果当前 Effect 不处理取消异常, 则异常将不会冒泡至父级
- cancel 是 **无阻塞调用**

### select(selector,  ...args)

指示 middleware 调用提供的选择器获取 Store state 的数据

- select: Function -- (state,  ...args) => args . 通过当前 state 和部分参数, 返回当前 Store state 上的部分数据
- args: Array -- 可选参数, 传递给 selector
- 若 select 参数为空, 则 Effect 将获得整个 state(即相对于调用 getState())

