### 4 Pod访问外部服务

比较简单，没太多好说的内容，直接访问即可

### 5 外部服务访问集群中的Pod

#### Service-NodePort

> 也是Service的一种类型，可以通过NodePort的方式
>
> 说白了，因为外部能够访问到集群的物理机器IP，所以就是在集群中每台物理机器上暴露一个相同的IP，比如32008

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

```
kubectl delete svc whoami-deployment

kubectl expose deployment whoami-deployment --type=NodePort

[root@master-kubeadm-k8s ~]# kubectl get svc
NAME                TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
kubernetes          ClusterIP   10.96.0.1      <none>        443/TCP          21h
whoami-deployment   NodePort    10.99.108.82   <none>        8000:32041/TCP   7s
```

> (3)注意上述的端口32041，实际上就是暴露在集群中物理机器上的端口

```
lsof -i tcp:32041
netstat -ntlp|grep 32041
```

> (4)浏览器通过物理机器的IP访问

```
http://192.168.0.51:32041
curl 192.168.0.61:32041
```

`conclusion`：NodePort虽然能够实现外部访问Pod的需求，但是真的好吗？其实不好，占用了各个物理主机上的端口

#### Service-LoadBalance

通常需要第三方云提供商支持，有约束性

#### Ingress

> `官网`：<https://kubernetes.io/docs/concepts/services-networking/ingress/>
>
> ```
> An API object that manages external access to the services in a cluster, typically HTTP.
> 
> Ingress can provide load balancing, SSL termination and name-based virtual hosting.
> ```

![](F:/work/kubernetes1117/images/29.png)

> 可以发现，Ingress就是帮助我们访问集群内的服务的。不过在看Ingress之前，我们还是先以一个案例出发。
>
> 很简单，在K8S集群中部署tomcat

浏览器想要访问这个tomcat，也就是外部要访问该tomcat，用之前的Service-NodePort的方式是可以的，比如暴露一个32008端口，只需要访问192.168.0.61:32008即可。

vi my-tomcat.yaml

kubectl apply -f my-tomcat.yaml

kubectl get pods

kubectl get deployment

kubectl get svc

`tomcat-service   NodePort    10.105.51.97   <none>        80:31032/TCP   37s`

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

> 显然，Service-NodePort的方式生产环境不推荐使用，那接下来就基于上述需求，使用Ingress实现访问tomcat的需求。
>
> `官网Ingress`:<https://kubernetes.io/docs/concepts/services-networking/ingress/>
>
> `GitHub Ingress Nginx`:<https://github.com/kubernetes/ingress-nginx>
>
> `Nginx Ingress Controller`:<https://kubernetes.github.io/ingress-nginx/

> (1)以Deployment方式创建Pod，该Pod为Ingress Nginx Controller，要想让外界访问，可以通过Service的NodePort或者HostPort方式，这里选择HostPort，比如指定worker01运行

```shell
# 确保nginx-controller运行到w1节点上
kubectl label node w1 name=ingress   

# 使用HostPort方式运行，需要增加配置
hostNetwork: true

# 搜索nodeSelector，并且要确保w1节点上的80和443端口没有被占用，镜像拉取需要较长的时间，这块注意一下哦
# mandatory.yaml在网盘中的“课堂源码”目录
kubectl apply -f mandatory.yaml  

kubectl get all -n ingress-nginx
```

> (2)查看**w1**的80和443端口

```
lsof -i tcp:80
lsof -i tcp:443
```

> (3)创建tomcat的pod和service
>
> > 记得将之前的tomcat删除：kubectl delete -f my-tomcat.yaml
>
> vi tomcat.yaml
>
> kubectl apply -f tomcat.yaml
>
> kubectl get svc 
>
> kubectl get pods

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
```

> (4)创建Ingress以及定义转发规则
>
> kubectl apply -f nginx-ingress.yaml
>
> kubectl get ingress
>
> kubectl describe ingress nginx-ingress

```yaml
#ingress
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: nginx-ingress
spec:
  rules:
  - host: tomcat.jack.com
    http:
      paths:
      - path: /
        backend:
          serviceName: tomcat-service
          servicePort: 80
```

> (5)修改win的hosts文件，添加dns解析

```
192.168.8.61 tomcat.jack.com
```

> (6)打开浏览器，访问tomcat.jack.com

`总结`：如果以后想要使用Ingress网络，其实只要定义ingress，service和pod即可，前提是要保证nginx ingress controller已经配置好了。

