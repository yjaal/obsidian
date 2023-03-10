
可以把本地的 `ssh` 公钥文件安装到远程主机对应的账户下。

使用模式：  
`ssh-copy-id [-i [identity_file]] [user@]machine_ip`
   
描述：    
`ssh-copy-id` 是一个实用 `ssh` 去登陆到远程服务器的脚本（假设使用一个登陆密码。因此，密码认证应该被激活直到你已经清理了做了多个身份的使用）。它也能够改变远程用户名的权限，`~/.ssh 和~/.ssh/authorized_keys`  删除群组写的权限（在其它方面，如果远程机上的 `sshd` 在它的配置文件中是严格模式的话，这能够阻止你登陆。）。如果这个 “`-i`”选项已经给出了，然后这个认证文件（默认是 `~/.ssh   /id_rsa.pub`）被使用，不管在你的 `ssh-agent` 那里是否有任何密钥。另外，命令 “`ssh-add -L`” 提供任何输出，它使用这个输出优先于身份认证文件。如果给出了参数“`-i`”选项，或者 ` ssh-add ` 不产生输出，然后它使用身份认证文件的内容。一旦它有一个或者多个指纹，它使用 `ssh` 将这些指纹填充到远程机 `~/.ssh/authorized_keys ` 文件中。

**使用**

```sh
# 一路回车即可
> ssh-keygen
# 注意: ssh-copy-id 将key写到远程机器的 ~/.ssh/authorized_key.文件中
> ssh-copy-id -i ~/.ssh/id_rsa.pub user@remote_ip
```