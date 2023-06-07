
```
# docker 中下载 mysql
docker pull mysql

#启动
docker run --name mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=walp1314 -d mysql

#进入容器
docker exec -it mysql bash
```

用户名：`root`，密码：`walp1314`


