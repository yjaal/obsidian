
`spring` 初始化 `bean` 有两种方式：  
- 实现 `InitializingBean` 接口，继而实现 `afterPropertiesSet` 的方法  
- 反射原理，配置文件使用 `init-method` 标签直接注入 `bean`

*通过此接口我们可以实现相关缓存的初始化工作。*

相同点： 实现注入 `bean` 的初始化。

不同点：  
- 实现的方式不一致。  
- 接口比配置效率高，但是配置消除了对 `spring` 的依赖。而实现 `InitializingBean` 接口依然采用对 `spring` 的依赖。


`InitializingBean` 接口为 `bean` 提供了初始化方法的方式，它只包括 `afterPropertiesSet` 方法，凡是继承该接口的类，在初始化 `bean` 的时候都会执行该方法。


接口定义

```java
public interface InitializingBean {
    void afterPropertiesSet() throws Exception;
}
```

  在 `initializeBean` 的方法中调用了 `invokeInitMethods` 方法，在 `invokeInitMethods` 会判断 `bean` 是否实现了 `InitializingBean` 接口，如果实现了个该接口，则调用 `afterPropertiesSet` 方法执行自定义的初始化过程，`invokeInitMethods` 实现代码如下：

  ```java
protected void invokeInitMethods(String beanName, final Object bean, 
								 RootBeanDefinition mbd) throws Throwable {
		  //判断是否实现了InitializingBean接口
	boolean isInitializingBean = (bean instanceof InitializingBean);
	if (isInitializingBean && 
	(mbd == null || !mbd.isExternallyManagedInitMethod("afterPropertiesSet"))) {
		if (System.getSecurityManager() != null) {
			try {
				AccessController.doPrivileged(new PrivilegedExceptionAction<Object>() {
					@Override
					public Object run() throws Exception {
						((InitializingBean) bean).afterPropertiesSet();
						return null;
					}
				}, getAccessControlContext());
			}
			catch (PrivilegedActionException pae) {
				throw pae.getException();
			}
		} else {
			//调用afterPropertiesSet方法执行自定义初始化流程
			((InitializingBean) bean).afterPropertiesSet();
		}
	}

	if (mbd != null) {
		String initMethodName = mbd.getInitMethodName();
		if (initMethodName != null && !(isInitializingBean && "afterPropertiesSet".equals(initMethodName)) &&
				!mbd.isExternallyManagedInitMethod(initMethodName)) {
			invokeCustomInitMethod(beanName, bean, mbd);
		}
	}
}
```

