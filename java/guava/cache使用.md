
```toc

```

摘自: https://www.cnblogs.com/rickiyang/p/11074159.html

缓存分为本地缓存和远端缓存。常见的远端缓存有 `Redis，MongoDB`；本地缓存一般使用 `map` 的方式保存在本地内存中。一般我们在业务中操作缓存，都会操作缓存和数据源两部分。如：`put` 数据时，先插入 `DB`，再删除原来的缓存；`get` 数据时，先查缓存，命中则返回，没有命中时，需要查询 `DB`，再把查询结果放入缓存中。如果访问量大，我们还得兼顾本地缓存的线程安全问题。必要的时候也要考虑缓存的回收策略。

今天说的 `Guava Cache` 是 `google guava` 中的一个内存缓存模块，用于将数据缓存到 `JVM` 内存中。他很好的解决了上面提到的几个问题：

-   很好的封装了 `get、put` 操作，能够集成数据源；
-   线程安全的缓存，与 `ConcurrentMap` 相似，但前者增加了更多的元素失效策略，后者只能显示的移除元素；
-   `Guava Cache` 提供了三种基本的缓存回收方式：基于容量回收、定时回收和基于引用回收。定时回收有两种：按照写入时间，最早写入的最先回收；按照访问时间，最早访问的最早回收；
-   监控缓存加载/命中情况

`Guava Cache` 的架构设计灵感 `ConcurrentHashMap`，在简单场景中可以通过 `HashMap` 实现简单数据缓存，但如果要实现缓存随时间改变、存储的数据空间可控则缓存工具还是很有必要的。`Cache` 存储的是键值对的集合，不同时是还需要处理缓存过期、动态加载等算法逻辑，需要额外信息实现这些操作，对此根据面向对象的思想，还需要做方法与数据的关联性封装，主要实现的缓存功能有：自动将节点加载至缓存结构中，当缓存的数据超过最大值时，使用 `LRU` 算法替换；它具备根据节点上一次被访问或写入时间计算缓存过期机制，缓存的 `key` 被封装在 `WeakReference` 引用中，缓存的 `value` 被封装在 `WeakReference` 或 `SoftReference` 引用中；还可以统计缓存使用过程中的命中率、异常率和命中率等统计数据。

## 构建

我们先看一个示例，再来讲解使用方式：

```java
package com.rickiyang.learn.cache;

import com.google.common.cache.CacheBuilder;
import com.google.common.cache.CacheLoader;
import com.google.common.cache.LoadingCache;

import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.Random;
import java.util.concurrent.TimeUnit;

public class GuavaCacheService {

    public void setCache() {
        LoadingCache<Integer, String> cache = CacheBuilder.newBuilder()
                //设置并发级别为8，并发级别是指可以同时写缓存的线程数
                .concurrencyLevel(8)
                //设置缓存容器的初始容量为10
                .initialCapacity(10)
                //设置缓存最大容量为100，超过100之后就会按照LRU最近虽少使用算法来移除缓存项
                .maximumSize(100)
                //是否需要统计缓存情况,该操作消耗一定的性能,生产环境应该去除
                .recordStats()
                //设置写缓存后n秒钟过期
                .expireAfterWrite(60, TimeUnit.SECONDS)
                //设置读写缓存后n秒钟过期,实际很少用到,类似于expireAfterWrite
                //.expireAfterAccess(17, TimeUnit.SECONDS)
                //只阻塞当前数据加载线程，其他线程返回旧值
                //.refreshAfterWrite(13, TimeUnit.SECONDS)
                //设置缓存的移除通知
                .removalListener(notification -> {
                    System.out.println(notification.getKey() + " " + notification.getValue() + " 被移除,原因:" + notification.getCause());
                })
                //build方法中可以指定CacheLoader，在缓存不存在时通过CacheLoader的实现自动加载缓存
                .build(new DemoCacheLoader());

        //模拟线程并发
        new Thread(() -> {
            //非线程安全的时间格式化工具
            SimpleDateFormat simpleDateFormat = new SimpleDateFormat("HH:mm:ss");
            try {
                for (int i = 0; i < 10; i++) {
                    String value = cache.get(1);
                    System.out.println(Thread.currentThread().getName() + " " + simpleDateFormat.format(new Date()) + " " + value);
                    TimeUnit.SECONDS.sleep(3);
                }
            } catch (Exception ignored) {
            }
        }).start();

        new Thread(() -> {
            SimpleDateFormat simpleDateFormat = new SimpleDateFormat("HH:mm:ss");
            try {
                for (int i = 0; i < 10; i++) {
                    String value = cache.get(1);
                    System.out.println(Thread.currentThread().getName() + " " + simpleDateFormat.format(new Date()) + " " + value);
                    TimeUnit.SECONDS.sleep(5);
                }
            } catch (Exception ignored) {
            }
        }).start();
        //缓存状态查看
        System.out.println(cache.stats().toString());

    }

    /**
     * 随机缓存加载,实际使用时应实现业务的缓存加载逻辑,例如从数据库获取数据
     */
    public static class DemoCacheLoader extends CacheLoader<Integer, String> {
        @Override
        public String load(Integer key) throws Exception {
            System.out.println(Thread.currentThread().getName() + " 加载数据开始");
            TimeUnit.SECONDS.sleep(8);
            Random random = new Random();
            System.out.println(Thread.currentThread().getName() + " 加载数据结束");
            return "value:" + random.nextInt(10000);
        }
    }
}

```

上面一段代码展示了如何使用 `Cache` 创建一个缓存对象并使用它。

`LoadingCache` 是 `Cache` 的子接口，相比较于 `Cache`，当从 `LoadingCache` 中读取一个指定 `key` 的记录时，如果该记录不存在，则 `LoadingCache` 可以自动执行加载数据到缓存的操作。

在调用 `CacheBuilder` 的 `build` 方法时，必须传递一个 `CacheLoader` 类型的参数，`CacheLoader` 的 `load` 方法需要我们提供实现。当调用 `LoadingCache` 的 `get` 方法时，如果缓存不存在对应 `key` 的记录，则 `CacheLoader` 中的 `load` 方法会被自动调用从外存加载数据，`load` 方法的返回值会作为 `key` 对应的 `value` 存储到 `LoadingCache` 中，并从 `get` 方法返回。

当然如果你不想指定重建策略，那么你可以使用无参的 `build()` 方法，它将返回 `Cache` 类型的构建对象。

`CacheBuilder` 是 `Guava` 提供的一个快速构建缓存对象的工具类。`CacheBuilder` 类采用 `builder` 设计模式，它的每个方法都返回 `CacheBuilder` 本身，直到 `build` 方法被调用。该类中提供了很多的参数设置选项，你可以设置 `cache` 的默认大小，并发数，存活时间，过期策略等等。

## 可选配置分析

### 缓存的并发级别

`Guava` 提供了设置并发级别的 `api`，使得缓存支持并发的写入和读取。同 `ConcurrentHashMap` 类似 `Guava cache` 的并发也是通过分离锁实现。在一般情况下，将并发级别设置为服务器 `cpu` 核心数是一个比较不错的选择。

```java
CacheBuilder.newBuilder()
		// 设置并发级别为cpu核心数
		.concurrencyLevel(Runtime.getRuntime().availableProcessors()) 
		.build();
```

### 缓存的初始容量设置

我们在构建缓存时可以为缓存设置一个合理大小初始容量，由于 `Guava` 的缓存使用了分离锁的机制，扩容的代价非常昂贵。所以合理的初始容量能够减少缓存容器的扩容次数。

```java
CacheBuilder.newBuilder()
		// 设置初始容量为100
		.initialCapacity(100)
		.build();
```

### 设置最大存储

`Guava Cache` 可以在构建缓存对象时指定缓存所能够存储的最大记录数量。当 `Cache` 中的记录数量达到最大值后再调用 `put` 方法向其中添加对象，`Guava` 会先从当前缓存的对象记录中选择一条删除掉，腾出空间后再将新的对象存储到 `Cache` 中。

1.  **基于容量的清除 (`size-based eviction`):** 通过 `CacheBuilder.maximumSize(long)` 方法可以设置 `Cache` 的最大容量数，当缓存数量达到或接近该最大值时，`Cache` 将清除掉那些最近最少使用的缓存;
2.  **基于权重的清除: ** 使用 `CacheBuilder.weigher(Weigher)` 指定一个权重函数，并且用 `CacheBuilder.maximumWeight(long)` 指定最大总重。比如每一项缓存所占据的内存空间大小都不一样，可以看作它们有不同的“权重”（`weights`）。

### 缓存清除策略

#### 1. 基于存活时间的清除

-   `expireAfterWrite` 写缓存后多久过期
-   `expireAfterAccess` 读写缓存后多久过期
-   `refreshAfterWrite` 写入数据后多久过期, 只阻塞当前数据加载线程, 其他线程返回旧值

这几个策略时间可以单独设置,也可以组合配置。

#### 2. 上面提到的基于容量的清除

#### 3. 显式清除

任何时候，你都可以显式地清除缓存项，而不是等到它被回收，`Cache` 接口提供了如下 `API`：

1.  个别清除：`Cache.invalidate(key)`
2.  批量清除：`Cache.invalidateAll(keys)`
3.  清除所有缓存项：`Cache.invalidateAll()`

#### 4. 基于引用的清除（Reference-based Eviction）

在构建 `Cache` 实例过程中，通过设置使用弱引用的键、或弱引用的值、或软引用的值，从而使 `JVM` 在 `GC` 时顺带实现缓存的清除，不过一般不轻易使用这个特性。

-   `CacheBuilder.weakKeys()`：使用弱引用存储键。当键没有其它（强或软）引用时，缓存项可以被垃圾回收。因为垃圾回收仅依赖恒等式，使用弱引用键的缓存用而不是 `equals` 比较键。
-   `CacheBuilder.weakValues()`：使用弱引用存储值。当值没有其它（强或软）引用时，缓存项可以被垃圾回收。因为垃圾回收仅依赖恒等式，使用弱引用值的缓存用而不是 `equals` 比较值。
-   `CacheBuilder.softValues()`：使用软引用存储值。软引用只有在响应内存需要时，才按照全局最近最少使用的顺序回收。考虑到使用软引用的性能影响，我们通常建议使用更有性能预测性的缓存大小限定（见上文，基于容量回收）。使用软引用值的缓存同样用 `==` 而不是 `equals` 比较值。

### 清理什么时候发生

也许这个问题有点奇怪，如果设置的存活时间为一分钟，难道不是一分钟后这个 `key` 就会立即清除掉吗？我们来分析一下如果要实现这个功能，那 `Cache` 中就必须存在线程来进行周期性地检查、清除等工作，很多 `cache` 如 `redis、ehcache` 都是这样实现的。

使用 `CacheBuilder` 构建的缓存不会”自动”执行清理和回收工作，也不会在某个缓存项过期后马上清理，也没有诸如此类的清理机制。相反，它会在写操作时顺带做少量的维护工作，或者偶尔在读操作时做——如果写操作实在太少的话。

这样做的原因在于：如果要自动地持续清理缓存，就必须有一个线程，这个线程会和用户操作竞争共享锁。此外，某些环境下线程创建可能受限制，这样 `CacheBuilder` 就不可用了。参考如下示例：

```java
package com.rickiyang.learn.cache;

import com.google.common.cache.Cache;
import com.google.common.cache.CacheBuilder;

import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.concurrent.TimeUnit;

public class GuavaCacheService {


    static Cache<Integer, String> cache = CacheBuilder.newBuilder()
            .expireAfterWrite(5, TimeUnit.SECONDS)
            .build();

    public static void main(String[] args) throws Exception {
        new Thread(() -> {
            while (true) {
                SimpleDateFormat sdf = new SimpleDateFormat("HH:mm:ss");
                System.out.println(sdf.format(new Date()) + " size: " + cache.size());
                try {
                    Thread.sleep(2000);
                } catch (InterruptedException e) {

                }
            }
        }).start();
        SimpleDateFormat sdf = new SimpleDateFormat("HH:mm:ss");
        cache.put(1, "a");
        System.out.println("写入 key:1 ,value:" + cache.getIfPresent(1));
        Thread.sleep(10000);
        cache.put(2, "b");
        System.out.println("写入 key:2 ,value:" + cache.getIfPresent(2));
        Thread.sleep(10000);
        System.out.println(sdf.format(new Date())
                + " sleep 10s , key:1 ,value:" + cache.getIfPresent(1));
        System.out.println(sdf.format(new Date())
                + " sleep 10s, key:2 ,value:" + cache.getIfPresent(2));
    }
}

部分输出结果：
23:57:36 size: 0
写入 key:1 ,value:a
23:57:38 size: 1
23:57:40 size: 1
23:57:42 size: 1
23:57:44 size: 1
23:57:46 size: 1
写入 key:2 ,value:b
23:57:48 size: 1
23:57:50 size: 1
23:57:52 size: 1
23:57:54 size: 1
23:57:56 size: 1
23:57:56 sleep 10s , key:1 ,value:null
23:57:56 sleep 10s, key:2 ,value:null
23:57:58 size: 0
23:58:00 size: 0
23:58:02 size: 0
    ...
    ...
```

上面程序设置了缓存过期时间为 `5S`，每打印一次当前的 `size` 需要 `2S`，打印了 `5` 次 `size` 之后写入 `key 2`，此时的 `size` 为 1，说明在这个时候才把第一次应该过期的 `key 1` 给删除。

**给移除操作添加一个监听器：**

可以为 `Cache` 对象添加一个移除监听器，这样当有记录被删除时可以感知到这个事件。

```java
RemovalListener<String, String> listener = notification -> System.out.println("[" + notification.getKey() + ":" + notification.getValue() + "] is removed!");
        Cache<String,String> cache = CacheBuilder.newBuilder()
                .maximumSize(5)
                .removalListener(listener)
                .build();
```

**但是要注意的是：**

**默认情况下，监听器方法是在移除缓存时同步调用的。因为缓存的维护和请求响应通常是同时进行的，代价高昂的监听器方法在同步模式下会拖慢正常的缓存请求。在这种情况下，你可以使用`RemovalListeners.asynchronous(RemovalListener, Executor)`把监听器装饰为异步操作。**

### 自动加载

上面我们说过使用 `get` 方法的时候如果 key 不存在你可以使用指定方法去加载这个 `key`。在 `Cache` 构建的时候通过指定 `CacheLoder` 的方式。如果你没有指定，你也可以在 `get` 的时候显式的调用 `call` 方法来设置 `key` 不存在的补救策略。

`Cache` 的 `get` 方法有两个参数，第一个参数是要从 `Cache` 中获取记录的 `key`，第二个记录是一个 `Callable` 对象。

当缓存中已经存在 `key` 对应的记录时，`get` 方法直接返回 `key` 对应的记录。如果缓存中不包含 `key` 对应的记录，`Guava` 会启动一个线程执行 `Callable` 对象中的 `call` 方法，`call` 方法的返回值会作为 `key` 对应的值被存储到缓存中，并且被 `get` 方法返回。

```java
package com.rickiyang.learn.cache;

import com.google.common.cache.Cache;
import com.google.common.cache.CacheBuilder;
import java.util.concurrent.Callable;
import java.util.concurrent.ExecutionException;

public class GuavaCacheService {

    private static Cache<String, String> cache = CacheBuilder.newBuilder()
            .maximumSize(3)
            .build();

    public static void main(String[] args) {

        new Thread(() -> {
            System.out.println("thread1");
            try {
                String value = cache.get("key", new Callable<String>() {
                    public String call() throws Exception {
                        System.out.println("thread1"); //加载数据线程执行标志
                        Thread.sleep(1000); //模拟加载时间
                        return "thread1";
                    }
                });
                System.out.println("thread1 " + value);
            } catch (ExecutionException e) {
                e.printStackTrace();
            }
        }).start();
        new Thread(() -> {
            System.out.println("thread2");
            try {
                String value = cache.get("key", new Callable<String>() {
                    public String call() throws Exception {
                        System.out.println("thread2"); //加载数据线程执行标志
                        Thread.sleep(1000); //模拟加载时间
                        return "thread2";
                    }
                });
                System.out.println("thread2 " + value);
            } catch (ExecutionException e) {
                e.printStackTrace();
            }
        }).start();
    }

}

输出结果为：
thread1
thread2
thread2
thread1 thread2
thread2 thread2

```

可以看到输出结果：两个线程都启动，输出 `thread1，thread2`，接着又输出了 `thread2`，说明进入了 `thread2` 的 `call` 方法了，此时 `thread1` 正在阻塞，等待 `key` 被设置。然后 `thread1` 得到了 `value` 是 `thread2`，`thread2` 的结果自然也是 `thread2`。

这段代码中有两个线程共享同一个 `Cache` 对象，两个线程同时调用 `get` 方法获取同一个 `key` 对应的记录。由于 `key` 对应的记录不存在，所以两个线程都在 `get` 方法处阻塞。此处在 `call` 方法中调用 `Thread.sleep (1000)` 模拟程序从外存加载数据的时间消耗。

从结果中可以看出，虽然是两个线程同时调用 `get` 方法，但只有一个 `get` 方法中的 `Callable` 会被执行 (没有打印出 `load2`)。`Guava` 可以保证当有多个线程同时访问 `Cache` 中的一个 `key` 时，如果 `key` 对应的记录不存在，`Guava` 只会启动一个线程执行 `get` 方法中 `Callable` 参数对应的任务加载数据存到缓存。当加载完数据后，任何线程中的 `get` 方法都会获取到 `key` 对应的值。

### 统计信息

可以对 `Cache` 的命中率、加载数据时间等信息进行统计。在构建 `Cache` 对象时，可以通过 `CacheBuilder` 的 `recordStats` 方法开启统计信息的开关。开关开启后 `Cache` 会自动对缓存的各种操作进行统计，调用 `Cache` 的 `stats` 方法可以查看统计后的信息。

```java
package com.rickiyang.learn.cache;

import com.google.common.cache.Cache;
import com.google.common.cache.CacheBuilder;

public class GuavaCacheService {


    public static void main(String[] args) {
        Cache<String, String> cache = CacheBuilder.newBuilder()
                .maximumSize(3)
                .recordStats() //开启统计信息开关
                .build();
        cache.put("1", "v1");
        cache.put("2", "v2");
        cache.put("3", "v3");
        cache.put("4", "v4");

        cache.getIfPresent("1");
        cache.getIfPresent("2");
        cache.getIfPresent("3");
        cache.getIfPresent("4");
        cache.getIfPresent("5");
        cache.getIfPresent("6");

        System.out.println(cache.stats()); //获取统计信息
    }

}

输出：
CacheStats{hitCount=3, missCount=3, loadSuccessCount=0, loadExceptionCount=0, totalLoadTime=0, evictionCount=1}
```


## Spring 中用法

```java
public interface CodeConfCache{
	CodeConf getByBizType(String bizType);
}

@Component
public class CodeConfCacheGuavaImpl extends CodeConfCache{
	private static LoadingCache<String, Optional<CodeConf>> CODE_CACHE = null;

	@Autowired
	private CodeConfDao codeConfDao;

	private CodeConfCacheGuavaImpl(){
		CODE_CACHE = CacheBuilder.newBuilder()
			.expireAfterWrite(60, TimeUnit.SECONDS)
			.maximumSize(50)
			.build(new CacheLoader<String, Optional<CodeConf>>(){
				@Override
				public Optional<CodeConf> load(String key){
					if(StringUtils.isEmpty(key)){
						return Optional.empty();
					}
					CodeConf conf = codeConfDao.selectByPrimaryKey(key);
					if(Objects.isNull(conf)){
						return Optional.empty();
					}
					return Optional.of(conf);
				}
			});
	}

	@Override
	public Optional<CodeConf> getByBizType(String bizType){
		try{
			return CODE_CACHE.get(bizType);
		}catch(ExecutionException e){
			return Optional.empty();
		}
	}
}
```

这里当缓存中没有我们要的数据时，就会执行 `load` 方法查找。注意：不能返回为 `null`，否则会报异常。如果确有这个需求，可以使用 `Optional`。