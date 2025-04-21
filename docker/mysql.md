
```
# docker 中下载 mysql
docker pull mysql

#启动
docker run --name mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=walp1314 -d mysql

#进入容器
docker exec -it mysql bash

# 登陆
mysql -u root -p

# 创建数据库
CREATE  DATABASE  x_app   CHARACTER  SET  utf8mb4  
  COLLATE  utf8mb4_general_ci;
```

用户名：`root`，密码：`walp1314`


