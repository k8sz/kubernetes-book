# Dockerfile

## 说明

　　Dockerfile是一个用于编写docker镜像生成过程的文件，其有特定的语法。在一个文件夹中，如果有一个名字为Dockerfile的文件，其内容满足语法要求，在这个文件夹路径下执行命令:docker  build --tag name:tag  .，就可以按照描述构建一个镜像了。name是镜像的名称，tag是镜像的版本或者是标签号，不写就是lastest。注意后面有一个空格和点。

## 语法
　　Dockerfile的基本指令有十三个，分别是：FROM、MAINTAINER、RUN、CMD、EXPOSE、ENV、ADD、COPY、ENTRYPOINT、VOLUME、USER、WORKDIR、ONBUILD
　　
### FROM 
用法： FROM  images
说明：第一个指令必须是FROM了，其指定一个构建镜像的基础源镜像，如果本地没有就会从公共库中拉取，没有指定镜像的标签会使用默认的latest标签，可以出现多次，如果需要在一个Dockerfile中构建多个镜像。

尽可能使用当前官方仓库作为你构建镜像的基础。推荐使用[Alpine](https://hub.docker.com/_/alpine/)镜像，因为它被严格控制并保持最小尺寸（目前小于 5 MB），但它仍然是一个完整的发行版

### LABEL
你可以给镜像添加标签来帮助组织镜像、记录许可信息、辅助自动化构建等。每个标签一行，由 LABEL 开头加上一个或多个标签对。

### MAINTAINER

用法：MAINTAINER name email
说明：描述镜像的创建者，名称和邮箱

### RUN
用法：RUN "command" "param1" "param2"
说明：RUN命令是一个常用的命令，执行完成之后会成为一个新的镜像，这里也是指镜像的分层构建。一句RUN就是一层，也相当于一个版本。这就是之前说的缓存的原理。我们知道docker是镜像层是只读的，所以你如果第一句安装了软件，用完在后面一句删除是不可能的。所以这种情况要在一句RUN命令中完成，可以通过&符号连接多个RUN语句。RUN后面的必须是双引号不能是单引号（没引号貌似也不要紧），command是不会调用shell的，所以也不会继承相应变量，要查看输入RUN  "sh" "-c" "echo" "$HOME"，而不是RUN "echo" "$HOME"。

### CMD
用法：CMD command param1 param2
说明：CMD在Dockerfile中只能出现一次，有多个，只有最后一个会有效。其作用是在启动容器的时候提供一个默认的命令项。如果用户执行docker run的时候提供了命令项，就会覆盖掉这个命令。没提供就会使用构建时的命令。

### EXPOSE
用法：EXPOSE port>[port...]
说明：告诉Docker服务器容器对外映射的容器端口号，在docker run -p的时候生效。

### ENV
用法：EVN key value 只能设置一个 
   EVN key=value允许一次设置多个
说明：设置容器的环境变量，可以让其后面的RUN命令使用，容器运行的时候这个变量也会保留。

### ADD
用法：ADD src   dest
说明：复制本机文件或目录或远程文件，添加到指定的容器目录，支持GO的正则模糊匹配。路径是绝对路径，不存在会自动创建。如果源是一个目录，只会复制目录下的内容，目录本身不会复制。ADD命令会将复制的压缩文件夹自动解压，这也是与COPY命令最大的不同。

### COPY
用法：COPY src dest
说明：COPY除了不能自动解压，也不能复制网络文件。其它功能和ADD相同

### ENTRYPOINT
用法：ENTRYPOINT "command" "param1" "param2"
说明：这个命令和CMD命令一样，唯一的区别是不能被docker run命令的执行命令覆盖，如果要覆盖需要带上选项--entrypoint，如果有多个选项，只有最后一个会生效。

### VOLUME
用法：VOLUME ["path"]
说明：在主机上创建一个挂载，挂载到容器的指定路径。docker run -v命令也能完成这个操作，而且更强大。这个命令不能指定主机的需要挂载到容器的文件夹路径。但docker run -v可以，而且其还可以挂载数据容器。

### USER
用法：USER daemon
说明：指定运行容器时的用户名或UID，后续的RUN、CMD、ENTRYPOINT也会使用指定的用户运行命令。

### WORKDIR
用法:WORKDIR path
说明：为RUN、CMD、ENTRYPOINT指令配置工作目录。可以使用多个WORKDIR指令，后续参数如果是相对路径，则会基于之前的命令指定的路径。如：WORKDIR   /home　　WORKDIR test  。最终的路径就是/home/test。path路径也可以是环境变量，比如有环境变量HOME=/home，WORKDIR  $HOME/test也就是/home/test。

### ONBUILD
用法：ONBUILD [INSTRUCTION]
说明：配置当前所创建的镜像作为其它新创建镜像的基础镜像时，所执行的操作指令。意思就是，这个镜像创建后，如果其它镜像以这个镜像为基础，会先执行这个镜像的ONBUILD命令。

## 例子
```bash
FROM centos
MAINTAINER zb zb@k8sz.com
RUN mkdir -p /opt/jdk/ && mkdir -p /opt/tomcat/ 
ADD jdk /opt/jdk/
ADD tomcat  /opt/tomcat/
ENV CATALINA_HOME /opt/tomcat
ENV JAVA_HOME /opt/jdk
EXPOSE  8080
ENV PATH $PATH:$JAVA_HOME/bin
CMD ["/opt/tomcat/bin/catalina.sh","run"]
```
保存为Dockerfile然后运行
```bash
docker build -t tomcat:1.0.0 .
```
