# 逻辑时钟

## Lamport timestamps

![Lamport timestamps](116770-20160501174922566-1686627384.png)

缺点:
- 没有判断同时发生的事务

## Vector clock

![Vector clock](116770-20160502134654404-1109556515.png)

缺点:
- 无法实时解决冲突

## Version vector

![Version vector](116770-20160502183034013-800335383.png)

缺点:
- 如果使用 client 作为 vector 源, 可能面临 vector 过多的情况(这种情况 Vector clock 也哟)
