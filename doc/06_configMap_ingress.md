#### ConfigMap在Ingress Controller中实战

> 在之前ingress网络中的mandatory.yaml文件中使用了ConfigMap，于是我们可以打开
>
> 可以发现有nginx-configuration、tcp-services等名称的cm
>
> 而且也可以发现最后在容器的参数中使用了这些cm
>
> ```yaml
> containers:
>         - name: nginx-ingress-controller
>        image: quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.26.1
>        args:
>             - /nginx-ingress-controller
>             - --configmap=$(POD_NAMESPACE)/nginx-configuration
>             - --tcp-services-configmap=$(POD_NAMESPACE)/tcp-services
>             - --udp-services-configmap=$(POD_NAMESPACE)/udp-services
>             - --publish-service=$(POD_NAMESPACE)/ingress-nginx
>             - --annotations-prefix=nginx.ingress.kubernetes.io
> ```
>
> **开启证明之旅和cm的使用方式**

> (1)查看nginx ingress controller的pod部署
>
> kubectl get pods -n ingress-nginx -o wide

```
NAME                                        READY   STATUS    RESTARTS   AGE
nginx-ingress-controller-7c66dcdd6c-v8grg   1/1     Running   0          8d

NAME                                        READY   STATUS    RESTARTS   AGE   IP              NODE   NOMINATED NODE   READINESS GATES
nginx-ingress-controller-7c66dcdd6c-v8grg   1/1     Running   0          8d    172.16.31.150   w1     <none>           <none>
```

> (2)发现运行在w1节点上，说明w1上一定有对应的container，来到w1节点
>
> docker ps | grep ingress

```
ddde4b354852        quay.io/kubernetes-ingress-controller/nginx-ingress-controller   "/usr/bin/dumb-init …"   8 days ago          Up 8 days                               k8s_nginx-ingress-controller_nginx-ingress-controller-7c66dcdd6c-v8grg_ingress-nginx_b3e2f9a5-0943-11ea-b2b3-00163e0edcbd_0

b6b7412855c5        k8s.gcr.io/pause:3.1                                             "/pause"                 8 days ago          Up 8 days                               k8s_POD_nginx-ingress-controller-7c66dcdd6c-v8grg_ingress-nginx_b3e2f9a5-0943-11ea-b2b3-00163e0edcbd_0
```

> (3)不妨进入容器看看？
>
> docker exec -it ddde4b354852 bash

> (4)可以发现，就是一个nginx嘛，而且里面还有nginx.conf文件，美滋滋

```
/etc/nginx/nginx.conf
```

> (5)不妨打开nginx.conf文件看看
>
> 假如已经配置过ingress，不妨尝试搜索一下"k8s.demoxxx"/"itcrazy2016.com"

```conf
server {
	server_name k8s.itcrazy2016.com ;
```

> (6)到这里，大家应该有点感觉了，原来nginx ingress controller就是一个nginx，而所谓的ingress.yaml文件中配置的内容像itcrazy2016.com就会对应到nginx.conf中。

> (7)但是，不可能每次都进入到容器里面来修改，而且还需要手动重启nginx，很麻烦
>
> 一定会有好事之者来做这件事情，比如在K8s中有对应的方式，修改了什么就能修改nginx.conf文件

> (8)先查看一下nginx.conf文件中的内容，比如找个属性：proxy_connect_timeout 5s
>
> 我们想要将这个属性在K8s中修改成8s，可以吗？
>
> kubectl get cm -n ingress-nginx
>
> **网盘/课堂源码/nginx-config.yaml**
>
> kubectl apply -f nginx-config.yaml
>
> kubectl get cm -n ingress-nginx

```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: nginx-configuration
  namespace: ingress-nginx
  labels:
    app: ingress-nginx
data:
  proxy-read-timeout: "208"
```

> (9)再次查看nginx.conf文件

> (10)其实定义规则都在nginx ingress controller的官网中
>
> <https://kubernetes.github.io/ingress-nginx/>
>
> <https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/>