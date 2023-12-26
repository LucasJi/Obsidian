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
