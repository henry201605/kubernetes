## 01 Storage

### 1 Volume

> `Volume`：https://kubernetes.io/docs/concepts/storage/volumes/
>
> ```
> On-disk files in a Container are ephemeral, which presents some problems for non-trivial applications when running in Containers. First, when a Container crashes, kubelet will restart it, but the files will be lost - the Container starts with a clean state. Second, when running Containers together in a Pod it is often necessary to share files between those Containers. The Kubernetes Volume abstraction solves both of these problems.
> ```

### 2 Host类型volume实战

`背景`：定义一个Pod，其中包含两个Container，都使用Pod的Volume

(1)创建yaml文件

```
vim  volume-pod.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-pod
spec:
  containers:
  - name: nginx-container
    image: nginx
    ports:
    - containerPort: 80
    volumeMounts:
    - name: volume-pod
      mountPath: /nginx-volume
  - name: busybox-container
    image: busybox
    volumeMounts:
    - name: volume-pod
      mountPath: /busybox-volume
  volumes:
  - name: volume-pod
    hostPath:
      path: /tmp/volume-pod
```

(1)创建资源并查看pod的运行情况

```shell
[root@henry001 storage]# kubectl apply -f volume-pod.yaml
pod/volume-pod created

[root@henry001 storage]# kubectl get pods -o wide
NAME         READY   STATUS    RESTARTS   AGE   IP                NODE       NOMINATED NODE   READINESS GATES
volume-pod   2/2     Running   0          38s   192.168.254.217   henry003   <none>           <none>
```



(3)来到运行的henry003节点

```shell
`查看containner`
[root@henry003 ~]# docker ps | grep volume
3948f073e103        busybox                "sh -c 'echo The app…"   About a minute ago   Up About a minute                       k8s_busybox-container_volume-pod_default_46419dfd-2f09-11ea-8e59-00163e2c61c9_0
d3a9697c2067        nginx                  "nginx -g 'daemon of…"   About a minute ago   Up About a minute                       k8s_nginx-container_volume-pod_default_46419dfd-2f09-11ea-8e59-00163e2c61c9_0
be985f406104        k8s.gcr.io/pause:3.1   "/pause"                 About a minute ago   Up About a minute                       k8s_POD_volume-pod_default_46419dfd-2f09-11ea-8e59-00163e2c61c9_0

`查看挂载的目录`
ls /tmp/volume-pod

`进入busybox容器中，查看busybox-volume`
docker exec -it 3948f073e103 sh   
ls /busybox-volume

`进入nginx容器中，查看nginx-volume`
docker exec -it d3a9697c2067 sh   
ls /nginx-volume
```

此时宿主机上的/tmp/volume-pod中无文件，同样各容器下对应的目录下也无文件；下面做以下操作：

1>在宿主机的/tmp/volume-pod目录下新建一个volum.txt文件，然后再到容器下各自对应的文件中进行查看。

2>删除pod之后再重新创建，发现在各自挂载对应的目录下仍存在volum.txt。



(4)查看pod中的容器里面的hosts文件发现是一样的，并且都是由pod管理的

```shell
`pod地址`
[root@henry001 storage]# kubectl get pod -o wide
NAME         READY   STATUS    RESTARTS   AGE     IP                NODE       NOMINATED NODE   READINESS GATES
volume-pod   2/2     Running   0          8m24s   192.168.254.220   henry003   <none>           <none>

[root@henry003 volume-pod]# docker exec -it 3948f073e103 cat /etc/hosts
# Kubernetes-managed hosts file.
127.0.0.1	localhost
::1	localhost ip6-localhost ip6-loopback
fe00::0	ip6-localnet
fe00::0	ip6-mcastprefix
fe00::1	ip6-allnodes
fe00::2	ip6-allrouters
192.168.254.220	volume-pod   #pod的IP

[root@henry003 volume-pod]# docker exec -it d3a9697c2067 cat /etc/hosts
# Kubernetes-managed hosts file.
127.0.0.1	localhost
::1	localhost ip6-localhost ip6-loopback
fe00::0	ip6-localnet
fe00::0	ip6-mcastprefix
fe00::1	ip6-allnodes
fe00::2	ip6-allrouters
192.168.254.220	volume-pod    #pod的IP
```

(5)一般container中的存储或者网络的内容，不要在container层面修改，而是在pod中修改，直接写在yaml文件里。

比如下面修改host文件的网络：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-pod
spec:
  hostNetwork: true
  hostPID: true
  hostAliases: 
  - ip: "192.168.8.61"
    hostnames: 
    - "test.henry.com" 
  containers:
  - name: nginx-container
    image: nginx
    ports:
    - containerPort: 80
    volumeMounts:
    - name: volume-pod
      mountPath: /nginx-volume
  - name: busybox-container
    image: busybox
    command: ['sh', '-c', 'echo The app is running! && sleep 3600']
    volumeMounts:
    - name: volume-pod
      mountPath: /busybox-volume
  volumes:
  - name: volume-pod
    hostPath:
      path: /tmp/volume-pod
```

