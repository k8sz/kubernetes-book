# Secret

Secret解决了密码、token、密钥等敏感数据的配置问题，而不需要把这些敏感数据暴露到镜像或者Pod Spec中。Secret可以以Volume或者环境变量的方式使用。

Secret有三种类型：

* **Service Account** ：用来访问Kubernetes API，由Kubernetes自动创建，并且会自动挂载到Pod的`/run/secrets/kubernetes.io/serviceaccount`目录中；
* **Opaque** ：base64编码格式的Secret，用来存储密码、密钥等；
* **kubernetes.io/dockerconfigjson** ：用来存储私有docker registry的认证信息。

## Opaque Secret

Opaque类型的数据是一个map类型，要求value是base64编码格式：

```sh
$ echo -n "admin" | base64
YWRtaW4=
$ echo -n "1f2d1e2e67df" | base64
MWYyZDFlMmU2N2Rm
```

secrets.yml

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  password: MWYyZDFlMmU2N2Rm
  username: YWRtaW4=
```

接着，就可以创建secret了：`kubectl create -f secrets.yml`。

创建好secret之后，有两种方式来使用它： 

* 以Volume方式
* 以环境变量方式

### 将Secret挂载到Volume中

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    name: db
  name: db
spec:
  volumes:
  - name: secrets
    secret:
      secretName: mysecret
  containers:
  - image: gcr.io/my_project_id/pg:v1
    name: db
    volumeMounts:
    - name: secrets
      mountPath: "/etc/secrets"
      readOnly: true
    ports:
    - name: cp
      containerPort: 5432
      hostPort: 5432
```

### 将Secret导出到环境变量中

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: wordpress-deployment
spec:
  replicas: 2
  strategy:
      type: RollingUpdate
  template:
    metadata:
      labels:
        app: wordpress
        visualize: "true"
    spec:
      containers:
      - name: "wordpress"
        image: "wordpress"
        ports:
        - containerPort: 80
        env:
        - name: WORDPRESS_DB_USER
          valueFrom:
            secretKeyRef:
              name: mysecret
              key: username
        - name: WORDPRESS_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysecret
              key: password
```

## kubernetes.io/dockerconfigjson

可以直接用`kubectl`命令来创建用于docker registry认证的secret：

```sh
$ kubectl create secret docker-registry myregistrykey --docker-server=DOCKER_REGISTRY_SERVER --docker-username=DOCKER_USER --docker-password=DOCKER_PASSWORD --docker-email=DOCKER_EMAIL
secret "myregistrykey" created.
```

也可以直接读取`~/.docker/config.json`的内容来创建：

```sh
$ cat ~/.docker/config.json | base64
$ cat > myregistrykey.yaml <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: myregistrykey
data:
  .dockerconfigjson: UmVhbGx5IHJlYWxseSByZWVlZWVlZWVlZWFhYWFhYWFhYWFhYWFhYWFhYWFhYWFhYWFhYWxsbGxsbGxsbGxsbGxsbGxsbGxsbGxsbGxsbGxsbGx5eXl5eXl5eXl5eXl5eXl5eXl5eSBsbGxsbGxsbGxsbGxsbG9vb29vb29vb29vb29vb29vb29vb29vb29vb25ubm5ubm5ubm5ubm5ubm5ubm5ubm5ubmdnZ2dnZ2dnZ2dnZ2dnZ2dnZ2cgYXV0aCBrZXlzCg==
type: kubernetes.io/dockerconfigjson
EOF
$ kubectl create -f myregistrykey.yaml
```

在创建Pod的时候，通过`imagePullSecrets`来引用刚创建的`myregistrykey`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: foo
spec:
  containers:
    - name: foo
      image: janedoe/awesomeapp:v1
  imagePullSecrets:
    - name: myregistrykey
```

### Service Account

Service Account用来访问Kubernetes API，由Kubernetes自动创建，并且会自动挂载到Pod的`/run/secrets/kubernetes.io/serviceaccount`目录中。

```sh
$ kubectl run nginx --image nginx
deployment "nginx" created
$ kubectl get pods
NAME                     READY     STATUS    RESTARTS   AGE
nginx-3137573019-md1u2   1/1       Running   0          13s
$ kubectl exec nginx-3137573019-md1u2 ls /run/secrets/kubernetes.io/serviceaccount
ca.crt
namespace
token
```

### 文件(TLS证书)

除了配置成环境变量我们在很多地方也会使用到文件的方式来存放密钥信息,最常用的就是HTTPS这样的TLS证书,使用证书程序(比如支付宝证书没法使用环境变量来配置证书)需要一个固定的物理地址去加载这个证书,我们吧之前配置用户名和密码作为文件的方式挂在到某个目录下

首先把所需要的证书挂载到指定的namespace
```sh
kubectl create secret generic alipay --from-file=*.crt --from-file=*.crt --from-file=*.crt -n stating
```

#### 挂载在指定的路径
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret
spec:
  containers:
  - name: alipay
    image: tomcat:1.0.0
    volumeMounts:
    - name: foo
      mountPath: "/home/tls"
      readOnly: true
  volumes:
  - name: foo
    secret:
      secretName: alipay
      defaultMode: 256
```
JSON规范不支持八进制表示法，因此对于0400权限使用值256,0777使用值为511。如果您使用yaml代替pod的json，则可以使用八进制表示法以更自然的方式指定权限,

#### 挂载在指定路径
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret
spec:
  containers:
  - name: alipay
    image: tomcat:1.0.0
    volumeMounts:
    - name: foo
      mountPath: "/home/tls"
      readOnly: true
  volumes:
  - name: alipay
    secret:
      secretName: alipay
      items:
      - key: *.car
        path: my-group/my-username
```
