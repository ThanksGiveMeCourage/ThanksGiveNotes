# Go Slice 源码解析

## Slice 与 Array
![](../../images/slice01.png) 

### Array(数组)
* 优势：
* * 长度固定，使得其在内存中是连续存储的，查询更为高效；
* * 存储上除了数据本身没有额外元信息数据，内存更加紧凑；
* 劣势：
* * 不可变长度，限制了其使用范围，在复杂的实际应用环境中无法动态加载调整大小；
* * 数组长度本身也是其类型的一部分，例如：[5]int 和 [10]int 是两种不同的类型，这在函数传参时需要特别注意；
* * 数组是值类型，赋值 和 函数传参都会复制整个数组数据，如果数据量过大，将造成大量不必要的内存消耗（注意：就算使用 数组指针进行函数传参，那本质上也是 值传递，相对于是 copy 一个 指针的副本进行传参，只不过是副本指针也是指向的同源底层数组，所以也能修改源数组的数据）；
```go
func fk2(a [1e7]int) {
	var m runtime.MemStats
	runtime.ReadMemStats(&m)
	fmt.Println("3 --- func Alloc: ", m.Alloc)
}
func main() {
	var m runtime.MemStats
	runtime.ReadMemStats(&m)
	fmt.Println("1 --- Alloc: ", m.Alloc)
	var a [1e7]int
	for i := 0; i < len(a); i++ {
		a[i] = i
	}
	runtime.ReadMemStats(&m)
	fmt.Println("2 --- Alloc: ", m.Alloc)
	fk2(a)
}
/* 
1 --- Alloc:  106664
2 --- Alloc:  80115736
3 --- func Alloc:  80117040
*/
```
### Slice(切片)
* 优势：
* * 动态大小，适配于更加广度的应用范围(容量大小的变化会基于扩容机制动态变更，可以一定程度上避免频繁的 底层数组的复制 与 重新分配，可以降低性能消耗)；
* * 函数传参时更为高效，因为它只涉及到表侧的 slice 结构体传递(64位系统下，默认 24 bytes)；
* * 共享底层数组，多个切片可以共享一个底层数组的不同部分，这使得在某些场景下(如从大数据集中提取子集)更加高效；
* 劣势：
* * 额外的元数据消耗，切片额外需要维护一个表侧的引用结构体，用于管理底层的数组数据；
* * 潜在的内存泄漏风险；
```go

var a []int

const needNum = 1

func fk13(b *[]int) {
	a = (*b)[:needNum]

	/* for i := 0; i < needNum; i++ {
		a = append(a, (*b)[i])
	} */
}

func sliceMemoryLeak() {
	var m runtime.MemStats

	runtime.ReadMemStats(&m)
	fmt.Printf("0 ------ allocated memory: %d bytes\n", m.Alloc)

	largeSlice := make([]int, 1e7)
	for i := range largeSlice {
		largeSlice[i] = i
	}

	runtime.ReadMemStats(&m)
	fmt.Printf("1 ------ allocated memory: %d bytes\n", m.Alloc)

	fk13(&largeSlice)

	largeSlice = nil
	runtime.GC()

	runtime.ReadMemStats(&m)
	fmt.Printf("2 ---[ after largeSlice = nil ]--- allocated memory: %d bytes\n", m.Alloc)

	fmt.Println("a ------> : ", a)

	a = nil
	runtime.GC()

	runtime.ReadMemStats(&m)
	fmt.Printf("3 ---[ after a = nil ]--- allocated memory: %d bytes\n", m.Alloc)
}
func main() {
	sliceMemoryLeak()
}
/* 
0 ------ allocated memory: 108312 bytes
1 ------ allocated memory: 80115560 bytes
2 ---[ after largeSlice = nil ]--- allocated memory: 80115560 bytes
a ------> :  [0]
3 ---[ after a = nil ]--- allocated memory: 112648 bytes
*/
```
上述代码实例模拟了 slice 的内存泄漏情况。当存在 全局变量时，这时对函数中调度的切片形成一个引用( a = (*b)[:needNum] ), 这样会造成一种很糟糕的情况( 那就是当 main 中的 原切片引用置为 nil 并 手动触发 GC，结果发现 内存依旧没有被释放 --- 如此便造成了内存泄漏了 )，而原因便是 存在某一变量仍引用着底层数组数据，哪怕仅仅只是只引用了一个元素，golang 的 GC 管理器也会认为此数据仍处于使用状态，如此就会造成，实际上已无用的内存数据，因存在不当的引用 而 无法得到释放。

而在函数中 想要完全隔绝与 源底层数组数据的关系，从而安全地获取数据的方案，比如：利用一个空容量的切片对象，然后使用 append，利用切片的扩容机制，底层数组在容量不足时会主动进行扩容（分配一个新容量的新数组，然后将数据复制到新数组中），如此便让两者的数据完全分离开了。
```go
// 替换上面 fk13 函数 并 再次运行查看结果
// a = (*b)[:needNum]
for i := 0; i < needNum; i++ {
	a = append(a, (*b)[i])
}
/* 
0 ------ allocated memory: 106456 bytes
1 ------ allocated memory: 80115560 bytes
2 ---[ after largeSlice = nil ]--- allocated memory: 112488 bytes
a ------> :  [0]
3 ---[ after a = nil ]--- allocated memory: 112560 bytes
*/
```

上面我们分析了 array 与 slice 的优劣，下面我们将基于源码来解析 slice 的实现原理，我们可以分为几个模块来进行分析
* 1、slice 的结构体
* 2、slice 的初始化
* 3、slice 的扩容机制
* 4、slice 的拷贝

## Slice 的 结构体

![](../../images/slice_struct01.png)

让我们先打开 slice 的源码 ：src/runtime/slice.go
```go
type slice struct {
	array unsafe.Pointer // 指向底层数组的指针
	len   int // 当前底层数组的长度
	cap   int // 当前底层数组的容量
}
/* 
注意：始终需要保证 len <= cap
*/
```
从源码中我们知晓了， 切片的本身是一个复合的结构体。通过一个指针引用底层数组，这样可以直接封装数组指针的功能，让开发者在使用体感上可以无缝贴合数组，再加上一个数组长度 和 一个数组容量，这两个字段可为切片的扩容机制服务，从而使切片得以升华为 动态扩容 的模式

在我们对 slice 的封装结构有了一个认识后，我们通过一个代码实例来进一步加深一下，同时也为后续的创建流程抛砖引玉一下
```go
/* 
	现在创建一个 len=5，cap=6 的切片。请试想如下问题：
		1、当前可访问的长度是 5 还是 6 ？
		2、当前底层数组的size是多少 bytes？（友情提示：int 在 64位系统中 8 bytes）

*/
func main() {
	s := make([]int, 5, 6)
	fmt.Println("s[4]: ", s[4]) // 可访问长度5, [0-4] ?
	fmt.Println("s[5]: ", s[5]) // 可访问长度6, [0-5] ?
	fmt.Println("s - len : ", len(s), " cap: ", cap(s))

	// 获取底层数组的指针
	sPtr := (*[6]int)(unsafe.Pointer(&s[0]))
	// 计算底层数组的大小
	sSize := unsafe.Sizeof(*sPtr)
	fmt.Println("sSize: ", sSize)
}
/* 
s[4]:  0
// s[5] -------> 越界了
s - len :  5  cap:  6
sSize:  48
*/
```
第一个问题，应该很好理解，长度为5，那么自然只能方位 s[0] - s[4]。
但这样就有个疑问了，我只初始化了 5 个长度的数据，为什么数组的大小是 48 = 6 * 8 呢？
这就是由 切片的 make 初始化创建过程决定了，让我们继续往下看。
## Slice 的初始化
![](../../images/slice02.png)
```go
/* 
et *_type : _type 的指针，用以表示底层数组元素的类型，_type 是所有类型最原始的元信息
len, cap int : len - 底层数组的长度     cap - 底层数组的容量
*/
func makeslice(et *_type, len, cap int) unsafe.Pointer {
	/* 
		math.MulUintptr : et.size 和 uintptr(cap) 的乘积 ---> 所想申请的最大内存容量
		et.size: 类型元素的字节大小
		uintptr(cap): 将容量转化为 uintptr 类型

		overflow: 乘积是否 大于了 uintptr的最大值(系统可分配的最大内存)
	*/
	mem, overflow := math.MulUintptr(et.size, uintptr(cap))
	// 判定 容量是否超载了 或 len 是否无效
	if overflow || mem > maxAlloc || len < 0 || len > cap {
		// NOTE: Produce a 'len out of range' error instead of a
		// 'cap out of range' error when someone does make([]T, bignumber).
		// 'cap out of range' is true too, but since the cap is only being
		// supplied implicitly, saying len is clearer.
		// See golang.org/issue/4085.
		// 容量申请内存是爆了，再checke 一下 len 申请的结果(保证: len <= cap)
		mem, overflow := math.MulUintptr(et.size, uintptr(len))
		if overflow || mem > maxAlloc || len < 0 {
			panicmakeslicelen()
		}
		panicmakeslicecap()
	}
	// cap 和 len 都是有效数值，使用 垃圾回收分配器 mallocgc 分配内存
	return mallocgc(mem, et, true)
}
```
上面从源码角度解析了 切片的 make 创建初始化逻辑。

我们由此也知晓了 [ s := make([]int, 5, 6) ] 的大小为什么是 48 = 6 * 8。因为内存分配时是根据容量的数值，预先就分配完成了的，只是还未使用而已。

而在 slice 的创建写法上的差异，而存在两种不同的概念：
* nil切片 ： 一个 [ 未初始化 ]的切片，其值为 nil，长度 和 容量 均为 0
* 空切片 ： 一个[ 已初始化 ]的切片，但这个切片不包含元素，其容量和长度也为0，但内存其实已经给其分配底层数组了，因为其指针已经被赋值了。
```go
func fkk() {
	// 定义一个 nil 切片
	var nilSlice []int
	// 定义一个 空 切片
	emptySlice := make([]int, 0)
	emptySliceLiteral := []int{}

	fmt.Printf("Nil Slice: %v, len: %d, cap: %d, address: %p\n", nilSlice, len(nilSlice), cap(nilSlice), nilSlice)
	fmt.Printf("Empty Slice(make): %v, len: %d, cap: %d, address: %p\n", emptySlice, len(emptySlice), cap(emptySlice), emptySlice)
	fmt.Printf("Empty Slice(Literal): %v, len: %d, cap: %d, address: %p\n", emptySliceLiteral, len(emptySliceLiteral), cap(emptySliceLiteral), emptySliceLiteral)
}
/* 
Nil Slice: [], len: 0, cap: 0, address: 0x0
Empty Slice(make): [], len: 0, cap: 0, address: 0x56c090
Empty Slice(Literal): [], len: 0, cap: 0, address: 0x56c090
*/
```
## Slice 的扩容机制

## Slice 的拷贝
