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

### 修改