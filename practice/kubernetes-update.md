# kubeadm手动升级Kubernetes集群

## 概要
检查可用于升级到哪些版本，并验证您当前的集群是否可升级。
kubeadm 搭建的集群来更新是非常方便的,但kubeadm 的更新是不支持跨多个主版本的,1.10的版本只能更新到 1.11 版本了，然后再重 1.11 更新到 1.12...... 不过版本更新的方式方法基本上都是一样的。

## 更新操作
### 更新kubeadm，kubelet，kubectl软件包
```bash
yum update -y kubeadm-1.15.4 kubectl-1.15.4 kubelet-1.15.4
kubeadm version
```
### 检查可更新版本
```bash
kubeadm upgrade plan
Upgrade to the latest version in the v1.15 series:

COMPONENT            CURRENT   AVAILABLE
API Server           v1.15.2   v1.15.4
Controller Manager   v1.15.2   v1.15.4
Scheduler            v1.15.2   v1.15.4
Kube Proxy           v1.15.2   v1.15.4
CoreDNS              1.3.1     1.3.1
Etcd                 3.3.10    3.3.10

You can now apply the upgrade by executing the following command:

	kubeadm upgrade apply v1.15.4
```
### 更新到可更新版本
```bash
kubeadm upgrade apply v1.15.4
[upgrade/successful] SUCCESS! Your cluster was upgraded to "v1.15.4". Enjoy!
[upgrade/kubelet] Now that your control plane is upgraded, please proceed with upgrading your kubelets if you haven't already done so.
```
### kubelet更新
```shell
# 检查kubelet版本
kubelet --version
# 然后重启 kubelet 服务
systemctl daemon-reload
systemctl restart kubelet
```
### 检查版本情况
```bash
kubectl get nodes
NAME           STATUS   ROLES    AGE   VERSION
master01       Ready    master   10d   v1.15.4
```