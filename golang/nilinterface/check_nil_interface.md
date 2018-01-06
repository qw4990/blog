# nil interface检测问题
在golang中, 对interface进行是否为nil的检测, 有时候是不可靠的;

直接先下面这个例子;

```
package main

import "fmt"

type Namer interface {
	Name() string
}

type People struct {
	name string
}

func (p *People) Name() string {
	return p.name
}

func queryAgeByName(namer Namer) (age int, err error) {
	if namer == nil {
		return 0, fmt.Errorf("nil namer")
	}

	name := namer.Name() // <<<<<<<<<------ panic here
	// query by name ...
	fmt.Println(name)

	return 0, nil
}

func main() {
	var someone *People // not inited
	queryAgeByName(someone)
}
```
上述例子中, 我们在queryAgeByName的第一行, 便对namer进行了是否为nil的检测, 然而还是会产生panic;

原因是因为, 对于namer来说, 他本身不为nil, 但是他内部的"值"是nil的;

对于这种情况, 直接用 "interface == nil", 是检测不出来的, 需要注意;

# 利用反射检测interface内部值
按理说上述问题, 应该由queryAgeByName的外部进行保证并解决;

但是如果你非要在queryAgeByName内解决的话, 可以利用反射进行一次更强的检测, 如下;

```
package main

import "fmt"
import "reflect"

type Namer interface {
	Name() string
}

type People struct {
	name string
}

func (p *People) Name() string {
	return p.name
}

func queryAgeByName(namer Namer) (age int, err error) {
	if namer == nil {
		return 0, fmt.Errorf("nil namer interface")
	}
	if reflect.ValueOf(namer).IsNil() {
		return 0, fmt.Errorf("nil namer value")
	}

	name := namer.Name() // <<<<<<<<<------ panic here
	// query by name ...
	fmt.Println(name)

	return 0, nil
}

func main() {
	var someone *People // not inited
	queryAgeByName(someone)
}
```
于是上述代码会返回一个"nil namer value"的错误;

虽然上述方式能够彻底检测到interface内部值是否为nil, 但是不是很推荐;

一是因为反射会对性能有一定影响;

二是queryAgeByName作为一个函数, 他接受的参数为namer, 当他在进行namer.Name()时, 相当于在对另一个函数进行调用了; 而这另一个函数是否panic, 不应该由他去控制, 而是应该由传递给他namer的外部环境去保证;