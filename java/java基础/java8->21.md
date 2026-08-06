
```toc
```

## 接口中定义私有方法

```java
public interface MyService {  
    void abstractMethod();  
    default void defaultMethod() {  
        System.out.println("this is default method");  
        // 调用私有方法
        privateMethod();  
    }  
    static void staticMethod() {  
        System.out.println("this is static method");  
    }  
    private void privateMethod() {  
        System.out.println("this is private method");  
    }  
}
```


## 废弃注解

```java
@Documented  
@Retention(RetentionPolicy.RUNTIME)  
@Target(value={CONSTRUCTOR, FIELD, LOCAL_VARIABLE, METHOD, PACKAGE, MODULE, PARAMETER, TYPE})  
public @interface Deprecated {  
     String since() default "";   
     boolean forRemoval() default false;  
}
```

这里有两个属性，since 标识某个方法从哪个版本就被废弃了，forRemoval 默认为 false，如果为 true 表示某个方法会在后续的某个版本可能会被移除掉。


## Jshell

这个其实和 python 那种命令行一样，就是可以直接执行 java 代码

```shell
$ jshell
|  欢迎使用 JShell -- 版本 21.0.2
|  要大致了解该版本, 请键入: /help intro

jshell> System.out.println("hello")
hello

jshell>
```


## 局部变量类型推断

```java
// 8
String name = "aa";
int age = 18

// 21
var name1 = "aa";
var age = 18;
```

## 直接运行

以前运行某个 java 文件是这样的

```java
javac Hello.java
java Hello
```

现在可以直接运行, 而且不产生字节码

```java
java Hello.java
```

## String 新增方法

空格去除

```java
// unicode空白字符
char c = '\u2000';
String str = c + "abc" + c;
System.out.println(str.trim());
System.out.println(str.strip());
```

以前使用的方法 trim 无法去除 unicode 空白字符，但是现在使用 strip 方法就可以了。

Java 自带空字符串判断

```java
String name = " ";
System.out.println(name.isBlank());
```


字符串重复

```java
String name = "aa";
System.out.println(name.repeat(3));
```


## Switch 语法

Java 8 写法

```java
int month = 3;
switch (month) {
	case 1:
	case 2:
	case 3:
		System.out.println("first");
		break;
	case 4:
	case 5:
		System.out.println("second");
		break;
	default:
		System.out.println("error");
}
```


```java
int month = 3;
switch (month) {
	case 1, 2, 3 -> System.out.println("first");
	case 4, 5 -> System.out.println("second");
	default -> System.out.println("error");
}
```

注意：冒号与 `->` 不能混用。而且不用写 break。

还可以有返回值

```java
int month = 3;
String str = switch (month) {
	case 1, 2, 3 -> "first";
	case 4, 5 -> "second";
	default -> "error";
};
```

**when 关键字**
```java
public static void yesOrNo(String obj) {
    switch (obj) {
        case null -> System.out.println("null");
        case String s
            when s.equalsIgnoreCase("yes") -> {
            System.out.println("确定");
        }
        case String s
            when s.equalsIgnoreCase("no") -> {
            System.out.println("取消");
        }
        case String s -> System.out.println("请输入yes或no");
    }
}
```

注意：最后一个 case 必须要有。


**简化强转**

```java
public void move(Animal animal) {
    switch (animal) {
        case Dog d -> d.run();
        case Bird d -> d.fly();
        default -> System.out.println("error");
    }
}

public class Animal {

}

public class Dog extends Animal {

    public void run() {
        System.out.println("run");
    }
}

public class Bird extends Animal {

    public void fly() {
        System.out.println("fly");
    }
}
```



## 文本块

```java
String s = """
	aaa
	bbb
	ccc
"""
```

## Instanceof

```java
// 旧写法
Object obj = 1;
if (obj instanceof Integer){
	Integer a = (Integer) obj;
}

// 新写法
Object obj = 1;
if (obj instanceof Integer a){
	
}
```


## NP 友好提示

```java
public class Demo {

    public A a;

    public static void main(String[] args) {
        System.out.println(new Demo().a.b);
    }

    public static class A {
        public B b;
    }

    public static class B{

    }
}
```

输出

```
Exception in thread "main" java.lang.NullPointerException: Cannot read field "b" because "a" is null
	at win.iot4yj.Demo.main(Demo.java:14)
```

可以看到现在异常错误信息更明确，以前则不会。


## Record 类型

```java
public class Demo {

    public static void main(String[] args) {
        var p = new Person("Tom", 13);
        System.out.println(p);
        System.out.println(p.name);
        System.out.println(p.name());
    }

    public record Person (String name, int age){
    }
}
```

输出
```
Person[name=Tom, age=13]
Tom
Tom
```

无需编写相关 get，set 方法。也可以添加一些方法。和 lombok 有点类似。


## 类约束


```java
// 声明一个密封类并指定允许的子类
public sealed class Shape 
    permits Circle, Square, Rectangle {  // 仅允许这3个子类
}
```

这里申明 Shape 为一个密封类，其只能有三个子类。

而其子类必须是以下之一：
- `final` 类（禁止进一步继承）
- `sealed` 类（继续限制子类）
- `non-sealed` 类（重新开放继承）

```java
// 示例1: final子类
public final class Circle extends Shape { /*...*/ }

// 示例2: 继续密封
public sealed class Square extends Shape 
    permits SmallSquare, LargeSquare { /*...*/ }

// 示例3: 重新开放继承
public non-sealed class Rectangle extends Shape { /*...*/ }
```

注意：密封类与子类必须在同一个模块或者包中。

用处感觉不是太大，但是有一个典型场景

```java
// 定义交易状态（仅允许3种具体状态）
public sealed interface TransactionState 
    permits Pending, Completed, Failed {
}

public final class Pending implements TransactionState {}
public final class Completed implements TransactionState {}
public final class Failed implements TransactionState {}

// 使用模式匹配处理所有可能状态
String processTransaction(TransactionState state) {
    return switch (state) {
        case Pending p -> "处理中...";
        case Completed c -> "成功";
        case Failed f -> "失败: " + f.getReason();
        // 不需要default分支（编译器检查穷举）
    };
}
```

以前我们必须要在 switch 中写 default，现在可以不用写了。


## 包装类和字符串不建议作为锁对象

```java
Integer a = 1;
synchronize(a){

}
```

**1. 数值类型包装类**

| 类型          | 缓存范围             | 风险场景示例                      |
| ----------- | ---------------- | --------------------------- |
| `Integer`   | -128 ~ 127       | `Integer.valueOf(1)` 返回同一对象 |
| `Long`      | -128 ~ 127       | `Long.valueOf(1L)` 缓存相同引用   |
| `Short`     | -128 ~ 127       | `Short.valueOf((short)1)`   |
| `Byte`      | 全部（-128 ~ 127）   | 所有 `Byte` 对象都可能被缓存          |
| `Character` | 0 ~ 127（ASCII范围） | `Character.valueOf('a')`    |

```java
Long lock1 = 100L;  // 缓存对象
Long lock2 = 100L;
System.out.println(lock1 == lock2); // true → 锁同一对象

Long lock3 = 200L;  // 新对象
Long lock4 = 200L;
System.out.println(lock3 == lock4); // false → 锁不同对象
```

**2. Boolean**

```java
Boolean lockA = true;  // 只有两个全局实例：Boolean.TRUE/FALSE
Boolean lockB = true;
System.out.println(lockA == lockB); // true → 锁同一对象
```

**3. String（尤其危险！）**

```java
String lock1 = "lock";  // 字符串常量池
String lock2 = "lock";
System.out.println(lock1 == lock2); // true → 锁同一对象

String lock3 = new String("lock"); // 新对象
String lock4 = new String("lock");
System.out.println(lock3 == lock4); // false → 锁不同对象
```

**风险**：
- 字面量字符串会从常量池复用，不同代码可能锁到同一对象。
- `intern()` 方法会强制放入常量池，加剧问题。


**根本原因**
1. **对象缓存机制**：  
    包装类和 `String` 的缓存/常量池导致相同值的对象可能指向同一内存地址。
2. **不可变性（Immutable）**：  
    这些类的实例无法修改，锁期间若重新赋值会指向新对象，导致锁失效。
3. **缺乏明确的锁标识**：  
    业务逻辑上无法保证锁对象的唯一性。


## Web server

```bash
jwebserver
```

启动后就可以通过页面访问了。

## 虚拟线程

主要用于提升服务器端端吞吐量。操作系统的线程和 jvm 中的线程是对应的，而且数量是有限制的。那现在虚拟线程其实就是在一个平台线程（老的线程概念）中拆分出来的多个，而这多个虚拟线程会由一个调度器进行调度，当可以运行时就调度到平台线程上执行，当进行 IO 等阻塞操作时，就从平台线程上面卸载。

### 基本使用

基本使用比较简单

```java
Thread thread = Thread.ofVirtual().name("duke").unstarted(runnable);
Thread.startVirtualThread(Runnable) 


// 以前的线程池
var vs = Executors.newFixedThreadPool(200)

// 现在
var vs = Executors.newVirtualThreadPerTaskExecutor();
```

### 作用

区别于虚拟线程，传统的线程对象叫做平台线程（platform thread）。平台线程在底层 OS 线程上运行 Java 代码，并在代码的整个生命周期中占用该 OS 线程，因此平台线程的数量受限于 OS 线程的数量。虚拟线程是 java.lang.Thread 的一个实例，它在底层 OS 线程上运行 Java 代码，但不会在代码的整个生命周期中占用该 OS 线程。也就是说，多个虚拟线程可以在同一个 OS 线程上运行其 Java 代码，可以有效地共享该线程。平台线程独占宝贵的 OS 线程，而虚拟线程则不会，因此虚拟线程的数量可以比 OS 线程的数量多得多，执行阻塞任务的整体吞吐量也就大了很多。

但如果上述任务不是简单的sleep 1s，而是计算了1s（例如做矩阵计算或数组排序等），用线程池和虚拟线程的执行时间区别就没有那么大。原因是虚拟线程虽然可以带来更大的吞吐量，但并不能让单个任务计算得更快，当使用平台线程执行任务已经让cpu没有任何空闲时，切换虚拟线程来执行也不会带来任何收益。

**虚拟线程可以发挥的最大作用是，可以让采用单请求单线程（thread-per-request）的方式编写的服务器程序最大化地利用CPU计算资源** **。**

### 避免使用 ThreadLocal 池化资源

虚拟线程支持ThreadLocal和InheritableThreadLocal，因此它们可以运行使用ThreadLocal的现有代码。然而，由于虚拟线程可能非常众多，使用ThreadLocal时必须仔细考虑，特别是**在线程池中共享同一线程的多个任务之间，不要使用ThreadLocal来池化过大的资源。**使用虚拟线程的预期是，只在其生命周期内运行单个任务，因此不应该对虚拟线程进行池化。在过去的一些版本中，java的基础库已经删除了许多ThreadLocal的使用，以减少在大规模使用虚拟线程时的内存占用。

### 虚拟线程固定问题

JDK中绝大多数的阻塞操作（如LockSupport、网络库API、大部分IO操作）都会卸载虚拟线程，释放其载体线程和底层操作系统线程以执行其他虚拟线程。然而，JDK中有一些阻塞操作不会卸载虚拟线程，从而阻塞其载体线程和底层操作系统线程，这是因为操作系统层面（例如许多文件系统操作）或JDK层面（例如`Object.wait()`）的限制。为了解决这些阻塞操作的问题，虚拟线程调度器会通过暂时扩展并行度来弥补平台线程被占用，因此在调度程序的`ForkJoinPool`中，平台线程的数量可能暂时超过可用处理器的数量，可以通过系统属性`jdk.virtualThreadScheduler.maxPoolSize`来调整调度器可用的平台线程的最大数量。

但有两种情况下虚拟线程在阻塞操作期间不能卸载，而是被固定（pinned）在其载体线程上：

1.  当虚拟线程在 synchronized 代码块或方法中执行代码时
2.  当它执行本地方法（native method ）或外部函数（foreign function）时

参考：[虚拟线程 Virtual Threads](https://juejin.cn/post/7280746515526058038)


## 字符串模板

```java
String str = "字符串1";  
String str2 = STR."i like \{str}";
```

不过这个特性在 21 版本中是预览版。


## Record pattern

```java
public class Demo {
    public static void main(String[] args) {
    }

    static void test(Object obj) {
        if (obj instanceof Student(String name, int age)) {
            System.out.println(name);
        }
    }
}

record Student(String name, int age) {
}
```

可以看到这里不仅仅进行了类型判断，还进行了相关变量赋值了。



## 结构化并发

这个特性在 21 中也只是预览。

下面通过一个例子说明其解决的问题

```java
Response handle() throws ExecutionException, InterruptedException {
    Future<String>  user  = esvc.submit(() -> findUser());
    Future<Integer> order = esvc.submit(() -> fetchOrder());
    String theUser  = user.get();   // Join findUser
    int    theOrder = order.get();  // Join fetchOrder
    return new Response(theUser, theOrder);
}
```

由于子任务并发执行，每个子任务可以独立成功或失败（在此上下文中，失败意味着抛出异常）。例如这个例子里，`handle()`的任务应在任何子任务失败时失败。当发生故障时，独立执行子任务的线程可能会出现意外的复杂情况：

1. 如果`findUser()`抛出异常，则在调用`user.get()`时，`handle()`将抛出异常，但`fetchOrder()`将继续在其自己的线程中运行。这是线程泄漏，最好情况下会浪费资源，最坏情况下，`fetchOrder()`线程将干扰其他任务。
    
2. 如果执行`handle()`的线程被中断，中断不会传播到子任务。无论是`findUser()`还是`fetchOrder()`线程都会泄漏，在`handle()`失败后继续运行。
    
3. 如果`findUser()`执行时间很长，但`fetchOrder()`在此期间失败，那么`handle()`将不必要地等待`findUser()`，因为它会在`user.get()`上阻塞而不是取消它。只有在`findUser()`完成并且`user.get()`返回后，`order.get()`才会抛出异常，导致`handle()`失败。

这些问题出现的根源，在于我们的程序在逻辑上是按照 “任务 - 子任务” 关系进行结构化的，但这些关系仅存在于开发人员的思想中，实际并发执行的子任务是非结构化执行的。这不仅增加了错误的可能性，而且使得诊断和排查此类错误更加困难，比如 `thread dump`等工具将在不相关线程的调用堆栈中显示`handle()、findUser()` 和`fetchOrder()`，而没有 “任务 - 子任务” 关系的信息。

  


```java
public class Demo {
    public static void main(String[] args) {
    }

    Food handle() throws InterruptedException, ExecutionException {
        try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
            Supplier<String> scope1 = scope.fork(() -> "上烤串");
            Supplier<String> scope2 = scope.fork(() -> "上啤酒");
            // 失败传播
            scope.join().throwIfFailed();
            // 两个任务都完成才可以
            return new Food(scope1.get(), scope2.get());
        }
    }
}
record Food(String eat, String drink) {
}
```































