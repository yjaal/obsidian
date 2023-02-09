
```toc

```

## java 基本加载流程

### 装载

- 通过类型的完全限定名，产生一个代表该类型的二进制数据流
- 解析这个二进制数据流为方法区内的内部数据结构
- 创建一个表示该类型的 `java.lang.Class` 类的实例。另外如果一个类装载器在预先装载的时遇到缺失或错误的 `class` 文件，它需要等到程序首次主动使用该类时才报告错误。

### 连接

- 验证，确认类型符合 Java 语言的语义，检查各个类之间的二进制兼容性 (比如 `final` 的类不用拥有子类等)，另外还需要进行符号引用的验证。
- 准备，Java 虚拟机为类变量分配内存，设置默认初始值。
- 解析 (可选的)，在类型的常量池中寻找类，接口，字段和方法的符号引用，把这些符号引用替换成直接引用的过程。
   
### 初始化

初始化就是执行类的构造器方法 `init()` 的过程。这个方法不需要定义，是 `javac` 编译器自动收集类中所有类变量的赋值动作和静态代码块中的语句合并来的。若该类具有父类，`jvm` 会保证父类的 `init` 先执行，然后在执行子类的 `init`。

## 初始化

当一个类被主动使用时，Java 虚拟机就会对其初始化，如下六种情况为主动使用：
- 当创建某个类的新实例时（如通过 `new` 或者反射，克隆，反序列化等）
- 当调用某个类的静态方法时
- 当使用某个类或接口的静态字段时
- 当调用 `Java API` 中的某些反射方法时，比如类 `Class` 中的方法，或者 `java.lang.reflect` 中的类的方法时
- 当初始化某个子类时
- 当虚拟机启动某个被标明为启动类的类（即包含 `main` 方法的那个类）Java 编译器会收集所有的类变量初始化语句和类型的静态初始化器，将这些放到一个特殊的方法中：`clinit`。 
    
    
案例：

```java
package io. netty. example;
import java. lang. reflect. Field;
import java. util. Vector;

class MyClass1 {

    static {
        System.out.println("MyClass1 static block ");
    }
}

class MyClass2 {

    static {
        System.out.println("MyClass2 static block ");
    }

    public static void print2() {
        System.out.println("MyClass2 print2");
    }
}

public class Hello {

    public static void main(String[] args) throws ClassNotFoundException {
        ClassLoader loader = Thread.currentThread().getContextClassLoader();
        printClassesOfClassLoader(loader);
        System.out
            .println("--------------------------------------------------------");
        System.out.println("hello word: " + MyClass1.class);
        // 不会初始化
        Class.forName("io.netty.example.MyClass1", false, Hello.class.getClassLoader());
        Class.forName("io.netty.example.MyClass1", false, Hello.class.getClassLoader());

        System.out
            .println("--------------------------------------------------------");
        // 初始化
        Class.forName("io.netty.example.MyClass1", true, Hello.class.getClassLoader());
        // 不会初始化
        Class.forName("io.netty.example.MyClass1", false, Hello.class.getClassLoader());
        // 已初始化，不会再次打印
        Class.forName("io.netty.example.MyClass1", true, Hello.class.getClassLoader());
        System.out
            .println("--------------------------------------------------------");
        MyClass2.print2();
    }


    public static void printClassesOfClassLoader(ClassLoader loader) {
        try {
            // 取出被class修饰的类
            Field classesF = ClassLoader.class.getDeclaredField("classes");
            classesF.setAccessible(true);
            Vector<Class<?>> classes = (Vector<Class<?>>) classesF.get(loader);
            for (Class c : classes) {
                System.out.println(c);
            }
        } catch (NoSuchFieldException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        }
    }
}
```

输出：

```
class com. intellij. rt. execution. application. AppMainV2$Agent
class com. intellij. rt. execution. application. AppMainV2
class com. intellij. rt. execution. application. AppMainV2$1
class io. netty. example. Hello
-----------------------------------------------------------------------
hello word: class io. netty. example. MyClass1
-----------------------------------------------------------------------
MyClass1 static block 
-----------------------------------------------------------------------
MyClass2 static block 
MyClass2 print2
```

