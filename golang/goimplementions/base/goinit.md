# 简介
本小节简单说明:
1. go的启动过程;
2. go对TLS的作用;

该小节会简单提及一些调度相关概念, 不过不用过度在意, 知道线程和协程的概念即可;

# 汇编启动
启动代码被定义在runtime/asm_{platform}.s的汇编文件中, 各个平台有不同的启动逻辑, 他们都被定义在rt0_go这个函数中;

我这里以amd64.s的代码为例, 整体看一遍启动流程;

### 初始化参数
```
TEXT runtime·rt0_go(SB),NOSPLIT,$0
	// copy arguments forward on an even stack
	MOVQ	DI, AX		// argc
	MOVQ	SI, BX		// argv
	SUBQ	$(4*8+7), SP		// 2args 2auto
	ANDQ	$~15, SP
	MOVQ	AX, 16(SP)
	MOVQ	BX, 24(SP)
    ...
```
这一段的主要作用是初始化argc和argv, 并把他挪动到相应的栈位置;

### 体系结构检查
接下来是一段体系结构的逻辑, 比如cpu信息等, 我们直接跳过;

### TLS
之后是设置TLS的逻辑, 如下:
```
    ...
	LEAQ	runtime·m0+m_tls(SB), DI
	CALL	runtime·settls(SB)

	// store through it, to make sure it works
	get_tls(BX)
	MOVQ	$0x123, g(BX)
	MOVQ	runtime·m0+m_tls(SB), AX
	CMPQ	AX, $0x123
	JEQ 2(PC)
	MOVL	AX, 0	// abort
    ...
```
下一章会说到runtime对TLS的运用, 它非常重要, 这里先简单提一句: TLS会被runtime用来存储"当前被执行的goroutine指针";

这里重点关注TLS的初始化;
```
	LEAQ	runtime·m0+m_tls(SB), DI
	CALL	runtime·settls(SB)
```
第一句话相当于 DI = &runtime.m0.m_tls, runtime.m0是定义在runtime/proc.go中的一个对象, 用来表示第一个系统启动的线程;

而m_tls是定义在m中的一段空间, 这里将它作为该线程的TLS空间;

m的大致定义如下:
```
type m struct {
    ...
    tls [6]uintptr  // thread-local storage (for x86 extern register)
    ...
}
```

接下来是对TLS的测试逻辑:
```
	get_tls(BX)                         // 把tls的值写入BX, 也就是BX = &m0.tls
	MOVQ	$0x123, g(BX)               // *BX = 0x123
	MOVQ	runtime·m0+m_tls(SB), AX    // AX = m0.tls
	CMPQ	AX, $0x123                  // AX == 0x123
```

### 初始化m0, g0
接下来是初始化m0, g0, 他们都是被定义在runtime/proc.go里面的全局变量, 分别表示第一个被启动的线程和协程;
```
	// set the per-goroutine and per-mach "registers"
	get_tls(BX)
	LEAQ	runtime·g0(SB), CX      // CX = &g0
	MOVQ	CX, g(BX)               // *BX = TLS = &g0
	LEAQ	runtime·m0(SB), AX      // AX = &m0
	
    // save m->g0 = g0
	MOVQ	CX, m_g0(AX)            // m0.g0 = &g0
	// save m0 to g0->m
	MOVQ	AX, g_m(CX)             // g0.m = &m0
```

### 真正的初始化
接下来才是真正的初始化, 也就十几二十行;
#### 体系结构检查
首先是做一些最基本的体系结构的检查:
```
	CLD				// convention is D is always left cleared
	CALL	runtime·check(SB)
```
check被定义在runtime/runtime1.go中;

#### 参数初始化
接下来是参数初始化:
```
	MOVL	16(SP), AX		// copy argc
	MOVL	AX, 0(SP)
	MOVQ	24(SP), AX		// copy argv
	MOVQ	AX, 8(SP)
	CALL	runtime·args(SB)
```
根据第一段汇编代码, argc和argv当前分别在16(SP)和24(SP)中;

这里移动到0(SP)和8(SP)的位置, 然后调用args;

args被定义在runtime/runtime1.go中, 定义如下:
```
func args(c int32, v **byte)
```
其argc和argv初始化的栈位置, 刚好对应这俩参数, 如同第一篇汇编讲解的那样;

#### 调度器初始化
```
	CALL	runtime·osinit(SB)
	CALL	runtime·schedinit(SB)
```
这俩函数被定义在runtime/proc.go中;

schedinit非常重要, 会执行各种各样的初始化操作, 下面是其内部实现的一部分:
```
func schedinit() {
    ...
	tracebackinit()
	moduledataverify()
	stackinit()
	mallocinit()
	mcommoninit(_g_.m)
	alginit()       // maps must not be used before this call
	modulesinit()   // provides activeModules
	typelinksinit() // uses maps, activeModules
	itabsinit()     // uses activeModules

    ...

	goargs()
	goenvs()
	parsedebugvars()
	gcinit()
    ...
}
```
没必要全看, 明白其大概过程即可;

#### 创建主协程
之后并不会直接执行用户的main函数, 而是将其封装为一个协程, 丢入执行队列, 代码如下:
```
	// create a new goroutine to start program
	MOVQ	$runtime·mainPC(SB), AX		// entry
	PUSHQ	AX
	PUSHQ	$0			// arg size
	CALL	runtime·newproc(SB)
	POPQ	AX
	POPQ	AX
```
其中mainPC是一个全局变量, 被定义在runtime/asm_{platform}.s中, 如下:
```
DATA	runtime·mainPC+0(SB)/8,$runtime·main(SB)
```
编译器会把main的地址, 写入到这个变量内;

上面逻辑中, 先把main的地址和main的参数大小入栈, 然后再调用runtime.newproc;

runtime.newproc的参数定义正好符合这个栈结构:
```
func newproc(siz int32, fn *funcval)
```
newproc会把fn代表的函数, 包装成一个协程, 并丢入执行队列;

#### 进入调度逻辑
最后执行
```
	// start this M
	CALL	runtime·mstart(SB)
```
mstart被定义在runtime/proc.go中, 会将当前的线程代入调度循环逻辑;

进入调度循环逻辑后, 该线程会不断的从执行队列中拿取协程并执行;

首先被拿取的, 便是之前被包装的用户main协程, 至此runtime启动过程结束;

# 其他
https://en.wikipedia.org/wiki/Thread-local_storage