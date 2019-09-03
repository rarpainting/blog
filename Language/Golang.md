# Golang

## Happens Before

**应该以通信作为手段来共享内存**

如果要让一个对变量 v 的写操作 w 所产生的结果能够被对该变量的读操作 r 观察到, 那么需要同时满足如下两个条件

- 读操作 r 未先于写操作 w 发生

- 没有其他对此变量的写操作后于写操作 w 且 先于读操作 r 发生

即在一轮读写操作中 其他写操作 / 本次写操作 w / 本次读操作 r
顺序:

其他写操作 -> 本次写操作 w -> 本次读操作 r

## channel

CSP 思想

![channel](channel-tricks.png)

channel 的 happen before 规则:
- 第 n 个 `send` 一定 `happen before` 第 n 个 `receive finished`, 不管是 buffered channel 还是 unbuffered channel
- 对于 capacity 为 m 的 channel, 第 n 个 `receive` 一定 `happen before` 第 (n+m) `send finished`
- m=0 unbuffered. 第 n 个 `receive` 一定 `happen before` 第 n 个 `send finished`
- channel 的 close 一定 `happen before` receive 端得到通知, 得到通知意味着 receive 收到一个 channel close 后的零值, 而 close 之后的 send 则发生 panic


## go runtime

### 线程实现模型基础

`[M:1]` 用户级线程模型

- 线程的创建, 调度, 同步都由用户的进程户态的线程库处理
- 避免了 户态 和 核态 的切换调度, 处理速度快
- 线程空间仅对应一个内核调度实体, (线程阻塞时)无法调度到其他处理器, 无法通过优先级调度

`[1:1]` 内核级线程模型

- 用户线程对应各自的调度实体
- 能依赖内核的调度能力
- 对性能影响大

`[M:N]` 两级线程模型

- 拥有多个调度实体
- 多个线程对应一个调度实体
- 需要 核态 和 户态 同时参与调度, 设计复杂

### go 线程实现模型

| 中文名称              | 源码中的名称          | 作用域     | 需要说明                             |
| :--:                  | :--:                  | :--:       | :--:                                 |
| 全局 M 列表           | runtime.allm          | 运行时系统 | 能用于存放所有 M 的列表              |
| 全局 P 列表           | runtime.allp          | 运行时系统 | 能用于存放所有 P 的列表              |
| 全局 G 列表           | runtime.allg          | 运行时系统 | 能用于存放所有 G 的列表              |
| 调度器的空闲 M 列表   | runtime ' sched.midle | 调度器     | 被用于存放空闲 M 的列表              |
| 调度器的空闲 P 列表   | runtime ' sched.pidle | 调度器     | 被用于存放空闲 P 的列表              |
| 调度器的可运行 G 队列 | runtime ' sched.runq  | 调度器     | 被用于存放可运行 G 的队列            |
| 调度器的的自由 G 队列 | runtime ' sched.gfree | 调度器     | 被用于存放自由 G 的列表              |
| P 的可运行 G 队列     | runq                  | 本地 P     | 被用于存放当前 P 中的可运行 G 的队列 |
| P 的自由 G 列表       | gfree                 | 本地 P     | 被用于存放当前 P 中的自由 G 的列表   |

### 调度器字段

| 字段名称   | 数据类型 | 用途描述                                                 |
| :--:       | :--:     | :--:                                                     |
| gcwaiting  | uint32   | 作为垃圾回收任务被执行期间的辅助标记, 停止计数和通知机制 |
| stopwait   | int32    |                                                          |
| stopnote   | Note     |                                                          |
| sysmonwait | uint32   | 作为系统检测任务被执行期间的停止计数和通知机制           |
| sysmonnote | Note     |                                                          |

## Go Ast

### ast.Ident

词信息

- .NamePost Token.Post -- 位置
- .Name string -- 名字
- .Obj *Object 具体内容

### ast.Object

- .Kind ObjKind 全局类型
- .Name string 声明的名字
- .Decl interface{} 对应以下字段:
  - XssSpec
  - FuncDecl
  - LabeledStmt
  - AssignStmt
  - Scope
  - nil
- .Data interface{}
  - 特殊对象数据 / nil
- .Type interface{} --

## 陷阱

### 返回值

- 有名返回值 -- func f() (v Type) {} -- 在函数声明时已经被定义
- 匿名返回值 -- func f() Type {} -- 在 return 执行时被声明

结论:
`defer` 语句能访问有名返回值
      不能直接访问匿名返回值

### defer

`defer` 的 颗粒度 是 函数级 的, 即 defer 会在函数结束时调用, 而不在 代码块

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

其中, `interface{}`和 `slice` 的零值为 `nil`

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

4. (??) 指针构成的 "循环引⽤" 加上 runtime.SetFinalizer 会导致内存泄露

### 使用指针接受方法

### String

`len(string)` 的结果是 []byte 的结果

但是 `range string` 的结果是 []rune 的结果

### Map

**基于 Golang1.12**

```golang
// A header for a Go map.
type hmap struct {
	// Note: the format of the hmap is also encoded in cmd/compile/internal/gc/reflect.go.
	// Make sur\e this stays in sync with the compiler's definition.
	count     int // # live cells == size of map.  Must be first (used by len() builtin)
	flags     uint8
	B         uint8  // log_2 of # of buckets (can hold up to loadFactor * 2^B items)
	noverflow uint16 // approximate number of overflow buckets; see incrnoverflow for details
	hash0     uint32 // hash seed

	buckets    unsafe.Pointer // array of 2^B Buckets. may be nil if count==0.
	oldbuckets unsafe.Pointer // previous bucket array of half the size, non-nil only when growing
	nevacuate  uintptr        // progress counter for evacuation (buckets less than this have been evacuated)

	extra *mapextra // optional fields
}

// mapextra holds fields that ar e not present on all maps.
type mapextra struct {
	// If both key and value do not contain pointers and ar\e inline, then we mark bucket
	// type as containing no pointers. This avoids scanning such maps.
	// However, bmap.overflow is a pointer. In order to keep overflow buckets
	// alive, we stor\e pointers to all overflow buckets in hmap.extra.overflow and hmap.extra.oldoverflow.
	// overflow and oldoverflow ar e only used if key and value do not contain pointers.
	// overflow contains overflow buckets for hmap.buckets.
	// oldoverflow contains overflow buckets for hmap.oldbuckets.
	// The indirection allows to stor\e a pointer to the slice in hiter.
	overflow    *[]*bmap
	oldoverflow *[]*bmap

	// nextOverflow holds a pointer to a free overflow bucket.
	nextOverflow *bmap
}

const (
	// Maximum number of key/value pairs a bucket can hold.
	bucketCntBits = 3
	bucketCnt     = 1 << bucketCntBits // 8
)

// A bucket for a Go map.
// golang 将 hash 中的 key 和 value 分开存放, 操作更繁琐但是优化了对齐
type bmap struct {
	// tophash generally contains the top byte of the hash value
	// for each key in this bucket. If tophash[0] < minTopHash,
  // tophash[0] is a bucket evacuation state instead.
  // 保存每个 hash 结果的顶部字节(8bit)
  // 如果 tophash[0] < minTopHash, 表示 bucket 在迁移过程中
	tophash [bucketCnt]uint8
	// Followed by bucketCnt keys and then bucketCnt values.
	// NOTE: packing all the keys together and then all the values together makes the
	// code a bit mor e complicated than alternating key/value/key/value/... but it allows
	// us to eliminate padding which would be needed for, e.g., map[int64]int8.
	// Followed by an overflow pointer.
}
```

key 的计算公式:

|key  | hash                                      | hashtop                                     | bucket index                                            |
| :-: | :-:                                       | :-:                                         | :-:                                                     |
| key | `hash := alg.hash(key, uintptr(h.hash0))` | `top := uint8(hash >> (sys.PtrSize*8 - 8))` | `bucket := hash & (uintptr(1)<<h.B - 1)`，即 hash % 2^B |

#### 创建 -- makemap

`make(map[k]v, hint)`

```golang
func makemap(t *maptype, hint int, h *hmap) *hmap {
  // Architectur\e  Name              Maximum Value (exclusive)
  // ---------------------------------------------------------------------
  // amd64         TASK_SIZE_MAX     0x007ffffffff000 (47 bit addresses)
  // arm64         TASK_SIZE_64      0x01000000000000 (48 bit addresses)
  // ppc64{,le}    TASK_SIZE_USER64  0x00400000000000 (46 bit addresses)
  // mips64{,le}   TASK_SIZE64       0x00010000000000 (40 bit addresses)
  // s390x         TASK_SIZE         1<<64 (64 bit addresses)
  mem, overflow := math.MulUintptr(uintptr(hint), t.bucket.size)
  if overflow || mem > maxAlloc {
    hint = 0
  }

  // B = log_2(hint/13) + 1
  for overLoadFactor(hint, B) { B++ }

  // if B == 0, the buckets field is allocated lazily later (in mapassign)
  // 懒操作
  if h.B != 0 {
    h.buckets, nextOverflow = makeBucketArray(t, h.B, nil)
  }
}
```

#### 访问 -- mapaccess

```golang
// mapaccess1 returns a pointer to h[key], renturn **zero object**
func mapaccess1(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
// func mapaccess2(t *maptype, h *hmap, key unsafe.Pointer) (unsafe.Pointer, bool) {
// func mapaccessK(t *maptype, h *hmap, key unsafe.Pointer) (unsafe.Pointer, unsafe.Pointer) {

  // hash(key)
  hash := alg.hash(key, uintptr(h.hash0)) // Hash = hash(key, seed)
  // 获得 key 在桶的索引的偏移
  m := bucketMask(h.B) // (1 << h.B -1)
  b := (*bmap)(add(h.buckets, (hash&m)*uintptr(t.bucketsize)))

  if c := h.oldbuckets; c != nil {
    // 新旧桶不是相同尺寸, 则认为旧桶为新桶的一半
    if !h.sameSizeGrow() {
    }

    // 有旧桶, 则可能使用旧桶查找
    oldb := (*bmap)(add(c, (hash&m)*uintptr(t.bucketsize)))
    if !evacuated(oldb) {
      b = oldb
    }
  }

  // 不停寻找
  for ; b != nil; b = b.overflow(t) {
      if alg.equal(key, k) {
        v := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.valuesize))
        return v
      }
  }

  // 找不到的 key 会返回 zoreVal[0]
  return unsafe.Pointer(&zeroVal[0])
}
```

#### 分配 -- mapassign

- 与 `mapaccess` 操作相似, 为一个 key 分配槽
- 添加了写保护和扩容操作

```golang
func mapassign(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
  // 写保护
  if h.flags&hashWriting != 0 {
    throw("concurrent map writes")
  }

again: // 扩容, k/v 写入的操作

  // 如果需要做扩容, 则开始扩容
  if h.growing() {
    growWork(t, h, bucket)
  }

bucketloop: // 找到合适的 bucket
 for {
    // already have a mapping for key. Update it.
    // 空间已有, 尝试替换
    if t.needkeyupdate() {
      typedmemmove(t.key, k, key)
    }
  }

  // Did not find mapping for key. Allocate new cell & add entry.
  // 为新的 k/v allocate 新的空间

  // growing 标志没有设置, 但是太多的桶已经满了
  // 需要返回并且扩容
  if !h.growing() && (overLoadFactor(h.count+1, h.B) || tooManyOverflowBuckets(h.noverflow, h.B)) {
    hashGrow(t, h)
    goto again
  }

  // 适合的桶都已经满了, 需要生成新的桶
  if inserti == nil {
    // all current buckets ar\e full, allocate a new one.
    newb := h.newoverflow(t, b)
  }

  // stor\e new key/value at insert position
  if t.indirectkey() { ... }
  if t.indirectvalue() { ... }
  typedmemmove(t.key, insertk, key)

done: // 结束 assign 操作
}
```

#### 删除 -- mapdelete

#### 扩容 -- growWork

### math/rand

- 全局的 `rand.globalRand` 使用 `lockedSource` 为 source
- 而自建的 `rand.NewSource` 使用 无锁的 `rngSource` 为 source
- 因此, 在 多线程且无互斥需求 的情况下, 随机数生成器尽量使用 `NewSource`(即 `rngSource`)

### MySQL

#### `go-sql-driver/mysql`

- 同一个事务每一个 Query 的结果集用的都是同一个缓存, 必须把 Query 后的结果集清空才能再次执行

- Stmt
  - `DB.Prepare()` 会重新从连接池获取一条新的连接
  - `Tx.Prepare()` 重用 事务(Tx) 的连接

MySQL 8.0 使用 `caching_sha2_password` 作为默认加密插件, `go-sql-driver/mysql` 本身不支持该插件

因此需要在 MySQL 执行以下步骤, 修改 `'root'@'%'` 的加密方式:

```sql
ALTER USER `root`@`%` IDENTIFIED WITH mysql_native_password BY 'password';

-- 以下方法经验证已失效
-- UPDATE mysql.user SET password=PASSWORD("password") where host='%';
```

同时修改 Golang-MySQL-URL , 添加 `allowNativePasswords=true` 参数(BUG? 在 go-sql-driver 的文档中, 该参数是默认开启的, 然而需要明文添加)

可选: 修改 mysql config
```config
[mysqld]
default-authentication-plugin = mysql_native_password
```

#### github issues

- [Add support for sha256_password pluggable authentication #625](https://github.com/go-sql-driver/mysql/issues/625)
- [Unable to connect to MySQL 8 #785](https://github.com/go-sql-driver/mysql/issues/785)

### Decimal

[Go如何精确计算小数](https://imhanjm.com/2017/08/27/go%E5%A6%82%E4%BD%95%E7%B2%BE%E7%A1%AE%E8%AE%A1%E7%AE%97%E5%B0%8F%E6%95%B0-decimal%E7%A0%94%E7%A9%B6/)
