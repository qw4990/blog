## 简介

今天阅读sync.cond的源码时, 发现了一个叫copyChecker的对象;

发现它可以用来检验对象是否经过拷贝; 

代码非常短, 如下:

```go
type copyChecker uintptr
  
  func (c *copyChecker) check() {
  	if uintptr(*c) != uintptr(unsafe.Pointer(c)) &&
  		!atomic.CompareAndSwapUintptr((*uintptr)(c), 0, uintptr(unsafe.Pointer(c))) &&
  		uintptr(*c) != uintptr(unsafe.Pointer(c)) {
  		panic("sync.Cond is copied")
  	}
  }
```

用法如下:

```go
type T struct {
    checker copyChecker
}

func main() {
    t := new(T)
    t.checker.check() // no thing happen

    coppied := *t
    coppied.checker.check() // panic here
}
```

## 原理

copyChecker的原理其实很简单;

注意到copyChecker实际上是一个uintptr, 于是他会在第一次check时, 将自己的地址放入自己内;

既, 令 c = &c;

一旦发生了对象拷贝, 新对象中, c本身地址肯定会变, 但是c内的值, 还是从原来的对象拷贝而来;

于是如果发现\(c != 0 && c != &c\), 则说明发生了对象拷贝;



上述整个逻辑则能浓缩成copyChecker实现中的if语句;





