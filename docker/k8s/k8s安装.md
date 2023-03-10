```toc

```
## Docker 安装

https://yeasy.gitbook.io/docker_practice/install/ubuntu

设置镜像源

```sh
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://hnkfbj7x.mirror.aliyuncs.com"]
}
EOF

sudo systemctl restart docker.service
sudo systemctl restart docker.socket

docker info
```

## 配置

设置 hostname

```sh
# 192.168.67.2
sudo hostnamectl set-hostname k8s-m1 # master 节点执行
# 192.168.67.3
sudo hostnamectl set-hostname k8s-n1 # node 节点执行
# 192.168.67.5
sudo hostnamectl set-hostname k8s-n2 # node 节点执行
```

设置 hosts

```sh
cat << EOF | sudo tee -a /etc/hosts
192.168.67.2 k8s-m1
192.168.67.3 k8s-n1
192.168.67.5 k8s-2
EOF
```

禁用虚拟内存 swap

```sh
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

加载内核模块

```sh
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```

网桥设置

```sh
# 设置所需的 sysctl 参数，参数在重新启动后保持不变
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
   
# 应用 sysctl 参数而不重新启动
sudo sysctl --system
```

systemd cgroup 驱动配置

```sh
# 生成默认配置
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1
# 修改 cgroup 为 systemd
sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
```

重启容器运行环境

```sh
sudo systemctl restart containerd
```


## 安装 kubeadm, kubelet, kubectl

更新 `apt` 包索引并安装使用 Kubernetes `apt` 仓库所需要的包：

```shell
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
```


下载 Google Cloud 公开签名秘钥：

```shell
sudo curl -fsSLo /etc/apt/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
```


添加 Kubernetes `apt` 仓库：

```shell
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

如果第二步出错，那么需要手动下载 `https://packages.cloud.google.com/apt/doc/apt-key.gpg`，然后拷贝到
`/usr/share/keyrings/kubernetes-archive-keyring.gpg`，注意，文件名也修改了。

然后将
```sh
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

修改为

```sh
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] http://mirrors.ustc.edu.cn/kubernetes/apt kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```


更新 `apt` 包索引，安装 kubelet、kubeadm 和 kubectl，并锁定其版本：

```sh
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

# 设置开机自启动
sudo systemctl start kubelet
sudo systemctrl enable kubelet
```

## 部署 master

```sh
# 修改为对应 master ip

# pod-network-cidr 是对应 pod ip 段

sudo kubeadm init --kubernetes-version=1.26.2 --apiserver-advertise-address=192.168.67.2 --service-cidr=10.96.0.0/12 --pod-network-cidr=10.244.0.0/16 --image-repository registry.aliyuncs.com/google_containers
```


Todo