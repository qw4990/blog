# BoltDB实现

BoltDB本身代码写得还是很不错的, 就不过多赘述了, 仅仅是挑几个觉得写得比较不错的地方;

## Struct到二进制转换

在做结构体到二进制的转换时, BoltDB直接使用go的unsafe包, 从而避免了序列化/反序列化的开销;

大概如下:

```
func main() {
	type T struct {
		A int
		B float64
	}

	buf := make([]byte, 1000)
	write := (*T)(unsafe.Pointer(&buf[0]))
	write.A = 23
	write.B = 23.33

	read := (*T)(unsafe.Pointer(&buf[0]))
	fmt.Println(*read)
}
```

## 利用mmap直接管理分页

BoltDB利用mmap, 把分页的管理交给了操作系统;

同时, 利用unsafe, 也避免了序列化/反序列化:

```
type DB {
    data *[maxMapSize]byte
    ...
}

type page sturct {
    id pgid
    ...
}

func open(file String) *DB {
    ...
    buffer, _ := syscall.Mmap(...file...)
    db.data = (*[maxMapSize]byte)(unsafe.Pointer(&b[0]))
    ...
}

func (db *DB) page(id pgid) *page {
    pos := id * db.pageSize
    return (*page)(unsafe.Pointer(&db.data[pos]))
}
```



