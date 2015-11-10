title: 创建PyPi镜像仓库
tags: PyPi
category: 黑科技
---
团队在日常开发和生产部署中，经常会用到PyPi的仓库，但是由于GFW的原因，经常出现丢包、超时等情况。因此在实验室开源镜像服务器定期同步一个PyPi镜像仓库。

# 环境描述

CentOS release 6.7 (Final)

如果你使用Ubuntu，配置非常简单，直接pip install bandersnatch即可，但公司现在主要环境是RedHat系列，有点小麻烦，具体过程如下。
<!--more-->

# 安装相关依赖


```
# yum -y install gcc automake autoconf libtool make //安装python需要
# yum -y install zlib zlib-devel bzip2-devel openssl-devel ncurses-devel sqlite-devel  //安装disttibute需要
# yum -y install unzip
```

# 安装python

centos自带python-2.6，这里需要python 2.7，目前最新版本的python是2.7.10，所以下载下载完成后解压安装。

```
# python -v
# cd /usr/src
# wget https://www.python.org/ftp/python/2.7.10/Python-2.7.10.tgz
# tar -zxvf Python-2.7.10.tgz
# cd Python-2.7.10
# ./configure --prefix=/usr/local
# make && make altinstall
//make altinstall is used to prevent replacing the default python binary file /usr/bin/python.
# ls -ltr /usr/bin/python*
# ls -ltr /usr/local/bin/python*
# /usr/local/bin/python2.7
```

# 安装distribute
```
# wget https://pypi.python.org/packages/source/d/distribute/distribute-0.7.3.zip
# unzip distribute-0.7.3.zip
# cd distribute-0.7.3
# /usr/local/bin/python2.7 setup.py install
# which easy_install
/usr/local/bin/easy_install
# which easy_install-2.7
/usr/local/bin/easy_install-2.7
```
> distribute是setuptools的取代。参考:https://pypi.python.org/pypi/distribute/0.7.3。

setuptools:setuptools 是一组由PEAK(Python Enterprise Application Kit)开发的 Python 的 distutils 工具的增强工具，可以让程序员更方便的创建和发布 Python的egg 包，特别是那些对其它包具有依赖性的状况。 由 setuptools 创建和发布的包看起来和基于 distutils 发布的包没什么不同。最终用户不需要事先安装 setuptools 甚至根本不需要知道 setuptools 的存在，而程序员也不需要附上完整的 setuptools，只需要包含一个大小约 8K 的ez_setup.py脚本作为启动模块，就可以在最终用户没有安装适当版本的 setuptools 时让这些包自动下载和安装 setuptools。
easy_install: 常使用python的人员，当需要安装第三方python包时，可能会用到easy_install命令。easy_install是由PEAK(Python Enterprise Application Kit)开发的setuptools包里带的一个命令，它用来自动地从http://pypi.python.org/simple/来安装egg包，相当于perl中的cpan或PPM、RedHat中的yum命令，但是系统都没有预装easy_install命令。

# 安装pip
```
# wget https://pypi.python.org/packages/source/p/pip/pip-7.1.2.tar.gz#md5=3823d2343d9f3aaab21cf9c917710196
# tar zxvf pip-7.1.2.tar.gz
# cd pip-7.1.2
# /usr/local/bin/python2.7 setup.py install
# which pip2.7
/usr/local/bin/pip2.7
```
>Pip 是安装python包的工具，是对easy_install的取代，提供了和easy_install相同的安装包、列出已经安装的包、查找包的功能。
参考：https://pypi.python.org/pypi/pip/

# 安装virtualenv
```
# wget https://pypi.python.org/packages/source/v/virtualenv/virtualenv-13.1.2.tar.gz
# tar -zxvf virtualenv-13.1.2.tar.gz
# cd virtualenv-13.1.2
# /usr/local/bin/python2.7 setup.py install
# which virtualenv-2.7
/usr/local/bin/virtualenv-2.7
```
> 参考：https://pypi.python.org/pypi/virtualenv

# 安装bandsnatch

```
# cd /opt
# /usr/local/bin/virtualenv-2.7 bandersnatch
# cd bandersnatch
# bin/pip install -r https://bitbucket.org/pypa/bandersnatch/raw/stable/requirements.txt
```
或者
```
# pip2.7 install bandersnatch
```

>由于众所周知的GFW网络问题，可能出现timeout错误，重新执行多次直到全部下载正常。
参考：https://pypi.python.org/pypi/bandersnatch

# 设置bandsnatch

```
# cd /opt/bandersnatch
# bin/bandersnatch mirror
2015-09-18 22:59:54,076 WARNING: Config file '/etc/bandersnatch.conf' missing, creating default config.
2015-09-18 22:59:54,076 WARNING: Please review the config file, then run 'bandersnatch' again.
```
编辑/etc/bandersnatch.conf文件，修改pypi源的存储路径。重新执行bin/bandersnatch mirror，就开始同步pip官方源到本地，此过程可能比较长，而且可能会由于网络原因超时报错，需要多次重复执行该命令
```
# mkdir -p /opt/pypi
# vi /etc/bandersnatch.conf
sed -i "s/192.168.200.35/192.168.200.25/g" `grep 192.168.200.35 -rl/home/weblogic/bea/user_projects/domains/base_domain/config/jdbc`
# bin/bandersnatch mirror
```

# 配置Apache，即将pip做成本地web源

```
#
# ln -s /opt/pypi/web /usr/share/nginx/html/pypi

```

# 设置pip客户端

* 方法一

在用户目录下
```
# mkdir ~/.pip
# vim ~/.pip/pip.conf
---------------pip.conf---------------
[global]
index-url = http://www.linuxhot.com/pypi/simple

pip install mitmproxy -i http://pypi.mirrors.ustc.edu.cn/simple --trusted-host pypi.mirrors.ustc.edu.cn
[global]
timeout = 6000
index-url = http://pypi.douban.com/simple/
[install]
use-mirrors = true
mirrors = http://pypi.douban.com/simple/
trusted-host = pypi.douban.com
阿里源：
http://mirrors.aliyun.com/pypi/simple/
```

* 方法二

```
# pip install xxx –i http://www.linuxhot.com/pypi/simple
```

至此，pip本地源服务器搭建完毕。
