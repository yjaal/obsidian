
一个简单的示例

`clock` 服务器每一个连接都会起一个 `goroutine`。在本节中我们会创建一个 `echo` 服务器，这个服务在每个连接中会有多个 `goroutine`。大多数 `echo` 服务仅仅会返回他们读取到的内容，就像下面这个简单的 `handleConn` 函数所做的一样：

```go
func handleConn(c net.Conn) {
    io.Copy(c, c) // NOTE: ignoring errors
    c.Close()
}
```

一个更有意思的 `echo` 服务应该模拟一个实际的 `echo` 的“回响”，并且一开始要用大写 `HELLO` 来表示“声音很大”，之后经过一小段延迟返回一个有所缓和的 `Hello`，然后一个全小写字母的 `hello` 表示声音渐渐变小直至消失，像下面这个版本的 `handleConn`：

```go
func echo(c net.Conn, shout string, delay time.Duration) {
    fmt.Fprintln(c, "\t", strings.ToUpper(shout))
    time.Sleep(delay)
    fmt.Fprintln(c, "\t", shout)
    time.Sleep(delay)
    fmt.Fprintln(c, "\t", strings.ToLower(shout))
}

func handleConn(c net.Conn) {
    input := bufio.NewScanner(c)
    for input.Scan() {
        echo(c, input.Text(), 1*time.Second)
    }
    // NOTE: ignoring potential errors from input.Err()
    c.Close()
}
```

我们需要升级我们的客户端程序，这样它就可以发送终端的输入到服务器，并把服务端的返回输出到终端上，这使我们有了使用并发的另一个好机会：

```go
func main() {
    conn, err := net.Dial("tcp", "localhost:8000")
    if err != nil {
        log.Fatal(err)
    }
    defer conn.Close()
    go mustCopy(os.Stdout, conn)
    mustCopy(conn, os.Stdin)
}
```

当 `main goroutine` 从标准输入流中读取内容并将其发送给服务器时，另一个 `goroutine` 会读取并打印服务端的响应。当 `main goroutine` 碰到输入终止时，这时程序就会被终止，尽管其它 `goroutine` 中还有进行中的任务。（在后面中引入了 `channels` 后我们会明白如何让程序等待两边都结束。）

下面这个会话中，客户端的输入是左对齐的，服务端的响应会用缩进来区别显示。 客户端会向服务器“喊三次话”：

```sh
$ go build gopl.io/ch8/reverb1
$ ./reverb1 &
$ go build gopl.io/ch8/netcat2
$ ./netcat2
Hello?
    HELLO?
    Hello?
    hello?
Is there anybody there?
    IS THERE ANYBODY THERE?
Yooo-hooo!
    Is there anybody there?
    is there anybody there?
    YOOO-HOOO!
    Yooo-hooo!
    yooo-hooo!
^D
$ killall reverb1
```

注意客户端的第三次 `sout` 在前一个 `sout` 处理完成之前一直没有被处理，这貌似看起来不是特别“现实”。真实世界里的回响应该是会由三次 `sout` 的回声组合而成的。为了模拟真实世界的回响，我们需要更多的 `goroutine` 来做这件事情。这样我们就再一次地需要 `go` 这个关键词了，这次我们用它来调用 `echo`：

```go
func handleConn(c net.Conn) {
    input := bufio.NewScanner(c)
    for input.Scan() {
        go echo(c, input.Text(), 1*time.Second)
    }
    // NOTE: ignoring potential errors from input.Err()
    c.Close()
}
```

`go` 后跟的函数的参数会在 `go` 语句自身执行时被求值；因此 `input.Text()` 会在 `main goroutine` 中被求值。现在回响是并发并且会按时间来覆盖掉其它响应了：

```sh
$ go build gopl.io/ch8/reverb2
$ ./reverb2 &
$ ./netcat2
Is there anybody there?
    IS THERE ANYBODY THERE?
Yooo-hooo!
    Is there anybody there?
    YOOO-HOOO!
    is there anybody there?
    Yooo-hooo!
    yooo-hooo!
^D
$ killall reverb2
```

让服务使用并发不只是处理多个客户端的请求，甚至在处理单个连接时也可能会用到，就像我们上面的两个 `go` 关键词的用法。然而在我们使用 `go` 关键词的同时，需要慎重地考虑 `net.Conn` 中的方法在并发地调用时是否安全，事实上对于大多数类型来说也确实不安全。我们会在下一章中详细地探讨并发安全性。