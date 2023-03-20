```toc

```
摘自： https://zhuanlan.zhihu.com/p/63219494

面向对象编程 (`OOP`) 中三个基本特征分别是封装，继承，多态。在 `Go` 语言中封装和继承是通过 ` struct` 来实现的，而多态则是通过接口 (`interface`) 来实现的。

### 什么是接口

在 Go 语言中接口包含两种含义:它既是方法的集合, 同时还是一种类型. 在Go 语言中是_隐式实现_的，意思就是对于一个具体的类型，不需要声明它实现了哪些接口，只需要提供接口所必需的方法。

在 Go 语言的类型系统中有一个核心概念: **我们不应该根据类型可以容纳哪种数据而是应该根据类型可以执行哪种操作来设计抽象类型**.

### 定义并实现接口

```go
//声明一个接口
type Human interface{
  Say()
}
//定义两个类，这两个类分别实现了 Human 接口的 Say 方法
type women struct {
}

type man struct {
}
func (w *women) Say() {
    fmt.Println("I'm a women")
}
func(m *man) Say() {
    fmt.Println("I'm a man")
}
func main() {
    w := new(women)
    w.Say()
    m := new(man)
    m.Say()
}
//output
//I'm a women
//I'm a man
```

如果一个具体类型实现了某个接口的所有方法, 我们则成为该具体类型实现了该接口.注意:**必须是所有方法**

### 接口类型

接口类型, 说白了就是空接口对于初学者来说很容易发生误解, 对于空接口来说, 任何具体类型都实现了空接口. 举个例子:

```go
func Say(s interface{}) {
    // ...
}
```

思考一下, 在 `Say` 函数内部, `s` 属于什么类型? 对于初学者来说很容易认为 `s` 属于任意类型, 其实 `s` 属于接口类型, 并不是任意类型, 但却可以转换成任意类型.

为什么呢? 因为当我们往 `Say` 方法传入值的时候, `Go runtime` 会自动的进行类型转换, 将该值转换成接口类型的值. 所有的值在运行时都只会有一个类型, `s` 的静态类型就是接口类型, 即 ` interface{} `

对于像 Go 这种静态类型的语言, 类型只是编译时候的概念. 那 Go 是如何实现接口值动态转换成任意类型值的呢?

在 Go 语言中, 接口值有两部分组成, 一个指向该接口的具体类型的指针和另外一个指向该具体类型真实数据的指针. (查看 `interface` 在 `runtime2.go` 定义可以获得)

```go
type iface struct {
    tab  *itab
    data unsafe.Pointer
}

type eface struct {
    _type *_type
    data  unsafe.Pointer
}
```

明白数据存储结构, 我们可以避免一些坑.例如下面的代码是有错误的:

```go
package main

import (
    "fmt"
)

func PrintAll(vals []interface{}) {
    for _, val := range vals {
        fmt.Println(val)
    }
}

func main() {
    names := []string{"stanley", "david", "oscar"}
    PrintAll(names)
}
```

编译会报错: `_cannot use names (type \[\]string) as type \[\]interface {} in argument to PrintAll`

因为 `PrintAll` 的入参是一个接口类型, 我们不能把 `string` 类型的值直接传入. 再传入之前需要进行转换, 或者 `PrintAll` 内部函数实现进行类型断言 (后面会讲到). 正确的代码:

```go
func main() {
    names := []string{"stanley", "david", "oscar"}
    vals := make([]interface{}, len(names))
    for i, v := range names {
        vals[i] = v
    }
    PrintAll(vals)
}
```

### 指针或值接收者的区别

我们都知道, 在 Go 语言中所有的数据都是值传递. 实现接口方法如果全部使用值接收者或者全部使用指针接收者, 都很好理解. 那如果实现的方法既存在值接收者, 又存在指针接收者呢? 这个地方有陷阱, 我们通过例子来说明:

```go
package main

import "fmt"

type Human interface {
    Say()
}

type Man struct {
}

type Woman struct {
}

func (m Man) Say() {
    fmt.Println("I'm a man")
}

func (w *Woman) Say() {
    fmt.Println("I'm a woman")
}

func main() {
    humans := []Human{Man{}, Woman{}}
    for _, human := range humans {
        human.Say()
    }
}
```

上面代码会报错: `_cannot use Woman literal (type Woman) as type Human in array or slice literal: Woman does not implement Human (Say method has pointer receiver) ` 提示 `Woman` 没有实现 `Human` 接口, 这是因为 `Woman` 实现 `Human` 接口定义的是指针接收者, 但我们在 ` main ` 方法中传入的是一个 `Woman` 的结构体转为 `Human` 的接口值, 并不是一个指针, 因此报错了. 如果我们将 ` main ` 函数略微改变一下:

```go
func main() {
    humans := []Human{&Man{}, &Woman{}}
    for _, human := range humans {
        human.Say()
    }
}
```

注意到在 `main` 方法中分别传入了 `Man` 和 `Woman` 的指针, 但是编译照样通过了. 为什么呢? `Man` 实现 `Human` 接口定义的是值接收者, 并不是指针接收者. 原因就是在 Go 语言中所有的都是值传递, 尽管传入的是 `Man` 的指针, 但是通过该指针我们可以找到其对应的值, *Go 语言隐式帮我们做了类型转换*。我们记住在 Go 语言中指针类型可以获得其关联的任意值类型, 但反过来却不行. 其实简单的想一下, 一个具体值可能有无数个指针指向它, 但一个指针只会指向一个具体的值.

### 类型断言

类型断言是作用在接口值上的操作, 类型断言的写法如下:

```go
<目标类型>, <布尔参数> := <表达式>.(目标类型) //这种是安全的类型断言, 不会引发 panic.
<目标类型> := <表达式>.(目标类型) //这种是非安全的类型断言, 如果断言失败会引发 panic.
```

我们看一个例子:

```go
package main

import "fmt"

type Shape interface {
    Area() float64
}

type Object interface {
    Volume() float64
}

type Skin interface {
    Color() float64
}

type Cube struct {
    side float64
}

func (c Cube)Area() float64 {
    return c.side * c.side
}

func (c Cube)Volume() float64 {
    return c.side * c.side * c.side
}

func main() {
    var s Shape = Cube{3.0}
    value1, ok1 := s.(Object)
    fmt.Printf("dynamic value of Shape 's' with value %v implements interface Object? %v\n", value1, ok1)
    value2, ok2 := s.(Skin)
    fmt.Printf("dynamic value of Shape 's' with value %v implements interface Skin? %v\n", value2, ok2)
}
```

因为在程序运行中, 有时会无法确定接口值的动态类型, 因此通过类型断言可以来检测其是否是一个特定的类型, 这样便可以针对性的进行业务处理.

结合类型断言, 我们就可以处理空接口的问题.比如说, 某个方法定义的入参类型为一个接口类型, 我们就可以在函数内部使用类型断言处理不同的业务.

Go 语言中 Println 的实现就是通过类型断言来处理的, 我们看一下源码的处理:

```go
func Println(a ...interface{}) (n int, err error) {
    return Fprintln(os.Stdout, a...)
}

func Fprintln(w io.Writer, a ...interface{}) (n int, err error) {
    p := newPrinter()
    p.doPrintln(a)
    n, err = w.Write(p.buf)
    p.free()
    return
}
func (p *pp) doPrintln(a []interface{}) {
    for argNum, arg := range a {
        if argNum > 0 {
            p.buf.WriteByte(' ')
        }
        p.printArg(arg, 'v')
    }
    p.buf.WriteByte('\n')
}

func (p *pp) printArg(arg interface{}, verb rune) {
    //此处省略部分代码
    //可以看到, 进行类型断言来判断需要输出的内容.
    switch f := arg.(type) {
    case bool:
        p.fmtBool(f, verb)
    case float32:
        p.fmtFloat(float64(f), 32, verb)
    case float64:
        p.fmtFloat(f, 64, verb)
    case complex64:
        p.fmtComplex(complex128(f), 64, verb)
    case complex128:
        p.fmtComplex(f, 128, verb)
    case int:
        p.fmtInteger(uint64(f), signed, verb)
    case int8:
        p.fmtInteger(uint64(f), signed, verb)
    case int16:
        p.fmtInteger(uint64(f), signed, verb)
    case int32:
        p.fmtInteger(uint64(f), signed, verb)
    case int64:
        p.fmtInteger(uint64(f), signed, verb)
    case uint:
        p.fmtInteger(uint64(f), unsigned, verb)
    case uint8:
        p.fmtInteger(uint64(f), unsigned, verb)
    case uint16:
        p.fmtInteger(uint64(f), unsigned, verb)
    case uint32:
        p.fmtInteger(uint64(f), unsigned, verb)
    case uint64:
        p.fmtInteger(f, unsigned, verb)
    //篇幅原因, 仅显示部分代码
}
```

### 总结

-   尽量考虑数据类型之间的相同功能来抽象接口, 而不是根据相同的字段
-   `interface{}` 是一个接口类型, 不是任意类型
-   接口的数据结构分两部分, 一部分指向其所表示的类型, 另一部分指向其具体类型的值
-   指针类型可以调用其指向的值的方法, 但是反过来处理不行
-   Go 语言中所有的都是值传递
-   使用安全的类型断言来判断接口所代表的动态类型, 通过类型匹配可以帮助我们写出更优雅通用并且安全的程序代码


