在`for a,b := range c` 遍历中， a 和 b 在内存中只会存在一份，即之后每次循环时遍历到的数据都是以值覆盖的方式赋给 a 和 b，a，b 的内存地址始终不变。

以下是错误代码示例：
```
    p := []s2.Rect{}
	var rs []*s2.Rect
	for _, r := range p.Rect {
		rs = append(rs, &r)
	}
```
因为 r 只会分配一次，因此 `&r` 始终都指向同一个内存地址，即 rs 切片中的指针对象都指向同一个地址，那么在读取 rs 时都会读到同一个值，for 循环最后一次赋给 r 的值。

实际开发中这样的情况并不常见，如果将上面的错误代码改成如下两种方式就不会出错：
1. 
```
    p := []s2.Rect{}
	var rs []*s2.Rect
	for _, r := range p.Rect {
		// The for-range only allocate r var once, so &r will point to the same instance within this for statement,
		// make a copy of r then append &c to rs instead of using &r directly.
		c := r
		rs = append(rs, &c)
	}
```
`c := r` 会进行值拷贝，c 是一个全新的对象，但值与此时的 r 相同。
2. 
```
    p := []*s2.Rect{}
	var rs []*s2.Rect
	for _, r := range p.Rect {
		rs = append(rs, c)
	}
```
这个 for-range 中 `r` 是指针对象，r 始终是同一个对象，但其值是地址，`rs = append(rs, c)`进行值(地址)拷贝。

总而言之，在 `for a,b := range c` 遍历中，要注意 `&a` `&b` 这样的操作，对 a 或 b 进行取地址时取到的值(地址)始终相同，可能引发错误。