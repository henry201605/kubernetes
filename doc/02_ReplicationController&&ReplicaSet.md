前一章讲解了Pod的属性，后续的几章节会陆续介绍管理Pod的Controller，主要包括ReplicationController、ReplicaSet、Deployment、StatefulSets、DaemonSet、Jobs及CronJob，下面开始我们的学习之旅，先来看下官网：

> `官网`：<https://kubernetes.io/docs/concepts/workloads/controllers/>

### 1、ReplicationController(RC)

> `官网`：https://kubernetes.io/docs/concepts/workloads/controllers/replicationcontroller/
>
> ```
> A ReplicationController ensures that a specified number of pod replicas are running at any one time. In other words, a ReplicationController makes sure that a pod or a homogeneous set of pods is always up and available.
> ```

ReplicationController定义了一个期望的场景，即声明某种Pod的副本数量在任意时刻都符合某个预期值，所以RC的定义包含以下几个部分：

- Pod期待的副本数（replicas）
- 用于筛选目标Pod的Label Selector
- 当Pod的副本数量小于预期数量时，用于创建新Pod的Pod模板（template）

也就是说通过RC实现了集群中Pod的高可用，减少了传统IT环境中手工运维的工作。

接下来做一个案例：

> (1)创建名为nginx_replication.yaml

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: nginx
spec:
  replicas: 3
  selector:
    app: nginx
  template:
    metadata:
      name: nginx
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
```

> yaml文件解析：

* kind：表示要新建对象的类型
* spec.selector：表示需要管理的Pod的label，这里表示包含app: nginx的label的Pod都会被该RC管理
* spec.replicas：表示受此RC管理的Pod需要运行的副本数
* spec.template：表示用于定义Pod的模板，比如Pod名称、拥有的label以及Pod中运行的应用等



> (2)根据nginx_replication.yaml创建pod
>
> 此时k8s会在所有可用的Node上，创建3个Pod，并且每个Pod都有一个app: nginx的label，同时每个Pod中都运行了一个nginx容器。

```
kubectl apply -f nginx_replication.yaml
```

> (3)查看pod

```shell
#获取replicationController
[root@henry001 ~]# kubectl get rc
NAME    DESIRED   CURRENT   READY   AGE
nginx   3         3         3       4m25s

#获取pod在各Node上的运行情况
[root@henry001 ~]# kubectl get pods -o wide
NAME          READY   STATUS    RESTARTS   AGE    IP                NODE       NOMINATED NODE   READINESS GATES
nginx-58v6t   1/1     Running   0          5m7s   192.168.254.193   henry003   <none>           <none>
nginx-rqb7q   1/1     Running   0          5m7s   192.168.217.2     henry002   <none>           <none>
nginx-xc64w   1/1     Running   0          5m7s   192.168.254.194   henry003   <none>           <none>

```

> (4)尝试删除一个pod
>
> 如果某个Pod发生问题，Controller Manager能够及时发现，然后根据RC的定义，创建一个新的Pod

```shell
#删除其中一个pod
[root@henry001 ~]# kubectl delete pod  nginx-58v6t
pod "nginx-58v6t" deleted

#发现henry003Node上的名为nginx-58v6t的pod被删除，但又在henry002的Node上生成了一个新的名为nginx-vtl84的pod
[root@henry001 ~]# kubectl get pods -o wide
NAME          READY   STATUS    RESTARTS   AGE     IP                NODE       NOMINATED NODE   READINESS GATES
nginx-rqb7q   1/1     Running   0          8m45s   192.168.217.2     henry002   <none>           <none>
nginx-vtl84   1/1     Running   0          28s     192.168.217.3     henry002   <none>           <none>
nginx-xc64w   1/1     Running   0          8m45s   192.168.254.194   henry003   <none>           <none>

```

> (5)对pod进行扩缩容

```shell
#对nginx进行扩容到6个
[root@henry001 ~]# kubectl scale rc nginx --replicas=6
replicationcontroller/nginx scaled

#查看到在两个Node上一共生成了6个pod
[root@henry001 ~]# kubectl get pods -o wide
NAME          READY   STATUS    RESTARTS   AGE     IP                NODE       NOMINATED NODE   READINESS GATES
nginx-4nzdl   1/1     Running   0          47s     192.168.254.195   henry003   <none>           <none>
nginx-nmm76   1/1     Running   0          47s     192.168.217.4     henry002   <none>           <none>
nginx-rh88z   1/1     Running   0          47s     192.168.254.196   henry003   <none>           <none>
nginx-rqb7q   1/1     Running   0          16m     192.168.217.2     henry002   <none>           <none>
nginx-vtl84   1/1     Running   0          8m29s   192.168.217.3     henry002   <none>           <none>
nginx-xc64w   1/1     Running   0          16m     192.168.254.194   henry003   <none>           <none>


```



> (6)删除pod

```shell
kubectl delete -f nginx_replication.yaml
```





### ReplicaSet(RS)

> `官网`：https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/
>
> ```
> A ReplicaSet’s purpose is to maintain a stable set of replica Pods running at any given time. As such, it is often used to guarantee the availability of a specified number of identical Pods.
> ```
>
> 在Kubernetes v1.2时，RC就升级成了另外一个概念：Replica Set，官方解释为“下一代RC”
>
> ReplicaSet和RC没有本质的区别，kubectl中绝大部分作用于RC的命令同样适用于RS
>
> RS与RC唯一的区别是：RS支持基于集合的Label Selector（Set-based selector），而RC只支持基于等式的Label Selector（equality-based selector），这使得Replica Set的功能更强

**Have a try**

```yaml
apiVersion: extensions/v1beta1
kind: ReplicaSet
metadata:
  name: frontend
spec:
  matchLabels: 
    tier: frontend
  matchExpressions: 
    - {key:tier,operator: In,values: [frontend]}
  template:
  ...
```

`注意`：一般情况下，我们很少单独使用Replica Set，它主要是被Deployment这个更高的资源对象所使用，从而形成一整套Pod创建、删除、更新的编排机制。当我们使用Deployment时，无须关心它是如何创建和维护Replica Set的，这一切都是自动发生的。同时，无需担心跟其他机制的不兼容问题（比如ReplicaSet不支持rolling-update但Deployment支持）。

### Deployment

> `官网`：https://kubernetes.io/docs/concepts/workloads/controllers/deployment/
>
> ```
> A Deployment provides declarative updates for Pods and ReplicaSets.
> 
> You describe a desired state in a Deployment, and the Deployment Controller changes the actual state to the desired state at a controlled rate. You can define Deployments to create new ReplicaSets, or to remove existing Deployments and adopt all their resources with new Deployments.
> ```
>
> Deployment相对RC最大的一个升级就是我们可以随时知道当前Pod“部署”的进度。
>
> 创建一个Deployment对象来生成对应的Replica Set并完成Pod副本的创建过程
>
> 检查Deploymnet的状态来看部署动作是否完成（Pod副本的数量是否达到预期的值）

> (1)创建nginx_deployment.yaml文件

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```

> (2)根据nginx_deployment.yaml文件创建pod

```shell
kubectl apply -f nginx_deployment.yaml
```

> (3)查看pod
>
> kubectl get pods -o wide
>
> kubectl get deployment
>
> kubectl get rs
>
> kubectl get deployment -o wide

```
nginx-deployment-6dd86d77d-f7dxb   1/1     Running   0      22s   192.168.80.198   w2 
nginx-deployment-6dd86d77d-npqxj   1/1     Running   0      22s   192.168.190.71   w1 
nginx-deployment-6dd86d77d-swt25   1/1     Running   0      22s   192.168.190.70   w1
```

nginx-deployment[deployment]-6dd86d77d[replicaset]-f7dxb[pod] 

> (4)当前nginx的版本

```
kubectl get deployment -o wide

NAME    READY   UP-TO-DATE   AVAILABLE   AGE     CONTAINERS   IMAGES      SELECTOR
nginx-deployment   3/3         3     3  3m27s      nginx    nginx:1.7.9   app=nginx
```

> (5)更新nginx的image版本

```
kubectl set image deployment nginx-deployment nginx=nginx:1.9.1
```

## 



> (6)升级镜像

通过改变RC里Pod模板中的镜像版本，可以实现Pod的升级功能

* 先查看nginx的镜像版本，用的是最新的：

  ```
  nginx                                latest              231d40e811cd        4 weeks ago         126MB
  ```

* 更新镜像

kubectl set image rc nginx=nginx:1.9 

kubectl set image deploy image-deployment  *=registry.cn-beijing.aliyuncs.com/mrvolleyball/nginx:v2



将deployment中的nginx容器镜像设置为“nginx：1.9.1”。

```
kubectl set image deployment/nginx busybox=busybox nginx=nginx:1.9.1
```

所有deployment和rc的nginx容器镜像更新为“nginx：1.9.1”

```
kubectl set image deployments,rc nginx=nginx:1.9.1 --all
```

将daemonset abc的所有容器镜像更新为“nginx：1.9.1”

```
kubectl set image daemonset abc *=nginx:1.9.1
```

从本地文件中更新nginx容器镜像

```
kubectl set image -f path/to/file.yaml nginx=nginx:1.9.1 --local -o yaml
```



