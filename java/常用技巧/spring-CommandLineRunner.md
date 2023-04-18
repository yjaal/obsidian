

`Spring boot` 的 ` CommandLineRunner ` 接口主要用于实现在应用初始化后，去执行一段代码块逻辑，这段初始化代码在整个应用生命周期内只会执行一次。


```java
@Component 
public class ApplicationStartupRunner implements CommandLineRunner { 
	protected final Log logger = LogFactory.getLog(getClass()); 
	
	@Override 
	public void run(String... args) throws Exception { 
		logger.info("ApplicationStartupRunner run method Started !!"); 
	} 
}
```

一般可以初始化一个线程池，在 `run` 方法中，具体的执行类可以实现 `Runnable` 接口。