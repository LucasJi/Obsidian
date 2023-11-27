本篇博文介绍如何使用两台轻量云服务器搭建一个k3s集群(包含一个master和一个agent).

## 准备工作

### 轻量云服务器配置

- 一台2核4g的轻量云服务器作为master节点
- 一台2核2g的轻量云服务器作为agent节点

两台云服务器的操作系统均为centOS7.6

### 服务器设置

1. 关闭防火墙
```
systemctl disable firewalld --now
```
2. 设置selinux(需要联网)
```
yum install -y container-selinux selinux-policy-base
yum install -y https://rpm.rancher.io/k3s/latest/common/centos/7/noarch/k3s-selinux-0.2-1.el7_8.noarch.rpm
```

## master节点安装

### 1. 修改服务器主机名

在安装master节点前, 为了使节点名称更易于分辨, 我们可以利用`hostnamectl`命令修改服务器主机名:

```
hostnamectl set-hostname k3s-master
```

### 2. 安装k3s

为了方便起见, 本文选用官方[快速入门](https://docs.rancher.cn/docs/k3s/quick-start/_index#%E5%AE%89%E8%A3%85%E8%84%9A%E6%9C%AC)中提供的脚本进行安装

```
curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn sh -
```

上述脚本会使用将云服务器内网ip作为api server的ip地址, 导致后续agent节点无法和master节点通信, 因此我们需要增加额外的参数来使用服务器外网地址`INSTALL_K3S_EXEC="--node-external-ip x.x.x.x --advertise-address x.x.x.x"`. 完整的脚本:

```
curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn INSTALL_K3S_EXEC="--node-external-ip x.x.x.x --advertise-address x.x.x.x" sh -
```

### 3. 查看安装结果

等待脚本执行完成之后, 运行`kubectl get node`命令会看到如下信息代表master节点安装成功并开始运行:

```
[root@VM-4-14-centos ~]# kubectl get node
NAME         STATUS   ROLES                  AGE   VERSION
k8s-master   Ready    control-plane,master   6s    v1.27.7+k3s2
```

### 3. 防火墙

为确保agent节点能和master节点顺利通讯, master节点所在服务器的6443端口需要打开

## agent节点安装

### 1. 修改服务器主机名

在安装agent节点前, 为了使节点名称更易于分辨, 我们可以利用`hostnamectl`命令修改服务器主机名:

```
hostnamectl set-hostname k3s-node-1
```

### 2. 安装k3s

为了方便起见, 本文选用官方[快速入门](https://docs.rancher.cn/docs/k3s/quick-start/_index#%E5%AE%89%E8%A3%85%E8%84%9A%E6%9C%AC)中提供的脚本进行安装

```
curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | INSTALL_K3S_EXEC="--node-external-ip x.x.x.x" INSTALL_K3S_MIRROR=cn K3S_URL=https://myserver:6443 K3S_TOKEN=mynodetoken sh -
```

上述脚本中的`K3S_URL`的值就是master节点的服务器地址`https://x.x.x.x:6443`. `K3S_TOKEN`的值可以通过在master节点所在服务器运行`cat /var/lib/rancher/k3s/server/node-token`命令获取

### 3. 查看安装结果

等待脚本执行完成之后, 在master节点运行`kubectl get node`命令会看到如下信息代表agent节点安装成功并连接到master节点

```
[root@VM-4-14-centos ~]# kubectl get node
NAME           STATUS   ROLES                  AGE   VERSION
k8s-master     Ready    control-plane,master   19m   v1.27.7+k3s2
k8s-worker-1   Ready    <none>                 7s    v1.27.7+k3s2
```

## 卸载k3s

请参考官方文档[卸载k3s](https://docs.rancher.cn/docs/k3s/installation/uninstall/_index/)
