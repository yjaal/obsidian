```toc

```


## 虚拟机克隆

选择某一个虚拟机右键，选择 `在Finder中显示`，然后直接拷贝。如果此时无法正常启动，可以通过命令行进入此克隆的虚拟机目录，删掉 `xx.vmdk.lck` 和 `xx.vmx.lck` 两个文件夹即可正常启动。

## docker 安装

安装

`https://yeasy.gitbook.io/docker_practice/install/ubuntu`

  

```sh
echo \
"deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
$(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

如果提示

`Package docker-ce is not available, but is referred to by another package`

打开 `/etc/apt/sources.list.d/docker.list` 中添加下面内容
`deb [arch=amd64] https://download.docker.com/linux/ubuntu bionic stable`

更新并安装

```sh
sudo apt-get update
sudo apt-get install docker-ce
```

给用户赋 `docker` 权限

```sh
sudo groupadd docker #添加docker用户组

sudo gpasswd -a $USER docker #将登陆用户加入到docker用户组中

newgrp docker #更新用户组
```

## 共享文件夹设置

### 配置 Vmware Fusion

选择某个虚拟机界面上面的小扳手，也就是设置，选择共享，就可以设置主机上共享的文件夹，比如 `share`

### 配置虚拟机

首先虚拟机要安装 `vmware tool`

进入虚拟机中运行命令：

```sh
sudo mkdir /mnt/hgfs
sudo /usr/bin/vmhgfs-fuse . host:/ /mnt/hgfs -o subtype=vmhgfs-fuse, allow_other
cd /mnt/hgfs
```

可以看到一个 `share` 的文件夹，这个文件夹下就是共享的内容了
