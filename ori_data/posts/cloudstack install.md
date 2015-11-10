title: CloudStack 安装和配置
tags: CloudStack
category: 云计算
---
CloudStack在实验室实践过程，，安装配置部分。

<!--more-->
# 环境要求

## 管理节点OS版本及软件版本
* RHEL versions 6.3, 6.5, 6.6 and 7.0
* CentOS versions 6.6, 7.0
* Ubuntu 14.04 LTS
* Java 1.7
* MySQL 5.6 (RHEL 7)
* MySQL 5.1 (RHEL 6.x)

## 虚拟化版本
* LXC Host Containers on RHEL 7
* Windows Server 2012 R2 (with Hyper-V Role enabled)
* Hyper-V 2012 R2
* CentOS 6.2+ with KVM
* Red Hat Enterprise Linux 6.2 with KVM
* XenServer versions 6.1, 6.2 SP1 and 6.5 with latest hotfixes
* VMware versions 5.0 Update 3a, 5.1 Update 2a, and 5.5 Update 2
* Bare metal hosts are supported, which have no hypervisor.
These hosts can run the following operating systems:
* RHEL or CentOS, v6.2 or 6.3
     > Note：Use libvirt version 0.9.10 for CentOS 6.3

* Fedora 17
* Ubuntu 12.04

## 外部设备

* Netscaler VPX and MPX versions 9.3, 10.1e and 10.5
* Netscaler SDX version 9.3, 10.1e and 10.5
* SRX (Model srx100b) versions 10.3 to 10.4 R7.5
* F5 11.X
* Force 10 Switch version S4810 for Baremetal Advanced Networks

## 浏览器

* Internet Explorer versions 10 and 11
* Firefox version 31 or later
* Google Chrome version 36.0.1985
* Safari 6+

# 环境规划

## 设备配置说明

### 路由器

路由器连接公网和内部网络，路由器内部ip地址为192.168.6.1 ，与三层交换机g0/1连接。

|设备   | 物理接口 |  地址           | 功能             |备注    |
|------ | :-----  | :----:          |  :----:          |:----:  |
| 路由器 | e0/0    |  172.28.23.1    |  连接internet     |   内部地址，非公网    |
|       | e0/1    |  192.168.0.1/24 | 内部接口，连接交换机|     |

### 交换机

三层交换机负责物理网络vlan划分，开启vlan间路由器，交换机的g0/1属于vlan 2，和路由器内网接口连接，vlan 2 ip地址为：192.168.6.254
（二层交换机也可以，但是需要在路由器上做单臂路由）。

|设备         | vlan    |  vlan  ip 网关     | 功能            |备注    |
|------       | :-----:| :----:             |  :----:        |:----:  |
| 交换机      | 2      |  192.168.0.254/24   | 连接出口路由器  |        |
|             | 20     |  192.168.20.1/24    | Storage&Manage Network | 计算节点出口网关 |
|             | 30     |  192.168.30.1/24    | Public Network | 公共网络，可以配置多个   |
|             | 300-999|                     | Guest Network  | 来宾网络       |

|接口序号| vlan | 接口模式 | 连接设备   |备注            |
|------  |----- | :----:  |  :----:    |:----:           |
| g0/24  | 2    |  Access | 路由器g0/1  |  路由器内网接口 |
| g0/1   | 1    |  Trunk  | Pod0 g0/24  | Pod0交换机（管理节点所在机柜）|
| g0/2   | 1    |  Trunk  | Pod1 g0/24  | Pod1交换机 |
| g0/3   | 1    |  Trunk  | Pod2 g0/24  | Pod2交换机 |

### 流量分类规划

|流量类型 |	VLAN |	        	CIDR  |		网关       |		起始IP    |		结束IP |备注|
|------   |------   | :-----       | :----:       |  :----:     |:----:    | :-----   |
|Public Network|30|	192.168.30.0/24 |	192.168.30.1 |	192.168.30.10	|192.168.20.250 |公共流量
|Manage Network|20|	192.168.21.0/24	|	192.168.21.1 |	192.168.21.200|	192.168.11.229|	提供点，内部系统用的IP，如系统vm |
|Guest Network |300-999	|	10.1.1.0/24	 |	10.1.1.1	 |	10.1.1.10	 |  10.1.1.250 |来宾流量，用户vm使用
|Storage Network|	20	  |	192.168.20.0/24|192.168.20.1|	192.168.20.170|192.168.20.199| 存储流量 |

- 高级区域配置成功后，客户vm将获得一个10.1.1.0网段的私有ip地址；
- 系统将建立一个虚拟路由器Vrouter，这个虚拟路由器将作为用户vm 10.1.2.0 guest网段网关；
- 虚拟路由器还将获得一个public的网段的ip地址；
- vm将通过Vrouter nat功能实现外部通信和对外提供服务。

Guest vm(10.1.1.x)→(10.1.1.1)Vrouter-nat(192.168.30.x) → (192.168.30.1)三层交换机(192.168.0.254）→
（192.168.0.1）路由器(nat)→wan

## 主机规划

物理主机和管理服务器属于vlan 11 , 交换机连接计算节点的端口配置为trunk模式，允许所有vlan通过，本征vlan 为11；nfs为存储设备，连接交换机的接口配置为access模式，属于vlan 12。

| 机柜 | 设备名称   | IP地址      |  	网关       |	vlan|用途    |系统             | 备注     |
| --   | -------- | :-----       | :----:       |:----:|:----:   | :----:         |:----:    |
| Pod0 |cs        |192.168.20.100|192.168.20.254|	20	| 管理节点| centos6.6 x64  |  8核，8G，120G |
| Pod0 |nfs       |192.168.20.101|192.168.20.254|	20	|	二级存储 |centos6.6 x64  | 	8核，8G，500G|
| Pod0 |nfs1      |192.168.20.102|192.168.20.254|	20	|	机柜1主存储|centos6.6 x64 | 8核，8G，1024G|
| Pod0 |nfs2      |192.168.20.103|192.168.20.254|	20	|	机柜2主存储|centos6.6 x64 | 8核，8G，1024G|
| Pod0 |nfs3      |192.168.20.104|192.168.20.254|	20	|	机柜2主存储|centos6.6 x64 | 8核，,8G，1024G|
| Pod0 |vcenter   |192.168.20.105|192.168.20.254|	20	|	vcenter   | VMWARE vCenter5.5U2 | 	16核,8G |
| Pod1 |nr1r01n05 |192.168.20.5  |192.168.20.254|	20	|	计算节点  | VMWARE esxi 5.5U2|  8核,8G|
| Pod1 |nr1r01n19 |192.168.20.19 |192.168.20.254|	20	|	计算节    | VMWARE esxi 5.5U2|	 8核,8G|
| Pod2 |nr1r02n09 |192.168.20.20 |192.168.20.254|	20	|	计算节点  | VMWARE esxi 5.5U2| 	12核,8G|
| Pod2 |nr1r02n28 |192.168.20.27 |192.168.20.254|	20	|	计算节点  | VMWARE esxi 5.5U2| 	12核,8G|
| Pod3 |nr1r03n05 |192.168.20.30 |192.168.20.254|	20	|	计算节点 	| VMWARE esxi 5.5U2| 	12核,64G|
| Pod3 |nr1r03n10 |192.168.20.35 |192.168.20.254|	20	|	计算节点  | VMWARE esxi 5.5U2| 	12核,64G|
| Pod4 |nr1r04n05 |192.168.20.40 |192.168.20.254|	20	|	计算节点  | VMWARE esxi 5.5U2| 	80核,128G|
| Pod4 |nr1r04n14 |192.168.20.49 |192.168.20.254|	20	|	计算节点  | VMWARE esxi 5.5U2| 	80核,128G|
| Pod5 |nr1r05n05 |192.168.20.50 |192.168.20.254|	20	|	计算节点  | VMWARE esxi 5.5U2| 	32核,64G|
| Pod5 |nr1r05n10 |192.168.20.55 |192.168.20.254|	20	|	计算节点  | VMWARE esxi 5.5U2| 	32核,64G|

## Basic Zone
Basic Zone需要配置3种网络流量类型，分别为Management Network、Guest Network、Storage Network，具体规划信息如下表：

|流量类型 |	VLAN |	        	CIDR  |		网关       |		起始IP    |		结束IP |备注|
|------  |------  | :-----| :----:  |  :----:          |:----:            | :-----   |
|Public Network |	 30  |	192.168.30.0/24  |	192.168.30.1 |	192.168.30.10	 |192.168.30.250 |公共流量, |
|Manage Network|	 20	 |	192.168.20.0/24	 |	192.168.20.1 |	192.168.20.200	 |	192.168.20.229	 |	提供点，内部系统用的IP，如系统vm |
|Guest Network  |	300-999|	10.1.1.0/24|		10.1.1.1	 |	10.1.1.10	 |  10.1.1.250 |来宾流量，用户vm使用, |
|Storage Network|	 20	 |	192.168.20.0/24	 |	192.168.20.1 |	192.168.20.230	 |	192.168.20.249	 | 存储流量 |

## Advanced Zone

|流量类型 |	VLAN |	        	CIDR  |		网关       |		起始IP    |		结束IP |备注|
|------  |------  | :-----| :----:  |  :----:          |:----:            | :-----   |
|Public Network |	30   |		192.168.30.0/24 |		192.168.30.1 |	192.168.30.10	 |192.168.30.250 |公共流量，后期可继续添加
|Manage Network|	20	 |	192.168.20.0/24	 |	192.168.10.1 |		192.168.10.120	 |	192.168.10.169	 |	提供点，内部系统用的IP，如系统vm |
|Guest Network |	300-999	|	10.1.1.0/24	 |	10.1.1.1	 |	10.1.1.10	 |  10.1.1.250 |来宾流量，用户vm使用
|Storage Network|	20	   |	192.168.20.0/24	 |	192.168.20.1	 |	192.168.20.170	 |	192.168.20.199	 | 存储流量，后期可继续添加 |

连接示意图如下图所示：
![Advanced-Zone](/images/Advanced-Zone.png)

# 部署CloudStack

## 配置系统相关服务

### 1. 配置IP
```shell
# echo "IPADDR=192.168.3.10
NETMASK=255.255.255.0
GATEWAY=192.168.3.254" >> /etc/sysconfig/network-scripts/ifcfg-eth0
# sed -i 's/dhcp/static' /etc/sysconfig/network-script/ifcfg-eth0
# sed -i 's/ONBOOT=no/ONBOOT=yes' /etc/sysconfig/network-script/ifcfg-eth0
```
### 2. 配置主机名
```shell
# echo "cs"> /etc/sysconfig/network
# hostname -F /etc/sysconfig/network
# echo "192.168.3.10   cs cs.cloud.com">> /etc/hosts
# hostname --fqdn
//检查配置是否有效
```
### 3. 关闭 selinux
```shell
# getenforce
//查看当前 selinux 状态
# setenforce 0
//临时设置 selinux 状态
# sed -i 's/enforcing/disabled/' /etc/selinux/config
//修改 selinux 配置文件，重启永久禁用
```
### 4. 配置系统的本地 yum 源
```shell
# yum clean all & yum makecache
```
### 5. 配置 ntp 服务器
```shell
# yum install ntp  -y
# vi /etc/ntp.conf
//编辑 ntp 配置文件，将服务器替换成可用对时服务器。
# service ntpd restart
# chkconfig ntpd on
//重启 ntp 服务，并且设置其开机启动
```
### 6. 确认java版本

管理节点不能安装JDK6或者其他较低的版本，如有安装，先卸载该JDK，再进行进行下面的步骤。（cs4.4开始改用JDK7）
```shell
# java -version
```

## 安装CloudStack

### 1. 配置cloudstack的yum源
```shell
# echo "[cloudstack-source]
name=cloudstack
baseurl=http://192.178.102.249/cloudstack/centos/4.5/
enabled=1
gpgcheck=0" > /etc/yum.repos.d/cloudstack.repo
```
实验室已将yum源同步到本地，不同版本配置不同，具体配置时以实际版本为准。
### 2. 安装cloudstack
```shell
# yum install -y cloudstack-management
```
### 3. 安装cloudstack-agent
```shell
# yum install -y cloudstack-agent
```
仅在KVM主机节点安装。

## 安装数据库

### 1. 安装mysql
```shell
# yum install mysql-server -y
```
CloudStack使用mysql管理数据，但安装cloudstack-management时没有包含mysql，需要手动安装，并导入数据。数据库可以被安装到其它机器上。
  注意：允许远程mysql连接，方便以后查找问题

### 2. 修改mysql配置
```shell
# echo "[mysqld]
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock
user=mysql
# Disabling symbolic-links is recommended to prevent assorted security risks
symbolic-links=0
innodb_rollback_on_timeout=1
innodb_lock_wait_timeout=600
max_connections=350
log-bin=mysql-bin
binlog-format = 'ROW'
[mysqld_safe]
log-error=/var/log/mysqld.log
pid-file=/var/run/mysqld/mysqld.pid" >/etc/my.cnf
```
max_connections 的参数应设置 350 乘以你准备部署的管
理节点的数量。这里假定只安装一个管理节点。

### 3. 修改mysql安全
```shell
# service mysqld start
# chkconfig mysqld on
# mysql_secure_installation
# service mysqld restart
Stopping mysqld:                                           [  OK  ]
Starting mysqld:                                           [  OK  ]
# /sbin/iptables -I INPUT -p tcp --dport 3306 -j ACCEPT
# /etc/rc.d/init.d/iptables save
```
缺省安装的 mysql 安全级别比较低，需要手工设置 mysql 下密码、禁用远程访问，删除无用账户及测试数据库。

### 4. 导入数据
```shell
# cloudstack-setup-databases cloud:111111@localhost --deploy-as=root:111111
```
数据库准备好后，需导入 CloudStack 的表及基础数据，这样云平台才能正常使用。如果没有意外的话，最后会输出 CloudStack has successfully initialized database
字样，表示数据库已经准备好了。

## 启动CloudStack

### 1. 配置管理服务器服务并启动服务，检查服务状态。
```shell
# cloudstack-setup-management
# service cloudstack-management status
# tail -n 200 -f /var/log/cloudstack/management/catalina.out
```

## 配置系统模版

### 1. 确定模版版本

  CloudStack使用一组系统虚机来提供访问虚机控台，各种网络服务和管理存储的功能。当你引导云的时候，该步骤会获取这些准备用于部署的系统镜像。现在我们要从刚刚挂载的共享存储上面下载虚机模板并部署它们。管理服务器上有一个脚本来操作这些系统虚机镜像。这里的模版已装好的数据库中看到系统使用的模版地址，下载对应版本即可：
```SQL
SQL> SELECT NAME,URL FROM vm_template;
```
直接下载即可：
```shell
# wget http://download.cloud.com/templates/4.5/systemvm64template-4.5-vmware.ova
```
### 2. 挂载nfs
```shell
# mount -t nfs 192.168.50.10:/export/secondary /mnt
```
在管理服务器上挂载二级存储.
### 3. 导入系统模版
```shell
# /usr/share/cloudstack-common/scripts/storage/secondary/cloud-install-sys-tmplt \
-m /mnt \
-u http://192.178.102.249/cloudstack/systemvm64template-4.4.1-7-vmware.ova \
-h vmware \
-F
```

## 数据库复制（可选）
CloudStack支持MySQL节点间的数据库复制。通过标准的MySQL复制功能来实现。这样做可能希望防止MySQL服务器或者存储损坏。MySQL复制使用master/slave的模型。master节点是直接为管理服务器所使用。slave节点为备用，接收来自master节点的所有写操作，并将它应用于本地冗余数据库。以下是实施数据库复制的步骤。

> 注： 创建复制并不等同于备份策略，你需要另外建立有别于复制的MySQL数据的备份机制。

### 1.配置主节点
确保这是一个全新安装且没有数据的master数据库节点。
编辑master数据库的my.cnf，在[mysqld]的datadir下增加如下部分。

```bash
log_bin=mysql-bin
server_id=1
```
考虑到其他的服务器，服务器id必须是唯一的。推荐的方式是将master的ID设置为1，后续的每个slave节点序号大于1，使得所有服务器编号如：1，2，3等。
重启MySQL服务。如果是RHEL/CentOS系统，命令为：

```bash
# service mysqld restart
```
如果是Debian/Ubuntu系统，命令为：

```bash
# service mysql restart
```
在master上创建一个用于复制的账户，并赋予权限。我们创建用户”cloud-repl”，密码为”password”。假设master和slave都运行在172.16.1.0/24网段。

```bash
# mysql -u root
mysql> create user 'cloud-repl'@'172.16.1.%' identified by 'password';
mysql> grant replication slave on *.* TO 'cloud-repl'@'172.16.1.%';
mysql> flush privileges;
mysql> flush tables with read lock;
```
离开当前正在运行的MySQL会话,在新的shell中打开第二个MySQL会话。检索当前数据库的位置点。

```bash
# mysql -u root
mysql> show master status;
+------------------+----------+--------------+------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB |
+------------------+----------+--------------+------------------+
| mysql-bin.000001 |      412 |              |                  |
+------------------+----------+--------------+------------------+
```

注意你数据库实例所返回的文件及位置点,退出该会话。完成master安装。返回到master的第一个会话，取消锁定并退出MySQL。

```bash
mysql> unlock tables;
```

### 2.配置从节点
安装并配置slave节点。在slave服务器上，运行如下命令。

```bash
# yum install mysql-server
# chkconfig mysqld on
```
编辑my.cnf，在[mysqld]的datadir下增加如下部分。

```bash
server_id=2
innodb_rollback_on_timeout=1
innodb_lock_wait_timeout=600
```
重启MySQL。对于RHEL/CentOS系统，使用”mysqld”

```bash
# service mysqld restart
```
对于Ubuntu/Debian系统，使用”mysql.”

```bash
# service mysql restart
```
引导slave连接master并进行复制。使用上面步骤中得到数据来替换IP地址，密码，日志文件，以及位置点。

```bash
mysql> change master to
    -> master_host='172.16.1.217',
    -> master_user='cloud-repl',
    -> master_password='password',
    -> master_log_file='mysql-bin.000001',
    -> master_log_pos=412;
    ```
在slave上启动复制功能。

```bash
mysql> start slave;
```
在slave上可能需要开启3306端口。这对复制来说不是必须的。但如果没有做，当需要进行数据库切换时，你仍然需要去做。

### 故障切换
这将为管理服务器提供一个复制的数据库，用于实现手动故障切换。管理员将CloudStack从一个故障MySQL实例切换到另一个。当数据库失效发生时，你应该：

#### 1.停止管理服务器：
```bash
service cloudstack-management stop
```

将数据库的副本服务器修改为master并重启，确保数据库的副本服务器的3306端口开放给管理服务器。
更改使得管理服务器使用这个新的数据库。最简单的操作是在管理服务器的/etc/cloudstack/management/db.properties文件中写入新的数据库IP地址。

#### 2.重启管理服务器：

```bash
# service cloudstack-management start
```
