
```sh
docker pull postgres:latest


docker run -d --name mypg -p 127.0.0.1:5432:5432 -e POSTGRES_PASSWORD=walp1314 \

-v /Users/YJ/study/docker-work/mypg/conf:/etc/postgresql/conf \

-v /Users/YJ/study/docker-work/mypg/data:/var/lib/postgresql/data \

postgres -c config_file=/etc/postgresql/conf/postgresql.conf
```

这里需要创建相关目录和配置文件，同时需要修改权限可写。这里只是设置了密码，其实默认用户名和默认数据库都是 `postgres`。

进入到容器之后

```sql
-- 进入到数据库中
psql
-- 获知
psql -U postgres

-- 查看所有数据库
postgres# \l
-- 切换数据库
postgres# \c xx
-- 退出
postgres# \q
```

使用相关客户端工具连接时可能会报错，需要修改配置文件

```sh
# /Users/YJ/study/docker-work/mypg/conf/postgresql.conf 添加
listen_addresses = '*'

# /Users/YJ/study/docker-work/mypg/data/pg_hba.conf添加
host all all 0.0.0.0/0 trust
```
记得重启，使用 `pg_ctl restart` 之后容器每次都只是关闭，无法启动。可以直接使用 `docker restrat xx`


1、使用 `psql` 命令登录 PostgreSQL 控制台。

```sh
su - postgres
psql
```

这时相当于系统用户 `postgres` 以同名数据库用户的身份，登录数据库，这是不用输入密码的。如果一切正常，系统提示符会变为” `postgres=#`”，表示这时已经进入了数据库控制台。以下的命令都在控制台内完成。 

2、为 postgres 用户设置一个密码, 当然如果创建容器的时候设置了这里就不用再次设置了

```
\password postgres
```

3、创建数据库用户 `dbuser `，并设置密码。

```
CREATE USER mypg WITH PASSWORD 'walp1314';
// 删除一个用户 drop user mypg;
// 切换用户 \c mypg;
```

4、创建用户数据库，这里为 `kotlinstart`，并指定所有者为 `mypg`。

```
CREATE DATABASE kotlinstart OWNER mypg;
```

5、将 `kotlinstart` 数据库的所有权限都赋予 `mypg`，否则 `mypg` 只能登录控制台，没有任何数据库操作权限。

```
GRANT ALL PRIVILEGES ON DATABASE kotlinstart to mypg;
```


