# 一些吐槽

这里是对golang的一些吐槽;

### 错误无法判断类型

error不能判断类型, 上层代码难以根据错误类型来进行不同的操作, 偶尔需要进行字符串比较, 如下:

```
data := makeData()
err := saveToDB(data)
if strings.Contains(err.Error(), "network") {
    // ... may retry again
}
if strings.Contains(err.Error(), "db error") {
    // ... 
}
```



这个问题目前有一些解决办法, 但都不彻底;

下面是官方库的解决办法:

1. 库函数实现自己的Error Struct, 如net.OpError;
2. 用户拿到err转换类型, 然后调用特定的接口;

如下:

```
err = netOp()
if netErr, ok := err.(net.OpError); ok {
    if netErr.Timeout() {
        // ... may retry later
    } else if netErr.Temporary() {
        // ... may retry again now
    }
}
```

然而这种解决办法, 一是麻烦, 二是很容易被绕过, 比如用户在netOp中自己包裹了一下错误:

```
func netOp() error {
    _, err := net.Dial("", "")
    if err != nil {
        return fmt.Errorf("net err: %s", err.Error())
    }
    return nil
}
```

上面的做法就没用了;

### init\(\)函数规范不强

golang提供了init\(\)函数, 该函数在包被加载时执行;

init\(\)的问题在于规范性不高, 你不知道应该在init\(\)内做什么操作;

另外在于init\(\)早于main\(\)执行, 这可能会让你在进行某些初始化操作时, 失败;

比如你可能想在init\(\)内读取配置文件, 但是可能你的flag.Parse\(\)写在了main\(\)里面, 于是在init\(\)内你可能根本解析不到配置文件路径;



官方对此的建议是, init\(\)中只做最基础的



