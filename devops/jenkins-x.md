# jenkins-x简介

Jenkins X 是一个高度集成化的CI/CD平台，基于Jenkins和Kubernetes实现，旨在解决微服务体系架构下的云原生应用的持续交付的问题，简化整个云原生应用的开发、运行和部署过程。

## gitops
jx 基于gitops，将k8s分为preview、staging、production几个环境
![gitops](https://www.k8sz.com/img/gitops.png)

## devops
详细的devops可以查看下图
![devops](https://www.k8sz.com/img/overview.png)

## 相关组件
### jenkins pipeline
jx使用Jenkins Pipeline来执行CI流程，Jenkins Pipeline是jenkins的一套插件，支持将连续输送Pipeline实施和整合到Jenkins。Pipeline 提供了一组可扩展的工具，用于通过Pipeline DSL为代码创建简单到复杂的传送Pipeline 。

### helm与charts
Helm是管理Kubernetes charts的工具，charts是预先配置好的安装包资源，有点类似于Ubuntu的APT和CentOS中的yum。
可以使用helm来：

* 查找并使用已打包为Helm charts的热门应用在Kubernetes中运行
* 封装并分享自己的应用
* 创建可重复的Kubernetes应用程序版本
* 智能管理应用依赖
* 管理Helm软件包的版本

### skaffold
Skaffold 是谷歌开源的简化本地 Kubernetes 应用开发的工具。它将构建镜像、推送镜像以及部署 Kubernetes 服务等流程自动化，可以方便地对 Kubernetes 应用进行持续开发。其功能特点包括

* 没有服务器组件
* 自动检测代码更改并自动构建、推送和部署服务
* 自动管理镜像标签
* 支持已有工作流
* 保存文件即部署

![skaffold](https://www.k8sz.com/img/skaffold.png)

### Draft
[draft](https://github.com/Azure/draft) 是微软开源的“A tool for developers to create cloud-native applications on Kubernetes”，一个为方便开发者在K8S创建云原生应用的工具，它可以帮助开发人员简化容器应用程序的开发流程。

上面我们了解了JENKINSFile，charts配置文件，难道每个项目需要按我们自己来写这些配置文件吗？
Draft告诉你，可以不！Draft最大的益处是，可以自动识别你的工程，然后根据模板库生成对应的配置文件，酷不酷？

* Draft 主要由三个命令组成
* draft init：初始化 docker registry 账号，并在 Kubernetes 集群中部署 draftd（负责镜像构建、将镜像推送到 docker registry 以及部署应用等）
* draft create：draft 根据 packs 检测应用的开发语言，并自动生成 Dockerfile 和 Kubernetes Helm Charts
* draft up：根据 Dockfile 构建镜像，并使用 Helm 将应用部署到 Kubernetes 集群（支持本地或远端集群）。同时，还会在本地启动一个 draft client，监控代码变化，并将更新过的代码推送给 draftd。

不过，在jx中，仅仅只使用了draft的识别语言，生成配置文件的功能，相关的draft模板可以在 [draft-packs](https://github.com/jenkins-x/draft-packs) 里看到。

### Nexus
jx使用Nexus 来做默认的制品仓库(Artifact repository),Nexus大家应该不默认，好多公司和团队的maven仓库均是通过Nexus搭建的。

Nexus还可以作为npm，nuget，docker仓库。

### Chartmuseum 与Monocular
Chartmuseum - 是一个helm chart仓库，jx用他来做chart仓库。

Monocular是一个web应用可以用来从helm charts仓库搜索和发现charts。

## 参考

*  https://www.k8sz.com/post/kubernetes-jx-component/ 