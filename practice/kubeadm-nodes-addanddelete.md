#  kubeadm Nodes添加和删除  

 Node是kubernetes集群的工作节点 ，是我们操作比较频繁的主机。

## 集群新增nodes节点
在我们使用kubeadm安装kubernetes Master节点时，安装成功后系统终端会提示如何将Nodes加入到集群中
```bash
kubeadm join k8s.xiaobu.vip:6443 --token 1q79dk.ljc2l4dfgsxqa2ib --discovery-token-ca-cert-hash sha256:0b240c66ed432fcbbcbaa966edc7d2371f43a429c95d1efeefd9f24dccd5eac8
```
### token过期hash忘记后解决办法
```bash
# 重新生成一条永久的token,在master节点上运行
kubeadm token create
# 列出token
kubeadm token list
# 获取ca证书sha256编码hash值
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
```
## 删除nodes节点
因为kubernetes的pod是调度运行在各个nodes上，日常工作中对node节点的维护，需要将pod重新调度到其他节点再进行操作。
### 标记nodes不可调度
```bash
kubectl cordon nodes
```
### 将nodes中pods调度到其他节点忽略daemonsets
```bash
kubectl drain nodes --delete-local-data --force --ignore-daemonsets
```
### 维护完成后，标记nodes可重新调度如删除nodes忽略此步骤
```bash
kubectl uncordon nodes
```

## master，node节点重置
master，nodes出现错误或者初始化出现问题需要重置
###  重置kubeadm
```bash
kubeadm reset -f
```
### 删除kubernetes各组件和关闭重启服务
```bash
iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X
ipvsadm --clear
systemctl stop kubelet
systemctl stop docker
rm -rf /var/lib/cni/
rm -rf /var/lib/kubelet/*
rm -rf /etc/cni/
ifconfig cni0 down
ifconfig flannel.1 down
ifconfig docker0 down
ip link delete cni0
ip link delete flannel.1
systemctl start docker
```
nodes加入集群请往前看，master参考[kubeadm安装kubernetes高可用集群](kubeadm-install-kubernetes.md)

