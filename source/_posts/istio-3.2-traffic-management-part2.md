---
layout: istio实践
title: istio实践 - 流量管理 - part2
date: 2019-03-29 15:35:24
tags:
  - Istio
  - service mesh
categories: 
  - Istio
---

### 控制 Egress 流量
默认情况下，Istio 服务网格内的 Pod，由于其 iptables 将所有外发流量都透明的转发给了 Sidecar，所以这些集群内的服务无法访问集群之外的 URL，而只能处理集群内部的目标。

本文的任务描述了如何将外部服务暴露给 Istio 集群中的客户端。你将会学到如何通过定义 ServiceEntry 来调用外部服务；或者简单的对 Istio 进行配置，要求其直接放行对特定 IP 范围的访问。
#### 准备
- 启动 sleep 示例应用，使用这一应用来完成对外部服务的调用过程。
```
⇒   kubectl apply -f samples/sleep/sleep.yaml
serviceaccount/sleep created
service/sleep created
deployment.extensions/sleep created
```
- 将 SOURCE_POD 环境变量设置为已部署的 sleep pod：
```
⇒  export SOURCE_POD=$(kubectl get pod -l app=sleep -o jsonpath={.items..metadata.name})
```
<!-- more -->
#### 在 Istio 中配置外部服务
通过配置 Istio ServiceEntry，可以从 Istio 集群中访问任何可公开访问的服务。 这里我们会使用 httpbin.org 以及 www.google.com 进行试验。
#### 配置外部服务
1. 创建一个 ServiceEntry 对象，放行对一个外部 HTTP 服务的访问：
```
⇒  kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: httpbin-ext
spec:
  hosts:
  - httpbin.org
  ports:
  - number: 80
    name: http
    protocol: HTTP
  resolution: DNS
  location: MESH_EXTERNAL

heredoc> EOF
serviceentry.networking.istio.io/httpbin-ext created
```
2. 创建一个 ServiceEntry 以及 VirtualService，允许访问外部 HTTPS 服务。注意：包括 HTTPS 在内的 TLS 协议，在 ServiceEntry 之外，还需要创建 TLS VirtualService。
```
kubectl exec -it $SOURCE_POD -c sleep sh
```
3. 向外部 HTTP 服务发出请求：
```
curl http://httpbin.org/headers
```
#### 配置外部 HTTPS 服务
1. 创建一个 ServiceEntry 以允许访问外部 HTTPS 服务。 对于 TLS 协议（包括 HTTPS），除了 ServiceEntry 之外，还需要 VirtualService。 没有 VirtualService, ServiceEntry 所暴露的服务将不被定义。VirtualService 必须在 match 子句中包含 tls 规则和 sni_hosts 以启用 SNI 路由
```
⇒ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: google
spec:
  hosts:
  - www.google.com
  ports:
  - number: 443
    name: https
    protocol: HTTPS
  resolution: DNS
  location: MESH_EXTERNAL
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: google
spec:
  hosts:
  - www.google.com
  tls:
  - match:
    - port: 443
      sni_hosts:
      - www.google.com
    route:
    - destination:
        host: www.google.com
        port:
          number: 443
      weight: 100

heredoc> EOF
serviceentry.networking.istio.io/google created
virtualservice.networking.istio.io/google created
```
2. 执行 sleep service 源 pod：
```
kubectl exec -it $SOURCE_POD -c sleep sh
```
3. kubectl exec -it $SOURCE_POD -c sleep sh
```
kubectl exec -it $SOURCE_POD -c sleep sh
```
#### 为外部服务设置路由规则
通过 ServiceEntry 访问外部服务的流量，和网格内流量类似，都可以进行 Istio 路由规则 的配置。下面我们使用 istioctl 为 httpbin.org 服务设置一个超时规则。
1. 在测试 Pod 内部，使用 curl 调用 httpbin.org 这一外部服务的 /delay 端点
```
⇒ kubectl exec -it $SOURCE_POD -c sleep sh
# time curl -o /dev/null -s -w "%{http_code}\n" http://httpbin.org/delay/5
200
real	0m 11.19s
user	0m 0.00s
sys	0m 0.00s
```
2. 退出测试 Pod，使用 kubectl 为 httpbin.org 外部服务的访问设置一个 3 秒钟的超时：
```
⇒  kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin-ext
spec:
  hosts:
    - httpbin.org
  http:
  - timeout: 3s
    route:
      - destination:
          host: httpbin.org
        weight: 100

heredoc> EOF
virtualservice.networking.istio.io/httpbin-ext created
```
3. 等待几秒钟之后，再次发起 curl 请求：
// TODO
```
⇒  kubectl exec -it $SOURCE_POD -c sleep sh
/ # time curl -o /dev/null -s -w "%{http_code}\n" http://httpbin.org/delay/5
200
real	0m 11.13s
user	0m 0.00s
sys	0m 0.00s
```
这一次会在 3 秒钟之后收到一个内容为 504 (Gateway Timeout) 的响应。虽然 httpbin.org 还在等待他的 5 秒钟，Istio 却在 3 秒钟的时候切断了请求。
#### 理解原理
这个任务中，我们使用两种方式从 Istio 服务网格内部来完成对外部服务的调用：

使用 ServiceEntry (推荐方式)

~~配置 Istio sidecar，从它的重定向 IP 表中排除外部服务的 IP 范围~~

第一种方式（ServiceEntry）中，网格内部的服务不论是访问内部还是外部的服务，都可以使用同样的 Istio 服务网格的特性。我们通过为外部服务访问设置超时规则的例子，来证实了这一优势。

### 熔断
本任务展示了用连接、请求以及外部检测来进行熔断配置的过程。
断路器是创建弹性微服务应用程序的重要模式。断路器允许您编写限制故障、延迟峰值以及其他不良网络特性影响的应用程序。
在此任务中，您将配置断路器规则，然后通过故意“跳闸”断路器来测试配置。
1. 准备
启动 httpbin 示例应用，这个应用将会作为本任务的后端服务。在部署 httpbin 应用之前手工注入 Sidecar
```
kubectl apply -f <(istioctl kube-inject -f samples/httpbin/httpbin.yaml)
```
2. 创建一个 目标规则，针对 httpbin 服务设置断路器：
```
⇒  kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: httpbin
spec:
  host: httpbin
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 1
      http:
        http1MaxPendingRequests: 1
        maxRequestsPerConnection: 1
    outlierDetection:
      consecutiveErrors: 1
      interval: 1s
      baseEjectionTime: 3m
      maxEjectionPercent: 100
    tls:
      mode: ISTIO_MUTUAL
EOF

destinationrule.networking.istio.io/httpbin created
```
2. 检查我们的目标规则，确定已经正确建立：
```
kubectl get destinationrule httpbin -o yaml
⇒  kubectl get destinationrule httpbin -o yaml

apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
 ...
  generation: 1
  name: httpbin
  namespace: default
  resourceVersion: "163136"
  selfLink: /apis/networking.istio.io/v1alpha3/namespaces/default/destinationrules/httpbin
  uid: 00b60dd1-5133-11e9-8049-0800278050ec
spec:
  host: httpbin
  trafficPolicy:
    connectionPool:
      http:
        http1MaxPendingRequests: 1
        maxRequestsPerConnection: 1
      tcp:
        maxConnections: 1
    outlierDetection:
      baseEjectionTime: 3m
      consecutiveErrors: 1
      interval: 1s
      maxEjectionPercent: 100
```
#### 设置客户端
现在我们已经设置了调用 httpbin 服务的规则，接下来创建一个客户端，用来向后端服务发送请求，观察是否会触发熔断策略。这里要使用一个简单的负载测试客户端，名字叫 fortio。这个客户端可以控制连接数量、并发数以及发送 HTTP 请求的延迟。使用这一客户端，能够有效的触发前面在目标规则中设置的熔断策略。
1. 这里我们会把给客户端也进行 Sidecar 的注入，以此保证 Istio 对网络交互的控制
```
kubectl apply -f <(istioctl kube-inject -f samples/httpbin/sample-client/fortio-deploy.yaml)
```
2. 接下来就可以登入客户端 Pod 并使用 Fortio 工具来调用 httpbin。-curl 参数表明只调用一次：
```
⇒  FORTIO_POD=$(kubectl get pod | grep fortio | awk '{ print $1 }')
kubectl exec -it $FORTIO_POD  -c fortio /usr/bin/fortio -- load -curl  http://httpbin:8000/get

HTTP/1.1 200 OK
server: envoy
date: Fri, 29 Mar 2019 01:16:41 GMT
content-type: application/json
content-length: 814
access-control-allow-origin: *
access-control-allow-credentials: true
x-envoy-upstream-service-time: 4

{
  "args": {},
  "headers": {
    "Content-Length": "0",
    "Host": "httpbin:8000",
    "User-Agent": "fortio.org/fortio-1.3.1",
    "X-B3-Sampled": "1",
    "X-B3-Spanid": "5f828321fef8360e",
    "X-B3-Traceid": "6d63083e713db3d25f828321fef8360e",
    "X-Envoy-Decorator-Operation": "httpbin.default.svc.cluster.local:8000/*",
    "X-Istio-Attributes": "Cj0KF2Rlc3RpbmF0aW9uLnNlcnZpY2UudWlkEiISIGlzdGlvOi8vZGVmYXVsdC9zZXJ2aWNlcy9odHRwYmluCj8KGGRlc3RpbmF0aW9uLnNlcnZpY2UuaG9zdBIjEiFodHRwYmluLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwKJQoYZGVzdGluYXRpb24uc2VydmljZS5uYW1lEgkSB2h0dHBiaW4KKgodZGVzdGluYXRpb24uc2VydmljZS5uYW1lc3BhY2USCRIHZGVmYXVsdApDCgpzb3VyY2UudWlkEjUSM2t1YmVybmV0ZXM6Ly9mb3J0aW8tZGVwbG95LTc4Yzc4ZmY4NDctcWpjbmouZGVmYXVsdA=="
  },
  "origin": "172.17.0.27",
  "url": "http://httpbin:8000/get"
}
```
#### 触发熔断机制
在上面的熔断设置中指定了 maxConnections: 1 以及 http1MaxPendingRequests: 1。这意味着如果超过了一个连接同时发起请求，Istio 就会熔断，阻止后续的请求或连接。
1. 接下来尝试一下两个并发连接（-c 2），发送 20 请求（-n 20）：
```
⇒  kubectl exec -it $FORTIO_POD  -c fortio /usr/bin/fortio -- load -c 2 -qps 0 -n 20 -loglevel Warning http://httpbin:8000/get

01:16:54 I logger.go:97> Log level is now 3 Warning (was 2 Info)
Fortio 1.3.1 running at 0 queries per second, 4->4 procs, for 20 calls: http://httpbin:8000/get
Starting at max qps with 2 thread(s) [gomax 4] for exactly 20 calls (10 per thread + 0)
01:16:54 W http_client.go:679> Parsed non ok code 503 (HTTP/1.1 503)
01:16:54 W http_client.go:679> Parsed non ok code 503 (HTTP/1.1 503)
Ended after 48.551417ms : 20 calls. qps=411.93
Aggregated Function Time : count 20 avg 0.0047319828 +/- 0.001582 min 0.000856705 max 0.008711503 sum 0.094639655
# range, mid point, percentile, count
>= 0.000856705 <= 0.001 , 0.000928353 , 5.00, 1
> 0.001 <= 0.002 , 0.0015 , 10.00, 1
> 0.004 <= 0.005 , 0.0045 , 55.00, 9
> 0.005 <= 0.006 , 0.0055 , 95.00, 8
> 0.008 <= 0.0087115 , 0.00835575 , 100.00, 1
# target 50% 0.00488889
# target 75% 0.0055
# target 90% 0.005875
# target 99% 0.0085692
# target 99.9% 0.00869727
Sockets used: 4 (for perfect keepalive, would be 2)
Code 200 : 18 (90.0 %)
Code 503 : 2 (10.0 %)
Response Header Sizes : count 20 avg 207 +/- 69 min 0 max 230 sum 4140
Response Body/Total Sizes : count 20 avg 961.3 +/- 248.1 min 217 max 1044 sum 19226
All done 20 calls (plus 0 warmup) 4.732 ms avg, 411.9 qps
```
这里可以看到，几乎所有请求都通过了。Istio-proxy 允许存在一些误差。
```
Code 200 : 18 (90.0 %)
Code 503 : 2 (10.0 %)
```
2. 接下来把并发连接数量提高到 3：
```
⇒  kubectl exec -it $FORTIO_POD  -c fortio /usr/bin/fortio -- load -c 3 -qps 0 -n 30 -loglevel Warning http://httpbin:8000/get

01:57:32 I logger.go:97> Log level is now 3 Warning (was 2 Info)
Fortio 1.3.1 running at 0 queries per second, 4->4 procs, for 30 calls: http://httpbin:8000/get
Starting at max qps with 3 thread(s) [gomax 4] for exactly 30 calls (10 per thread + 0)
01:57:32 W http_client.go:679> Parsed non ok code 503 (HTTP/1.1 503)
01:57:32 W http_client.go:679> Parsed non ok code 503 (HTTP/1.1 503)
01:57:32 W http_client.go:679> Parsed non ok code 503 (HTTP/1.1 503)
01:57:32 W http_client.go:679> Parsed non ok code 503 (HTTP/1.1 503)
01:57:32 W http_client.go:679> Parsed non ok code 503 (HTTP/1.1 503)
01:57:32 W http_client.go:679> Parsed non ok code 503 (HTTP/1.1 503)
01:57:32 W http_client.go:679> Parsed non ok code 503 (HTTP/1.1 503)
01:57:32 W http_client.go:679> Parsed non ok code 503 (HTTP/1.1 503)
01:57:32 W http_client.go:679> Parsed non ok code 503 (HTTP/1.1 503)
01:57:32 W http_client.go:679> Parsed non ok code 503 (HTTP/1.1 503)
01:57:32 W http_client.go:679> Parsed non ok code 503 (HTTP/1.1 503)
01:57:32 W http_client.go:679> Parsed non ok code 503 (HTTP/1.1 503)
01:57:32 W http_client.go:679> Parsed non ok code 503 (HTTP/1.1 503)
01:57:32 W http_client.go:679> Parsed non ok code 503 (HTTP/1.1 503)
01:57:33 W http_client.go:679> Parsed non ok code 503 (HTTP/1.1 503)
Ended after 80.416474ms : 30 calls. qps=373.06
Aggregated Function Time : count 30 avg 0.006774503 +/- 0.005788 min 0.000560746 max 0.021858561 sum 0.203235091
# range, mid point, percentile, count
>= 0.000560746 <= 0.001 , 0.000780373 , 13.33, 4
> 0.001 <= 0.002 , 0.0015 , 23.33, 3
> 0.002 <= 0.003 , 0.0025 , 33.33, 3
> 0.003 <= 0.004 , 0.0035 , 40.00, 2
> 0.004 <= 0.005 , 0.0045 , 50.00, 3
> 0.005 <= 0.006 , 0.0055 , 60.00, 3
> 0.006 <= 0.007 , 0.0065 , 66.67, 2
> 0.008 <= 0.009 , 0.0085 , 73.33, 2
> 0.011 <= 0.012 , 0.0115 , 76.67, 1
> 0.012 <= 0.014 , 0.013 , 86.67, 3
> 0.014 <= 0.016 , 0.015 , 93.33, 2
> 0.02 <= 0.0218586 , 0.0209293 , 100.00, 2
# target 50% 0.005
# target 75% 0.0115
# target 90% 0.015
# target 99% 0.0215798
# target 99.9% 0.0218307
Sockets used: 18 (for perfect keepalive, would be 3)
Code 200 : 15 (50.0 %)
Code 503 : 15 (50.0 %)
Response Header Sizes : count 30 avg 115.13333 +/- 115.1 min 0 max 231 sum 3454
Response Body/Total Sizes : count 30 avg 630.63333 +/- 413.6 min 217 max 1045 sum 18919
All done 30 calls (plus 0 warmup) 6.775 ms avg, 373.1 qps
```
这时候会观察到，熔断行为按照之前的设计生效了，只有 50% 的请求获得通过，剩余请求被断路器拦截了：
```
Code 200 : 15 (50.0 %)
Code 503 : 15 (50.0 %)
```
3. 我们可以查询 istio-proxy 的状态，获取更多相关信息：
```
⇒  kubectl exec -it $FORTIO_POD  -c istio-proxy  -- sh -c 'curl localhost:15000/stats' | grep httpbin | grep pending

cluster.outbound|8000||httpbin.default.svc.cluster.local.circuit_breakers.default.rq_pending_open: 0
cluster.outbound|8000||httpbin.default.svc.cluster.local.circuit_breakers.high.rq_pending_open: 0
cluster.outbound|8000||httpbin.default.svc.cluster.local.upstream_rq_pending_active: 0
cluster.outbound|8000||httpbin.default.svc.cluster.local.upstream_rq_pending_failure_eject: 0
cluster.outbound|8000||httpbin.default.svc.cluster.local.upstream_rq_pending_overflow: 40
cluster.outbound|8000||httpbin.default.svc.cluster.local.upstream_rq_pending_total: 71
cluster.outbound|80||httpbin.org.circuit_breakers.default.rq_pending_open: 0
cluster.outbound|80||httpbin.org.circuit_breakers.high.rq_pending_open: 0
cluster.outbound|80||httpbin.org.upstream_rq_pending_active: 0
cluster.outbound|80||httpbin.org.upstream_rq_pending_failure_eject: 0
cluster.outbound|80||httpbin.org.upstream_rq_pending_overflow: 0
cluster.outbound|80||httpbin.org.upstream_rq_pending_total: 0
```
upstream_rq_pending_overflow 的值是 40，说明有 40 次调用被标志为熔断（因为有多次尝试）
#### 清理
1. 清理规则
```
⇒  kubectl delete destinationrule httpbin

destinationrule.networking.istio.io "httpbin" deleted
```
2. 关闭 httpbin 服务和客户端：
```
⇒  kubectl delete deploy httpbin fortio-deploy
kubectl delete svc httpbin

deployment.extensions "httpbin" deleted
deployment.extensions "fortio-deploy" deleted
service "httpbin" deleted
```