# JS 拾忆

## 事件阻止

- event.stopPropagation -- **阻止**事件冒泡(和捕捉?), 但**不会阻止**默认行为(链接跳转等)
- return false; -- **阻止**事件冒泡(和捕捉?), 也**阻止**了默认行为
- event.preventDefault -- **不阻止**事件冒泡, 但**阻止**默认行为
- event.stopImmediatePropagation -- 等价于 stopPropagation + return false ?

## 内存泄漏

- 全局变量
- console 输出
- dom 间接引用
- 闭包作用域
- jQuery/组件销毁时 事件没有解除绑定

## link 优化页面

- dns-prefetch -- 预解析域名
- subresource -- 将用到的关键性资源
- preload -- 之后 **必定** 需要的资源, 加载等级比 prefetch 高
- prefetch -- 之后 **可能** 需要的资源
- prerender -- 预渲染页面

*IE9 支持 "DNS-prefetch", 但标签是 "prefetch" ; 在 IE10+ 中, dns-prefetch 和 prefetch 等价 都能用于 DNS 预获取*

*preload 与 prefetch 混用, 会导致重复加载*

## npm

### Puppeteer

```shell
env PUPPETEER_SKIP_CHROMIUM_DOWNLOAD=1 npm install puppeteer
```
