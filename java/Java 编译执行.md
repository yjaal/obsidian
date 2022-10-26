# 基本的编译执行

这里记录下`java` 最基本的编译执行
基本结构 
`/Users/YJ/study/java-work/javaTest`
`        javaTest/src/Main.java`
`        javaTest/lib`

```java
// Main.java
package src;

public class Main{
	public static void main(String[] args){
		System.out.println("Hello world");
	}
}
```


编译 `javac ./src/Main.java`
执行 `java src.Main`

如果我们需要依赖某个`jar`包，那么需要将该`jar`包添加到`classpath`，使用 `-cp`设定。如果不执行，那么默认的`classpath`路径就是当前目录，所以上面我们在执行的时候就会在当前目录寻找要执行的`class`文件。

`classpath`就是一组目录的集合，它设置的搜索路径与操作系统相关。例如，在`Windows`系统上，用 `;` 分隔，带空格的目录用 `""` 括起来，可能长这样：`C:\work\project1\bin;C:\shared;"D:\My Documents\project1\bin"`。
在`linux`或`mac`上面使用` : `分隔。

# 添加 jar包编译执行

下面我们在`javaTest/lib`中添加`commons-lang3-3.12.0.jar`包，然后修改上面的类
```java
package src;
import org.apache.commons.lang3.StringUtils;
public class Main{
        public static void main(String[] args){
                if(!StringUtils.equals("aa", "bb")){
                        System.out.println("aa != bb");
                }
                System.out.println("Hello world");
        }
}
```

编译 `javac -cp .:./lib/commons-lang3-3.12.0.jar ./src/Main.java`
执行 `java -cp .:./lib/commons-lang3-3.12.0.jar src.Main`