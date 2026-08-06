
登录报错

```
Add correct host key in C:\\Users\\joyang/.ssh/known_hosts to get rid of this message. Offending RSA key in C:\\Users\\joyang/.ssh/known_hosts:6 Host key for [uatywts.weoa.com](https://uatywts.weoa.com/) has changed and you have requested strict checking. Host key verification failed.
```


这个就是需要重新生成一个 key

```
ssh-keygen -R uatywts.weoa.com
```

生成过程中如果要输入密码，可以不用输入，三次回车即可。