# React

## 数据请求应该放在 constructor 而不是 componentWillMount

## 在 static getDerivedStateFromProps 中根据 props 更新 state, 在 componentDidUpdate 中做事件触发等副作用

## 渲染顺序 componentWillUpdate -> render -> getSnapshotBeforeUpdate(?) -> commit -> componentDidUpdate

## React Suspense -- 用于数据请求

数据请求的步骤:
- 在 render 中, 写入一个异步请求, 请求数据
- react 从缓存中读取数据
- 有缓存: 正常 render
- 没有缓存, **抛出一个 Promise 的异常, 中断本次渲染(这时渲染的是什么?)**
- 当这个 Promise 完成后(请求数据完成), react 会继续回到原来的 render 中(重新执行一次 render), 运行 render

## React Hooks

```js
const { state0, setState0 } = useState(0)
```

用于把 state 的内容分割到各个实际的地方, 但是各个 state 的映射还是不那么自然

