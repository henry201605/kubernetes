### ConfigMap

> `官网`：<https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/>
>
> ```
> ConfigMaps allow you to decouple configuration artifacts from image content to keep containerized applications portable. 
> ```
>
> 说白了就是用来保存配置数据的键值对，也可以保存单个属性，也可以保存配置文件。
>
> 所有的配置内容都存储在etcd中，创建的数据可以供Pod使用。

configMap可以通过以下四种方式进行创建，下面来一一进行讲述。



#### 1 命令行创建

```shell
# 创建一个名称为my-config的ConfigMap，key值时db.port，value值是'3306'
kubectl create configmap my-config --from-literal=db.port='3306'

#查看生成的configMap
[root@henry001 ~]# kubectl get configMap
NAME        DATA   AGE
my-config   1      79s
```

查看yaml的详情信息：

```shell
[root@henry001 ~]# kubectl get configmap my-config -o yaml
apiVersion: v1
data:
  db.port: "3306"
kind: ConfigMap
metadata:
  creationTimestamp: "2020-01-01T01:55:18Z"
  name: my-config
  namespace: default
  resourceVersion: "63283"
  selfLink: /api/v1/namespaces/default/configmaps/my-config
  uid: c3358683-2c39-11ea-8e59-00163e2c61c9

```



#### 2 从配置文件中创建

创建一个文件，名称为myconfig.properties

```
name=henry
age=18
```

```shell
`创建configmap`
[root@henry001 ~]# kubectl create configmap myconfig --from-file=./myconfig.properties 
configmap/myconfig created

`查看生成的configmap`
[root@henry001 ~]# kubectl get configmap
NAME        DATA   AGE
my-config   1      39m
myconfig    1      19s

`查看configmap的yaml文件`
[root@henry001 ~]# kubectl get configmap myconfig -o yaml
apiVersion: v1
data:
  myconfig.properties: |
    name=henry       #myconfig.properties中的内容
    age=18
kind: ConfigMap
metadata:
  creationTimestamp: "2020-01-01T02:34:45Z"
  name: myconfig
  namespace: default
  resourceVersion: "66726"
  selfLink: /api/v1/namespaces/default/configmaps/myconfig
  uid: 45d3f742-2c3f-11ea-8e59-00163e2c61c9

```

#### 3 从目录中创建



```shell
#创建目录
mkdir config
#进入目录
cd config
#创建两个文件夹
mkdir a
mkdir b
cd ..
```

```shell
`创建configmap`
[root@henry001 ~]# kubectl create configmap config --from-file=config/
configmap/config created

`查看configmap`
[root@henry001 ~]# kubectl get configmap
NAME        DATA   AGE
config      0      13s   #从目录中创建
my-config   1      56m   #命令行生成
myconfig    1      17m   #配置文件生成

```

查看

```shell
[root@henry001 ~]# kubectl get configmap config -o yaml
apiVersion: v1
data:
  a: |
    aaaaaaa
    cccccc            #a文件中的内容
    ss1234
  b: |
    xxxxxxxxxxxxx
    yyyyyyyyyyyyy      #b文件中的内容
    zzzzzzzzzzzz123
kind: ConfigMap
metadata:
  creationTimestamp: "2020-01-01T06:58:55Z"
  name: config
  namespace: default
  resourceVersion: "89928"
  selfLink: /api/v1/namespaces/default/configmaps/config
  uid: 2d473189-2c64-11ea-8e59-00163e2c61c9

```

**可以看到指定目录创建时configmap内容中的各个文件会创建一个key/value对，key是文件名，value是文件内容。**



#### 4 通过yaml文件创建

01 创建configmaps.yaml文件

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: special-config
  namespace: default
data:
  special.how: very
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: env-config
  namespace: default
data:
  log_level: INFO
```

```shell
kubectl apply -f configmaps.yaml
kubectl get configmap
```

02 执行yaml文件，创建configmap

```shell
`创建configmap`
[root@henry001 ~]# kubectl apply -f configmaps.yaml 
configmap/special-config created
configmap/env-config created

`查看创建的configmap`
[root@henry001 ~]# kubectl get configmap
NAME             DATA   AGE
config           0      14m    #从目录中创建
env-config       1      18s    #yaml文件创建
my-config        1      70m    #命令行创建
myconfig         1      31m    #配置文件创建
special-config   1      18s    #yaml文件创建

```



#### 5 ConfigMap的使用

- 使用方式

```
(1)通过环境变量的方式，直接传递给pod
	使用configmap中指定的key
	使用configmap中所有的key
(2)通过在pod的命令行下运行的方式(启动命令中)
(3)作为volume的方式挂载到pod内
```

- 注意

```
(1)ConfigMap必须在Pod使用它之前创建
(2)使用envFrom时，将会自动忽略无效的键
(3)Pod只能使用同一个命名空间的ConfigMap
```

##### 5.1 通过环境变量使用

> 使用valueFrom、configMapKeyRef、name
>
> key的话指定要用到的key
>

01 创建yaml文件

```
vim  test-pod.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
    - name: test-container
      image: busybox
      command: [ "/bin/sh", "-c", "env" ]
      env:
        # Define the environment variable
        - name: SPECIAL_LEVEL_KEY
          valueFrom:
            configMapKeyRef:
              # The ConfigMap containing the value you want to assign to SPECIAL_LEVEL_KEY
              name: special-config     
              # Specify the key associated with the value
              key: special.how
  restartPolicy: Never
```

02 执行yaml文件

```
kubectl apply -f test-pod.yaml
```

03 查看

```shell
`查看pod`
[root@henry001 controller]# kubectl get pods
NAME            READY   STATUS      RESTARTS   AGE
dapi-test-pod   0/1     Completed   0          89s

`查看日志`
[root@henry001 controller]# kubectl logs dapi-test-pod
KUBERNETES_SERVICE_PORT=443
KUBERNETES_PORT=tcp://10.96.0.1:443
HOSTNAME=dapi-test-pod
SHLVL=1
HOME=/root
KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
KUBERNETES_PORT_443_TCP_PORT=443
KUBERNETES_PORT_443_TCP_PROTO=tcp
SPECIAL_LEVEL_KEY=very              #名称为special-config的configmap中special.how值为very
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
KUBERNETES_SERVICE_HOST=10.96.0.1
PWD=/

```



##### 5.2 用作命令行参数

在命令行下引用时，需要先设置为环境变量，之后可以用过$(VAR_NAME)设置容器启动命令的启动参数

01 创建yaml文件

```
vim  test-pod2.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod2
spec:
  containers:
    - name: test-container
      image: busybox
      command: [ "/bin/sh", "-c", "echo $(SPECIAL_LEVEL_KEY)" ]
      env:
        - name: SPECIAL_LEVEL_KEY
          valueFrom:
            configMapKeyRef:
              name: special-config
              key: special.how
  restartPolicy: Never
```

02 执行yaml文件

```
kubectl apply -f test-pod2.yaml
```

03 查看

```shell
`查看pod`
[root@henry001 controller]# kubectl get pods
NAME             READY   STATUS      RESTARTS   AGE
dapi-test-pod    0/1     Completed   0          14m
dapi-test-pod2   0/1     Completed   0          5s

`查看打印的日志`
[root@henry001 controller]# kubectl logs dapi-test-pod2
very

```

##### 5.3 作为volume挂载使用

将创建的ConfigMap直接挂载至Pod的/etc/config目录下，其中每一个key-value键值对都会生成一个文件，key为文件名，value为内容。

01 创建yaml文件

```
vim  pod-myconfigmap-volume.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-configmap2
spec:
  containers:
    - name: test-container
      image: busybox
      volumeMounts:
      - name: config-volume
        mountPath: /etc/config
  volumes:
    - name: config-volume
      configMap:
        name: special-config
  restartPolicy: Never
```

02 执行yaml文件

```
kubectl apply -f  pod-myconfigmap-volume.yaml
```

03 查看

```shell
`查看pod`
[root@henry001 controller]# kubectl get pods
NAME             READY   STATUS      RESTARTS   AGE
dapi-test-pod    0/1     Completed   0          3h28m
dapi-test-pod2   0/1     Completed   0          3h13m
pod-configmap2   0/1     Running     0          3m56s

`进入pod的容器中`
[root@henry001 controller]# kubectl exec -it pod-configmap2 bash
root@pod-configmap3:/# cd /etc/config/

`在/etc/config的文件目录下有一个special.how命名的文件，是名为special-config的configmap的key值`
root@pod-configmap3:/etc/config# ls
special.how

`查看内容，是名为special-config的configmap的value值`
root@pod-configmap3:/etc/config# cat special.how 
very

```