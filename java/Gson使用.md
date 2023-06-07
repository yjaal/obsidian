```toc
```


摘自： `https://www.cnblogs.com/qinxu/p/9504412.html`

## 一、Gson 的基本用法
Gson 提供了 fromJson () 和 toJson () 两个直接用于解析和生成的方法，前者实现反序列化，后者实现了序列化；同时每个方法都提供了重载方法

### 1.1 基本数据类型的解析
```java
Gson gson = new Gson();
int i = gson.fromJson("100", int.class); //100
double d = gson.fromJson("\"99.99\"", double.class);  //99.99
boolean b = gson.fromJson("true", boolean.class);     // true
String str = gson.fromJson("String", String.class);   // String
```

### 1.2 基本数据类型的生成　
```java
Gson gson = new Gson();
String jsonNumber = gson.toJson(100);       // 100
String jsonBoolean = gson.toJson(false);    // false
String jsonString = gson.toJson("String"); //"String"
```


### 1.3 POJO 类的生成与解析

```java
public class User {
    //省略其它
    public String name;
    public int age;
    public String emailAddress;
}
```
生成 JSON：　
```java
Gson gson = new Gson();
User user = new User("张三",24);
String jsonObject = gson.toJson(user); // {"name":"张三kidou","age":24}
```
解析 JSON：　
```java
Gson gson = new Gson();
String jsonString = "{\"name\":\"张三\",\"age\":24}";
User user = gson.fromJson(jsonString, User.class);
```

## 二、属性重命名 @SerializedName 
也就是传入的报文不符合我们本地字段命名，我们可以进行标记

```
期望的json格式：{"name":"张三","age":24,"emailAddress":"zhangsan@ceshi.com"}
实际：{"name":"张三","age":24,"email_address":"zhangsan@ceshi.com"}
```

```java
@SerializedName("email_address")
public String emailAddress;
```

当然还可以更灵活

```java
@SerializedName(value = "emailAddress", alternate = {"email", "email_address"})
public String emailAddress;
//当三个属性(email_address、email、emailAddress)都中出现任意一个时均可以得到正确的结果
```

## 三、Gson 中使用泛型

例如：JSON 字符串数组：`["Android","Java","PHP"]`
当要通过 Gson 解析这个 json 时，一般有两种方式：使用数组或者 List；而 List 对于增删都是比较方便的，所以实际使用是还是 List 比较多

数组比较简单：

```java
Gson gson = new Gson();
String jsonArray = "[\"Android\",\"Java\",\"PHP\"]";
String[] strings = gson.fromJson(jsonArray, String[].class);
```

对于 List 将上面的代码中的 `String[].class` 直接改为 `List<String>.class` 是不行的，对于 Java 来说 `List<String>` 和 `List<User>` 这俩个的字节码文件只一个那就是 `List.class`，这是 Java 泛型使用时要注意的问题-泛型擦除。为了解决的上面的问题，Gson 提供了 TypeToken 来实现对泛型的支持，所以将以上的数据解析为 `List<String>` 时需要这样写
  
```java
Gson gson = new Gson();
String jsonArray = "[\"Android\",\"Java\",\"PHP\"]";
String[] strings = gson.fromJson(jsonArray, String[].class);
// TypeToken的构造方法是protected修饰的,所以上面才会写成new TypeToken<List<String>>() {}.getType() 而不是 new TypeToken<List<String>>().getType()
// 这里其实就是进行一次转换
List<String> stringList = gson.fromJson(jsonArray, new TypeToken<List<String>>(){}.getType());
```
泛型解析对接口 POJO 的设计影响

泛型的引入可以减少无关的代码：　　
```java
{"code":"0","message":"success","data":{}}
{"code":"0","message":"success","data":[]}
```

我们真正需要的 data 所包含的数据，而 code 只使用一次，message 则几乎不用，如果 Gson 不支持泛型或不知道 Gson 支持泛型的同学一定会这么定义 POJO
```java
public class UserResponse {
    public int code;
    public String message;
    public User data;
}
```
当其它接口的时候又重新定义一个 XXResponse 将 data 的类型改成 XX，很明显 code，和 message 被重复定义了多次，通过泛型可以将 code 和 message 字段抽取到一个 Result 的类中，这样只需要编写 data 字段所对应的 POJO 即可：

```java
public class Result<T> {
    public int code;
    public String message;
    public T data;
}  
//对于data字段是User时则可以写为 Result<User> ,当是个列表的时候为 Result<List<User>>
```

## 四、使用 GsonBuilder 导出 null 值、格式化输出、日期时间

其实就是设置一些自定义属性，因为默认 Gson 在转换时，是不输出 null 的，需要修改其默认设置才行

```java
Gson gson = new GsonBuilder()
    //序列化null
    .serializeNulls()
    // 设置日期时间格式，另有2个重载方法
    // 在序列化和反序化时均生效
    .setDateFormat("yyyy-MM-dd")
    // 禁此序列化内部类
     .disableInnerClassSerialization()
    //生成不可执行的Json（多了 )]}' 这4个字符）
    .generateNonExecutableJson()
     //禁止转义html标签
    .disableHtmlEscaping()
    //格式化输出
    .setPrettyPrinting()
    .create();
//：内部类(Inner Class)和嵌套类(Nested Class)的区别
```



