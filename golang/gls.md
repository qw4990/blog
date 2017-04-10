# Goroutine Local Storage

## 简介

该篇文档描述一种不需要修改GO源码的GLS实现方式;

我目前从两个地方看到了方法的实现:

1. github.com/jtolds/gls
2. github.com/xiezhenye/gls

第一库较早, 第二个更加简单;

总的来说, 两个库的代码都只有几百行, 但是都相当tricky;

故写下这篇文档用来描述其思路.

## 什么是GLS

全称为协程本地存储\(goroutine local storage\);

说通俗一点, 你可以直接是为为每个goroutine绑定了一个map;

在该goroutine的存在时间中, 你可以调用GLS的接口, 向这个map中存入或读取数据;

每个goroutine的数据都是相互独立的;

下面是一个例子:

```
pcakge main

func setGLS(key, val string) {
    // impl GLS here
}

func getGLS(key string) string {
    // impl GLS here
    return ""
}

func deepCall(index int, expect string) {
    if index == 10 {
        if getGLS("key") != expect {
            panic("err")
        }
    } else {
        deepCall(index+1, expect)
    }
}

func main() {
    go func() {
        setGLS("key", "goroutine1")
        deepCall(1, "goroutine1")
    }()
    go func() {
        setGLS("key", "goroutine2")
        deepCall(1, "goroutine2")
    }()
}
```

有了GLS, 可以直接将数据和当前的goroutine进行绑定;

比如你当前的goroutine将会进行一系列很深的调用, 你想控制调用的超时;

于是你可以这样:

1. 调用开始时, 利用setGLS, 放入你的调用起始和超时时间;
2. 在每层调用中, 利用getGLS取出起始和超时时间;
3. 进行检测, 发现如果已经超时, 则直接返回, 不再向下进行调用.

**注意:**

GLS主要是解决同一个协程中, 调用链信息的传递问题;

在Go1.8中, 已经有了新的解决方案, 即context;

在Go1.8

![](/golang/gls_assert/import.png)

