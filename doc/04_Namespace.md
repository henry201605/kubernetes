# 1、 Namespace



>`官网`：https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/

```
Kubernetes supports multiple virtual clusters backed by the same physical cluster. These virtual clusters are called namespaces.
```



> kubectl get pods
>
> kubectl get pods -n kube-system

比较一下，上述两行命令的输入是否一样，发现不一样，是因为Pod属于不同的Namespace。

> 查看一下当前的命名空间：kubectl get namespaces/ns
>
> ```
> NAME              STATUS   AGE
> default           Active   27m
> kube-node-lease   Active   27m
> kube-public       Active   27m
> kube-system       Active   27m
> ```

其实说白了，命名空间就是为了隔离不同的资源，比如：Pod、Service、Deployment等。可以在输入命令的时候指定命名空间`-n`，如果不指定，则使用默认的命名空间：default。

### 创建命名空间

> myns-namespace.yaml
>
> ```yaml
> apiVersion: v1
> kind: Namespace
> metadata:
> name: myns
> ```

kubectl apply -f myns-namespace.yaml

kubectl get namespaces/ns

```
NAME              STATUS   AGE
default           Active   38m
kube-node-lease   Active   38m
kube-public       Active   38m
kube-system       Active   38m
myns              Active   6s
```

### 指定命名空间下的资源

> 比如创建一个pod，属于myns命名空间下
>
> vi nginx-pod.yaml
>
> kubectl apply -f nginx-pod.yaml 

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  namespace: myns    # 命名空间
spec:
  containers:
  - name: nginx-container
    image: nginx
    ports:
    - containerPort: 80
```

> 查看myns命名空间下的Pod和资源
>
> kubectl get pods     #default命名空间下的pod
>
> kubectl get pods -n myns   #myns  命名空间下的pod
>
> kubectl get all -n myns     #myns  命名空间下所有的资源
>
> kubectl get pods --all-namespaces    #查找所有命名空间下的pod



# 2、 Labels and Selectors

> 在前面的yaml文件中，看到很多label，顾名思义，就是给一些资源打上标签的
>
> `官网`：https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/
>
> ```
> Labels are key/value pairs that are attached to objects, such as pods. Labels are intended to be used to specify identifying attributes of objects that are meaningful and relevant to users, but do not directly imply semantics to the core system. Labels can be used to organize and to select subsets of objects. 
> ```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
```

表示名称为nginx-pod的pod，有一个label，key为app，value为nginx。

我们可以将具有同一个label的pod，交给selector管理

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: 
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:             # 匹配具有同一个label属性的pod标签
    matchLabels:
      app: nginx         
  template:             # 定义pod的模板
    metadata:
      labels:
        app: nginx      # 定义当前pod的label属性，app为key，value为nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```

> 查看pod的label标签：kubectl get pods --show-labels
>