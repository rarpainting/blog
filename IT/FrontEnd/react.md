# React

## 数据请求应该放在 constructor 而不是 componentWillMount

## 在 static getDerivedStateFromProps 中根据 props 更新 state, 在 componentDidUpdate 中做事件触发等副作用

## 渲染顺序 componentWillUpdate -> render -> getSnapshotBeforeUpdate(?) -> commit -> componentDidUpdate

