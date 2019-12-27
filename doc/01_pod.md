> `官网`：<https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/>

### 1 Pod 

> #### What is a Pod

```
A Pod (as in a pod of whales or pea pod) is a group of one or more containers (such as Docker containers), with shared storage/network, and a specification for how to run the containers. A Pod’s contents are always co-located and co-scheduled, and run in a shared context. A Pod models an application-specific “logical host” - it contains one or more application containers which are relatively tightly coupled — in a pre-container world, being executed on the same physical or virtual machine would mean being executed on the same logical host.
```

**shared storage/network ------>pod是共享网络和存储**

- Networking

  `官网`：https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/#networking

```
Each Pod is assigned a unique IP address. Every container in a Pod shares the network namespace, including the IP address and network ports. 
```

- Storage

  `官网`：https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/#storage

```
A Pod can specify a set of shared storage Volumes. All containers in the Pod can access the shared volumes, allowing those containers to share data. 
```

> **静态Pod**

静态Pod是由kubelet进行管理的，并且存在于特定的Node上。

不能通过API Server进行管理，无法与ReplicationController,Ddeployment或者DaemonSet进行关联，也无法进行健康检查。

此处不做详细介绍，会在后续的DaemonSet一章处做介绍；

https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/#static-pods

### 2 Case

> (1)创建一个pod的yaml文件，名称为nginx_pod.yaml

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

> (2)根据该nginx_pod.yaml文件创建pod

```
kubectl apply -f nginx_pod.yaml
```

> (3)查看pod

* 01 kubectl get pods

```
NAME        READY   STATUS    RESTARTS   AGE
nginx-pod   1/1     Running   0          22s
```

* 02 kubectl get pods -o wide

```
NAME        READY   STATUS    RESTARTS   AGE   IP              NODE       NOMINATED NODE   READINESS GATES
nginx-pod   1/1     Running   0          48s   192.168.217.1   henry002   <none>           <none>
```

* 03 kubectl describe pod nginx-pod

  可以查看pod描述的详细信息

```
Name:               nginx-pod
Namespace:          default
Priority:           0
PriorityClassName:  <none>
Node:               henry002/192.168.0.5
Start Time:         Fri, 27 Dec 2019 15:27:11 +0800
Labels:             app=nginx
Annotations:        cni.projectcalico.org/podIP: 192.168.217.1/32
                    kubectl.kubernetes.io/last-applied-configuration:
                      {"apiVersion":"v1","kind":"Pod","metadata":{"annotations":{},"labels":{"app":"nginx"},"name":"nginx-pod","namespace":"default"},"spec":{"c...
Status:             Running
IP:                 192.168.217.1
Containers:
  nginx-container:
    Container ID:   docker://a2d60afe575b7b2597adc5d6af37d4f2fc7895c0f7a91a1548de4e01edfe4715
    Image:          nginx
    Image ID:       docker-pullable://nginx@sha256:50cf965a6e08ec5784009d0fccb380fc479826b6e0e65684d9879170a9df8566
    Port:           80/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Fri, 27 Dec 2019 15:27:17 +0800
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-8qgpx (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  default-token-8qgpx:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-8qgpx
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type    Reason     Age    From               Message
  ----    ------     ----   ----               -------
  Normal  Scheduled  2m19s  default-scheduler  Successfully assigned default/nginx-pod to henry002
  Normal  Pulling    2m18s  kubelet, henry002  Pulling image "nginx"
  Normal  Pulled     2m13s  kubelet, henry002  Successfully pulled image "nginx"
  Normal  Created    2m13s  kubelet, henry002  Created container nginx-container
  Normal  Started    2m13s  kubelet, henry002  Started container nginx-container
```

> (4)可以发现该pod运行在henry002节点上

* 于是来到henry002节点，docker ps一下

```
[root@henry002 ~]# docker ps |grep nginx
a2d60afe575b        nginx                  "nginx -g 'daemon of…"   10 minutes ago      Up 10 minutes                           k8s_nginx-container_nginx-pod_default_4bd661c2-287a-11ea-b01b-00163e1651b0_0
bb0e45325596        k8s.gcr.io/pause:3.1   "/pause"                 10 minutes ago      Up 10 minutes                           k8s_POD_nginx-pod_default_4bd661c2-287a-11ea-b01b-00163e1651b0_0

```

* 可以进入该容器：

```
[root@henry002 ~]# docker exec -it a2d60afe575b bash
root@nginx-pod:/# 
```

如果是从master节点进入容器，可以使用如下命令：

```
[root@henry001 ~]# kubectl exec -it nginx-pod bash
root@nginx-pod:/# 
```



> (5)访问nginx容器

```
curl 192.168.217.1    OK，并且在任何一个集群中的Node上访问都成功
```

> (6)删除Pod

```
[root@henry001 ~]# kubectl delete -f ngnix_pod.yaml 
pod "nginx-pod" deleted
[root@henry001 ~]# kubectl get pod
No resources found.
```

### 3 Lifecycle

> `官网`：https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/

Pod的生命周期有以下五种：

- 挂起（Pending）：Pod 已被 Kubernetes 系统接受，但有一个或者多个容器镜像尚未创建。等待时间包括调度 Pod 的时间和通过网络下载镜像的时间，这可能需要花点时间。
- 运行中（Running）：该 Pod 已经绑定到了一个节点上，Pod 中所有的容器都已被创建。至少有一个容器正在运行，或者正处于启动或重启状态。
- 成功（Succeeded）：Pod 中的所有容器都被成功终止，并且不会再重启。
- 失败（Failed）：Pod 中的所有容器都已终止了，并且至少有一个容器是因为失败终止。也就是说，容器以非0状态退出或者被系统终止。
- 未知（Unknown）：因为某些原因无法取得 Pod 的状态，通常是因为与 Pod 所在主机通信失败。

### 4 restartPolicy

> `官网`：<https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#restart-policy>
>
> ```
> A PodSpec has a restartPolicy field with possible values Always, OnFailure, and Never. The default value is Always. restartPolicy applies to all Containers in the Pod. restartPolicy only refers to restarts of the Containers by the kubelet on the same node. Exited Containers that are restarted by the kubelet are restarted with an exponential back-off delay (10s, 20s, 40s …) capped at five minutes, and is reset after ten minutes of successful execution. As discussed in the Pods document, once bound to a node, a Pod will never be rebound to another node.
> ```

- Always：容器失效时，即重启
- OnFailure：容器终止运行且退出码不为0时重启
- Never:永远不重启



### 5 Probes

> `官网`：<https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes>

对于Pod的健康状态检测，kubernetes提供了两类探针(Probe)来执行对Pod的健康状态检测：

- LivenessProbe探针：
  用于判断容器是否存活，即Pod是否为running状态，如果LivenessProbe探针探测到容器不健康，则kubelet将kill掉容器，并根据容器的重启策略是否重启，如果一个容器不包含LivenessProbe探针，则Kubelet认为容器的LivenessProbe探针的返回值永远成功。

- ReadinessProbe探针：
  用于判断容器是否启动完成，即容器的Ready是否为True，可以接收请求，如果ReadinessProbe探测失败，则容器的Ready将为False，控制器将此Pod的Endpoint从对应的service的Endpoint列表中移除，从此不再将任何请求调度此Pod上，直到下次探测成功。

每次探测都将获得以下三种结果之一：

- Success（成功）：容器通过了检查。
- Failure （失败）：容器未通过检查。
- Unknown（未知）：诊断失败，因此不会采取任何行动。

具体实例，可以参考这篇文章：

https://www.cnblogs.com/kenken2018/p/10337471.html

https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/