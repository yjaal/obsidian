
如果你使用过操作系统或者其它语言提供的线程，那么你可以简单地把 `goroutine` 类比作一个线程，这样你就可以写出一些正确的程序了。`goroutine` 和线程的本质区别会在后面章节中讲。

当一个程序启动时，其主函数即在一个单独的 `goroutine` 中运行，我们叫它 `main goroutine`。新的 `goroutine` 会用 `go` 语句来创建。在语法上，`go` 语句是一个普通的函数或方法调用前加上关键字 `go`。`go` 语句会使其语句中的函数在一个新创建的 `goroutine` 中运行。而 `go` 语句本身会迅速地完成。

```go
f()    // call f(); wait for it to return
go f() // create a new goroutine that calls f(); don't wait
```

下面的例子，`main goroutine` 将计算菲波那契数列的第 45 个元素值。由于计算函数使用低效的递归，所以会运行相当长时间，在此期间我们想让用户看到一个可见的标识来表明程序依然在正常运行，所以来做一个动画的小图标：

```go
func main() {
    go spinner(100 * time.Millisecond)
    const n = 45
    fibN := fib(n) // slow
    fmt.Printf("\rFibonacci(%d) = %d\n", n, fibN)
}

func spinner(delay time.Duration) {
    for {
        for _, r := range `-\|/` {
            fmt.Printf("\r%c", r)
            time.Sleep(delay)
        }
    }
}

func fib(x int) int {
    if x < 2 {
        return x
    }
    return fib(x-1) + fib(x-2)
}
```

动画显示了几秒之后，`fib(45)` 的调用成功地返回，并且打印结果：

```go
Fibonacci(45) = 1134903170
```

然后主函数返回。主函数返回时，所有的 `goroutine` 都会被直接打断，程序退出。除了从主函数退出或者直接终止程序之外，没有其它的编程方法能够让一个 `goroutine` 来打断另一个的执行，但是之后可以看到一种方式来实现这个目的，通过 `goroutine` 之间的通信来让一个 `goroutine` 请求其它的 `goroutine`，并让被请求的 `goroutine` 自行结束执行。