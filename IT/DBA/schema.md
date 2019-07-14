# Schema

## 数据库实现

- MySQL: 创建 Schema 和 Database 是等价的
- SQL Server 2000: user 和 schema 总有一层隐含的关系, 例如: 如果在数据库中创建了 user Bos , 那么后台也会默认创建一个 schema Bos, 且 user Bos 获得 schema Bos 的全部权限
- SQL Server 2005:
  - 向后兼容, 通过 **sp_adduser** 存储过程创建一个用户的时候, sqlserver 2005 同时也创建了一个和用户名相同的 schema
  - 通过 `create user` 创建数据库用户时, 可以指定一个已经存在的 schema 作为默认的 schema
  - 如果创建时不指定, 则新用户以 **dbo schema** 为默认 schema
- Oracle: **不能新建一个 schema, 要想创建一个 schema, 只能通过创建一个用户的方法解决** ; 在创建一个用户的同时为这个用户创建一个与用户名同名的 schema 并作为该用户的缺省 schema
  - 即 schema 的个数同 user 的个数相同, 而且 schema 名字同 user 名字一一对应并且相同
