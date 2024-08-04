# Go Map 源码解析

在我们开始对 map 的源码进行解析前，我想先梳理一些有关 map 的额外信息，希望的是能有助于阅者们少走弯路。
* 1、基于 map 的 Go 源码版本路径差异：
* * 1.1、Go1.10 及 以下版本的 map 源码路径：src/runtime/hashmap.go
* * 1.2、Go1.11 及 以上版本（目前截止到 Go1.23）的 map 源码路径：src/runtime/map.go
* 2、map 的 make 初始化创建的底层源码实现，是有两套方案：
* * 2.1、小容量 map 的初始化：func makemap_small() *hmap
* * 2.2、大容量 map 的初始化：func makemap(t *maptype, hint int, h *hmap) *hmap

***Now that the digression is over, let’s get back to the point.***

## 1、map 的数据结构
说起 map 这个概念，那就不得不提起国内版本中永远的“八股文之王”了，java 的 HashMap(jdK8)。虽说 java 因为 Oracle 的付费操作，而让国内的玩家们长期停滞于 JDK8 的版本。但我们也不得不承认，HashMap 的底层设计堪称一代经典之作。如果你已经有了对 HashMap 的了解，那么关于 Go map 的设计理念将会更容易去理解。没有也无妨，我们简单描述一下 HashMap 的数据结构，也算为 Go map 做个铺垫。

HashMap(jdk8) 图解

![](../../images/map001.png)

* HashMap，它是以 key-value 的方式进行数据存储，在底层的构成上，是基于 数组+链表 的方式来实现；
* 存储上，根据 key 进行一个哈希运算，从而得到一个关于 key 的 hash值。通过这个 hash值 来定位 key-value 在数组上的存储位置。理想情况下，自然是一个 hash值 只占一个位置。但现实往往是不遂人愿的，哈希运算的结果不可避免的在大数据压力下会造成重复的 hash值，这样的结果就是，虽然 key 不一致，但运算后的 hash值 却是一样的，这样就会造成过失性的数据覆盖，也就是我们常说的 哈希冲突；
* 所以为了容错这一不可避免的 哈希冲突，在 map 中，key 的 hash值 相同时，会在对应的位置，延伸出一个 链表，用于存储 相同 hash值 的不同 key-value；
* other：从 java8 开始，当链表长度超过一定阈值( 链表长度大于 8 )时，会转换为 红黑树，用以提高性能。
---
在简单梳理了一下 HashMap 的设计想法后，我们可以简单地总结一下有关 map 的组成模块
* 1、存储数据的方式：key-value
* 2、用于定位底层存储位置的标识，哈希值，将 key 进行哈希运算而得到；
* 3、底层的数据结构组成：
* * 3.1、在没有 hash 冲突的理想情况下，就是一个 数组，也可以说是一个 哈希表；
* * 3.2、在存在 hash 冲突的现实情况下，数组 + 链表(或任意存储模块，只要能存放值，理论上都可以)；
---
基于上述简单总结，我们终于是可以打开源码的文件了，/src/runtime/map.go (当前 Go1.20 版本)

而在打开源码后，我们要做的第一步，并不是马不停蹄地开始阅读源码，如果你是第一次打开源码，相信我，你根本不知道从哪里开始起头阅读。而这个时候，我的建议是，从开头第一行开始，不用管它是什么，因为源码开发者就是如此地纯粹，任何源码中的存在都是具有价值的。哪怕你英文阅读不好，也不要怕，open 百度，open Google。

只要你细细阅读 map.go 的注释，你就会发现，其实关于 Go map 的设计想法，Google的大佬们早在一开始就说明了，只是 use English，而我们只需要将其翻译成中文即可。

以下为 map.go 第7行 --- 第31行的简略译文：
* map 只是一个 哈希表。数据被安排存储到一个 buckets(桶) 数组。每个 bucket 包含 8对 key/value。哈希的低位被用于选择一个 bucket，每个 bucket 都包含着 每个 哈希的几个高位，高位的意义是用于区分单个 bucket 内部的条目。
* 如果一个 bucket 中的条目超过了 8对 key/value， 我们会链接一个额外的 bucket。
* 当 哈希表扩容 时，我们会分配一个 两倍大的 新的 bucket 数组。而后将数据逐渐从 旧的bucket数组 复制到 新的bucket数组。
* Map 迭代器遍历 buckets 数组并按遍历顺序返回 key（先在 桶编号，然后是 溢出链顺序，最后是 桶内索引）。为了保持迭代语义，我们从不移动存储在桶内 keys 的位置（如果我们移动了，那么 keys 可能会返回 0 或 2 次）。当我们扩容表时，迭代器仍在旧表上进行迭代，如果他们正在遍历的桶已经被移动("evacuated")到新的哈希表中，则必须检测新的哈希表。
---
从上面的简略译文，我们也就了解到 Go map 的一个大致模型了：
* Go map 本质上是一个由 bucket 组成的 哈希表；
* 每个 bucket 最多包含 8 对 key/value;
* key 的 哈希值，分为 高位 和 低位，低位用于索引 bucket 的位置，高位用于区分 bucket内部的条目位置；
* bucket 内部超过 8对 key/value 后，额外链接一个 bucket；

![](../../images/map002.png)

是不是发现 Go map 和 HashMap 有那么一点儿相似的味道了。

而在做了这么多铺垫后，我们终于可以开始美妙的源码阅读时间了。

先让我们整合一下 map.go 中的结构体
* 1、hmap：Go map 的头部数据结构，包含了 map 的元数据 和 状态信息
* 2、bmap：Go map 的 bucket 数据结构，由四个部分组成，tophash 为 bmap 的结构体部分，其他三个组成部分为编译器在底层为其内存布局而预留的空间
* * 2.1、tophash [bucketCnt]uint8 : 存储每个 key 的哈希值的高字节(top byte)；
* * 2.2、然后续接 最多 bucketCnt 个 keys；
* * 2.3、然后续接 最多 bucketCnt 个 elems；
* * 2.4、最后额外接一个 overflow(溢出桶)指针；
* 3、mapextra：Go map 的额外信息

```go
// Go map 的头部
type hmap struct {
	// 笔记: hmap 的格式在 cmd/compile/internal/reflectdata/reflect.go. 中被编码
	// 请确保与编译器的定义保持一致
	count     int // # 活跃值 == map的容量.  必须在第一个 (被 len() 内建函数使用)
	flags     uint8 // 标记字段，用于存储与 map 状态相关的标记位
    
    /* 
        B 表示 桶数量的以2为底的对数。
        也就是说，桶数量是 2 的 `B` 次方。

        `loadFactor` 为预定义的常量或变量，用于调整哈希表的 负载因子
    */
	B         uint8  // log以2为底，桶数量的对数 (可以存储 loadFactor * 2^B 个元素)
	noverflow uint16 // 溢出桶的大致数量
	hash0     uint32 // 哈希种子

	buckets    unsafe.Pointer // 桶数量的指针，规模为 2^B 。如果 count==0，则可能为 nil
	oldbuckets unsafe.Pointer // 旧的桶数组指针，其大小为当前数组的一半，仅在扩展过程中非空。（ 在哈希表扩展(rehash)过程中，指向旧的桶数组。当新的桶数组正在填充时，这个字段保存旧的桶数组 ）
	nevacuate  uintptr        // 搬迁进度计数器（小于此数值的桶已经搬迁了）在扩展过程中，指示已经完成搬迁的桶的数量

	extra *mapextra // 额外字段（包含一些非必要但有用的信息，如 溢出桶的列表）
}
```

```go
// Go map 中的 bucket
type bmap struct {
    /* 
    译文：
        tophash 通常情况下，存储了 bucket 中每个 key 的哈希值的 高字节(top byte)。
        如果 tophash[0] < minTopHash，则 tophash[0] 表示这个桶正处于迁移(evacuation)状态
    */
	tophash [bucketCnt]uint8
    /* 
    译文：
        再接着，bucketCnt 个 keys，然后是 bucketCnt 个 elems。
        笔记：将所有 keys 打包放一起，然后把所有 elems 放一起。这样做的话，虽然使得代码比起交替 key/elem/key/elem/... 更为复杂。
            但它能消除因内存对齐需求而产生的填充空间，从而更高效地利用内存。例如：map[int64]int8。
        再接着是一个 overflow(溢出桶) 指针。
    */
}
```
```go
// Go map 的额外数据，并不是所有的 map 都需要的
type mapextra struct {
	/* 
    译文：
        1、当键和值不包含指针时，可以标记桶类型为不包含指针，从而避免垃圾回收器扫描这些桶
        2、尽管如此，桶本身的 overflow 字段是一个指针，为了保持这些溢出桶的存活，需要将它们的指针存储在 hmap.extra.overflow 和 hmap.extra.oldoverflow 中。
        3、overflow 和 oldoverflow 字段分别用于 当前哈希表的溢出桶 和 旧哈希表的溢出桶
        4、通过这种方式，可以在迭代器中灵活地管理和引用这些溢出桶
    */
	overflow    *[]*bmap // 指向当前哈希表的溢出桶的切片
	oldoverflow *[]*bmap // 指向旧哈希表的溢出桶的切片。这在哈希表扩容时使用，以保持对老桶的引用，直到所有的旧数据被搬移到新的桶中

	// nextOverflow holds a pointer to a free overflow bucket.
	nextOverflow *bmap // 指向一个空闲的溢出桶，用于快速分配新的溢出桶
}
```
在对源码中的 struct 进行解析后，我们可以将上面的粗略 Go map 图解进行细分。
* 1、hmap 作为 Go map 的头部信息，也是它的表侧结构体。其中字段 buckets 是 指向底层的哈希表的指针引用；
* 2、底层的 哈希表 是一个 bucket 数组，容量规模为 2^B;
* 3、bucket 的桶结构体是由 bmap 组成，bmap 的结构体虽然只有 tophash 一个字段，但编译器会将 keys、elems、overflow的指针在内存中逐次接续起来，从而形成一个 bucket 的整体内存布局；
* 4、bmap 中 keys 和 elems 的排列方式是：key1/key2.../key8 + elem1/elem2/.../elem8;
* 5、hmap.extra.overflow 指向 当前哈希表的溢出桶的切片
![](../../images/map003.png)

在解析完 Go map 的底层数据结构后，我们应该需要注意到几个优秀的特异点

***第一个特异点：存储方式上的差异***

那就是在 bucket 中 keys、elems 的存储布局

不是我们想象中的 交替 key/elem/key/elem/... 

***而是 key1/key2.../key8 + elem1/elem2/.../elem8***  

注释文中也有提及，这样的存储方式有利于消除 字节对齐 的影响，从而提高内存效用。同时Google 的大佬们也十分贴心地给出了一个例子：map[int64]int8，我们就以此来验证一下上述两种存储布局方式的内存大小吧

关于内存中的字节对齐的详解，可以阅读 [字节对齐](alignment.md) 进行了解

这里简单讲一下内存中的字节对齐，我们就以 map[int64]int8 的两种存储方式为例。

字节对齐：为了内存布局 和 数据读取的需要，指对不同类型的组合，在同一存储空间内要保证内存占用的一致性，而对齐的标准则是以 类型中字节占比最大者决定的，而字节占比不足者，则需要通过填充字节来进行补齐（本质上就是 空间换时间）。

* 1、交替 key/elem/key/elem/... 这样的方式，我们可以抽象为类似一个结构体
```go
type mapS struct {
	k int64
	v int8
}
```
* 2、key1/key2.../key8 + elem1/elem2/.../elem8 则可以理解为，分别使用 []int64 来存储 keys，使用 []int8 来存储 elems
```go
var keys []int64
var elems []int8
```

```go
type mapS struct {
	k int64
	v int8
}
func main() {
	a := mapS{}
	// 计算 map 的内存占用
	mapSize := unsafe.Sizeof(a)
	fmt.Println("mapSize : ", mapSize)

	fmt.Println("k-size: ", unsafe.Sizeof(a.k))
	fmt.Println("v-size: ", unsafe.Sizeof(a.v))
}
/* 
mapSize :  16
k-size:  8
v-size:  1
*/
```
![](../../images/map004.png)
---

***第二个特异点：bmap 的内存布局***
```go
// A bucket for a Go map.
type bmap struct {
	tophash [bucketCnt]uint8
}
```
单从源码中来看，bucket 的 结构体 bmap 不就是只有一个 tophash 字段嘛。为什么注释描述中却提及了，keys、elems、overflow 的存储了。说是编译器的底层实现，我们如何来证实呢？
```go

const bucketCntBits = 3
const bucketCnt = 1 << bucketCntBits

// A header for a Go map.
type hmap struct {
	count      int // # live cells == size of map.  Must be first (used by len() builtin)
	flags      uint8
	B          uint8          // log_2 of # of buckets (can hold up to loadFactor * 2^B items)
	noverflow  uint16         // approximate number of overflow buckets; see incrnoverflow for details
	hash0      uint32         // hash seed
	buckets    unsafe.Pointer // array of 2^B Buckets. may be nil if count==0.
	oldbuckets unsafe.Pointer // previous bucket array of half the size, non-nil only when growing
	nevacuate  uintptr        // progress counter for evacuation (buckets less than this have been evacuated)
	extra      *mapextra      // optional fields
}

// mapextra holds fields that are not present on all maps.
type mapextra struct {
	overflow     *[]*bmap
	oldoverflow  *[]*bmap
	nextOverflow *bmap
}

// A bucket for a Go map.
type bmap struct {
	tophash [bucketCnt]uint8
}

func main() {
	// 构建一个 Go map (刚好一个 满 bucketCnt 的 bucket)
	a := make(map[int]int, bucketCnt)
	for i := 0; i < bucketCnt; i++ {
		a[i] = i * 10
	}
	// 利用反射来获取 Go map 的头部信息 hmap
	val := reflect.ValueOf(a)
	hdr := (*hmap)(unsafe.Pointer(val.Pointer()))
	fmt.Printf("Map metadata: count=%d, flags=%d, B=%d, noverflow=%d, hash0=%d\n",
		hdr.count, hdr.flags, hdr.B, hdr.noverflow, hdr.hash0)
	fmt.Println("----------------------------------------------------------------------------------------------")
	// 遍历 bucket 并打印内容
	bucketSize := uintptr(unsafe.Sizeof(bmap{}))
	for i := 0; i < 1<<int(hdr.B); i++ {
		// hmap.buckets 为 底层buckets数组的指针。叠加 第i个 bucket结构体的size，从而获取第i个 bucket 的起始地址
		bucketAddr := uintptr(hdr.buckets) + bucketSize*uintptr(i)
		// 获取 bmap
		bucket := (*bmap)(unsafe.Pointer(bucketAddr))
		fmt.Printf("Bucket %d: tophash=%v\n", i, bucket.tophash)
		// keys 的起始地址 = bucket的起始地址 + tophash的size
		keysStart := bucketAddr + unsafe.Sizeof(bucket.tophash)
		// elems 的起始地址 = keys的起始地址 + keys的size
		elemsStart := keysStart + bucketCnt*unsafe.Sizeof(int(0))
		fmt.Println("keys: ")
		for j := 0; j < bucketCnt; j++ {
			key := *(*int)(unsafe.Pointer(keysStart + uintptr(j)*unsafe.Sizeof(0)))
			fmt.Printf("	key[%d]: %d\n", j, key)
		}

		fmt.Println("elems: ")
		for j := 0; j < bucketCnt; j++ {
			elem := *(*int)(unsafe.Pointer(elemsStart + uintptr(j)*unsafe.Sizeof(0)))
			fmt.Printf("	elem[%d]: %d\n", j, elem)
		}
		// 溢出桶的指针地址 = elems 的起始地址 + elems的size
		overflowPtr := elemsStart + bucketCnt*unsafe.Sizeof(int(0))
		overflow := *(*uintptr)(unsafe.Pointer(overflowPtr))
		fmt.Printf("overflow pointer: %p\n", unsafe.Pointer(overflow))
	}
}
/* 
Map metadata: count=8, flags=0, B=0, noverflow=0, hash0=3136877097
----------------------------------------------------------------------------------------------
Bucket 0: tophash=[95 24 47 137 25 208 213 100]
keys: 
	key[0]: 0
	key[1]: 1
	key[2]: 2
	key[3]: 3
	key[4]: 4
	key[5]: 5
	key[6]: 6
	key[7]: 7
elems: 
	elem[0]: 0
	elem[1]: 10
	elem[2]: 20
	elem[3]: 30
	elem[4]: 40
	elem[5]: 50
	elem[6]: 60
	elem[7]: 70
overflow pointer: 0x0
*/
```

## 2、map 的常量配置
```go
const (
	// 表示每个桶（bucket）能容纳的键值对的数量的位数
	bucketCntBits = 3
	bucketCnt     = 1 << bucketCntBits // 每个桶的最大容量 1 << 3 ---> 8

	/* 
    loadFactorNum 和 loadFactorDen：这两个常量定义了触发哈希表增长的负载因子。负载因子是桶的平均负载量的最大值，当超过这个值时哈希表会进行扩展。

    这里的负载因子被定义为 13/2 = 6.5。即当平均每个桶超过6.5个键值对时，哈希表会触发扩展。
    */
	loadFactorNum = 13
	loadFactorDen = 2

	/* 
    maxKeySize 和 maxElemSize：定义了键和元素的最大大小。
    如果键或元素的大小超过这个值，将会被分配到堆上而不是内联存储。单位是字节。
    */
	maxKeySize  = 128
	maxElemSize = 128

	/* 
    dataOffset：定义了哈希桶结构体 bmap 的数据部分的偏移量。为了保证对齐，特别是在某些架构上（例如 amd64p32），这个偏移量需要是64位对齐的。
    */
	dataOffset = unsafe.Offsetof(struct {
		b bmap
		v int64
	}{}.v)

	// tophash 相关的常量，用于标识单元格的状态
	emptyRest      = 0 // 这个单元为空，且之后的单元（包括溢出桶）也是空的。
	emptyOne       = 1 // 这个单元为空
	evacuatedX     = 2 // key/elem 有效，已经被迁移到较大表的前半部分。
	evacuatedY     = 3 // key/elem 有效，已经被迁移到较大表的后半部分。
	evacuatedEmpty = 4 // 单元为空，桶已经被迁移。
	minTopHash     = 5 //  普通已填充单元的最小 tophash 值。

	// 常量是标志位，用于标识哈希表操作的不同状态
	iterator     = 1 // 可能有迭代器正在使用桶
	oldIterator  = 2 // 可能有迭代器正在使用旧桶
	hashWriting  = 4 // 表示有一个 goroutine 正在写入哈希表
	sameSizeGrow = 8 // 表示当前哈希表的扩展是扩展到与原来大小相同的新哈希表。

	// 定义了一个哨兵值，用于迭代器检查。这个值表示在特定条件下不进行检查。它是通过对指针大小进行位移操作计算出来的，确保了适用于不同架构。
	noCheck = 1<<(8*goarch.PtrSize) - 1
)
```
在常量配置中，我们应该需要着重关注两个值
* 1、负载因子 : 6.5
```go
// 负载因子 : 6.5 = loadFactorNum/loadFactorDen
const (
    loadFactorNum = 13
	loadFactorDen = 2
)
```
负载因子的作用：负载因子是桶的平均负载量的最大值，当超过这个值时哈希表会进行扩展。那么为什么会选择 6.5 作为这个阈值呢？其实这个在源码中也是早早就说明了因由的（mao.go 第33行 - 第54行）

总结一下就是，***负载因子是哈希表中元素数量与桶（bucket）数量的比值。过高的负载因子会导致大量的溢出桶（overflow buckets），而过低的负载因子会浪费大量空间。***

然后基于数值量化的测试结果，选择了一个最佳项
![](../../images/map005.png)
* %overflow: 使用溢出桶的百分比。意思是，当设定了一个溢出阈值后，有多少百分比的桶会在此阈值限制下触发溢出；
* bytes/entry: 每个键/元素对的开销字节数。负载因子增加时，虽然溢出桶增加，但每个条目的字节开销减少。
* hitprobe: 查找存在键时需要检查的条目数。命中探测次数随着负载因子的增加而增加，因为更多的元素会导致更多的碰撞。
* missprobe: 查找不存在键时需要检查的条目数。未命中探测次数随着负载因子的增加而增加，因为更多的元素会导致更多的碰撞。

总结：  
负载因子的选择需要在溢出桶的数量、空间开销和查找性能之间进行权衡。注释中的数据表明：
* 较低的负载因子（如4.0）会导致较少的溢出桶和较高的每个条目的字节开销，但查找性能更好（较少的探测次数）。
* 较高的负载因子（如8.0）会导致较多的溢出桶和较低的每个条目的字节开销，但查找性能较差（更多的探测次数）。

---
* 2、哈希桶结构体 bmap 的数据部分的偏移量  
bmap 的结构体是哈希桶的内部表示。主要由四个部分组成：
* * 2.1、元数据（tophash）
* * 2.2、键数组
* * 2.3、值数组
* * 2.4、overflow ：溢出桶指针(当键值对数量大于8后，额外添加一个溢出桶)
```go
/* 
unsafe.Offsetof 是一个 Go 语言中的内置函数，用于计算结构体字段相对于结构体起始地址的偏移量。
这里用于返回结构体字段相对于结构体起始地址的偏移量 v int64 = 8 bytes 正好等于 [8]uint8

dataOffset 确定了数据部分的起始位置，便于哈希表实现中访问桶中的键值对。
*/
const (
	dataOffset = unsafe.Offsetof(struct {
		b bmap
		v int64
	}{}.v)
)
```
---
## 3、map 的初始化
* map 的 make 初始化创建的底层源码实现，是有两套方案：
* * 1、小容量 map 的初始化：func makemap_small() *hmap
* * 2、大容量 map 的初始化：func makemap(t *maptype, hint int, h *hmap) *hmap


***小容量 map 的初始化***
```go
// makemap_small implements Go map creation for make(map[k]v) and
// make(map[k]v, hint) when hint is known to be at most bucketCnt
// at compile time and the map needs to be allocated on the heap.
/* 
译文：
	makemap_small 用于实现 Go map 的创建，通过 make(map[k]v) 和 make(map[k]v, hint) 。
	前提是，在编译时期 就已知 hint 最多为 bucketCnt。并且 map 需要被分配到 heap 上
*/
func makemap_small() *hmap {
	h := new(hmap)
	// fastrand() 是一个快速生成随机数的函数，用于初始化 map 的哈希种子（hash0）。
	h.hash0 = fastrand()
	return h
}
```
所以由注释可知，当 运行 make(map[k]v) 和 make(map[k]v, hint)，hint < bucketCnt(8) 时，底层调用的是 makemap_small()

这样做的好处是：
* 1、快速分配：直接调用 new(hmap) 在堆上分配内存，实现快速分配，简洁高效；
* 2、减少内存碎片：通过在堆上分配小型 map，可以有效减少内存碎片。这对于需要频繁创建和销毁小型 map 的场景非常有用。
---
***大容量 map 的初始化(通用型创建流程)***
```go

// makemap implements Go map creation for make(map[k]v, hint).
// If the compiler has determined that the map or the first bucket
// can be created on the stack, h and/or bucket may be non-nil.
// If h != nil, the map can be created directly in h.
// If h.buckets != nil, bucket pointed to can be used as the first bucket.
/* 
译文：
	1、makemap 实现了 Go map 中通过 make(map[k]v, hint) 创建 map 的功能
	2、如果编译器已确定 map 或 第一个 bucket 可以在栈上创建，那么 h and/or bucket 可能 非空。
	3、如果 h != nil，map 可以直接在 h 中 创建。
	4、如果 h.bucket != nil，那么可以使用 h.bucket 所指向的桶 来 作为第一个桶。
*/
/* 
params:
	t *maptype: 映射类型信息
	hint int: 提示的初始大小
	h *hmap: 可能是预先分配的map指针（如果不为 nil，可以直接在其中创建 map）
*/
func makemap(t *maptype, hint int, h *hmap) *hmap {
	// 计算内存需要 并 检测是否溢出
	mem, overflow := math.MulUintptr(uintptr(hint), t.bucket.size)
	// 溢出了 或 申请内存 大于 最大内存空间
	// 将 hint 赋值为 0，保证 map 创建完成 且 不会溢出
	if overflow || mem > maxAlloc {
		hint = 0
	}

	// initialize Hmap
	if h == nil {
		h = new(hmap)
	}
	h.hash0 = fastrand()

	// Find the size parameter B which will hold the requested # of elements.
	// For hint < 0 overLoadFactor returns false since hint < bucketCnt.
	/* 
		循环调用 overLoadFactor(hint, B)，直到 B 能够容纳 hint 个元素 
		overLoadFactor(hint, B) ：用于判断当前的 B 是否满足负载因子的要求
	*/
	B := uint8(0)
	for overLoadFactor(hint, B) {
		B++
	}
	h.B = B

	// allocate initial hash table
	// if B == 0, the buckets field is allocated lazily later (in mapassign)
	// If hint is large zeroing this memory could take a while.
	/* 
		if B == 0，则延迟分配桶数组，稍后在插入操作(mapassign)中再分配
		if hint 很大，那么清零这个内存可能需要一段时间
	*/
	if h.B != 0 {
		var nextOverflow *bmap
		// 调用 makeBucketArray(t, h.B, nil) 创建桶数组，并返回第一个空闲的溢出桶指针 nextOverflow
		h.buckets, nextOverflow = makeBucketArray(t, h.B, nil)
		// if nexeOverflow 不为空，则分配一个 mapextra，并赋值对应字段
		if nextOverflow != nil {
			h.extra = new(mapextra)
			h.extra.nextOverflow = nextOverflow
		}
	}

	return h
}
```
基于源码，我们可以对 Go map 通用的初始化创建流程进行一个总结：
* 1、计算所有内存并判定是否溢出；
* 2、初始化 hmap 并 赋值其 哈希种子；
* 3、计算哈希表的大小参数 B；
* 4、根据 B 初始化 哈希桶数组 并 处理溢出桶的管理器

对于 makemap 函数中的代码逻辑，前面的 1、2、3 点 都是对于数值的计算，目的只是为了给第4点，桶数组的构建服务的。毕竟 Go map 的本质就是一个
哈希表而已，所以我们还需进一步探查 Go map 的核心逻辑，makeBucketArray 函数
```go

// makeBucketArray initializes a backing array for map buckets.
// 1<<b is the minimum number of buckets to allocate.
// dirtyalloc should either be nil or a bucket array previously
// allocated by makeBucketArray with the same t and b parameters.
// If dirtyalloc is nil a new backing array will be alloced and
// otherwise dirtyalloc will be cleared and reused as backing array.
/* 
译文：
	1、makeBucketArray 为 map 的 buckets 初始化一个底层数组。
	2、1<<b 是分配的最小桶数量。
	3、dirtyalloc 要么为 nil，要么是一个 由 makeBucketArray函数 基于相同 t 和 b 参数而分配的 桶数组。
	4、如果 dirtyalloc 为 nil，则会重新分配一个底层数组，否则，就将 dirtyalloc 清空并重新作为底层数组使用。
*/
/* 
params:
	t *maptype : map 的类型信息
	b uint8 : 桶数量的对数
	dirtyalloc unsafe.Pointer : 一个指向先前桶数组的指针（可能为 nil）
*/
func makeBucketArray(t *maptype, b uint8, dirtyalloc unsafe.Pointer) (buckets unsafe.Pointer, nextOverflow *bmap) {
	// bucketShift(b) 通常返回 1 << b，表示需要的最小桶数量
	base := bucketShift(b)
	nbuckets := base
	// For small b, overflow buckets are unlikely.
	// Avoid the overhead of the calculation.
	/* 
		对于较小的 b，溢出桶不太可能（出现）。（直接过滤）避免计算开销
		这里选择 b >= 4 作为判定条件的原因：我认为是和上面的 makemap_small 形成呼应，因为Google的大佬们将 小于等于 bucketCnt(8 == 1 << 3)的容量，归结于小容量判定中了，而在 makemap_small 中是直接 new(hmap) 就快速结束了，那么证明，在其底层的数值验算中，b <= 3 的容量，是几乎不可能出现溢出桶的。

		故而，在通用型的 哈希表 创建流程中，将溢出桶的判定限制在了 b >= 4.
	*/
	if b >= 4 {
		// Add on the estimated number of overflow buckets
		// required to insert the median number of elements
		// used with this value of b.

		// bucketShift(b - 4) ：计算所需的溢出桶数量，并叠加到初始桶数量上
		nbuckets += bucketShift(b - 4)
		// 重新计算 总size ，并将其向上取整(roundupsize(sz))
		sz := t.bucket.size * nbuckets
		up := roundupsize(sz)
		// 数值不统一时，则根据向上取整后的数值，重新计算 初始桶的数量
		if up != sz {
			nbuckets = up / t.bucket.size
		}
	}

	/* 
	dirtyalloc 为 nil，则重新 new 一个底层数组
	dirtyalloc 不为 nil，则将其清空（根据是否包含指针进行判定） 并 重新作为底层数组
	*/
	if dirtyalloc == nil {
		buckets = newarray(t.bucket, int(nbuckets))
	} else {
		// dirtyalloc was previously generated by
		// the above newarray(t.bucket, int(nbuckets))
		// but may not be empty.
		buckets = dirtyalloc
		size := t.bucket.size * nbuckets
		if t.bucket.ptrdata != 0 {
			// memclrHasPointers : 用于清除包含指针的桶数组 
			memclrHasPointers(buckets, size)
		} else {
			// memclrNoHeapPointers : 用于清除不包含指针的桶数组
			memclrNoHeapPointers(buckets, size)
		}
	}
	// 如果 base != nbuckets，表示预分配了一些溢出桶
	if base != nbuckets {
		// We preallocated some overflow buckets.
		// To keep the overhead of tracking these overflow buckets to a minimum,
		// we use the convention that if a preallocated overflow bucket's overflow
		// pointer is nil, then there are more available by bumping the pointer.
		// We need a safe non-nil pointer for the last overflow bucket; just use buckets.
		/* 
		译文：
			1、我们预分配了一些溢出桶；
			2、为了将跟踪这些溢出桶的开销保持在最低限度；
			3、我们使用了一个约定，如果预分配的 溢出桶 的 overflow指针 是 nil，则可以通过指针递增来获取更多可用的溢出桶；
			4、对于最后一个溢出桶，我们需要一个安全的 非nil 指针，可以直接使用 buckets
		*/
		/* 
			add(buckets, base*uintptr(t.bucketsize)) : 将 buckets(桶数组的起始指针) 指针加上一个 base*uintptr(t.bucketsize) 的偏移量
			这样就可以计算得出 下一次 可使用的 预分配溢出桶的地址。			
		*/
		nextOverflow = (*bmap)(add(buckets, base*uintptr(t.bucketsize)))
		// 计算出预分配桶数组的最后一个桶的地址：buckets(桶数组的起始指针) + (分配的总桶数 - 1)*桶size
		last := (*bmap)(add(buckets, (nbuckets-1)*uintptr(t.bucketsize)))
		// 将最后一个桶的 overflow 赋上一个有效值，确保遍历时正确终止
		last.setoverflow(t, (*bmap)(buckets))
	}
	return buckets, nextOverflow
}
```
关于上述的 哈希表 初始化源码，其中关于 溢出桶的预分配机制，有两个比较复杂的点，我想费些口舌来整理一下。

***1 - if a preallocated overflow bucket's overflow pointer is nil, then there are more available by bumping the pointer.***  
译文：如果预分配的 溢出桶 的 overflow指针 是 nil，则可以通过指针递增来获取更多可用的溢出桶

我们梳理一下重点：

* 1、预分配的溢出桶的作用：我们知晓当 一个 bucket 装满后，数据就会溢出，这时数据就需要存入一个溢出桶中，而预分配的作用就是减少频繁的内存分配；
* 2、预分配的 溢出桶 overflow 为 nil 的意义，这里首先需要区分两种不同环境下的 溢出桶overflow为nil：
* * 2.1、常态情况下，常规 Go map 运行时，此时 Go map 中有一个 bucketA，当数据插入这个 bucketA 直到溢出时，如果此时 bucketA 的 overflow 仍为 nil，那么说明此时 Go map 中已经没有额外的预留空间用于数据分配了，只能向内存中重新申请了。
* * 2.2、在初始化 Go map 时的预留溢出桶机制下，此时的 溢出桶 属于预分配的，还未被使用，那么它的 overlflow 字段自然是nil 的
* 3、bumping the pointer(指针递增)的概念:  意味通过增加指针地址的方式来查找下一个内存位置。（这里是基于桶数组内存分配的连续性而得到的性质。由上文得知，桶数组是基于一个数量值( 本桶数 + 预分配的溢出桶数 )进行底层统一分配的。）

***2 - We need a safe non-nil pointer for the last overflow bucket; just use buckets.***  
译文：对于最后一个溢出桶，我们需要一个安全的 非nil 指针，可以直接使用 buckets

也是一样的，check 以下文字重点：

* 1、为什么最后一个溢出桶需要一个 安全的 非 nil 指针？
* * 上一段已经描述了，在 预分配时期，溢出桶的overfl 为 nil ，那么就代表后续还有更多的溢出桶了。所以为了确保系统在处理溢出桶时正常终止，最后一个溢出桶不能为 nil。
* 2、为什么可以直接使用 buckets？
* * 这里是一个取巧的方式，只要保证 指针指向的是一个 有效的 非 nil 值即可。

就此，我们结束了关于 Go map 的初始化创建流程和源码分析。而在我们阅读 makemap 源码时，在进入 makeBucketArray 前的注释中，我们注意到了一点，如果 B == 0，则会延迟分配桶数组，并在插入操作(mapassign)中再分配。

如此，我们关于 Go map 的下一关挑战开始了，插入操作(mapassign)

## 4、map 的数据查询
上面说了，我们想要挑战 插入操作(mapassign)，但不巧的事时有发生，我们发现直接挑战 插入操作(mapassign) 这个Boss，似乎有一点难度，我们当前等级还不够。于是，我们退而求其次，选择了先去刷刷小怪 ---> 查询操作(mapaccess)。

为什么说 插入操作(mapassign) 是个 Boss 呢，因为它有很多不同的招数啊：
* 1、根据提供的 key 去 map 中查询其 是否已存在；
* 2、如果 key 存在，则更新 key 的 value 
* 3、如果 key 不存在，则分配一个新的槽位来插入这对新的 key-value；
* * 3.1、还有隐藏大招-1：如果插入新的key-value溢出时，还要考虑溢出桶的问题；
* * 3.2、还有隐藏大招-2：如果插入新的key-value空间不足时，还要考虑扩容的问题；

这样看下来，是不是感觉这个 Boss 有点难打了。不利于一次性吸收，所以我们再看看 查询操作(mapaccess) 

而 查询操作(mapaccess) ，我们只需要做第一步，根据提供的 key 去 map 中查询其 是否存在 即可。是不是感觉要 easy 得多了。😜

但在我们想要阅读有关 查询操作(mapaccess) 的源码时，却是再次出现了问题（这小怪居然会一气化三清，wdnmd(我带你们打)！！！）。

在 Go1.20 的版本中，有关 查询操作(mapaccess)的操作，Google 的大佬们再次玩了些花活儿，将 mapaccess 细分为了三个分支；
```go
// (1)
// mapaccess1 returns a pointer to h[key].  Never returns nil, instead
// it will return a reference to the zero object for the elem type if
// the key is not in the map.
// NOTE: The returned pointer may keep the whole map live, so don't
// hold onto it for very long.
/* 
译文：
	mapaccess1 返回一个指向 h[key] 的指针。如果键不在map中，它不会返回nil，而是返回元素类型的零值对象的引用。
	笔记：返回的指针可能会导致整个map被保持在内存中，因此不要长时间持有这个指针。
*/
func mapaccess1(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer
// ---------------------------------------------------------------------------------------------

// (2)
// 从哈希表中获取指定键对应的值，同时返回一个 bool 值表示键是否存在（功能主体 与 mapaccess1 基本一致）
func mapaccess2(t *maptype, h *hmap, key unsafe.Pointer) (unsafe.Pointer, bool)
// ---------------------------------------------------------------------------------------------

// (3)
// returns both key and elem. Used by map iterator.
// 从哈希表中获取 指定键 和 对应值
func mapaccessK(t *maptype, h *hmap, key unsafe.Pointer) (unsafe.Pointer, unsafe.Pointer)
```
![](../../images/map006.png)
整理一番后发现，三个方法的实现逻辑大同小异，只是为了适应不同的 return 而独立了相应的逻辑。  
这里仅以 mapaccess1 为例来梳理 查询操作(mapaccess) 逻辑
```go
/* 
params:
	t *maptype: 哈希表的map类型
	h *hmap: Go map 的头部信息
	key unsafe.Pointer: 需要查询的key的指针
*/
func mapaccess1(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
	// 并发检测 和 内存安全检测 --------- start ---->
	// racereadpc 和 raceReadObjectPC 用于数据竞争检测。
	if raceenabled && h != nil {
		callerpc := getcallerpc()
		pc := abi.FuncPCABIInternal(mapaccess1)
		racereadpc(unsafe.Pointer(h), callerpc, pc)
		raceReadObjectPC(t.key, key, callerpc, pc)
	}
	// msanread 用于内存清理检测。
	if msanenabled && h != nil {
		msanread(key, t.key.size)
	}
	// asanread 用于地址清理检测。
	if asanenabled && h != nil {
		asanread(key, t.key.size)
	}
	// 并发检测 和 内存安全检测 --------- end ---->

	// 空 map 检查（map 为 nil 或 map 中 元素数量为 0） 
	if h == nil || h.count == 0 {
		if t.hashMightPanic() {
			t.hasher(key, 0) // see issue 23734
		}
		return unsafe.Pointer(&zeroVal[0])
	}
	// 并发写检测
	if h.flags&hashWriting != 0 {
		fatal("concurrent map read and map write")
	}
	// 计算哈希值：使用哈希函数计算 key 的哈希值
	hash := t.hasher(key, uintptr(h.hash0))
	// 计算桶掩码： 通过 bucketMask 计算桶数组的 掩码
	m := bucketMask(h.B)
	// 定位桶：通过 哈希值 和 掩码 计算 桶的索引，并定位到相应的桶
	b := (*bmap)(add(h.buckets, (hash&m)*uintptr(t.bucketsize)))

	// 处理查找时正在扩容中的旧桶
	if c := h.oldbuckets; c != nil {
		if !h.sameSizeGrow() {
			// There used to be half as many buckets; mask down one more power of two.
			m >>= 1
		}
		oldb := (*bmap)(add(c, (hash&m)*uintptr(t.bucketsize)))
		if !evacuated(oldb) {
			b = oldb
		}
	}

	// 查找元素 ---- start ---->
	// 计算 hash值 的 高字节(top byte) ---> 对应桶内的条目索引
	top := tophash(hash)
bucketloop:
// 遍历桶数组：遍历当前桶 及其对应的 溢出桶，查找匹配的键
	for ; b != nil; b = b.overflow(t) {
		for i := uintptr(0); i < bucketCnt; i++ {
			// 如果对应高字节不匹配 且 剩余部分为空，则跳出循环
			if b.tophash[i] != top {
				if b.tophash[i] == emptyRest {
					break bucketloop
				}
				continue
			}
			// 定位到当前桶内的 key
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
			// 如果 key 是间接的（指针），则解引用获取 实际key
			if t.indirectkey() {
				k = *((*unsafe.Pointer)(k))
			}
			// 比较 key 的实际值 是否相同
			if t.key.equal(key, k) {
				// 定位到当前桶内的 elem, 如果 elem 是间接的（指针），则解引用获取 实际elem
				e := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
				if t.indirectelem() {
					e = *((*unsafe.Pointer)(e))
				}
				return e
			}
		}
	}
	// 查找元素 ---- end ---->
	
	//没有找到，则返回 零值对象的指针 
	return unsafe.Pointer(&zeroVal[0])
}
```
在梳理了一番 查询操作(mapaccess) 的源码后，我们可以开始总结重点了
* 1、有关 定位桶 的逻辑解析：通过 哈希值 和 掩码 计算 桶的索引，并定位到相应的桶
* 2、
* 3、

## 5、map 的数据插入
```go

// Like mapaccess, but allocates a slot for the key if it is not present in the map.
// 与 mapaccess 相似，但如果 key 不在 map 中，则分配一个新的槽位给这个 key-value 对
/* 
params：
	t *maptype : 哈希表的 map 类型信息
	h *hmap : 指向 map 的头部信息 hmap 的指针
	key unsafe.Pointer : 指向要插入的 key 的指针
*/
func mapassign(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
	if h == nil {
		panic(plainError("assignment to entry in nil map"))
	}
	if raceenabled {
		callerpc := getcallerpc()
		pc := abi.FuncPCABIInternal(mapassign)
		racewritepc(unsafe.Pointer(h), callerpc, pc)
		raceReadObjectPC(t.key, key, callerpc, pc)
	}
	if msanenabled {
		msanread(key, t.key.size)
	}
	if asanenabled {
		asanread(key, t.key.size)
	}
	if h.flags&hashWriting != 0 {
		fatal("concurrent map writes")
	}
	hash := t.hasher(key, uintptr(h.hash0))

	// Set hashWriting after calling t.hasher, since t.hasher may panic,
	// in which case we have not actually done a write.
	h.flags ^= hashWriting

	if h.buckets == nil {
		h.buckets = newobject(t.bucket) // newarray(t.bucket, 1)
	}

again:
	bucket := hash & bucketMask(h.B)
	if h.growing() {
		growWork(t, h, bucket)
	}
	b := (*bmap)(add(h.buckets, bucket*uintptr(t.bucketsize)))
	top := tophash(hash)

	var inserti *uint8
	var insertk unsafe.Pointer
	var elem unsafe.Pointer
bucketloop:
	for {
		for i := uintptr(0); i < bucketCnt; i++ {
			if b.tophash[i] != top {
				if isEmpty(b.tophash[i]) && inserti == nil {
					inserti = &b.tophash[i]
					insertk = add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
					elem = add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
				}
				if b.tophash[i] == emptyRest {
					break bucketloop
				}
				continue
			}
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
			if t.indirectkey() {
				k = *((*unsafe.Pointer)(k))
			}
			if !t.key.equal(key, k) {
				continue
			}
			// already have a mapping for key. Update it.
			if t.needkeyupdate() {
				typedmemmove(t.key, k, key)
			}
			elem = add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
			goto done
		}
		ovf := b.overflow(t)
		if ovf == nil {
			break
		}
		b = ovf
	}

	// Did not find mapping for key. Allocate new cell & add entry.

	// If we hit the max load factor or we have too many overflow buckets,
	// and we're not already in the middle of growing, start growing.
	if !h.growing() && (overLoadFactor(h.count+1, h.B) || tooManyOverflowBuckets(h.noverflow, h.B)) {
		hashGrow(t, h)
		goto again // Growing the table invalidates everything, so try again
	}

	if inserti == nil {
		// The current bucket and all the overflow buckets connected to it are full, allocate a new one.
		newb := h.newoverflow(t, b)
		inserti = &newb.tophash[0]
		insertk = add(unsafe.Pointer(newb), dataOffset)
		elem = add(insertk, bucketCnt*uintptr(t.keysize))
	}

	// store new key/elem at insert position
	if t.indirectkey() {
		kmem := newobject(t.key)
		*(*unsafe.Pointer)(insertk) = kmem
		insertk = kmem
	}
	if t.indirectelem() {
		vmem := newobject(t.elem)
		*(*unsafe.Pointer)(elem) = vmem
	}
	typedmemmove(t.key, insertk, key)
	*inserti = top
	h.count++

done:
	if h.flags&hashWriting == 0 {
		fatal("concurrent map writes")
	}
	h.flags &^= hashWriting
	if t.indirectelem() {
		elem = *((*unsafe.Pointer)(elem))
	}
	return elem
}
```