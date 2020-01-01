

## 1 Job

> `官网`：https://kubernetes.io/docs/concepts/workloads/controllers/jobs-run-to-completion/
>
> ```
> A Job creates one or more Pods and ensures that a specified number of them successfully terminate. As pods successfully complete, the Job tracks the successful completions. When a specified number of successful completions is reached, the task (ie, Job) is complete. Deleting a Job will clean up the Pods it created.
> ```
>

* 对于RS，RC之类的控制器，能够保持Pod按照预期数目持久地运行下去，它们针对的是持久性的任务，比如web服务。

* 而有些操作其实不需要持久，比如压缩文件，我们希望任务完成之后，Pod就结束运行，不需要保持在系统中，此时就需要用到Job。

* 可以这样理解，Job是对RS、RC等持久性控制器的补充。

* 负责批量处理短暂的一次性任务，仅执行一次，并保证处理的一个或者多个Pod成功结束。

>  **Job定义方法与ReplicaSet等控制器相似，只有细微差别，如下：**

- Job中的restart policy必需是"Never"或者"OnFailure"，这个很好理解，因为pod要运行到结束，而不是反复重新启动。
- Job不需要选择器，其中的pod也不需要标签，系统在创建Job时会自动添加相关内容。当然用户也可以出于资源组织的目的添加标签，但这个与Job本身的实现没有关系。
- Job新增加两个字段：.spec.completions、.spec.parallelism。
- backoffLimit字段



下面做一个j简单的案列：

> 1、 新建一个yaml文件，用来创建一个job

```
vim job.yaml
```

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: job-demo
spec:
  template:
    metadata:
      name: job-demo
    spec:
      restartPolicy: Never
      containers:
      - name: counter
        image: busybox
        command:
        - "bin/sh"
        - "-c"
        - "for i in 9 8 7 6 5 4 3 2 1; do echo $i; done"   #输出9-1的数字，然后结束
```

>2、执行yaml文件，产生一个job 

```
kubectl apply -f job.yaml
```

>3、查看一下job的信息

```shell
`pod的状态是Completed`
[root@henry001 controller]# kubectl get pods
NAME             READY   STATUS      RESTARTS   AGE
job-demo-ffwkt   0/1     Completed   0          42s

``
[root@henry001 controller]# kubectl describe jobs/job-demo
Name:           job-demo
Namespace:      default
Selector:       controller-uid=677b8c78-2be4-11ea-8e59-00163e2c61c9
Labels:         controller-uid=677b8c78-2be4-11ea-8e59-00163e2c61c9
                job-name=job-demo
Annotations:    kubectl.kubernetes.io/last-applied-configuration:
                  {"apiVersion":"batch/v1","kind":"Job","metadata":{"annotations":{},"name":"job-demo","namespace":"default"},"spec":{"template":{"metadata"...
Parallelism:    1
Completions:    1
Start Time:     Tue, 31 Dec 2019 23:44:17 +0800
Completed At:   Tue, 31 Dec 2019 23:44:26 +0800
Duration:       9s
Pods Statuses:  0 Running / 1 Succeeded / 0 Failed
Pod Template:
  Labels:  controller-uid=677b8c78-2be4-11ea-8e59-00163e2c61c9
           job-name=job-demo
  Containers:
   counter:
    Image:      busybox
    Port:       <none>
    Host Port:  <none>
    Command:
      bin/sh
      -c
      for i in 9 8 7 6 5 4 3 2 1; do echo $i; done
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Events:
  Type    Reason            Age    From            Message
  ----    ------            ----   ----            -------
  Normal  SuccessfulCreate  8m36s  job-controller  Created pod: job-demo-ffwkt

```

> 4、查看输出的日志

```shell
[root@henry001 controller]# kubectl logs job-demo-ffwkt
9
8
7
6
5
4
3
2
1
```

**job的分类**：

- 非并行Job:
  - 通常只运行一个Pod，Pod成功结束Job就退出。
- 固定完成次数的并行Job:
  - 并发运行指定数量的Pod，直到指定数量的Pod成功，Job结束。
- 带有工作队列的并行Job:
  - 用户可以指定并行的Pod数量，当任何Pod成功结束后，不会再创建新的Pod
  - 一旦有一个Pod成功结束，并且所有的Pods都结束了，该Job就成功结束。
  - 一旦有一个Pod成功结束，其他Pods都会准备退出。

## 2 CronJob

> `官网`：https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/
>
> ```
> A Cron Job creates Jobs on a time-based schedule.
> 
> One CronJob object is like one line of a crontab (cron table) file. It runs a job periodically on a given schedule, written in Cron format.
> ```
>
> cronJob是基于时间进行任务的定时管理。

- 在特定的时间点运行任务
- 反复在指定的时间点运行任务：比如定时进行数据库备份，定时发送电子邮件等等。

