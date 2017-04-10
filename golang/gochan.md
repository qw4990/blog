# nil chan & closed chan

channel是Go的重要抽象, channel的zero type为nil, 且channel可以被关闭;

那么, 结合对channel的读写操作, 应该注意会有下面4个神奇的现象:

1. 对closed channel写会产生panic;

2. 对closed channel读会得到对应类型的zero type;

3. 对nil channel写会阻塞;

4. 对nil channel读会阻塞;

这四个现象在官方的[specification](https://golang.org/ref/spec#Receive_operator)中也有说明, 分别在Send Operation和Receive Operation中, 有如下两句话:

Receiving from a nil channel blocks forever. A receive operation on a closed channel can always proceed immediately, yielding the element type's zero value after any previously sent values have been received.

A send on a closed channel proceeds by causing a run-time panic. A send on a nil channel blocks forever.

可惜的是只描述了现象, 没说明原因.  


我认为上述四种现象的原因, 并不是因为技术问题, 而是设计问题.

本文尝试对这四种现象的原因做一些猜测和分析.

本文分两大部分, 分别探讨closed channel和nil channel.  


# Closed Channel

## Send Operation On Closed Channel

对于closed channel进行写, 产生panic, 我认为是很自然的.

试想, 对于一个channel进行send operation, 无非有三种结果, "发送成功", "阻塞", "panic".

发送成功是不可能的, 于是需要在"阻塞"和"panic"选一个.

其实个人认为"阻塞"和"panic"都是可以的, 如果是"阻塞"的话, 那么对于下面这种情况:

```
select {
    case chA <- msg:
        // do something
    case chB <- msg:
        // do something
    default:
        // do something
}
```

其实是更好的.

如果chA的接收方不想接受数据了, 则直接close\(chA\)即可.

不需要通知发送方, 因为到chA的send operation一定会被阻塞.

但事实上, 如果你直接这么做, 那么会产生panic. \(见下面这个demo\)

```
package main

import (
	"fmt"
	"time"
)

type receiver struct {
	name    string
	ch      chan string
	chClose chan struct{}
}

func newReceiver(name string) *receiver {
	return &receiver{
		name,
		make(chan string),
		make(chan struct{}),
	}
}

func (r *receiver) run() {
	go func() {
		for {
			select {
			case msg := <-r.ch:
				fmt.Println(r.name+" : ", msg)
				time.Sleep(time.Millisecond * 20)
			case <-r.chClose:
				break
			}
		}
	}()
}

func (r *receiver) close() {
	close(r.ch)
	r.chClose <- struct{}{}
}

func send(cha, chb chan string) {
	select {
	case cha <- "xixi":
	case chb <- "xixi":
	}
}

func main() {
	ra := newReceiver("aaa")
	rb := newReceiver("bbb")
	ra.run()
	rb.run()

	ra.close()
	send(ra.ch, rb.ch)
	send(ra.ch, rb.ch)
	time.Sleep(time.Second)
}
```

在"阻塞"和"panic"之间, 设计者选择了后者, 我猜测是为了go的安全性.

一是阻止了上述这种比较tricky的用法;

二是强迫程序员更加细心的梳理go routine间的依赖关系, 避免由于阻塞造成资源浪费, 甚至是死锁.

## Receive Operation On Closed Channel

对closed channel的发送会造成panic, 为什么接受就会返回对应类型zero type?

"对应类型zero type", 先举几个例子:

chan \[\]byte, 返回nil;

chan int, 返回0;

chan string, 返回"";  


和panic比起来, 这个操作结果, 显得非常"不自然".

我探究和揣度了一下, 最后做出的结论是, 这个行为是为了"广播".

在Go程序中, 多个goroutine经常通过channel交换数据, 这个数据交换可能是多对多的, 如A同时接受B, C的数据.

那么当A决定退出时, 怎么告诉B, C自己不想要数据了?

首先B和C对A发送数据, 一般是有一段类似于下面的gorouting:  


```
for {
	select {
	case chA <- msg:
	default:
	}
}
```

当然, 让B, C知道A已经停止的方法有很多, 大致说几种, 来和最后的方案做对比:

### 方案1

让B和C分别提供一个channel给A, 当A结束时, A向这些channel内发送信号;

于是, B和C的程序会变成这样:  


```
for {
	select {
	case chA <- msg:
    case <-closeA:
        break
	default:
	}
}
```

这样做的缺点很明显, 既B和C需要向A进行"注册".

如果后面有D, E, F, Z, X都准备向A发送数据, 那么他们都得向A提供一个channel, 让A在结束时发送信号.

### 方案2

方案2能够使得B, C不需要向A进行"注册".

我们利用"广播"的概念, golang的官方lib中, 有"广播"的概念, 在sync.Cond中提供.

我们给A增加一个sync.Cond, 命名为AClosed.

当B准备给A发送信息时, 先自己创建一个channel, closeA, 然后获取A的sync.Cond.

接下来启动两个goroutine, 分别为:

```
================goroutine1================
for {
	select {
	case chA <- msg:
    case <-closeA:
        break
	default:
	}
}
================goroutine2================
AClosed := A.AClosed
AClosed.Wait()
closeA <- struct{}{}
```

goroutine1和之前一样, 用来向A发数据, 并检测A是否已经停止, 唯一不一样的是, 之前的closeA是从A中直接获取的. 

而这里的closeA是B自己维护的. goroutine2利用了Cond提供的广播, 对A进行检测, 并向自己维护的closeA发送信息. 

当A结束时, 只需要A.AClosed.Broadcase\(\)即可让B, C都停止.

方案二不需要注册了, 但是却变得很麻烦, 你需要另起一个goroutine.

而你必须要另起goroutine的原因, 是因为, go的lib虽然提供了"广播"的概念, 但是其API不可直接用在select中.

如果sync.Cond.Wait\(\)直接返回的是一个channel, 就好了, 那么你可以不用起goroutine2, 直接把goroutine1写为:

```
for {
	select {
	case chA <- msg:
	case <-A.AClosed.Wait():
		break
	default:
	}
}
```

然而我觉得让sync.Cond.Wait\(\)返回一个channel, 是不可能实现的.

这是go channel本身设计上的原因.

### 方案3:

现在利用go channel的性质来产生"广播".

我们让A多维护一个channel, Done, 注意, 这是一个channel.

然后一切都好办了, B, C的代码直接如下:

```
for {
	select {
	case chA <- msg:
	case <-A.Done:
		break
	default:
	}
}
```

当A结束时, 直接close\(A.Done\), B和C会收到一个zero type, 然后break.

"广播"的功能得以实现, 且能够配合select直接使用.  


但是用channel进行"广播", 其缺点也显而易见, 那就是只能广播一次.

如果想实现多次广播, 那就得去关闭一个"已经被关闭的channel", 这非常不自然, 不应该去这样定义close操作.

当然, 你也可以用一些tricky的方法来达到多次广播, 如把A.Done搞成一个channel的指针, 在关闭过后马上再给他make一个新的.

但是这又会带来许多的危险性和复杂性, 因此也不推崇.

所以, 方案2的最后我觉得"sync.Cond.Wait\(\)返回一个channel, 是不可能实现", 就是上述原因.

### 方案3的实现

方案3其实已经在go的context包中实现了, [Context](https://godoc.org/golang.org/x/net/context#Context)中的Done\(\), 返回的就是这样作用的一个channel.  
 另外, 在[这篇文档](https://blog.golang.org/pipelines)中, 也描述了该做法.

# Nil Channel

select和case的组合实际就是一套"门卫指令" [\(Guarded Command\)](https://en.wikipedia.org/wiki/Guarded_Command_Language).

在Hoare的1978年那篇[经典的CSP论文](http://dl.acm.org/citation.cfm?doid=359576.359585)里, 他一开始设计的模型中, 是有"门卫指令"的概念的, 对channel的io操作, 是可以和一些表达式一起使用的.

如你应该可以写类似于下面的代码:  


```
for {
    select {
    case a > b && ch1 <- data:
    case a < b && ch2 <- data:
    default: // a==b
        ch3 <- data
    }
}
```

但是在目前的go中, 很可惜, 这样的语法是不允许的.

目前不允许出现这样语法的原因, 我揣测可能有下面2个原因:

1\) 技术原因, 但是不是绝对的技术原因, 意思是该功能可以实现, 但是会降低编译效率等;

2\) 设计原因, 为了维持go语法简洁性, 语义简单性, 避免出现过于tricky的代码和bug.

但是为了满足对门卫指令的一定需求, 从而定义了该规则, 某些时候, 你可以利用该规则写出一些简洁的代码.

如你想实现这么一段逻辑:

A不断的从chA内读取数据, 但是在某种特定条件下, 会设置超时, 于是, 你可以这样写:

```
for {
	var timeoutCh <-chan Time
	if timeoutFlag {
		timeoutCh = time.After(TIME_OUT)
	}
	select {
	case <-chA:
	case <-timeoutCh:
		log("Timeout.")
	}
}
```

关于"门卫指令"的思路来自[这里的讨论](https://groups.google.com/forum/#!topic/golang-nuts/QltQ0nd9HvE).  


好吧, 说实话, 其实对于nil channel的操作, 我更倾向于产生panic, 我感觉产生panic会安全一点.

而上述说的用于一定程度上实现"门卫指令", 能够在某些时候方便写码, 也只是说法之一, 我目前也不是很认同这个说法.

至于真正的原因, 或许得去问问三位设计者大人了, 或许也就只是一个"tea or coffee"的问题而已.

