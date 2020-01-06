### 1.6 Secret

> `官网`：<https://kubernetes.io/docs/concepts/configuration/secret/>
>
> ```
> Kubernetes secret objects let you store and manage sensitive information, such as passwords, OAuth tokens, and ssh keys.
> ```

#### 1 分类

- Opaque：使用base64编码存储信息，可以通过`base64 --decode`解码获得原始数据，因此安全性弱。
- kubernetes.io/dockerconfigjson：用于存储docker registry的认证信息。
- kubernetes.io/service-account-token：用于被 serviceaccount 引用。serviceaccout 创建时 Kubernetes 会默认创建对应的 secret。Pod 如果使用了 serviceaccount，对应的 secret 会自动挂载到 Pod 的 /run/secrets/kubernetes.io/serviceaccount 目录中。

#### 2 Opaque Secret

> Opaque类型的Secret的value为base64位编码后的值

##### 2.1 从文件中创建

```shell
`创建两个文件`
echo -n "admin" > ./username.txt
echo -n "1f2d1e2e67df" > ./password.txt
```

```shell
kubectl create secret generic db-user-pass --from-file=./username.txt --from-file=./password.txt
```

```shell
`查看生成的secret`
[root@henry001 secrete]# kubectl get secret
NAME                  TYPE                                  DATA   AGE
db-user-pass          Opaque                                2      41s
default-token-ql8nb   kubernetes.io/service-account-token   3      17h

```

##### 2.2 使用yaml文件创建

> (1)对数据进行64位编码

```shell
[root@henry001 secrete]# echo -n 'admin' | base64
YWRtaW4=
[root@henry001 secrete]# echo -n '1f2d1e2e67df' | base64
MWYyZDFlMmU2N2Rm

```

> (2)定义mysecret.yaml文件

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  username: YWRtaW4=
  password: MWYyZDFlMmU2N2Rm
```

> (3)根据yaml文件创建资源并查看

```shell
`创建secret`
[root@henry001 secrete]# kubectl apply -f mysecret.yaml 
secret/mysecret created

`查看secret`
[root@henry001 secrete]# kubectl get secret
NAME                  TYPE                                  DATA   AGE
db-user-pass          Opaque                                2      6m
default-token-ql8nb   kubernetes.io/service-account-token   3      17h
mysecret              Opaque                                2      7s

`查看yaml文件`
[root@henry001 secrete]# kubectl get secret mysecret -o yaml
apiVersion: v1
data:
  password: MWYyZDFlMmU2N2Rm
  username: YWRtaW4=
kind: Secret
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","data":{"password":"MWYyZDFlMmU2N2Rm","username":"YWRtaW4="},"kind":"Secret","metadata":{"annotations":{},"name":"mysecret","namespace":"default"},"type":"Opaque"}
  creationTimestamp: "2020-01-01T07:36:45Z"
  name: mysecret
  namespace: default
  resourceVersion: "93259"
  selfLink: /api/v1/namespaces/default/secrets/mysecret
  uid: 76382c7c-2c69-11ea-8e59-00163e2c61c9
type: Opaque

```

#### 3 Secret使用

- 以Volume方式
- 以环境变量方式

##### 3.1 将Secret挂载到Volume中

01 创建yaml文件

```
vim mypod-secret.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: mypod
    image: redis
    volumeMounts:
    - name: foo
      mountPath: "/etc/foo"
      readOnly: true
  volumes:
  - name: foo
    secret:
      secretName: mysecret
```

02 执行yaml文件

```
[root@henry001 secrete]# kubectl apply -f mypod-secret.yaml 
pod/mypod created

[root@henry001 secrete]# kubectl get pods
NAME             READY   STATUS      RESTARTS   AGE
mypod            1/1     Running     0          12s
pod-configmap2   0/1     Completed   0          60m

```

03查看信息

```shell
`进入pod容器`
[root@henry001 secrete]# kubectl exec -it mypod bash

`进入挂载目录`
root@mypod:/data# cd /etc/foo/
root@mypod:/etc/foo# ls
password  username

`查看文件内容`
root@mypod:/etc/foo# cat password 
1f2d1e2e67df
root@mypod:/etc/foo# cat username 
admin

```

##### 3.2 将Secret设置为环境变量

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-env-pod
spec:
  containers:
  - name: mycontainer
    image: redis
    env:
      - name: SECRET_USERNAME
        valueFrom:
          secretKeyRef:
            name: mysecret
            key: username
      - name: SECRET_PASSWORD
        valueFrom:
          secretKeyRef:
            name: mysecret
            key: password
  restartPolicy: Never
```

redis的环境变量SECRET_USERNAME的值为mysecret中key为username的value值，SECRET_PASSWORD的值为mysecret中key为password的value值。



#### 4 kubernetes.io/dockerconfigjson

kubernetes.io/dockerconfigjson用于存储docker registry的认证信息，可以直接使用`kubectl create secret`命令创建。



#### 5 kubernetes.io/service-account-token

用于被 serviceaccount 引用。serviceaccout 创建时 Kubernetes 会默认创建对应的 secret。Pod 如果使用了 serviceaccount，对应的 secret 会自动挂载到 Pod 的 /run/secrets/kubernetes.io/serviceaccount 目录中。

```shell
kubectl get secret   # 可以看到service-account-token

kubectl run nginx --image nginx
kubectl get pods
kubectl exec -it nginx-pod-name bash
ls /run/secrets/kubernetes.io/serviceaccount
```

```shell
`查看系统中的service-account-token`
[root@henry001 ~]# kubectl get secret
NAME                  TYPE                                  DATA   AGE
db-user-pass          Opaque                                2      23m
default-token-ql8nb   kubernetes.io/service-account-token   3      17h   #this one
mysecret              Opaque                                2      17m

```



```shell
`运行一个pod`
[root@henry001 ~]# kubectl run  nginx --image nginx
kubectl run --generator=deployment/apps.v1 is DEPRECATED and will be removed in a future version. Use kubectl run --generator=run-pod/v1 or kubectl create instead.
deployment.apps/nginx created

[root@henry001 ~]# kubectl get pods
NAME                     READY   STATUS      RESTARTS   AGE
mypod                    1/1     Running     0          16m
nginx-7db9fccd9b-89ljz   1/1     Running     0          99s
pod-configmap2           0/1     Completed   0          76m
`进入容器`
[root@henry001 ~]# kubectl exec -it nginx-7db9fccd9b-89ljz bash

`查看secret信息`
root@nginx-7db9fccd9b-89ljz:/# ls /run/secrets/kubernetes.io/serviceaccount   
ca.crt	namespace  token

```

查看yaml信息

```shell
[root@henry001 ~]# kubectl get pods nginx-7db9fccd9b-89ljz -o yaml   

  volumeMounts:
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount   #挂载目录
      name: default-token-ql8nb     
      readOnly: true
 ..........
 ..........
  volumes:
  - name: default-token-ql8nb      
    secret:
      defaultMode: 420
      secretName: default-token-ql8nb   #service-account-token
```

重点关注：

* volumes选项，定位到-name，secretName

* volumeMounts选项，定位到mountPath: /var/run/secrets/kubernetes.io/serviceaccount



**小结**：无论是ConfigMap，Secret，还是DownwardAPI，都是通过ProjectedVolume实现的，可以通过APIServer将信息放到Pod中进行使用。