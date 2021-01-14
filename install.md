更新源

```
yum  install epel-release 
```
更新镜像

```
kubeadm config images pull --kubernetes-version=v1.20.1 --image-repository registry.aliyuncs.com/google_containers
```
```
sudo yum update -y kubelet-1.20.1-0 kubeadm-1.20.1-0 kubectl-1.20.1-0  --disableexcludes=kubernetes
```

```
kubectl drain k8s-node1 --ignore-daemonsets
```
```
kubeadm upgrade apply 1.20.1
```