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
