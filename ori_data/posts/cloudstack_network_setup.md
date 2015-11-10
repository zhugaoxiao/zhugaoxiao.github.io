title: CloudStack 网络配置
tags: CloudStack
category: 云计算
---
CloudStack在实验室实践过程，网络配置部分。
<!--more-->

# 网络配置

正确的网络设置对于CloudStack能否成功安装至关重要。

## 基本和高级网络

CloudStack提供二种网络类型:

**Basic**
适用于AWS类型的网络。提供单一的来宾网络类型，通过三层安全组进行隔离（源IP地址过滤）。

**Advanced**
适用于更复杂的网络拓扑。这种网络模型在定义来宾网络时提供了最大限度的灵活性，但是比基本网络需要更多的配置。

每个区域都有基本或高级网络。一个区域的整个生命周期中，不论是基本或高级网络。一旦在CloudStack中选择并配置区域的网络类型，就无法再更改。

下表是两种网络类型的功能比较。

| Networking Feature       |  Basic Network                     |   Advanced Network|
| :----: | :----: |:----:|
| Number of networks       |  Single network                    |   Multiple networks
| Firewall type            |  Physical                          |   Physical and Virtual
| Load balancer            |  Physical                          |   Physical and Virtual
| Isolation type           |  Layer 3                           |   Layer 2 and Layer 3
| VPN support              |  No                                |   Yes
| Port forwarding          |  Physical                          |   Physical and Virtual
| 1:1 NAT                  |  Physical                          |   Physical and Virtual
| Source NAT               |  No                                |   Physical and Virtual
| Userdata                 |  Yes                               |   Yes
| Network usage monitoring |  sFlow / netFlow at physical router|   Hypervisor and Virtual Router
| DNS and DHCP             |  Yes                               |   Yes

在一个云中可能会存在二种网络类型。但无论如何，一个给定的区域要么是基本网络要么是高级网络。

单一的物理网络可以被分割不同类型的网络流量。账户也可以分割来宾流量。你可以通过划分VLAN来隔离流量。如果你在物理网络中划分了VLAN，确保VLAN标签的数值在不同范围。

## VLAN分配示例

公共和来宾流量要求使用VLAN，下面是一个VLAN分配的示例：

| VLAN IDs   |          Traffic type        |                                        Scope|
| :----: | :----: |:----:|
| less than 500    |    Management traffic. Reserved for administrative purposes. |  CloudStack software can access this, hypervisors, system VMs.
| 500-599          |    VLAN carrying public traffic.                            |   CloudStack accounts.
| 600-799          |    VLANs carrying guest traffic.                             |  CloudStack accounts. Account-specific VLAN is chosen from this pool.
| 800-899          |    VLANs carrying guest traffic.                            |   CloudStack accounts. Account-specific VLAN chosen by CloudStack admin to assign to that account.
| 900-999           |   VLAN carrying guest traffic                              |   CloudStack accounts. Can be scoped by project, domain, or all accounts.
| greater than 1000 |   Reserved for future use| - |

## 硬件配置示例

本节包含了一个特定交换机型号的示例配置，作为区域级别的三层交换。假设VLAN管理协议，如VTP、GVRP等已经被禁用。如果你选择使用VTP或GVRP，那么你必须修改示例脚本。

### 三层交换（Cisco 3750）

以下步骤显示如何配置Cisco 3750作为区域级别的三层交换机。这些步骤假设VLAN 201用于Pod1中未标记的私有IP线路。并且Pod1中的二层交换机已经连接到GigabitEthernet0/1端口。

-  设置VTP为透明模式，以便可以使用超过1000的VLAN。因为我们只用到999，VTP透明模式不做严格要求。

```shell
en
show running-config
show vlan
show interfaces switchport
show interfaces trunk
show vtp status
//先确认原来配置
configure terminal
vtp mode transparent
vlan 200-999
exit
```

-  配置GigabitEthernet0/1。

```shell
interface GigabitEthernet0/1
switchport trunk encapsulation dot1q
switchport mode trunk
switchport trunk allowed vlan all
switchport trunk native vlan 201
exit
```

端口GigabitEthernet1/g1的配置说明：

-  端口GigabitEthernet1/0/1的本征VLAN为201（属于VLAN201）。
-  Cisco默认允许所有VLAN，因此，VLAN（300-999）都被允许并可访问所有的Pod级别的二层交换机。

### 二层交换（Cisco 3750）

在Pod中的二层交换机提供接入层功能。

-  它​通​过​trunk连​接​所​有​VLANS中​的​计​算​主​机。
-  它​为​管​理​网​络​提​供​​计​算​和​存​储​主​机​的​流​量​交​换​。​三​层​交​换​机​将​作​为​这​个​管​理​网​络​的​网​关。

以​下​步​骤​将​演​示​如何​配​置​cisco 3750作​为​Pod级​别​的​二层交​换​机，假设VTP、GVRP等已经被禁用。

-  设置VTP为透明模式，以便我们使用超过1000的VLAN。因为我们只用到999，VTP透明模式并不做严格要求。

```shell
Switch> en
Switch#
show running-config
show vlan
show interfaces switchport
show interfaces trunk
show vtp status
//先确认原来配置
configure terminal
vtp mode transparent
vlan 300-999
exit
```

-  配置所有的端口使用dot1q协议，并设置201为本征VLAN。

```shell
Switch#
interface range GigabitEthernet 0/1-24
switchport trunk encapsulation dot1q
switchport mod  e trunk
switchport trunk allowed vlan all
//switchport trunk native vlan 201
copy running-config startup-config
exit
```

默​认​情​况下​，Cisco允​许​所有​VLAN通过。​如果本征VLAN ID不相同的2个​​端​口​连​接​在​一​起时​​，​Cisco交​换​机​提出控诉。​这​就​是​为​什​么​必​须​指​定​VLAN201为​二层交​换​机​的​本​征​VLAN。


### 二层交换（H3C S5800X）

```
<DeviceA> system-view

[H3C] reset saved-configuretion
[H3C] display version
[H3C] display current-configuretion
[H3C] display saved-configuretion
[H3C] display vlan

[H3C] vlan 300 to 999
[H3C]interface range Ten-GigabitEthernet1/0/1 to Ten-GigabitEthernet1/0/24
[H3C-if-range]port link-type trunk
[H3C-if-range]port trunk permit vlan all
[DeviceA-GigabitEthernet1/0/3] quit
[H3C]save
[H3C]quit

```
## 硬件防火墙

所有部署都应该有一个防火墙保护管理服务器。根据情况不同，一些部署也可能用到Juniper SRX防火墙作为来宾网络的默认网关；参阅 “集成外部来宾防火墙Juniper SRX （可选）”。

### 防火墙的一般规定

硬件防火墙必需服务于两个目的：

-  Protect the Management Servers. NAT and port forwarding should be
   configured to direct traffic from the public Internet to the
   Management Servers.
   保护管理服务器。从互联网到管理服务器应该配置NAT和端口转发。

-  路由多个区域之间的管理网络流量。站点到站点VPN应该在多个区域之间配置。

为了达到上述目的，你必须设置固定配置的防火墙。当用户被配置到云中，不需要改变防火墙规则和策略。任何支持NAT和站点到站点VPN的硬件防火墙都可以使用。

### 集成Juniper SRX（可选）外部来宾防火墙

>注意：客户使用高级网络时才有效。

CloudStack中提供了对Juniper SRX系列防火墙的直接管理。这使得CloudStack能建立公共IP到客户虚拟机的静态NAT映射，并利用Juniper设备替代虚拟路由器提供防火墙服务。每个区域中可以有一个或多个Juniper SRX设备。这个特性是可选的。如果不提供Juniper设备集成，CloudStack会使用虚拟路由器提供这些服务。

Juniper SRX可以和任意的外部负载均衡器一起使用。外部网络元素可以部署为并排或内联结构。

![parallel-mode](/images/parallel-mode.png)

### 集成 Cisco VNMC (可选)外部来宾防火墙

Cisco虚拟网络管理中心（VNMC）为Cisco网络虚拟服务提供集中式多设备和策略管理。您可以通过集成Cisco VNMC到CloudStack，使用ASA 1000v云防火墙提供防火墙和NAT服务。在开启了Cisco Nexus 1000v分布式虚拟交换机功能的CloudStack群集中。这样部署，你可以：

-  配置Cisco ASA 1000v防火墙，你可以为每个来宾创建一个网络。

-  使用思科ASA 1000v防火墙创建和应用安全配置，包含入口和出口流量的ACL策略集。

-  使用Cisco ASA 1000v防火墙创建和应用源地址NAT，端口转发，静态NAT策略集。

CloudStack对于Cisco VNMC的支持基于开启了Cisco Nexus 1000v分布式虚拟交换机功能的VMware虚拟化管理平台。

## 整合外部来宾负载均衡（可选）

CloudStack可以使用Citrix NetScaler或BigIP F5负载均衡器为客户提供负载平衡服务。如果未启用,CloudStack将使用虚拟路由器中的软件负载平衡。

在CloudStack管理端中安装和启用外部负载均衡：

-  根据供应商提供的说明书安装你的设备。

-  连接到承载公共流量和管理流量的网络（可以是同一个网络）。

-  记录IP地址、用户名、密码、公共接口名称和专用接口名称。接口名称类似 “1.1” 或 “1.2”。

-  确保VLAN可以通过trunk到管理网络接口。

-  安装好CloudStack管理端后，使用管理员帐号登录CloudStack用户界面。

-  在左侧导航栏中，点击基础架构。

-  点击区域中的查看更多。

-  选择你要设置的区域。

-  点击网络选项卡。

-  点击示意图网络服务提供程序中的配置（你可能需要向下滚动才能看到）。

-  点击NetScaler或F5。

-  点击（+）添加按钮并提供如下信息：

对于NetScaler:

   - IP地址：SRX设备的IP地址。

   - 用户名/密码：访问设备的身份验证凭证。CloudStack使用这些凭证来访问设备。

   - 类型：添加设备的类型。可以是F5 BigIP负载均衡器、NetScaler VPX、NetScaler MPX或 NetScaler SDX等设备。关于NetScaler的类型比较，请参阅CloudStack管理指南。

   - 公共接口: 配置为公共网络部分的设备接口。

   - 专用接口: 配置为专用网络部分的设备接口。

   - 重试次数：尝试控制设备失败时重试的次数。默认值是2。

   - 容量：该设备能处理的网络数量。

   - 专用: 当标记为专用后，这个设备只对单个帐号专用。该选项被勾选后，容量选项就没有了实际意义且值会被置为1。

#. 点击确定。

外部负载平衡器的安装和配置完成后，你就可以开始添加虚拟机和NAT或负载均衡规则。

## 管理服务器负载均衡

CloudStack可以使用负载均衡器为多管理服务器提供一个虚拟IP。管理员负责创建管理服务器的负载均衡规则。应用程序需要跨多个持久性或stickiness的会话。下表列出了需要进行负载平衡的端口和是否有持久性要求。

即使不需要持久性，也使它是允许的。

| Source Port |  Destination Port    |        Protocol   |  Persistence Required?
| ---         |---                   |---                |
| 80 or 443   |8080(or 20400 with AJP)|  HTTP (or AJP)   |  Yes
| 8250        | 8250                  |   TCP            |  Yes
| 8096        |      8096             |   HTTP           |  No

除了上面的设置，管理员还负责设置‘host’全局配置值，由管理服务器IP地址更改为负载均衡虚拟IP地址。如果‘host’值未设置为VIP的8250端口并且一台管理服务器崩溃时，用户界面依旧可用，但系统虚拟机将无法与管理服务器联系。

## 拓扑结构要求

### 安全需求

公共互联网时，必须禁止管理服务器8096和8250的端口访问。

### 运行时内部通信需求

-  管理服务器需要跟其他主机进行任务协调。该通信使用TCP协议的8250和9090端口。

-  CPVM跟区域中的所有主机通过管理网络通信。因此区域中任何一个Pod都必须能通过管理网络连接到其他Pod。

-  SSVM和CPVM通过8250端口与管理服务器通信。如果你使用了多个管理服务器，确保负载均衡器IP地址到管理服务器的8250端口是可达的。

### 存储网络拓扑要求

SSVM需要挂载辅助存储中的NFS共享目录。即使有一个单独的存储网络，辅助存储的流量也会通过管理网络。如果存储网络可用，主存储的流量会通过存储网络。如果你选择辅助存储也使用存储网络，你必须确保有一条从管理网络到存储网络的路由。

### 外部防火墙拓扑要求

如果整合了外部防火墙设备，公共IP的VLAN必须通过trunk到达主机。这是SSVM和CPVM要求必须满足的。

### 高级区域拓扑要求

使用高级网络，专用和公共网络必须分离子网。

### XenServer拓扑要求

管理服务器与XenServer服务器通过22（ssh）、80（http）和443（https）端口通信。

### VMware拓扑要求

-  管理服务器和SSVM必须能够访问区域中的vCenter和所有的ESXi主机。必须保证在防火墙中允许443端口。

-  管理服务器与VMware vCenter服务器通过443（https）端口通信。

-  管理服务器与系统VM使用管理网络通过3922（ssh）端口通信。

### Hyper-V拓扑要求

CloudStack管理服务器通过https与Hyper-V代理通信。管理服务器与Hyper-V主机之间的安全通信端口为8250。

### KVM拓扑要求

管理服务器与KVM主机通过22（ssh）端口通信。

### LXC拓扑要求

管理服务器与LXC主机通过22（ssh）端口通信。

## Guest Network Usage Integration for Traffic Sentinel
## 通过流量哨兵整合来宾网络使用情况

为了在来宾网络中收集网络数据，CloudStack需要从网络中安装的外部网络数据收集器中提取。通过整合CloudStack和inMon流量哨兵实现来宾网络的数据统计。

网络哨兵是一个收集网络流量使用数据的套件。CloudStack可以将流量哨兵中的信息统计到自己的使用记录中,为云基础架构的计费用户提供了依据。流量哨兵使用的流量监控协议为sFlow。路由器和交换机将生成的sFlow记录提供给流量哨兵，然后CloudStack通过查询流量哨兵的数据库来获取这些信息。

为了构建查询，CloudStack确定了当前来宾IP的查询时间间隔。这既包含新分配的IP地址，也包含之前已被分配并且在继续使用的IP地址。CloudStack查询流量哨兵，在分配的时间段内这些IP的网络统计信息。返回的数据是账户每个IP地址被分配和释放的时间戳，以便在CloudStack中为每个账户创建计费记录。当使用服务运行时便会收集这些数据

配置整合CloudStack和流量哨兵：

-  在网络基础设施中，安装和配置流量哨兵收集流量数据。关于安装和配置步骤，请查阅inMon的[Traffic Sentinel Documentation](http://inmon.com)。

-  在流量哨兵用户界面中，配置流量哨兵允许来宾用户使用脚本查询。CloudStack将通过执行远程查询为来宾用户的一个或多个IP和收集网络使用情况。
   点击 File > Users > Access Control > Reports Query, 然后从下拉列表中选择来宾。

-   在CloudStack中，使用API中的addTrafficMonitor命令添加流量哨兵主机。传入的流量哨兵URL类似于protocol + host + port (可选)；例如，http://10.147.28.100:8080 。关于addTrafficMonitor命令用法，请参阅[API文档](http://cloudstack.apache.org/docs/api/index.html)。

   关于如何调用CloudStack API，请参阅[CloudStack API Developer's Guide](http://docs.cloudstack.apache.org/en/latest/index.html#developers)。

-   作为管理员登录到CloudStack用户界面。

-   在全局设置页面中，查找如下参数进行设置：

   direct.network.stats.interval: 你希望CloudStack多久收集一次流量数据。


## 设置区域中VLAN和虚拟机的最大值

在外部网络案例中，每个虚拟机都必须有独立的来宾IP地址。在CloudStack中有两个参数你需要考虑如何进行配置：在同一时刻，你希望区域中有多少VLAN和多少虚拟机在运行。

使用如下表格来确定如何在CloudStack部署时修改配置。

| guest.vlan.bits |   Maximum Running VMs per Zone  |  Maximum Zone VLANs
|- |- |
| 12              |   4096                         |   4094
| 11              |   8192                         |   2048
| 10              |   16384                        |   1024
| 10              |   32768                        |   512

基于部署需求，选择合适的guest.vlan.bits参数值。在全局配置中编辑该参数，并重启管理服务。
