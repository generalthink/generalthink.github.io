title: 在Docker中使用HBase
date: 2018-11-24 14:53:39
tags: HBase
categories: HBase
description: 我们都知道HBase底层依赖Hadoop，那么在使用HBase的时候必须保证安装Hadoop,而平时我们想要快捷的使用HBase的时候则可以借助docker来使用别人安装好的HBase镜像,这样大大的省时省力,还能更快的对HBase有一个深入的了解。所以本文就介绍如何在Docker中使用Hbase
keywords: docker,HBabse
---

由于hbase依赖于hadoop,所以想要使用hbase必须先要安装hadoop(主要是我懒),我们可以通过docker使用别人安装好的hbase.

### 安装docker

docker的安装很简单,请参考<http://www.runoob.com/docker/centos-docker-install.html>


### 查看hbase镜像

首先查看关于hbase的镜像有哪些,运行 `docker search hbase`

![查看镜像](/images/hbase-docker-search-image.png)

我们就使用第一个好了,毕竟star最多。

### 拉取镜像

使用命令 `docker pull harisekhon/hbase` 就会将镜像拉取到本地。然后查看当前本地存在哪些镜像

![列出本地镜像](/images/hbase-docker-list-images.png)


### 运行hbase容器并进入容器内部

1. `docker run -dit harisekhon/hbase`

  启动容器,默认的版本是latest,如果运行的版本不是需要加上版本,比如`docker run -dit harisekhon/hbase:1.0`
![启动HBase容器](/images/hbase-docker-run-image.png)

2. `docker exec -it dc0716 bash`(进入容器内部,这里的dc0716是容器id前几位)

![进入容器内部](/images/hbase-docker-exec-bash.png)

可以看到此时容器内的hbase目录


### 运行hbase命令

![运行hbase shell](/images/hbase-docker-hbase-shell.png)

运行了`./hbase shell`命令后就进入了hbase shell命令行，此时可以用命令的方式查看hbase状态,新建表等
由于我们安装的hbase镜像是一个伪分布式的，只有一个服务器，所以查看状态显示只有一个server

![Hbase status](/images/hbase-docker-hbase-status.png)

更多的hbase命令将在下一篇文章中讲解

### 连接服务器的hbase

hbase默认使用的配置文件是hbase-site.xml位于hbase安装路径下的conf目录，有时候我们可能没有权限连接远程服务器,但是又想要看hbase的数据怎么办呢？这个时候可以修改hbase-site.xml中的配置连接上远程服务器。将项目中的hbase-site.xml替换默认的hbase-site.xml然后重新进入hbase shell即可。但是需要注意的是由于我们使用的docker，所以每次虚拟机关闭之后对hbase容器的修改都会还原，所以如果想要修改之后一直生效可以将修改保存成另一个镜像。

	1. 替换自带的hbase-site.xml
	2. docker commit containerId  saveImageName:version
    	比如： docker commit 430d centor-tar:1.0

之前也提到了hbase需要依赖于hadoop以及zookeeper,hbase自带了一个zookeeper,可以不使用它，而使用已有的zookeeper。所以能连接远程hbase服务器主要是因为我们在hbase-site.xml中配置了hadoop以及zookeeper的地址,这两个配置项的属性如下

```xml
<property>
    <name>hbase.rootdir</name>
    <value>hdfs://localhost:9020/hbase</value>
</property>
<property>
    <name>hbase.zookeeper.quorum</name>
    <value>localhost</value>
  </property>

```
这里的localhost:9020需要替换成对应服务器上Hadoop的地址以及端口,通过它们的配置我们就可以去hbase shell上愉快的操作hbase了。
