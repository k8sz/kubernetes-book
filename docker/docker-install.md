# Docker安装

## 在centos系统上安装Docker

### 安装依赖包
```bash
sudo yum install -y yum-utils device-mapper-persistent-data lvm2
```
### 添加软件仓库
```bash
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```
### 将服务器上的软件包信息 现在本地缓存,以提高 搜索 安装软件的速度
```bash
yum makecache fast
```
### 列出Docker版本
```bash
yum list docker-ce --showduplicates | sort -r

```
### 安装指定Docker指定版本
```bash
yum -y install docker-ce-18.09.8-3
```
### 修改Docker启动参数
```bash
sed -i "13i ExecStartPost=/usr/sbin/iptables -P FORWARD ACCEPT" /usr/lib/systemd/system/docker.service
mkdir -p /etc/docker
cat > /etc/docker/daemon.json <<EOF
{
"exec-opts": ["native.cgroupdriver=systemd"],
"log-driver": "json-file",
"log-opts": {
"max-size": "100m"
},
"storage-driver": "overlay2",
"registry-mirrors": ["https://uyah70su.mirror.aliyuncs.com"]
}
EOF
```
### 设置Docker开机启动
```bash
systemctl daemon-reload
systemctl enable docker
systemctl start docker
```
