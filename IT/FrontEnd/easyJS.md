# JS 拾忆

## 事件阻止

- event.stopPropagation -- **阻止**事件冒泡(和捕捉?), 但**不会阻止**默认行为(链接跳转等)
- return false; -- **阻止**事件冒泡(和捕捉?), 也**阻止**了默认行为
- event.preventDefault -- **不阻止**事件冒泡, 但**阻止**默认行为
- event.stopImmediatePropagation -- 等价于 stopPropagation + return false ?
