
# kubernetes之集群数据备份
Etcd是Kubernetes集群中的一个十分重要的组件，用于保存集群所有的网络配置和对象的状态信息

* 网络插件flannel、对于其它网络插件也需要用到etcd存储网络的配置信息
* kubernetes本身，包括各种对象的状态和元信息配置

数据备份重中之重，本文主要介绍备份kubernetes etcd数据和kubernetes所使用的数据

## ETCD原理
Etcd使用的是raft一致性算法来实现的，是一款分布式的一致性KV存储，主要用于共享配置和服务发现。关于raft一致性算法请参考[该动画演示](http://thesecretlivesofdata.com/raft/)
关于Etcd的原理解析请参考[Etcd 架构与实现解析](http://jolestar.com/etcd-architecture/)

## etcdctl安装
```bash
##下载二进制安装包
https://github.com/etcd-io/etcd/releases
wget https://github.com/etcd-io/etcd/releases/download/v3.4.0/etcd-v3.4.0-linux-amd64.tar.gz
tar xf etcd-v3.4.0-linux-amd64.tar.gz
cp -a etcdctl /usr/bin/
etcdctl version
```

## ETCD在Kubernetes集群中注意事项
flannel操作etcd使用的是v2的API，而kubernetes操作etcd使用的v3的API，所以在下面我们执行etcdctl的时候需要设置ETCDCTL_API环境变量，该变量默认值为2。

## 环境说明
3台kubeadm安装的kubernetes1.15.3

## ETCD集群查看
```bash
# 列出成员
etcdctl --endpoints=https://192.168.1.25:2379,https://192.168.1.26:2379,https://192.168.1.35:2379 --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key --cacert=/etc/kubernetes/pki/etcd/ca.crt member list
# 列出kubernetes数据
export ETCDCTL_API=3
etcdctl get / --prefix --keys-only --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key --cacert=/etc/kubernetes/pki/etcd/ca.crt
```

## 需要备份数据

* 备份 /etc/kubernetes/ 目录下的所有文件(证书，manifest文件)
* /var/lib/kubelet/ 目录下所有文件(plugins容器连接认证)
* etcd V3版api数据

### 备份脚本
```bash
#!/bin/bash
```

### 备份命令
```bash
#

#

#etcd备份
etcdctl --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key --cacert=/etc/kubernetes/pki/etcd/ca.crt snapshot save 20190917k8s-snapshot.db
```

## etcd数据恢复
### 注意事项
> 注意:
数据恢复操作，会停止全部应用状态和访问！！！

### 步骤

* 停止kube-apiserver
* 停止etcd
* 恢复数据
* 启动etcd
* 启动kube-apiserver

### 停止服务
首先需要分别停掉三台Master机器的kube-apiserver,etcd，确保kube-apiserver,etcd已经停止了。
```bash
mv /etc/kubernetes/manifests /etc/kubernetes/manifests.bak
```

### 数据恢复操作
etcd集群用同一份snapshot恢复。3台master节点上运行
```bash
##注意修改不同节点的IP
export ETCDCTL_API=3
etcdctl snapshot restore 2019-09-97-k8s-snapshot.db --endpoints=192.168.1.25:2379 --name=proapis001 --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key --cacert=/etc/kubernetes/pki/etcd/ca.crt --initial-advertise-peer-urls=https://192.168.1.25:2380 --initial-cluster-token=etcd-cluster-0 --initial-cluster=proapis001=https://192.168.1.25:2380,proapis002=https://192.168.1.26:2380,proapis003=https://192.168.1.35:2380 --data-dir=/var/lib/etcd
```

### 重新启动服务
全部恢复完成后，三台Master机器恢复manifests
```bash
mv /etc/kubernetes/manifests.bak /etc/kubernetes/manifests
```

### 确认数据恢复情况
```bash
etcdctl get / --prefix --keys-only --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key --cacert=/etc/kubernetes/pki/etcd/ca.crt
```

## 参考
* https://blog.csdn.net/ygqygq2/article/details/82753840