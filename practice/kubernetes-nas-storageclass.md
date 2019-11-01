# kubernetes挂载阿里云nas存储作为StorageClass

Kubernetes集群管理员通过提供不同的存储类，可以满足用户不同的服务质量级别、备份策略和任意策略要求的存储需求。动态存储卷供应使用StorageClass进行实现，其允许存储卷按需被创建。如果没有动态存储供应，Kubernetes集群的管理员将不得不通过手工的方式类创建新的存储卷。通过动态存储卷，Kubernetes将能够按照用户的需要，自动创建其需要的存储。

## 新建SA，并进行RBAC
```YAML
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-admin
  namespace: kube-system
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nfs-admin
  namespace: kube-system
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "update", "patch"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nfs-admin
  namespace: kube-system
subjects:
  - kind: ServiceAccount
    name: nfs-admin
    namespace: kube-system
roleRef:
  kind: ClusterRole
  name: nfs-admin
  apiGroup: rbac.authorization.k8s.io
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-provisioner
  namespace: kube-system
rules:
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-provisioner
  namespace: kube-system
subjects:
  - kind: ServiceAccount
    name: nfs-admin
    namespace: kube-system
roleRef:
  kind: Role
  name: leader-locking-nfs-provisioner
  apiGroup: rbac.authorization.k8s.io
```

## 使用deployment挂载nas，设置为StorageClass
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: alicloud-disk-available
  namespace: kube-system
provisioner: example.com/nfs
reclaimPolicy: Retain
---
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: alicloud-nas-controller
  namespace: kube-system
spec:
  replicas: 1
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: alicloud-nas-controller
    spec:
      tolerations:
      - effect: NoSchedule
        operator: Exists
        key: node-role.kubernetes.io/master
      - effect: NoSchedule
        operator: Exists
        key: node.cloudprovider.kubernetes.io/uninitialized
      nodeSelector:
        node-role.kubernetes.io/master: ""
      serviceAccount: nfs-admin
      containers:
        - name: alicloud-nas-controller
          image: gmoney23/nfs-client-provisioner:1.0
          volumeMounts:
          - mountPath: /persistentvolumes
            name: nfs-client-root
          env:
            - name: PROVISIONER_NAME
              value: example.com/nfs
            - name: NFS_SERVER
              value: 2f71e4829f-tlh16.cn-shenzhen.nas.aliyuncs.com
            - name: NFS_PATH
              value: /
      volumes:
      - name: nfs-client-root
        nfs:
            path: /
            server: 2f71e4829f-tlh16.cn-shenzhen.nas.aliyuncs.com
```

## PVC连接StorageClass
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  labels: 
    app: jenkins
  name: jenkins
  namespace: jx
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests: 
      storage: 30Gi
  storageClassName: alicloud-disk-available
```

## NAS插件安装
```bash
# Nodes 无法挂载nas，需要安装nas插件
yum install nfs-common  nfs-utils -y
```