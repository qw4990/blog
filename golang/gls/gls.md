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

在Go1.8中, 不少包的API中都新增了一个参数, context, 如http和sql;

于是你可以把信息放入context, 并向下进行传递.

虽然GLS不被提倡, 但是该GLS实现的方式确实很cool\(tricky\), 故用这篇文档进行记录;

## GLS实现难点

GLS实现难点在于, 你无法通过runtime等官方库, 拿到当前goroutine的标识;

试想如果runtime提供了一个类似于GetGID\(\)之类的函数, 用来返回当前goroutine的ID; 那么这个问题就简单了, 直接维护一个全局map即可;

因此, GLS的另一种实现方式为直接修改GOROOT下的源码库, 实现一个类似于GetGID\(\)的函数.

## 不修改源码的GLS实现

下面直接描述这个方法

### 前提

虽然runtime不能拿到GID, 但是他能拿到当前的栈信息;

如: 能拿到"当前栈深度为10, 第10层在a10.go文件中的第27行, 第9层在a9.go文件的第78行, 等..."

### 思路

总体思路为:

1. 在每个协程开始时, 为其分配一个GID;
2. 在协程开始时, 强制增加几层特殊的调用, 将GID“编码”进调用栈中;
3. 根据GID, 维护一个全局的map;

这样, 在协程之后的任何地方, 通过runtime得到调用栈信息, 然后利用调用栈信息将GID还原出来, 再根据GID去全局map中存取数据.

上面讲得可能过于简单, 下面将举例解释;

再Go中, 使用go func\(\)语句新启协程;

现在假设, 我们把所有的逻辑, 都实现再task中;

于是, 通常情况下, 我们就直接写: go task\(\);

其调用栈大致如下:

![](/golang/gls_assets/import0.png)

现在， 我们实现一个函数， CodeGIDToStack\(task\);

他接受一个task, 新起一个协程来执行这个task;

并为这个协程分配一个GID, 并将其编码进这个协程的调用栈中;

整个代码片段大致如下:

```
var GIDCounter uint
type task func()

func stackTag0(t task) { t() }
func stackTag1(t task) { t() }

func CodeGIDToStack(t task) {
    gid := GIDCounter
    GIDCounter++
    go func() {
        if gid == 0 {
            stackTag0(t)
        } else if gid == 1 {
            stackTag1(t)
        } else {
            panic("gid is too large")
        }
    }()
}
```

我们暂且只考虑对0和1进行编码的情况;

当我们调用CodeGIDToStack时, task的栈就变成了下面这样:![](/golang/gls_assets/import1.png)现在考虑怎么将大于1的GID编码进栈中;

思路还是比较简单的, 假设我们想编码6;

其二进制表示为110, 则我们设法构成下面的调用栈:![](/golang/gls_assets/import2.png)代码大致如下:

```
var GIDCounter uint

type task func()

func stackTag0(remains uint, t task) {
	if remains == 0 {
		t()
	} else {
		if remains&1 == 0 {
			stackTag0(remains>>1, t)
		} else {
			stackTag1(remains>>1, t)
		}
	}
}

func stackTag1(remains uint, t task) {
	if remains == 0 {
		t()
	} else {
		if remains&1 == 0 {
			stackTag0(remains>>1, t)
		} else {
			stackTag1(remains>>1, t)
		}
	}
}

func CodeGIDToStack(t task) {
	gid := GIDCounter
	GIDCounter++
	go func() {
		if gid&1 == 0 {
			stackTag0(gid>>1, t)
		} else {
			stackTag1(gid>>1, t)
		}
	}()
}
```

### 具体实现

上小结描述了编码GID到调用栈的思路;

至于怎么利用调用栈还原, 利用runtime的哪些接口, 和其他一些优化;

请直接参考开头提供的两份代码库.



























































