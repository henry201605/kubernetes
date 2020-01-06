`Service官网`：<https://kubernetes.io/docs/concepts/services-networking/service/>

```
An abstract way to expose an application running on a set of Pods as a network service.
With Kubernetes you don’t need to modify your application to use an unfamiliar service discovery mechanism. Kubernetes gives Pods their own IP addresses and a single DNS name for a set of Pods, and can load-balance across them.
```



### 1 Service 类型

在定义Service的时候可以指定一个自己需要的类型的Service，如果不指定的话默认是ClusterIP类型。

可以使用的服务类型如下：

1、`ClusterIP`：通过集群的内部 IP 暴露服务，选择该值，服务只能够在集群内部可以访问，这也是默认的Service类型。ClusterIP类型的service创建时，k8s会通过etcd从可分配的IP池中分配一个IP，该IP全局唯一，且不可修改。所有访问该IP的请求，都会被iptables转发到后端的endpoints中。

 2、`NodePort`：通过每个 Node 节点上的 IP 和静态端口（NodePort）暴露服务。NodePort 服务会路由到 ClusterIP 服务，这个 ClusterIP 服务会自动创建。通过请求 <NodeIP>:<NodePort>，可以从集群的外部访问一个 NodePort 服务。

 3、`LoadBalancer`：需要外部支持（GCP and Azure），用户访问service.spec.external-ip,该IP对应到一个外部负载均衡的vip，外部服务对这个vip的请求，会被loadbalancer通过健康检查和转发，发送到一个运行着该服务pod的node上，并同样通过nodePort里的端口映射，发送给容器。

 4、`ExternalName`：用户可以指定一个任意的名字，作为该service被解析的CNAME,这种类型的servcie不用指定clusterIP，因此kube-proxy不会管理这类service，这类service需要使用1.7版本以上的kubedns。



### 2 Cluster IP

>  (1)创建whoami-deployment.yaml文件

```
vim whoami-deployment.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: whoami-deployment
  labels:
    app: whoami
spec:
  replicas: 3
  selector:
    matchLabels:
      app: whoami
  template:
    metadata:
      labels:
        app: whoami
    spec:
      containers:
      - name: whoami
        image: jwilder/whoami
        ports:
        - containerPort: 8000
```



> (2)运行yaml文件并查看pod以及service



```shell
[root@henry001 network]# kubectl apply -f whoami-deployment.yaml 
deployment.apps/whoami-deployment created

`查看生成的pod详细信息`
[root@henry001 network]# kubectl get pods -o wide
NAME                                 READY   STATUS    RESTARTS   AGE     IP                NODE       NOMINATED NODE   READINESS GATES
whoami-deployment-678b64444d-lgvck   1/1     Running   0          28s     192.168.254.207   henry003   <none>           <none>
whoami-deployment-678b64444d-mszwp   1/1     Running   0          28s     192.168.217.18    henry002   <none>           <none>
whoami-deployment-678b64444d-xz66k   1/1     Running   0          28s     192.168.254.208   henry003   <none>           <none>

`查看service，此时并未有whoami相关的service`
[root@henry001 network]# kubectl get svc
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   23h
```



> (3)在集群内正常访问

```
curl 192.168.254.207:8000
curl 192.168.217.18:8000
curl 192.168.254.208:8000
```

> (4)创建whoami的service
>
> `注意`：该地址只能在集群内部访问

```shell
`为deployment的whoami-deployment创建service`
[root@henry001 network]# kubectl expose deployment whoami-deployment
service/whoami-deployment exposed

`查看service`
[root@henry001 network]# kubectl get svc
NAME                TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
kubernetes          ClusterIP   10.96.0.1       <none>        443/TCP    24h
whoami-deployment   ClusterIP   10.97.233.149   <none>        8000/TCP   12s

```

**可以发现有一个Cluster IP类型的service，名称为whoami-deployment，IP地址为10.97.233.149

> (5)通过Service的Cluster IP访问

```shell
[root@henry001 network]# curl 10.97.233.149:8000
I'm whoami-deployment-678b64444d-xz66k
[root@henry001 network]# curl 10.97.233.149:8000
I'm whoami-deployment-678b64444d-lgvck
[root@henry001 network]# curl 10.97.233.149:8000
I'm whoami-deployment-678b64444d-mszwp
```

> (6)具体查看一下whoami-deployment的详情信息，发现有一个Endpoints连接了具体3个Pod

```shell
[root@henry001 network]# kubectl describe svc whoami-deployment
Name:              whoami-deployment
Namespace:         default
Labels:            app=whoami
Annotations:       <none>
Selector:          app=whoami
Type:              ClusterIP
IP:                10.97.233.149
Port:              <unset>  8000/TCP
TargetPort:        8000/TCP
Endpoints:         192.168.217.18:8000,192.168.254.207:8000,192.168.254.208:8000
Session Affinity:  None
Events:            <none>
```

> (7)下面通过deployment对whoami扩容成5个

```
kubectl scale deployment whoami-deployment --replicas=5
```

```shell
`查看pod,已经扩容为5个`
[root@henry001 network]# kubectl get pods
NAME                                 READY   STATUS    RESTARTS   AGE     IP                NODE       NOMINATED NODE   READINESS GATES
whoami-deployment-678b64444d-lgvck   1/1     Running   0          23m     192.168.254.207   henry003   <none>           <none>
whoami-deployment-678b64444d-mszwp   1/1     Running   0          23m     192.168.217.18    henry002   <none>           <none>
whoami-deployment-678b64444d-q4wzx   1/1     Running   0          4m50s   192.168.254.209   henry003   <none>           <none>
whoami-deployment-678b64444d-xz66k   1/1     Running   0          23m     192.168.254.208   henry003   <none>           <none>
whoami-deployment-678b64444d-zj82s   1/1     Running   0          4m50s   192.168.217.19    henry002   <none>           <none>


```

> (8)再次访问：curl 10.97.233.149:8000

```shell
[root@henry001 network]# curl 10.97.233.149:8000
I'm whoami-deployment-678b64444d-mszwp
[root@henry001 network]# curl 10.97.233.149:8000
I'm whoami-deployment-678b64444d-xz66k
[root@henry001 network]# curl 10.97.233.149:8000
I'm whoami-deployment-678b64444d-q4wzx
[root@henry001 network]# curl 10.97.233.149:8000
I'm whoami-deployment-678b64444d-zj82s
[root@henry001 network]# curl 10.97.233.149:8000
I'm whoami-deployment-678b64444d-xz66k
[root@henry001 network]# curl 10.97.233.149:8000
I'm whoami-deployment-678b64444d-lgvck
```



> (9)再次查看service具体信息：kubectl describe svc whoami-deployment

```shell
[root@henry001 network]# kubectl describe svc whoami-deployment
Name:              whoami-deployment
Namespace:         default
Labels:            app=whoami
Annotations:       <none>
Selector:          app=whoami
Type:              ClusterIP
IP:                10.97.233.149
Port:              <unset>  8000/TCP
TargetPort:        8000/TCP
Endpoints:         192.168.217.18:8000,192.168.217.19:8000,192.168.254.207:8000 + 2 more...    #这里有5个pod的信息
Session Affinity:  None
Events:            <none>

```

> (10)其实对于Service的创建，不仅仅可以使用kubectl expose，也可以定义一个yaml文件

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: MyApp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
  type: Cluster
```

`总结`：其实Service存在的意义就是为了Pod的不稳定性，而上述探讨的就是关于Service的一种类型Cluster IP，只能供集群内访问。

###  3 NodePort

> 因为外部能够访问到集群的物理机器IP，所以就是在集群中每台物理机器上暴露一个相同的IP，从给定的配置范围内（默认：30000-32767）分配端口

> (1)根据whoami-deployment.yaml创建pod

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: whoami-deployment
  labels:
    app: whoami
spec:
  replicas: 3
  selector:
    matchLabels:
      app: whoami
  template:
    metadata:
      labels:
        app: whoami
    spec:
      containers:
      - name: whoami
        image: jwilder/whoami
        ports:
        - containerPort: 8000
```

> (2)创建NodePort类型的service，名称为whoami-deployment

```shell
`查看service`
[root@henry001 network]# kubectl get svc
NAME                TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
kubernetes          ClusterIP   10.96.0.1       <none>        443/TCP    41h
whoami-deployment   ClusterIP   10.97.233.149   <none>        8000/TCP   17h

`删除ClusterIP类型的service`
[root@henry001 network]# kubectl delete svc whoami-deployment
service "whoami-deployment" deleted

`创建NodePort类型的service`
[root@henry001 network]# kubectl expose deployment whoami-deployment --type=NodePort
service/whoami-deployment exposed
[root@henry001 network]# kubectl get svc
NAME                TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
kubernetes          ClusterIP   10.96.0.1       <none>        443/TCP          41h
whoami-deployment   NodePort    10.103.129.91   <none>        8000:31999/TCP   3s

```

> (3)注意上述的端口31999，实际上就是暴露在集群中物理机器上的端口

```shell
`在每台机器上查看一下31999端口监听情况`
[root@henry001 network]# lsof -i tcp:31999
COMMAND     PID USER   FD   TYPE   DEVICE SIZE/OFF NODE NAME
kube-prox 11693 root   11u  IPv6 15525671      0t0  TCP *:31999 (LISTEN)
[root@henry001 network]# netstat -nlp |grep 31999
tcp6       0      0 :::31999                :::*                    LISTEN      11693/kube-proxy
```

> (4)浏览器通过物理机器的IP访问

* 用service访问的时候端口号为8000；
* 使用内网Ip或者外网Ip访问时需要用端口号31999

```shell
`master节点`
[root@henry001 network]# curl 192.168.0.8:31999
I'm whoami-deployment-678b64444d-q9mzt
[root@henry001 network]# curl 10.103.129.91:8000
I'm whoami-deployment-678b64444d-q9mzt

`henry002节点`
[root@henry002 ~]# curl 10.103.129.91:8000
I'm whoami-deployment-678b64444d-89dzd
[root@henry002 ~]# curl 192.168.0.8:31999
I'm whoami-deployment-678b64444d-q9mzt

`henry003节点`
[root@henry003 ~]# curl 10.103.129.91:8000
I'm whoami-deployment-678b64444d-d7w64
[root@henry003 ~]# curl 192.168.0.7:31999
I'm whoami-deployment-678b64444d-89dzd

`集群外机器`
[root@w1 ~]# curl 182.92.168.144:31999
I'm whoami-deployment-678b64444d-d7w64

```

* 使用浏览器访问：

![image-20200102154935787](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200102154935787.png)

`总结`：NodePort虽然能够实现外部访问Pod的需求，但这种方法有许多缺点：

1. 每个端口只能是一种服务
2. 端口范围只能是 30000-32767
3. 如果节点/VM 的 IP 地址发生变化，你需要能处理这种情况

基于以上原因，我不建议在生产环境上用这种方式暴露服务。如果你运行的服务不要求一直可用，或者对成本比较敏感，你可以使用这种方法。这样的应用的最佳例子是 demo 应用，或者某些临时应用。
