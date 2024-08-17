# Go 的 内存模型

一切以官方文档为主：[Go dev: memory](https://go.dev/ref/mem)

Go 的内存模型描述了在多线程程序中，Go 是如何 ***对共享变量的访问进行排序*** 和 ***保证一致性***。

我们这里对文档内容进行一下总结，主要可分为三个基本概念：
* 1、顺序一致性；
* 2、同步事件；
* 3、Happens-Before 关系；

## 1、顺序一致性
* 针对单个Goroutine而言，所有的操作是按程序中指定的顺序执行的，即代码中的顺序就是实际执行顺序
* 针对多个Goroutine而言，其Goroutine之间的操作顺序是无法保证顺序的，只能通过某种形式的同步(例如：channel、Mutex、atomic 等等)

## 2、同步事件

而为了保证多个Goroutine之间的内存操作顺序，可基于下述同步原语来实现：

* 1、sync.Mutex：互斥锁，悲观锁；
* 2、sync.RwMutex：读写锁，它允许多个读操作并发进行，但写操作是互斥的；
* 3、sync.WaitGroup：用于等待一组 Goroutine 的完成，Done 操作发生在随后的 Wait 操作之前。
* 4、sync.Once：只执行一次；
* 5、sync.Cond：条件变量，用于复杂的同步场景
* 6、sync/atomic：Go 提供了原子操作函数，如 atomic.AddInt32、atomic.LoadInt32 等，这些操作对于并发访问是安全的，并且可以用来构建更复杂的同步机制。
* 7、channel：：通过通道的发送和接收操作是同步事件，发送操作保证在接收操作完成之前发生。

## 3、Happens-Before 关系
* 1、如果在 Goroutine A 中的一个操作发生在 Goroutine B 中的另一个操作之前，且这种关系是通过某种同步手段（如上面提到的通道、锁等）确定的，那么我们说 A 发生在 B 之前（A "happens-before" B）。
* 2、Happens-before 关系确保在不同 Goroutine 中的操作是有序的。