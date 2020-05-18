---
layout: istio实践
title: istio实践 - 示例（BookInfo）
date: 2019-03-26 10:35:24
tags:
  - Istio
  - service mesh
categories: 
  - Istio
---
### 部署应用(在 Kubernetes 中运行)
#### 进入 Istio 安装目录。
```
cd /Users/sipeng.zhu/work_tools/istio/istio-1.1.0
```
#### 启动应用容器(集群用的是手工 Sidecar 注入)：
```
$ kubectl apply -f <(istioctl kube-inject -f samples/bookinfo/platform/kube/bookinfo.yaml)
```
#### 确认所有的服务和 Pod 都已经正确的定义和启动：
```
$ kubectl get services
NAME          TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
details       ClusterIP   10.103.79.50     <none>        9080/TCP   65m
kubernetes    ClusterIP   10.96.0.1        <none>        443/TCP    5d21h
productpage   ClusterIP   10.106.117.25    <none>        9080/TCP   65m
ratings       ClusterIP   10.102.201.140   <none>        9080/TCP   65m
reviews       ClusterIP   10.106.229.52    <none>        9080/TCP   65m
```
```
$ kubectl get pods
NAME                              READY   STATUS    RESTARTS   AGE
details-v1-8584b997b5-z9nlb       2/2     Running   0          65m
hello-minikube-6fd785d459-s5wkw   1/1     Running   2          5d4h
productpage-v1-7c54588b59-t5h8v   2/2     Running   0          65m
ratings-v1-857d476c65-q5jjj       2/2     Running   0          65m
reviews-v1-7746774bd6-hmf2x       2/2     Running   0          65m
reviews-v2-7c687fc54c-mrqms       2/2     Running   0          65m
reviews-v3-8565479856-j9x8m       2/2     Running   0          65m
```
<!-- more -->
#### 确认 Bookinfo 应用程序正在运行，请通过某个 pod 中的 curl 命令向其发送请求，例如来自 ratings
```
$ kubectl exec -it $(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}') -c ratings -- curl productpage:9080/productpage | grep -o "<title>.*</title>"
<title>Simple Bookstore App</title>
```
### 确定 Ingress 的 IP 和端口
#### 为应用定义入口网关
```
$ kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml
```
#### 确认网关创建完成：
```
$ kubectl get gateway
NAME               AGE
bookinfo-gateway   32m
```
#### 设置访问网关的 INGRESS_HOST 和 INGRESS_PORT 变量。
```
 $ export INGRESS_HOST=$(minikube ip)
 $ export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')
 $ export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].nodePort}')
 $ export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT
```
#### 确认设置
```
⇒  echo $GATEWAY_URL
192.168.99.100:31380
```
#### 验证结果
> Istio Bookinfo 示例包含四个独立的微服务，每个微服务都有多个版本。 其中一个微服务 reviews 的三个不同版本已经部署并同时运行。 为了说明这导致的问题，在浏览器中访问 Bookinfo 应用程序的 /productpage 并刷新几次。 您会注意到，有时书评的输出包含星级评分，有时则不包含。 这是因为没有明确的默认服务版本路由，Istio 将以循环方式请求路由到所有可用版本。
```
http://192.168.99.100:31380/productpage
```

    

