# 使用kubesphere管理集群日志和监控
KubeSphere 是在目前主流容器调度平台 Kubernetes 之上构建的企业级分布式多租户容器平台，提供简单易用的操作界面以及向导式操作方式，在降低用户使用容器调度平台学习成本的同时，极大减轻开发、测试、运维的日常工作的复杂度，旨在解决 Kubernetes 本身存在的存储、网络、安全和易用性等痛点。除此之外，平台已经整合并优化了多个适用于容器场景的功能模块，以完整的解决方案帮助企业轻松应对敏捷开发与自动化运维、微服务治理、多租户管理、工作负载和集群管理、服务与网络管理、应用编排与管理、镜像仓库管理和存储管理等业务场景。

KubeSphere为我们提供了可视化 CI/CD 流水线、多维度监控告警日志、多租户管理、LDAP 集成、新增支持 HPA (水平自动伸缩) 、容器健康检查以及 Secrets、ConfigMaps 的配置管理等功能，新增微服务治理、灰度发布、s2i、代码质量检查等。一定程度上简化了我们管理kubesnetes集群。

## 在 Kubernetes 在线部署 KubeSphere
### 创建Namespace
在 Kubernetes 集群中创建名为 kubesphere-system 和 kubesphere-monitoring-system 的 namespace
```bash
cat <<EOF | kubectl create -f -
---
apiVersion: v1
kind: Namespace
metadata:
    name: kubesphere-system
---
apiVersion: v1
kind: Namespace
metadata:
    name: kubesphere-monitoring-system
EOF
```
### 创建 Kubernetes 集群 CA 证书的 Secret
```bash
kubectl -n kubesphere-system create secret generic kubesphere-ca --from-file=ca.crt=/etc/kubernetes/pki/ca.crt --from-file=ca.key=/etc/kubernetes/pki/ca.key 
```
### 创建 etcd 的证书 Secret
```bash
kubectl -n kubesphere-monitoring-system create secret generic kube-etcd-client-certs --from-file=etcd-client-ca.crt=/etc/kubernetes/pki/etcd/ca.crt --from-file=etcd-client.crt=/etc/kubernetes/pki/etcd/healthcheck-client.crt --from-file=etcd-client.key=/etc/kubernetes/pki/etcd/healthcheck-client.key
```
### 克隆 kubesphere-installer 仓库至本地
```bash
git clone https://github.com/kubesphere/ks-installer.git
```
### 修改安装配置文件
```bash
cd ks-installer/deploy
vim kubesphere-installer.yaml
```
```yaml
apiVersion: v1
data:
  ks-config.yaml: |
    kube_apiserver_host: 192.168.1.1:6443
    etcd_tls_enable: True
    etcd_endpoint_ips: 192.168.1.1
    disableMultiLogin: False
    istio_enable: False
    keep_log_days: 30
    sonarqube_enable: False
    metrics_server_enable: False
    elk_prefix: logstash
    persistence:
      enable: True
      storageClass: "alicloud-disk-available"
......
```
配置文件关闭了istio，sonarqube，metrics_server之前有安装所以这里不安装，日志保存30天，开启storageClass参考[kubernetes 挂载阿里云nas存储作为StorageClass](https://www.k8sz.com/post/kubenetes-nas-storageclass),开启账户可以多终端登录
### 配置文件参数
| 参数                             | 描述                                                         | 默认值                                                       |      |
| -------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ---- |
| kube_apiserver_host              | 当前集群kube-apiserver地址（ip:port）                        |                                                              |      |
| etcd_tls_enable                  | 是否开启etcd TLS证书认证（True / False）                     | True                                                         |      |
| etcd_endpoint_ips                | etcd地址，如etcd为集群，地址以逗号分离（如：192.168.0.7,192.168.0.8,192.168.0.9） |                                                              |      |
| etcd_port                        | etcd端口 (默认2379，如使用其它端口，请配置此参数)            | 2379                                                         |      |
| disableMultiLogin                | 是否关闭多点登录  （True / False）                           | True                                                         |      |
| elk_prefix                       | 日志索引                                                     | logstash                                                     |      |
| keep_log_days                    | 日志留存时间（天）                                           | 7                                                            |      |
| metrics_server_enable            | 是否安装metrics_server    （True / False）                   | True                                                         |      |
| istio_enable                     | 是否安装istio       （True / False）                         | True                                                         |      |
| persistence                      | enable                                                       | 是否启用持久化存储  （True /  False）（非测试环境建议开启数据持久化） |      |
| storageClass                     | 启用持久化存储要求环境中存在已经创建好的 StorageClass（默认为空，则使用 default StorageClass） | “”                                                           |      |
| containersLogMountedPath（可选） | 容器日志挂载路径                                             | "/var/lib/docker/containers"                                 |      |
| external_es_url（可选）          | 外部es地址，支持对接外部es用                                 |                                                              |      |
| external_es_port（可选）         | 外部es端口，支持对接外部es用                                 |                                                              |      |
| local_registry (离线部署使用)    | 离线部署时，对接本地仓库 （使用该参数需将安装镜像使用scripts/download-docker-images.sh导入本地仓库中） |                                                              |      |
运行部署
```shell
kubectl apply -f kubesphere-installer.yaml
```
### 查看部署日志
```shell
kubectl logs -n kubesphere-system $(kubectl get pod -n kubesphere-system -l job-name=kubesphere-installer -o jsonpath='{.items[0].metadata.name}') -f
```
### kubesphere镜像无法下载
```shell
#登陆到镜像仓库，每个nodes都需要操作
docker login -u guest -p guest dockerhub.qingcloud.com
```
### 查看控制台的服务端口
```shell
# 查看 ks-console 服务的端口  默认为 NodePort: 30880 默认的集群管理员账号为 admin/P@88w0rd
kubectl get svc -n kubesphere-system
```

## 参考
* https://kubesphere.io/docs/v2.0/zh-CN/installation/install-on-k8s/

