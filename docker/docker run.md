
`docker run` ：创建一个新的容器并运行一个命令

## 语法

`docker run [OPTIONS] IMAGE [COMMAND] [ARG...]`

`OPTIONS` 说明：

-   **-a stdin:** 指定标准输入输出内容类型，可选 `STDIN/STDOUT/STDERR` 三项；
    
-   **-d:** 后台运行容器，并返回容器 `ID`；
    
-   **-i:** 以交互模式运行容器，通常与 `-t` 同时使用；
    
-   **-P:** 随机端口映射，容器内部端口**随机**映射到主机的端口
    
-   **-p:** 指定端口映射，格式为：主机(宿主)端口:容器端口
    
-   **-t:** 为容器重新分配一个伪输入终端，通常与 `-i` 同时使用；
    
-   **--name="nginx-lb":** 为容器指定一个名称；
    
-   **--dns 8.8.8.8:** 指定容器使用的 `DNS` 服务器，默认和宿主一致；
    
-   **--dns-search example. com:** 指定容器 `DNS` 搜索域名，默认和宿主一致；
    
-   **-h "mars":** 指定容器的 `hostname`；
    
-   **-e username="ritchie":** 设置环境变量；
    
-   **--env-file=[]:** 从指定文件读入环境变量；
    
-   **--cpuset="0-2" or --cpuset="0, 1, 2":** 绑定容器到指定 `CPU` 运行；
    
-   **-m :**设置容器使用内存最大值；
    
-   **--net="bridge":** 指定容器的网络连接类型，支持 `bridge/host/none/container`: 四种类型；
    
-   **--link=[]:** 添加链接到另一个容器；
    
-   **--expose=[]:** 开放一个端口或一组端口；
    
-   **--volume , -v:** 绑定一个卷
    

## 实例

使用 `docker` 镜像 `nginx:latest` 以后台模式启动一个容器, 并将容器命名为 `mynginx`。

```sh
docker run --name mynginx -d nginx:latest
```

使用镜像 `nginx:latest` 以后台模式启动一个容器, 并将容器的 `80` 端口映射到主机随机端口。

```sh
docker run -P -d nginx:latest
```

使用镜像 `nginx:latest`，以后台模式启动一个容器, 将容器的 `80` 端口映射到主机的 `80` 端口, 主机的目录 `/data` 映射到容器的 `/data`。

```sh
docker run -p 80:80 -v /data:/data -d nginx:latest
```

绑定容器的 `8080` 端口，并将其映射到本地主机 `127.0.0.1` 的 `80` 端口上。

```sh
docker run -p 127.0.0.1:80:8080/tcp ubuntu bash
```

使用镜像 `nginx:latest` 以交互模式启动一个容器, 在容器内执行 `/bin/bash` 命令。

```sh
docker run -it nginx:latest /bin/bash
```