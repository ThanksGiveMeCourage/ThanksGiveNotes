Week 1	
	Go Runtime 调度机制	
	GMP 模型、抢占调度、work stealing、runtime 源码（proc.go）	
	使用 trace/pprof 分析调度轨迹	
	输出：《Go 调度模型与工作窃取机制图解》

Week 2	
	Channel 与锁机制	
	channel 数据结构、select、sync.Mutex/RWMutex、atomic 包	
	实现简化版 channel，测试性能差异
	输出：《从源码看 Go Channel 的阻塞唤醒机制》

Week 3	
	内存管理与 GC	
	三色标记清除算法、对象逃逸分析、GOGC 调优
	模拟对象逃逸场景，分析 GC 日志	
	输出：《Go 内存分配模型与三色标记机制》

Week 4	
	网络 I/O 与 Netpoll	
	epoll/kqueue、goroutine-per-conn、net/http 底层
	实现高并发 echo server，对比 event loop 模式
	输出：《Go Netpoll 模型原理与高并发实测分析》

Week 5	
	RPC 框架与 gRPC 源码
	gRPC 架构、Protobuf 编解码、连接管理、超时熔断机制	
	实现 mini-RPC 框架，阅读 clientconn.go	
	输出：《自研 RPC 框架设计与 gRPC 对比分析》

Week 6	
	分布式系统基础	
	CAP/BASE 理论、一致性协议(Raft)、分布式锁、MQ
	使用 etcd/redis 实现分布式锁服务
	输出：《从 CAP 到 Raft：分布式系统的核心逻辑》

Week 7	
	游戏服务器架构与 Actor 模型
	Actor 模型思想、本地与远程调用、消息调度
	在 mini_server 中实现 Actor 调度与异步消息系统	
	输出：《mini_server 的 Actor 模型架构设计》

Week 8	
	系统设计与架构复盘
	高并发排行榜、登录、匹配系统设计
	绘制分布式架构图（网关+逻辑服+存储层）	
	输出：《Go 分布式游戏服务端架构复盘与设计要点》

Week 9	
	MySQL 深度原理与优化
	InnoDB 结构、事务隔离、MVCC、索引优化、连接池
	EXPLAIN 分析慢查询、模拟索引失效	
	输出：《MySQL InnoDB 存储与事务机制全解析》

Week 10	
	Redis 原理与工程实践
	SDS、跳表、RDB/AOF、过期淘汰、缓存一致性
	实现延迟任务队列、缓存一致性层、pipeline 测试	
	输出：《Redis 内部机制与缓存一致性设计》
