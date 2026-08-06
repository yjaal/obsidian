
1、先安装 nvm
直接下载安装包安装，最好使用默认目录，不然会比较麻烦。不要下载 1.2.x 版本（有 bug）


2、添加镜像源
```
安装目录中找到settings.txt
添加

node_mirror: https://nodejs.org/dist/
npm_mirror: https://npm.taobao.com/mirrors/npm/
```

3、安装 node

```
nvm install lts --verbose
```


4、添加 npm 镜像源

```
npm config set registry https://registry.npmmirror.com
```



注意：

清楚缓存
```
nvm cache clear
```



5、设置 npm 镜像源

```
npm config set registry https://registry.npmmirror.com
```

检查生效
```
npm config get registry
```