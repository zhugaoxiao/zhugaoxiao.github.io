title: 在Docker中运行你的Mesos集群
tags: Docker
category: 容器
---

我们经常为我们的客户和自己用[Apache Mesos](http://mesos.apache.org/) 集群运行容器化的应用。虽然我们已经在容器中运行应用，我们还是从标准的[Debian packages](https://mesosphere.com/downloads/)安装Mesos相关的依赖和框架。虽说这是很多Mesos用户的做法，而且是安装Mesos最直接和保险的方式，我们想可以略微调整一下Mesos的安装方式并尝试在容器中运行。

除了有很大的乐趣和教育意义外，这种方式会帮助我们解决一个非常实际的问题。现在可以运行任意版本的Mesos和它的框架，包括发布的候选版本，因为在我们的设置下，这个问题变成只要运行不同版本的Docker镜像就可以了。

当我们开发[Mesos框架](https://github.com/mesos/elasticsearch)时，能使用不同版本的Mesos（和其它工具）是非常重要的。除此之外，这给我们更多时间进行安全问题修复，有时甚至赶在它们发布之际。
<!--more-->
# Mesos, Marathon 和 Zookeeper Docker 镜像

目前在Docker Hub上有几个不同的[Apache Mesos Docker镜像](https://registry.hub.docker.com/search?q=mesos)可用 (注意Mesos master 和 slave是两个镜像)。我们选择[Mesosphere](https://registry.hub.docker.com/repos/mesosphere/)的镜像，一个卓越的容器生态系统贡献者（也是我们的朋友）。[Redjack](https://registry.hub.docker.com/repos/redjack/)的仓库同样提供高质量、文档丰富的Mesos镜像。使用哪个取决于你，哪个用起来都很正常，不过需要注意配置可能有些不同，因为这篇文章是为Mesosphere镜像而写。

上周[Thijs Schnitger](https://github.com/thijsschnitger) 创建了一个[Zookeeper3.5的镜像](https://registry.hub.docker.com/u/containersol/zookeeper/)。Apache Zookeeper在这个版本引入了动态主机重配置，一个非常酷的特性，特别是在一个经常变化的环境中特别有用，集群也是。所以我们也使用Zookeeper容器化的解决方案。

我们还会用到[Mesosphere的Marathon Docker 镜像](https://registry.hub.docker.com/u/mesosphere/marathon/)。

# 容器化集群配置

我们使用[Terraform](https://terraform.io/)启动集群。参考[terraform-mesos](https://github.com/ContainerSolutions/terraform-mesos)的GitHub repository并下载它的[容器](https://github.com/ContainerSolutions/terraform-mesos/tree/containers)分支，你会找到解决方案（至少在写这篇文章时）。不久，它会被合并到master分支。

让我们来看一下配置过程的有趣部分，docker运行Mesos，Marathon和Zookeeper命令。

## Mesos 主节点

```shell
QUORUM=2 # number of masters divided by 2 plus 1
CLUSTERNAME="cluster7"
ZK="zk://<quorum_string>/mesos"
MESOS_VERSION="0.22.1-1.0.ubuntu1404"

docker run -d \
-e MESOS_QUORUM=${QUORUM} \
-e MESOS_WORK_DIR=/var/lib/mesos \
-e MESOS_LOG_DIR=/var/log \
-e MESOS_CLUSTER=${CLUSTERNAME} \
-e MESOS_ZK=${ZK}/mesos \
--net="host" \
redjack/mesos-master:${MESOSVERSION}
--mesosphere/mesos-master:${MESOSVERSION}
```

如你所见，我们仅仅传递给镜像几个相关的版本tag就能运行指定版本的Mesos。除此之外，为了Mesos正常工作还需要传递几个环境变量给容器并启用[host networking](https://docs.docker.com/articles/networking/)。
简而言之，带`MESOS_`前缀的变量保存配置的值，我们一般会写到主机`/etc/mesos*`目录下的配置文件。

在集群中的每台主机上运行主节点容器。

## Mesos从节点

```shell
ZK="zk://<quorum_string>/mesos"
HOSTNAME="host1" # use a hostname of the host
IP="10.20.30.2" # use an IP address of the host
MESOSVERSION="0.22.1-1.0.ubuntu1404"

docker run -d \
 -e MESOS_LOG_DIR=/var/log/mesos \
 -e MESOS_MASTER=${ZK} \
 -e MESOS_EXECUTOR_REGISTRATION_TIMEOUT=5mins \
 -e MESOS_HOSTNAME=${HOSTNAME} \
 -e MESOS_ISOLATOR=cgroups/cpu,cgroups/mem \
 -e MESOS_CONTAINERIZERS=docker,mesos \
 -e MESOS_PORT=5051 \
 -e MESOS_IP=${IP} \
 -v /var/run/docker.sock:/var/run/docker.sock \
 -v /usr/bin/docker:/usr/bin/docker \
 -v /sys:/sys:ro \
 --net="host" \
redjack/mesos-slave
 --mesosphere/mesos-slave:${MESOSVERSION}
 ```

对Mesos从节点，配置有点复杂，主要是需要确保从节点可以访问主机上的Docker后台进程（同一个后台进程用来运行这个Mesos从节点实例）。通过挂载`/var/run/docker.sock`文件，`/usr/bin/docker`执行文件和`/sys` 目录（只读）到Mesos从节点容器实现。请注意，这还不是一个完美的解决方案，例如当从Marathon运行另一个Mesos框架，需要再进行细微调整。

## Zookeeper


# 第一个节点
```shell
docker run -d -p 2181:2181 containersol/zookeeper 1
```
# 其它节点
 ```shell
MYID="2" # 3,4,5,...
FIRST_NODE="cluster7-mesos-master-0" # hostname of the first node

docker run -d -p 2181:2181 containersol/zookeeper ${MYID} ${FIRST_NODE}
--需要更改端口2888:2888，3888:3888
```

在这个配置中，运行容器化的Zookeeper需要一点小技巧。在第一个节点以“standalone” 模式运行，等其他所有节点Zookeeper实例能连接到它并运行正常，Zookeeper接着会自动重新配置它自己向所有节点同步。

## Marathon

因为我们想运行应用容器和其它框架，我们在Mesos之上安装“datacenter init system” Marathon框架。

```shell
MARATHONVERSION="v0.8.2"
ZK="zk://<quorum>" # zookeeper quorum string

docker run -d \
 -p 8080:8080 \
 -p 5051:5051 \
 mesosphere/marathon:${MARATHONVERSION} \
 --master ${ZK}/mesos \
 --zk ${ZK}/marathon
 ```

 注意，我们指定Marathon UI 运行在8080端口，监听5051端口。我们运行任意版本镜像并传递两个强制参数：

 `master`  – quorum hostname/ip and Mesos registration path
 `zk`  – quorum hostname/ip and Marathon registration path

这样就可以在Docker容器中运行Mesos集群的关键组件了。除了运行不同版本的组件，这个我们最大的动机，这种方式自然而然的带来在集群管理层次容器化的所有好处：快速部署，维护简单和设备可移植性。

在这一过程，我们还需要解决haproxy-marathon bridge和[Mesos DNS](http://mesosphere.github.io/mesos-dns/)配置的问题。

如果你有任何问题或者要进行评论，请直接在文章下面留言或者在 [GitHub上创建issues](https://github.com/ContainerSolutions/terraform-mesos/issues)。

原文链接：[Containerized Mesos Cluster](http://container-solutions.com/containerized-mesos-cluster/) （翻译：朱高校）
