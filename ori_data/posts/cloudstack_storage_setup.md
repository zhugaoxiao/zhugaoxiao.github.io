title: CloudStack 存储配置
tags: CloudStack
category: 云计算
---
CloudStack在实验室实践过程，存储配置部分。
<!--more-->
# 存储说明

CloudStack使用了两种网络存储。一种是主存储，主存储用于存放虚拟机硬盘文件。主存储也可使用本地存储，并非必需使用网络存储。二级存储用于存放虚拟机模板/快照/ISO文件，二级存储只能使用网络存储。

- Primary Storage

Primary Storage（主存储）和Cluster/Zone相关，它为运行于集群内的所有虚拟机提供磁盘存储卷。 你可以为一个区域或集群配置一个或多个主存储服务器（至少需要为一个集群配置一个主存储服务器）， 主存储一般靠近主机，以便提高性能。 Cloudstack负责把主存储分配给虚拟机。

主存储使用存储标签的概念，每个主存储可以和0个，1个或多个存储标签关联。

当某个虚拟机启动的时候，或某个数据磁盘第一个挂载到某个虚拟机的时候，存储标签就可以用于标识哪个主存储可以挂载到虚拟机。 主存储可以是静态的或动态的。

在静态情况下，管理员必须给Cloudstack预先配置一定数量的存储（例如一个SAN的一个Volumn），Cloudstack可以放很多Volumn到它的存储上。

在动态模式下，管理员可以给Cloudstack分配一个存储系统（例如一个SAN），Cloudstak通过和这个存储系统的插件配合，可以实现动态建立存储卷。

CloudStack被设计为可以支持广泛的商业级和企业级存储设备。持所有标准的iSCSI和NFS Server，如果所选的hypervisor支持，也可以使用本地存储。所支持的虚拟磁盘类型取决于不同的hypervisor。

| 存储类型 | XenServer| 	vSphere 	| KVM|
| --| --|-- |-- |
| NFS支持 | 支持 | 支持|
| iSCSI | 支持 通过VMFS支持 | 通过集群文件系统支持|
| Fiber Channel | 通过Pre-existing SR支持 | 支持|  通过集群文件系统支持|
| Local Disk | 支持 | 支持|  支持|

> 针对KVM使用集群逻辑卷管理器(CLVM)，不受CloudStack官方支持。

- Secondary Storage

Secondary storage（二级存储）主要保存以下数据：
- Templates（模板）
OS 镜像，可用于引导虚拟机，可以包含其它配置信息，如安装好的软件。
- ISO镜像
包含数据或引导媒介的磁盘镜像，就是可以刻盘启动的iso文件。
- 磁盘存储卷的快照
保存虚拟机数据的拷贝，可用于数据恢复或建立新的模板。

二级存储的内容对对其范围内的所有主机都是可用的，范围一般指一个区域（Zone）或一个区划（Region）。
二级存储一般配置为对一个Zone可用的NFS形式，为了使副存储对所有云的主机可用，可以引入对象存储，这样就不必在Zone间复制模板和快照。

Cloudstack提供了Swift和Amazon S3的插件。当使用以上两种存储之一，你可以首先为整个Cloudstack配置对象存储，然后为每个Zone建立NFS中间存储，中间存储的作用是为所有模板和其它副存储数据存储到对象存储系统时提供临时存储。

> 注意
存储服务器应该有大量磁盘，理想情况下应该配备硬件RAID控制器。硬件RAID控制器支持热插拔功能并独立于操作系统之外，这样你就可以替换损坏的磁盘而不影响正在运行的操作系统。

![store](/images/store.png)

# 存储配置

## 基于本地硬盘和DAS的NFS服务

在存储服务器上安装RHEL/CentOS 发行版。如果根卷的大小超过2TB，创建一个小点的启动卷用于安装RHEL/CentOS，根分区保留20GB就足够了。
系统安装好以后，创建一个名为/export的目录。可以在根分区下创建，或者挂载一个较大的磁盘卷。
如果你在一台主机上有超过16TB的存储空间， 则创建多个EXT3文件系统和多个NFS输出，单个EXT3文件系统不能超过16TB。
### 1. 安装NFS
```shell
# yum install nfs-utils -y
```
### 2. 创建存储目录
```shell
# mkdir -p /export/primary
# mkdir -p /export/secondary
```
### 3. 配置NFS
```shell
# echo "/export *(rw,async,no_root_squash,no_subtree_check)" >/etc/exports
# exportfs -a
```

强烈建议通过指定子网掩码来限制NFS访问权限，（例如“192.168.20.0/24”）。只允许规定的集群才能访问，避免非集群成员访问。
且管理网络和存储网络必须能够访问；如果2个网络相同，那么一个CIDR就足够了。如果你是单独的存储网络则必须提供独立的CIDR或者一个足够大的CIDR可以包含它们2个。

移除async异步标记. 标志允许NFS服务器在提交并写入磁盘操作前就先返回相应，来提高性能。请在关键生产部署中移除异步标志。

编辑/etc/exports文件，设置存储的路径,Export /export 目录。
```shell
#
echo "LOCKD_TCPPORT=32803
LOCKD_UDPPORT=32769
MOUNTD_PORT=892
RQUOTAD_PORT=875
STATD_PORT=662
STATD_OUTGOING_PORT=2020
RPCNFSDARGS="-N	4"
#	对于KVM集群是必须的,	 否则存储异常导致系统虚机无法启动
" >>/etc/sysconfig/nfs
```

修改 /etc/sysconfig/nfs，将其中的端口号全部打开。
### 4. 配置防火墙
```shell
#
iptables -A INPUT -s 0.0.0.0/0 -m state --state NEW -p udp --dport 111 -j ACCEPT
iptables -A INPUT -s 0.0.0.0/0 -m state --state NEW -p tcp --dport 111 -j ACCEPT
iptables -A INPUT -s 0.0.0.0/0 -m state --state NEW -p tcp --dport 2049 -j ACCEPT
iptables -A INPUT -s 0.0.0.0/0 -m state --state NEW -p tcp --dport 32803 -j ACCEPT
iptables -A INPUT -s 0.0.0.0/0 -m state --state NEW -p udp --dport 32769 -j ACCEPT
iptables -A INPUT -s 0.0.0.0/0 -m state --state NEW -p tcp --dport 892 -j ACCEPT
iptables -A INPUT -s 0.0.0.0/0 -m state --state NEW -p udp --dport 892 -j ACCEPT
iptables -A INPUT -s 0.0.0.0/0 -m state --state NEW -p tcp --dport 875 -j ACCEPT
iptables -A INPUT -s 0.0.0.0/0 -m state --state NEW -p udp --dport 875 -j ACCEPT
iptables -A INPUT -s 0.0.0.0/0 -m state --state NEW -p tcp --dport 662 -j ACCEPT
iptables -A INPUT -s 0.0.0.0/0 -m state --state NEW -p udp --dport 662 -j ACCEPT
# iptables-save > /etc/sysconfig/iptables
# service iptables status
```

### 5. 启动NFS服务
```shell
# service nfs start
# service rpcbind start
```

### 6. 设置服务为自动启动
```shell
# chkconfig nfs on
# chkconfig rpcbind on
```

## 基于iSCSI的NFS服务

使用以下步骤创建基于iSCSI卷的NFS服务。这些步骤适用于RHEL/CentOS 5发行版。

### 1. 安装iscsiadm

```bash
# yum install iscsi-initiator-utils
# service iscsi start
# chkconfig --add iscsi
# chkconfig iscsi on
```

### 2. 发现ISCSI target

```bash
# iscsiadm -m discovery -t st -p <iSCSI Server IP address>:3260
// 例如
# iscsiadm -m discovery -t st -p 172.23.10.240:3260 172.23.10.240:3260,1 iqn.2001-05.com.equallogic:0-8a0906-83bcb3401-16e0002fd0a46f3d-rhel5-test
```
### 3. 登录

```bash
# iscsiadm -m node -T <Complete Target Name> -l -p <Group IP>:3260
//例如
# iscsiadm -m node -l -T iqn.2001-05.com.equallogic:83bcb3401-16e0002fd0a46f3d-rhel5-test -p 172.23.10.240:3260
```
### 4. 发现SCSI磁盘

```bash
# iscsiadm -m session -P3 | grep Attached
Attached scsi disk sdb State: running
```
### 5. 格式化磁盘为ext3类型并挂载卷

```bash
# mkfs.ext4 /dev/sdb
# mkdir -p /export
# mount /dev/sdb /export
```
### 6. 添加到/etc/fstab

在启动时能被挂载。

```bash
/dev/sdb /export ext3       _netdev 0 0
```
现在可以建立/export共享目录，其他步骤参考上一节。

## 文件服务器

### 1. 安装httpd服务
```shell
# yum install httpd -y
# service httpd start
```
### 2. 上传系统镜像

将系统模版、操作系统iso镜像上传至/var/www/html目录。
