# React & Redux

## 1.Redux 流程图

![enter image description here](http://www.ruanyifeng.com/blogimg/asset/2016/bg2016091802.jpg)

- Store obj -- [数据中心]
	- createStore(Reducer, applyMiddleware(middlewares)) -- 生成[Data Center]
- State - obj -- [**当前**数据中心的快照] [通过 Store.getState 获取]
- Action - obj -- [与 View 相对应的 State 的当前变化]
	- type -- [Action 的行为/目的] [string常量 or Symbol] [必须]
	- error
	- payload -- [Action 的信息载体]
	- meta -- [额外的任何信息]
	- Action Creator -- 生成不同 Action 的函数
- Store.dispatch - func -- [View 派发 Aciton 的唯一方法]
- Reducer - func -- [oldState->newState 的计算过程] [纯函数]
- store.subscribe - func -- [设置监听函数 实现自动渲染]

## 2.Redux 中间件 --- Redux.applyMiddleware

#### 源码

```js
export default function applyMiddleware(...middlewares) {
  return (createStore) => (reducer, preloadedState, enhancer) => {
    var store = createStore(reducer, preloadedState, enhancer);
    var dispatch = store.dispatch;
    var chain = [];

    var middlewareAPI = {
      getState: store.getState,
      dispatch: (action) => dispatch(action)
    };
    chain = middlewares.map(middleware => middleware(middlewareAPI));
    dispatch = compose(...chain)(store.dispatch);

    return {...store, dispatch}
  }
}
```

### 1.redux-thunk

### 2.redux-promise

## 3.React-Redux

### 1.connect
	假定 
	* mapStateToProps --- 定义UI的输入逻辑 -- store.getState() 映射到 UI 组件
	* mapDispatchToProps --- 定义UI的输出逻辑
    
## 4.增强器 

增强器接口:

```js
const doNothingEnhancer = (createStore) => (ruducer, preloadedState, enhancer) => {
  const store = createStore(reducer, preloaedState, enhancer)
  return store
}
```
    
## 一点小疑惑

[x]1. 作为以大型系统为目标的 redux , 怎么应付大型系统中的数据解耦, 还是说与其单纯的数据解耦, 隔离应用才是正确的玩法?

答: 解耦在 reducer , 或者说通过 combineReducers 实行解耦
