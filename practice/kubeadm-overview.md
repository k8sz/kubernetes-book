# kubeadm概述

Kubeadm是专为提供的工具kubeadm init和kubeadm join最佳实践创建Kubernetes集群“快速通道”。

kubeadm执行必要的操作，以启动和运行最低限度的集群。从设计上讲，它只关心引导程序，而不关心配置机器。同样地，安装各种出色的附加组件（例如Kubernetes仪表板，监视解决方案和特定于云的附加组件）也不在范围之内。

相反，我们希望在kubeadm的基础上构建更高级别，更定制的工具，并且理想情况下，使用kubeadm作为所有部署的基础将使创建一致的集群更加容易。

