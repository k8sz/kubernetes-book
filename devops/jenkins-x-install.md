# jenkins-x安装

## 安装git

 jx 安装要求git版本最低位2.15.0，而在centos7.5使用yum源安装的git版本为1.8.x，git版本低，会出现jx安装过程中使用到的githun 仓库无法正常clone。 

### git依赖安装

```bash
yum -y install curl-devel expat-devel gettext-devel openssl-devel zlib-devel gcc perl-ExtUtils-MakeMaker
```

### 下载git源码并编译安装

```bash
wget https://mirrors.edge.kernel.org/pub/software/scm/git/git-2.23.0.tar.xz
tar xf git-2.23.0.tar.xz
cd git-2.23.0
./configure
make && make install
```

### 创建软连接

```bash
cd /usr/bin
rm -f git
ln -s  /usr/local/git/bin/git git
```

## 安装jenkins-x（精简版）

根据需求增减所需插件，jenkins-x对github支持比较好，但目前国内主要用的git仓库为自建gitlab，并且很多服务前期已经搭建完成比如，nexus，gitlab

### jx命令行安装

```bash
wget https://github.com/jenkins-x/jx/releases/download/v2.0.696/jx-linux-amd64.tar.gz
tar xf jx-linux-amd64.tar.gz  -C ~/.jx/bin
export PATH=$PATH:~/.jx/bin
echo 'export PATH=$PATH:~/.jx/bin' >> ~/.bashrc
```

### jx系统安装
```bash
jx install --on-premise --cloud-environment-repo='https://github.com/linjunjianbaba/cloud-environments.git' --provider=kubernetes --domain='domain.com' --no-default-environments --static-jenkins --git-provider-url='https://github.com' --git-username='githubname' --git-api-token='xxxxxxxxxxxxxxxxxxxxxxxxxxxxx' --ingress-service=nginx-ingress
```

### 系统镜像修改
由于有一部分镜像服务器在国外，国内需要科学上网才能下载，也可以直接修改镜像服务器
```bash
gcr.io         对应国内 gcr.azk8s.cn
k8s.gcr.io     对应国内 mirrorgooglecontainers
#如：image: gcr.azk8s.cn/jenkinsxio/jenkinsx:0.0.80
```
安装完成后终端会打印对应的密码，也可以在~/.jx/ 下查看

### 在K8S上添加gitlab token
jenkins需要到gitlab上拉取代码，需要凭证，将gitab 的用户和token直接挂载在kubernetes sectet上即可
```yaml
apiVersion: v1
data:
  password: YTdBZHE2TWlBS1RaWF
  username: bGluemhpYmlhbwo
kind: Secret
metadata:
  annotations:
    jenkins.io/credentials-description: API Token for acccessing http://gitlab.com
      Git service inside pipelines
    jenkins.io/name: gitlab
    jenkins.io/url: http://gitlab.com
  labels:
    jenkins.io/created-by: jx
    jenkins.io/credentials-type: usernamePassword
    jenkins.io/kind: git
    jenkins.io/service-kind: gitlab
  name: jx-pipeline-git-gitlab-gitlab
  namespace: jx
type: Opaque
```

当然如果你的代码仓库使用了github可以使用jx的原生功能，这样更符合gitops

## 参考

* https://jenkins-x.io/docs/managing-jx/old/install-on-cluster/