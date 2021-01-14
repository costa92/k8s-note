## Rook使用

Rook是一个自管理的分布式存储编排系统，可以为Kubernetes提供便利的存储解决方案。Rook本身并不提供存储，而是在kubernetes和存储系统之间提供适配层，简化存储系统的部署与维护工作。目前，rook支持的存储系统包括：Ceph、CockroachDB、Cassandra、EdgeFS、Minio、NFS，其中Ceph为Stable状态，其余均为Alpha。本文仅介绍Ceph相关内容。

##  Rook由Operator和Cluster两部分组成：

Operator：由一些CRD和一个All in one镜像构成，包含包含启动和监控存储系统的所有功能。
   
Cluster：负责创建CRD对象，指定相关参数，包括ceph镜像、元数据持久化位置、磁盘位置、dashboard等等…


### Rook:

  Agent: 在每个储存节点运行，用于配置一个 FlexVolume 插件，和k8s的存储卷进行集成，挂载网络存储、加载存储卷、格式化文件系统
  
  Discover: 主要用于检测链接到存储节点上的存储设备

Rook的体系结构图：

![Rook的体系结构图](https://img-blog.csdnimg.cn/20190327143640947.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2NoZW5sZWlraW5n,size_16,color_FFFFFF,t_70)


### Ceph:

 OSD: 直接连接每一个集群节点的物理磁盘或者是目录，集群的副本数，高可用性和容错性
MON: 集群监控，所有集群的节点都会向 Mon 汇报，他记录了集群的拓扑以及数据存储位置的信息
MOS: 元数据服务器，负责跟踪文件层次结构并存储ceph 元数据
 RGW：restful API 接口
MGR：提供额外的监控和界面


Rook的官网文档:
 https://www.rook.io/docs/rook/v1.5/ceph-quickstart.html

### 安装 Rook

查看设备的存储信息
```shell
lsblk -f
```

下载 rook 包为文件
```shell
git clone --single-branch --branch v1.5.5 https://github.com/rook/rook.git
cd rook/cluster/examples/kubernetes/ceph
kubectl create -f crds.yaml -f common.yaml -f operator.yaml
kubectl create -f cluster.yaml
```

查看安装的状态
```shell
kubectl get po -n rook-ceph
```