## Consume  接口

此接口是消费接口，有入参无出参，可以理解将传入数据消费掉了

### 1.1 Consume

定义
```java
//对给定的参数执行操作
Consume<T> void accept (T t);

//返回一个组合函数，after 将会在该函数执行之后应用
default Consumer andThen (Consumer<? super T> after);
```

案例
```java
StringBuilder sb = new StringBuilder ("Hello ");
Consumer<StringBuilder> consumer = (str) -> str. append ("Jack!");
Consumer<StringBuilder> consumer1 = (str) -> str. append (" Bob!");
consumer. andThen (consumer1). accept (sb);
System. out. println (sb. toString ());	// Hello Jack! Bob!
```


### 1.2 BiConsume

定义
```java
//对给定的参数执行操作
BiConsume<T, U> void accept (T t, U u);

//返回一个组合函数，after 将会在该函数执行之后应用
default BiConsumer<T, U> andThen (BiConsumer<? super T,? super U> after);
```

案例

```java
StringBuilder sb = new StringBuilder ();
BiConsumer<String, String> biConsumer = (a, b) -> {
	sb. append (a);
	sb. append (b);
};
BiConsumer<String, String> biConsumer1 = (a, b) -> {
	System. out. println (a + b);
};
// Hello Jack!
biConsumer. andThen (biConsumer1). accept ("Hello", " Jack!"); 
```


## Function  接口

此接口是函数式接口，有入参也有出参，但是类型不同，可以理解为进行一种转换

### 2.1 Function

定义
```java
// 将此参数应用到函数中
Function<T, R> R apply (T t);

// 返回一个组合函数，该函数结果应用到 after 函数中
Function<T, R> andThen (Function<? super R,? extends V> after);

// 返回一个组合函数，首先将入参应用到 before 函数，再将 before 函数结果
// 应用到该函数中
Function<T, R> compose (Function<? super V,? extends T> before);
```


案例
```java
//1
Function<String, String> function = a -> a + " Jack!";
System. out. println (function. apply ("Hello")); // Hello Jack!
//2
Function<String, String> function = a -> a + " Jack!";
Function<String, String> function1 = a -> a + " Bob!";
String greet = function. andThen (function1). apply ("Hello");
System. out. println (greet); // Hello Jack! Bob!
//3
Function<String, String> function = a -> a + " Jack!";
Function<String, String> function1 = a -> a + " Bob!";
String greet = function. compose (function1). apply ("Hello");
System. out. println (greet); // Hello Bob! Jack!
```


### 2.2 BiFunction

定义
```java
//将参数应用于函数执行
BiFunction<T, U, R> R apply (T t, U u);

//返回一个组合函数，after 函数应用于该函数之后
BiFunction<T, U, V> andThen (Function<? super R,? extends V> after);
```

案例
```java
//1
BiFunction<String, String, String> biFunction = (a, b) -> a + b;
// Hello Jack!
System. out. println (biFunction. apply ("Hello ", "Jack!")); 
//2
BiFunction<String, String, String> biFunction = (a, b) -> a + b;
Function<String, String> function = (a) -> a + "!!!";
// Hello Jack!!!
System. out. println (biFunction. andThen (function). apply ("Hello", " Jack")); 
```


## Operator 接口

`operator` 是数据处理操作，如过滤等，有入参也有处参，只是类型相同，可以理解为对数据做一些组合过滤，不改变其类型

### 3.1 UnaryOperator

定义
```java
// 将给定参数应用到函数中
UnaryOperator<T> T apply (T t);

// 返回一个组合函数，该函数结果应用到 after 函数中
Function<T, R> andThen (Function<? super R,? extends V> after);

// 返回一个组合函数，首先将入参应用到 before 函数，再将 before 函数
// 结果应用到该函数中
Function<T, R> compose (Function<? super V,? extends T> before);
```

案例
```java
//1
UnaryOperator<String> unaryOperator = greet -> greet + " Bob!";
// Hello Bob!
System. out. println (unaryOperator. apply ("Hello")); 
//2
UnaryOperator<String> unaryOperator = greet -> greet + " Bob!";
UnaryOperator<String> unaryOperator1 = greet -> greet + " Jack!";
String greet = unaryOperator. andThen (unaryOperator1). apply ("Hello");
System. out. println (greet); // Hello Bob! Jack!
//3
UnaryOperator<String> unaryOperator = greet -> greet + " Bob!";
UnaryOperator<String> unaryOperator1 = greet -> greet + " Jack!";
String greet = unaryOperator. compose (unaryOperator1). apply ("Hello");
System. out. println (greet); // Hello Jack! Bob!
```


### 3.2 BinaryOperator

定义
```java
//根据给定参数执行函数
BinaryOperator<T> T apply (T t, T u);

//返回一个组合函数，after 应用于该函数之后
BiFunction<T, T, T> andThen (Function<? super T,? extends T> after);

//返回二元操作本身，通过特殊比较器返回最大的元素
BinaryOperator maxBy (Comparator<? super T> comparator);

//返回二元操作本身，通过特殊比较器返回最小的元素
BinaryOperator minBy (Comparator<? super T> comparator);
```



案例
```java
//1
BinaryOperator<String> binaryOperator = (flag, flag1) -> flag + flag1;
System. out. println (binaryOperator. apply ("Hello ", "Jack!")); // Hello Jack!
//2
BinaryOperator<String> binaryOperator = (flag, flag1) -> flag + flag1;
Function<String, String> function = a -> a + "!!!";
System. out. println (binaryOperator. andThen (function). apply ("Hello", " Jack")); // Hello Jack!!!
//3
BinaryOperator<Integer> integerBinaryOperator = BinaryOperator. maxBy (Integer::compareTo);
Integer max = integerBinaryOperator. apply (12, 10);
System. out. println (max); // 12
//4
BinaryOperator<Integer> integerBinaryOperator1 = BinaryOperator. minBy (Integer::compare);
Integer min = integerBinaryOperator1. apply (12, 10);
System. out. println (min); // 10

```


## Predicate 接口

此接口其实就是对给定的数据进行一个判断，返回判断结果

### 4.1 Predicate
定义

```java
Predicate<T> boolean test (T t);//根据给定的参数进行判断

//返回一个组合判断，将 other 以短路与的方式加入到函数的判断中
Predicate and (Predicate<? super T> other);

//返回一个组合判断，将 other 以短路或的方式加入到函数的判断中
Predicate or (Predicate<? super T> other);

Predicate negate ();//将函数的判断取反
```


### 4.2 BiPredicate

定义
```java
BiPredicate<T, U> boolean test (T t, U u);
```


案例
```java
//1
BiPredicate<Integer, Integer> biPredicate = (a, b) -> a != b;
System. out. println (biPredicate. test (1, 2)); // true
//2
BiPredicate<Integer, Integer> biPredicate = (a, b) -> a != b;
biPredicate = biPredicate. and ((a, b) -> a.equals (b));
System. out. println (biPredicate. test (1, 2)); // false
//3
BiPredicate<Integer, Integer> biPredicate = (a, b) -> a != b;
biPredicate = biPredicate. or ((a, b) -> a == b);
System. out. println (biPredicate. test (1, 1)); // true
//4
BiPredicate<Integer, Integer> biPredicate = (a, b) -> a != b;
biPredicate = biPredicate. negate ();
System. out. println (biPredicate. test (1, 2)); // false
```


## Supplier 接口

此接口可以理解为提供数据的接口，没有入参但是有出参

定义
```java
Supplier<T> T get ();//获取结果值
```


案例
```java
Supplier<String> supplier = () -> "Hello Jack!";
System. out. println (supplier. get ()); // Hello Jack!
```
