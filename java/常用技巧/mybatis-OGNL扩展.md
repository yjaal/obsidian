
mybatis 中经常需要判断是否为空等情况，如下面判断一个集合是否为空，如果不为空，则执行相关动作

```xml
<if test="collection!=null and collection.size()> 0"> 
	and some_col = #{some_val} 
</if>
```

但是这里使用这种写法比较麻烦，我们可以自定义一个工具类

```java
package cn.demo; 

public final class CollectionUtils { 
	public static boolean isNotEmpty( Collection<?> collection) {
		return (collection != null && !collection.isEmpty()); 
	} 
}
```

然后在就可以在 mybatis 中这样写了

```xml
<if test="@cn.demo.CollectionUtils@isNotEmpty(collection)"> 
	and some_col = #{some_val} 
</if>
```