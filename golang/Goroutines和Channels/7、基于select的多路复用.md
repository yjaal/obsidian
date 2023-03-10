
下面的程序会进行火箭发射的倒计时。`time.Tick` 函数返回一个 `channel`，程序会周期性地像一个节拍器一样向这个 `channel` 发送事件。每一个事件的值是一个时间戳，不过更有意思的是其传送方式。

```go
// countdown1/countdown.go
func main() {
    fmt.Println("Commencing countdown.")
    tick := time.Tick(1 * time.Second)
    for countdown := 10; countdown > 0; countdown-- {
        fmt.Println(countdown)
        <-tick
    }
    launch()
}
func launch() {
	fmt.Println("Lift off!")
}
```

这里可能不太懂，没有往 `tick` 中写入数据，为啥可以直接读取？可以从源码中找到答案，其中会有一个 `sendTime` 方法，会定时往 `tick` 中写入当前时间，而且是一个无缓存的 `channel`。

现在我们让这个程序支持在倒计时中，用户按下 `return` 键时直接中断发射流程。首先，我们启动一个 `goroutine`，这个 `goroutine` 会尝试从标准输入中读入一个单独的 `byte` 并且，如果成功了，会向名为 `abort` 的 `channel` 发送一个值。

```go
abort := make(chan struct{})
go func() {
    os.Stdin.Read(make([]byte, 1)) // read a single byte
    abort <- struct{}{}
}()

fmt.Println("Commencing countdown. Press return to abort.")
select {
case <-time.After(10 * time.Second):
	// Do nothing.
case x := <-abort:
	fmt.Println("Launch aborted!")
	return
}
launch()
```

这里其实就是有两种方式来控制发射，一个是计数，另外一个是从键盘输入任意内容来暂停发射。这里使用了 `select` 语句，这是 `select` 语句的一般形式，和 `switch` 有点类似。

每一个 `case` 代表一个通信操作（在某个 `channel` 上进行发送或者接收），并且会包含一些语句组成的一个语句块。一个接收表达式可能只包含接收表达式自身（译注：不把接收到的值赋值给变量什么的），就像上面的第一个 `case`，或者包含在一个简短的变量声明中，像第二个 `case` 里一样；第二种形式让你能够引用接收到的值。

`select` 会等待 `case` 中有能够执行的 `case` 时去执行。当条件满足时，`select` 才会去通信并执行 `case` 之后的语句；**这时候其它通信是不会执行的**。一个没有任何 `case` 的 `select` 语句写作 `select{}`，会永远地等待下去。

下面这个例子更微妙。`ch` 这个 `channel` 的 `buffer` 大小是 1，所以会交替的为空或为满，所以只有一个 `case` 可以进行下去，无论 `i` 是奇数或者偶数，它都会打印 `0 2 4 6  8`。

```go
ch := make(chan int, 1)
for i := 0; i < 10; i++ {
    select {
    case x := <-ch:
        fmt.Println(x) // "0" "2" "4" "6" "8"
    case ch <- i:
    }
}
```

如果多个 `case` 同时就绪时，`select` 会随机地选择一个执行，这样来保证每一个 `channel` 都有平等的被 `select` 的机会。增加前一个例子的 `buffer` 大小会使其输出变得不确定，因为当 `buffer` 既不为满也不为空时，`select` 语句的执行情况就像是抛硬币的行为一样是随机的。

下面让我们的发射程序打印倒计时。这里的 `select` 语句会使每次循环迭代等待一秒来执行退出操作。


```go
func main() {
    abort := make(chan struct{})  
	go func() {  
	   os.Stdin.Read(make([]byte, 1))  
	   abort <- struct{}{}  
	}()

    fmt.Println("Commencing countdown.  Press return to abort.")
    tick := time.Tick(1 * time.Second)
    for countdown := 10; countdown > 0; countdown-- {
        fmt.Println(countdown)
        select {
        case <-tick:
            // Do nothing.
        case <-abort:
            fmt.Println("Launch aborted!")
            return
        }
    }
    launch()
}
```

`time.Tick` 函数表现得好像它创建了一个在循环中调用 `time.Sleep` 的 `goroutine`，每次被唤醒时发送一个事件。当 `countdown` 函数返回时，它会停止从 `tick` 中接收事件，但是 `ticker` 这个 `goroutine` 还依然存活，继续徒劳地尝试向 `channel` 中发送值，然而这时候已经没有其它的 `goroutine` 会从该 `channel` 中接收值了——这被称为 `goroutine` 泄露。

`Tick` 函数挺方便，但是只有当程序整个生命周期都需要这个时间时我们使用它才比较合适。否则的话，我们应该使用下面的这种模式：

```go
ticker := time.NewTicker(1 * time.Second)
<-ticker.C    // receive from the ticker's channel
ticker.Stop() // cause the ticker's goroutine to terminate
```

有时候我们希望能够从 `channel` 中发送或者接收值，并避免因为发送或者接收导致的阻塞，尤其是当 `channel` 没有准备好写或者读时。`select` 语句就可以实现这样的功能。`select` 会有一个 `default` 来设置当其它的操作都不能够马上被处理时程序需要执行哪些逻辑。

下面的 `select` 语句会在 `abort channel` 中有值时，从其中接收值；无值时什么都不做。这是一个非阻塞的接收操作；反复地做这样的操作叫做“轮询 `channel`”。

```go
select {
case <-abort:
    fmt.Printf("Launch aborted!\n")
    return
default:
    // do nothing
}
```

`channel` 的零值是 `nil`。也许会让你觉得比较奇怪，`nil` 的 `channel` 有时候也是有一些用处的。因为对一个 `nil` 的 `channel` 发送和接收操作会永远阻塞，在 `select` 语句中操作 `nil` 的 `channel` 永远都不会被 `select` 到。