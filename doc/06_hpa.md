##  Horizontal Pod Autoscaler

> `官网`：https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/
>
> ```
> The Horizontal Pod Autoscaler automatically scales the number of pods in a replication controller, deployment or replica set based on observed CPU utilization (or, with custom metrics support, on some other application-provided metrics). Note that Horizontal Pod Autoscaling does not apply to objects that can’t be scaled, for example, DaemonSets.
> ```
>
> 使用Horizontal Pod Autoscaling，Kubernetes会自动地根据观察到的CPU利用率(或者通过一些其他应用程序提供的自定义的指标)自动地缩放在replication controller、deployment或replica set上pod的数量。



**案列：**

> 1、创建yaml文件

```
vim nginx-hpa.yaml
```

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
        image: nginx
        ports:
        - containerPort: 80
```



> 2、执行yaml文件

kubectl apply -f nginx-hpa.yaml

> 3、创建hpa

```shell
# 使nginx pod的数量介于2和10之间，CPU使用率维持在50％
kubectl autoscale deployment nginx-deployment --min=2 --max=10 --cpu-percent=50
```

> 4、查看所有创建的资源

```shell
`查看pod`
[root@henry001 controller]# kubectl get pods
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-56db997f77-52fqn   1/1     Running   0          95s
nginx-deployment-56db997f77-v6q5f   1/1     Running   0          95s
nginx-deployment-56db997f77-vxq66   1/1     Running   0          95s

`查看deployment生成了三个副本`
[root@henry001 controller]# kubectl get deploy
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3/3     3            3           2m46s

`查看hpa,最少个数是2，最大是10`
[root@henry001 controller]# kubectl get hpa
NAME               REFERENCE                     TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
nginx-deployment   Deployment/nginx-deployment   <unknown>/50%   2         10        3          87s
```

> 5、修改deployment的副本数

```
kubectl edit deployment nginx-deployment
```

* 修改副本数为1，可以看到最少还是有2个pod运行

```shell
`先终止掉2个再重新新建一个pod来达到设定的个数`
[root@henry001 controller]# kubectl get pods
NAME                                READY   STATUS              RESTARTS   AGE
nginx-deployment-56db997f77-52fqn   1/1     Running             0          8m32s
nginx-deployment-56db997f77-ll75d   0/1     ContainerCreating   0          1s
nginx-deployment-56db997f77-v6q5f   0/1     Terminating         0          8m32s
nginx-deployment-56db997f77-vxq66   0/1     Terminating         0          8m32s
[root@henry001 controller]# kubectl get pods
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-56db997f77-52fqn   1/1     Running   0          8m42s
nginx-deployment-56db997f77-ll75d   1/1     Running   0          11s
```

* 修改副本数为15，可以看到最多还是有10个pod运行

```shell
[root@henry001 controller]# kubectl get pods
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-56db997f77-44bgp   1/1     Running   0          68s
nginx-deployment-56db997f77-52fqn   1/1     Running   0          13m
nginx-deployment-56db997f77-6lm7k   1/1     Running   0          68s
nginx-deployment-56db997f77-9fxv2   1/1     Running   0          68s
nginx-deployment-56db997f77-9xs9z   1/1     Running   0          68s
nginx-deployment-56db997f77-bkcxv   1/1     Running   0          68s
nginx-deployment-56db997f77-f4v8k   1/1     Running   0          68s
nginx-deployment-56db997f77-jwn9x   1/1     Running   0          68s
nginx-deployment-56db997f77-ll75d   1/1     Running   0          5m3s
nginx-deployment-56db997f77-xwml9   1/1     Running   0          68s

```

**再次理解什么是hpa**

Horizontal Pod Autoscaling可以根据CPU使用率或应用自定义metrics自动扩展Pod数量（支持replication controller、deployment和replica set）。

- 控制管理器每隔30s（可以通过–horizontal-pod-autoscaler-sync-period修改）查询metrics的资源使用情况
- 支持三种metrics类型
  - 预定义metrics（比如Pod的CPU）以利用率的方式计算
  - 自定义的Pod metrics，以原始值（raw value）的方式计算
  - 自定义的object metrics
- 支持两种metrics查询方式：Heapster和自定义的REST API
- 支持多metrics

参阅：https://www.kubernetes.org.cn/horizontal-pod-autoscaling