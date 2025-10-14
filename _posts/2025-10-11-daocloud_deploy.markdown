---
title:  "daocloud deploy"
date:   2025-10-11 15:01:00 +0800
categories: ["tech","k8s"]
tags: ["daocloud","k8s"]
---

# daocloud 部署

[文档地址](https://docs.daocloud.io/install/community/k8s/k8s-dce-deploy#_1)


文档中标明8C16G，宿主机达不到使用标准，4C8G尝试部署。

## 虚拟机资源

| role | host name | IP |
| ---- | ---- | ---- |
|control-plane | k8s-master01 | 192.168.50.197 | 
| worker-node | k8s-work01 | 192.168.50.150 | 
| worker-node | k8s-work02 | 192.168.50.131 |

1. CentOS Linux release 7.9.2009 (Core)
2. Kubernetes 1.25.8
3. Containerd 1.6.33
4. DCE5
## 环境准备阶段

### 1. CentOS 关闭图形化界面
```bash
systemctl set-default multi-user.target
```

### 2. 设置时区
```bash
timedatectl set-timezone Asia/Shanghai
```

### 3. 设置hostname 
```bash
hostnamectl set-hostname k8s-master01
```
### 4. 关闭swap
```bash
swapoff -a
sed -i '/ swap / s/^/#/' /etc/fstab
```
### 5. 关闭防火墙
```bash
systemctl stop firewalld
systemctl disable firewalld
```
```bash
systemctl status firewalld
```
### 6. 设置内核参数并允许iptables进行桥接流量
```bash
## 加载 br_netfilter 模块
cat <<EOF | tee /etc/modules-load.d/kubernetes.conf
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
## 修改内核参数如 ip_forward 和 bridge-nf-call-iptables：

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# 刷新配置
sysctl --system
```

```bash
cat /etc/modules-load.d/kubernetes.conf
lsmod | grep  br_netfilter
lsmod | grep  overlay
modprobe --show-depends overlay
modprobe --show-depends br_netfilter
sysctl -a | grep net.ipv4.ip_forward
```

### 7. SElinux 设置为 permissive 模式（相当于将其禁用）
```bash
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```

### 8. 修改repo源，安装依赖

```bash
sudo cd /etc/yum.repos.d/
sudo mkdir bak
sudo mv CentOS-*.repo ./bak
sudo curl -o CentOS-base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
sudo yum clean all
sudo yum install -y yum-utils device-mapper-persistent-data lvm2
```

## 安装阶段

### 1. 安装containerd

```bash
sudo yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
sudo yum makecache
yum install containerd.io -y
ctr -v

# 删除自带的 config.toml，避免后续 kubeadm 出现错误 CRI v1 runtime API is not implemented for endpoint
mv /etc/containerd/config.toml /etc/containerd/config.toml.old
# 重新初始化配置
sudo containerd config default | sudo tee /etc/containerd/config.toml
# 更新配置文件内容: 使用 systemd 作为 cgroup 驱动，并且替代 pause 镜像地址
sed -i 's/SystemdCgroup\ =\ false/SystemdCgroup\ =\ true/' /etc/containerd/config.toml
sed -i 's/k8s.gcr.io\/pause/k8s-gcr.m.daocloud.io\/pause/g' /etc/containerd/config.toml 
sed -i 's/registry.k8s.io\/pause/k8s-gcr.m.daocloud.io\/pause/g' /etc/containerd/config.toml
sudo systemctl daemon-reload
sudo systemctl restart containerd
sudo systemctl enable containerd
```

### 2. 安装CNI
```bash
curl -JLO https://github.com/containernetworking/plugins/releases/download/v1.2.0/cni-plugins-linux-amd64-v1.2.0.tgz
mkdir -p /opt/cni/bin &&  tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.2.0.tgz
```
CNI 是 容器网络接口 (Container Network Interface) 的缩写。它是一套标准的规范和接口，用于配置 Linux 容器的网络CNI 本身只是一套规范。实际执行网络操作的是 CNI 插件（如 bridge、loopback、host-local、Calico、Flannel 等）。这些插件通常安装在 /opt/cni/bin 目录下

### 3. 安装nerdctl
```bash
curl -LO https://github.com/containerd/nerdctl/releases/download/v1.2.1/nerdctl-1.2.1-linux-amd64.tar.gz
tar xzvf nerdctl-1.2.1-linux-amd64.tar.gz
mv nerdctl /usr/local/bin
nerdctl -n k8s.io ps 
```

nerdctl 是一个 兼容 Docker CLI 的命令行工具（CLI） 它提供了像 docker run、docker ps、docker network 等用户熟悉的操作界面，但底层是与 containerd 交互，弥补了 containerd 自身 CLI (ctr) 在用户体验上的不足。

### 4. 安装kubernets
```bash
# 设置源
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

# 安装k8s组件，master、node节点都进行安装
export K8sVersion=1.25.8
sudo yum install -y kubelet-$K8sVersion
sudo yum install -y kubeadm-$K8sVersion
sudo yum install -y kubectl-$K8sVersion
sudo systemctl enable --now kubelet

# 在master节点执行，安装成功后会打印节点加入集群的命令，其中包含了token信息，保存下来在node节点执行
kubeadm init --kubernetes-version=v1.25.8 --image-repository=registry.aliyuncs.com/google_containers --pod-network-cidr=172.16.0.0/16

# 在node节点执行
kubeadm join xxx.xxx.xxx.xxx:xxxx --token xxxxxxxxx \
        --discovery-token-ca-cert-hash sha256:xxxxxxx

# 在主节点配置 kubeconfig 文件，以便用 kubectl 更方便管理集群

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
kubectl get no
```

### 5. 安装 Calico
```bash
wget https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/calico.yaml
sed -i 's?docker.io?docker.m.daocloud.io?g' calico.yaml
kubectl apply -f calico.yaml
# 查看calico状态,默认安装在kube-system命名空间下
kubectl get po -n kube-system -w 
```

### 6. 安装默认存储 SCI
```bash
wget https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.24/deploy/local-path-storage.yaml
sed -i "s/image: rancher/image: docker.m.daocloud.io\/rancher/g" local-path-storage.yaml 
sed -i "s/image: busybox/image: docker.m.daocloud.io\/busybox/g" local-path-storage.yaml
kubectl apply -f local-path-storage.yaml
kubectl get po -n local-path-storage -w 

# 把 local-path 设置为默认 SC
kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
kubectl get sc

```
参考： https://github.com/rancher/local-path-provisioner

### 7. 安装daocloud

```bash
# 安装基础依赖
curl -LO https://proxy-qiniu-download-public.daocloud.io/DaoCloud_Enterprise/dce5/install_prerequisite.sh
bash install_prerequisite.sh online community 
# 下载 dce5-installer
export VERSION=v0.29.0
curl -Lo ./dce5-installer https://proxy-qiniu-download-public.daocloud.io/DaoCloud_Enterprise/dce5/dce5-installer-$VERSION
chmod +x ./dce5-installer
# 执行安装
./dce5-installer install-app -z 
```



## Q&A
### 1. 初始化master时coredns异常 

```bash
[config/images] Pulled k8s-gcr.m.daocloud.io/etcd:3.5.6-0
failed to pull image "k8s-gcr.m.daocloud.io/coredns:v1.9.3": output: E1013 10:28:46.518723    4656 remote_image.go:171] "PullImage from image service failed" err="rpc error: code = Unknown desc = failed to pull and unpack image \"k8s-gcr.m.daocloud.io/coredns:v1.9.3\": failed to resolve reference \"k8s-gcr.m.daocloud.io/coredns:v1.9.3\": unexpected status from HEAD request to https://k8s-gcr.m.daocloud.io/v2/coredns/manifests/v1.9.3: 403 Forbidden" image="k8s-gcr.m.daocloud.io/coredns:v1.9.3"
time="2025-10-13T10:28:46+08:00" level=fatal msg="pulling image: rpc error: code = Unknown desc = failed to pull and unpack image \"k8s-gcr.m.daocloud.io/coredns:v1.9.3\": failed to resolve reference \"k8s-gcr.m.daocloud.io/coredns:v1.9.3\": unexpected status from HEAD request to https://k8s-gcr.m.daocloud.io/v2/coredns/manifests/v1.9.3: 403 Forbidden"
, error: exit status 1
To see the stack trace of this error execute with --v=5 or higher
[root@k8s-master01 ~]# kubeadmin config image list
bash: kubeadmin: command not found...
[root@k8s-master01 ~]# kubeadm config image list
invalid subcommand: "image"
To see the stack trace of this error execute with --v=5 or higher
```
在初始化master节点时coredns遇到异常，参照[k8s-集群初始化kubeadm init踩的坑和解决方法](https://juejin.cn/post/7291570449334812735)使用阿里云的镜像解决。
关于coredns init异常的问题，参考
https://github.com/DaoCloud/public-image-mirror/issues/36080#issuecomment-2568590033
### 2. 在1.8节中安装的组件是什么？
   
| 组件名称 | 作用类别 | 详细作用描述 | 容器化（Docker/K8s）关联 |
| :--- | :--- | :--- | :--- |
| **yum-utils** | 软件包管理工具 | 提供 YUM 实用工具，最常用的是 `yum-config-manager`，用于 **管理和配置外部软件仓库**。 | 用于在安装 Docker 等非官方源软件包前，添加其官方软件源。 |
| **device-mapper-persistent-data** | 存储底层依赖 | 提供了 Linux 内核 **Device Mapper** 框架所需的持久化数据管理工具和库。 | 确保 Docker 等容器运行时可以利用 Device Mapper 的**精简配置和快照**功能（例如在旧版或特定配置下使用 `devicemapper` 存储驱动）。 |
| **lvm2** | 存储管理工具 | **逻辑卷管理器 (LVM)** 的命令行工具集。LVM 是 Device Mapper 的一个重要应用。 | 作为 Device Mapper 框架的底层依赖包，确保系统中具备完整的 **块级存储管理能力**，是容器运行时环境的通用要求之一。 |

### 3. 如何卸载DCE5
https://docs.daocloud.io/install/uninstall#_2

```bash
kubectl -n mcamel-system delete mysql mcamel-common-mysql-cluster
kubectl -n mcamel-system delete mysql mcamel-common-kpanda-mysql-cluster
kubectl -n mcamel-system delete elasticsearches mcamel-common-es-cluster-masters
kubectl -n mcamel-system delete redisfailover mcamel-common-redis-cluster
kubectl -n mcamel-system delete tenant mcamel-common-minio-cluster
kubectl -n ghippo-system delete gateway ghippo-gateway
kubectl -n istio-system delete requestauthentications ghippo
helm -n mcamel-system uninstall eck-operator mysql-operator redis-operator minio-operator
helm -n ghippo-system uninstall ghippo
helm -n insight-system uninstall insight-agent insight
helm -n ipavo-system uninstall ipavo
helm -n kpanda-system uninstall kpanda
helm -n istio-system uninstall istio-base istio-ingressgateway istiod
kubectl delete namespace mcamel-system ghippo-system insight-system ipavo-system kpanda-system istio-system
```
### 4. kubelet 状态异常未能正常启动

```bash
# 查看服务日志，通过journalctl命令查看日志，根据日志提示丁伟问题，我这边是未进行containerd配置初始化步骤
journalctl -xeu kubelet
```

### 5. 安装k8s一些命令集合


```bash

kubeadm config print init-defaults > init.yaml

kubeadm config images pull --image-repository k8s-gcr.m.daocloud.io --kubernetes-version=v1.25.8

kubeadm config images pull --config init.yaml --kubernetes-version=v1.25.8

kubeadm config images list

kubeadm init --kubernetes-version=v1.25.8 --image-repository=registry.aliyuncs.com/google_containers 

kubeadm init --config init.yaml

kubeadm reset 

kubectl -n kube-system get cm kubeadm-config -o yaml

kubeadm create token --print-join-command

kubectl logs kube-scheduler-k8s-master01 -n kube-system
```

### 6. 关于1.6节 设置br_netfilter内核参数并允许iptables进行桥接流量的一些知识点

/etc/modules-load.d systemd 目录是 Linux 系统中用于配置内核模块在系统启动时自动加载的关键位置。systemd服务读取和管理，静态配置系统启动时需要加载的内核模块列表

tee 命令的作用是 既将内容输出到屏幕，又将其写入指定文件

/etc/modules-load.d/kubernetes.conf 永久和临时加载 br_netfilter 模块允许 iptables 进行桥接流量过滤功能所必需的 。

modprobe：是 Linux 下用于加载内核模块的命令。

overlay：指的是 OverlayFS (Overlay File System) 模块。这是 Docker 和 Kubernetes 容器运行时 最常用的文件系统驱动之一，因为它高效、支持层叠（layering）

br_netfilter：指的是 Bridge Netfilter 模块。这个模块是实现 桥接流量（Bridged Traffic）可以被 Netfilter/iptables 框架处理 的关键

/etc/sysctl.d/ 目录：用于存放内核参数配置文件。文件名以 .conf 结尾的文件在系统启动时会被自动加载。

| 内核参数名称 | 默认值 | 描述 |
| ---- | ---- | ---- |
| net.bridge.bridge-nf-call-iptables	| 1|	开启 IPv4 桥接流量的 Netfilter/iptables 过滤。 这是为了让运行在 Linux 网桥 上的数据包（例如 Docker 或 K8s 容器之间、或容器进出主机的流量）能够被主机的 防火墙规则（iptables） 看到并处理。|
|net.bridge.bridge-nf-call-ip6tables	|1	|开启 IPv6 桥接流量的 ip6tables 过滤。 与上面类似，只是针对 IPv6 流量。|
|net.ipv4.ip_forward|	1|	开启 IPv4 数据包转发（路由功能）。 这意味着内核将作为路由器工作，允许将数据包从一个网络接口转发到另一个网络接口。这对于 K8s 和 Docker 至关重要，因为它允许： - 容器流量从内部虚拟网络转发到主机的物理网络（出站）。 - 外部流量通过 NAT 转发到容器（入站，如 NodePort 或 Ingress 访问）。|
