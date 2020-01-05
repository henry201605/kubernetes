## 01 Storage

### 1.1 Volume

> `Volume`：https://kubernetes.io/docs/concepts/storage/volumes/
>
> ```
> On-disk files in a Container are ephemeral, which presents some problems for non-trivial applications when running in Containers. First, when a Container crashes, kubelet will restart it, but the files will be lost - the Container starts with a clean state. Second, when running Containers together in a Pod it is often necessary to share files between those Containers. The Kubernetes Volume abstraction solves both of these problems.
> ```

### 1.2 Host类型volume实战

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



### 1.3 PersistentVolume

> `官网`：https://kubernetes.io/docs/concepts/storage/persistent-volumes/

PV是K8s中的资源，volume的plugin实现，生命周期独立于Pod，封装了底层存储卷实现的细节。

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 5Gi    # 存储空间大小
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce     # 只允许一个Pod进行独占式读写操作
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: slow
  mountOptions:
    - hard
    - nfsvers=4.1
  nfs:
    path: /tmp            # 远端服务器的目录
    server: 172.17.0.2    # 远端的服务器
```



`注意`：PV的维护通常是由运维人员、集群管理员进行维护的。

### 1.4 PersistentVolumeClaim

> `官网`：<https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims>
>



有了PV，那Pod如何使用呢？为了方便使用，我们可以设计出一个PVC来绑定PV，然后把PVC交给Pod来使用。

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myclaim
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 8Gi
  storageClassName: slow
  selector:
    matchLabels:
      release: "stable"
    matchExpressions:
      - {key: environment, operator: In, values: [dev]}
```

​    通俗点说，PVC会匹配满足要求的PV[**是根据size和访问模式进行匹配的**]，进行一一绑定，然后它们的状态都会变成Bound。也就是PVC负责请求PV的大小和访问方式，然后Pod中就可以直接使用PVC。

`注意`：PVC通常由开发小伙伴维护，开发小伙伴无需关注与存储细节。

### 1.5 Pod中如何使用PVC

> `官网`：<https://kubernetes.io/docs/concepts/storage/persistent-volumes/#claims-as-volumes>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
    - name: myfrontend
      image: nginx
      volumeMounts:
      - mountPath: "/var/www/html"
        name: mypd
  volumes:
    - name: mypd
      persistentVolumeClaim:
        claimName: myclaim
```

### 1.6 Pod中使用PVC实战

说了这么多，也大概讲述了一下各自的含义，下面开始做个实战以便更好的理解PV及PVC.

`背景`：使用nginx持久化存储演示

> ```
>(1)共享存储使用nfs，比如选择在m节点
> (2)创建pv和pvc
> (3)nginx pod中使用pvc
> ```

#### 1.6.1 master节点搭建nfs

PV的类型有多种，参考https://kubernetes.io/docs/concepts/storage/persistent-volumes/#types-of-persistent-volumes，本案例采用NFC。

在master节点上搭建一个NFS服务器，目录为/nfs/data

> ```shell
>nfs(network file system)网络文件系统，是FreeBSD支持的文件系统中的一种，允许网络中的计算机之间通过TCP/IP网络共享资源
> 
> 01 选择一台服务器安装nfc,如果资源有限也可以直接选择master节点
> # 安装nfs
> 	yum install -y nfs-utils
>  	# 创建nfs目录
> 	mkdir -p /nfs/data/
> 	mkdir -p /nfs/data/mysql
> 	# 授予权限
> 	chmod -R 777 /nfs/data
> 	# 编辑export文件
> 	vi /etc/exports
> 	  /nfs/data *(rw,no_root_squash,sync)  `下面有说明`
> 	# 使得配置生效
> 	exportfs -r
> 	# 查看生效
> 	exportfs
> 	# 启动rpcbind、nfs服务
> 	systemctl restart rpcbind && systemctl enable rpcbind
> 	systemctl restart nfs && systemctl enable nfs
> 	# 查看rpc服务的注册情况
> 	rpcinfo -p localhost
> 	# showmount测试(Hostip为自己的主机ip)
> 	showmount -e 39.105.208.211  
> 	
> 02 所有node上安装客户端
> 	yum -y install nfs-utils
> 	systemctl start nfs && systemctl enable nfs
> ```

```shell
#/etc/exports文件中，/nfs/data *(rw,no_root_squash,sync)
说明：
/data 要共享的目录
*  允许访问的ip段，这里是允许所有网络访问
括号里的含义如下：
rw ：读写；
ro ：只读；
sync ：同步模式，内存中数据时时写入磁盘；
async ：不同步，把内存中数据定期写入磁盘中；
no_root_squash ：加上这个选项后，root用户就会对共享的目录拥有至高的权限控制，就像是对本机的目录操作一样。
root_squash ：和上面的选项对应，root用户对共享目录的权限不高，只有普通用户的权限，即限制了root；
all_squash ：不管使用NFS的用户是谁，他的身份都会被限定成为一个指定的普通用户身份；
anonuid/anongid ：要和root_squash 以及 all_squash一同使用，用于指定使用NFS的用户限定后的uid和gid，前提是本机的/etc/passwd中存在这个uid和gid。
```



#### 1.6.2 创建PV&PVC&Nginx

(1)在nfs服务器创建所需要的目录

```shell
  mkdir -p /nfs/data/nginx
```

(2)定义PV，PVC和Nginx的yaml文件

```yaml
# 定义PV
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nginx-pv
spec:
  accessModes:
    - ReadWriteMany
  capacity:
    storage: 2Gi    
  nfs:
    path: /nfs/data/nginx     
    server: 39.105.208.211  
    
---
# 定义PVC，用于消费PV
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nginx-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 2Gi
  
---
# 定义Pod，指定需要使用的PVC
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: nginx
spec:
  selector:
    matchLabels: 
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: mysql
        ports:
        - containerPort: 80
        volumeMounts:
        - name: nginx-persistent-storage
          mountPath: /usr/share/nginx/html
      volumes:
      - name: nginx-persistent-storage
        persistentVolumeClaim:
          claimName: nginx-pvc
```



(3)根据yaml文件创建资源并查看资源

```shell
`创建资源`
[root@henry001 storage]# kubectl apply -f nginx-pv-demo.yaml
persistentvolume/nginx-pv created
persistentvolumeclaim/nginx-pvc created
deployment.apps/nginx created

`查看pv\pvc`
[root@henry001 storage]# kubectl get pv,pvc
NAME                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM               STORAGECLASS   REASON   AGE
persistentvolume/nginx-pv   2Gi        RWX            Retain           Bound    default/nginx-pvc                           27s

`查看pod`
[root@henry001 storage]# kubectl get pods -o wide
NAME                    READY   STATUS    RESTARTS   AGE   IP                NODE       NOMINATED NODE   READINESS GATES
nginx-c4998fd44-8bbjk   1/1     Running   0          51s   192.168.254.221   henry003   <none>           <none>
```

(4)测试持久化存储

* 进入nfs

```shell
`进入到nfs挂载的目录中`
[root@henry001 storage]# cd /nfs/data/nginx
[root@henry001 nginx]# vim henry.html
[root@henry001 nginx]# curl 192.168.254.221/henry.html

#henry.html文件内容
<html>
      <head><title>hello henry</title></head>
      <body>
        Hello World
      </body>
</html>

```

* 进入到容器中/usr/share/nginx/html目录查看

```shell
`进入到容器中`
[root@henry001 nginx]# kubectl exec -it nginx-c4998fd44-8bbjk bash
root@nginx-c4998fd44-8bbjk:/# cd /usr/share/nginx/html
root@nginx-c4998fd44-8bbjk:/usr/share/nginx/html# cat henry.html 
<html>
      <head><title>hello henry</title></head>
      <body>
        Hello World
      </body>
</html>
```

* 删除掉pod后，再查看一下

```shell

[root@henry001 nginx]# kubectl delete pod nginx-c4998fd44-8bbjk
pod "nginx-c4998fd44-8bbjk" deleted
[root@henry001 nginx]# kubectl get pods -o wide
NAME                    READY   STATUS    RESTARTS   AGE   IP               NODE       NOMINATED NODE   READINESS GATES
nginx-c4998fd44-gwwxb   1/1     Running   0          18s   192.168.217.23   henry002   <none>           <none>
`访问henry.html`
[root@henry001 nginx]# curl 192.168.217.23/henry.html

<html>
      <head><title>hello henry</title></head>
      <body>
        Hello World
      </body>
</html>

```

发现pod被删除后，重新创建的pod可以通过挂载的目录，同步到新创建的pod目录中。

### 1.7 StorageClass

一般情况下，我们不会去手动管理PV，我们会采用自动创建的方式来实现，先来了解一下StorageClass。

`官网`：<https://kubernetes.io/docs/concepts/storage/storage-classes/>

`nfs github`：`github`：<https://github.com/kubernetes-incubator/external-storage/tree/master/nfs>

```
A StorageClass provides a way for administrators to describe the “classes” of storage they offer. Different classes might map to quality-of-service levels, or to backup policies, or to arbitrary policies determined by the cluster administrators. Kubernetes itself is unopinionated about what classes represent. This concept is sometimes called “profiles” in other storage systems.
```

```
Each StorageClass contains the fields provisioner, parameters, and reclaimPolicy, which are used when a PersistentVolume belonging to the class needs to be dynamically provisioned.

The name of a StorageClass object is significant, and is how users can request a particular class. Administrators set the name and other parameters of a class when first creating StorageClass objects, and the objects cannot be updated once they are created.
```

StorageClass声明存储插件，用于自动创建PV。通俗点说，StorageClass就是创建PV的模板，其中包含有两个重要部分：PV属性和创建此PV所需要的插件。

这样PVC就可以按“Class”来匹配PV。

可以为PV指定storageClassName属性，标识PV归属于哪一个Class。

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
reclaimPolicy: Retain
allowVolumeExpansion: true
mountOptions:
  - debug
volumeBindingMode: Immediate
```

```shell
01 对于PV或者StorageClass只能对应一种后端存储
02 对于手动的情况，一般我们会创建很多的PV，等有PVC需要使用的时候就可以直接使用了
03 对于自动的情况，那么就由StorageClass来自动管理创建
04 如果Pod想要使用共享存储，一般会在创建PVC，PVC中描述了想要什么类型的后端存储、空间等，K8s从而会匹配对应的PV，如果没有匹配成功，Pod就会处于Pending状态。Pod中使用只需要像使用volumes一样，指定名字就可以使用了
05 一个Pod可以使用多个PVC，一个PVC也可以给多个Pod使用
06 一个PVC只能绑定一个PV，一个PV只能对应一种后端存储
```

有了StorageClass之后的PVC可以变成如下的形式：

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
name: test-claim1
spec:
accessModes:
    - ReadWriteMany
resources:
 requests:
    storage: 1Mi
  storageClassName: nfs
```



StorageClass之所以能够动态供给PV，是因为Provisioner，也就是Dynamic Provisioning，通过Provisioner插件可以根据不同服务器的类型进行创建不同类型的PV。

但是NFS这种类型，K8s中默认是没有Provisioner插件的，需要自己创建。

![image-20200105205809042](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200105205809042.png)

下面我们就来看一个storageClass的一个实战。

### 1.8 StorageClass实战

上面提到的nfs的Provisioner可以到GitHub上查到，`github`：<https://github.com/kubernetes-incubator/external-storage/tree/master/nfs>

下面开始使用StorageClass创建pv:

(1)使用前面安装好的NFS服务器，创建目录；

```shell
mkdir -p /nfs/data/henry
```

(2)在k8s的master节点上，根据rbac.yaml文件创建资源

执行rabc.yaml文件的原因是创建StorageClass的资源，需要有访问api-server的权限。

```shell
#到 https://github.com/kubernetes-incubator/external-storage/blob/master/nfs/deploy/kubernetes/rbac.yaml下载下载rbac.yaml文件

#执行yaml文件创建资源
[root@henry001 storage]# kubectl apply -f rbac.yaml
clusterrole.rbac.authorization.k8s.io/nfs-provisioner-runner created
clusterrolebinding.rbac.authorization.k8s.io/run-nfs-provisioner created
role.rbac.authorization.k8s.io/leader-locking-nfs-provisioner created
rolebinding.rbac.authorization.k8s.io/leader-locking-nfs-provisioner created
```

(3)根据deployment.yaml文件创建资源

创建访问nfs服务器的插件nfs-provisioner，注意此时并不是创建应用的pod

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-provisioner
---
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: nfs-provisioner
spec:
  replicas: 1
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: nfs-provisioner
    spec:
      serviceAccount: nfs-provisioner
      containers:
        - name: nfs-provisioner
          image: registry.cn-hangzhou.aliyuncs.com/open-ali/nfs-client-provisioner
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: example.com/nfs
            - name: NFS_SERVER
              value: 39.105.208.211  
            - name: NFS_PATH
              value: /nfs/data/henry
      volumes:
        - name: nfs-client-root
          nfs:
            server: 39.105.208.211 
            path: /nfs/data/henry
```



```shell
[root@henry001 storage]# kubectl apply -f deployment.yaml
serviceaccount/nfs-provisioner created
deployment.extensions/nfs-provisioner created
[root@henry001 storage]# kubectl get deployment
NAME              READY   UP-TO-DATE   AVAILABLE   AGE
nfs-provisioner   1/1     1            1           16s
```

(4)根据class.yaml创建资源

```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: example-nfs
provisioner: example.com/nfs
```

```shell
#创建storageClass
[root@henry001 storage]# kubectl apply -f class.yaml
storageclass.storage.k8s.io/example-nfs create
```

(5)根据my-pvc.yaml创建资源

```shell
`创建pvc资源`
[root@henry001 storage]# kubectl apply -f my-pvc.yaml
persistentvolumeclaim/my-pvc created


[root@henry001 storage]# kubectl get pvc
NAME        STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
my-pvc      Bound    pvc-3c920a3a-2fc2-11ea-8e59-00163e2c61c9   1Mi        RWX            example-nfs    14s
nginx-pvc   Bound    nginx-pv                                   2Gi        RWX                           8h
[root@henry001 storage]# kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM               STORAGECLASS   REASON   AGE
nginx-pv                                   2Gi        RWX            Retain           Bound    default/nginx-pvc                           8h
pvc-3c920a3a-2fc2-11ea-8e59-00163e2c61c9   1Mi        RWX            Delete           Bound    default/my-pvc      example-nfs             27s

```

(6)根据nginx-pod创建资源

```shell
kubectl apply -f nginx-pod.yaml
kubectl exec -it nginx bash
cd /usr/jack
# 进行同步数据测试
```

### 1.9 PV的状态和回收策略

- PV的状态

Available：表示当前的pv没有被绑定

Bound：表示已经被pvc挂载

Released：pvc没有在使用pv, 需要管理员手工释放pv

Failed：资源回收失败

- PV回收策略

Retain：表示删除PVC的时候，PV不会一起删除，而是变成Released状态等待管理员手动清理

Recycle：在Kubernetes新版本就不用了，采用动态PV供给来替代

Delete：表示删除PVC的时候，PV也会一起删除，同时也删除PV所指向的实际存储空间

`注意`：目前只有NFS和HostPath支持Recycle策略。AWS EBS、GCE PD、Azure Disk和Cinder支持Delete策略

