
```java
ClassPathResource privateKeyResources = new ClassPathResource("sys_priv.pem");
            
File privateKeyFile = privateKeyResources.getFile();
```

这里文件 `sys_priv.pem` 是存放在 `resource` 目录中的，这里不需要这样使用 `classpath:sys_priv.pem`

