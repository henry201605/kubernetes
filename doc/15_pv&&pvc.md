### 1 PersistentVolume

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

### 2 PersistentVolumeClaim

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

### 3 Pod中如何使用PVC

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

### 4 Pod中使用PVC实战

说了这么多，也大概讲述了一下各自的含义，下面开始做个实战以便更好的理解PV及PVC.

`背景`：使用nginx持久化存储演示

> ```
>(1)共享存储使用nfs，比如选择在m节点
> (2)创建pv和pvc
> (3)nginx pod中使用pvc
> ```

#### 4.1 master节点搭建nfs

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



#### 4.2 创建PV&PVC&Nginx

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
