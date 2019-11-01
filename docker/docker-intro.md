# Docker

## Docker简介
Docker 和我们传统的虚拟机比较类似，只是更加轻量级，更加方便使用。比如我们要运行一个应用，需要在虚拟机或者物理机上安装系统，并对系统进行初始设置和优化，安装应用的各种依赖，这是一个漫长的过程，需要像孩子一样呵护，生怕会出现各种各样的问题，Docker就不一样了，应用是运行在Docker中，只要有运行了Docker进程的虚拟机或物理机就能运行。

## Docker底层技术支持
Namespaces（做隔离）、CGroups（做资源限制）、UnionFS（镜像和容器的分层） the-underlying-technology Docker 底层架构分析 

