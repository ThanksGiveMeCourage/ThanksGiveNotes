# Go sync.Map 源码解析

在前篇 [Go map 源码解析](map.md) 中，我们详细讨论了 Go map 的底层源码实现，也因此学习到了许多优秀的设计 和 实现原理。但纵使是优秀如 Go map，却也有着难以掩饰的致命缺点：***Go map 是线程不安全的***

而要说明什么是线程不安全，那么就必须先说 ***线程安全***。

## 1、什么是线程安全？
 线程安全(Thread Safety) 是指在多线程环境中，当多个线程同时访问 和 操作 共享资源时，程序能够正确地执行而不会出现数据不一致或程序崩溃的情况。线程安全意味着无论线程如何交替执行，程序总能产生预期的结果。

 而在实现 线程安全 通常需要对共享资源进行适当的同步，以确保只有一个线程在任意给定时间内访问或修改这部分资源。其常用方法有：
 * 1、锁(Locks): 通过使用 互斥锁(Mutex)、读写锁(Read-Write Lock) 等机制来保护共享资源，确保同一时间只有一个线程可以访问这些资源。
 * 2、原子操作(Atomic Operations): 使用硬件 或 软件提供的原子操作来保证对共享变量的修改是不可分割的，从而避免竞争条件。
 * 3、线程本地存储(Thread Local Storage，TLS): 为每个线程提供独立的存储空间，使得每个线程都有自己的一份数据副本，从而避免共享资源的竞争。
 * 4、不可变对象(Immutable Objects): 使用不可变对象，确保对象一旦创建后其状态就不能再改变，这样多个线程可以安全地共享这些对象而无需同步。
 * 5、并发集合(Concurrent Collections): 使用语言 或 库提供的线程安全集合：
 * * 5.1、Java 中的 ConcurrentHashMap
 * * 5.2、C++ 中的 concurrent_unordered_map
 * * 5.3、***Go 中的 sync.Map***

 ## 2、what is sync.Map

 sync.Map 的源码文件：/src/sync/map.go

 让我们保持一个良好的源码阅读习惯，先看看大佬们的源码注释，我认为在面对一份陌生的源码时，先看看注释是一个很好的切入点。因为任何时候，注释都能给我们提供最为直观的解读信息。

 以下为 基于源码的 第 11 行 --- 第 34 行的源码注释
 * 1、sync.Map 的基本特性
 * * 1.1、sync.Map 类型类似于 Go 语言中的 map[interface]interface{}，但其具备线程安全。
 * * 1.2、sync.Map 类型是专门化的，读、写 和 删除 在平均情况下都是常数时间复杂度。
 * 2、使用建议
 * * 2.1、对于大多数代码来说，使用普通的 Go map，并配合单独的锁（如 Mutex 和 RWMutex）来实现线程安全更好。因为这样可以获得更好的类型安全，而且更易于维护与 map 内容相关的其他不变性条件。
 * 3、sync.Map 的优化场景，在下述的两种情况下，相比于搭配单独的 Mutex 或 RWMutex 的 Go map，使用 sync.Map 可能会显著减少锁竞争：
 * * 3.1、某个键的条目只写入一次，但被读取多次，例如仅增长的缓存；
 * * 3.2、多个 goroutine 对不同键集的条目进行读、写 和 覆盖；
 * 4、初始化：
 * * 4.1、零值的 sync.Map 是空的 且 可以直接使用
 * * 4.2、sync.Map 一旦使用后就不能复制。
 * 5、内存模型：
 * * 5.1、根据 Go 内存模型的术语，sync.Map 确保 写操作 在 “先行发生” 于任何观察到该写操作的读操作之前；
 * * 5.2、基于内存模型下的操作分类：
 * * * 5.2.1、读操作：Load、LoadAndDelete、LoadOrStore、Swap、CompareAndSwap 和 CompareAndDelete
 * * * 5.2.2、写操作：Delete、LoadAndDelete、Store、Swap
 * * * 5.2.3、当 LoadOrStore 返回 loaded 为 false 时，它是写操作
 * * * 5.2.4、当 CompareAndSwap 返回 swapped 为 true 时，它是写操作
 * * * 5.2.5、当 CompareAndDelete 返回 deleted 为 true 时，它是写操作

 从源码注释中，我们得知了，在常规情况下，使用普通的 Go map，并配合单独的锁（如 Mutex 和 RWMutex）来实现线程安全更好。因为这样可以获得更好的类型安全，而且更易于维护与 map 内容相关的其他不变性条件。

 这里，需要注意 ***类型安全*** 这个点

 ***类型安全[Type Safety] : 是编程语言中的一个重要概念，它指的是在程序执行过程中，保证各种值始终具有与其声明类型相符的行为。类型安全有助于防止一类常见的 编译错误，从而提高代码的可靠性 和 可维护性***
 * 1、类型一致性：程序中的每个变量和表达式都有一个明确的类型，在类型安全的环境中，赋值、函数调用等操作都必须满足预期的类型。例如：一个整数不能被赋值为 字符串类型的值
 * 2、编译时检查：类型安全的语言通常在编译时进行类型检查，确保所有操作符合类型要求，这可以提前捕获许多潜在错误。例如：基本类型 和 普通的 Go map 在编译器都是检查类型安全性
 * 3、运行时保护：即使在一些动态类型语言中，也会在运行时进行类型检查，以确保类型一致性。这虽然可能会影响性能，但可以提供额外的安全保障。

 而 sync.Map 却无法保证类型安全，这是因为 sync.Map 中的 key/value 的类型都是 interface{}，这意味着它们可以接受任何类型，如此将大大降低 类型安全性
 ```go
 func main() {
	var a sync.Map
	a.Store("key", 111)
	a.Store("key", "111")
	value, ok := a.Load("key")
	if ok {
		strV := value.(int)
		fmt.Println(strV)
	}
}
// panic: interface conversion: interface {} is string, not int
 ```

 这是否说明，sync.Map 的存在十分鸡肋呢？

 这也不然，sync.Map 需要适配特定的场景，这样才能发挥它的优势：
 * 1、某个键的条目只写入一次，但被读取多次，例如仅增长的缓存；
 * 2、多个 goroutine 对不同键集的条目进行读、写 和 覆盖；

 我们接下来，将基于源码逐步进行解析

 ## 3、sync.Map 的 数据结构
 ```go
 type Map struct {
	mu Mutex

    /* 
    译文：
        1、read 字段所包含的 map 中的这部分内容 在 并发访问时是 线程安全的(无论是否持有 mu)
        2、read 字段 本身在加载时总是安全的，但必须持有 mu
        3、存储在 read 中的条目可以在没有 mu 的情况下进行并发更新，但如果要更新一个之前已经删除的条目，则需要在持有 mu 的情况下，将该条目复制到 dirty 中，并取消其删除状态。
    */
	read atomic.Pointer[readOnly]

    /*
    译文：
        1、dirty map 包含那些需要持有 mu(互斥锁) 才能访问的部分内容。为了确保可以快速将 dirty map 提升为 read map，dirty map 中还包含所有 read map 中 “non-expunged” 的条目
        2、被 “expunged”(可能指的是标记为 无效 或 删除)的条目不会存储在 dirty map 中。如果一个条目在 clean map 中被标记为 “expunged”，在存储新值之前，必须先将其从 “expunged” 状态恢复过来，并添加到 dirty map 中。
        3、如果 dirty map 是空的(nil)，则下一次写操作会通过 浅拷贝 clean map 来初始化 dirty map，省略陈旧的条目。（浅拷贝意味着只复制指向原始数据的引用，而不是创建数据的完全副本。这是一种高效的复制方式，因为它避免了重复数据）
    */
	dirty map[any]*entry

	/* 
    译文：
        1、misses 记录自上次更新 read map 以来，需要通过锁定 mu 来确定键是否存在的加载操作次数。（简单来说，当从 read map 读取失败-即未命中目标键，需要访问 dirty map 时，会增加 misses 计数。）
        2、一旦发生足够多的未命中，使得这些未命中的代价等同于复制 dirty map 的成本，系统就会将 dirty map 提升为 read map（在未修改状态下）。下一次对该映射的存储操作将创建一个新的 dirty 副本。
    */
	misses int
}

// readOnly 结构体是不可变的，并且它被原子性存储在 Map.read 字段中
type readOnly struct {
	m       map[any]*entry // 是一个不可变的映射，包含当前 read map 的所有条目
	amended bool // 如果 amended 为 true，则表示 dirty map 包含了一些 m 没有的键。
}  

// An entry is a slot in the map corresponding to a particular key.
type entry struct {
	/* 
    译文：
        1、p 指向存储在此条目中的 interface{}值
        2、如果 p == nil，表示该条目已经被删除，并且 m.dirty 可能为 nil 或 m.dirty[key] 是当前条目 e
        3、如果 p == expunged（expunged 是某个特殊标记值），表示条目已被删除，并且 m.dirty 不为空 且 当前条目从 m.dirty 消失。
        4、否则，该条目是有效的，并记录在 m.read[key] 中，如果 m.dirty[key] 不为空，也会记录在 m.dirty[key] 中。
        5、可以通过将 p 原子性地替换为 nil 来删除一个条目：当 m.dirty 下次创建时，它会原子性地用 expunged 替换为 nil，并使 m.dirty[key] 保持未设置状态。
        6、条目的关联值可以通过原子替换进行更新，前替是 p != expunged。如果 p == expunged，那么只能先设置 m.dirty[key] = e 后才能更新条目的关联值，以便使用 dirty map 查找该条目。
    */
	p atomic.Pointer[any]
}
 ```
 在我们解析了 sync.Map 的数据结构后，我们可以一窥其设计上的一些特点：
 * 1、为什么说 sync.Map 在设计上更适合于 多读少写 的场景的？原因：其数据结构在设计上采取了一种近似 “读写分离” 的数据结构
 * * 1.1、读优化：read 字段 本身基于 atomic.Pointer 的原子性维护，在 并发读取上 无需锁的加持而自由使用。只有在 写操作 的时候才需要 锁加持；
 * * 1.2、增量更新：dirty 字段，则是强制性的 锁加持 字段，只有在 写操作 或 读read失败后 才会涉及到 mu 和 dirty，从而减少锁竞争；
 * * 1.3、渐进式升级：再基于 misses 字段的 计数器逻辑(对 read 访问 key 失败而 不得不去锁定 dirty 进行访问的次数进行叠加)，在数值达到阈值后，将 dirty 进阶为 read。

 ## 4、sync.Map 的数据读取
在 

```go
// Load returns the value stored in the map for a key, or nil if no
// value is present.
// The ok result indicates whether value was found in the map.
func (m *Map) Load(key any) (value any, ok bool) {
    // 优先检测 只读映射 read，并尝试获取 key 的值
	read := m.loadReadOnly()
	e, ok := read.m[key]
    // read 查询失败 且 amended 为 true(表示 dirty map 中包含了一些 read 中没有的 key)
	if !ok && read.amended {
		m.mu.Lock()
		/* 
        译文：
            避免报告的是虚假的未命中，如果在等待 m.mu 锁的过程中，m.dirth 被提升为了 read map。(如果后续对相同键的再次加载也未命中，那就不得不为了这个键去检测 dirty map)

            上述译文，提出了一种可能性，那就是 当线程被阻塞在获取锁的过程中，dirty map  可能已经被提升为 read map 了。所以这里需要在已锁定 map 的情况下，再次检测 read。
        */
		read = m.loadReadOnly()
		e, ok = read.m[key]
        // 在锁定 map 后的第二次检测 read 也失败了，那么就不得不去检测 dirty 了
		if !ok && read.amended {
			e, ok = m.dirty[key]
			/* 
            译文：
                无论该条目是否存在，都会记录一次未命中。这意味着在 diry map 被提升为 read map 之前，该 key 都会走 慢路径(需要通过 dirty)
            */
			m.missLocked()
		}
		m.mu.Unlock()
	}
	if !ok {
		return nil, false
	}
    // 从条目中加载实际存储的值
	return e.load()
}
```

## 5、sync.Map 的 数据插入
```go
// Store sets the value for a key.
func (m *Map) Store(key, value any) {
	_, _ = m.Swap(key, value)
}

// Swap swaps the value for a key and returns the previous value if any.
// The loaded result reports whether the key was present.
func (m *Map) Swap(key, value any) (previous any, loaded bool) {
    // 优先检测 只读映射 read，并尝试获取 key 的值
	read := m.loadReadOnly()
    // 如果 read 检测到 key 已存在，则尝试进行替换
	if e, ok := read.m[key]; ok {
        // 尝试替换旧值，如果成功，则返回旧值(底层执行 CAS，CompareAndSwap)
		if v, ok := e.trySwap(&value); ok {
			if v == nil {
				return nil, false
			}
			return *v, true
		}
	}

	m.mu.Lock()
	read = m.loadReadOnly()
    // 加锁后，二次检测 read。这是一个优化策略(因为加锁阶段，dirty 可能升阶为 read)
	if e, ok := read.m[key]; ok {
        // unexpungeLocked 方法确保该条目没有被标记为清除。这意味着如果条目之前被标记为已清除，现在它将被恢复为未清除状态。(底层使用 CAS)
		if e.unexpungeLocked() {
			/* 
            译文：
                这个条目之前已经被标记为删除，这暗示着当前存在一个非空的 dirty map，而且这个条目不在 dirty map 中
            */
            // 由于该条目已被从 dirty 中删除，所有直接添加
			m.dirty[key] = e
		}
        // 执行交换操作
		if v := e.swapLocked(&value); v != nil {
			loaded = true
			previous = *v
		}
	} else if e, ok := m.dirty[key]; ok { // 检查 key 是否在 dirty map 中
		if v := e.swapLocked(&value); v != nil {
			loaded = true
			previous = *v
		}
	} else {
        /* 
            当前 key 即不在 read 也不在 dirty 中
            如果 amended 为 false，则表示 read 和 dirty 数据是一致的。
        */
		if !read.amended {
			/* 
            译文：
                我们正在向 dirty map 中添加第一个新 key
                确保它已被分配 且 标记只读映射为 不完整状态
            */
			m.dirtyLocked() // dirtyLocked 将 read 中的数据 与 dirty 进行同步
			m.read.Store(&readOnly{m: read.m, amended: true}) // 标记 read 中的 amended 为 true
		}
        // dirty 录入新数据
		m.dirty[key] = newEntry(value)
	}
	m.mu.Unlock()
	return previous, loaded
}
```

## 6、sync.Map 的 数据删除
```go
// Delete deletes the value for a key.
func (m *Map) Delete(key any) {
	m.LoadAndDelete(key)
}

// LoadAndDelete deletes the value for a key, returning the previous value if any.
// The loaded result reports whether the key was present.
func (m *Map) LoadAndDelete(key any) (value any, loaded bool) {
    // 优先检测 只读映射 read，并尝试获取 key 的值
	read := m.loadReadOnly()
	e, ok := read.m[key]
    // read 查询失败 且 amended 为 true(表示 dirty map 中包含了一些 read 中没有的 key)
	if !ok && read.amended {
		m.mu.Lock()
		read = m.loadReadOnly()
		e, ok = read.m[key]
        // 在锁定 map 后的第二次检测 read 也失败了，那么就不得不去检测 dirty 了
		if !ok && read.amended {
			e, ok = m.dirty[key]
			delete(m.dirty, key) // 删除 dirty map 中的 key
			// Regardless of whether the entry was present, record a miss: this key
			// will take the slow path until the dirty map is promoted to the read
			// map.
            // 无论 key 是否存在，都记录一次 未命中。在 dirty 提升为 read 前，当前 key 将始终走慢路径
			m.missLocked()
		}
		m.mu.Unlock()
	}
	if ok {
		return e.delete() // 删除条目并返回 旧值 和 true(表示命中)，底层也是基于 CAS
	}
	return nil, false
}
```
sync.Map 的实现。它的实现本质是基于 Go map + atomic.Pointer + Mutex 的二次封装。

***sync.Map = Go map + Mutex + atomic.Pointer + CAS***

