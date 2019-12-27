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

```shell
#查看pod的详细信息
[root@henry001 ~]# kubectl get pods -o wide
NAME                               READY   STATUS    RESTARTS   AGE   IP                NODE       NOMINATED NODE   READINESS GATES
nginx-deployment-6dd86d77d-7kwn5   1/1     Running   0          65s   192.168.217.8     henry002   <none>           <none>
nginx-deployment-6dd86d77d-dbghj   1/1     Running   0          65s   192.168.254.200   henry003   <none>           <none>
nginx-deployment-6dd86d77d-jv8zm   1/1     Running   0          65s   192.168.217.9     henry002   <none>           <none>

#查看deployment
[root@henry001 ~]# kubectl get deployment
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3/3     3            3           79s

#查看replicaSet
[root@henry001 ~]# kubectl get rs
NAME                         DESIRED   CURRENT   READY   AGE
nginx-deployment-6dd86d77d   3         3         3       86s
```





kubectl get deployment

kubectl get rs

kubectl get deployment -o wide

```
nginx-deployment-6dd86d77d-f7dxb   1/1     Running   0      22s   192.168.80.198   w2 
nginx-deployment-6dd86d77d-npqxj   1/1     Running   0      22s   192.168.190.71   w1 
nginx-deployment-6dd86d77d-swt25   1/1     Running   0      22s   192.168.190.70   w1
```

> (4)查看当前nginx的版本

```shell
#查看deployment的详细信息
[root@henry001 ~]# kubectl get deployment -o wide
NAME               READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES        SELECTOR
nginx-deployment   3/3     3            3           97s   nginx        nginx:1.7.9   app=nginx
```

> (5)更新nginx的image版本

```shell
#升级nginx的镜像 1.7.9————>1.9.1
[root@henry001 ~]# kubectl set image deployment nginx-deployment nginx=nginx:1.9.1
deployment.extensions/nginx-deployment image updated

#查看deployment，发现镜像已经更新为1.9.1
[root@henry001 ~]# kubectl get deployment -o wide
NAME               READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES        SELECTOR
nginx-deployment   3/3     1            3           10m   nginx        nginx:1.9.1   app=nginx

```

-----------------------------

> 对于镜像更新有一些方法：

* 将deployment中的nginx容器镜像设置为“nginx：1.9.1”。

```
kubectl set image deployment/nginx busybox=busybox nginx=nginx:1.9.1
```

* 所有deployment和rc的nginx容器镜像更新为“nginx：1.9.1”

```
kubectl set image deployments,rc nginx=nginx:1.9.1 --all
```

* 将daemonset abc的所有容器镜像更新为“nginx：1.9.1”

```
kubectl set image daemonset abc *=nginx:1.9.1
```

* 从本地文件中更新nginx容器镜像

```
kubectl set image -f path/to/file.yaml nginx=nginx:1.9.1 --local -o yaml
```



