
实现此接口用于在 Bean 的生命周期结束前做一些收尾清理工作，接口定义

```java
public interface DisposableBean {
	void destory() throws Exception;
}
```

