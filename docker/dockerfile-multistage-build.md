# Dockerfile多阶段构建

docker 的口号是 **Build,Ship,and Run Any App,Anywhere**，在我们使用 Docker 的大部分时候，的确能感觉到其优越性，但是往往在我们 Build 一个应用的时候，是将我们的源代码也构建进去的，这对于类似于 golang 这样的编译型语言肯定是不行的，因为实际运行的时候我只需要把最终构建的二进制包给你就行，把源码也一起打包在镜像中，需要承担很多风险，即使是脚本语言，在构建的时候也可能需要使用到一些上线的工具，这样无疑也增大了我们的镜像体积。

## 多阶段构建
**Docker 17.05**版本以后，官方就提供了一个新的特性：Multi-stage builds（多阶段构建）。 使用多阶段构建，你可以在一个Dockerfile中使用多个 FROM 语句。每个 FROM 指令都可以使用不同的基础镜像，并表示开始一个新的构建阶段。你可以很方便的将一个阶段的文件复制到另外一个阶段，在最终的镜像中保留下你需要的内容即可。

以下已一个beego框架的项目实现Dockerfile多阶段构建。
#### beego基础像的Dockerfile
```yaml
FROM golang:1.12.10

MAINTAINER ZB zb@k8sz.com

RUN go get github.com/astaxie/beego \
&& go get github.com/beego/bee \
&& go get github.com/go-sql-driver/mysql \
&& go get github.com/go-redis/redis \
&& go get gopkg.in/guregu/null.v3 

CMD ["bee","run"]
```
#### 构建beego基础镜像并上传到镜像仓库
```bash
docker build -t k8sz/beego:1.12.10 .
docker login 
docker push k8sz/beego:1.12.10
```

#### 多阶段beego项目的Dockerfile
```yaml
FROM k8sz/beego:1.12.10 AS build

RUN  mkdir -p /go/src/yiigo

WORKDIR /go/src/yiigo

COPY . .

RUN CGO_ENABLED=0 go build -ldflags '-d -w -s'

FROM alpine:latest AS production

MAINTAINER ZB zb@k8sz.com

WORKDIR /root/

COPY --from=build /go/src/yiigo/ ./

EXPOSE 8080

CMD ["./app"]
```
FROM ..AS 后面的是别名
#### 构建beego项目镜像
```bash
docker build -t k8sz/app:1.0.0 .
```
多阶段构建的Dockerfile一般都会放在跟项目代码同一路径，结合CI/CD进行操作。
