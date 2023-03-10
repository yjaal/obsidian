
```go
func crawl(url string) []string {
    fmt.Println(url)
    list, err := links.Extract(url)
    if err != nil {
        log.Print(err)
    }
    return list
}
```
这里是从一个页面中提取出所有的链接

主函数其实就是一个广度优先搜索。一个 `worklist` 是一个记录了需要处理的元素的队列，每一个元素都是一个需要抓取的 `URL` 列表，不过这一次我们用 `channel` 代替 `slice` 来做这个队列。每一个对 `crawl` 的调用都会在他们自己的 `goroutine` 中进行并且会把他们抓到的链接发送回 `worklist`。

```go
func main() {
    worklist := make(chan []string)

    // Start with the command-line arguments.
    go func() { worklist <- os.Args[1:] }()

    // Crawl the web concurrently.
    seen := make(map[string]bool)
    for list := range worklist {
        for _, link := range list {
            if !seen[link] {
                seen[link] = true
                go func(link string) {
                    worklist <- crawl(link)
                }(link)
            }
        }
    }
}
```

这里有几个问题，第一个是并发可能会很高，我们需要做一些限制；另外一个就是程序不会终止。

对第一个问题，我们可以通过 `channel` 的缓存来控制

```go
// tokens is a counting semaphore used to
// enforce a limit of 20 concurrent requests.
// 最多只能有20个调用
var tokens = make(chan struct{}, 20)

func crawl(url string) []string {
    fmt.Println(url)
    tokens <- struct{}{} // acquire a token
    list, err := links.Extract(url)
    <-tokens // release the token
    if err != nil {
        log.Print(err)
    }
    return list
}
```

第二个问题可以通过在 `worklist` 为空或者没有 `crawl` 的 `goroutine` 在运行时退出主循环。

```go
func main() {
    worklist := make(chan []string)
    var n int // number of pending sends to worklist

    // Start with the command-line arguments.
    n++
    go func() { worklist <- os.Args[1:] }()

    // Crawl the web concurrently.
    seen := make(map[string]bool)

    for ; n > 0; n-- {
        list := <-worklist
        for _, link := range list {
            if !seen[link] {
                seen[link] = true
                n++
                go func(link string) {
                    worklist <- crawl(link)
                }(link)
            }
        }
    }
}
```

这里使用计数信号量来进行控制，当然还可以通过控制创建固定数量的 `goroutines`

```go
func main() {
    worklist := make(chan []string)  // lists of URLs, may have duplicates
    unseenLinks := make(chan string) // de-duplicated URLs

    // Add command-line arguments to worklist.
    go func() { worklist <- os.Args[1:] }()

    // Create 20 crawler goroutines to fetch each unseen link.
    // 创建20个goroutines来处理unseenLinks
    for i := 0; i < 20; i++ {
        go func() {
            for link := range unseenLinks {
                foundLinks := crawl(link)
                go func() { worklist <- foundLinks }()
            }
        }()
    }

    // The main goroutine de-duplicates worklist items
    // and sends the unseen ones to the crawlers.
    seen := make(map[string]bool)
    for list := range worklist {
        for _, link := range list {
            if !seen[link] {
                seen[link] = true
                unseenLinks <- link
            }
        }
    }
}
```