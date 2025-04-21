
```
brew install colima
```


启动

```bash
# 开机自启动
brew services start colima

#  手动启动
/usr/local/opt/colima/bin/colima start -f
```


配置镜像源

```
sudo vim ~/.colima/default
```

```bash
docker:
  registry-mirrors:
    # 网易云
    - https://hub-mirror.c.163.com
    # 百度云
    - https://mirror.baidubce.com
    # Azure中国
    - https://dockerhub.azk8s.cn
    # 科大
    - https://docker.mirrors.ustc.edu.cn
    # 南京大学
    - https://docker.nju.edu.cn
```


以下是一些常见的 Colima CLI 命令及其说明：

- `colima start` - 启动 Colima 容器运行时。
- `colima stop` - 停止 Colima 容器运行时。
- `colima restart` - 重启 Colima 容器运行时。
- `colima status` - 显示 Colima 容器运行时的状态信息。
- `colima ssh` - 通过 SSH 连接到 Colima 容器运行时。
- `colima ip` - 显示 Colima 容器运行时的 IP 地址。
- `colima info` - 显示有关 Colima 容器运行时的详细信息，包括版本、磁盘使用情况和安装路径等。
- `colima doctor` - 运行诊断程序以检查 Colima 容器运行时的配置和设置是否正确。
- `colima web` - 在本地浏览器中打开容器中运行的应用程序。

  
