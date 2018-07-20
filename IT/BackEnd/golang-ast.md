# golang ast 分析

## 关键结构体

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
