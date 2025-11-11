容器化技术对于应用程序的开发部署的效率和稳定性至关重要，当容器的数量和架构的复杂度增加时，管理、维护容器就非常重要。kubernetes是一个容器编排引擎，帮助我们管理容器化的应用程序和服务。

# 1. 核心概念
## pod, service和Ingress
一个结点Node就是一个服务器（物理机）或虚拟机，在一个结点中可以运行一个或多个pod，pod是k8s的最小调度单元，一个pod就是一个或者多个应用容器的组合，pod创建了一个容器的运行环境，在环境中容器可以共享一些资源（如网络、存储以及运行时配置等）

一般情况下，建议一个pod中只运行一个容器，例如一个存在一个应用程序+一个数据库的系统，最好将应用程序和数据库分别放置在不同的pod中，除非多个容器高度耦合才可能需要一个pod运行多个容器（例如sidecar模式，一个容器+它的辅助容器，辅助容器负责日志收集、监控或配置管理等）

还是应用程序+数据库的例子，应用程序如果希望访问数据库，它需要知道数据库的IP地址，这个IP地址是在pod创建时自动分配的集群内部的IP地址，pod之间通过这个IP地址来通信。因此，**pod提供的这个IP地址只用于集群内部，无法在集群外部被访问**

另外，pod并不是一个稳定的实体，非常容易被创建或者销毁：例如发生故障时k8s会自动将pod销毁掉，然后重新创建一个新的相同的pod替代，但是此时**新创建的pod的内部IP地址会发生变动，其他pod不能再使用原来那个IP地址访问了**
为此，k8s提供了称为Service（svc）的资源对象，它将一组pod封装成为一个服务，服务通过一个统一的入口访问，即service的IP地址。svc的IP地址保持不变，哪怕其中的pod因为重建导致ip地址不稳定。

svc也分为内部服务和外部服务。
内部服务是我们不希望暴露给外部的服务，例如数据库、缓存、消息队列等等
有些服务则需要暴露给外部，例如后端接口或者前端界面，这些外部服务的类型包括：
- node:port：在节点node上开放一个端口，将其映射到外部服务svc的ip地址和端口上，这样就可以在外部直接通过节点的ip地址访问这个服务了，主要用于开发和测试阶段

然而，在实际生产使用时，通常使用域名来访问整个集群的服务，此时k8s提供了另一个资源对象Ingress：负责管理从集群外部访问集群内部服务的入口和方式，能够通过ingress配置不同的转发规则，通过不同规则来访问集群内部不同的service和其对应的后端pod，可以用于配置域名
Ingress还可以配置负载均衡、SSL证书等功能，后续会详细介绍


## configMap和Secret
应用程序和数据库之间的耦合问题也是需要注意的，例如应用程序需要访问数据库时，最常见的做法是把数据库的地址、端口等信息写到程序的配置文件或环境变量中，这就产生了耦合。如果这些信息可能存在变动，那么就得把集群停下来，重新编译整个程序，非常麻烦

configMap负责将配置信息进行封装，以将配置信息和应用程序的镜像内容分离开来，以保证容器化应用程序的可移植性：当配置信息出现变动，只需要重新加载配置信息，然后重启pod即可，不需要重新编译部署应用程序
注意，configMap的配置信息使用明文存储，不建议将敏感的密码等信息直接放置在其中，会存在安全问题

为了解决安全问题，k8s提供了名为secret的组件，它是专门用于存储敏感信息的，会对信息进行一层简单的Base64编码。但是这样并不能起到加密的作用，不能保证安全，需要配合k8s提供的其他安全组件使用。

## Volume组件持久化存储
容器被销毁或重启后，其中的数据也会消失，因此对于一些需要持久化存储数据的容器，k8s提供了volume组件：它将持久化存储的资源挂载至集群中的本地磁盘或集群外部的远程存储上，这样重启后就可以恢复这些数据而无需担心消失了

## Deployment组件保证无状态pot容错性
对于应用程序需要升级或发生故障导致重启，同时需要向应用程序访问服务时，此时需要通过克隆多个pod副本，在一个pod需要更新时将服务请求转发到其他pod实现
deployment就负责定义和管理pod的副本数量、更新策略等，具有副本控制、滚动更新、自动扩缩容等高级功能

对于应用程序来说，它的pod的多个副本不存在各自的状态，也就不需要考虑状态的问题，因此使用deployment；但是类似于数据库这样的，具有自己持久化存储的数据，多个副本如果不做规范可能会导致每个副本存在一份自己的数据（状态），因此需要确保多个副本的状态一致，需要同步或者使用共享存储

## StatefulSet组件保证有状态pot容错性
类似于Deployment也提供定义和管理副本数量、动态扩缩容等等功能，不同的是它保证了每个副本具有自己稳定的网络标识符和持久化存储，适合部分有状态的pod使用

值得注意的是，传统的数据库并不推荐使用容器部署和k8s管理，最好的方法还是将数据库部署在k8s集群以外；但是向量化数据库和小型系统的数据库不需要考虑这些，也都有k8s的支持

# 2. Kubernetes架构
k8s使用典型的Master-Worker架构，master负责管理整个集群，worker负责运行应用程序和服务
kubernetes官方给出的定义是k8s通过将容器放入在节点node上运行的pod中来执行用户的工作负载，这里的node就是k8s集群中真正完成实际工作的节点，也称为工作节点
为了提供对外服务，每个node上包含以下三个组件：
- kubelet ：管理维护每个节点上的pod，监控工作节点的运行情况并将信息汇报给apiServer；从apiServer中获取新的pod管理规范
- kube-proxy：负责为pod对象提供网络代理和负载均衡服务
- container-runtime：负责运行容器的软件集合，拉取镜像、创建容器、启动停止容器等

k8s集群中的多个节点通过Service节点进行通信，这需要一个负载均衡器接受不同的请求，然后将请求发送到不同的节点上，以完成负载均衡的工作。kube-proxy就负责这个功能，它在每个Node上启动一个网络代理，确保发往service的流量以高效的方式路由到正确的pod中，例如多个节点副本下，应用程序向数据库的服务请求会优先路由到与它在同一个节点的数据库的pod中

对于节点的管理、监控、pod调度、增删的工作，则由master节点完成，它具有以下组件：
- kube-apiServer：负责提供k8s集群的API接口服务，所有的组件都会通过这个接口来进行通信。例如用户需要在集群中部署一个新的应用时。用户与api server的交互方式通过客户端进行，例如kubectl命令行工具/Dashboard等图形化工具
	- api server是整个系统的入口，所有用户的请求都需要经过它，由它分发到不同的组件进行处理，包括增删/查询等等请求
	- api server 还负责对所有资源对象的增删改查等操作进行授权、认证、验证权限等，以保证系统的安全性
- scheduler调度器：负责监控集群中所有节点的资源使用情况，根据一些调度策略将pod调度到合适的节点上运行
- controllerManager：负责管理集群中各种资源对象的状态，监控它们的状态，根据状态进行对应的相应
- etcd：键值存储系统，存储集群中所有资源的状态信息，不包含pod中应用的数据
- cloud control manager：负责与云平台的API进行交互

# 3. 使用multipass+k3s本地搭建集群
创建一个虚拟机节点，会默认拉取ubuntu的官方镜像
```
multipass launch --name NODE_A --cpus 2 -memory 8G --disk 5G
```
如果拉取镜像的速度很慢而且哈希校验不通过，多半是因为国内ip不行，在clash中使用自己的梯子设置系统代理，然后在`~/.bashrc`中将代理设置为本机端口即可
```
# Proxy settings
export http_proxy="http://127.0.0.1:7890"
export https_proxy="http://127.0.0.1:7890"
export all_proxy="socks5://127.0.0.1:7890"
export no_proxy="localhost,127.0.0.1"

# 快捷命令
alias proxy-on='export http_proxy=http://127.0.0.1:7890 https_proxy=http://127.0.0.1:7890'
alias proxy-off='unset http_proxy https_proxy all_proxy'
```

如前所述，一个k8s集群中master节点负责管理集群，而worker节点负责执行实际的工作，在我们的虚拟机集群中，我们选择一个节点作为master节点，通过
```
multipass shell name
```
或者
```
ssh ubuntu@节点ip
```
（如果设置了ssh）
的方式进入该虚拟机，在它上面安装k3s
```
curl -sfL https://get.k3s.io | sh -
```
此时使用`sudo kubectl get nodes`就可以看到该节点被设置为了master节点：
```
ubuntu@testA:~$ sudo kubectl get nodes
NAME    STATUS   ROLES                  AGE   VERSION
testa   Ready    control-plane,master   42s   v1.33.5+k3s1
```

worker节点加入一个master节点管理的集群中时需要使用token，这个token被生成在master节点的`/var/lib/rancher/k3s/server/node-token`文件中：
```
ubuntu@testA:~$ sudo cat /var/lib/rancher/k3s/server/node-token
K102a449f5a6e3cd6ceef5b4abe34cb109087180e9e6de176405966972318f90a6a::server:7892131dabde81f1bdb828ba95c91a5a

```

在宿主机上使用：
```
TOKEN=$(multipass exec name sudo cat /var/lib/rancher/k3s/server/node-token)
```
就可以在外部保存这个密钥到`TOKEN`环境变量了，然后用这个密钥开始加入worker节点并搭建集群
使用
```
MASTER_IP=$(multipass info name | grep IPv4 | awk '{print $2}')
```
在宿主机上保存master节点的IP地址（将multipass info中IPv4开头的那一行，以空格为分界，打印出第二个字段，该位置就是节点的IP地址）

接着使用`multipass launch`创建worker虚拟机，然后在worker节点中安装k3s，但要注意下载命令和之前不同：
```
multipass exec name -- bash -c "curl -sfL https://get.k3s.io | K3S_URL=https://$MASTER_IP:6443 K3S_TOKEN=$TOKEN sh -"
```
这里`$MASER_IP`是宿主节点的IPv4地址，而`$TOKEN`则是之前所说的存储在`/var/lib/rancher/k3s/server/node-token`文件中的密钥，`name`是宿主机的名字

由于k3s默认使用countainerd作为容器运行时，其默认镜像在国外，如果在国内使用而不用代理，需要配置国内的镜像源，这里不多叙述

在这样配置完成之后，回到master虚拟机上使用`sudo kubectl get nodes`查看集群节点，就可以看到两个worker节点被加入集群了：
```
ubuntu@testA:~$ sudo kubectl get nodes
NAME      STATUS   ROLES                  AGE     VERSION
testa     Ready    control-plane,master   4h38m   v1.33.5+k3s1
worker1   Ready    <none>                 39m     v1.33.5+k3s1
worker2   Ready    <none>                 2m6s    v1.33.5+k3s1

```

# 4. kubectl常用命令
无论使用什么方式搭建集群，用户与k8s集群之间的交互都是通过kubectl完成的。在使用k3s搭建的多节点集群中，我们首先需要登陆到master节点
```
multipass shell testA
```
然后再执行与kubectl有关的命令，例如查看集群节点信息：
```
ubuntu@testA:~$ sudo kubectl get nodes
NAME      STATUS   ROLES                  AGE   VERSION
testa     Ready    control-plane,master   40h   v1.33.5+k3s1
worker1   Ready    <none>                 36h   v1.33.5+k3s1
worker2   Ready    <none>                 35h   v1.33.5+k3s1

```

而`sudo kubectl get svc`则会查看集群中所有的服务
```
ubuntu@testA:~$ sudo kubectl get svc
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.43.0.1    <none>        443/TCP   40h

```
这里因为我们的集群尚未部署服务，只有一个名为kubernetes的默认服务

`sudo kubectl get pod`则查看集群中所有pod的状态
```
ubuntu@testA:~$ sudo kubectl get pod
No resources found in default namespace.

```
这里我们尚未创建任何一个pod，因此我们可以通过使用`sudo kubectl run pod-name --image=image-name`的方式来创建一个使用`imageName`镜像的pod：
```
ubuntu@testA:~$ sudo kubectl run flink-pod --image=flink
pod/flink-pod created
ubuntu@testA:~$ sudo kubectl get pod
NAME        READY   STATUS         RESTARTS   AGE
flink-pod   0/1     ErrImagePull   0          10s

```
（注意k8s的命名规范是只能有小写字母、连字符`-`和点符号`.`的）
这里我们会发现这个pod的状态是`ErrImagePull`，代表镜像拉取失败，我们可以通过`sudo kubectl describe pod pod-name`来查看具体信息：
```
ubuntu@testA:~$ sudo kubectl describe pod flinkpod
Name:             flinkpod
Namespace:        default
Priority:         0
Service Account:  default
Node:             worker2/10.45.83.119
Start Time:       Thu, 30 Oct 2025 09:24:28 +0800
Labels:           run=flinkpod
Annotations:      <none>
Status:           Pending
IP:               10.42.3.3
IPs:
  IP:  10.42.3.3
Containers:
  flinkpod:
    Container ID:   
    Image:          flink:latest
    Image ID:       
    Port:           <none>
    Host Port:      <none>
    State:          Waiting
      Reason:       ImagePullBackOff
    Ready:          False
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-x5crq (ro)
Conditions:
  Type                        Status
  PodReadyToStartContainers   True 
  Initialized                 True 
  Ready                       False 
  ContainersReady             False 
  PodScheduled                True 
Volumes:
  kube-api-access-x5crq:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    Optional:                false
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type     Reason     Age                From               Message
  ----     ------     ----               ----               -------
  Normal   Scheduled  87s                default-scheduler  Successfully assigned default/flinkpod to worker2
  Normal   Pulling    49s (x3 over 87s)  kubelet            Pulling image "flink:latest"
  Warning  Failed     49s (x3 over 87s)  kubelet            Failed to pull image "flink:latest": failed to pull and unpack image "docker.io/library/flink:latest": failed to resolve reference "docker.io/library/flink:latest": failed to do request: Head "https://registry-1.docker.io/v2/library/flink/manifests/latest": EOF
  Warning  Failed     49s (x3 over 87s)  kubelet            Error: ErrImagePull
  Normal   BackOff    14s (x5 over 87s)  kubelet            Back-off pulling image "flink:latest"
  Warning  Failed     14s (x5 over 87s)  kubelet            Error: ImagePullBackOff

```