## 1. 找到节点管理中适合Go优化的模块
1 调研用Go替换当前自动安装功能的可行性。
2 调研是否可以利用Go高并发特点，尝试替换PipeLine，提升节点管理并发安装量。

## 2.具体实施
### 2.1 验证：单线程对比

Python 单线程 1亿减到0 代码：

```
import datetime

def decrement(n):
    while n > 0:
         n -= 1

begin = datetime.datetime.now()
decrement(100000000)
end = datetime.datetime.now()

print(f"{(end - begin).microseconds}ms")
```

耗时不稳定，在4到8秒之间

![image.png](/uploads/929910BBCBB740918BE149F3B1DE4298/image.png)


Go 单线程 1亿减到0 代码：
```
package main

import "fmt"
import "time"

var c chan int

func decrement(n int) {
    for n > 0 {
        n -= 1
    }
}

func main() {
    start := time.Now()
    decrement(100000000)
    fmt.Println(time.Since(start))
}
```

耗时：

![image.png](/uploads/F717A7751B6B4ECFBEC6637C4078EF46/image.png)

可见在计算速度上，两者的差距比较大。

### 2.2 Python+Go优化Python单线程性能

我们试试看能否在Python中调用Go的方法，在Go中从1亿减到1。

首先，将刚刚go语言版的1亿减到1改为在一个函数中进行，并返回结果：

```
package main
import (
    "C"
    "time"
)
var c chan int
func decrement(n int) {
    for n > 0 {
        n -= 1
    }
}
//export count_time
func count_time() *C.char {
    start := time.Now()
    decrement(100000000)
    total_time := time.Since(start).String()
    return C.CString(total_time)
}
func main() {}
```

然后生成动态链接库以便Python调用Go里写的函数：

```
go build -buildmode=c-shared -o main.so count.go
```

这样会在当前文件夹中生成 main.so 和 main.h.

在Python中我们需要加载该生成的main.so动态链接库，并配置好输出变量的类型，最后调用方法得到结果：

```
import time
from ctypes import cdll, c_char_p
start = time.time()
# 加载动态链接库
lib = cdll.LoadLibrary('./main.so')
# 配置输出参数变量类型
lib.count_time.restype = c_char_p
# 调用方法
rest = lib.count_time()
end = time.time()
print(f"Go 内部执行时间：{rest}")
print(f"Python 整体执行时间: {end - start}s")
```

结果如下：

![2020062619370279.png](/uploads/C26F37403C634B1FA25A381B74E779F6/2020062619370279.png)

可以看到，使用这个方案将Python和Go两者结合起来的性能依然非常高，但就是多了一个生成和调用动态链接库的过程，增加了代码的耦合性。

其实，这也是C+Python的开发方式，只不过我们将C换成了Go，因为Go开发起来实在是舒服多了。

如果你的Python代码中有某个部分计算特别复杂，你可以尝试将其改写成go，通过动态链接库的方式调用go写的代码，将能大大提高性能。

### 2.3 Python与Go在CPU上的消耗对比

TODO...