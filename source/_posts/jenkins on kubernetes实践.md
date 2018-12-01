# jenkins on kubernetes实践

title: jenkins on kubernetes实践
date: 2017-08-26 21:05:46
categories: 容器技术
tags: [jenkins,docker,kubernetes]

------

**jenkins是什么？**
Jenkins是一个开源的持续集成工具，可用于自动化的执行与构建，测试和交付或部署软件有关的各种任务,有非常丰富的插件支持。
**kubernetes是什么？**
Kubernetes是容器集群管理系统，是一个开源的平台，可以实现容器集群的自动化部署、自动扩缩容、维护等功能。[这个视频生动地介绍了k8s][1]
**jenkins on k8s 有什么好处？**
jenkins通过单Master多个Slave的方式提供服务，Master保存了任务的配置信息，安装的插件等等，而slave主要负责执行任务，在使用中存在以下几个问题：
<!--more-->
1. 当存在多个slave时，运行slave的机器难以统一管理，每次添加新节点时总要做大量的重复工作。
2. 由于不同业务的构建频率并不相同，在使用会发现有很多slave大多数时间都处于空闲状态，造成资源浪费
3. jenkins默认采取保守的调度方式，造成某些slave的负载过高，任务不能平均分配
![jenkins架构][2]

使用k8s管理jenkins具有以下优势：

1. 使用docker运行jenkins保证环境的一致性，可以根据不同业务选择合适的镜像
2. k8s对抽象后的资源（pods）进行统一的管理调度，提供资源隔离和共享，使机器计算资源变得弹性可扩展,避免资源浪费。
3. k8s提供容器的自愈功能，能够保证始终有一定数量的容器是可用的
4. k8s默认的调度器提供了针对节点当前资源分配容器的调度策略，调度器支持插件化部署便于自定义。

### **一，搭建环境**

#### 工具准备

> kubernetes v1.8.4
docker v1.12.6
jenkins master镜像 jenkins/jenkins:lts（v2.73.3）
slave镜像 jenkinsci/jnlp-slave
Kubernetes plugin (v1.1)

#### 安装kubernetes集群

中文教程：https://www.kubernetes.org.cn/2906.html
安装成功之后访问dashboard地址就可以看到集群的控制面板：
![dashboard][3]
集群安装完成后，创建命名空间kubernetes-plugin
```
kubectl create namespace kubernetes-plugin
kubectl config set-context $(kubectl config current-context) --namespace=kubernetes-plugin
kubectl create -f service-account.yml
```

### **二，创建StatefulSet**
>**StatefulSet(有状态副本集)**：Deployments适用于运行无状态应用，StatefulSet则为有状态的应用提供了支持，可以为应用提供有序的部署和扩展，稳定的持久化存储，我们使用SS来运行jenkins master。

创建完整的Stateful Set需要依次创建一下对象：
1、Persistent Volume
2、Persistent Volume Claim
3、StatefulSet
4、Service


#### **创建PersistentVolume：**

为了保存应用运行时的数据需要先创建k8s的卷文件，K8s中存在Volume和PersistentVolume两种类型：

 >- Volume：与docker中的volume不同在于Volume生命周期是和pod绑定的，与pod中的container无关。k8s为Volume提供了多种类型文件系统（cephfs,nfs...,简单起见我直接选择了hostPath，使用的node节点本地的存储系统）
 >- PersistentVolume:从名字可以看出来，PV的生命周期独立于使用它的pod，不会像volume随pod消失而消失，而是做为一个集群中的资源存在（像node节点一样），同时PV屏蔽了使用具体存储系统的细节。

k8s中的对象都是通过[yaml文件][4]来定义的，首先创建名为`jenkins-volume.yml`的文件:

**注意**：PV的创建有静态，动态两种方式，动态创建可以减少管理员的操作步骤，需要提供指定的[StorageClass][5]。为了测试方便，所以我们直接选择静态创建，manual是一个不存在的storage class
```
kind: PersistentVolume
apiVersion: v1
metadata:
  name: jenkins-volume
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/tmp/data"

```
master节点执行下面的命令，PV就手动创建完了
`kubectl create -f jenkins-volume1.yaml`
#### **创建PersistentVolumeClaim：**
>PersistentVolumeClaim(PVC):
持久化存储卷索取，如果说PV是集群中的资源，PVC就是资源的消费者，PVC可以指定需要的资源大小和访问方式,pod不会和PV直接接触，而是通过PVC来请求资源，PV的生成阶段叫做provision,生成PV后会绑定到PVC对象，然后才能被其他对象使用。

PV和PVC的生命周期如下图：
![pv life][6]

创建文件`jenkins-claim.yaml`
**注意：** name必须为jenkins-home-jenkins-0否则会绑定失败
```
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: jenkins-home-jenkins-0
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 3Gi
```
执行命令`kubectl create -f jenkins-claim.yaml`
然后查看PVC是否创建成功，status为bound说明PVC已经绑定
```
[root@master ~]# kubectl describe pvc jenkins-home-jenkins-0
Name:          jenkins-home-jenkins-0
Namespace:     kubernetes-plugin
StorageClass:  manual
Status:        Bound
Volume:        jenkins-volume
Labels:        <none>
Annotations:   pv.kubernetes.io/bind-completed=yes
               pv.kubernetes.io/bound-by-controller=yes
Capacity:      10Gi
Access Modes:  RWO
Events:        <none>
```
#### **创建StatefulSet和Service：**

从kubernetes-plugin github仓库下载jenkins.yml文件
```
wget https://raw.githubusercontent.com/jenkinsci/kubernetes-plugin/master/src/main/kubernetes/jenkins.yml
```
修改jenkins.yml：
去掉87行`externalTrafficPolicy: Local`（这是GKE使用的）
修改83行` type: LoadBalancer`改为` type: NodePort`

**注意：** 
service type=ClusterIP时只允许从集群内部访问， type设置为NodePort是为了从集群外的机器访问jenkins,请谨慎使用，开启NodePort会在所有节点（含master）的统一固定端口开放服务。

执行命令
```
[root@master ~]# kubectl create -f jenkins.yml 
statefulset "jenkins" created
service "jenkins" created
```
访问jenkins master,地址为`masterip:32058`
```
#查看映射的端口
[root@master ~]# kubectl get service jenkins
NAME      TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)                        AGE
jenkins   NodePort   10.96.82.68   <none>        80:32058/TCP,50000:30345/TCP   1m
```
查看pod : jenkins-0的容器日志，粘贴下面的密码进入jenkins,jenkins安装完成。
> Jenkins initial setup is required. An admin user has been created and a password generated.
Please use the following password to proceed to installation:
70aa7b41ba894855abccd09306625b8a 

刷新dashboard，切换命名空间到kubernetes-plugin,结果如下：
![success][7]

### 问题分析
1.创建stateful set时失败，提示"PersistentVolumeClaim is not bound: "jenkins-home-jenkins-0"："
因为采用静态创建PV时，StatefulSet会按照固定名称查找PVC，PVC的名字要满足
>PVC_name == volumeClaimTemplates_name + "-" + pod_name

这里的名字就是`jenkins-home-jenkins-0`

2.pod启动失败，jenkins用户没有目录权限
错误提示"touch: cannot touch ‘/var/jenkins_home/copy_reference_file.log’: Permission denied
Can not write to /var/jenkins_home/copy_reference_file.log. Wrong volume permissions?"
要确保节点目录开放权限,在node上执行命令：
```
sudo chown -R 1000:1000 /var/jenkins_home/
sudo chown -R 1000:1000 /tmp/data
##如果仍然失败，尝试在node上重启docker
systemctl restart docker

```
注意pv指定的hostPath权限也要修改，否则是无效的



###  三 ，配置jenkins
#### 创建jenkins服务账号
```
wget https://raw.githubusercontent.com/jenkinsci/kubernetes-plugin/master/src/main/kubernetes/service-account.yml
kubectl create -f service-account.yml
```
#### 配置插件
访问`http://masterip:32058/pluginManager/`,搜索插件Kubernetes plugin安装；
访问 http://masterip:32058/configure
选择新建云--kubernetes,在URl填写api server地址，
执行kubectl describe命令，复制output中的token，填入到 ‘Kubernetes server certificate key’
```
[root@master ~]# kubectl get secret 
NAME                  TYPE                                  DATA      AGE
default-token-4kb54   kubernetes.io/service-account-token   3         1d
jenkins-token-wzbsx   kubernetes.io/service-account-token   3         1d
[root@master ~]# kubectl describe secret/jenkins-token-wzbsx
...
```
jenkins url,tunnel填写service的CLUSTER-IP即可，结果如图：
![peizhi1][8]
选择add pod template，填写下面的内容，retain slave可以设置运行jenkins slave 的container空闲后能存活多久。
![content][9]
插件配置完成。
### 四 ，测试
#### 1. 扩容测试
**StatefulSet扩容：**
首先需要手动创建PV，PVC(见第二步),然后执行扩容命令
` kubectl scale statefulset/jenkins --replicas=２`
查看StatefulSet,此时已经拥有两个master节点，访问service时会**随机**将请求发送给后端的master。
```
[root@master ~]# kubectl get statefulset/jenkins 
NAME      DESIRED   CURRENT   AGE
jenkins   2         2         5d
```
虽然通过k8s可以轻松实现jenkins master节点的拓展，但是由于jenkins存储数据的方式通过本地文件存储，master之间的数据同步还是一个麻烦的问题，参考[jenkins存储模型][10]。

*jenkins master上保存的文件：*
```
ls /temp/data
jenkins.CLI.xml
jenkins.install.InstallUtil.lastExecVersion
jenkins.install.UpgradeWizard.state
jenkins.model.ArtifactManagerConfiguration.xml
jenkins.model.JenkinsLocationConfiguration.xml
jobs
logs
nodeMonitors.xml
nodes
```

#### 2. 高可用测试
现在stateful set中已经有两个pod,在jenkins-1所在的节点执行`docker stop`停止运行jenkins-master的容器，同时在命令行查看pod的状态，可以看到jenkins-1异常（Error状态）之后慢慢恢复了运行状态（Running）：
```
[root@master ~]# kubectl get pods -w
NAME        READY     STATUS    RESTARTS   AGE
jenkins-0   1/1       Running   0          1d
jenkins-1   0/1       Running   1         20h
jenkins-1   1/1       Running   1         20h
jenkins-1   0/1       Error     1         20h
jenkins-1   0/1       CrashLoopBackOff   1         20h
jenkins-1   0/1       Running   2         20h
jenkins-1   1/1       Running   2         20h
```
`kubectl describe pod jenkins-1`查看pod的事件日志，k8s通过探针(probe)接口检测到服务停止之后自动执行了拉取镜像，重启container的操作。
```
Events:
  Type     Reason      Age                From                              Message
  ----     ------      ----               ----                              -------
  Warning  Unhealthy   27m (x2 over 27m)  kubelet, iz8pscwd1fv6kprs1zw21kz  Readiness probe failed: HTTP probe failed with statuscode: 503
  Warning  Unhealthy   27m (x2 over 27m)  kubelet, iz8pscwd1fv6kprs1zw21kz  Liveness probe failed: HTTP probe failed with statuscode: 503
  Warning  Unhealthy   24m                kubelet, iz8pscwd1fv6kprs1zw21kz  Readiness probe failed: Get http://192.168.24.4:8080/login: dial tcp 192.168.24.4:8080: getsockopt: connection refused
  Warning  BackOff     20m (x2 over 20m)  kubelet, iz8pscwd1fv6kprs1zw21kz  Back-off restarting failed container
  Warning  FailedSync  20m (x2 over 20m)  kubelet, iz8pscwd1fv6kprs1zw21kz  Error syncing pod
  Normal   Pulling     19m (x3 over 20h)  kubelet, iz8pscwd1fv6kprs1zw21kz  pulling image "jenkins/jenkins:lts-alpine"
  Normal   Started     19m (x3 over 20h)  kubelet, iz8pscwd1fv6kprs1zw21kz  Started container
  Normal   Pulled      19m (x3 over 20h)  kubelet, iz8pscwd1fv6kprs1zw21kz  Successfully pulled image "jenkins/jenkins:lts-alpine"
  Normal   Created     19m (x3 over 20h)  kubelet, iz8pscwd1fv6kprs1zw21kz  Created container
```
#### 3. jenkins构建测试
当前集群中使用的jenkins slave镜像只包含一个java运行环境来运行jenkins-slave.jar,在实际使用中需要自定义合适的镜像。选择自定义镜像之后需要修改插件的配置，同样name命名为jnlp替换默认镜像，arguments安装工具提示填写即可。
![container][11]
创建job，同时开始构建,k8s会在不同节点上创建pod来运行任务
![此处输入图片的描述][13]
构建全部完成后，资源随即被释放
![此处输入图片的描述][14]

**jenkins默认调度策略**
1. 尝试在上次构建的节点上构建，指定某台slave之后会一直使用。
2.当队列有2个构建时，不会立刻创建两个executor,而是先创建一个executor然后尝试等待executor空闲，目的是保证每个executor被充分利用。
**k8s调度策略**
1. 使用Pod.spec.nodeSelector根据label为pod选择node
2.调度器scheduler有Predicates，Priorities两个阶段，分别负责节点过滤和评分排序，各个阶段都有k8s提供的检查项，我们可以自由组合。
（比如PodFitsResources检查cpu内存等资源，PodFitsHostPorts检查端口占用，SelectorSpreadPriority要求一个服务尽量分散分布）[自定义schduler参考][15]
**资源不足时会发生什么**
当前集群中有3个节点，我在node2运行一个CPU占用限制在80%的程序,然后设置jenkins插件ContainerTemplate的request和limit均为`cpu 500m,内存500Mi`,（500m代表单核CPU的50%）看一下pod会怎么调度
k8s仍然尝试在node2分配节点（为什么其他节点不行），结果POD处于pending状态：
```"status": {
    "phase": "Pending",
    "conditions": [
      {
        "type": "PodScheduled",
        "status": "False",
        "lastProbeTime": null,
        "lastTransitionTime": "2017-12-09T08:29:10Z",
        "reason": "Unschedulable",
        "message": "No nodes are available that match all of the predicates: Insufficient cpu (4), PodToleratesNodeTaints (1)."
      }
    ],
    "qosClass": "Guaranteed"
```
最后pod被删除，而jenkins任务会阻塞一直到有其他空闲的slave出现。
### **四 ，总结**
本文介绍了在k8s集群部署jenkins服务的方式和k8s带来的资源管理便捷，由于我也是刚开始接触k8s,所用的实例只是搭建了用于测试的实验环境，离在实际生产环境中使用还有问题需要验证。


  [1]: http://docs.kubernetes.org.cn/227.html
  [2]: http://7xl4v5.com1.z0.glb.clouddn.com/jenkins-arch.png
  [3]: http://7xl4v5.com1.z0.glb.clouddn.com/kongzhimianban.png
  [4]: http://www.ruanyifeng.com/blog/2016/07/yaml.html
  [5]: https://kubernetes.io/docs/concepts/storage/storage-classes/
  [6]: http://7xl4v5.com1.z0.glb.clouddn.com/pvlife.png
  [7]: http://7xl4v5.com1.z0.glb.clouddn.com/success.png
  [8]: http://7xl4v5.com1.z0.glb.clouddn.com/config1.png
  [9]: http://7xl4v5.com1.z0.glb.clouddn.com/config.png
  [10]: https://yq.aliyun.com/articles/224376
  [11]: http://7xl4v5.com1.z0.glb.clouddn.com/%E7%81%AB%E7%8B%90%E6%88%AA%E5%9B%BE_2017-12-08T08-06-23.616Z.png
  [12]: http://7xl4v5.com1.z0.glb.clouddn.com/jobs.png
  [13]: http://7xl4v5.com1.z0.glb.clouddn.com/pods1.png
  [14]: http://7xl4v5.com1.z0.glb.clouddn.com/result2.png
  [15]: http://dockone.io/article/2885