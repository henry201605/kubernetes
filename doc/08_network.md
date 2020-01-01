## Network

### 1 同一个Pod中的容器通信

我们都知道K8S最小的操作单位是Pod，先思考一下同一个Pod中多个容器要进行通信，我们看一下官网中给出的一个描述：

```
Each Pod is assigned a unique IP address. Every container in a Pod shares the network namespace, including the IP address and network ports. 
```

由官网的这段话可以看出，同一个pod中的容器是共享网络ip地址和端口号的，通信显然没问题

那如果是通过容器的名称进行通信呢？就需要将所有pod中的容器加入到同一个容器的网络中，我们把该容器称作为pod中的pause container。

![image-20200101170650692](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200101170650692.png)



### 2 集群内Pod之间的通信

Pod会有独立的IP地址，这个IP地址是被Pod中所有的Container共享的，那多个Pod之间的通信能通过这个IP地址吗？我们可以将此类pod间的通信分为两个维度：

* 集群中同一台机器中的Pod；
* 集群中不同机器中的Pod

**准备两个pod，一个nginx，一个busybox**

> 01 新建nginx_pod.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
  - name: nginx-container
    image: nginx
    ports:
    - containerPort: 80
```



> 02 新建busybox_pod.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox
  labels:
    app: busybox
spec:
  containers:
  - name: busybox
    image: busybox
    command: ['sh', '-c', 'echo The app is running! && sleep 3600']
```



03执行yaml文件，并且查看运行情况

```shell
`创建pod`
[root@henry001 network]# kubectl apply -f nginx_pod.yaml
pod/nginx-pod created
[root@henry001 network]# vim busybox_pod.yaml 
[root@henry001 network]# kubectl apply -f busybox_pod.yaml 
pod/busybox created

`查看运行情况`
[root@henry001 network]# kubectl get pods -o wide
NAME                     READY   STATUS    RESTARTS   AGE     IP                NODE       NOMINATED NODE   READINESS GATES
busybox                  1/1     Running   0          3m19s   192.168.254.206   henry003   <none>           <none>
mypod                    1/1     Running   0          99m     192.168.217.15    henry002   <none>           <none>
nginx-7db9fccd9b-nxnvf   1/1     Running   0          27s     192.168.217.17    henry002   <none>           <none>
nginx-pod                1/1     Running   0          4m18s   192.168.217.16    henry002   <none>           <none>

```

`发现`：nginx-pod的ip为192.168.217.16     busybox-pod的ip为192.168.254.206

#### 2.1同一个集群中同一台机器

> 进入到henry002这台机器的mypod中

```shell
`进入mypod`
[root@henry001 network]# kubectl exec -it mypod bash
`访问nginx-pod，可以访问`
root@mypod:/data# curl 192.168.217.16
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
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



#### 2.2同一个集群中不同机器

![image-20200101202421901](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200101202421901.png)

> (1)来到henry003节点的机器上：ping 192.168.217.16

```shell
`可以ping通henry003上的busybox的pod的ip`
[root@henry003 ~]# ping 192.168.217.16
PING 192.168.217.16 (192.168.217.16) 56(84) bytes of data.
64 bytes from 192.168.217.16: icmp_seq=1 ttl=63 time=0.369 ms
64 bytes from 192.168.217.16: icmp_seq=2 ttl=63 time=0.262 ms
64 bytes from 192.168.217.16: icmp_seq=3 ttl=63 time=0.386 ms

```

> (2)在节点为henry003的机器上：curl 192.168.217.16，同样可以访问nginx

```shell
[root@henry003 ~]# curl 192.168.217.16
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
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

> (3)来到master节点：

ping/curl 192.168.217.16          访问的是henry002上的nginx-pod

ping 192.168.254.206         访问的是henry003上的busybox-pod



> (4)来到henry002：

ping 192.168.254.206         访问的是henry003上的busybox-pod



**原理：**

集群中不同机器之间的通信是通过网络插件实现的，本文中使用的是Calico插件，具体的可以参考官网：

`链接`：<https://kubernetes.io/docs/concepts/cluster-administration/networking/#the-kubernetes-network-model>



### 3 集群内Service-Cluster IP

​      对于上述的Pod虽然实现了集群内部互相通信，但是Pod是不稳定的，比如通过Deployment管理Pod，随时可能对Pod进行扩缩容，这时候Pod的IP地址是变化的。能够有一个固定的IP，使得集群内能够访问。也就是之前在架构描述的时候所提到的，能够把相同或者具有关联的Pod，打上Label，组成Service。而Service有固定的IP，不管Pod怎么创建和销毁，都可以通过Service的IP进行访问。

> `Service官网`：<https://kubernetes.io/docs/concepts/services-networking/service/>
>
> ```
> An abstract way to expose an application running on a set of Pods as a network service.
> With Kubernetes you don’t need to modify your application to use an unfamiliar service discovery mechanism. Kubernetes gives Pods their own IP addresses and a single DNS name for a set of Pods, and can load-balance across them.
> ```

> (1)创建whoami-deployment.yaml文件

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

`conclusion`：其实Service存在的意义就是为了Pod的不稳定性，而上述探讨的就是关于Service的一种类型Cluster IP，只能供集群内访问。后续将继续探讨集群中的Pod访问外部服务，以及外部服务访问集群中的Pod。