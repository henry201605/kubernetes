### 1 StatefulSet

`官网`：<https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/>

```
StatefulSet is the workload API object used to manage stateful applications.
Manages the deployment and scaling of a set of Pods, and provides guarantees about the ordering and uniqueness of these Pods.
```

- Stable, unique network identifiers.

- Stable, persistent storage.

- Ordered, graceful deployment and scaling.

- Ordered, automated rolling updates.

  

> 之前接触的Pod的管理对象比如RC、Deployment、DaemonSet和Job都是面向无状态的服务，但是现实中有很多服务是有状态的，比如MySQL集群、MongoDB集群、ZK集群等，它们都有以下共同的特点：

- 每个节点都有固定的ID，通过该ID，集群中的成员可以互相发现并且通信
- 集群的规模是比较固定的，集群规模不能随意变动
- 集群里的每个节点都是有状态的，通常会持久化数据到永久存储中
- 如果磁盘损坏，则集群里的某个节点无法正常运行，集群功能受损

​     由于之前的RC/Deployment没办法满足要求，所以从Kubernetes v1.4版本就引入了PetSet资源对象，在v1.5版本时更名为StatefulSet。从本质上说，StatefulSet可以看作是Deployment/RC对象的特殊变种

- StatefulSet里的每个Pod都有稳定、唯一的网络标识，可以用来发现集群内其他的成员
- Pod的启动顺序是受控的，操作第n个Pod时，前n-1个Pod已经是运行且准备好的状态
- StatefulSet里的Pod采用稳定的持久化存储卷，通过PV/PVC来实现，删除Pod时默认不会删除与StatefulSet相关的存储卷
- StatefulSet需要与Headless Service配合使用

下面做一个实例：

>1、先创建一个yaml文件

```
vim nginx_statefulSet.yaml
```

```yaml
# 定义Service
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx
---
# 定义StatefulSet
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  selector:
    matchLabels:
      app: nginx 
  serviceName: "nginx"  
  replicas: 3 
  template:
    metadata:
      labels:
        app: nginx 
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
          name: web
```

>2、执行yaml文件

```
kubectl apply nginx_statefulSet.yaml
```



>3、查看

```shell
kubectl get pods -w    # 观察pod的创建顺序，以及pod的名字
```

```shell
[root@henry001 ~]#  kubectl get pods -w
NAME    READY   STATUS    RESTARTS   AGE
web-0   0/1     Pending   0          0s
web-0   0/1     Pending   0          0s
web-0   0/1     ContainerCreating   0          0s
web-0   0/1     ContainerCreating   0          1s
web-0   1/1     Running             0          14s
web-1   0/1     Pending             0          0s
web-1   0/1     Pending             0          0s
web-1   0/1     ContainerCreating   0          0s
web-1   0/1     ContainerCreating   0          1s
web-1   1/1     Running             0          15s
web-2   0/1     Pending             0          0s
web-2   0/1     Pending             0          0s
web-2   0/1     ContainerCreating   0          0s
web-2   0/1     ContainerCreating   0          1s
web-2   1/1     Running             0          6s
```

通过观察的信息我们可以看出，web-0，web-1，web-2是按照顺序，等前一个pod创建完成之后才开始创建下一个pod。

### 2 DaemonSet

> `官网`：https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/
>
> ```
> A DaemonSet ensures that all (or some) Nodes run a copy of a Pod. As nodes are added to the cluster, Pods are added to them. As nodes are removed from the cluster, those Pods are garbage collected. Deleting a DaemonSet will clean up the Pods it created.
> ```
>
> **DaemonSet应用场景**
>
> - 运行集群存储 daemon，例如在每个节点上运行 `glusterd`、`ceph`。
> - 在每个节点上运行日志收集 daemon，例如`fluentd`、`logstash`。
> - 在每个节点上运行监控 daemon，例如 [Prometheus Node Exporter](https://github.com/prometheus/node_exporter)、`collectd`、Datadog 代理、New Relic 代理，或 Ganglia `gmond`。

下面我来查看一下k8s自带的DaemonSet：

```shell
`查看k8s自带的daemonSet`
[root@henry001 controller]# kubectl get daemonSet -n kube-system
NAME          DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR                 AGE
calico-node   3         3         3       3            3           beta.kubernetes.io/os=linux   28m
kube-proxy    3         3         3       3            3           <none>                        47m
```

为什么这k8s集群在搭建完成后就会有这两个Daemonset呢，对应的yaml文件

```shell
#calico的yaml文件
......
kind: DaemonSet     #DaemonSet类型
apiVersion: apps/v1
metadata:
  name: calico-node
  namespace: kube-system
  labels:
    k8s-app: calico-node
spec:
  selector:
    matchLabels:
      k8s-app: calico-node
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  template:
    metadata:
      labels:
        k8s-app: calico-node
 ......       
```

下面再看一下生成的pod，calico以及kube-proxy在每一个Node上都存在

```shell
[root@henry001 ~]# kubectl get pod -n kube-system -o wide
NAME                                       READY   STATUS    RESTARTS   AGE   IP              NODE       NOMINATED NODE   READINESS GATES
calico-kube-controllers-594b6978c5-z48d2   1/1     Running   0          39m   192.168.132.3   henry001   <none>           <none>
calico-node-br7sq                          1/1     Running   0          38m   192.168.0.7     henry003   <none>           <none>
calico-node-hgk9h                          1/1     Running   0          39m   192.168.0.6     henry002   <none>           <none>
calico-node-mskwp                          1/1     Running   0          39m   192.168.0.8     henry001   <none>           <none>
coredns-fb8b8dccf-6nw9v                    1/1     Running   0          57m   192.168.132.1   henry001   <none>           <none>
coredns-fb8b8dccf-mcqrx                    1/1     Running   0          57m   192.168.132.2   henry001   <none>           <none>
etcd-henry001                              1/1     Running   0          57m   192.168.0.8     henry001   <none>           <none>
kube-apiserver-henry001                    1/1     Running   0          57m   192.168.0.8     henry001   <none>           <none>
kube-controller-manager-henry001           1/1     Running   0          56m   192.168.0.8     henry001   <none>           <none>
kube-proxy-f4hjd                           1/1     Running   0          39m   192.168.0.6     henry002   <none>           <none>
kube-proxy-lmflt                           1/1     Running   0          57m   192.168.0.8     henry001   <none>           <none>
kube-proxy-rt4vk                           1/1     Running   0          38m   192.168.0.7     henry003   <none>           <none>
kube-scheduler-henry001                    1/1     Running   0          56m   192.168.0.8     henry001   <none>           <none>
```

