```go
package eval

import (
	"fmt"
	"math"
)

type Expr interface {
	// 根据给定的变量返回表达式的值，也就是计算
	Eval(env Env) float64
	// 表达式合法性检查
	Check(vars map[Var]bool) error
}

// 表示对一个变量对引用
type Var string

// 表示一个浮点型常量
type literal float64

type unary struct {
	op rune // + -
	x  Expr
}

type binary struct {
	op   rune // + - * /
	x, y Expr
}

type call struct {
	fn   string // pow sin sqrt
	args []Expr
}

// 表示一个变量需要映射成对值
type Env map[Var]float64

func (v Var) Eval(env Env) float64 {
	// a.Eval 表示从env中寻找 key=a 的值
	return env[v]
}

func (l literal) Eval(_ Env) float64 {
	// 返回其实际的值
	return float64(l)
}

// 这里可以理解为取正负值
func (u unary) Eval(env Env) float64 {
	switch u.op {
	case '+':
		return +u.x.Eval(env)
	case '-':
		return -u.x.Eval(env)
	}
	panic(fmt.Sprintf("unsupported unary operator: %q", u.op))
}

func (b binary) Eval(env Env) float64 {
	switch b.op {
	case '+':
		return b.x.Eval(env) + b.y.Eval(env)
	case '-':
		return b.x.Eval(env) - b.y.Eval(env)
	case '*':
		return b.x.Eval(env) * b.y.Eval(env)
	case '/':
		return b.x.Eval(env) / b.y.Eval(env)
	}
	panic(fmt.Sprintf("unsupported unary operator: %q", b.op))
}

func (c call) Eval(env Env) float64 {
	switch c.fn {
	case "pow":
		return math.Pow(c.args[0].Eval(env), c.args[1].Eval(env))
	case "sin":
		return math.Sin(c.args[0].Eval(env))
	case "sqrt":
		return math.Sqrt(c.args[0].Eval(env))
	}
	panic(fmt.Sprintf("unsupported function call: %s", c.fn))
}
```
