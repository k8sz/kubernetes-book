# kubeadm安装kubernetes高可用集群

## 基础环境

Master3台Nodes1台，kubernetes版本1.16.1，kubernetes网络flannel

| 主机名称 | IP地址      | 系统      |
| -------- | ----------- | --------- |
| master01 | 192.168.1.1 | Centos7.6 |
| master01 | 192.168.1.2 | Centos7.6 |
| master01 | 192.168.1.3 | Centos7.6 |
| node01   | 192.168.1.4 | Centos7.6 |

### 基础环境准备(所有主机操作)
#### 确认系统信息
```bash
[user@S001 ~]# cat /etc/redhat-release 
CentOS Linux release 7.6.1810 (Core)
[user@S001 ~]# uname -r
3.10.0-957.21.3.el7.x86_64
```
#### 修改主机名称
```bash
hostnamectl set-hostname --static mastar01
```
#### 修改/etc/hosts 
```bash
vim /etc/hosts
192.168.1.1	master01
192.168.1.2	master02
192.168.1.3	master03
192.168.1.4	node01

# 这里为了方便使用了DNS实现高可用
192.168.1.1	k8sz.local
192.168.1.2	k8sz.local
192.168.1.3	k8sz.local
```
#### 关闭防火墙和selinux
```bash
systemctl stop firewalld && systemctl disable firewalld
setenforce 0
sed -i s/SELINUX=enforcing/SELINUX=disabled/g /etc/selinux/config 
```
#### 开启IPVS
```bash
cat > /etc/sysconfig/modules/ipvs.modules <<EOF
#!/bin/bash
ipvs_modules="ip_vs ip_vs_lc ip_vs_wlc ip_vs_rr ip_vs_wrr ip_vs_lblc ip_vs_lblcr ip_vs_dh ip_vs_sh ip_vs_fo ip_vs_nq ip_vs_sed ip_vs_ftp nf_conntrack"
for kernel_module in \${ipvs_modules}; do
/sbin/modinfo -F filename \${kernel_module} > /dev/null 2>&1
if [ $? -eq 0 ]; then
/sbin/modprobe \${kernel_module}
fi
done
EOF
chmod 755 /etc/sysconfig/modules/ipvs.modules && bash /etc/sysconfig/modules/ipvs.modules && lsmod | grep ip_vs
```

## 安装docker
参考[Docker安装](../docker/docker-install.html)

## 安装Kubernetes相关组件
### 添加Kubernetes yum源
```bash
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```
### 安装指定kubernetes相关组件版本
```bash
# 列出kubectl版本
yum list kubectl --showduplicates | sort -r
# 安装kuberlet kuberadm kubectl
yum install -y kubelet-1.16.1 kubeadm-1.16.1 kubectl-1.16.1
# 设置kubelet开机启动
systemctl enable kubelet
```

## 通过kubradm初始化集群
### kubeadm配置文件
```bash
echo """
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
kubernetesVersion: v1.16.1
imageRepository: registry.aliyuncs.com/google_containers
controlPlaneEndpoint: "k8sz.local:6443"
apiServer:
certSANS:
- "k8sz.local"
networking:
podSubnet: 10.244.0.0/16
serviceSubnet: 10.88.0.0/16
etcd:
  external:
    endpoints:
    - https://192.168.1.1:2379
    - https://192.168.1.2:2379
    - https://192.168.1.3:2379
    caFile: /etc/kubernetes/pki/etcd/ca.crt
    certFile: /etc/kubernetes/pki/apiserver-etcd-client.crt
    keyFile: /etc/kubernetes/pki/apiserver-etcd-client.key
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: ipvs
""" > /etc/kubernetes/kubeadm-config.yaml
```

* 指定Kubernetes版本
* 指定apiServer，这里使用了域名，主要是为以后高可用做准备
* 指定etcd IP和证书
* 指定kube-proxy为IPVS模式，默认为iptbles

### 初始化master的kubelet（master01操作）
```bash
# 安装集群并打印安装日志
kubeadm init --config=/etc/kubernetes/kubeadm-config.yaml --upload-certs | tee kubeadm-init.log
```
### kubectl配置
```bash
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
```
### kubectl自动补全
```bash
source /usr/share/bash-completion/bash_completion
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc
echo "export KUBECONFIG=/etc/kubernetes/admin.conf" >> ~/.bash_profile
source ~/.bash_profile
```
### 安装flannel集群网络插件
```bash
kubectl apply -f  https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```
### 集群验证
```bash
kubectl get nodes
kubectl get pods --all-namespaces
```

## master02 master03加入到集群中
```bash
kubeadm join k8sz.local:6443 --token 8lp09d.m961tfj5cctkq76s --discovery-token-ca-cert-hash sha256:f0553763add9e232b4eebde18b018f5cd09a0c43afe6f490e38d0f30ba4f2708 --control-plane --certificate-key 305bb07984ed0f9a552fa0a4b172163f5cc1f772f13ad4b8daf983bc35eb0547
```
### token,certificate过期hash忘记后解决办法
```bash
# 重新生成一条永久的token,在master节点上运行
kubeadm token create
# 列出token
kubeadm token list
# 获取ca证书sha256编码hash值
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
# 重新生成certificate-key，在master节点上运行
kubeadm init phase upload-certs --upload-certs
```

## 将Nodes加入到集群中
```bash
kubeadm join k8s.xiaobu.vip:6443 --token 1q79dk.ljc2l4dfgsxqa2ib --discovery-token-ca-cert-hash sha256:0b240c66ed432fcbbcbaa966edc7d2371f43a429c95d1efeefd9f24dccd5eac8 
```

## 其他
### master node去除污点限制
```bash
# 查询污点信息
kubectl describe node master01 | grep Taint
# 去除污点
kubectl taint nodes master01 node-role.kubernetes.io/master-
# 设置污点
kubectl taint nodes master01 node-role.kubernetes.io/master=:NoSchedule
```
### pod无法访问api-service
```bash
iptables -t nat -I POSTROUTING -s 10.244.0.0/16 -j MASQUERADE
## 其中10.244.0.0/16为POD使用的网络
```

## 参考
* https://www.k8sz.com/post/kuberneteskubeadm/