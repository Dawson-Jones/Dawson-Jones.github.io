---
title: "kubernetes 最佳实践"
date: 2023-08-01T00:48:09+08:00
draft: false
tags: ["tech", "clound native", "kubernetes"]
categories: ["Kubernetes"]
---

# 准备工作

- [Kubernetes安装](/posts/install_k8s_from_scratch)

- 安装 docker, 用来 make 和 push images
- docker login 登录



# Container

## Code

```go
package main

import (
	"io"
	"net/http"
)

func hello(w http.ResponseWriter, r *http.Request) {
	io.WriteString(w, "[v1] Hello, Kubernetes!")
}

func main() {
	http.HandleFunc("/", hello)
	http.ListenAndServe(":3000", nil)
}
```



## Dockerfile

```dockerfile
FROM golang:1.16-buster AS builder
RUN mkdir /src
ADD . /src
WORKDIR /src

RUN go env -w GO111MODULE=auto
RUN go build -o main .

FROM ubuntu:latest

WORKDIR /

COPY --from=builder /src/main /main
EXPOSE 3000
ENTRYPOINT ["/main"]
```



## docker build

`docker build -t 957360688/hellok8s:v1 . `

可以通过`docker image ls`查看刚才制作的镜像

## docker run

`docker run -p 3000:3000 --name hellok8s -d 957360688/hellok8s:v1`

此时通过 curl 命令确定容器已经运行

```bash
$ curl http://127.0.0.1:3000
[v1] Hello, Kubernetes!
```

## docker push

```bash
$ docker push 957360688/hellok8s:v1
The push refers to repository [docker.io/957360688/hellok8s]
743735510543: Pushed
59c56aee1fb4: Mounted from library/ubuntu
v1: digest: sha256:ee9c7cea8b31f6878456ds3bf45ef26bbe429edsgc4dbcb78d11bdgeb8660a47 size: 740
```



# Pod

Pod 是 Kubernetes 中可以创建管理的最小单元

Pod 和 Container 的关系可以参考 [Pod 和 Container 的关系](posts/relationship_between_pod_and_container)

## 创建一个简单的 Pod

```yaml
# nginx.yaml
apiVersion: v1
kind: Pod										# 创建的资源类型是 Pod
metadata:
  name: nginx-pod						# Pod 的名字
spec:
  #hostNetwork: true  			# 设置使用宿主机的网络
  containers:
  - name: nginx-container		# 容器名称
    image: nginx:1.14.2			# 镜像, 默认来源 DockerHub
    ports:
    - containerPort: 80
```



```bash
$ kubectl apply -f nginx.yaml
pod/nginx-pod created

$ kubectl get pods
NAME        READY   STATUS    RESTARTS   AGE
nginx-pod   1/1     Running   0          5m10s

$ kubectl exec nginx-pod -i -t -- bash		# 进入 pod 中配置
root@nginx-pod:/# echo "hello kubernetes by nginx!" > /usr/share/nginx/html/index.html
root@nginx-pod:/# exit
exit
$ kubectl port-forward nginx-pod 4000:80	# 开启端口转发, 将本机 4000 的流量转发到 Pod 的 80 端口
Forwarding from 127.0.0.1:4000 -> 80
Forwarding from [::1]:4000 -> 80
^Z
[1]+  Stopped                 kubectl port-forward nginx-pod 4000:80
$ bg
[1]+ kubectl port-forward nginx-pod 4000:80 &
$ curl http://127.0.0.1:4000
Handling connection for 4000							# 转发代理的输出
hello kubernetes by nginx!								# nginx 的 reply
```

## Pod 其他命令

```bash
$ kubectl logs --follow nginx-pod
127.0.0.1 - - [31/Jul/2023:18:29:04 +0000] "GET / HTTP/1.1" 200 27 "-" "curl/7.81.0" "-"

$ kubectl delete pod nginx-pod
pod "nginx-pod" deleted

kubectl delete -f nginx.yaml
pod "nginx-pod" deleted
```



## 启动自定义镜像

```yaml
# hellok8s.yaml

apiVersion: v1
kind: Pod
metadata:
  name: hellok8s
spec:
  containers:
    - name: hellok8s-container
      image: 957360688/hellok8s:v1
```

```bash
$ kubectl apply -f hellok8s.yaml
pod/hellok8s created

$ kubectl port-forward hellok8s 3000:3000
Forwarding from 127.0.0.1:3000 -> 3000
Forwarding from [::1]:3000 -> 3000
Handling connection for 3000								# curl 后出现

# -------另一个 shell
$ curl http://127.0.0.1:3000
[v1] Hello, Kubernetes!
```



# Deployment

在生产环境中, 一般情况会存在多个 Pod, 用来做容灾、备份, 这个时候自己去创建 Pod 会很麻烦. 此时需要 deployment 来管理 Pod, 比如**自动扩容, 自动升级**等

```yaml
# deployment.yaml

apiVersion: apps/v1
kind: Deployment            # 创建资源的类型是 deployment 类型
metadata:
  name: hellok8s-deployment # deployment 的名字
spec:
  strategy:
    rollingUpdate:
      maxSurge: 1           # 最大创建 4 个 pod (replicas + maxSurge)
      maxUnavailable: 1     # 最小 2 个 pod 存活 (replicas - maxUnavailable)
  replicas: 3               # pod 副本数量
  selector:                 # deployment 资源和 pod 资源的关联方式
    matchLabels:            # 管理所有 labels=hellok8s 的 pod
      app: hellok8s
  template:                 # 定义 pod 资源
    metadata:
      labels:               # 加上 labels 和 selector 的 matchLables 对应起来, 不需要加上 meta.name, deployment 会自动创建 pod 的 name
        app: hellok8s
    spec:
      containers:
        - name: hellok8s-container
          image: 957360688/hellok8s:v1
```

```bash
$ kubectl get pods -o wide -A
NAMESPACE     NAME                                   READY   STATUS    RESTARTS      AGE     IP             NODE    NOMINATED NODE   READINESS GATES
default       hellok8s-deployment-8668fb9d6b-56npv   1/1     Running   0             4m19s   10.0.0.185     node2   <none>           <none>
default       hellok8s-deployment-8668fb9d6b-7c46c   1/1     Running   0             4m19s   10.0.1.36      node3   <none>           <none>
default       hellok8s-deployment-8668fb9d6b-9m8s8   1/1     Running   0             4m19s   10.0.1.104     node3   <none>           <none>
kube-system   cilium-dsr52                           1/1     Running   1 (25h ago)   26h     172.17.8.101   node1   <none>           <none>
kube-system   cilium-lj888                           1/1     Running   1 (25h ago)   26h     172.17.8.103   node3   <none>           <none>
kube-system   cilium-operator-76c55fc6b6-ggtsw       1/1     Running   2 (11h ago)   26h     172.17.8.103   node3   <none>           <none>
kube-system   cilium-qqcvx                           1/1     Running   1 (25h ago)   26h     172.17.8.102   node2   <none>           <none>
kube-system   coredns-5d78c9869d-8kcxf               1/1     Running   1 (25h ago)   27h     10.0.0.131     node2   <none>           <none>
kube-system   coredns-5d78c9869d-wc7fm               1/1     Running   1 (25h ago)   27h     10.0.0.155     node2   <none>           <none>
kube-system   etcd-node1                             1/1     Running   1 (25h ago)   27h     172.17.8.101   node1   <none>           <none>
kube-system   kube-apiserver-node1                   1/1     Running   1 (25h ago)   27h     172.17.8.101   node1   <none>           <none>
kube-system   kube-controller-manager-node1          1/1     Running   1 (25h ago)   27h     172.17.8.101   node1   <none>           <none>
kube-system   kube-proxy-pv755                       1/1     Running   1 (25h ago)   27h     172.17.8.102   node2   <none>           <none>
kube-system   kube-proxy-wpk5w                       1/1     Running   1 (25h ago)   27h     172.17.8.103   node3   <none>           <none>
kube-system   kube-proxy-zwdqs                       1/1     Running   1 (25h ago)   27h     172.17.8.101   node1   <none>           <none>
kube-system   kube-scheduler-node1                   1/1     Running   1 (25h ago)   27h     172.17.8.101   node1   <none>           <none>
```

可以看到 pod 在 node2 上部署了 1 个, 在 node3 上部署了 2 个



> 以下内容设计到 namespace 设计, 与本文相关性不大

此时如果在 node2 和 node3 上分别操作会观察到以下现象

1. 每个 pod 中都包含一个 pause 的 container, 应该与容器网络的初始化有关. TODO: 待确认

```bash
# node3

$ sudo lsns -t net
        NS TYPE NPROCS   PID USER     NETNSID NSFS                                                COMMAND
4026531840 net     156     1 root  unassigned                                                     /sbin/init
4026532226 net       2 12090 65535          1 /run/netns/cni-bcc47fa8-7e5a-4b87-90f3-5178f1f6183c /pause
4026532296 net       1  3260 root           0                                                     cilium-health-resp
4026532358 net       2 12127 65535          2 /run/netns/cni-8c2a8fc6-95aa-46e6-9b1d-597997e29670 /pause

# 此时在 node3 上有两个 pause, 对应两个 pod
$ ps -ef | grep main
root       12167   12068  0 18:48 ?        00:00:00 /main
root       12200   12104  0 18:48 ?        00:00:00 /main
vagrant    12369    4833  0 19:02 pts/0    00:00:00 grep --color=auto main
$ sudo readlink /proc/12167/ns/net
net:[4026532226]
$ sudo readlink /proc/12200/ns/net
net:[4026532358]
```
```bash
# node2

$ sudo lsns -t net
        NS TYPE NPROCS   PID USER     NETNSID NSFS                                                COMMAND
4026531840 net     155     1 root  unassigned                                                     /sbin/init
4026532220 net       2  3338 65535          1 /run/netns/cni-5013ee55-281a-dd96-8b27-b6d5bcfdb395 /pause
4026532281 net       2  3331 65535          2 /run/netns/cni-1f5d8bbb-739c-b257-51bc-dd59e2a933e9 /pause
4026532349 net       1  3236 root           0                                                     cilium-health-resp
4026532430 net       2 10884 65535          3 /run/netns/cni-04d339aa-57a1-2adc-5820-908dfc3a9c20 /pause

# 此时的 node2 上有三个 pod, 但只有一个对应自己创建的 pod, 还有两个对应 coredns
$ ps -ef | grep main
root       10918   10861  0 18:49 ?        00:00:00 /main
vagrant    11107    4325  0 19:05 pts/0    00:00:00 grep --color=auto main
$ sudo readlink /proc/10918/ns/net
net:[4026532430]
$ ps -ef | grep core
root        3390    3294  0 09:58 ?        00:01:20 /coredns -conf /etc/coredns/Corefile
root        3398    3281  0 09:58 ?        00:01:20 /coredns -conf /etc/coredns/Corefile
vagrant    11120    4325  0 19:06 pts/0    00:00:00 grep --color=auto core
$ sudo readlink /proc/3390/ns/net
net:[4026532281]
$ sudo readlink /proc/3398/ns/net
net:[4026532220]
```

2. Pod 间网络是相通的, 这是由于 cni 导致的, 在我的环境下是 cilium_vxlan 打通了容器网络

```bash
# node3

# 进入 pod 所在的 net namespace
$ sudo nsenter -n -t 12167 bash
root@node3:/home/vagrant# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
20: eth0@if21: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 42:99:b3:9e:f6:32 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.0.1.36/32 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::4099:b3ff:fe9e:f632/64 scope link
       valid_lft forever preferred_lft forever
root@node3:/home/vagrant# ping 10.0.0.185
PING 10.0.0.185 (10.0.0.185) 56(84) bytes of data.
64 bytes from 10.0.0.185: icmp_seq=1 ttl=63 time=2.13 ms
64 bytes from 10.0.0.185: icmp_seq=2 ttl=63 time=0.690 ms
```

```bash
# node2

# 进入 pod 所在的 net namespace
$ sudo nsenter -n -t 10918
root@node2:/home/vagrant# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
16: eth0@if17: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 22:aa:bc:e9:99:5a brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.0.0.185/32 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::20aa:bcff:fee9:995a/64 scope link
       valid_lft forever preferred_lft forever
root@node2:/home/vagrant# tcpdump -i eth0
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
19:13:02.197513 IP 10.0.1.36 > 10.0.0.185: ICMP echo request, id 44304, seq 1, length 64
19:13:02.197550 IP 10.0.0.185 > 10.0.1.36: ICMP echo reply, id 44304, seq 1, length 64
```



## 管理 Pod

当 pod 被删除时, 会自动拉起新的 pod

```bash
$ kubectl get pods -o wide
NAME                                   READY   STATUS    RESTARTS   AGE    IP           NODE    NOMINATED NODE   READINESS GATES
hellok8s-deployment-8668fb9d6b-56npv   1/1     Running   0          9m7s   10.0.0.185   node2   <none>           <none>
hellok8s-deployment-8668fb9d6b-7c46c   1/1     Running   0          9m7s   10.0.1.36    node3   <none>           <none>
hellok8s-deployment-8668fb9d6b-9m8s8   1/1     Running   0          9m7s   10.0.1.104   node3   <none>           <none>
$ kubectl delete pod hellok8s-deployment-8668fb9d6b-7c46c
pod "hellok8s-deployment-8668fb9d6b-7c46c" deleted
$ kubectl get pods -o wide
NAME                                   READY   STATUS    RESTARTS   AGE   IP           NODE    NOMINATED NODE   READINESS GATES
hellok8s-deployment-8668fb9d6b-56npv   1/1     Running   0          27m   10.0.0.185   node2   <none>           <none>
hellok8s-deployment-8668fb9d6b-9m8s8   1/1     Running   0          27m   10.0.1.104   node3   <none>           <none>
hellok8s-deployment-8668fb9d6b-brcs5   1/1     Running   0          65s   10.0.0.121   node2   <none>           <none>
```

## 自动扩容

改变 replicas 值, 将会自动扩容

注意: 虽然重新 apply 了 yaml, 但 pod 只是在原有的 3 个的基础上重新添加了一个

```bash
$ kubectl apply -f deployment.yaml
deployment.apps/hellok8s-deployment configured

$ kubectl get pods -o wide
NAME                                   READY   STATUS    RESTARTS   AGE     IP           NODE    NOMINATED NODE   READINESS GATES
hellok8s-deployment-8668fb9d6b-56npv   1/1     Running   0          29m     10.0.0.185   node2   <none>           <none>
hellok8s-deployment-8668fb9d6b-9m8s8   1/1     Running   0          29m     10.0.1.104   node3   <none>           <none>
hellok8s-deployment-8668fb9d6b-brcs5   1/1     Running   0          3m20s   10.0.0.121   node2   <none>           <none>
hellok8s-deployment-8668fb9d6b-ddk7t   1/1     Running   0          5s      10.0.1.45    node3   <none>           <none>
```

## 升级版本

```go
package main

import (
	"io"
	"net/http"
)

func hello(w http.ResponseWriter, r *http.Request) {
	io.WriteString(w, "[v2] Hello, Kubernetes!\n")
}

func main() {
	http.HandleFunc("/", hello)
	http.ListenAndServe(":3000", nil)
}
```

将 v2 推送到 docker hub

```bash
docker build . -t 957360688/hellok8s:v2
docker push 957360688/hellok8s:v2
```

将 `deployment.yaml` 修改为 v2 版本

```yaml
    spec:
      containers:
        - name: hellok8s-container
          image: 957360688/hellok8s:v2
```



```bash
$ kubectl apply -f deployment.yaml
deployment.apps/hellok8s-deployment configured

$ kubectl get pods --watch
NAME                                   READY   STATUS    RESTARTS   AGE
hellok8s-deployment-76db5f8f49-2txjp   1/1     Running   0          2m26s
hellok8s-deployment-76db5f8f49-bv7r9   1/1     Running   0          2m25s
hellok8s-deployment-76db5f8f49-nf4p6   1/1     Running   0          2m24s
hellok8s-deployment-76db5f8f49-ngrg2   1/1     Running   0          2m26s

hellok8s-deployment-8668fb9d6b-gllmt   0/1     Pending   0          0s
hellok8s-deployment-8668fb9d6b-gllmt   0/1     Pending   0          0s
hellok8s-deployment-76db5f8f49-nf4p6   1/1     Terminating   0          2m31s
hellok8s-deployment-8668fb9d6b-gllmt   0/1     ContainerCreating   0          0s
hellok8s-deployment-8668fb9d6b-9nc7q   0/1     Pending             0          0s

$ kubectl describe pod hellok8s-deployment-8668fb9d6b-vlmrm
Containers:
  hellok8s-container:
    Container ID:   containerd://01c75227330b7e79550c0d5552441a19027b537f7625869fcd453748eb4f294c
    Image:          957360688/hellok8s:v2   
```

## 回滚版本

```bash
$ kubectl rollout undo deployment hellok8s-deployment
deployment.apps/hellok8s-deployment rolled back

$ kubectl get pods
NAME                                   READY   STATUS    RESTARTS   AGE
hellok8s-deployment-76db5f8f49-fx4bd   1/1     Running   0          17s
hellok8s-deployment-76db5f8f49-pnm5m   1/1     Running   0          17s
hellok8s-deployment-76db5f8f49-xnb5b   1/1     Running   0          15s
hellok8s-deployment-76db5f8f49-z8mjv   1/1     Running   0          15s
$ kubectl describe pod hellok8s-deployment-76db5f8f49-fx4bd
Containers:
  hellok8s-container:
    Container ID:   containerd://0ba2eaae0d0f38ce5f65db5e259442ded2eed900b2da43dfce757103c2cf6f9d
    Image:          957360688/hellok8s:v1
```

查看历史版本

`kubectl rollout history deployment hellok8s-deployment`

回滚到指定版本

`kubectl rollout undo deployment/hellok8s-deployment --to-revision=2`



## 存活探针

存活探针来确定什么时候重启容器

在生产环境中, 因为某些未知 bug 导致死锁或者线程耗尽, 如果没有手段来监控和处理这个问题, 可能会导致很长一段时间没有人发现该问题

下面的代码模拟了出现问题的情况, `/healthz` 会在 15 秒后返回 500



- 代码

```go
package main

import (
	"fmt"
	"io"
	"net/http"
	"time"
)

func hello(w http.ResponseWriter, r *http.Request) {
	io.WriteString(w, "[v2] Hello, Kubernetes!\n")
}

func main() {
	started := time.Now()
	http.HandleFunc("/healthz", func(writer http.ResponseWriter, request *http.Request) {
		duration := time.Since(started)
		if duration.Seconds() > 15 {
			writer.WriteHeader(500)
			writer.Write([]byte(fmt.Sprintf("error: %v", duration.Seconds())))
		} else {
			writer.WriteHeader(200)
			writer.Write([]byte("ok"))
		}
	})
	http.HandleFunc("/", hello)
	http.ListenAndServe(":3000", nil)
}
```

- 构建镜像

```bash
docker build -t 957360688/hellok8s:liveness .
docker push 957360688/hellok8s:liveness 
```

- 编写 `deployment.yaml`

```yaml
# deployment.yaml

apiVersion: apps/v1
kind: Deployment            # 创建资源的类型是 deployment 类型
metadata:
  name: hellok8s-deployment # deployment 的名字
spec:
  strategy:
    rollingUpdate:
      maxSurge: 1           # 最大创建 4 个 pod (replicas + maxSurge)
      maxUnavailable: 1     # 最小 2 个 pod 存活 (replicas - maxUnavailable)
  replicas: 3               # pod 副本数量
  selector:                 # deployment 资源和 pod 资源的关联方式
    matchLabels:            # 管理所有 labels=hellok8s 的 pod
      app: hellok8s
  template:                 # 定义 pod 资源
    metadata:
      labels:               #|\ 加上 labels 和 selector 的 matchLables 对应起来
        app: hellok8s       #|/ 不需要加上 meta.name, deployment 会自动创建 pod 的 name
    spec:
      containers:
        - name: hellok8s-container
          image: 957360688/hellok8s:liveness
          livenessProbe:    # 存活探针用来确定什么时候需要重启容器
            httpGet:
              path: /healthz
              port: 3000
            initialDelaySeconds: 3
            periodSeconds: 3

```

- 运行

```bash
$ kubectl apply -f deployment.yaml
deployment.apps/hellok8s-deployment configured

$ kubectl get pods
NAME                                   READY   STATUS    RESTARTS   AGE
hellok8s-deployment-5fd9dc769c-7ncst   1/1     Running   0          33s
hellok8s-deployment-5fd9dc769c-ppgk2   1/1     Running   0          34s
hellok8s-deployment-5fd9dc769c-qhf5z   1/1     Running   0          20s
# 每过一段时间, pod 就会重启
# 注意: pod 重启, pod 的名字不会改变
$ kubectl get pods
NAME                                   READY   STATUS    RESTARTS      AGE
hellok8s-deployment-5fd9dc769c-7ncst   1/1     Running   1 (38s ago)   41s
hellok8s-deployment-5fd9dc769c-ppgk2   1/1     Running   1 (57s ago)   42s
hellok8s-deployment-5fd9dc769c-qhf5z   1/1     Running   1 (56s ago)   28s

# describe 也可以看到相应事件
$ kubectl describe pod hellok8s-deployment-5fd9dc769c-7ncst
Events:
  Type     Reason     Age                    From               Message
  ----     ------     ----                   ----               -------
  Normal   Scheduled  3m39s                  default-scheduler  Successfully assigned default/hellok8s-deployment-5fd9dc769c-7ncst to node2
  Normal   Pulling    4m12s                  kubelet            Pulling image "957360688/hellok8s:liveness"
  Normal   Pulled     4m                     kubelet            Successfully pulled image "957360688/hellok8s:liveness" in 11.316962482s (11.317024706s including waiting)
  Normal   Created    2m48s (x4 over 4m)     kubelet            Created container hellok8s-container
  Normal   Started    2m48s (x4 over 4m)     kubelet            Started container hellok8s-container
  Warning  Unhealthy  2m48s (x9 over 3m42s)  kubelet            Liveness probe failed: HTTP probe failed with statuscode: 500
  Normal   Killing    2m48s (x3 over 3m36s)  kubelet            Container hellok8s-container failed liveness probe, will be restarted
  Normal   Pulled     2m48s (x3 over 3m36s)  kubelet            Container image "957360688/hellok8s:liveness" already present on machine
```



## 就绪探针

用来判断 pod 是否就绪, 如果不就绪则不会被 Service 负责均衡

在 [存活探针](#存活探针) 的基础上添加以下代码, 用来表示这是一个有问题的版本



- 代码

```go
http.HandleFunc("/ready", func(writer http.ResponseWriter, request *http.Request) {
	writer.WriteHeader(500) // 有问题的版本
})
```

- 构建镜像

```bash
docker build -t 957360688/hellok8s:bad .
docker push 957360688/hellok8s:bad
```



- 编写 `deployment.yaml`

```yaml
# deployment.yaml

apiVersion: apps/v1
kind: Deployment            # 创建资源的类型是 deployment 类型
metadata:
  name: hellok8s-deployment # deployment 的名字
spec:
  strategy:
    rollingUpdate:
      maxSurge: 1           # 最大创建 4 个 pod (replicas + maxSurge)
      maxUnavailable: 1     # 最小 2 个 pod 存活 (replicas - maxUnavailable)
  replicas: 3               # pod 副本数量
  selector:                 # deployment 资源和 pod 资源的关联方式
    matchLabels:            # 管理所有 labels=hellok8s 的 pod
      app: hellok8s
  template:                 # 定义 pod 资源
    metadata:
      labels:               #|\ 加上 labels 和 selector 的 matchLables 对应起来
        app: hellok8s       #|/ 不需要加上 meta.name, deployment 会自动创建 pod 的 name
    spec:
      containers:
        - name: hellok8s-container
          image: 957360688/hellok8s:bad
          readinessProbe:   # 就绪指针, 用来获知容器合适准备好接受请求流量
            httpGet:
              path: /ready
              port: 3000
            initialDelaySeconds: 1  # 容器启动后，Kubernetes 等待多少秒开始进行就绪检查
            successThreshold: 5     # 就绪检查成功所需的连续成功次数
            failureThreshold: 3     # 失败次数到达阈值则重启, 默认为 3
          livenessProbe:    # 存活探针用来确定什么时候需要重启容器
            httpGet:
              path: /healthz
              port: 3000
            initialDelaySeconds: 3
            periodSeconds: 3
```



- 运行

```bash
# 先回退到 v2
$ kubectl rollout undo deployment/hellok8s-deployment --to-revision=2
deployment.apps/hellok8s-deployment rolled back
$ kubectl get pods
NAME                                   READY   STATUS    RESTARTS   AGE
hellok8s-deployment-8668fb9d6b-5g86c   1/1     Running   0          12s
hellok8s-deployment-8668fb9d6b-6z9h8   1/1     Running   0          9s
hellok8s-deployment-8668fb9d6b-lbpnl   1/1     Running   0          12s
hellok8s-deployment-8668fb9d6b-zwljj   1/1     Running   0          10s

# 使用新的 deployment
$ kubectl apply -f deployment.yaml
deployment.apps/hellok8s-deployment configured
$ kubectl get pods
NAME                                   READY   STATUS             RESTARTS      AGE
hellok8s-deployment-6db5d84444-6hv6q   1/1     Running            1 (99s ago)   24s
hellok8s-deployment-6db5d84444-8qcch   1/1     Running            2 (65s ago)   46s
hellok8s-deployment-6db5d84444-zs654   0/1     CrashLoopBackOff   2 (93s ago)   46s
```

可以发现有 pod 始终没有 ready, 但是有因为设置了 `maxUnavailable` 最大不可用, 所有始终会有 2 个 pod 是 ready 的, 能够继续提供服务

 

# Service

Service 位于 Pod 的前面, 为 Pod 提供了一个稳定的 Endpoint, 负责接收请求并传递给后面的所有的 Pod

PS: 有点类似于 LVS or Nginx 这种反向代理工具

一旦 Pod 中的集合发生了更改, Endpoints 会被更新, 请求也会重新导向到最新的 Pod



service 类型

- ClusterIP: 通过集群内部 IP 暴露服务, 只能在集群内部访问

- NodePort: 通过每个节点上的 IP 和静态端口(NodePort) 暴露服务, 可以从集群外部访问

- LoadBalancer: 使用负载均衡器向外暴露服务, 负载均衡器将流量路由到 NodePort 或者 ClusterIP 上

- ExternalName: 通过返回 CNAME 值, 将服务映射到 externalName 字段的内容, 无需创建任何代理

  

## ClusterIP

- 代码

```go
package main

import (
	"fmt"
	"io"
	"net/http"
	"os"
)

func hello(w http.ResponseWriter, r *http.Request) {
	host, _ := os.Hostname()
	io.WriteString(w, fmt.Sprintf("[v3] Hello, Kubernetes! From host: %s", host))
}

func main() {
	http.HandleFunc("/healthz", func(writer http.ResponseWriter, request *http.Request) {
		writer.WriteHeader(200)
		writer.Write([]byte("ok"))
	})
	http.HandleFunc("/ready", func(writer http.ResponseWriter, request *http.Request) {
		writer.WriteHeader(200)
	})
	http.HandleFunc("/", hello)
	http.ListenAndServe(":3000", nil)
}
```

- 构建镜像

```bash
docker build -t 957360688/hellok8s:v3 .
docker push 957360688/hellok8s:v3
```

- 编写 deployment 和 service

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment                # 创建资源的类型是 deployment 类型
metadata:
  name: hellok8s-deployment     # deployment 的名字
spec:
  strategy:
    rollingUpdate:
      maxSurge: 1               # 最大创建 4 个 pod (replicas + maxSurge)
      maxUnavailable: 1         # 最小 2 个 pod 存活 (replicas - maxUnavailable)
  replicas: 3                   # pod 副本数量
  selector:                     # deployment 资源和 pod 资源的关联方式
    matchLabels:                # 管理所有 labels=hellok8s 的 pod
      app: hellok8s
  template:                     # 定义 pod 资源
    metadata:
      labels:                   # 加上 labels 和 selector 的 matchLables 对应起来
        app: hellok8s
    spec:
      containers:
        - name: hellok8s-container
          image: 957360688/hellok8s:v3
          readinessProbe:       # 就绪指针, 用来获知容器合适准备好接受请求流量
            httpGet:
              path: /ready
              port: 3000
            initialDelaySeconds: 1
            successThreshold: 5
          livenessProbe:        # 存活探针用来确定什么时候需要重启容器
            httpGet:
              path: /healthz
              port: 3000
            initialDelaySeconds: 3
            periodSeconds: 3
```

```yaml
# service-hellok8s-clusterip.yaml
apiVersion: v1
kind: Service
metadata:
  name: serivice-hellok8s-clusterip
spec:
  type: ClusterIP
  selector:
    app: hellok8s
  ports:
    - port: 3000
      targetPort: 3000
```

- 运行

```bash
$ kubectl apply -f deployment.yaml
deployment.apps/hellok8s-deployment configured
$ kubectl get pods -o wide
NAME                                   READY   STATUS    RESTARTS   AGE     IP          NODE    NOMINATED NODE   READINESS GATES
hellok8s-deployment-76759bb898-2z5ns   1/1     Running   0          2m44s   10.0.0.55   node2   <none>           <none>
hellok8s-deployment-76759bb898-cmgsf   1/1     Running   0          2m43s   10.0.1.44   node3   <none>           <none>
hellok8s-deployment-76759bb898-cqvjz   1/1     Running   0          2m13s   10.0.1.37   node3   <none>           <none>

$ kubectl apply -f service-hellok8s-clusterip.yaml
service/serivice-hellok8s-clusterip created

$ kubectl get endpoints
NAME                          ENDPOINTS                                      AGE
kubernetes                    172.17.8.101:6443                              2d17h
serivice-hellok8s-clusterip   10.0.0.55:3000,10.0.1.37:3000,10.0.1.44:3000   24s
$ kubectl get service
NAME                          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
kubernetes                    ClusterIP   10.96.0.1       <none>        443/TCP    2d17h
serivice-hellok8s-clusterip   ClusterIP   10.100.77.149   <none>        3000/TCP   76s

# 访问这个 service 的 IP
$ curl http://10.100.77.149:3000
[v3] Hello, Kubernetes! From host: hellok8s-deployment-76759bb898-cqvjz
$ curl http://10.100.77.149:3000
[v3] Hello, Kubernetes! From host: hellok8s-deployment-76759bb898-2z5ns
$ curl http://10.100.77.149:3000
[v3] Hello, Kubernetes! From host: hellok8s-deployment-76759bb898-cmgsf

# -------------------- 在另一台 worker node 尝试
# 1. 在 node2 的 host 环境中
vagrant@node2:~$ curl http://10.100.77.149:3000
[v3] Hello, Kubernetes! From host: hellok8s-deployment-76759bb898-cqvjz
vagrant@node2:~$ curl http://10.100.77.149:3000
[v3] Hello, Kubernetes! From host: hellok8s-deployment-76759bb898-cmgsf
# 2. 在 node2 的 container 中
vagrant@node2:~$ sudo lsns -t net
        NS TYPE NPROCS   PID USER     NETNSID NSFS                                                COMMAND
4026531840 net     158     1 root  unassigned                                                     /sbin/init
4026532220 net       2  3338 65535          1 /run/netns/cni-5013ee55-281a-dd96-8b27-b6d5bcfdb395 /pause
4026532281 net       2  3331 65535          2 /run/netns/cni-1f5d8bbb-739c-b257-51bc-dd59e2a933e9 /pause
4026532349 net       1  3236 root           0                                                     cilium-health-re
4026532436 net       2 22875 65535          4 /run/netns/cni-d3e64301-9780-0484-f6ed-de060935f144 /pause

vagrant@node2:~$ sudo nsenter -u -n -t 22875
root@hellok8s-deployment-76759bb898-2z5ns:/home/vagrant# curl http://10.100.77.149:3000
[v3] Hello, Kubernetes! From host: hellok8s-deployment-76759bb898-cmgsf
root@hellok8s-deployment-76759bb898-2z5ns:/home/vagrant# curl http://10.100.77.149:3000
[v3] Hello, Kubernetes! From host: hellok8s-deployment-76759bb898-2z5ns
root@hellok8s-deployment-76759bb898-2z5ns:/home/vagrant# curl http://10.100.77.149:3000
[v3] Hello, Kubernetes! From host: hellok8s-deployment-76759bb898-cqvjz
# --------------------
```

```
                        | ----------------- |
                        | service(hellok8s) |
                        | ----------------- |
                          /       |        \
                         /        |         \
                 | ------ |   | ------ |   | ------ |
                 |  Pod1  |   |  Pod2  |   |  Pod3  |
                 | ------ |   | ------ |   | ------ |
```



## NodePort

端口段只能在 30000-32767, 不能使用 80 or 443 等

```yaml
# service-hellok8s-nodeport.yaml
apiVersion: v1
kind: Service
metadata:
  name: serivice-hellok8s-nodeport
spec:
  type: NodePort
  selector:
    app: hellok8s
  ports:
    - port: 3000
      nodePort: 30000
```



```bash
$ kubectl apply -f kubectl apply -f service-hellok8s-nodeport.yaml
service/serivice-hellok8s-clusterip configured
$ kubectl get endpoints
NAME                          ENDPOINTS                                      AGE
kubernetes                    172.17.8.101:6443                              2d17h
serivice-hellok8s-clusterip   10.0.0.55:3000,10.0.1.37:3000,10.0.1.44:3000   41m
$ kubectl get service
NAME                         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
kubernetes                   ClusterIP   10.96.0.1        <none>        443/TCP          2d18h
serivice-hellok8s-nodeport   NodePort    10.101.113.176   <none>        3000:30000/TCP   2m
```

用另一台不在该 kubernetes 的机器访问

```bash
# 1. 要使用映射的端口进行访问
# 2. 访问 kubernetes 集群中的任意一个 node, 都可以访问成功
# 3. 发送到哪个 pod 是随机的(根据 kubernetes 的策略)
$ curl http://172.17.8.101:30000
[v3] Hello, Kubernetes! From host: hellok8s-deployment-76759bb898-cmgsf
$ curl http://172.17.8.102:30000
[v3] Hello, Kubernetes! From host: hellok8s-deployment-76759bb898-cqvjz
```



## LoadBalancer

是一种服务, 通常有云提供商作为外部服务实现, 也可以通过 meralLB 在 Kubernetes 集群内部安装, 负载均衡器提供一个单一的 IP 地址来访问你的服务, 这些服务可以在多个 Node 上运行



# Ingress

整合多个应用程序的路由规则到一个实体中

公开从集群外部到集群内[服务](https://kubernetes.io/zh-cn/docs/concepts/services-networking/service/)的 HTTP 和 HTTPS 路由。 流量路由由 Ingress 资源上定义的规则控制 通常需要一个 ingress 控制器

## [安装 Ingress-Nginx 控制器](https://kubernetes.github.io/ingress-nginx/deploy/)

可以使用 helm 安装或使用 yaml 清单安装

- helm 安装

[安装 helm](https://helm.sh/zh/docs/intro/install/)

`sudo snap install helm --classic`

[安装 ingress-nginx](https://kubernetes.github.io/ingress-nginx/deploy/#quick-start)

```bash
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace
```

- 使用 yaml manifest

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.1/deploy/static/provider/cloud/deploy.yaml
```

### 检查 ingress controller

现在 `EXTERNAL-IP` 是 <pending> 状态, 这是因为 Kubernetes 无法配置 load balancer, 因为现在还没有 load balancer

```bash
# 现在 ingress-nginx-controller 运行在 node2 上
$ kubectl get pods -o wide -n ingress-nginx
NAME                                        READY   STATUS    RESTARTS   AGE   IP           NODE    NOMINATED NODE   READINESS GATES
ingress-nginx-controller-5fcb5746fc-bpskp   1/1     Running   0          14h   10.0.0.144   node2   <none>           <none>

# 外部 30721 会 nat 10.100.237.56:80 端口, 使用 sudo iptables -t nat -vnL 查看
$ kubectl get services -n ingress-nginx
NAME                                 TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.100.237.56   <pending>     80:30721/TCP,443:31176/TCP   12h
ingress-nginx-controller-admission   ClusterIP      10.99.222.0     <none>        443/TCP                      12h
```



从 v1.0.0 开始, ingress-nginx controller, 需要一个 ingressclass 对象, 默认安装中, 已经创建了一个名为 nginx 的 ingressclass 对象

```bash
$ kubectl -n ingress-nginx get ingressclasses
NAME    CONTROLLER             PARAMETERS   AGE
nginx   k8s.io/ingress-nginx   <none>       13h
```

如果没有的话, 可以添加一个

`kubectl -n ingress-nginx annotate ingressclasses nginx ingressclass.kubernetes.io/is-default-class="true"`



## 使用 ingress

该项目使用了两个不同的 deployment, 同时使用 ingress 来进行路由分发

- /hello, 会发送到 hellok8s V3
- 其他的则会发送到 nginx

## 验证

文件在: https://raw.githubusercontent.com/Dawson-Jones/k8s_tutorial/main/VI_ingress/two_services_with_ingress.yaml

```bash
# two_services_with_ingress.yaml
$ kubectl apply -f two_services_with_ingress.yaml
deployment.apps/hellok8s-deployment created
deployment.apps/nginx-deployment created
service/service-hellok8s-clusterip created
service/service-nginx-clusterip created
ingress.networking.k8s.io/hello-ingress created
```



在一台外部机器访问, 这里我使用了宿主机

可得到如下结论

1. 访问任意 node 的 ingress-nginx-controller service, 都可以流量最终访问到 pod 中
2. 流量通过路径被区分了

```bash
# 端口是 ingress-nginx-controller service 暴露的端口
$ curl http://172.17.8.101:30721/hello                         *[master]
[v3] Hello, Kubernetes! From host: hellok8s-deployment-6b9f96b687-pv8x7
$ curl http://172.17.8.102:30721/hello                   *[master]
[v3] Hello, Kubernetes! From host: hellok8s-deployment-6b9f96b687-7k5t7
$ curl http://172.17.8.101:30721                               *[master]
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

