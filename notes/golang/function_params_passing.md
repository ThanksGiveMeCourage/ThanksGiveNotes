# Go 的函数传参都是 值拷贝

### [在 Go 中，函数传参都是通过值拷贝的，这意味着当一个变量被传递给函数时，函数接收的是该变量的副本，而不是原始变量本身]，先说明，这个结论是正确的。

但相信你也遇到过这样的情况，当传递引用类型（如 切片、映射、指针、通道等）时，在函数内的对其传入的变量进行修改后却会影响到源数据身上，这个行为逻辑不就是 引用(拷贝)传递的表现吗？  

如果你这样想，你就被其表现所迷惑了，其根本原因是：[引用类型传递函数时，依旧是值拷贝，只是由于这些类型本身是引用(即指向底层数据的指针)，因此通过这个副本引用到底层的数据，依旧是本源]

下面我们将分从 两个方面证明上述结论

###  基本类型 和 值类型

对于基本类型（如 int、float、bool 等）和值类型（如结构体），传递给函数时，会创建一个该变量的副本。函数对该副本的任何修改都不会影响到原始变量。
```go
func modifyValue(s SDemo) {
	s.x = 0
	s.y = ""
}
func main() {
	s := SDemo{111, "11111"}
	modifyValue(s)
	fmt.Println(s)
}
/* 
{111 11111}
*/
```
### 引用类型

对于引用类型（如切片、映射、指针、通道等），传递给函数时，依然是值拷贝。但由于这些类型的值本身是引用（即指向底层数据的指针），因此通过这个副本引用到的底层数据可以被修改。
```go
func modifyReference(s []int) {
	fmt.Println("--------------------------------------------------------------------------------")
	fmt.Printf("2 --- slice-address: %p, origin-array-address: %p\n", &s, &s[0])
	fmt.Println("--------------------------------------------------------------------------------")
	s[0] = 100
}

func main() {
	s := []int{1, 2, 3}
	fmt.Printf("before call function, slice: %v\n", s)
	// 查看切片变量的地址 和 源数据存储的底层数组的地址
	fmt.Printf("1 --- slice-address: %p, origin-array-address: %p\n", &s, &s[0])

	modifyReference(s)

	fmt.Printf("after call function, slice: %v\n", s)

	// 查看切片变量的地址 和 源数据存储的底层数组的地址
	fmt.Printf("3 --- slice-address: %p, origin-array-address: %p\n", &s, &s[0])
}
/* 
before call function, slice: [1 2 3]
1 --- slice-address: 0xc0001acdb0, origin-array-address: 0xc0001d8b28
--------------------------------------------------------------------------------
2 --- slice-address: 0xc0001acde0, origin-array-address: 0xc0001d8b28
--------------------------------------------------------------------------------
after call function, slice: [100 2 3]
3 --- slice-address: 0xc0001acdb0, origin-array-address: 0xc0001d8b28
*/
```
这里我们可以看出，引用类型其实是由两部分组成的
* 表侧：引用部分（或 引用指针 或引用结构体），例如：slice，是一个 slice 结构体(详情可以查看: [Go-正谈-slice](slice.md))
```go
/* 
src/runtime/slice.go
*/
type slice struct {
	array unsafe.Pointer // 指向内存中底层数组的指针
	len   int // 当前切片的长度 
	cap   int // 当前切片的容量
}
```
* 里侧：本源存储于内存中的数据结构，例如：slice中的数据是一个存储在内存中的 [底层数组] 

所以我们得知了引用类型在函数传递时，其实是对表侧的引用部分进行了一次值拷贝，而这个副本中的数据与其传入参数一致，这样的话，其中的地址指针依旧指向了同源数据，所以可以间接影响同源数据的修改。