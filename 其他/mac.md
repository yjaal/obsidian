
```toc

```

## 管理 NTFS 硬盘

```sh
# 查看磁盘挂载
diskutil list
```

这里找到自己挂载的磁盘，名字一般就是 disk2s1。默认挂在是只读挂载。首先第一步是要解除挂载

```sh
sudo umount /dev/disk2s1
```

重新挂载 (事先创建好)

```sh
sudo mount -t ntfs -o rw,nobrowse /dev/disk2s1 /Users/YJ/Desktop/MyDisk
```

## 本地相关软件安装记录

```sh
1、安装homebrew和git
/usr/bin/ruby -e "$(curl -fsSL https://gitee.com/xueweihan/codes/vfrgh7z8qcjlx1ubwt6nk71/raw\?blob_name\=brew_
install.sh)"

2、下载iterm2安装
然后离线安装 oh-my-zsh
# 从git上把oh-my-zsh clone下来到根目录下
git clone git://github.com/robbyrussell/oh-my-zsh.git ~/.oh-my-zsh
# 再在根目录下copy一份.zshrc配置
cp ~/.oh-my-zsh/templates/zshrc.zsh-template ~/.zshrc


3、安装7zip
brew install p7zip
7z检查
压缩 sputnik 文件夹下的所有文件
$ 7z a heed.7z sputnik
解压 heed.7z
$ 7z x heed.7z

4、相关环境变量配置在
vim ~/.zshrc

5、maven本地仓库地址
/Users/YJ/.m2/repository

6、golang安装
https://www.jianshu.com/p/79bdd20c46cf

7、在mac终端下执行：brew install glew
结果报错：

Error: Another active Homebrew update process is already in progress.
Please wait for it to finish or terminate it to continue.

解决方法：rm -rf /usr/local/var/homebrew/locks

8、rocketMQ
在虚拟机中安装的

jdk dmg包解压,不过后面都用openjdk比较好
1. 7z x jdk-8u341-macosx-x64.dmg
2. cd JDK\ 8\ Update\ 341
3. 7z x JDK\ 8\ Update\ 341.pkg
4. 7z x jdk1.8.0_341.pkg
5. 7z x Payload\~
6. cd Content
7. mv Home jdk8

openjdk二进制包及源码下载导入
下载地址
https://github.com/adoptium/temurin8-binaries/releases/tag/jdk8u345-b01
在idea中将源码jdk-source/jdk/src/share/classes目录添加到SDK->源路径

9、github超时
在https://www.ipaddress.com中搜索github实时ip，然后添加到
/etc/hosts中，如
192.168.0.1 github.com


10、oh_my_zsh相关插件和主题放在ohmyzsh目录中，主题现在使用的是powelevel10k
升级命令 omz update
```