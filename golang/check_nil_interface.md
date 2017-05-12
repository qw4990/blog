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