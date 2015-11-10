title: CloudStack 部署架构
tags: CloudStack
category: 云计算
---
CloudStack在实验室实践过程，架构部分。

CloudStack是一个开源的具有高可用性及扩展性的云计算平台。支持管理大部分主流的hypervisors，如KVM，XenServer，VMware，Oracle VM，Xen等。同时CloudStack是一个开源云计算解决方案。可以加速高伸缩性的公共和私有云（IaaS）的部署、管理、配置。使用CloudStack作为基础，数据中心操作者可以快速方便的通过现存基础架构创建云服务。
<!--more-->
# 架构示意图
![large-scale-redundant-setup](/images/large-scale-redundant-setup.png)

这个图展示了CloudStack在大规模部署时的网络结构。

* 三层交换层处于数据机房的核心位置。应该部署类似VRRP的冗余路由协议实现设备热备份。通常，高端的核心交换机上包含防火墙模块。如果三层交换机上没有集成防火墙功能，也可以使用独立防火墙设备。防火墙一般会配置成NAT模式，它能提供如下功能：

 * 将来自Internet的HTTP访问和API调用转发到管理节点服务器。管理服务器的网络属于管理网络.
 * 当云跨越多个区域（Zone）时，防火墙之间应该启用site-to-site VPN，以便让不同区域中的服务器之间可以直接互通。

* layer-2的接入交换层连接到每个提供点（POD），也可以用多个交换机堆叠来增加端口数量。无论哪种情况下，都应该部署冗余的二层交换机。

* 管理服务器集群(包括前端负载均衡节点，管理节点及MYSQL数据库节点)通过两个负载均衡节点接入管理网络。

* 辅助存储服务器接入管理网络。

* 每一个机柜提供点（POD）包括存储和计算节点服务器。每一个存储和计算节点服务器都需要有冗余网卡连接到不同的 layer-2 接入交换机上。


# 存储网络
大规模冗余部署时，存储的数据流量过大可能使得管理网络过载。部署时可选择将存储网络分离出来。存储协议如iSCSI，对网络延迟非常敏感。独立的存储网络能够使其不受来宾网络流量波动的影响。

![separate-storage-network](/images/separate-storage-network.png)

这个图展示了使用独立存储网络的设计。每一个物理服务器有四块网卡，其中两块连接到提供点级别的交换机，而另外两块网卡连接到用于存储网络的交换机。

有两种方式配置存储网络：

* 为NFS配置网卡绑定和冗余交换机。在NFS部署中，冗余的交换机和网卡绑定仍然处于同一网络。(同一个CIDR 段 + 默认网关地址)
* iSCSI能同时利用两个独立的存储网络(两个拥有各自默认网关的不同CIDR段)。支持iSCSI多路径的客户端能在两个独立的存储网络中实现故障切换和负载均衡。

![nic-bonding-and-multipath-io](/images/nic-bonding-and-multipath-io.png)

此图展示了网卡绑定与多路径IO(MPIO)之间的区别，网卡绑定的配置仅涉及一个网段，MPIO涉及两个独立的网段。

<a name="多节点管理服务器"></a>
# 多节点管理服务器
CloudStack管理服务器可以部署一个或多个前端服务器并连接单一的MySQL数据库。可视需求使用一对硬件负载均衡对Web请求进行分流。另一备份管理节点可使用远端站点的MySQL复制数据以增加灾难恢复能力。

![multi-node-management-server](/images/multi-node-management-server.png)

系统管理人员必须决定如下事项：

- 是否需要使用负载均衡器。
- 需要部署多少台管理节点服务器。
- 如何部署MYSQL复制以便启用灾难恢复。

# 选择Hypervisor
CloudStack支持多种流行的Hypervisor，你可以在所有的主机上使用一种，也可以使用不同的Hypervisor，但在同一个群集内的主机必须使用相同的Hypervisor。

你可能已经安装并运行了特定的hypervisor节点，在这种情况下，你的Hypervisor选择已经确定了。如果还处于初步规划阶段，那么就需要决定那种Hypervisor能切合你的需求。各种Hypervisor的利弊讨论不在本文档之列。但无论如何，CloudStack可以支持每种Hypervisor的那些功能的详细信息还是能够帮到你的。下面的表格就提供了这些信息：

| Feature                          | XenServer | vSphere      | KVM - RHEL | LXC | HyperV | Bare Metal |
|--|
| Network Throttling   | Yes       | Yes          | No         | No  | ?      | N/A        |
| Security groups in zones that use basic networking | Yes       | No           | Yes        | Yes | ?      | No         |
| iSCSI                            | Yes       | Yes          | Yes        | Yes | Yes    | N/A        |
| FibreChannel                     | Yes       | Yes          | Yes        | Yes | Yes    | N/A        |
| Local Disk                       | Yes       | Yes          | Yes        | Yes | Yes    | Yes        |
| HA                               | Yes       | Yes (Native) | Yes        | ?   | Yes    | N/A        |
| Snapshots of local disk          | Yes       | Yes          | Yes        | ?   | ?      | N/A        |
| Local disk as data disk          | Yes       | No           | Yes        | Yes | Yes    | N/A        |
| Work load balancing              | No        | DRS          | No         | No  | ?      | N/A        |
| Manual live migration of VMs from host to host | Yes       | Yes          | Yes        | ?   | Yes    | N/A        |        
| Conserve management traffic IP address by using link local network to communicate with virtual router    | Yes       | No           | Yes        | Yes | ?      | N/A        |


# Hypervisor Support for Primary Storage

The following table shows storage options and parameters for different
hypervisors.

| Primary Storage Type             | XenServer   | vSphere       | KVM - RHEL     | LXC            | HyperV |
|--|
| Format for Disks, Templates, and Snapshots    | VHD         | VMDK          | QCOW2      |                | VHD    |               
| iSCSI support                    | CLVM        | VMFS          | Yes via Shared Mountpoint  | Yes via Shared Mountpoint | No     |
| Fiber Channel support            | Yes, Via  existing SR   | VMFS          | Yes via Shared Mountpoint   | Yes via Shared Mountpoint  | No     |
| NFS support                      | Yes         | Yes           | Yes            | Yes            | No     |
| Local storage support            | Yes         | Yes           | Yes            | Yes            | Yes    |
| Storage over-provisioning        | NFS         | NFS and iSCSI | NFS            |                | No     |
| SMB/CIFS                         | No          | No            | No             | No             | Yes    |

XenServer uses a clustered LVM system to store VM images on iSCSI and Fiber Channel volumes and does not support over-provisioning in the hypervisor. The storage server itself, however, can support thin-provisioning. As a result the CloudStack can still support storage over-provisioning by running on thin-provisioned storage volumes.

KVM supports “Shared Mountpoint” storage. A shared mountpoint is a file system path local to each server in a given cluster. The path must be the same across all Hosts in the cluster, for example /mnt/primary1. This shared mountpoint is assumed to be a clustered filesystem such as OCFS2. In this case the CloudStack does not attempt to mount or unmount the storage as is done with NFS. The CloudStack requires that the administrator insure that the storage is available

With NFS storage, CloudStack manages the overprovisioning. In this case the global configuration parameter storage.overprovisioning.factor controls the degree of overprovisioning. This is independent of hypervisor type.

Local storage is an option for primary storage for vSphere, XenServer, and KVM. When the local disk option is enabled, a local disk storage pool is automatically created on each host. To use local storage for the System Virtual Machines (such as the Virtual Router), set system.vm.use.local.storage to true in global configuration.

CloudStack supports multiple primary storage pools in a Cluster. For example, you could provision 2 NFS servers in primary storage. Or you could provision 1 iSCSI LUN initially and then add a second iSCSI LUN when the first approaches capacity.


# 最佳实践

部署云计算服务是一项挑战。这需要在很多不同的技术之间做出选择，CLOUDSTACK以其配置灵活性可以使用很多种方法将不同的技术进行整合和配置。这个章节包含了一些在云计算部署中的建议及需求。

这些内容应该被视为建议而不是绝对性的。然而，我们鼓励想要部署云计算的朋友们，除了这些建议内容之外，最好从CLOUDSTACK的项目邮件列表中获取更多建议指南性内容。

## 实施最佳实践
* 强烈建议在系统部署至生产环境之前，有一个完全模拟生产环境的集成系统。对于已经在CloudStack中做了自定义修改的系统来说，更为重要了。

*  应该为安装，学习和测试系统预留充足的时间。简单网络模式的安装可以在几个小时内完成。但首次尝试安装高级网络模式通常需要花费几天的时间，完全安装则需要更长的时间。正式生产环境上线前，通常需要4-8周用以排除集成过程中的问题，你也可从cloudstack-users的邮件列表里得到更多帮助。

## 配置最佳实践
* 每一个主机都应该配置为只接受已知设备的连接，如CLOUDSTACK管理节点或相关的网络监控软件。

* 如果需要达到一定的高密度，可以在每个机柜提供点里部署多个集群。

* 主存储的挂载点或是LUN不应超过6TB大小。每个集群中使用多个小一些的主存储比只用一个超大主存储的效果要好。

* 在主存储上输出共享数据时，可用限制访问IP地址的方法避免数据丢失。更多详情，可参考”Linux NFS on Local Disks and DAS” “Linux NFS on iSCSI”这些章节。

* 网卡绑定技术可以明显的增加系统的可靠性。

* 当有大量服务器支持相当多的虚拟机时，推荐在存储访问的网络上采用将10G的带宽。

* 主机可创建的虚拟机的能力，主要取决于提供给客户虚拟机的内存。因为主机的存储和CPU均可超配，但内存却基本不可以。所以内存是在系统容量设计时的主要限制因素。

* (XenServer)可以为Xenserver的dom0分配更多的内存来让其支持更多的虚拟机。我们推荐为dom0设置的内存数值为2940 MB。至于具体操作，可以参见如下URL：http://support.citrix.com/article/CTX126531。这篇文章可同时适用于XenServer 5.6和6.0版本。

## 维护最佳实践
* 监视主机的磁盘空间。很多主机故障的原因都是日志将主机的硬盘空间占满导致的。

* 要监控每个集群里的虚拟机总量，如果达到了hypervisor所能承受的最大虚拟机数量时，就要禁止向此集群分配虚机。并且，要确定预留一定的安全迁移容量，以防止群集中有主机故障，这将增大其他主机运行虚拟机压力，就像是重新部署一批虚拟机一样。咨询你选择 hypervisor的文档，找到每台主机所能支持的最大虚拟机数量，并将此数值作为默认限制配置在CLOUDSTACK的全局设置里。监控每个群集中虚拟机的活动，保持虚拟机数量在安全线内，以防止偶然的主机故障。例如：如果集群里有N个主机，如果要让集群中一主机在任意时间停机，那么，此集群最多允许的虚拟机数量值为：(N-1) * (每宿主机最大虚拟量数量限值)。一旦达到此数量，必须在CLOUDSTACK的UI里禁止向此群集增加新的虚拟机。
