---
title: k8s部署--kubeadm
date: 2020-07-08 17:36:38
tags: 
  - 容器技术
categories: k8s
---

# 为什么需要kubeadm

## 如何容器化kubelet？
* 假设：将Kebernetes组件做成一个容器镜像，然后docker run启动。实际使用中会遇到一个很麻烦的问题，kebelet除了需要和容器运行大较大，在操作容器网络、管理数据卷时，都需要直接操作**宿主机**
  * 操作网络配置来说，docker容器可以通过不开启NetworkNamespace（即host network模式）的方式共享主机的网络栈
  * 操作宿主机的文件系统就有些困难了。如果想要使用NFS做容器的持久化数据卷，那么kebelet就需要在容器绑定前，在宿主机的指定目录上，先挂载NFS的远程目录。此时kubelet如果运行在容器中，"mount -F nfs"被隔离在一个单独的Mount Namespace中，不能传播到宿主机上
* 最终，kubeadm选择了妥协方案：直接在宿主机上运行kebelet，使用容器部署其他组件

## kebeadm init初始化
* 首先对机器进行检查，确保可以部署Kebernetes。包括，linux内核版本检测，Cgroup模块检测，机器hostname是否标准，kubeadm和kubelet版本是否匹配，端口是否占用，Docker是否安装等等
* 生成Kubernetes对外提供服务的各种证书和对应的目录
  * kubernetes对外提供服务，除非开启"不安全模式"，否则需要通过HTTPS才能访问kube-apiserver。因此需要证书文件
  * 证书放在Master节点的/etc/kubernetes/pki目录下。目录中主要证书文件是ca.crt和对应的私钥ca.key
  * 当用户使用kubectl获取容器日志等streaming操作之，需要通过kube-apiserver向kubelet发起请求，这个连接必须是安全的。kubeadm为这一步生成的apiserver-kubelet-client.crt，对应的死于是apiserver-kubelet-client.key
  * 此外还有一些其他的证书
  * 特别的，可以选择不让kubeadm生成这些证书，而是考培现有的证书到/etc/kubernetes/pki内
* 证书生成后，kubeadm会为其他组件生成访问kube-apiserver所需的配置文件，文件如下
```bash
ls /etc/kubernetes/ |grep conf
admin.conf controller-manager.conf kubelet.conf scheduler.conf
```
* kubeadm为Master组件（kube-apiserver，kube-controller-manager，kube-scheduler）生成Pod配置文件

## static pod
* 当Kubernetes集群上不存在,kubeadm不会直接执行docker run来启动这些容器。有一种特殊的容器启动方法，Static Pod，允许把要部署的Pod的Yaml文件放在指定的目录里。当机器上kubelet启动时，自动检查这个目录，加载所有的Pod Yaml文件，然后启动他们
* 可以看出kubelet在kubernetes中地位很高，设计上就是一个独立的组件
* kubeadm中，Master组件的YAML文件会被生成在/etc/kubernetes/manifests 路径下。比如kube-apiserver.yaml
```yaml
apiVersion: v1
kind: Pod
metadata:
annotations:
scheduler.alpha.kubernetes.io/critical-pod: ""
creationTimestamp: null
labels:
component: kube-apiserver
tier: control-plane
name: kube-apiserver
namespace: kube-system
spec:
containers:
# 启动命令 kube-apiserver是一个二进制文件，后面是配置
- command:
- kube-apiserver
- --authorization-mode=Node,RBAC
- --runtime-config=api/all=true
- --advertise-address=10.168.0.2
...
- --tls-cert-file=/etc/kubernetes/pki/apiserver.crt
- --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
#定义了一个容器，它使用的镜像是：k8s.gcr.io/kube-apiserver-amd64:v1.11.1
image: k8s.gcr.io/kube-apiserver-amd64:v1.11.1
imagePullPolicy: IfNotPresent
livenessProbe:
...
name: kube-apiserver
resources:
requests:
cpu: 250m
volumeMounts:
- mountPath: /usr/share/ca-certificates
name: usr-share-ca-certificates
readOnly: true
...
hostNetwork: true
priorityClassName: system-cluster-critical
volumes:
- hostPath:
path: /etc/ca-certificates
type: DirectoryOrCreate
name: etc-ca-certificates
```

* 如果要修改一个已有集群的kube-apiserver的配置，需要修改这个yaml文件
* 之后，kubeadm还会再生成一个Etcd的pod yaml文件，永阳的Static Pod方式启动Etcd。最后 Master 组件的 Pod YAML 文件如下所示
```
$ ls /etc/kubernetes/manifests/
etcd.yaml kube-apiserver.yaml kube-controller-manager.yaml kube-scheduler.yaml
```
* 一旦这些文件出现在被kubeadm监视的/etc/kubernetes/manifest内，kubelet就会自动穿件这些YAML文件中定义的Pod，即Master组件的容器
* Master容器启动后，kubeadm检查localhost:6443/healthz这个Master组件的健康检查URL，等待Master组件完全运行起来
* 然后，Kubeadm为集群生成一个**bootstrap token**，只要持有这个token任何一个安装了kubelet和kubadm的节点，都可以通过kubeadm join加入到这个集群当中。这个token会在kubeadm init结束后被打印出来
* token生成后，kubeadm会将ca.crt等Master节点重要信息，通过ConfigMap的方式保存在Etcd中，供后续部署Node节点使用。这个ConfigMap的名字是cluster-info

## 插件安装
* kube-proxy和DNS这两个插件是必须安装的。他们用来提供整个集群的服务发现和DNS功能。他们也只是两个容器镜像，所以kubeadm只要用kubernetes客户端创建两个POD就可以了


## kubeadm join
* 为什么需要bootstrap token？Kubeadm需要使用token至少发起一次“不安全模式”访问kube-apiserver，从而拿到保存在ConfigMap中的cluster-info(保存APIServer的授权信息)
* 拿到cluster-info里的kube-apiserver的地址、端口、证书。kubelet就可以以“安全模式”连接到apiserver上，这样一个新的节点就部署完成了


## 配置kubeadm的部署参数
* 推荐在使用kubeadm init部署Master节点时，使用如下指令
```bash
kubeadm init --config kubeadm.yaml
```
* 给kubeadm提供一个yaml文件，
```yaml
apiVersion: kubeadm.k8s.io/v1alpha2
kind: MasterConfiguration
kubernetesVersion: v1.11.0
api:
    advertiseAddress: 192.168.0.102
    bindPort: 6443
...
etcd:
    local:
    dataDir: /var/lib/etcd
    image: ""
imageRepository: k8s.gcr.io
kubeProxy:
    config:
    bindAddress: 0.0.0.0
...
kubeletConfiguration:
    baseConfig:
    address: 0.0.0.0
...
networking:
    dnsDomain: cluster.local
    podSubnet: ""
    serviceSubnet: 10.96.0.0/12
nodeRegistration:
    criSocket: /var/run/dockershim.sock
...
apiServerExtraArgs:
    advertise-address: 192.168.0.103
    anonymous-auth: false
    enable-admission-plugins: AlwaysPullImages,DefaultStorageClass
    audit-log-path: /home/johndoe/audit.log
```
* kubeadm会使用对应的信息替换/etc/kubernetes/manifests/*.yaml中的command指令的参数

# 总结
* kubeadm不太能用于生产环境。因为Etcd、Master等组件都应该是多节点集群，而不是上面使用的单点。这也正是kubeadm的发展方向
* 如果有部署规模化生产环境的需求，推荐使用kops或者 SaltStack 这样更复杂的部署工
具

# 最终需要的docker镜像总结
* kube-apiserver
* kube-controller-manager
* kube-scheduler
* kube-proxy
* DNS
