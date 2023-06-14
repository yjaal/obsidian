
之前学习 CAS 时生成过证书，后面经常就报错

```
Unexpected error: java.security.InvalidAlgorithmParameterException: the trustAnchors parameter must be non-empty
```

```
mvn compile
-Dmaven.wagon.http.ssl.insecure=true -Dmaven.wagon.http.ssl.allowall=true -Dmaven.wagon.http.ssl.ignore.valid
```

如果还是不行，就重装jdk