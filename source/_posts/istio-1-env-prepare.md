---
layout: istio实践
title: istio实践 - 环境准备
date: 2019-03-25 10:35:24
tags:
  - Istio
  - service mesh
categories: 
  - Istio
---
### 下载&安装 Istio 发布包
#### 下载和自动解压缩istio安装文件

```
$ curl -L https://git.io/getLatestIstio | ISTIO_VERSION=1.1.0 sh -
...

  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:--  0:00:01 --:--:--     0
100  1631  100  1631    0     0    599      0  0:00:02  0:00:02 --:--:--  3592
Downloading istio-1.1.0 from https://github.com/istio/istio/releases/download/1.1.0/istio-1.1.0-osx.tar.gz ...
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   612    0   612    0     0    425      0 --:--:--  0:00:01 --:--:--   425
100 14.8M  100 14.8M    0     0   326k      0  0:00:46  0:00:46 --:--:--  573k
Downloaded into istio-1.1.0:
LICENSE		README.md	bin		install		istio.VERSION	samples		tools
Add /Users/sipeng.zhu/work_tools/istio/istio-1.1.0/bin to your path; e.g copy paste in your shell and/or ~/.profile:
export PATH="$PATH:/Users/sipeng.zhu/work_tools/istio/istio-1.1.0/bin"
```
<!-- more -->

#### 进入 Istio 包目录。例如，假设这个包是 istio-1.1.0.0：
```
$ cd istio-1.1.0
```
#### 安装目录中包含：

> - 在 install/ 目录中包含了 Kubernetes 安装所需的 .yaml 文件
> - samples/ 目录中是示例应用
> - istioctl 客户端文件保存在 bin/ 目录之中。istioctl 的功能是手工进行 Envoy Sidecar 的注入。
> - istio.VERSION 配置文件

#### 把 istioctl 客户端加入 PATH 环境变量，
```
$ export PATH=$PWD/bin:$PATH
```
### 使用homebrew安装kubectl
```
$ brew install kubernetes-cli
$ kubectl version
Client Version: version.Info{Major:"1", Minor:"13", GitVersion:"v1.13.4", GitCommit:"c27b913fddd1a6c480c229191a087698aa92f0b1", GitTreeState:"clean", BuildDate:"2019-03-01T23:36:43Z", GoVersion:"go1.12", Compiler:"gc", Platform:"darwin/amd64"}
Server Version: version.Info{Major:"1", Minor:"13", GitVersion:"v1.13.4", GitCommit:"c27b913fddd1a6c480c229191a087698aa92f0b1", GitTreeState:"clean", BuildDate:"2019-02-28T13:30:26Z", GoVersion:"go1.11.5", Compiler:"gc", Platform:"linux/amd64"}
```

### 安装virtualbox
https://www.virtualbox.org/wiki/Downloads
### 安装Minikube
#### 使用homebrew安
```
$ brew cask install minikube
...
🍺  minikube was successfully installed!
```
#### 启动minikube
```
$ minikube start --memory=8192 --cpus=4 --kubernetes-version=v1.13.0 \
    --vm-driver=`VirtualBox`
```
```
😄  minikube v0.35.0 on darwin (amd64)
💡  Tip: Use 'minikube start -p <name>' to create a new cluster, or 'minikube delete' to delete this one.
🔄  Restarting existing virtualbox VM for "minikube" ...
⌛  Waiting for SSH access ...
📶  "minikube" IP address is 192.168.99.100
🐳  Configuring Docker as the container runtime ...
✨  Preparing Kubernetes environment ...
🚜  Pulling images required by Kubernetes v1.13.4 ...
🔄  Relaunching Kubernetes v1.13.4 using kubeadm ...
⌛  Waiting for pods: apiserver proxy etcd scheduler controller addon-manager dns
📯  Updating kube-proxy configuration ...
🤔  Verifying component health ......
💗  kubectl is now configured to use "minikube"
🏄  Done! Thank you for using minikube!
```
#### 查看minikube状态
```
$ minikube status
...
host: Running
kubelet: Running
apiserver: Running
kubectl: Correctly Configured: pointing to minikube-vm at 192.168.99.100
```

### 在 Kubernetes 中快速开始
#### 使用 kubectl apply 安装 Istio 的自定义资源定义（CRD）
```
$ for i in install/kubernetes/helm/istio-init/files/crd*yaml; do kubectl apply -f $i; done
```

#### 使用 mutual TLS 的宽容模式安装
```
$ kubectl apply -f install/kubernetes/istio-demo.yaml
```
### 确认部署结果
#### 确认下列 Kubernetes 服务已经部署并都具有各自的 CLUSTER-IP：

```
$ kubectl get svc -n istio-system
```
```
NAME                     TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                                                                                                                                      AGE
grafana                  ClusterIP      10.104.12.220    <none>        3000/TCP                                                                                                                                     17h
istio-citadel            ClusterIP      10.102.110.82    <none>        8060/TCP,15014/TCP                                                                                                                           17h
istio-egressgateway      ClusterIP      10.105.70.166    <none>        80/TCP,443/TCP,15443/TCP                                                                                                                     17h
istio-galley             ClusterIP      10.97.151.94     <none>        443/TCP,15014/TCP,9901/TCP                                                                                                                   17h
istio-ingressgateway     LoadBalancer   10.104.89.245    <pending>     80:31380/TCP,443:31390/TCP,31400:31400/TCP,15029:31225/TCP,15030:30228/TCP,15031:31039/TCP,15032:30667/TCP,15443:31626/TCP,15020:30118/TCP   17h
istio-pilot              ClusterIP      10.102.83.181    <none>        15010/TCP,15011/TCP,8080/TCP,15014/TCP                                                                                                       17h
istio-policy             ClusterIP      10.98.102.117    <none>        9091/TCP,15004/TCP,15014/TCP                                                                                                                 17h
istio-sidecar-injector   ClusterIP      10.109.199.161   <none>        443/TCP                                                                                                                                      17h
istio-telemetry          ClusterIP      10.101.8.184     <none>        9091/TCP,15004/TCP,15014/TCP,42422/TCP
```
> 因为我们使用的集群是Minikube （没有外部负载均衡器支持的环境中运行），istio-ingressgateway 的 EXTERNAL-IP 会是 <pending>。要访问这个网关，只能通过服务的 NodePort 或者使用端口转发来进行访问。
```
minikube service -n istio-system istio-ingressgateway
```

#### 确认必要的 Kubernetes Pod 都已经创建并且其 STATUS 的值是 Running：
```
$ kubectl get pods -n istio-system
```
```
NAME                                      READY   STATUS      RESTARTS   AGE
grafana-57586c685b-9rnfb                  1/1     Running     1          11m
istio-citadel-645ffc4999-tlpx2            1/1     Running     1          11m
istio-cleanup-secrets-1.1.0-d2nr4         0/1     Completed   0          11m
istio-cleanup-secrets-9rjtg               0/1     Completed   0          33m
istio-egressgateway-5c7fd57fdb-ctx2l      1/1     Running     1          11m
istio-galley-978f9447f-sg7ls              1/1     Running     2          11m
istio-grafana-post-install-1.1.0-7p49p    0/1     Completed   0          11m
istio-ingressgateway-8ccdc79bc-jbvz5      1/1     Running     1          11m
istio-pilot-649455846-k6v49               2/2     Running     0          11m
istio-policy-7b7d7f644b-lmkb5             2/2     Running     4          11m
istio-security-post-install-1.1.0-qxqs6   0/1     Completed   0          11m
istio-sidecar-injector-6dcc9d5c64-gctbh   1/1     Running     2          11m
istio-telemetry-6d494cd676-7phwq          2/2     Running     4          11m
istio-tracing-656f9fc99c-fm8ft            1/1     Running     1          11m
kiali-69d6978b45-69g7d                    1/1     Running     1          11m
prometheus-66c9f5694-l6qn8                1/1     Running     1          11m
servicegraph-cb9b94c-qn5mk                1/1     Running     12         33m
```


