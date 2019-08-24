# TIDB

## 概念

- Timestamp Oracle (TO)
- The Server Oracle (TSO)
- Commit Table (CT)
- Shadow Cells (SCs)
- Transactional Clients (TCs)

![架构](tidb-architecture.png)

- TiDB-Server:
  - 负责接收 SQL 请求, 处理 SQL 相关的逻辑, 并通过 PD 找到存储计算所需数据的 TiKV 地址, 与 TiKV 交互获取数据, 最终返回结果
  - TiDB 无状态, 本身不存储数据, 只负责计算
  - 可无限水平扩展, 可通过均衡负载组件(LVS/HAProxy/F5) 对外提供统一的接入地址
- PD-Server:
  - 集群管理模块
  - 存储集群的元数据
  - 对 TiKV 集群进行调度和负载均衡(数据迁移/Raft group leader 迁移)
  - 分配全局唯一且递增的事务 ID , 提供全局时钟服务
  - PD 通过 Raft 协议保证数据安全性; 因此建议部署奇数个 PD 节点
- TiKV-Server:
  - 分布式, 带事务的 Key-Value 存储引擎
  - 存储数据的基本单位是 Region, Region 负责存储一个 Key Range(从 StartKey 到 EndKey 的左闭右开区间)
  - 以 Raft 协议做复制, 保证数据一致性和容灾; 副本以 Region 为单位调度管理; 多个 Region 以 (?) 组成 Raft Group, 互为副本
- TiSpark
  - 用于解决复杂的 OLAP 需求, 将 Spark SQL 直接运行在 TiDB 存储层上

特性:
- 水平扩展:
  - 计算能力: TiDB
  - 存储能力: TiKV
  - 调度能力: PD

## TIDB

### 存储

- RocksDB: 存储引擎
- Raft: 分布式一致性
  - Leader 选举
  - 成员变更
  - 日志复制

TiKV 写数据到 RocksDB 的过程:

![TiKV 写到 RocksDB](tikv2rocksdb.png)

#### Region

![Region 的 Raft Group](region-raft-group.png)

- 对于一个 KV 系统, TiKV 以一个连续的 Key 组作为一个 Region , 保存在一个存储节点上
- Region 的规则规定:
  - 以 Region 为单位, 将数据分散在集群中所有的节点上, 并且尽量保证每个节点上服务的 Region 数量差不多
  - 以 Region 为单位做 Raft 的复制和成员管理

#### 事务 -- Percolator 模型

Percolator 的两阶段提交
- `Client.Set()` 数据仅被缓存, 而不做刷写
- 在第二阶段, 只要有一个 percolator 事务的 [primary row 的 lock 字段] 被清除并且 [write record] 写好(这两个动作是在同一个 bigtable 事务中原子完成的), 那么就认为事务提交成功

### 计算

#### 数据编码规则

每行数据的编码规则:

```shell
Key: tablePrefix{tableID}_recordPrefixSep{rowID}
Value: [col1, col2, col3, col4]
```

(Unique )Index 数据的编码规则:

```shell
Key: tablePrefix{tableID}_indexPrefixSep_indexID_indexedColumnsValue
Value: rowID
```

(Ununique ) Index 数据的编码规则:

```shell
Key: tablePrefix{tableID}_indexPrefixSep_indexID_indexedColumnsValue_rowID
Value: null
```

上文的部分源码:

```go
var(
	tablePrefix     = []byte{'t'}
	recordPrefixSep = []byte("_r")
	indexPrefixSep  = []byte("_i")
)
```


#### SQL 运算

认为 SQL 可以映射到 KV , 那么一般的 SQL 运算:
1. 构建 Key-Range: 由于表中的 RowID 都在 [0, MaxInt64) 这个范围; 用 0 和 MaxInt64 根据 Row 的 Key 编码规则, 就能构造出一个 [StartKey, EndKey) 的左闭右开区间
2. 扫描 Key-Range
3. 过滤数据
4. 计算 Count

优化:
- 为了减少 RPC 开销, 将计算往 存储节点 移

![sql 计算](sql-compute.png)

#### SQL 架构

![SQL 架构](sql-arch.png)

### 调度

#### 调度需求

调度的基本要求:
- 副本数量不能多也不能少
- 副本需要分布在不同的机器上
- 新加节点后, 可以将其他节点上的副本迁移过来
- 节点下线后, 需要将该节点的数据迁移走

调度的优化需求:
- 维持整个集群的 Leader 分布均匀
- 维持每个节点的储存容量均匀
- 维持访问热点分布均匀
- 控制 Balance 的速度, 避免影响在线服务
- 管理节点状态, 包括手动上线/下线节点, 以及自动下线失效节点

#### 调度的基本操作(Raft)

- 增加一个 Replica(AddReplica)
- 删除一个 Replica(RemoveReplica)
- 将 Leader 角色在一个 Raft Group 的不同 Replica 之间 transfer(TransferLeader)

#### 信息收集

TiKV 集群定期向 PD 汇报

**每个 TiKV 节点会定期向 PD 汇报节点的整体信息**:
- 总磁盘容量
- 可用磁盘容量
- 承载的 Region 数量
- 数据写入速度
- 发送 / 接受的 Snapshot 数量(Replica 之间可能会通过 Snapshot 同步数据)
- 是否过载
- 标签信息(标签是具备层级关系的一系列 Tag)

**每个 Raft Group 的 Leader 会定期向 PD 汇报信息**:
- Leader 的位置
- Followers 的位置
- 掉线 Replica 的个数
- 数据写入 / 读取的速度

#### 调度策略

## SQL 流程

### 协议层

![sql 流程](sql-process.png)

![sql 协议层简析](SQL-流程-协议层简析.png)

### SQL 处理

![sql SQL 处理](SQL-流程-SQL处理.png)

### 执行 SQL 计划

TiDB 的执行引擎是以 Volcano 模型运行的

如果以下 SQL 语句的执行计划

```sql
SELECT c1 FROM t WHERE c2 > 1;
```

![运行执行器](executeStatement.png)

#### Volcano Optimizer

基于成本(Cost)的优化算法

"成本最优假设" 认为, 如果一个方案是由几个局部区域组合而成, 各个局部 "相似" , 那么在计算总成本时, 我们只考虑每个局部目前已知的最优方案和成本即可

![Cumulative Cost](v2-53cbb1026cfb70310b022a884988c2ee_hd.jpg)

Calcite 实现的 Volcano Optimizer 支持以下三种终止条件:
- 时钟: 使用最大迭代计数或最大物理执行时间作为限制
- 成本阈值: 当优化方案的成本低于某个阈值是否结束算法(相比原始成本或固定值)
- 规则穷尽: 当无法再应用规则获得 新的关系代数结构 的时候结束算法

注意事项:
- 在最初的 Volcano Optimizer 论文中, 算法存在 逻辑优化 和 物理优化 两个步骤; 但是在后续的 Cascades 论文和 Calcite 的实现中, 逻辑变换的规则和物理变换的规则没有本质差别, 两者在一轮优化中同时使用, 以快速从 逻辑表示 转换为 物理执行
- Calcite 的 `Cost` 有 rows / IO / CPU 等字段, 但是计算依然以 rows 为基准, 因此需要考虑把其他方面的成本转换为行数
- 在关系代数树上查找 匹配的结构 是优化过程中最频繁的操作. Calcite 实现: 如果一个节点和每一匹配模式的根节点相互匹配, 则从该节点进行一次校验
- Calcite 默认并不进行枚举式优化方案计算, 而是结合启发式计算进行有限的搜索

各种优化方式:
- Vectorized Execution: 元素在关系代数节点之间的 获取批量化 以及利用 SIMD 指令集 优化, 借以提高 **并行性**
- Query Compilation: 通过 LLVM 等工具将优化后的执行方案编译成机器码
- 利用索引和物化视图

## Insert 概览

以以下语句为例

```sql
INSERT INTO t VAULES ("pingcap001", "pingcap", 3);
```

```golang
// .../parser/ast/dml.go
type InsertStmt {
  IsReplace   bool
  IgnoreErr   bool
  Table       *TableRefsClause
  Columns     []*ColumnName
  Lists       [][]ExprNode
  Setlist     []*Assignment
  Priority    mysql.PriorityEnum
  OnDuplicate []*Assignment
  Select      ResultSetNode
}

// .../tidb/planner/core/planbuilder.go
func (b *PlanBuilder) Build(ctx context.Context, node ast.Node) (Plan, error)
...
	case *ast.InsertStmt:
		return b.buildInsert(ctx, x)
...

// .../tidb/planner/core/plan.go
type Plan interface {
	// Get the schema.
	Schema() *expression.Schema
	// Get the ID.
	ID() int
	// Get the ID in explain statement
	ExplainID() fmt.Stringer
	// replaceExprColumns replace all the column reference in the plan's expression node.
	replaceExprColumns(replace map[string]*expression.Column)

	context() sessionctx.Context

	// property.StatsInfo will return the property.StatsInfo for this plan.
	statsInfo() *property.StatsInfo
}
// .../tidb/planner/core/common_plans.go
type Insert struct {
	baseSchemaProducer

	Table       table.Table
	tableSchema *expression.Schema
	Columns     []*ast.ColumnName
	Lists       [][]expression.Expression
	SetList     []*expression.Assignment

	OnDuplicate        []*expression.Assignment
	Schema4OnDuplicate *expression.Schema

	IsReplace bool

	// NeedFillDefaultValue is true when expr in value list reference other column.
	NeedFillDefaultValue bool

	GenCols InsertGeneratedColumns

	SelectPlan PhysicalPlan
}

// .../tidb/executor/build.go
func (b *executorBuilder) build(p plannercore.Plan) Executor
...
	case *plannercore.Insert:
		return b.buildInsert(v)
...

// .../tidb/executor/executor.go
type Executor interface {
	base() *baseExecutor
	Open(context.Context) error
	Next(ctx context.Context, req *chunk.Chunk) error
	Close() error
	Schema() *expression.Schema
}
// .../tidb/executor/insert.go
type InsertExec struct {
        *InsertValues
        OnDuplicate []*expression.Assignment
        Priority    mysql.PriorityEnum
}

// .../tidb/executor/insert.go
insertExec.Next
↓
func (e *InsertValues) insertRows(ctx context.Context,
  exec func(ctx context.Context, rows [][]types.Datum) error) (err error)
↓
insertExec.exec
...
		for _, row := range rows {
			if _, err := e.addRecord(row); err != nil {
				return err
			}
    }
...

// .../tidb/executor/insert_common.go
type InsertValues struct {
        SelectExec Executor

        Table   table.Table
        Columns []*ast.ColumnName
        Lists   [][]expression.Expression
        SetList []*expression.Assignment

        GenColumns []*ast.ColumnName
        GenExprs   []expression.Expression

        // Has unexported fields.
}

func (e *InsertValues) addRecord(row []types.Datum) (int64, error)
...
	h, err := e.Table.AddRecord(e.ctx, row)
...

// .../tidb/table/table.go
type Table interface {
...
	// Indices returns the indices of the table.
  Indices() []Index
...
	// AddRecord inserts a row which should contain only public columns
  AddRecord(ctx sessionctx.Context, r []types.Datum, opts ...AddRecordOption) (recordID int64, err error)
...
}

// .../tidb/table/tables/tables.go
type tableCommon struct {
	tableID int64
	// physicalTableID is a unique int64 to identify a physical table.
	physicalTableID int64
	Columns         []*table.Column
	publicColumns   []*table.Column
	writableColumns []*table.Column
	writableIndices []table.Index
	indices         []table.Index
	meta            *model.TableInfo
	alloc           autoid.Allocator

  // recordPrefix / indexPrefix -- generated using physicalTableID
	recordPrefix kv.Key
	indexPrefix  kv.Key
}

// 索引创建
// addIndices adds data into indices.
func (t *tableCommon) addIndices(ctx sessionctx.Context, recordID int64, r []types.Datum, rm kv.RetrieverMutator,
  opts []table.CreateIdxOptFunc) (int64, error)
// .../tidb/table/tables/index.go
Index.Create(ctx sessionctx.Context, rm kv.RetrieverMutator, indexedValues []types.Datum, h int64, opts ...CreateIdxOptFunc) (int64, error)

// k/v 创建

tableCommon.AddRecord()
...
  key := t.RecordKey(recordID) // get key
  writeBufs.RowValBuf, err = tablecodec.EncodeRow(sc, row, colIDs, writeBufs.RowValBuf, writeBufs.AddRowValues) // get value
  txn.Set(key, value) // set into storage engine by transaction
...
```

## SQL Parser 的实现

- 词法解析 -- [lexer.go](https://github.com/pingcap/tidb/blob/source-code/parser/lexer.go)
- 语法解析 -- goyacc -> [parser.y](https://github.com/pingcap/tidb/blob/source-code/parser/parser.y)

### definitions

```yacc
%union {
	offset    int // offset
	item      interface{}
	ident     string
	expr      ast.ExprNode
	statement ast.StmtNode
}
```

### rules

TODO:

## Select 概览

```golang
// .../parser/ast/dml.go
type SelectStmt struct {
	*SelectStmtOpts
	Distinct    bool
	From        *TableRefsClause
	Wher e       ExprNode
	Fields      *FieldList
	GroupBy     *GroupByClause
	Having      *HavingClause
	WindowSpecs []WindowSpec
	OrderBy     *OrderByClause
	Limit       *Limit
	LockTp      SelectLockType
	TableHints []*TableOptimizerHint
	IsAfterUnionDistinct bool
	IsInBraces  bool
}

// .../tidb/planner/core/logical_plan_builder.go
func (b *PlanBuilder) buildSelect(ctx context.Context, sel *ast.SelectStmt) (p LogicalPlan, err error) {
...
	if sel.Wher e != nil {
		p, err = b.buildSelection(ctx, p, sel.Where, nil)
	}
...
}
func (b *PlanBuilder) buildSelection(ctx context.Context, p LogicalPlan, wher e ast.ExprNode, AggMapper map[*ast.AggregateFuncExpr]int) (LogicalPlan, error)

// .../tidb/planner/core/plan.go
type LogicalPlan interface {
	Plan
	PredicatePushDown([]expression.Expression) ([]expression.Expression, LogicalPlan)
	PruneColumns([]*expression.Column) error
	DeriveStats(childStats []*property.StatsInfo) (*property.StatsInfo, error)
	MaxOneRow() bool
	Children() []LogicalPlan
	SetChildren(...LogicalPlan)
	SetChild(i int, child LogicalPlan)
}

// .../tidb/planner/core/logical_plans.go
type LogicalSelection struct {
	baseLogicalPlan
	Conditions []expression.Expression
}

// .../tidb/session/session.go
func (s *session) Execute(ctx context.Context, sql string) (recordSets []sqlexec.RecordSet, err error) {
	...
	func (s *session) execute(ctx context.Context, sql string) (recordSets []sqlexec.RecordSet, err error) {
		...
		stmt, err := compiler.Compile(ctx, stmtNode)
		==> func (c *Compiler) Compile(ctx context.Context,
		stmtNode ast.StmtNode) (*ExecStmt, error) {
			...
			planner.Optimize()
			==> func Optimize(ctx context.Context, sctx sessionctx.Context, node ast.Node, is infoschema.InfoSchema) (plannercore.Plan, error) {
                plannercore.TryFastPlan(sctx, node)
				...
				plannercore.DoOptimize(ctx, builder.GetOptFlag(), logic)
			}
			...
		}
		...
	}
	...
}


// .../tidb/planner/core/optimizer.go
func DoOptimize(ctx context.Context, flag uint64, logic LogicalPlan) (PhysicalPlan, error) {
	logic, err := logicalOptimize(ctx, flag, logic)
	...
	physical, err := physicalOptimize(logic)
	...
	finalPlan := postOptimize(physical)
	...
}

// 逻辑优化 基于规则的优化
func logicalOptimize(ctx context.Context, flag uint64, logic LogicalPlan) (LogicalPlan, error)
// 物理优化 基于代价的优化
func physicalOptimize(logic LogicalPlan) (PhysicalPlan, error)

// TODO:
```

以上是 Select 的流程, 下面重点关注 `logicalOptimize`, `physicalOptimize` 的实现流程

### 逻辑优化 -- RBO/rule based optimization

```golang
var optRuleList = []logicalOptRule{
	&columnPruner{},
	&buildKeySolver{},
	&decorrelateSolver{},
	&aggregationEliminator{},
	&projectionEliminater{},
	&maxMinEliminator{},
	&ppdSolver{},
	&outerJoinEliminator{},
	&partitionProcessor{},
	&aggregationPushDownSolver{},
	&pushDownTopNOptimizer{},
	&joinReOrderSolver{},
}

// 逻辑优化 基于规则的优化
func logicalOptimize(ctx context.Context, flag uint64, logic LogicalPlan) (LogicalPlan, error)
```

以下 RBO 理论部分看 [PostgreSQL](IT/DBA/PostgreSQL.md)

部分重要函数:

```golang
// 从表达式中提取列及其信息
func ExtractColumnsFromExpressions(result []*Column, exprs []Expression, filter func(*Column) bool) []*Column
```

#### 列裁剪

```golang
func (p *LogicalJoin) PruneColumns(parentUsedCols []*expression.Column) error
```

#### 最大最小消除

#### 投影消除

```golang
func (pe *projectionEliminater) eliminate(p LogicalPlan, replace map[string]*expression.Column, canEliminate bool) LogicalPlan
```

#### 谓词下推

```golang
func (p *baseLogicalPlan) PredicatePushDown(predicates []expression.Expression) ([]expression.Expression, LogicalPlan)
```

#### 构建节点属性

主要是构建 `unique key` 和 `MaxOneRow` 属性

如果一个算子满足:
- 该算子的子节点是 `MaxOneRow` 算子
- 该算子是 `Limit 1`
- `Selection`: 且满足 `unique_key = CONSTANT`
- `Join`: 左右节点是 `MaxOneRow`

以上情况保证了 **该节点的输出肯定只有一行** , 因此可以设置该节点为 `MaxOneRow`

### 物理优化 -- CBO/cost based optimization

![逻辑算子与物理算子对应](2.png)

以以下 SQL 为例:

```sql
> select sum(s.a),count(t.b) from s join t on s.a = t.a and s.c < 100 and t.c > 10 group bys.a
```

![前面流程](3.jpeg)

![完整流程](4.jpeg)

#### 代价评估

## Hash Join

### Classic Hash Join

构建哈希表的连接关系称为 "构建(build)" 输入, 而另一个输入称为 "探测(probe)" 输入

1. For each tuple ${\displaystyle r}$ in the build input ${\displaystyle R}$
  1. Add ${\displaystyle r}$ to the in-memory hash table
  2. If the size of the hash table equals the maximum in-memory size:
    1. Scan the probe input ${\displaystyle S}$, and add matching join tuples to the output relation
    2. Reset the hash table, and continue scanning the build input ${\displaystyle R}$
2. Do a final scan of the probe input ${\displaystyle S}$ and add the resulting join tuples to the output relation

翻译:
- 如果 Hash Join 小于 最大可用内存 , 则将 匹配的连续元组 写入到 Hash Join memory 中
- 如果超过 最大可用内存 , 则在写满可用内存后, 删除已构建的元组, 从当前元组点重新构建(??)

适合于小表写入

### Grace Hash Join

特点:
- 首先 Hash R 和 S 元组并为此构建分区, 并将分区写入到 template disk file

### TiDB 的 Hash Join

- Build 阶段, 对 Inner 表建哈希表
- Probe 阶段, 对由 Outer 表驱动执行 Join 过程

Hash Join 实现:
- Main Thread, 一个, 负责:
  - 读取所有的 Inner 表数据
  - 根据 Inner 表数据构造哈希表
  - 启动 `Outer Fetcher` 和 `Join Worker` 开始后台工作, 生成 Join 结果, 各个 goroutine 的启动过程由 `fetchOuterAndProbeHashTable` 这个函数完成
  - 将 `Join Worker` 计算出的 Join 结果返回给 NextChunk 接口的调用方法
- `Outer Fetcher`: 一个, 负责读取 Outer 表的数据并分发给各个 `Join Worker`
- `Join Worker`: 多个, 负责查哈希表、Join 匹配的 Inner 和 Outer 表的数据, 并把结果传递给 Main Thread

#### 关键函数

```golang
func (e *HashJoinExec) fetchOuterAndProbeHashTable(ctx context.Context)
```

```golang
func (e *HashJoinExec) runJoinWorker(workerID uint)
```

```golang
func (e *HashJoinExec) waitJoinWorkersAndCloseResultChan()
```

# [builddatabase](https://github.com/ngaut/builddatabase)

## TiDB 的异步 schema 变更实现

### 概念

- 元数据记录: 为了简化设计, 引入 system database 和 system table 来记录异步 schema 变更过程中的一些元数据
- state: 根据 F1 的异步 schema 变更过程, 中间引入了一些状态, 该状态和 columns, index, table 和 database 绑定
  - 主要包括 none, delete-only, write-only, write-reorganization, public
  - 创建操作的状态与顺序相反(??), write-reorganization 改为 delete-reorganization
- Lease: 同时刻系统所有节点的 schema 最多只有 **两** 个不同版本
  - 一个租期内每个正常节点都主动加载 schema 信息; 如果不能租期内正常加载, 该节点自动退出系统
  - 即: **确保整个系统的所有节点都已经从某个状态更新到下个状态, 需要 2 倍的租期时间**
- Job: 每个单独的 DDL (Data Defined Language)操作都可看作一个 job
  - 在一个 DDL 操作开始时, 会将此操作封装成一个 job 并存放到 job queue, 等此操作完成时, 会将此 job 从 job queue 删除, 并在存入 history job queue, 便于查看历史 job
- Worker: 每个节点都有一个 worker 用于处理 job
- Owner
  - 一个独立的系统只有一个节点的 worker 能当选 owner
  - owner 角色有任期, owner 的信息会存储在 KV 层中, worker 定期获取 KV 层中的 owner 信息, 如果其中 ownerID 为空, 或者当前的 owner 超过了任期, 则 worker 可以尝试更新 KV 层中的 owner 信息(设置 ownerID 为自身的 workerID); 如果更新成功, 则该 worker 称为 owner
  - 这个用来确保整个系统同一时间只有一个节点在处理 schema 变更
- Background operations: 主要用于 delete reorganization 的优化处理; 总共引入了 background job, background job queue, background job history queue, background worker 和 background owner

### 变更流程

![结构流程图](1.jpg)

#### 模块

- TiDB Server: TiDB SQL 层中涉及异步 schema 变更的基本模块
- load schema: 每个节点启动时创建的 goroutine , 用于节点到达租期后加载 schema; 如果某个节点(因为超时等原因)加载失败, 该节点的 TiDB Server 会主动挂掉
- start job: TiDB SQL 层收到请求后, 为 job 分配 ID 并存入 KV 层; 之后等待 job 完成即返回
- worker: 每个节点起一个处理 job 的 goroutine, 它会定期检查是否有待处理的 job; 既会等待本节点的 start job 模块通知, 也会主动检查是否有待执行的 job
- owner: 可以认为是一个角色, 信息存储在 KV 层, 包括记录当前当选此角色的节点信息
- job queue: 是一个存放 job 的队列, 存储在 KV 层; 逻辑上整个集群系统只有一个
- job history queue: 是一个存放已经处理完成的 job 的队列, 存储在 KV 层; 逻辑上整个系统只有一个

#### 基本流程

![TiDB Server 1 流程图](2.jpg)

![TiDB Server 2 流程图](3.jpg)

1. MySQL Client 发送给 TiDB Server 一个 DDL 请求
2. 某个 TiDB Server 收到请求, SQL 层执行(MySQL Protocol 收到请求/解析/优化); 这时 Server 生成一个 **Start Job** 并根据请求将请求内容封装成特定的 **DDL Job** , 并将 job 存储到 KV 层, 并通知 worker 有 job 可执行
3. 收到请求的 TiDB Server 的 worker 接收到处理 job 的通知后, 先判断自身是否是 owner(只有 owner 可以处理 DDL) , 如果是则直接处理该 job , 否则直接退出; 而没有收到请求的 worker 通过定时轮循检测未执行的 job , 发现该 DDL Job , 且该 worker 为 owner , 则获取该 job 权限, 且开始处理该 job
4. 当 worker 处理完 job 后, 它会将此 job 从 KV 层的 job queue 中移除, 并放入 job history queue
5. 之前封装 job 的 Start Job 模块会定期去 job history queue 查看是否有之前放进去的 job 对应 ID 的 job, 如果有则整个 DDL 操作结束
6. TiDB Server 将 response 返回 MySQL Client

#### 详细流程

下面是处理 **在 Table 中添加 columns** 的 worker 处理 job 的整个流程:

![add columns 具体流程](4.jpg)

```golang
type Job struct {
	ID       int64      `json:"id"`
	Type     ActionType `json:"type"`
	SchemaID int64      `json:"schema_id"`
	TableID  int64      `json:"table_id"`
	State    JobState   `json:"state"`
	Error    string     `json:"err"`
	// every time we meet an error when running job, we will increase it
	ErrorCount int64         `json:"err_count"`
	Args       []interface{} `json:"-"`
	// we must use json raw message for delay parsing special args.
	RawArgs     json.RawMessage `json:"raw_args"`
	SchemaState SchemaState     `json:"schema_state"`
	// snapshot version for this job.
	SnapshotVer uint64 `json:"snapshot_ver"`
	// unix nano seconds
	// TODO: use timestamp allocated by TSO
	LastUpdateTS int64 `json:"last_update_ts"`
}
```

TiDB Server 1 流程:
1. start job 给 start worker(??) 传递了 job 已经准备完成的信号
2. worker 开启一个事务, 检查自己是否是 owner 角色, 发现不是 owner 角色, 则提交事务退出处理 job 循环, 回到 start worker 等待信号的循环

TiDB Server 2 流程:
1. start worker 中定时器到达时间
2. 开启一个事务, 检查发现本节点为 worker 角色
3. 从 KV 层获取队列中第一个 job , 判断该 job 类型能否处理
4. 此时 job 类型为 add columns , 于是执行流程到达图中 get column information
&emsp;&emsp;a. 取对应 table info (通过 job 中的 schemaID 和 tableID), 然后确定添加的 columns 在原先表中是否不存在或者不可见
&emsp;&emsp;b. 如果新添加的 columns 在原先表中不存在, 那么将新的 columns 信息关联到 table info
&emsp;&emsp;c. 如果在未完成以上步骤, 且发生 job 被标记为 cancel 状态, 并返回 error , 之后到达图中 return error 流程(如: 对应的数据库、数据表的状态为不存在或者不可见(状态不为 public), columns 已存在并为可见状态)
5. (异步 schema update 开始) schema_version += 1
6. 将 job 状态的 schema 状态 和 table 中 columns 状态标记为 delete only , 更新 table info 到 KV 层
7. 由于 job 状态没有 finish (done 或 cancel) , 直接将 job 在上一步更新的信息写入到 KV 层
8. 在执行前面的操作时消耗了一定的时间, 所以这里将更新 owner 的 last update timestamp 为当前时间(防止经常将 owner 角色在不同服务器中切换), 并提交事务
9. 循环执行步骤 2、 3、 4.a、5、 6、 7 、8, 6 中的状态为 write only
10. 循环执行步骤 2、 3、 4.a、5、 6、 7 、8, 6 中的状态为 write reorganization
11. 循环执行步骤 2、 3、 4.a、5，获取当前事务的快照版本, 开始给新添加的列填写数据; 得到对应版本的表的所有 handle (rowID) , 批处理 handle , 然后针对每行新添加的列做数据填充(以下操作都在一个事务中完成):
&emsp;&emsp;a. 用先前取到的 handle 确定对应行存在, 如果不存在则不对此行做任何操作
&emsp;&emsp;b. 如果存在, 通过 handle 和 新添加的 columnID 拼成的 key 获取对应列; 获取的值不为空, 则不对此行做任何操作
&emsp;&emsp;c. 如果值为空, 则通过对应的新添加行的信息获取默认值, 并存储到 KV 层
&emsp;&emsp;d. 将当前的 handle 信息存储到当前 job reorganization handle 字段, 并存储到 KV 层; 这样, 假如步骤 12 执行到一半, 由于某些原因要重新执行 write reorganization 状态的操作, 那么可以直接从这个 handle 开始操作(, 那么之前的 handle 不再处理了 ??)
12. 将调整 table info 中 column 和 index column 中的位置, 将 job 的 schema 和 table info 中新添加的 column 的状态设置为 public, 更新 table info 到 KV 层; 最后将 job 的状态改为 done
13. 因为 job 状态已经 finish, 将此 job 从 job queue 中移除并放入 job history queue 中
14. 执行步骤 8, 此时 12, 13, 14 和 15(??) 在一个事务中

TiDB Server 1 流程:
1. start job 的定时检查触发后, 会检查 job history queue 是否有之前自己放入 job queue 中的 job(通过 job.ID); 如果有则此 DDL 操作在 TiDB SQL 完成, 上抛到 MySQL Protocol 层, 最后返回给 Client, 结束这个操作

#### 优化

原本对于删除操作的状态变化是
 public -> write only -> delete only -> delete reorganization -> none,
 优化的处理是去掉 delete reorganization 状态, 并把此状态需要处理的元数据的操作放到 delete only 状态时,
 把具体删除数据的操作放到后台处理, 然后直接把状态标为 none

那么更改后对数据完整性和一致性的判断:
- 将删除具体数据这个操作异步处理: 在数据放到后台处理前, 此数据表的元数据已经被删除并事先放置到 background job 中; 保证元数据对以后的 job 不可见, 且完整暂存在 background job 中
- 客户端访问表处于 none 状态的 TiDB Server: 这个其实更没有做优化前是一致的, 即访问不到此表
- 客户端访问表处于 delete only 状态的 TiDB Server: 此时客户端对此表做读写操作会失败, 因为 delete only 状态对它们都不可见

##### 实现

此优化对于原先的代码逻辑基本没有变化, 除了对于删除操作(目前还只是删除数据库和表操作)在其处于 delete only 状态时, 就会把元数据删除, 而对起表中具体数据的删除则推迟到后台运行, 然后结束 DDL job

放到后台运行的任务的流程跟之前处理任务的流程类似, 详细过程如下:

1. 在 [add columns 具体流程](4.jpg) 中判定 finish 操作为 true 后, 判断如果是可以放在后台运行, 那么将其封装成 background job 放入 background job queue, 并通知本机后台的 worker 将其处理
2. 后台 job 也有对应的 owner, 假设本机的 backgroundworker 就是 background owner 角色, 那么他将从 background job queue 中取出第一个 background job, 然后执行对应类型的操作(删除表中具体的数据)
3. 执行完成, 那么从 background job queue 中将此 job 删除, 并放入 background job history queue 中(步骤 2 和步骤 3 需要在一个事务中执行)

## 附加

### 概念

- OLTP(联机事务处理/On-Line Transaction Processing):
  - 强调数据库内存效率, 强调内存各种指标的命令率, 强调绑定变量, 强调并发操作
- OLAP(联机分析处理/On-Line Analytical Processing):
  - 强调数据分析, 强调 SQL 执行市场, 强调磁盘 I/O, 强调分区
- DW(数据仓库/Data-Warehouse):
  - 存储从 DB 截取出的视图
- ETL(数据清洗/Extraction-Transformation-Loading):
  - 用于完成 DB 到 DW 的数据转存
  - 一般的 DB 都是 ER 模型, 遵从范式化设计原则; DW 则是 面向主题/面向问题 的, 一般是 星型/雪花型, 两者模型结构不同
- DM(数据挖掘/Data Mining):
  - 根据统计学理论, 将 DW 中的数据进行分析, 找出不能直观发现的规律
- BI(商业智能):
  - 获取了 OLAP 的统计信息, 和 DM 得到的科学规律之后, 对生产进行适当的调整
