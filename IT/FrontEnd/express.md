# ExpressJS
## 路由
### 路由方法
### 路由路径

*通过 Express Route Tester 做路径检测*

- 字符串
- 字符串模式
- 正则

### 路由句柄
	通过next()在多个回调中传递请求
### 响应方法 res 对象

	.download() -- 提示(?)下载文件
	.end()
	.json()
	.jsonp() -- 发送支持JSONP的JSON格式的响应
	.redirect() -- 【重点】重定向请求
	.render() -- 渲染[, 终结请求(?)]
	.send() -- 发送响应[, 终结请求(?)]
	.sendFile()
	.sendStatus()

### app.route()

	用于路由路径的链式路由句柄

```javascript
app.route('/book')
.get(function(){
	res.send('Get a random book')
}).post(function(){
	res.send('Add a book')
}).put(function(){
	res.send('Update the book')
})
```

### app.Router

	定义子路由路径
## 中间件
### 中间件的功能

- 执行代码
- 修改请求和响应对象(重点)
- 终结请求-响应循环
- 调用堆栈的下一个中间件

*如果当前中间件没有**[终结请求-响应循环]**阶段, 则必须调用next(), 否则该请求被**挂起***

### 应用级中间件

**var app = express();**
	app.use()
	app.METHOD()
*next('route') //忽略当前路由的中间件栈, 跳到下一路由*

### 路由级中间件

**var router = express.Router()**

### 错误处理中间件

*定义错误处理中间件时必须使用 4 个参数(err, req, res, next)*
**next(err) 跳过*[错误处理]*以外的中间件**

### 内置中间件

**express.static(root, [options]) -- 用于托管静态资源**

| 属性				   | 描述				                                                              | 类型				  | 缺省				   |
| :------                | :-----:                                                                           | :-----:               | -----:                 |
| dotfiles               | 是否对外输出文件名以点(.)开头的文件; 可选值为 “allow”、“deny” 和 “ignore” | String                | “ignore”             |
| etag				   | 是否启用 etag 生成	                                                            | Boolean			   | true				   |
| extensions			 | 设置文件扩展名备份选项	                                                        | 	Array			 | []					 |
| index				  | 发送目录索引文件, 设置为 false 禁用目录索引                                       | Mixed                 | “index.html”	     |
| lastModified           | 设置 Last-Modified 头为文件在操作系统上的最后修改日期, 可能值为 true 或 false     | Boolean               | true                   |
| maxAge				 | 以毫秒或者其字符串格式设置 Cache-Control 头的 max-age 属性                        | Number                | 	0                  |
| redirect			   | 当路径为目录时, 重定向至 “/”                                                    | Boolean		       | true				   |
| setHeaders			 | 设置 HTTP 头以提供文件的函数                                                      | Function		      | 					   |

### 第三方中间件
