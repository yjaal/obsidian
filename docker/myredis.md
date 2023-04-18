
启动
```sh
docker run -p 127.0.0.1:6379:6379 -v /Users/YJ/study/docker-work/myredis/conf:/usr/local/etc/redis --name myredis redis redis-server /usr/local/etc/redis/redis.conf
```

这里把配置文件放在了本地
`/Users/YJ/study/docker-work/myredis/conf/redis.conf`
里面添加了密码。里面的 `data` 目录可以用来挂载数据，目前还没用到。

```sh
requirepass walp1314
```

可以通过 docker 管理台登陆到容器中

```
> redis-cli
> auth walp1314
```

通过相关工具连接的时候不需要填用户名，只需要填密码即可。