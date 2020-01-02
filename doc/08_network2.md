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

### 4 Service-LoadBalance

通常需要第三方云提供商支持，有约束性

#### Ingress

> `官网`：<https://kubernetes.io/docs/concepts/services-networking/ingress/>
>
> `GitHub Ingress Nginx`:<https://github.com/kubernetes/ingress-nginx>
>
> `Nginx Ingress Controller`:<https://kubernetes.github.io/ingress-nginx/
>
> ```
> An API object that manages external access to the services in a cluster, typically HTTP.
> Ingress can provide load balancing, SSL termination and name-based virtual hosting.
> ```

Ingress exposes HTTP and HTTPS routes from outside the cluster to [services](https://kubernetes.io/docs/concepts/services-networking/service/) within the cluster. Traffic routing is controlled by rules defined on the Ingress resource.

```none
    internet
        |
   [ Ingress ]
   --|-----|--
   [ Services ]
```

可以发现，Ingress就是帮助我们访问集群内的服务的。为了彰显其优势，我们在使用Ingress之前，先以一个简单案例出发。

#### 4.1使用NodePort类型的service在K8S集群中部署tomcat

（也为了演示将service写在yaml文件中）

浏览器想要访问这个tomcat，也就是外部要访问该tomcat，用之前的Service-NodePort的方式是可以的，比如暴露一个<NodePort>端口，只需要访问 <NodeIP>:<NodePort>即可。

01 创建yaml文件

```vim my-tomcat.yaml
vim my-tomcat.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tomcat-deployment
  labels:
    app: tomcat
spec:
  replicas: 1
  selector:
    matchLabels:
      app: tomcat
  template:
    metadata:
      labels:
        app: tomcat
    spec:
      containers:
      - name: tomcat
        image: tomcat
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: tomcat-service
spec:
  ports:
  - port: 80   
    protocol: TCP
    targetPort: 8080
  selector:
    app: tomcat
  type: NodePort  
```

02 创建service

```shell
`创建pod、service`
[root@henry001 network]# kubectl apply -f my-tomcat.yaml 
deployment.apps/tomcat-deployment created
service/tomcat-service created

`查看service`
[root@henry001 network]# kubectl get svc
NAME                TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
kubernetes          ClusterIP   10.96.0.1        <none>        443/TCP          42h
tomcat-service      NodePort    10.106.112.183   <none>        80:30747/TCP     31s
whoami-deployment   NodePort    10.103.129.91    <none>        8000:31999/TCP   42m

`查看pod`
[root@henry001 network]# kubectl get pods
NAME                                 READY   STATUS              RESTARTS   AGE
tomcat-deployment-6b9d6f8547-6mmh2   1/1     Running   0          69s
tomcat-deployment-6b9d6f8547-79nck   1/1     Running   0          69s
tomcat-deployment-6b9d6f8547-c8bps   1/1     Running   0          69s

`查看deployment`
[root@henry001 network]# kubectl get deploy
NAME                READY   UP-TO-DATE   AVAILABLE   AGE
nginx               1/1     1            1           24h
tomcat-deployment   3/3     3            3           97s
```

显然，Service-NodePort的方式生产环境不推荐使用，那接下来就基于上述需求，使用Ingress实现访问tomcat的需求。下面就开始讲解使用ingress插件来实现外网访问集群pod。



#### 4.2 使用ingress实现

4.2.1架构图

  ![image-20200102163429809](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200102163429809.png)

说明：

* 本文中采用的ingress-controller是nginx-ingress-controller，具体详情可以参考官网：https://www.nginx.com/products/nginx/kubernetes-ingress-controller；

* 大家也可以根据自己需要采用不同的ingress-controller，可参考https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/

  

4.2.2 实例

> (1)以Deployment方式创建Pod，该Pod为Ingress Nginx Controller，要想让外界访问，可以通过Service的NodePort或者HostPort方式，这里选择HostPort，比如指定henry002机器上运行：

```shell
# 确保nginx-controller运行到henry002节点上
kubectl label node henry002 name=ingress   

`先下载mandatory.yaml文件，下载地址：https://github.com/kubernetes/ingress-nginx/blob/nginx-0.20.0/deploy/mandatory.yaml，并对mandatory.yaml并进行修改，如下：
# 使用HostPort方式运行，需要增加配置
hostNetwork: true   #使用hostport
      nodeSelector:  
        name: ingress  #指定节点
# 搜索nodeSelector，并且要确保henry002节点上的80和443端口没有被占用，镜像拉取需要较长的时间，这块要特别注意一下

#运行mandatory.yaml
kubectl apply -f mandatory.yaml  

#查看ingress-nginx命名空间下的资源
kubectl get all -n ingress-nginx
```

> (2)查看henry002的80和443端口

```
lsof -i tcp:80
lsof -i tcp:443
```

> (3)创建tomcat的pod和service

记得将之前的tomcat删除：kubectl delete -f my-tomcat.yaml

```
vim tomcat.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tomcat-deployment
  labels:
    app: tomcat
spec:
  replicas: 3
  selector:
    matchLabels:
      app: tomcat
  template:
    metadata:
      labels:
        app: tomcat
    spec:
      containers:
      - name: tomcat
        image: tomcat
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: tomcat-service
spec:
  ports:
  - port: 80   
    protocol: TCP
    targetPort: 8080
  selector:
    app: tomcat
```

```shell
`执行yaml文件`
[root@henry001 network]# kubectl apply -f tomcat-ingress.yaml
deployment.apps/tomcat-deployment created
service/tomcat-service created

`查看service`
[root@henry001 network]# kubectl get svc 
NAME                TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
kubernetes          ClusterIP   10.96.0.1        <none>        443/TCP          43h
tomcat-service      ClusterIP   10.101.231.253   <none>        80/TCP           20s
whoami-deployment   NodePort    10.103.129.91    <none>        8000:31999/TCP   150m

`查看pod`
[root@henry001 network]# kubectl get pods
NAME                                 READY   STATUS    RESTARTS   AGE
tomcat-deployment-6b9d6f8547-8wxgx   1/1     Running   0          42s
tomcat-deployment-6b9d6f8547-hrhrr   1/1     Running   0          42s
tomcat-deployment-6b9d6f8547-p7zhz   1/1     Running   0          42s
```



kubectl get svc 

kubectl get pods

> (4)创建Ingress以及定义转发规则

1>创建 nginx-ingress.yaml文件

```shell
vim  nginx-ingress.yaml
```



```yaml
#ingress
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: nginx-ingress
spec:
  rules:
  - host: tomcat.henry.com
    http:
      paths:
      - path: /
        backend:
          serviceName: tomcat-service
          servicePort: 80
```

2>创建ingress并查看

```shell
`创建ingress`
[root@henry001 network]# kubectl apply -f nginx-ingress.yaml 
ingress.extensions/nginx-ingress created

`查看ingress`
[root@henry001 network]# kubectl get ingress
NAME            HOSTS              ADDRESS   PORTS   AGE
nginx-ingress   tomcat.henry.com             80      47s

`查看ingress详细信息`
[root@henry001 network]# kubectl describe ingress nginx-ingress
Name:             nginx-ingress
Namespace:        default
Address:          
Default backend:  default-http-backend:80 (<none>)
Rules:
  Host              Path  Backends
  ----              ----  --------
  tomcat.henry.com  
                    /   tomcat-service:80 (192.168.217.22:8080,192.168.254.215:8080,192.168.254.216:8080)
Annotations:
  kubectl.kubernetes.io/last-applied-configuration:  {"apiVersion":"extensions/v1beta1","kind":"Ingress","metadata":{"annotations":{},"name":"nginx-ingress","namespace":"default"},"spec":{"rules":[{"host":"tomcat.henry.com","http":{"paths":[{"backend":{"serviceName":"tomcat-service","servicePort":80},"path":"/"}]}}]}}

Events:
  Type    Reason  Age   From                      Message
  ----    ------  ----  ----                      -------
  Normal  CREATE  53s   nginx-ingress-controller  Ingress default/nginx-ingress

```



> (5)修改win的hosts文件，添加dns解析

```
182.92.105.161 tomcat.henry.com
```

> (6)打开浏览器，访问tomcat.henry.com

![image-20200102180102084](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200102180102084.png)

`总结`：如果以后想要使用Ingress网络，其实只要定义ingress，service和pod即可，前提是要保证nginx ingress controller已经配置好了。

