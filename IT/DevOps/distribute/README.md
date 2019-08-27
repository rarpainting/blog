# 分布式系统的运维

![运维的发展](9e95e41a2fa724dc66b2f49a967845c6.png)

- 集中式日志系统(Logging): 用于记录离散的事件(应用程序的调试信息或错误信息), 它是我们诊断问题的依据
- 集中式度量系统(Metrics): 用于记录可聚合的数据(队列的当前深度可被定义为一个度量值, 在元素入队或出队时被更新; HTTP 请求个数可被定义为一个计数器, 新请求到来时进行累加)
- 分布式追踪系统(Tracing): 用于记录请求范围内的信息(一次远程方法调用的执行过程和耗时), 它是我们排查系统性能问题的利器

![Logging/Metrics/Tracing](92806aa2426813a4f47e6ba9b01f76f7.png)

[Metrics, tracing, and logging](http://peter.bourgon.org/blog/2017/02/21/metrics-tracing-and-logging.html)
