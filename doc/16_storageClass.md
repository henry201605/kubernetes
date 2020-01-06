### 1 StorageClass

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

### 2 StorageClass实战

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



```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Mi
  # 这个名字要和上面创建的storageclass名称一致
  storageClassName: example-nfs
```



```shell
`创建pvc资源`
[root@henry001 storage]# kubectl apply -f my-pvc.yaml
persistentvolumeclaim/my-pvc created

`查看pvc已处于Bound状态,使用的storageClass是example-nfs`
[root@henry001 storage]# kubectl get pvc
NAME        STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
my-pvc      Bound    pvc-3c920a3a-2fc2-11ea-8e59-00163e2c61c9   1Mi        RWX            example-nfs    14s

`查看pv已处于Bound状态,使用的storageClass是example-nfs`
[root@henry001 storage]# kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM               STORAGECLASS   REASON   AGE
pvc-3c920a3a-2fc2-11ea-8e59-00163e2c61c9   1Mi        RWX            Delete           Bound    default/my-pvc      example-nfs             27s

```

(6)根据nginx-pod创建资源

```shell
vim nginx-pod.yaml
```

```yaml
kind: Pod
apiVersion: v1
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
      - name: my-pvc
        mountPath: "/usr/henry"
  restartPolicy: "Never"
  volumes:
    - name: my-pvc
      persistentVolumeClaim:
        claimName: my-pvc
```



```shell
`创建pod`
[root@henry001 storage]# kubectl apply -f nginx-pod.yaml
pod/nginx created
[root@henry001 storage]# kubectl get pods -o wide
NAME                               READY   STATUS    RESTARTS   AGE   IP                NODE       NOMINATED NODE   READINESS GATES
nfs-provisioner-7484889c75-8grtp   1/1     Running   0          20h   192.168.254.224   henry003   <none>           <none>
nginx                              1/1     Running   0          50s   192.168.254.225   henry003   <none>           <none>
nginx-c4998fd44-gwwxb              1/1     Running   0          28h   192.168.217.23    henry002   <none>           <none>

-----------------`下面做一些同步的测试`-------------------------
`01 进入容器,此时/usr/henry下无文件`
[root@henry001 storage]# kubectl exec -it nginx bash
root@nginx:/# cd /usr/henry
root@nginx:/usr/henry# ls

`02 进入nfs共享路径下/nfs/data/henry，发现有一个文件夹，文件夹下内容为空`
[root@henry001 ~]# cd /nfs/data/henry
[root@henry001 henry]# ls
default-my-pvc-pvc-3c920a3a-2fc2-11ea-8e59-00163e2c61c9
[root@henry001 henry]# cd default-my-pvc-pvc-3c920a3a-2fc2-11ea-8e59-00163e2c61c9/
[root@henry001 default-my-pvc-pvc-3c920a3a-2fc2-11ea-8e59-00163e2c61c9]# ls

`03在共享文件夹下新建一个文件henry.html`
[root@henry001 default-my-pvc-pvc-3c920a3a-2fc2-11ea-8e59-00163e2c61c9]# echo 'hello henry' >henry.html
[root@henry001 default-my-pvc-pvc-3c920a3a-2fc2-11ea-8e59-00163e2c61c9]# cat henry.html 
hello henry

`04 进入容器的/usr/henry文件下查看`
root@nginx:/usr/henry# ls /usr/henry
henry.html
root@nginx:/usr/henry# cat /usr/henry/henry.html 
hello henry
```

`总结：` 通过上面的例子我们可以看到使用storageClass来动态创建pv，这样就大大节省了先创建pv闲置造成的资源浪费。

### 3 PV的状态和回收策略

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

