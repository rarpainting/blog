# DVA 笔记

从 ant-design-pro 分析 dva 框架的点滴

## 源码解析

### common

#### menu.js

```js
// 补全路由的信息 完整信息为 [{ name, path, authority, (icon, )children }]
function formatter(data, parentPath="/", parentAuthority) {
  //...
}
```

#### router.js

```js
// 用于动态加载组件
function dynamicWrapper = (app, models, component) => {
  //...
}
```

