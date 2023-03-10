网络资源及 NAS 管理工具

相关安装目录是soft

相关文档参考：
https://alist.nn.ci/zh/guide/install/manual.html#%E5%AE%88%E6%8A%A4%E8%BF%9B%E7%A8%8B


文件
`~/Library/LaunchAgents/ci.nn.alist.plist`
是开机自启动脚本，
需要首先使用
`sudo chown root ~/Library/LaunchAgents/ci.nn.alist.plist`
修改属主，然后使用
`launchctl load ~/Library/LaunchAgents/ci.nn.alist.plist`
加载脚本，后面就可以使用本目录下的两个开启和关闭脚本进行管理了

网页端地址
http://127.0.0.1:5244
joyang/walp1314


相关问题参考
https://github.com/alist-org/alist/discussions/1759