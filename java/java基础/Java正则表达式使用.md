```toc

```

## 正则表达式实例

一个字符串其实就是一个简单的正则表达式，例如 **`Hello World`** 正则表达式匹配 "`Hello World`" 字符串。`.（点号）` 也是一个正则表达式，它匹配任何一个字符如："` a `" 或 "` 1 `"。

下表列出了一些正则表达式的实例及描述：

|正则表达式|描述|
|-|-|
|this is text|匹配字符串 "`this is text`"|
| `this\s+is\s+text` |注意字符串中的 **`\s+`**。匹配单词 "`this`" 后面的 **`\s+`** 可以匹配多个空格，之后匹配 `is` 字符串，再之后 **`\s+`** 匹配多个空格然后再跟上 `text` 字符串。可以匹配这个实例：`this is text ` |
| `^\d+(\.\d+)?` | `^` 定义了以什么开始, `\d+` 匹配一个或多个数字, `?` 设置括号内的选项是可选的, `\.` 匹配 "`.`", 可以匹配的实例："5", "1.5" 和 "2.21"。|


`java.util.regex` 包主要包括以下三个类：

-   **Pattern 类：**
    `pattern` 对象是一个正则表达式的编译表示。`Pattern` 类没有公共构造方法。要创建一个 `Pattern` 对象，你必须首先调用其公共静态编译方法，它返回一个 `Pattern` 对象。该方法接受一个正则表达式作为它的第一个参数。
    
-   **Matcher 类：**
    `Matcher` 对象是对输入字符串进行解释和匹配操作的引擎。与 ` Pattern ` 类一样，`Matcher` 也没有公共构造方法。你需要调用 ` Pattern ` 对象的 `matcher` 方法来获得一个 ` Matcher ` 对象。
    
-   **PatternSyntaxException：**
    `PatternSyntaxException` 是一个非强制异常类，它表示一个正则表达式模式中的语法错误。


以下实例中使用了正则表达式 **`.*runoob.*`** 用于查找字符串中是否包了 **`runoob`** 子串：

```java
import java.util.regex.*; 

class RegexExample1{ 

	public static void main(String[] args){ 
		String content = "I am noob " + "from runoob.com."; 
		String pattern = ".*runoob.*"; 
		boolean isMatch = Pattern.matches(pattern, content); 
		System.out.println("字符串中是否包含了 'runoob' 子字符串? " + isMatch); } 
}

// 输出
// 字符串中是否包含了 'runoob' 子字符串? true
```


## 捕获组

捕获组是把多个字符当一个单独单元进行处理的方法，它通过对括号内的字符分组来创建。

例如，正则表达式 `(dog)` 创建了单一分组，组里包含 `"d"，"o"，和"g"`。

捕获组是通过从左至右计算其开括号来编号。例如，在表达式 `(A)(B(C)))`，有四个这样的组：

-   `((A)(B(C)))`
-   `(A)`
-   `(B(C))`
-   `(C)`

可以通过调用 `matcher` 对象的 `groupCount` 方法来查看表达式有多少个分组。`groupCount` 方法返回一个 `int` 值，表示 `matcher` 对象当前有多个捕获组。还有一个特殊的组（`group(0)`），它总是代表整个表达式。该组不包括在 `groupCount` 的返回值中。

下面的例子说明如何从一个给定的字符串中找到数字串：

```java
import java.util.regex.Matcher; 
import java.util.regex.Pattern; 

public class RegexMatches { 
	public static void main( String[] args ){ 
		// 按指定模式在字符串查找 
		String line = "This order was placed for QT3000! OK?"; 
		String pattern = "(\\D*)(\\d+)(.*)"; 
		// 创建 Pattern 对象 
		Pattern r = Pattern.compile(pattern); 
		// 现在创建 matcher 对象 
		Matcher m = r.matcher(line); 
		if (m.find( )) { 
			System.out.println("Found value: " + m.group(0) ); 
			System.out.println("Found value: " + m.group(1) ); 
			System.out.println("Found value: " + m.group(2) ); 
			System.out.println("Found value: " + m.group(3) ); 
		} else { 
			System.out.println("NO MATCH"); 
		} 
	} 
}

// 输出
// Found value: This order was placed for QT3000! OK?
// Found value: This order was placed for QT
// Found value: 3000
// Found value: ! OK?
```


## 语法

[[正则]]

一些重要的

|字符|描述|
|-|-|
| `\` |转义，如 `n` 可以匹配 `n`，而 `\n` 则匹配一个换行符|
| `^` |匹配开始位置|
| `$` |匹配结束位置，也就是结束为止必须是前面表达式匹配的样式|
| `*` |匹配前面的子表达式 0 次或多次，如 `zo*` 可以匹配 `z, zo, zoo`, 等价于 `{0,}` |
| `+` |匹配前面的子表达式 1 次或多次，如 `zo+` 可以匹配 `zo, zoo` |
| `?` |匹配前面子表达式 0 次或 1 次，等价于 `{0,1}` |
| `{n}` |n 是一个非负整数。匹配确定的 `n` 次，如 `o{2}` 只能匹配 `food` 中的两个 `o`，而不能匹配 `god` 中的一个 `o` |
| `{n,}` |表示至少匹配 `n` 次|
| `{n,m}` |n<=m, 表示至少匹配 `n` 次，最多匹配 `m` 次|
| `.` |匹配除 `\r`, `\n` 之外的任何单个字符|
| `x\|y` |没有包围在 () 里，其范围是整个正则表达式。例如，“`z\|food`”能匹配“`z`”或“`food`”。“`(?:z\|f)ood`”则匹配“`zood`”或“`food`”。|
| `[xyz]` |字符集合。匹配所包含的任意一个字符。例如，“`[abc]`”可以匹配“`plain`”中的“`a`”。特殊字符仅有反斜线保持特殊含义，用于转义字符。其它特殊字符如星号、加号、各种括号等均作为普通字符。脱字符^如果出现在首位则表示负值字符集合；如果出现在字符串中间就仅作为普通字符。连字符 – 如果出现在字符串中间表示字符范围描述；如果如果出现在首位（或末尾）则仅作为普通字符。右方括号应转义出现，也可以作为首位字符出现。|
| `[^xyz]` |排除型字符集合。匹配未列出的任意字符。例如，“`[^abc]`”可以匹配“`plain`”中的“`plain`”。|
| `[a-z]	` |字符范围。匹配指定范围内的任意字符。例如，“`[a-z]`”可以匹配“`a`”到“`z`”范围内的任意小写字母字符。|
| `[^a-z]` |排除型的字符范围。匹配任何不在指定范围内的任意字符。例如，“`[^a-z]`”可以匹配任何不在“`a`”到“`z`”范围内的任意字符。|
| `\b` |匹配一个单词边界，也就是指单词和空格间的位置。例如，“`erb`”可以匹配“`never`”中的“`er`”，但不能匹配“`verb`”中的“`er`”。|
| `\B` |匹配非单词边界。“`erB`”能匹配“`verb`”中的“`er`”，但不能匹配“`never`”中的“`er`”。|
| `\d` |匹配一个数字字符。等价于 `[0-9]`。注意 `Unicode` 正则表达式会匹配全角数字字符。|
| `\D` |匹配一个非数字字符。等价于 `[^0-9]`。|
| `\w` |匹配包括下划线的任何单词字符。等价于“`[A-Za-z0-9_]`”。注意 Unicode 正则表达式会匹配中文字符。|
| `\W` |匹配任何非单词字符。等价于“`[^A-Za-z0-9_]`”。|

## Matcher 类方法

