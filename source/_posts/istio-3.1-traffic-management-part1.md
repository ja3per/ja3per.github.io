---
layout: istio实践
title: istio实践 - 流量管理 - part1
date: 2019-03-27 15:35:24
tags:
  - Istio
  - service mesh
categories: 
  - Istio
---
### 配置请求路由
#### 规则配置
Istio 中包含有四种流量管理配置资源，分别是 VirtualService、DestinationRule、ServiceEntry 以及 Gateway。下面会讲一下这几个资源的一些重点。在网络参考中可以获得更多这方面的信息。
> 
> - **VirtualService** 在 Istio 服务网格中定义路由规则，控制路由如何路由到服务上。
> 
> - **DestinationRule** 是 VirtualService 路由生效后，配置应用与请求的策略集。
> 
> - **ServiceEntry** 是通常用于在 Istio 服务网格之外启用对服务的请求。
> 
> - **Gateway** 为 HTTP/TCP 流量配置负载均衡器，最常见的是在网格的边缘的操作，以启用应用程序的入口流量。

#### 应用 virtual service
运行以下命令以应用 virtual service, 将流量路由到一个版本(v1)的服务。：
```
kubectl apply -f samples/bookinfo/networking/virtual-service-all-v1.yaml

virtualservice.networking.istio.io/productpage created
virtualservice.networking.istio.io/reviews created
virtualservice.networking.istio.io/ratings created
virtualservice.networking.istio.io/details created
```
#### 测试新的路由配置：
在浏览器中打开 Bookinfo 站点。 URL 为 http://$GATEWAY_URL/productpage，其中 GATEWAY_URL 是外部的入口 IP 地址，如 Bookinfo 文档中所述。

请注意，无论您刷新多少次，页面的评论部分都不会显示评级星标。这是因为您将 Istio 配置为将评论服务的所有流量路由到版本 reviews:v1，并且此版本的服务不访问星级评分服务。
<!-- more -->
#### 基于用户身份的路由
更改路由配置，将来自特定用户的所有流量路由到特定服务版本。
在这种情况下，来自名为 Jason 的用户的所有流量将被路由到服务 reviews:v2（包含星级评分功能的版本）。
1. 运行以下命令以启用基于用户的路由：
```
kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-test-v2.yaml

virtualservice.networking.istio.io/reviews configured
```
2. 确认规则已创建：
```
kubectl get virtualservice reviews -o yaml

 apiVersion: networking.istio.io/v1alpha3
    kind: VirtualService
    metadata:
      name: reviews
      ...
    spec:
      hosts:
      - reviews
      http:
      - match:
        - headers:
            end-user:
              exact: jason
        route:
        - destination:
            host: reviews
            subset: v2
      - route:
        - destination:
            host: reviews
            subset: v1
```
3. 刷新Bookinfo 站点， 以jason用户登录，发现星级评分显示在每个评论旁边。
4. 点击右上角的sign out，发现星级评分消失了。

#### 理解原理
在此任务中，您首先使用 Istio 将 100% 的请求流量都路由到了 Bookinfo 服务的 v1 版本。 然后再设置了一条路由规则，在 productpage 服务中添加了路由规则，这一规则根据请求的 end-user 自定义 header 内容，选择性地将特定的流量路由到了 reviews 服务的 v2 版本。
#### 清除
删除应用程序 virtual service。
```
kubectl delete -f samples/bookinfo/networking/virtual-service-all-v1.yaml
```

### 故障注入
#### 使用 HTTP 延迟进行故障注入
1. 创建故障注入规则以延迟来自用户 jason（我们的测试用户）的流量
```
kubectl apply -f samples/bookinfo/networking/virtual-service-ratings-test-delay.yaml

virtualservice.networking.istio.io/ratings configured

```
2. 确认已创建规则：
```
kubectl get virtualservice ratings -o yaml

   apiVersion: networking.istio.io/v1alpha3
    kind: VirtualService
    metadata:
      name: ratings
      ...
    spec:
      hosts:
      - ratings
      http:
      - fault:
          delay:
            fixedDelay: 7s
            percent: 100
        match:
        - headers:
            end-user:
              exact: jason
        route:
        - destination:
            host: ratings
            subset: v1
      - route:
        - destination:
            host: ratings
            subset: v1
```
3. 验证
使用用户 jason 登陆到 /productpage 界面
>Bookinfo 主页在大约 7 秒钟加载完成并且没有错误。但是，出现了一个问题，Reviews 部分显示了错误消息。在微服务中有硬编码超时，导致 reviews 服务失败。
```
Error fetching product reviews!
    Sorry, product reviews are currently unavailable for this book.
```

#### 使用 HTTP abort 进行故障注入
1. 为用户 jason 创建故障注入规则发送 HTTP abort
```
kubectl apply -f samples/bookinfo/networking/virtual-service-ratings-test-abort.yaml
```
2. 确认已创建规则
```
kubectl get virtualservice ratings -o yaml

    apiVersion: networking.istio.io/v1alpha3
    kind: VirtualService
    metadata:
      name: ratings
      ...
    spec:
      hosts:
      - ratings
      http:
      - fault:
          abort:
            httpStatus: 500
            percent: 100
        match:
        - headers:
            end-user:
              exact: jason
        route:
        - destination:
            host: ratings
            subset: v1
      - route:
        - destination:
            host: ratings
            subset: v1
```
3. 验证abort配置
  >使用用户 jason 登陆到 /productpage 界面
  页面立即加载并看到 Ratings service is currently unavailable 消息

### 流量转移
#### 应用基于权重的路由
1. 首先，运行此命令将所有流量路由到 v1 版本的各个微服务。
```
kubectl apply -f samples/bookinfo/networking/virtual-service-all-v1.yaml

virtualservice.networking.istio.io/productpage unchanged
virtualservice.networking.istio.io/reviews configured
virtualservice.networking.istio.io/ratings configured
virtualservice.networking.istio.io/details unchanged
```
2. 使用下面的命令把 50% 的流量从 reviews:v1 转移到 reviews:v3：
```
kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-50-v3.yaml

virtualservice.networking.istio.io/reviews configured
```
3. 确认规则
```
kubectl get virtualservice reviews -o yaml
 apiVersion: networking.istio.io/v1alpha3
    kind: VirtualService
    metadata:
      name: reviews
      ...
    spec:
      hosts:
      - reviews
      http:
      - route:
        - destination:
            host: reviews
            subset: v1
          weight: 50
        - destination:
            host: reviews
            subset: v3
          weight: 50
```
4. 验证
刷新浏览器中的 /productpage 页面，大约有 50% 的几率会看到页面中出带红色星级的评价内容
5. 全流量转移
如果您认为 reviews:v3 微服务已经稳定，你可以通过应用此 virtual service 将 100% 的流量路由到 reviews:v3：
```
kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-v3.yaml

virtualservice.networking.istio.io/reviews configured
```
6. 验证
多次刷新 /productpage 页面，始终看到带有红色星级评分的书评。

### 控制 Ingress 流量
配置 Istio 以使用 Istio 在服务网格外部公开服务 Gateway。
#### 启动 httpbin 样本，该样本将用作要在外部公开的目标服务。
手动注入 Sidecar：
```
⇒  kubectl apply -f samples/httpbin/httpbin.yaml
service/httpbin created
```
#### 使用 Istio 网关配置 Ingress
Ingress Gateway 描述了在网格边缘操作的负载平衡器，用于接收传入的 HTTP/TCP 连接。它配置暴露的端口、协议等，但与 Kubernetes Ingress Resources 不同，它不包括任何流量路由配置。流入流量的流量路由使用 Istio 路由规则进行配置，与内部服务请求完全相同。
1. 创建一个 Istio Gateway：
```
⇒  kubectl apply -f - <<EOF
    apiVersion: networking.istio.io/v1alpha3
    kind: Gateway
    metadata:
      name: httpbin-gateway
    spec:
      selector:
        istio: ingressgateway # use Istio default gateway implementation
      servers:
      - port:
          number: 80
          name: http
          protocol: HTTP
        hosts:
        - "httpbin.example.com"
heredoc> EOF
gateway.networking.istio.io/httpbin-gateway created
```
2. 为通过 Gateway 进入的流量配置路由：
```
⇒  kubectl apply -f - <<EOF
    apiVersion: networking.istio.io/v1alpha3
    kind: VirtualService
    metadata:
      name: httpbin
    spec:
      hosts:
      - "httpbin.example.com"
      gateways:
      - httpbin-gateway
      http:
      - match:
        - uri:
            prefix: /status
        - uri:
            prefix: /delay
        route:
        - destination:
            port:
              number: 8000
            host: httpbin
heredoc> EOF
virtualservice.networking.istio.io/httpbin created
```
>在这里，我们为服务创建了一个虚拟服务配置 httpbin ，其中包含两条路由规则，允许路径 /status 和 路径的流量 /delay。
>
>该网关列表指定，只有通过我们的要求 httpbin-gateway 是允许的。所有其他外部请求将被拒绝，并返回 404 响应。
>
>请注意，在此配置中，来自网格中其他服务的内部请求不受这些规则约束，而是简单地默认为循环路由。要将这些（或其他规则）应用于内部调用，我们可以将特殊值 mesh 添加到 gateways 的列表中。
3. 使用 curl 访问 httpbin 服务
```
⇒  curl -I -HHost:httpbin.example.com http://$INGRESS_HOST:$INGRESS_PORT/status/200

HTTP/1.1 200 OK
server: istio-envoy
date: Thu, 28 Mar 2019 02:51:29 GMT
content-type: text/html; charset=utf-8
access-control-allow-origin: *
access-control-allow-credentials: true
content-length: 0
x-envoy-upstream-service-time: 2
```
>请注意，这里使用该 -H 标志将 Host HTTP Header 设置为 “httpbin.example.com”。这一操作是必需的，因为上面的 Ingress Gateway 被配置为处理 “httpbin.example.com”，但在测试环境中没有该主机的 DNS 绑定，只是将请求发送到 Ingress IP。
4. 访问任何未明确公开的其他 URL，应该会看到一个 HTTP 404 错误：
```
⇒  curl -I -HHost:httpbin.example.com http://$INGRESS_HOST:$INGRESS_PORT/headers

HTTP/1.1 404 Not Found
date: Thu, 28 Mar 2019 02:52:31 GMT
server: istio-envoy
transfer-encoding: chunked
```
#### 使用浏览器访问 Ingress 服务
在浏览器中输入 httpbin 服务的地址是不会生效的，这是因为因为我们没有办法让浏览器像 curl 一样装作访问 httpbin.example.com。而在现实世界中，因为有正常配置的主机和 DNS 记录，这种做法就能够成功了——只要简单的在浏览器中访问由域名构成的 URL 即可，例如 https://httpbin.example.com/status/200。

要解决此问题以进行简单的测试和演示，我们可以在 Gateway 和 VirtualService 配置中为主机使用通配符值 *。例如，如果我们将 Ingress 配置更改为以下内容：

```
⇒ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: httpbin-gateway
spec:
  selector:
    istio: ingressgateway # use Istio default gateway implementation
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin
spec:
  hosts:
  - "*"
  gateways:
  - httpbin-gateway
  http:
  - match:
    - uri:
        prefix: /headers
    route:
    - destination:
        port:
          number: 8000
        host: httpbin

heredoc> EOF
gateway.networking.istio.io/httpbin-gateway configured
virtualservice.networking.istio.io/httpbin configured
```
接下来就可以在浏览器的 URL 中使$INGRESS_HOST:$INGRESS_PORT（也就是 192.168.99.100:31380）进行访问，输入 http://192.168.99.100:31380/headers 网址之后，应该会显示浏览器发送的请求 Header。
#### 理解
Gateway 配置资源允许外部流量进入 Istio 服务网，并使 Istio 的流量管理和策略功能可用于边缘服务。

