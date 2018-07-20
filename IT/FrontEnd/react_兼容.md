# 通用方案
	https://github.com/xcatliu/react-ie8
# 阿里巴巴方案
	http://www.aliued.com/?p=3240

## 其中
### 一般疑问

#### 一.es3ify解决es3环境兼容
		'es3ify' => 'es3ify-webpack-plugin'
#### 二.webpack2.1 使用了 Object.defineProperty
		需要把 webpack 退版本？还是在 webpack 后续转换？
#### 三.Object.defineProperty
		不了解 webpack 目前对 {export from} 是否依旧采用 Object.defineProperty

### 目前的疑问

#### 一. export ... from ... 测试
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

### 完整的 webpack 文件
### 各库的版本
