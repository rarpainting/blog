# React

## React 16

### Context API

TODO:

### Time Slicing -- 缓解 CPU 处理

- React 延长 render 的时间, 但是不阻塞当前线程, 避免了卡顿

### React Suspense -- 用于数据请求

数据请求的步骤:
- 在 render 中, 写入一个异步请求, 请求数据
- react 从缓存中读取数据
- 有缓存: 正常 render
- 没有缓存, **抛出一个 Promise 的异常, 中断本次渲染(这时渲染的是什么?)**
- 当这个 Promise 完成后(请求数据完成), react 会继续回到原来的 render 中(重新执行一次 render), 运行 render

### React Hooks



```js
const { state0, setState0 } = useState(0)
```

用于把 state 的内容分割到各个实际的地方, 但是各个 state 的映射还是不那么自然


## 注意事项

### 数据请求应该放在 constructor 而不是 componentWillMount

### 在 static getDerivedStateFromProps 中根据 props 更新 state, 在 componentDidUpdate 中做事件触发等副作用

### 渲染顺序 componentWillUpdate -> render -> getSnapshotBeforeUpdate(?) -> commit -> componentDidUpdate


## 向上兼容性
### 通用方案
	https://github.com/xcatliu/react-ie8
### 阿里巴巴方案
	http://www.aliued.com/?p=3240

#### 其中
##### 一般疑问

- es3ify 解决 es3 环境兼容
		'es3ify' => 'es3ify-webpack-plugin'
- webpack2.1 使用了 Object.defineProperty
		需要把 webpack 退版本？还是在 webpack 后续转换？
- Object.defineProperty
		不了解 webpack 目前对 {export from} 是否依旧采用 Object.defineProperty

##### 目前的疑问

##### 一. export ... from ... 测试
	使用 export ... from ... 编译结果是

```javascript
(function(module, __webpack_exports__, __webpack_require__) {
"use strict";
...

Object.defineProperty(exports, "__esModule", {
    value: true
});
exports.Mater = undefined;
```

不使用 export ... from ...;

```js
(function(module, exports, __webpack_require__) {
"use strict";
Object.defineProperty(exports, "__esModule", {
    value: true
});
exports.Mater = undefined;
```

使用 core-js/es6/object 添加方法

