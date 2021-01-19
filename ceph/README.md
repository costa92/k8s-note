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

###  获取镜像
由于国内环境无法镜像拉取 Pull 镜像，所以需要提前拉取镜像

```shell

# 拉取镜像 
docker pull ceph/ceph:v15.2.8 
docker pull rook/ceph:v1.5.5
docker pull registry.aliyuncs.com/it00021hot/cephcsi:v3.1.2
docker pull registry.aliyuncs.com/it00021hot/csi-node-driver-registrar:v2.0.1 
docker pull registry.aliyuncs.com/it00021hot/csi-attacher:v3.0.0 
docker pull registry.aliyuncs.com/it00021hot/csi-provisioner:v2.0.0 
docker pull registry.aliyuncs.com/it00021hot/csi-snapshotter:v3.0.0 
docker pull registry.aliyuncs.com/it00021hot/csi-resizer:v1.0.0 

# 设置tag 
docker tag registry.aliyuncs.com/it00021hot/csi-snapshotter:v3.0.0 k8s.gcr.io/sig-storage/csi-snapshotter:v3.0.0
docker tag registry.aliyuncs.com/it00021hot/csi-resizer:v1.0.0 k8s.gcr.io/sig-storage/csi-resizer:v1.0.0 
docker tag registry.aliyuncs.com/it00021hot/cephcsi:v3.1.2 quay.io/cephcsi/cephcsi:v3.1.2 
docker tag registry.aliyuncs.com/it00021hot/csi-node-driver-registrar:v2.0.1 k8s.gcr.io/sig-storage/csi-node-driver-registrar:v2.0.1 
docker tag registry.aliyuncs.com/it00021hot/csi-attacher:v3.0.0 k8s.gcr.io/sig-storage/csi-attacher:v3.0.0 
docker tag registry.aliyuncs.com/it00021hot/csi-provisioner:v2.0.0 k8s.gcr.io/sig-storage/csi-provisioner:v2.0.0
```


### 修改 cluster.yaml 配置文件

```yaml
#################################################################################################################
# Define the settings for the rook-ceph cluster with common settings for a production cluster.
# All nodes with available raw devices will be used for the Ceph cluster. At least three nodes are required
# in this example. See the documentation for more details on storage settings available.

# For example, to create the cluster:
#   kubectl create -f crds.yaml -f common.yaml -f operator.yaml
#   kubectl create -f cluster.yaml
#################################################################################################################

apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph # namespace:cluster
spec:
  cephVersion:
    # The container image used to launch the Ceph daemon pods (mon, mgr, osd, mds, rgw).
    # v13 is mimic, v14 is nautilus, and v15 is octopus.
    # RECOMMENDATION: In production, use a specific version tag instead of the general v14 flag, which pulls the latest release and could result in different
    # versions running within the cluster. See tags available at https://hub.docker.com/r/ceph/ceph/tags/.
    # If you want to be more precise, you can always use a timestamp tag such ceph/ceph:v15.2.8-20201217
    # This tag might not contain a new Ceph version, just security fixes from the underlying operating system, which will reduce vulnerabilities
    image: ceph/ceph:v15.2.8
    # Whether to allow unsupported versions of Ceph. Currently `nautilus` and `octopus` are supported.
    # Future versions such as `pacific` would require this to be set to `true`.
    # Do not set to true in production.
    allowUnsupported: false
  # The path on the host where configuration files will be persisted. Must be specified.
  # Important: if you reinstall the cluster, make sure you delete this directory from each host or else the mons will fail to start on the new cluster.
  # In Minikube, the '/data' directory is configured to persist across reboots. Use "/data/rook" in Minikube environment.
  dataDirHostPath: /var/lib/rook
  # Whether or not upgrade should continue even if a check fails
  # This means Ceph's status could be degraded and we don't recommend upgrading but you might decide otherwise
  # Use at your OWN risk
  # To understand Rook's upgrade process of Ceph, read https://rook.io/docs/rook/master/ceph-upgrade.html#ceph-version-upgrades
  skipUpgradeChecks: false
  # Whether or not continue if PGs are not clean during an upgrade
  continueUpgradeAfterChecksEvenIfNotHealthy: false
  mon:
    # Set the number of mons to be started. Must be an odd number, and is generally recommended to be 3.
    count: 3
    # The mons should be on unique nodes. For production, at least 3 nodes are recommended for this reason.
    # Mons should only be allowed on the same node for test environments where data loss is acceptable.
    allowMultiplePerNode: false
  mgr:
    modules:
    # Several modules should not need to be included in this list. The "dashboard" and "monitoring" modules
    # are already enabled by other settings in the cluster CR.
    - name: pg_autoscaler
      enabled: true
  # enable the ceph dashboard for viewing cluster status
  dashboard:
    enabled: true
    # serve the dashboard under a subpath (useful when you are accessing the dashboard via a reverse proxy)
    # urlPrefix: /ceph-dashboard
    # serve the dashboard at the given port.
    # port: 8443
    # serve the dashboard using SSL
    ssl: false
  # enable prometheus alerting for cluster
  monitoring:
    # requires Prometheus to be pre-installed
    enabled: false
    # namespace to deploy prometheusRule in. If empty, namespace of the cluster will be used.
    # Recommended:
    # If you have a single rook-ceph cluster, set the rulesNamespace to the same namespace as the cluster or keep it empty.
    # If you have multiple rook-ceph clusters in the same k8s cluster, choose the same namespace (ideally, namespace with prometheus
    # deployed) to set rulesNamespace for all the clusters. Otherwise, you will get duplicate alerts with multiple alert definitions.
    rulesNamespace: rook-ceph
  network:
    # enable host networking
    #provider: host
    # EXPERIMENTAL: enable the Multus network provider
    #provider: multus
    #selectors:
      # The selector keys are required to be `public` and `cluster`.
      # Based on the configuration, the operator will do the following:
      #   1. if only the `public` selector key is specified both public_network and cluster_network Ceph settings will listen on that interface
      #   2. if both `public` and `cluster` selector keys are specified the first one will point to 'public_network' flag and the second one to 'cluster_network'
      #
      # In order to work, each selector value must match a NetworkAttachmentDefinition object in Multus
      #
      #public: public-conf --> NetworkAttachmentDefinition object name in Multus
      #cluster: cluster-conf --> NetworkAttachmentDefinition object name in Multus
    # Provide internet protocol version. IPv6, IPv4 or empty string are valid options. Empty string would mean IPv4
    #ipFamily: "IPv6"
  # enable the crash collector for ceph daemon crash collection
  crashCollector:
    disable: false
  # enable log collector, daemons will log on files and rotate
  # logCollector:
  #   enabled: true
  #   periodicity: 24h # SUFFIX may be 'h' for hours or 'd' for days.
  # automate [data cleanup process](https://github.com/rook/rook/blob/master/Documentation/ceph-teardown.md#delete-the-data-on-hosts) in cluster destruction.
  cleanupPolicy:
    # Since cluster cleanup is destructive to data, confirmation is required.
    # To destroy all Rook data on hosts during uninstall, confirmation must be set to "yes-really-destroy-data".
    # This value should only be set when the cluster is about to be deleted. After the confirmation is set,
    # Rook will immediately stop configuring the cluster and only wait for the delete command.
    # If the empty string is set, Rook will not destroy any data on hosts during uninstall.
    confirmation: ""
    # sanitizeDisks represents settings for sanitizing OSD disks on cluster deletion
    sanitizeDisks:
      # method indicates if the entire disk should be sanitized or simply ceph's metadata
      # in both case, re-install is possible
      # possible choices are 'complete' or 'quick' (default)
      method: quick
      # dataSource indicate where to get random bytes from to write on the disk
      # possible choices are 'zero' (default) or 'random'
      # using random sources will consume entropy from the system and will take much more time then the zero source
      dataSource: zero
      # iteration overwrite N times instead of the default (1)
      # takes an integer value
      iteration: 1
    # allowUninstallWithVolumes defines how the uninstall should be performed
    # If set to true, cephCluster deletion does not wait for the PVs to be deleted.
    allowUninstallWithVolumes: false
  # To control where various services will be scheduled by kubernetes, use the placement configuration sections below.
  # The example under 'all' would have all services scheduled on kubernetes nodes labeled with 'role=storage-node' and
  # tolerate taints with a key of 'storage-node'.
#  placement:
#    all:
#      nodeAffinity:
#        requiredDuringSchedulingIgnoredDuringExecution:
#          nodeSelectorTerms:
#          - matchExpressions:
#            - key: role
#              operator: In
#              values:
#              - storage-node
#      podAffinity:
#      podAntiAffinity:
#      topologySpreadConstraints:
#      tolerations:
#      - key: storage-node
#        operator: Exists
# The above placement information can also be specified for mon, osd, and mgr components
#    mon:
# Monitor deployments may contain an anti-affinity rule for avoiding monitor
# collocation on the same node. This is a required rule when host network is used
# or when AllowMultiplePerNode is false. Otherwise this anti-affinity rule is a
# preferred rule with weight: 50.
#    osd:
#    mgr:
#    cleanup:
  annotations:
#    all:
#    mon:
#    osd:
#    cleanup:
#    prepareosd:
# If no mgr annotations are set, prometheus scrape annotations will be set by default.
#    mgr:
  labels:
#    all:
#    mon:
#    osd:
#    cleanup:
#    mgr:
#    prepareosd:
  resources:
# The requests and limits set here, allow the mgr pod to use half of one CPU core and 1 gigabyte of memory
#    mgr:
#      limits:
#        cpu: "500m"
#        memory: "1024Mi"
#      requests:
#        cpu: "500m"
#        memory: "1024Mi"
# The above example requests/limits can also be added to the mon and osd components
#    mon:
#    osd:
#    prepareosd:
#    crashcollector:
#    logcollector:
#    cleanup:
  # The option to automatically remove OSDs that are out and are safe to destroy.
  removeOSDsIfOutAndSafeToRemove: false
#  priorityClassNames:
#    all: rook-ceph-default-priority-class
#    mon: rook-ceph-mon-priority-class
#    osd: rook-ceph-osd-priority-class
#    mgr: rook-ceph-mgr-priority-class
  storage: # cluster level storage configuration and selection
    useAllNodes: false
    useAllDevices: false
    #deviceFilter:
    config:
      # crushRoot: "custom-root" # specify a non-default root label for the CRUSH map
      # metadataDevice: "md0" # specify a non-rotational storage so ceph-volume will use it as block db device of bluestore.
      # databaseSizeMB: "1024" # uncomment if the disks are smaller than 100 GB
      # journalSizeMB: "1024"  # uncomment if the disks are 20 GB or smaller
      # osdsPerDevice: "1" # this value can be overridden at the node or device level
      # encryptedDevice: "true" # the default value for this option is "false"
# Individual nodes and their config can be specified as well, but 'useAllNodes' above must be set to false. Then, only the named
# nodes below will be used as storage resources.  Each node's 'name' field should match their 'kubernetes.io/hostname' label.
    nodes:  # 打开配置
    - name: "192.168.11.132" #设置主机位置
      devices: # specific devices to use for storage can be specified for each node
      - name: "sda2"    # 硬盘的名字
    - name: "192.168.11.133"  # 设置主机位置
      directorios:
      - path: "/data/ceph"  # 133的文件路径
  # The section for configuring management of daemon disruptions during upgrade or fencing.
  disruptionManagement:
    # If true, the operator will create and manage PodDisruptionBudgets for OSD, Mon, RGW, and MDS daemons. OSD PDBs are managed dynamically
    # via the strategy outlined in the [design](https://github.com/rook/rook/blob/master/design/ceph/ceph-managed-disruptionbudgets.md). The operator will
    # block eviction of OSDs by default and unblock them safely when drains are detected.
    managePodBudgets: false
    # A duration in minutes that determines how long an entire failureDomain like `region/zone/host` will be held in `noout` (in addition to the
    # default DOWN/OUT interval) when it is draining. This is only relevant when  `managePodBudgets` is `true`. The default value is `30` minutes.
    osdMaintenanceTimeout: 30
    # A duration in minutes that the operator will wait for the placement groups to become healthy (active+clean) after a drain was completed and OSDs came back up.
    # Operator will continue with the next drain if the timeout exceeds. It only works if `managePodBudgets` is `true`.
    # No values or 0 means that the operator will wait until the placement groups are healthy before unblocking the next drain.
    pgHealthCheckTimeout: 0
    # If true, the operator will create and manage MachineDisruptionBudgets to ensure OSDs are only fenced when the cluster is healthy.
    # Only available on OpenShift.
    manageMachineDisruptionBudgets: false
    # Namespace in which to watch for the MachineDisruptionBudgets.
    machineDisruptionBudgetNamespace: openshift-machine-api

  # healthChecks
  # Valid values for daemons are 'mon', 'osd', 'status'
  healthCheck:
    daemonHealth:
      mon:
        disabled: false
        interval: 45s
      osd:
        disabled: false
        interval: 60s
      status:
        disabled: false
        interval: 60s
    # Change pod liveness probe, it works for all mon,mgr,osd daemons
    livenessProbe:
      mon:
        disabled: false
      mgr:
        disabled: false
      osd:
        disabled: false
```


```yaml
#################################################################################################################
# Define the settings for the rook-ceph cluster with common settings for a production cluster.
# All nodes with available raw devices will be used for the Ceph cluster. At least three nodes are required
# in this example. See the documentation for more details on storage settings available.

# For example, to create the cluster:
#   kubectl create -f crds.yaml -f common.yaml -f operator.yaml
#   kubectl create -f cluster.yaml
#################################################################################################################

apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph # namespace:cluster
spec:
  cephVersion:
    # The container image used to launch the Ceph daemon pods (mon, mgr, osd, mds, rgw).
    # v13 is mimic, v14 is nautilus, and v15 is octopus.
    # RECOMMENDATION: In production, use a specific version tag instead of the general v14 flag, which pulls the latest release and could result in different
    # versions running within the cluster. See tags available at https://hub.docker.com/r/ceph/ceph/tags/.
    # If you want to be more precise, you can always use a timestamp tag such ceph/ceph:v15.2.8-20201217
    # This tag might not contain a new Ceph version, just security fixes from the underlying operating system, which will reduce vulnerabilities
    image: ceph/ceph:v15.2.8
    # Whether to allow unsupported versions of Ceph. Currently `nautilus` and `octopus` are supported.
    # Future versions such as `pacific` would require this to be set to `true`.
    # Do not set to true in production.
    allowUnsupported: false
  # The path on the host where configuration files will be persisted. Must be specified.
  # Important: if you reinstall the cluster, make sure you delete this directory from each host or else the mons will fail to start on the new cluster.
  # In Minikube, the '/data' directory is configured to persist across reboots. Use "/data/rook" in Minikube environment.
  dataDirHostPath: /var/lib/rook
  # Whether or not upgrade should continue even if a check fails
  # This means Ceph's status could be degraded and we don't recommend upgrading but you might decide otherwise
  # Use at your OWN risk
  # To understand Rook's upgrade process of Ceph, read https://rook.io/docs/rook/master/ceph-upgrade.html#ceph-version-upgrades
  skipUpgradeChecks: false
  # Whether or not continue if PGs are not clean during an upgrade
  continueUpgradeAfterChecksEvenIfNotHealthy: false
  mon:
    # Set the number of mons to be started. Must be an odd number, and is generally recommended to be 3.
    count: 3
    # The mons should be on unique nodes. For production, at least 3 nodes are recommended for this reason.
    # Mons should only be allowed on the same node for test environments where data loss is acceptable.
    allowMultiplePerNode: false
  mgr:
    modules:
    # Several modules should not need to be included in this list. The "dashboard" and "monitoring" modules
    # are already enabled by other settings in the cluster CR.
    - name: pg_autoscaler
      enabled: true
  # enable the ceph dashboard for viewing cluster status
  dashboard:
    enabled: true
    # serve the dashboard under a subpath (useful when you are accessing the dashboard via a reverse proxy)
    # urlPrefix: /ceph-dashboard
    # serve the dashboard at the given port.
    # port: 8443
    # serve the dashboard using SSL
    ssl: true
  # enable prometheus alerting for cluster
  monitoring:
    # requires Prometheus to be pre-installed
    enabled: false
    # namespace to deploy prometheusRule in. If empty, namespace of the cluster will be used.
    # Recommended:
    # If you have a single rook-ceph cluster, set the rulesNamespace to the same namespace as the cluster or keep it empty.
    # If you have multiple rook-ceph clusters in the same k8s cluster, choose the same namespace (ideally, namespace with prometheus
    # deployed) to set rulesNamespace for all the clusters. Otherwise, you will get duplicate alerts with multiple alert definitions.
    rulesNamespace: rook-ceph
  network:
    # enable host networking
    #provider: host
    # EXPERIMENTAL: enable the Multus network provider
    #provider: multus
    #selectors:
      # The selector keys are required to be `public` and `cluster`.
      # Based on the configuration, the operator will do the following:
      #   1. if only the `public` selector key is specified both public_network and cluster_network Ceph settings will listen on that interface
      #   2. if both `public` and `cluster` selector keys are specified the first one will point to 'public_network' flag and the second one to 'cluster_network'
      #
      # In order to work, each selector value must match a NetworkAttachmentDefinition object in Multus
      #
      #public: public-conf --> NetworkAttachmentDefinition object name in Multus
      #cluster: cluster-conf --> NetworkAttachmentDefinition object name in Multus
    # Provide internet protocol version. IPv6, IPv4 or empty string are valid options. Empty string would mean IPv4
    #ipFamily: "IPv6"
  # enable the crash collector for ceph daemon crash collection
  crashCollector:
    disable: false
  # enable log collector, daemons will log on files and rotate
  # logCollector:
  #   enabled: true
  #   periodicity: 24h # SUFFIX may be 'h' for hours or 'd' for days.
  # automate [data cleanup process](https://github.com/rook/rook/blob/master/Documentation/ceph-teardown.md#delete-the-data-on-hosts) in cluster destruction.
  cleanupPolicy:
    # Since cluster cleanup is destructive to data, confirmation is required.
    # To destroy all Rook data on hosts during uninstall, confirmation must be set to "yes-really-destroy-data".
    # This value should only be set when the cluster is about to be deleted. After the confirmation is set,
    # Rook will immediately stop configuring the cluster and only wait for the delete command.
    # If the empty string is set, Rook will not destroy any data on hosts during uninstall.
    confirmation: ""
    # sanitizeDisks represents settings for sanitizing OSD disks on cluster deletion
    sanitizeDisks:
      # method indicates if the entire disk should be sanitized or simply ceph's metadata
      # in both case, re-install is possible
      # possible choices are 'complete' or 'quick' (default)
      method: quick
      # dataSource indicate where to get random bytes from to write on the disk
      # possible choices are 'zero' (default) or 'random'
      # using random sources will consume entropy from the system and will take much more time then the zero source
      dataSource: zero
      # iteration overwrite N times instead of the default (1)
      # takes an integer value
      iteration: 1
    # allowUninstallWithVolumes defines how the uninstall should be performed
    # If set to true, cephCluster deletion does not wait for the PVs to be deleted.
    allowUninstallWithVolumes: false
  # To control where various services will be scheduled by kubernetes, use the placement configuration sections below.
  # The example under 'all' would have all services scheduled on kubernetes nodes labeled with 'role=storage-node' and
  # tolerate taints with a key of 'storage-node'.
#  placement:
#    all:
#      nodeAffinity:
#        requiredDuringSchedulingIgnoredDuringExecution:
#          nodeSelectorTerms:
#          - matchExpressions:
#            - key: role
#              operator: In
#              values:
#              - storage-node
#      podAffinity:
#      podAntiAffinity:
#      topologySpreadConstraints:
#      tolerations:
#      - key: storage-node
#        operator: Exists
# The above placement information can also be specified for mon, osd, and mgr components
#    mon:
# Monitor deployments may contain an anti-affinity rule for avoiding monitor
# collocation on the same node. This is a required rule when host network is used
# or when AllowMultiplePerNode is false. Otherwise this anti-affinity rule is a
# preferred rule with weight: 50.
#    osd:
#    mgr:
#    cleanup:
  annotations:
#    all:
#    mon:
#    osd:
#    cleanup:
#    prepareosd:
# If no mgr annotations are set, prometheus scrape annotations will be set by default.
#    mgr:
  labels:
#    all:
#    mon:
#    osd:
#    cleanup:
#    mgr:
#    prepareosd:
  resources:
# The requests and limits set here, allow the mgr pod to use half of one CPU core and 1 gigabyte of memory
#    mgr:
#      limits:
#        cpu: "500m"
#        memory: "1024Mi"
#      requests:
#        cpu: "500m"
#        memory: "1024Mi"
# The above example requests/limits can also be added to the mon and osd components
#    mon:
#    osd:
#    prepareosd:
#    crashcollector:
#    logcollector:
#    cleanup:
  # The option to automatically remove OSDs that are out and are safe to destroy.
  removeOSDsIfOutAndSafeToRemove: false
#  priorityClassNames:
#    all: rook-ceph-default-priority-class
#    mon: rook-ceph-mon-priority-class
#    osd: rook-ceph-osd-priority-class
#    mgr: rook-ceph-mgr-priority-class
  storage: # cluster level storage configuration and selection
    useAllNodes: false
    useAllDevices: false
    #deviceFilter:
    config:
      # crushRoot: "custom-root" # specify a non-default root label for the CRUSH map
      # metadataDevice: "md0" # specify a non-rotational storage so ceph-volume will use it as block db device of bluestore.
      # databaseSizeMB: "1024" # uncomment if the disks are smaller than 100 GB
      # journalSizeMB: "1024"  # uncomment if the disks are 20 GB or smaller
      # osdsPerDevice: "1" # this value can be overridden at the node or device level
      # encryptedDevice: "true" # the default value for this option is "false"
# Individual nodes and their config can be specified as well, but 'useAllNodes' above must be set to false. Then, only the named
# nodes below will be used as storage resources.  Each node's 'name' field should match their 'kubernetes.io/hostname' label.
    nodes:
    - name: "192.168.11.133"
      devices:
      - name: "vdb"
#   - name: "192.168.11.132"  # 设置主机位置
#     directorios:
#     - path: "/data/ceph"  # 132的文件路径
  # The section for configuring management of daemon disruptions during upgrade or fencing.
  disruptionManagement:
    # If true, the operator will create and manage PodDisruptionBudgets for OSD, Mon, RGW, and MDS daemons. OSD PDBs are managed dynamically
    # via the strategy outlined in the [design](https://github.com/rook/rook/blob/master/design/ceph/ceph-managed-disruptionbudgets.md). The operator will
    # block eviction of OSDs by default and unblock them safely when drains are detected.
    managePodBudgets: false
    # A duration in minutes that determines how long an entire failureDomain like `region/zone/host` will be held in `noout` (in addition to the
    # default DOWN/OUT interval) when it is draining. This is only relevant when  `managePodBudgets` is `true`. The default value is `30` minutes.
    osdMaintenanceTimeout: 30
    # A duration in minutes that the operator will wait for the placement groups to become healthy (active+clean) after a drain was completed and OSDs came back up.
    # Operator will continue with the next drain if the timeout exceeds. It only works if `managePodBudgets` is `true`.
    # No values or 0 means that the operator will wait until the placement groups are healthy before unblocking the next drain.
    pgHealthCheckTimeout: 0
    # If true, the operator will create and manage MachineDisruptionBudgets to ensure OSDs are only fenced when the cluster is healthy.
    # Only available on OpenShift.
    manageMachineDisruptionBudgets: false
    # Namespace in which to watch for the MachineDisruptionBudgets.
    machineDisruptionBudgetNamespace: openshift-machine-api

  # healthChecks
  # Valid values for daemons are 'mon', 'osd', 'status'
  healthCheck:
    daemonHealth:
      mon:
        disabled: false
        interval: 45s
      osd:
        disabled: false
        interval: 60s
      status:
        disabled: false
        interval: 60s
    # Change pod liveness probe, it works for all mon,mgr,osd daemons
    livenessProbe:
      mon:
        disabled: true
      mgr:
        disabled: true
      osd:
        disabled: true
```




### 检查配置文件
执行命令查询命令
```
grep -v -e '#' -e ^$ cluster.yaml
```
执行结果
```yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster  # k8s 没有 CephCluster的
metadata:
  name: rook-ceph
spec:
  cephVersion:
    image: ceph/ceph:v15.2.8
    allowUnsupported: false
  dataDirHostPath: /var/lib/rook
  skipUpgradeChecks: false
  continueUpgradeAfterChecksEvenIfNotHealthy: false
  mon:
    count: 3
    allowMultiplePerNode: false
  mgr:
    modules:
    - name: pg_autoscaler
      enabled: true
  dashboard:
    enabled: true
    ssl: true
  monitoring:
    enabled: false
    rulesNamespace: rook-ceph
  network:
  crashCollector:
    disable: false
  cleanupPolicy:
    confirmation: ""
    sanitizeDisks:
      method: quick
      dataSource: zero
      iteration: 1
    allowUninstallWithVolumes: false
  annotations:
  labels:
  resources:
  removeOSDsIfOutAndSafeToRemove: false
    useAllNodes: false
    useAllDevices: false
    config:
    nodes:
    - name: "192.168.11.133"
      devices:
      - name: "vdb"
  disruptionManagement:
    managePodBudgets: false
    osdMaintenanceTimeout: 30
    pgHealthCheckTimeout: 0
    manageMachineDisruptionBudgets: false
    machineDisruptionBudgetNamespace: openshift-machine-api
  healthCheck:
    daemonHealth:
      mon:
        disabled: false
        interval: 45s
      osd:
        disabled: false
        interval: 60s
      status:
        disabled: false
        interval: 60s
    livenessProbe:
      mon:
        disabled: true
      mgr:
        disabled: true
      osd:
        disabled: true
```

### 安装错误信息

问题1:MountVolume.SetUp failed for volume "rook-ceph-crash-collector-keyring" : secret "rook-ceph-crash-collector-keyring" not found

解决方式：

 1. 清除`./var/lib/rook/`，`./var/lib/kubelet/plugins/`和`./var/lib/kubelet/plugins_registry/`目录，然后重新安装rook服务。
 2. 通过执行以下命令手动创建`rook-ceph-crash-collector-keyring`密码:`kubectl -n rook-ceph create secret generic rook-ceph-crash-collector-keyring`。

问题2： node(s) had taint {node-role.kubernetes.io/master: }, that the pod didn't tolerate, 2 node(s) didn't match Pod's node affinity

解决方式：
重新安装 rook-ceph 

```shell
kubectl delete -f cluster.yaml
kubectl delete -f crds.yaml -f common.yaml -f operator.yaml
```

删除各个节点的配置文件
```shell
rm -rvf /var/lib/rook/*
rm -rvf /var/lib/kubelet/plugins/rook-ceph.*
```

参考解决方式为 [rook/issues/6922](https://github.com/rook/rook/issues/6922)

###  配置ceph dashboard

在cluster.yaml文件中默认已经启用了ceph dashboard，查看dashboard的service：
```shell
kubectl get service -n rook-ceph
```
结果：
```shell
NAME                       TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
csi-cephfsplugin-metrics   ClusterIP   10.96.65.161    <none>        8080/TCP,8081/TCP   31m
csi-rbdplugin-metrics      ClusterIP   10.96.83.17     <none>        8080/TCP,8081/TCP   31m
rook-ceph-mgr              ClusterIP   10.96.18.217    <none>        9283/TCP            27m
rook-ceph-mgr-dashboard    ClusterIP   10.96.63.86     <none>        8443/TCP            27m
rook-ceph-mon-a            ClusterIP   10.96.241.97    <none>        6789/TCP,3300/TCP   31m
rook-ceph-mon-b            ClusterIP   10.96.229.154   <none>        6789/TCP,3300/TCP   29m
rook-ceph-mon-c            ClusterIP   10.96.161.185   <none>        6789/TCP,3300/TCP   28m
```

rook-ceph-mgr-dashboard监听的端口是8443，创建nodeport类型的service以便集群外部访问。
```shell
kubectl create -f dashboard-external-https.yaml
```
查看一下nodeport暴露的端口，这里是31695端口
```shell
kubectl get service -n rook-ceph |  grep dashboard
```
结果：
```shell
rook-ceph-mgr-dashboard                  ClusterIP   10.96.63.86     <none>        8443/TCP            29m
rook-ceph-mgr-dashboard-external-https   NodePort    10.96.135.174   <none>        8443:31695/TCP      7s
```

获取Dashboard的登陆账号和密码
