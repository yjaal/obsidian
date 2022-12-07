## 1 . toList
```java
public static <T> Collector<T, ?, List<T>> toList () {
    return new CollectorImpl<>(
	    (Supplier<List<T>>) ArrayList:: new, 
	    List::add,
        (left, right) -> { left.addAll(right); return left; }, 
        CH_ID);
}
```


## 2 . toSet
```java
public static <T> Collector<T, ?, Set<T>> toSet () {
    return new CollectorImpl<>(
	    (Supplier<Set<T>>) HashSet:: new, 
	    Set:: add,
		(left, right) -> { left. addAll (right); return left; }, 
		CH_UNORDERED_ID);
}
```


## 3 . joining
```java
public static Collector<CharSequence, ?, String> joining () {
    return new CollectorImpl<CharSequence, StringBuilder, String>(
            StringBuilder:: new, 
            StringBuilder:: append,
            (r1, r2) -> { r1. append (r2); return r1; }, 
            StringBuilder:: toString, 
            CH_NOID);
}

public static Collector<CharSequence, ?, String> joining (CharSequence delimiter) {
    return joining (delimiter, "", "");
}

public static Collector<CharSequence, ?, String> joining (
	CharSequence delimiter,CharSequence prefix, CharSequence suffix) {
    return new CollectorImpl<>(
            () -> new StringJoiner (delimiter, prefix, suffix),
            StringJoiner:: add, 
            StringJoiner:: merge, 
            StringJoiner:: toString, 
            CH_NOID);
}
```


## 4 . mapping
```java
public static <T, U, A, R>
    Collector<T, ?, R> mapping (Function<? super T, ? extends U> mapper,
                               Collector<? super U, A, R> downstream) {
        BiConsumer<A, ? super U> downstreamAccumulator = downstream. accumulator ();
        return new CollectorImpl<>(downstream. supplier (),
                                   (r, t) -> downstreamAccumulator. accept (r, mapper. apply (t)),
                                   downstream. combiner (), downstream. finisher (),
                                   downstream. characteristics ());
    }
```
此方法看似复杂，实际使用较为简单，参数 1 就是传一个函数，即将一种类型数据转换成另外一种类型，参数 2 就是传递一个收集器，可以理解为先将数据进行转换然后进行收集，如
```java
// [张三, 李四, 王五]
students. stream ()
        . collect (Collectors. mapping (Student:: getName, Collectors. toList ()));
// 这种方式等同于
students. stream (). map (Student::getName). collect (Collectors. toList ());
```

注意：如果是两个 stream 需要打平，那么需要使用 flatMap
```java
List<String> words = new ArrayList<String>();
words. add ("your");
words. add ("name");
public static Stream<Character> characterStream (String s){  
    List<Character> result = new ArrayList<>();  
    for (char c : s.toCharArray ()) 
        result. add (c);
    return result. stream ();  
}
// [['y', 'o', 'u', 'r'], ['n', 'a', 'm', 'e']]
List<List<String>> result = words. map (w -> characterStream (w)). collect (Collectors. toList ());;  
// ['y', 'o', 'u', 'r', 'n', 'a', 'm', 'e']
List<String> letters = words. flatMap (w -> characterStream (w)). collect (Collectors. toList ());;  
```

    
## 5 . counting
```java
public static <T> Collector<T, ?, Long> counting () {
    return reducing (0L, e -> 1L, Long::sum);
}
```
此方法就是使用下面 reducing 方法实现，可以参考后面的说明。

    
## 6 . minBy maxBy
```java
public static <T> Collector<T, ?, Optional<T>> minBy (Comparator<? super T> comparator) {
    return reducing (BinaryOperator. minBy (comparator));
}

public static <T> Collector<T, ?, Optional<T>> maxBy (Comparator<? super T> comparator) {
    return reducing (BinaryOperator. maxBy (comparator));
}
```
这里需要传入一个比较器，我们可以自己实现，也可以使用现成的
`Comparator. comparing (Student::getAge))`。自己实现 `Comparator` 需要实现 `compare` 和 `equals` 方法

    
## 7 . summingInt
```java
public static <T> Collector<T, ?, Integer> summingInt (ToIntFunction<? super T> mapper) {
    return new CollectorImpl<>(
            () -> new int[1],
            (a, t) -> { a[0] += mapper. applyAsInt (t); },
            (a, b) -> { a[0] += b[0]; return a; },
            a -> a[0], CH_NOID);
}
需要进行一个转换方法然后进行计算
// 66
students. stream (). collect (Collectors. summingInt (s -> (int)s.getScore ()))
```


## 8 . averagingInt
```java
public static <T> Collector<T, ?, Double> averagingInt (ToIntFunction<? super T> mapper) {
    return new CollectorImpl<>(
            () -> new long[2],
            (a, t) -> { a[0] += mapper. applyAsInt (t); a[1]++; },
            (a, b) -> { a[0] += b[0]; a[1] += b[1]; return a; },
            a -> (a[1] == 0) ? 0.0d : (double) a[0] / a[1], CH_NOID);
}
```
此方法与上面的方法类似 

## 9 . reducing
```java
public static <T> Collector<T, ?, T> reducing (T identity, BinaryOperator<T> op) {
    return new CollectorImpl<>(
            boxSupplier (identity),
            (a, t) -> { a[0] = op. apply (a[0], t); },
            (a, b) -> { a[0] = op. apply (a[0], b[0]); return a; },
            a -> a[0],
            CH_NOID);
}
```
可以使用例子进行理解
```java
// 66.369
students. stream (). map (Student::getScore). collect (Collectors. reducing (0.0, Double::sum));
```
这里可以看到参数 `identify` 传入的是 `0.0`，而 `op` 传入的是累加器。然后看实现中的参数，首先要创建一个 `Objects[] ` 的数组，这个数组中只有一个元素，那就是 `0.0`。然后看第二个参数，`a` 表示创建的数组，而 `t` 表示传入的数据，每一个数据都要和 `a[0]` 相加，然后再赋值给 `a[0]`; 第三个参数是最后的操作，`a`  和 `b` 表示两个小的结果集合，要将这两个结果集使用累加器，然后赋值给 `a[0]`。然后返回 `a` 数组。最后取出数组中的第一个元素就是最终结果。

```java
public static <T> Collector<T, ?, Optional<T>> reducing (BinaryOperator<T> op) {
    class OptionalBox implements Consumer<T> {
        T value = null;
        boolean present = false;

        @Override
        public void accept (T t) {
            if (present) {
                value = op. apply (value, t);
            }
            else {
                value = t;
                present = true;
            }
        }
    }

    return new CollectorImpl<T, OptionalBox, Optional<T>>(
            OptionalBox:: new, OptionalBox:: accept,
            (a, b) -> { if (b.present) a.accept (b.value); return a; },
            a -> Optional. ofNullable (a.value), CH_NOID);
}
// 66.369
students. stream (). map (Student::getScore). collect (Collectors. reducing (Double::sum));
```
此实现可以这样理解，如果第一次使用第一个元素是空，那就取传入数据的第一个元素作为首元素，后面的就基本相同了。要注意，返回的是一个 `Optional`，如果没有传入数据，那就取不到值。
    
```java
public static <T, U> Collector<T, ?, U> reducing (U identity,
                                Function<? super T, ? extends U> mapper,
                                BinaryOperator<U> op) {
    return new CollectorImpl<>(
            boxSupplier (identity),
            (a, t) -> { a[0] = op. apply (a[0], mapper. apply (t)); },
            (a, b) -> { a[0] = op. apply (a[0], b[0]); return a; },
            a -> a[0], CH_NOID);
}

// 66.369
students. stream ()
        . collect (Collectors. reducing (0.0, Student:: getScore, Double::sum));
```
这个很好理解，就是多传入了一个处理操作。


## 10 . groupingBy
```java
public static <T, K> Collector<T, ?, Map<K, List<T>>>
    groupingBy (Function<? super T, ? extends K> classifier) {
    return groupingBy (classifier, toList ());
}
    
public static <T, K, A, D> Collector<T, ?, Map<K, D>> 
    groupingBy (Function<? super T, ? extends K> classifier,
            Collector<? super T, A, D> downstream) {
    return groupingBy (classifier, HashMap:: new, downstream);
}

public static <T, K, D, A, M extends Map<K, D>>
    Collector<T, ?, M> groupingBy (Function<? super T, ? extends K> classifier,
                                  Supplier<M> mapFactory,
                                  Collector<? super T, A, D> downstream) {
        Supplier<A> downstreamSupplier = downstream. supplier ();
        BiConsumer<A, ? super T> downstreamAccumulator = downstream. accumulator ();
        BiConsumer<Map<K, A>, T> accumulator = (m, t) -> {
            K key = Objects. requireNonNull (classifier. apply (t), "element cannot be mapped to a null key");
            A container = m.computeIfAbsent (key, k -> downstreamSupplier. get ());
            downstreamAccumulator. accept (container, t);
        };
        BinaryOperator<Map<K, A>> merger = Collectors.<K, A, Map<K, A>>mapMerger (downstream. combiner ());
        @SuppressWarnings ("unchecked")
        Supplier<Map<K, A>> mangledFactory = (Supplier<Map<K, A>>) mapFactory;

        if (downstream. characteristics (). contains (Collector. Characteristics. IDENTITY_FINISH)) {
            return new CollectorImpl<>(mangledFactory, accumulator, merger, CH_ID);
        }
        else {
            @SuppressWarnings ("unchecked")
            Function<A, A> downstreamFinisher = (Function<A, A>) downstream. finisher ();
            Function<Map<K, A>, M> finisher = intermediate -> {
                intermediate. replaceAll ((k, v) -> downstreamFinisher. apply (v));
                @SuppressWarnings ("unchecked")
                M castResult = (M) intermediate;
                return castResult;
            };
            return new CollectorImpl<>(mangledFactory, accumulator, merger, finisher, CH_NOID);
        }
    }
```
上面三个方法类似，可以这样理解，首先需要定义是用传入数据的哪个属性进行分组，于是需要传入一个转换器，比如 `Student::getAge`。其次还需要一个收集器来收集最终结果，默认的容器就是 `Map`。最终默认收集器是 `List`。
```java
// List: {10=[Student (id=3, name=王五, birthday=2011-03-03, age=10, score=32.123)], 11=[Student (id=2, name=李四, birthday=2010-02-02, age=11, score=22.123)], 12=[Student (id=1, name=张三, birthday=2009-01-01, age=12, score=12.123)]}
final Map<Integer, List<Student>> map1 = students. stream (). collect (Collectors. groupingBy (Student::getAge));
// Set: {10=[Student (id=3, name=王五, birthday=2011-03-03, age=10, score=32.123)], 11=[Student (id=2, name=李四, birthday=2010-02-02, age=11, score=22.123)], 12=[Student (id=1, name=张三, birthday=2009-01-01, age=12, score=12.123)]}
final Map<Integer, Set<Student>> map12 = students. stream (). collect (Collectors. groupingBy (Student:: getAge, Collectors. toSet ()));
```

如果想要线程安全，可以使用 `groupingByConcurrent`

注意：如果上述例子想返回 `Map<Integer, Student>` 该如何处理？
那需要使用下面的 `toMap` 方法


## 11 . partitioningBy
```java
public static <T> Collector<T, ?, Map<Boolean, List<T>>> 
    partitioningBy (Predicate<? super T> predicate) {
        return partitioningBy (predicate, toList ());
}
    
public static <T, D, A> Collector<T, ?, Map<Boolean, D>> 
partitioningBy (Predicate<? super T> predicate, Collector<? super T, A, D> downstream) {
    BiConsumer<A, ? super T> downstreamAccumulator = downstream. accumulator ();
    BiConsumer<Partition<A>, T> accumulator = (result, t) ->
            downstreamAccumulator. accept (predicate. test (t) ? result. forTrue : result. forFalse, t);
    BinaryOperator<A> op = downstream. combiner ();
    BinaryOperator<Partition<A>> merger = (left, right) ->
            new Partition<>(op. apply (left. forTrue, right. forTrue),
                            op. apply (left. forFalse, right. forFalse));
    Supplier<Partition<A>> supplier = () ->
            new Partition<>(downstream. supplier (). get (),
                            downstream. supplier (). get ());
    if (downstream. characteristics (). contains (Collector. Characteristics. IDENTITY_FINISH)) {
        return new CollectorImpl<>(supplier, accumulator, merger, CH_ID);
    }
    else {
        Function<Partition<A>, Map<Boolean, D>> finisher = par ->
                new Partition<>(downstream. finisher (). apply (par. forTrue),
                                downstream. finisher (). apply (par. forFalse));
        return new CollectorImpl<>(supplier, accumulator, merger, finisher, CH_NOID);
    }
}
```
此方法其实和 `grouping` 方法类似，只是更为简单，直接将数据分为了两组，其 `key` 只有 `false` 和 `true`。


## 12 . toMap
```java
public static <T, K, U>
    Collector<T, ?, Map<K, U>> toMap (Function<? super T, ? extends K> keyMapper,
                                    Function<? super T, ? extends U> valueMapper) {
        return toMap (keyMapper, valueMapper, throwingMerger (), HashMap::new);
}

public static <T, K, U>
    Collector<T, ?, Map<K, U>> toMap (Function<? super T, ? extends K> keyMapper,
                                    Function<? super T, ? extends U> valueMapper,
                                    BinaryOperator<U> mergeFunction) {
        return toMap (keyMapper, valueMapper, mergeFunction, HashMap::new);
    }

public static <T, K, U, M extends Map<K, U>>
    Collector<T, ?, M> toMap (Function<? super T, ? extends K> keyMapper,
                                Function<? super T, ? extends U> valueMapper,
                                BinaryOperator<U> mergeFunction,
                                Supplier<M> mapSupplier) {
    BiConsumer<M, T> accumulator
            = (map, element) -> map. merge (keyMapper. apply (element),
                                            valueMapper. apply (element), mergeFunction);
    return new CollectorImpl<>(mapSupplier, accumulator, mapMerger (mergeFunction), CH_ID);
}
```
此方法最基本的功能就是将传入数据使用其中的某个属性转换成 `Map`。如
```java
// {1=Student (id=1, name=张三, birthday=2009-01-01, age=12, score=12.123), 2=Student (id=2, name=李四, birthday=2010-02-02, age=11, score=22.123), 3=Student (id=3, name=王五, birthday=2011-03-03, age=10, score=32.123)}
final Map<String, Student> map11 = students. stream ()
    . collect (Collectors. toMap (Student:: getId, Function. identity ()));
```
这里 `Function. identity ()` 表示数据不变，即不转换
当然数据以其中一个属性为 `key`，可能出现重复 `key`，那样可以选择最终收集哪个，如
```java
// {1=Student (id=1, name=张三, birthday=2009-01-01, age=12, score=12.123), 2=Student (id=2, name=李四, birthday=2010-02-02, age=11, score=22.123), 3=Student (id=3, name=王五, birthday=2011-03-03, age=10, score=32.123)}
final Map<String, Student> map2 = students. stream ()
    . collect (Collectors. toMap (Student:: getId, Function. identity (), (x, y) -> x));
```
这里如果出现重复，那么取第一个为准。当然我们对于 `value` 值也是可以进行转换的
```java
// {1=张三, 2=李四, 3=王五}
final Map<String, String> map3 = students. stream ()
    . collect (Collectors. toMap (Student:: getId, Student:: getName, (x, y) -> x));
```
而对于最终收集哪个不仅仅可以直接去第一个，还可以自定义取哪个，这就是一个 `Function` 类型，如
```java
// {10=Student (id=3, name=王五, birthday=2011-03-03, age=10, score=32.123), 11=Student (id=2, name=李四, birthday=2010-02-02, age=11, score=22.123), 12=Student (id=1, name=张三, birthday=2009-01-01, age=12, score=12.123)}
final Map<Integer, Student> map5 = students. stream ()
    . collect (Collectors. toMap (Student:: getAge, Function. identity (), BinaryOperator. maxBy (Comparator. comparing (Student::getScore))));
```
这里取年龄的得分最高的集合


## 13 . summarizingInt
```java
public static <T>
    Collector<T, ?, IntSummaryStatistics> summarizingInt (ToIntFunction<? super T> mapper) {
    return new CollectorImpl<T, IntSummaryStatistics, IntSummaryStatistics>(
            IntSummaryStatistics:: new,
            (r, t) -> r.accept (mapper. applyAsInt (t)),
            (l, r) -> { l.combine (r); return l; }, CH_ID);
}
```
这就是一个统计方法，类似的还有 `summarizingDouble、summarizingLong`。

## 14 . collectingAndThen
```java
public static<T, A, R, RR> Collector<T, A, RR> collectingAndThen (Collector<T, A, R> downstream,
                                                                Function<R, RR> finisher) {
        ...
        return new CollectorImpl<>(downstream. supplier (),
                                   downstream. accumulator (),
                                   downstream. combiner (),
                                   downstream. finisher (). andThen (finisher),
                                   characteristics);
    }
```
这个方法其实就是将某个用收集器收集好了的结果重新用另外一个容器就行收集，也就是转换下容器。一个典型的例子就是有一个集合，我们想根据集合中元素的某些字段进行去重
```java
students. stream (). collect (Collectors. collectingAndThen (
	Collectors. toCollection (() -> new TreeSet (Comparator. comparing (Student::getId))), ArrayList::new
));
```
可以看到其实第一个参数也是一个收集器，正常来说是可以直接使用的，但是第二个参数我们重新指定的收集容器，这样就实现了转换。同时这里还内含了排序，也可以使用 `SortedSet`