一个简单的案例

网络编程是并发大显身手的一个领域，由于服务器是最典型的需要同时处理很多连接的程序，这些连接一般来自于彼此独立的客户端。在本小节中，我们会讲解 go 语言的 `net` 包，这个包提供编写一个网络客户端或者服务器程序的基本组件，无论两者间通信是使用 `TCP、UDP` 或者 `Unix domain sockets`。在第一章中我们使用过的 `net/http` 包里的方法，也算是 `net` 包的一部分。

我们的第一个例子是一个顺序执行的时钟服务器，它会每隔一秒钟将当前时间写到客户端：

```go
// Clock1 is a TCP server that periodically writes the time.
package main

import (
    "io"
    "log"
    "net"
    "time"
)

func main() {
    listener, err := net.Listen("tcp", "localhost:8000")
    if err != nil {
        log.Fatal(err)
    }

    for {
        conn, err := listener.Accept()
        if err != nil {
            log.Print(err) // e.g., connection aborted
            continue
        }
        handleConn(conn) // handle one connection at a time
    }
}

func handleConn(c net.Conn) {
    defer c.Close()
    for {
        _, err := io.WriteString(c, time.Now().Format("15:04:05\n"))
        if err != nil {
            return // e.g., client disconnected
        }
        time.Sleep(1 * time.Second)
    }
}
```

`Listen` 函数创建了一个 `net.Listener` 的对象，这个对象会监听一个网络端口上到来的连接，在这个例子里我们用的是 `TCP` 的 `localhost: 8000` 端口。`listener` 对象的 `Accept` 方法会直接阻塞，直到一个新的连接被创建，然后会返回一个 `net. Conn` 对象来表示这个连接。`handleConn` 函数会处理一个完整的客户端连接。

`time.Time.Format` 方法提供了一种格式化日期和时间信息的方式。它的参数是一个格式化模板，标识如何来格式化时间，而这个格式化模板限定为 `Mon Jan 2 03:04:05 PM 2006 UTC-0700`。有 8 个部分（周几、月份、一个月的第几天……）。可以以任意的形式来组合前面这个模板；出现在模板中的部分会作为参考来对时间格式进行输出。在上面的例子中我们只用到了小时、分钟和秒。`time` 包里定义了很多标准时间格式，比如 `time.RFC1123`。在进行格式化的逆向操作 `time. Parse` 时，也会用到同样的策略。（译注：这是 go 语言和其它语言相比比较奇葩的一个地方。你需要记住格式化字符串是 `1 月 2 日下午 3 点 4 分 5 秒零六年 UTC-0700`，而不像其它语言那样 ` Y-m-d H:i: s ` 一样，当然了这里可以用 1234567 的方式来记忆，倒是也不麻烦。）

为了连接例子里的服务器，我们需要一个客户端程序，比如 `netcat` 这个工具（`nc` 命令），这个工具可以用来执行网络连接操作。

```
$ go build gopl.io/ch8/clock1
$ ./clock1 &
$ nc localhost 8000
13:58:54
13:58:55
13:58:56
13:58:57
^C
```

客户端将服务器发来的时间显示了出来。如果你的系统没有装 `nc` 这个工具，你可以用 `telnet` 来实现同样的效果，或者也可以用我们下面的这个用 `go` 写的简单的 `telnet` 程序，用 `net.Dial` 就可以简单地创建一个 `TCP` 连接：

```go
// Netcat1 is a read-only TCP client.
package main

import (
    "io"
    "log"
    "net"
    "os"
)

func main() {
    conn, err := net.Dial("tcp", "localhost:8000")
    if err != nil {
        log.Fatal(err)
    }
    defer conn.Close()
    mustCopy(os.Stdout, conn)
}

func mustCopy(dst io.Writer, src io.Reader) {
    if _, err := io.Copy(dst, src); err != nil {
        log.Fatal(err)
    }
}
```

这个程序会从连接中读取数据，并将读到的内容写到标准输出中。这里可以开两个终端窗口，下面左边的是其中的一个的输出，右边的是另一个的输出：(最好使用 `nc` 模拟，会比较方便)

```shell
$ go build gopl.io/ch8/netcat1
$ ./netcat1
13:58:54                               $ ./netcat1
13:58:55
13:58:56
^C
                                       13:58:57
                                       13:58:58
                                       13:58:59
                                       ^C
$ killall clock1
```


第二个客户端必须等待第一个客户端完成工作，这样服务端才能继续向后执行；因为我们这里的服务器程序同一时间只能处理一个客户端连接。我们这里对服务端程序做一点小改动，使其支持并发：在 `handleConn` 函数调用的地方增加 `go` 关键字，让每一次 `handleConn` 的调用都进入一个独立的 `goroutine`。


```go
for {
    conn, err := listener.Accept()
    if err != nil {
        log.Print(err) // e.g., connection aborted
        continue
    }
    go handleConn(conn) // handle connections concurrently
}
```

现在多个客户端可以同时接收到时间了：

```shell
$ go build gopl.io/ch8/clock2
$ ./clock2 &
$ go build gopl.io/ch8/netcat1
$ ./netcat1
14:02:54                               $ ./netcat1
14:02:55                               14:02:55
14:02:56                               14:02:56
14:02:57                               ^C
14:02:58
14:02:59                               $ ./netcat1
14:03:00                               14:03:00
14:03:01                               14:03:01
^C                                     14:03:02
                                       ^C
```
