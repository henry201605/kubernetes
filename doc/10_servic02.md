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

