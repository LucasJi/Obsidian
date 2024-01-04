## 简介

由于国内云服务商的优惠通常仅限于一台服务器，因此想低成本地获取多台服务器，只能向多个云服务商购买服务器。不同服务商之间地服务器只能通过公网通信，这会导致使用常用方法搭建的k3s集群的节点之间无法正常通信。为了解决这个问题，我们需要使用 WireGuard 来组网。

## 安装 WireGuard

在搭建跨云的 k3s 集群之前，需要将 WireGuard 安装在集群中的每台服务器上。WireGuard 对内核有要求，需要升级到`5.15.2-1.el7.elrepo.x86_64`以上。

本人使用两台分别来自腾讯云和阿里云的轻量云服务器组建 k3s 集群。来自腾讯云的服务器将作为 master 节点，来自阿里云的服务器将作为 node-1 节点。两台服务器均安装 `CentOS7.6`。

分别在master和node-1节点执行如下命令开启IP地址转发

``` bash
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
echo "net.ipv4.conf.all.proxy_arp = 1" >> /etc/sysctl.conf
sysctl -p /etc/sysctl.conf
```

修改两台服务器的名称

```bash
# 腾讯云执行
hostnamectl set-hostname k3s-master

# 阿里云执行
hostnamectl set-hostname k3s-node-1
```

### 修改 iptables 以允许 NAT

对两台服务器都进行如下设置，添加 iptables 规则，允许本机的 NAT 转换。

```bash
iptables -A INPUT -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
iptables -A FORWARD -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i wg0 -o wg0 -m conntrack --ctstate NEW -j ACCEPT
iptables -t nat -A POSTROUTING -s 192.168.1.1/24 -o eth0 -j MASQUERADE
```

参数解释：

- `wg0`: 虚拟网卡的名称，两台服务器均使用 `wg0`
- `192.168.1.1/24`: 虚拟 ip 地址段，master 节点设置为 `192.168.2.1`，node-1 节点设置为 `192.168.2.2`
- `eth0`: 为服务器的物理网卡

### 升级内核

在所有节点都执行如下操作

#### 1. 载入公钥

```shell
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
```

#### 2. 升级安装 `elrepo`

```shell
rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-5.el7.elrepo.noarch.rpm
```

#### 3. 载入 `elrepo-kernel` 元数据

```shell
yum --disablerepo=\* --enablerepo=elrepo-kernel repolist
```

#### 4. 安装最新版本的内核

```shell
yum --disablerepo=\* --enablerepo=elrepo-kernel install kernel-ml.x86_64 -y
```

#### 5. 删除旧版本工具包

```
yum remove kernel-tools-libs.x86_64 kernel-tools.x86_64 -y
```

#### 6. 查看当前内核启动顺序

```
grub2-editenv list
```

#### 7. 查看内核插入顺序

```
grep "^menuentry" /boot/grub2/grub.cfg | cut -d "'" -f2
```

#### 8. 将步骤 6 中的最新内核设置为默认启动

```
grub2-set-default 'xxx'
```

#### 9. 重新创建内核配置

```
grub2-mkconfig -o /boot/grub2/grub.cfg
```

#### 10. 重启服务器

```
reboot
```

#### 11. 验证当前内核版本

```
uname -r
```

### 安装 WireGuard

在两台服务器中执行如下操作。由于内核升级之后已经包含 WireGuard 模块，因此我们只需要安装 `wireguard-tools` 包即可。

```shell
yum install epel-release https://www.elrepo.org/elrepo-release-7.el7.elrepo.noarch.rpm

yum install yum-plugin-elrepo kmod-wireguard wireguard-tools -y
```

### 配置 WireGuard

`wireguard-tools` 包提供了 `wg` 和 `wg-quick` 两种工具，我们可以使用它们分别进行手动和自动部署。

首先我们需要生成用于加密解密的秘钥。

#### 在 master 节点

执行如下操作在当前目录生成 `privatekey` 和 `publickey` 两个文件。`privatekey` 配置在本机而`publickey`配置在其他机器。

```
wg genkey | tee privatekey | wg pubkey > publickey
```

编写 `wg-quick` 配置文件

```
vim /etc/wireguard/wg0.conf
```

在配置文件中写入如下内容

```
[Interface]
PrivateKey = master 的 privatekey
Address = 192.168.2.1
ListenPort = 5418

[Peer]
PublicKey = node-1 的 publickey（node-1生成之后再改）
EndPoint = node-1 公网ip
AllowedIPs = 192.168.2.2/32
```

#### 在 node-1 节点

执行如下操作在当前目录生成 `privatekey` 和 `publickey` 两个文件。`privatekey` 配置在本机而`publickey`配置在其他机器。

```
wg genkey | tee privatekey | wg pubkey > publickey
```

编写 `wg-quick` 配置文件

```
vim /etc/wireguard/wg0.conf
```

在配置文件中写入如下内容

```
[Interface]
PrivateKey = node-1 的 privatekey
Address = 192.168.2.2
ListenPort = 5418

[Peer]
PublicKey = master 的 publickey
EndPoint = master 公网ip
AllowedIPs = 192.168.2.1/32
```

### 开放端口

到云厂商的防火墙（安全组）规则配置处开放 `5418` 端口，下行规则，协议为UDP

### 启动 WireGuard

所有节点执行 `wg-quick` 工具来创建虚拟网卡

```shell
wg-quick up wg0
```

上面命令中的 `wg0` 对应的是 `/etc/wireguard/wg0.conf` 这个配置文件，其自动创建的网卡设备，名字就是 `wg0`

创建好虚拟网卡之后就可以测试是否联通了。我们在 master 节点执行如下命令（`192.168.2.2` 是 node-1 节点的虚拟ip）

```
ping 192.168.2.2
```

如果之前配置都正确的话，会看到如下结果

```
[root@k3s-master ~]# ping 192.168.2.2
PING 192.168.2.2 (192.168.2.2) 56(84) bytes of data.
64 bytes from 192.168.2.2: icmp_seq=1 ttl=64 time=8.64 ms
64 bytes from 192.168.2.2: icmp_seq=2 ttl=64 time=8.58 ms
64 bytes from 192.168.2.2: icmp_seq=3 ttl=64 time=8.64 ms
^C
--- 192.168.2.2 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
rtt min/avg/max/mdev = 8.587/8.623/8.642/0.080 ms
```

从 node-1 节点 `ping` master 节点也能看到类似的结果

### 自动化配置

系统重启后，`WireGuard` 创建的网卡设备就会丢失，`WireGuard`提供了自动化的脚本来解决这件事，在两台服务器都执行如下操作

```
systemctl enable wg-quick@wg0
```

使用上述命令生成 systemd 守护脚本，开机会自动运行 up 指令

### 配置热重载

wg-quick 并未提供重载相关的指令，但是提供了 `strip` 指令，可以将 conf 文件转换为 wg 指令可以识别的格式

```
wg syncconf wg0 <(wg-quick strip wg0)
```

## 安装 k3s 集群

### master 节点安装

配置参数

```
export MASTER_EXTERNAL_IP=master 节点的公网ip
export MASTER_VIRTUAL_IP=192.168.2.1
```

安装 k3s

```bash
curl -sfL http://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn sh  -s -  --node-external-ip $MASTER_EXTERNAL_IP --advertise-address $MASTER_EXTERNAL_IP --node-ip $MASTER_VIRTUAL_IP --flannel-iface wg0
```

参数说明：

- `--node-external-ip`:  当前节点的外部 IP 即公网 IP
- `--advertise-address`:  用于设置 kubectl 工具以及子节点进行通讯使用的地址，可以是 IP，也可以是域名，在创建 apiserver 证书时会将此设置到有效域中。使用公网 IP
- `--node-ip`：上文中约定的的虚拟 IP
- `--flannel-iface wg0`:  wg0 是 WireGuard 创建的网卡设备。因为使用虚拟局域网来进行节点间的通信，所以这里需要指定为 wg0。

在主节点执行上述命令后，一分钟不到就可以看到脚本提示安装完成。通过命令查看下主控端的运行情况

```
systemctl status k3s
```

如果运行正常，那么就看看容器的运行状态是否正常

```
kubectl get pods -A
```

### node-1 节点安装

```
export MASTER_VIRTUAL_IP=192.168.2.1
export NODE_EXTERNAL_IP=node-1 节点公网 IP
export NODE_VIRTUAL_IP=192.168.2.2
export K3S_TOKEN=
```

`K3S_TOKEN` 的值在 master 节点执行 `/var/lib/rancher/k3s/server/node-token` 后得到

安装 k3s-agent

```bash
curl -sfL http://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn K3S_URL=https://$MASTER_VIRTUAL_IP:6443 K3S_TOKEN=$K3S_TOKEN sh -s - --node-external-ip $NODE_EXTERNAL_IP --node-ip $NODE_VIRTUAL_IP --flannel-iface wg0
```

待脚本执行完成之后，执行以下命令查看服务状态

```
systemctl status k3s-agent
```

在 master 节点执行 `kubectl get nodes -o wide` 可以看到 master 和 node-1 节点

```
[root@k3s-master ~]# kubectl get nodes -o wide
NAME         STATUS   ROLES                  AGE   VERSION        INTERNAL-IP   EXTERNAL-IP       OS-IMAGE                KERNEL-VERSION              CONTAINER-RUNTIME
k3s-master   Ready    control-plane,master   37d   v1.27.7+k3s2   192.168.2.1   x.x.x.x      CentOS Linux 7 (Core)   6.6.2-1.el7.elrepo.x86_64   containerd://1.7.7-k3s1.27
k3s-node-1   Ready    <none>                 37d   v1.27.7+k3s2   192.168.2.2   x.x.x.x   CentOS Linux 7 (Core)   6.6.2-1.el7.elrepo.x86_64   containerd://1.7.7-k3s1.27
```

至此，整个跨云服务商 k3s 集群搭建完毕~
