# 自动化ingress-nginx HTTPS
cert-manager是本地[Kubernetes](https://kubernetes.io/)证书管理控制器。它可以帮助从各种来源颁发证书，例如 [Let's Encrypt](https://letsencrypt.org/)，[HashiCorp Vault](https://www.vaultproject.io/)，[Venafi](https://www.venafi.com/)，简单签名密钥对或自签名

本文主要说明如何在kubernetes安装和设置cert-manager。

## HELM和Inagess-nginx
```bash
helm安装请参考[kubernetes之Helm安装](https://www.k8sz.com/post/kuberneteshelm/)
# ingress-nginx
helm install -n nginx-ingress --namespace kube-system stable/nginx-ingress
```

## 配置
在使用的时候我们需要配置一个缺省的[cluster issuer](http://docs.cert-manager.io/en/latest/tasks/issuing-certificates/ingress-shim.html)，当部署Cert manager的时候，用于支持kubernetes.io/tls-acme: "true"annotation 来自动化 TLS：
```bash
--set ingressShim.defaultIssuerName=letsencrypt-prod
--set ingressShim.defaultIssuerKind=ClusterIssuer
```

## 安装
```bash
# 安装CRD资源
kubectl apply -f https://raw.githubusercontent.com/jetstack/cert-manager/release-0.11/deploy/manifests/00-crds.yaml
# 禁用资源验证
kubectl label namespace kube-system certmanager.k8s.io/disable-validation=true
# 增加helm repo
helm repo add jetstack https://charts.jetstack.io
# 更新helm repo
helm repo update
# 安装
[root@master01 tmp]# helm install --name cert-manager --namespace kube-system --set ingressShim.defaultIssuerName=letsencrypt-prod --set ingressShim.defaultIssuerKind=ClusterIssuer jetstack/cert-manager --version v0.11.0
NAME:   cert-manager
LAST DEPLOYED: Tue Oct  8 13:42:00 2019
NAMESPACE: kube-system
STATUS: DEPLOYED

RESOURCES:
==> v1/ClusterRole
NAME                                    AGE
cert-manager-edit                       1s
cert-manager-view                       1s
cert-manager-webhook:webhook-requester  1s

==> v1/Deployment
NAME                     READY  UP-TO-DATE  AVAILABLE  AGE
cert-manager             0/1    1           0          1s
cert-manager-cainjector  0/1    1           0          1s
cert-manager-webhook     0/1    1           0          1s

==> v1/Pod(related)
NAME                                      READY  STATUS             RESTARTS  AGE
cert-manager-7b867c7d5-jd7mw              0/1    ContainerCreating  0         1s
cert-manager-cainjector-57f6bd5577-zfhqn  0/1    ContainerCreating  0         1s
cert-manager-webhook-66df9566cb-2rx82     0/1    ContainerCreating  0         1s

==> v1/Service
NAME                  TYPE       CLUSTER-IP     EXTERNAL-IP  PORT(S)   AGE
cert-manager          ClusterIP  10.88.165.143  <none>       9402/TCP  1s
cert-manager-webhook  ClusterIP  10.88.59.130   <none>       443/TCP   1s

==> v1/ServiceAccount
NAME                     SECRETS  AGE
cert-manager             1        1s
cert-manager-cainjector  1        1s
cert-manager-webhook     1        1s

==> v1beta1/APIService
NAME                                AGE
v1beta1.webhook.certmanager.k8s.io  1s

==> v1beta1/ClusterRole
NAME                                    AGE
cert-manager-cainjector                 1s
cert-manager-controller-certificates    1s
cert-manager-controller-challenges      1s
cert-manager-controller-clusterissuers  1s
cert-manager-controller-ingress-shim    1s
cert-manager-controller-issuers         1s
cert-manager-controller-orders          1s
cert-manager-leaderelection             1s

==> v1beta1/ClusterRoleBinding
NAME                                    AGE
cert-manager-cainjector                 1s
cert-manager-controller-certificates    1s
cert-manager-controller-challenges      1s
cert-manager-controller-clusterissuers  1s
cert-manager-controller-ingress-shim    1s
cert-manager-controller-issuers         1s
cert-manager-controller-orders          1s
cert-manager-leaderelection             1s
cert-manager-webhook:auth-delegator     1s

==> v1beta1/MutatingWebhookConfiguration
NAME                  AGE
cert-manager-webhook  1s

==> v1beta1/RoleBinding
NAME                                                AGE
cert-manager-webhook:webhook-authentication-reader  1s

==> v1beta1/ValidatingWebhookConfiguration
NAME                  AGE
cert-manager-webhook  1s


NOTES:
cert-manager has been deployed successfully!

In order to begin issuing certificates, you will need to set up a ClusterIssuer
or Issuer resource (for example, by creating a 'letsencrypt-staging' issuer).

More information on the different types of issuers and how to configure them
can be found in our documentation:

https://docs.cert-manager.io/en/latest/reference/issuers.html

For information on how to configure cert-manager to automatically provision
Certificates for Ingress resources, take a look at the `ingress-shim`
documentation:

https://docs.cert-manager.io/en/latest/reference/ingress-shim.html
```

## 检查pod状态
```bash
[root@master01 tmp]# kubectl get pod -n kube-system --selector=app=cert-manager
NAME                           READY   STATUS    RESTARTS   AGE
cert-manager-7b867c7d5-jd7mw   1/1     Running   0          91s
```

## 创建证书签发服务
```bash
vim issuer.yaml
apiVersion: cert-manager.io/v1alpha2
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: 512251296@qq.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
```

## 创建这个ClusterIssuer资源：
```bash
[root@master01 tmp]# kubectl apply -f issuer.yaml 
clusterissuer.certmanager.k8s.io/letsencrypt-prod created
[root@master01 tmp]# kubectl get clusterissuer
NAME               AGE
letsencrypt-prod   2m49s
```

## 在ingress中使用cert-manager
```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
    kubernetes.io/tls-acme: "true"
  name: gitlab
  namespace: kube-ops
spec:
  tls:
  - hosts:
    - git.zb.vip
    secretName: gitlab
  rules:
  - host: git.zb.vip
    http:
      paths:
      - backend:
          serviceName: gitlab
          servicePort: http
```

## 参考
* http://docs.cert-manager.io/en/latest/getting-started/install/kubernetes.html
* https://www.qikqiak.com/post/automatic-kubernetes-ingress-https-with-lets-encrypt/