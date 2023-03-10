```toc

```

一种下载工具

## 安装配置

```sh
# 安装
brew install aria2

# 配置
$ cd ~ 
$ mkdir .aria2 
$ cd .aria2 
$ vim aria2.conf
```

```sh
## 文件保存相关 ##

# 文件的保存路径(可使用绝对路径或相对路径), 默认: 当前启动位置
dir=/Users/YJ/Downloads

# 启用磁盘缓存, 0为禁用缓存, 需1.16以上版本, 默认:16M
#disk-cache=32M
#disk-cache=32M
# 文件预分配方式, 能有效降低磁盘碎片, 默认:prealloc
# 预分配所需时间: none < falloc ? trunc < prealloc
# falloc和trunc则需要文件系统和内核支持
# NTFS建议使用falloc, EXT3/4建议trunc, MAC 下需要注释此项
file-allocation=prealloc
# 断点续传
continue=true

## 下载连接相关 ##

# 最大同时下载任务数, 运行时可修改, 默认:5
max-concurrent-downloads=10
# 同一服务器连接数, 添加时可指定, 默认:1
max-connection-per-server=10
# 最小文件分片大小, 添加时可指定, 取值范围1M -1024M, 默认:20M
# 假定size=10M, 文件为20MiB 则使用两个来源下载; 文件为15MiB 则使用一个来源下载
min-split-size=10M
# 单个任务最大线程数, 添加时可指定, 默认:5
split=5
# 整体下载速度限制, 运行时可修改, 默认:0
#max-overall-download-limit=0
# 单个任务下载速度限制, 默认:0
#max-download-limit=0
# 整体上传速度限制, 运行时可修改, 默认:0
#max-overall-upload-limit=0
# 单个任务上传速度限制, 默认:0
#max-upload-limit=0
# 禁用IPv6, 默认:false
disable-ipv6=true

## 进度保存相关 ##

# 从会话文件中读取下载任务
input-file=/Users/YJ/.aria2/aria2.session
# 在Aria2退出时保存`错误/未完成`的下载任务到会话文件
save-session=/Users/YJ/.aria2/aria2.session
# 定时保存会话, 0为退出时才保存, 需1.16.1以上版本, 默认:0
save-session-interval=60

## RPC相关设置 ##

enable-rpc=true
pause=false
rpc-allow-origin-all=true
rpc-listen-all=true
rpc-save-upload-metadata=true
rpc-secure=false

#设置加密的密钥，自己使用就不设置了
# rpc-secret=160131
# 启用RPC, 默认:false
enable-rpc=true
# 允许所有来源, 默认:false
rpc-allow-origin-all=true
# 允许非外部访问, 默认:false
rpc-listen-all=true
# 事件轮询方式, 取值:[epoll, kqueue, port, poll, select], 不同系统默认值不同
#event-poll=select
# RPC监听端口, 端口被占用时可以修改, 默认:6800
rpc-listen-port=6800
# 设置的RPC授权令牌, v1.18.4新增功能, 取代 --rpc-user 和 --rpc-passwd 选项
#rpc-secure=<taken>
# 设置的RPC访问用户名, 此选项新版已废弃, 建议改用 --rpc-secret 选项
#rpc-user=joyang
# 设置的RPC访问密码, 此选项新版已废弃, 建议改用 --rpc-secret 选项
#rpc-passwd=walp1314

## BT/PT下载相关 ##

# 当下载的是一个种子(以.torrent结尾)时, 自动开始BT任务, 默认:true
#follow-torrent=true
# BT监听端口, 当端口被屏蔽时使用, 默认:6881-6999
listen-port=51413
# 单个种子最大连接数, 默认:55
#bt-max-peers=55
# 打开DHT功能, PT需要禁用, 默认:true
enable-dht=true
# 打开IPv6 DHT功能, PT需要禁用
#enable-dht6=false
# DHT网络监听端口, 默认:6881-6999
#dht-listen-port=6881-6999
# 本地节点查找, PT需要禁用, 默认:false
bt-enable-lpd=true
# 种子交换, PT需要禁用, 默认:true
enable-peer-exchange=false
# 每个种子限速, 对少种的PT很有用, 默认:50K
#bt-request-peer-speed-limit=50K
# 客户端伪装, PT需要
#peer-id-prefix=-TR2770-
user-agent=Transmission/2.92
#user-agent=netdisk;4.4.0.6;PC;PC-Windows;6.2.9200;WindowsBaiduYunGuanJia
# 当种子的分享率达到这个数时, 自动停止做种, 0为一直做种, 默认:1.0
seed-ratio=1.0
#作种时间大于30分钟，则停止作种
seed-time=30
# 强制保存会话, 话即使任务已经完成, 默认:false
# 较新的版本开启后会在任务完成后依然保留.aria2文件
#force-save=false
# BT校验相关, 默认:true
#bt-hash-check-seed=true
# 继续之前的BT任务时, 无需再次校验, 默认:false
bt-seed-unverified=true
# 保存磁力链接元数据为种子文件(.torrent文件), 默认:false
bt-save-metadata=true
#下载完成后删除.ara2的同名文件
on-download-complete=/Users/YJ/.aria2/delete_aria2
#on-download-complete=/home/pi/aria2/rasp.sh
```


## 启动

```sh
# star.sh
# 禁用ipv6
# -D表示后台静默启动，我们不会在前台看到任何变化。
aria2c --conf-path=/Users/YJ/.aria2/aria2.conf --disable-ipv6 -D
```

开机启动

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
	<dict>
		<key>KeepAlive</key>
		<true/>
		<key>Label</key>
		<string>com.aria2c</string>
		<key>ProgramArguments</key>
		<array>
			<string>/usr/local/Cellar/aria2/1.36.0_1/bin/aria2c</string>
			<string>--conf-path=/Users/YJ/.aria2/aria2.conf</string>
		</array>
		<key>RunAtLoad</key>
		<true/>
	</dict>
</plist>
```

## web 界面

https://aria2c.com/
这个会占用端口，所以直接通过此界面进行上传种子下载，如果此时在终端使用命令进行下载，那么会报端口被占用。



## 电影下载

rarbg 可靠的镜像和代理

1.  **[https://rarbg.to/](https://rarbg.to/)** （官方网站）
2.  [https://proxyrarbg.org/](https://proxyrarbg.org/) （代理人）
3.  [https://rarbg.is/](https://www.rarbg.is/) （代理人）
4.  [https://rarbgunblock.com/](https://rarbgunblock.com/) （代理人）
5.  [https://rarbgmirror.com/](https://rarbgmirror.com/) （镜子）
6.  [https://rarbgprx.org](https://rarbgprx.org/) （代理人）
7.  [http://rarbgproxy.org/](http://rarbgproxy.org/) （代理人）
8.  [http://rarbgaccess.org/](http://rarbgaccess.org/) （代理人）
9.  [http://rarbgmirror.org/](http://rarbgmirror.org/) （镜子）
10.  [https://rarbgproxied.org/](https://rarbgproxied.org/) （代理人）
11.  [https://rargb.to/](https://rargb.to/) （克隆）
12.  [https://rarbggo.org](https://rarbggo.org/) （克隆）

