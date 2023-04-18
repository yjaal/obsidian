
参考： https://www.jianshu.com/p/4c0723615a52

在我们的 `web` 程序中，用 `spring` 来管理各个实例 (`bean`), 有时在程序中为了使用已被实例化的 `bean`, 通常会用到这样的代码：

```java
ApplicationContext appContext = new ClassPathXmlApplicationContext("applicationContext-common.xml");  
AbcService abcService = (AbcService)appContext.getBean("abcService");  
```

但是这样就会存在一个问题：因为它会重新装载 `applicationContext-common.xml` 并实例化上下文 `bean`，如果有些线程配置类也是在这个配置文件中，那么会造成做相同工作的的线程会被启两次。一次是 `web` 容器初始化时启动，另一次是上述代码显示的实例化了一次。当于重新初始化一遍！这样就产生了冗余。

## 解决方法

不用类似 `new ClassPathXmlApplicationContext()` 的方式，从已有的 `spring` 上下文取得已实例化的 `bean`。通过 `ApplicationContextAware` 接口进行实现。

当一个类实现了这个接口（`ApplicationContextAware`）之后，这个类就可以方便获得 `ApplicationContext` 中的所有 `bean`。换句话说，就是这个类可以直接获取 `spring`  配置文件中，所有有引用到的 `bean` 对象。

## ApplicationContextAware怎么用

实现 `ApplicationContextAware` 接口

```java
@Component
public class AppUtil implements ApplicationContextAware {

    private static ApplicationContext ac;
    
    @Override
    public void setApplicationContext(ApplicationContext ac) throws BeansException {
        this.ac = ac;
    }

    public static Object getObject(String id) {
        Object object = null;
        object = applicationContext.getBean(id);
        return object;
    }
}
```

  
## 使用场景

`ApplicationContextAware` 获取 `ApplicationContext` 上下文的情况，适用于当前运行的代码和已启动的 `Spring` 代码处于同一个 `Spring` 上下文，否则获取到的 `ApplicationContext` 是空的.

常见的使用场景有:

-   在监听器中, 去监听到事件的发生, 然后从 `Spring` 的上下文中获取 `Bean` 对象, 去修改保存相关的业务数据.
-   在某个工具类中, 需要提供处理通用业务数据的方法等

  