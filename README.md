# Amazon EKS node group management 

# 1 工作节点维护

## 1.1 Kubectl cordon命令

执行kubectl cordon命令时，工作节点上运行的Pod等对象会继续运行，但节点会被标记为SchedulingDisabled状态，此时用户新建的Pod等对象不会调度到此节点。 

查看所有节点状态，默认情况下均为Ready状态

```
kubectl get nodes
NAME                                               STATUS   ROLES    AGE   VERSION
ip-192-168-3-197.cn-northwest-1.compute.internal   Ready    <none>   10h   v1.14.9-eks-1f0ca9
ip-192-168-33-87.cn-northwest-1.compute.internal   Ready    <none>   10h   v1.14.9-eks-1f0ca9
```
执行kubectl cordon命令使节点进入维护状态，可以看到节点状态变为*Ready,SchedulingDisabled*

```
kubectl cordon ip-192-168-3-197.cn-northwest-1.compute.internal

kubectl get nodes
NAME                                               STATUS                     ROLES    AGE   VERSION
ip-192-168-3-197.cn-northwest-1.compute.internal   Ready,SchedulingDisabled   <none>   10h   v1.14.9-eks-1f0ca9
ip-192-168-33-87.cn-northwest-1.compute.internal   Ready                      <none>   10h   v1.14.9-eks-1f0ca9
```
创建Nginx Deployment，验证节点是否可以被调度

```
kubectl create deployment nginx --image=nginx

kubectl get deployment nginx
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
nginx   1/1     1            1           9s

kubectl scale deployment nginx --replicas=4

kubectl get deployment nginx
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
nginx   4/4     4            4           35s
```
可以看到，所有Pod均调度到另外1台节点

```
kubectl get pods -o wide

NAME                     READY   STATUS    RESTARTS   AGE   IP               NODE                                               NOMINATED NODE   READINESS GATES
nginx-554b9c67f9-b59sc   1/1     Running   0          2s    192.168.81.68    ip-192-168-85-96.cn-northwest-1.compute.internal   <none>           <none>
nginx-554b9c67f9-kr828   1/1     Running   0          25s   192.168.91.54    ip-192-168-85-96.cn-northwest-1.compute.internal   <none>           <none>
nginx-554b9c67f9-qjdgp   1/1     Running   0          2s    192.168.87.179   ip-192-168-85-96.cn-northwest-1.compute.internal   <none>           <none>
nginx-554b9c67f9-w4vg4   1/1     Running   0          2s    192.168.83.202   ip-192-168-85-96.cn-northwest-1.compute.internal   <none>           <none>
```

执行kubectl uncordon，将工作节点恢复到节点组中

```
kubectl uncordon ip-192-168-3-197.cn-northwest-1.compute.internal

kubectl get pods -o wide
NAME                     READY   STATUS    RESTARTS   AGE     IP               NODE                                               NOMINATED NODE   READINESS GATES
nginx-65f88748fd-fqzm2   1/1     Running   0          7m59s   192.168.24.171   ip-192-168-3-197.cn-northwest-1.compute.internal   <none>           <none>
nginx-65f88748fd-glm4q   1/1     Running   0          8m39s   192.168.44.164   ip-192-168-33-87.cn-northwest-1.compute.internal   <none>           <none>

```
调整Nginx Deployment副本数量，使Pod重新分布

```
kubectl scale deployment nginx --replicas=1
kubectl scale deployment nginx --replicas=4
```
再次查看Pod在节点上的分布状态，可以看到Pod被调度到所有节点

```
kubectl get pods -o wide

NAME                     READY   STATUS    RESTARTS   AGE     IP               NODE                                                NOMINATED NODE   READINESS GATES
nginx-554b9c67f9-7w24w   1/1     Running   0          3s      192.168.9.108    ip-192-168-30-112.cn-northwest-1.compute.internal   <none>           <none>
nginx-554b9c67f9-dszwj   1/1     Running   0          3s      192.168.28.3     ip-192-168-30-112.cn-northwest-1.compute.internal   <none>           <none>
nginx-554b9c67f9-kr828   1/1     Running   0          5m13s   192.168.91.54    ip-192-168-85-96.cn-northwest-1.compute.internal    <none>           <none>
nginx-554b9c67f9-n29br   1/1     Running   0          3s      192.168.85.206   ip-192-168-85-96.cn-northwest-1.compute.internal    <none>           <none>
```
## 1.2 Kubectl drain命令

执行Kubectl drain命令时，工作节点会被标记SchedulingDisabled，用户新建的Pod等对象不可调度到节点，同时节点上运行的Pod等对象也会被强行驱逐。

查看所有节点状态，默认情况下均为Ready状态

```
kubectl get nodes
NAME                                               STATUS   ROLES    AGE   VERSION
ip-192-168-3-197.cn-northwest-1.compute.internal   Ready    <none>   10h   v1.14.9-eks-1f0ca9
ip-192-168-33-87.cn-northwest-1.compute.internal   Ready    <none>   10h   v1.14.9-eks-1f0ca9
```
查看Pod在节点上的分布情况

```
kubectl get pods -o wide

NAME                     READY   STATUS    RESTARTS   AGE   IP               NODE                                                NOMINATED NODE   READINESS GATES
nginx-554b9c67f9-7w24w   1/1     Running   0          10m   192.168.9.108    ip-192-168-30-112.cn-northwest-1.compute.internal   <none>           <none>
nginx-554b9c67f9-dszwj   1/1     Running   0          10m   192.168.28.3     ip-192-168-30-112.cn-northwest-1.compute.internal   <none>           <none>
nginx-554b9c67f9-kr828   1/1     Running   0          15m   192.168.91.54    ip-192-168-85-96.cn-northwest-1.compute.internal    <none>           <none>
nginx-554b9c67f9-n29br   1/1     Running   0          10m   192.168.85.206   ip-192-168-85-96.cn-northwest-1.compute.internal    <none>           <none>
```
执行kubectl drain命令，使节点进入维护状态，同时驱逐节点上运行的Pod

注意：如果提示error: cannot delete Pods with local storage，请在命令后加上--delete-local-data参数

```
kubectl drain ip-192-168-30-112.cn-northwest-1.compute.internal --ignore-daemonsets --delete-local-data

node/ip-192-168-30-112.cn-northwest-1.compute.internal cordoned
WARNING: ignoring DaemonSet-managed Pods: kube-system/aws-node-rdzrz, kube-system/calico-node-m242x, kube-system/ebs-csi-node-fbqnr, kube-system/efs-csi-node-j6q5f, kube-system/kube-proxy-9pz4h
evicting pod "ebs-csi-controller-6dcc4dc6f4-6rb86"
evicting pod "nginx-554b9c67f9-7w24w"
evicting pod "nginx-554b9c67f9-dszwj"
evicting pod "calico-typha-horizontal-autoscaler-5ddb46bbd8-5d9f4"
evicting pod "cluster-autoscaler-989ffb-pvkjw"
evicting pod "coredns-6565755d58-c7q7s"
pod/coredns-6565755d58-c7q7s evicted
pod/calico-typha-horizontal-autoscaler-5ddb46bbd8-5d9f4 evicted
pod/nginx-554b9c67f9-dszwj evicted
pod/ebs-csi-controller-6dcc4dc6f4-6rb86 evicted
pod/nginx-554b9c67f9-7w24w evicted
pod/cluster-autoscaler-989ffb-pvkjw evicted
node/ip-192-168-30-112.cn-northwest-1.compute.internal evicted
```
查看节点状态

```
kubectl get nodes

ip-192-168-30-112.cn-northwest-1.compute.internal   Ready,SchedulingDisabled   <none>   36h   v1.15.10-eks-bac369
ip-192-168-85-96.cn-northwest-1.compute.internal    Ready                      <none>   35h   v1.15.10-eks-bac369
```
查看Pod在节点上的分布情况，可以看到Pod已经迁移到另外的节点

```
kubectl get pods --all-namespaces -o wide

NAME                     READY   STATUS    RESTARTS   AGE    IP               NODE                                               NOMINATED NODE   READINESS GATES
nginx-554b9c67f9-c7r82   0/1     Pending   0          102s   <none>           <none>                                             <none>           <none>
nginx-554b9c67f9-g4d7v   1/1     Running   0          103s   192.168.87.179   ip-192-168-85-96.cn-northwest-1.compute.internal   <none>           <none>
nginx-554b9c67f9-kr828   1/1     Running   0          20m    192.168.91.54    ip-192-168-85-96.cn-northwest-1.compute.internal   <none>           <none>
nginx-554b9c67f9-n29br   1/1     Running   0          14m    192.168.85.206   ip-192-168-85-96.cn-northwest-1.compute.internal   <none>           <none>
```
将节点恢复到工作节点组中，可以看到，Pod又可以被调度到该节点

```
kubectl uncordon ip-192-168-3-197.cn-northwest-1.compute.internal

kubectl get pods --all-namespaces -o wide

NAME                     READY   STATUS    RESTARTS   AGE     IP               NODE                                                NOMINATED NODE   READINESS GATES
nginx-554b9c67f9-c7r82   1/1     Running   0          3m49s   192.168.17.18    ip-192-168-30-112.cn-northwest-1.compute.internal   <none>           <none>
nginx-554b9c67f9-g4d7v   1/1     Running   0          3m50s   192.168.87.179   ip-192-168-85-96.cn-northwest-1.compute.internal    <none>           <none>
nginx-554b9c67f9-kr828   1/1     Running   0          22m     192.168.91.54    ip-192-168-85-96.cn-northwest-1.compute.internal    <none>           <none>
nginx-554b9c67f9-n29br   1/1     Running   0          17m     192.168.85.206   ip-192-168-85-96.cn-northwest-1.compute.internal    <none>           <none>
```

# 2 工作节点升级 

## 2.1 前提要求

创建1.14版本的集群和托管工作节点组

```
eksctl create cluster \
--name prod \
--version 1.14 \
--region cn-northwest-1 \
--managed \
--nodegroup-name managed-1-14 \
--node-type t3.medium \
--nodes 2 \
--nodes-min 1 \
--nodes-max 4 \
--alb-ingress-access
```
参照官方文档，升级EKS集群Kubernetes版本到1.15
https://docs.aws.amazon.com/zh_cn/eks/latest/userguide/update-cluster.html

## 2.2 更新托管工作节点组

获取托管节点组名字

```
eksctl get nodegroups --cluster test
```
参考输出

```
CLUSTER	NODEGROUP	CREATED			MIN SIZE	MAX SIZE	DESIRED CAPACITY	INSTANCE TYPE	IMAGE ID
test	test-nodegroup	2020-04-07T16:16:54Z	2		6		2			t3.micro
```
查看托管工作节点组AMI发行版本

```
aws eks describe-nodegroup --cluster-name test --nodegroup-name test-nodegroup | grep releaseVersion
```
参考输出

```
"releaseVersion": "1.14.9-20200228",
```
执行aws eks update-nodegroup-version命令升级AMI发行版本

```
aws eks update-nodegroup-version --cluster-name test --nodegroup-name test-nodegroup
```
参考输出

```
{
    "update": {
        "id": "ce13106a-5127-33e7-8b6d-07a0b262ebbb",
        "status": "InProgress",
        "type": "VersionUpdate",
        "params": [
            {
                "type": "Version",
                "value": "1.14"
            },
            {
                "type": "ReleaseVersion",
                "value": "1.14.9-20200228"
            }
        ],
        "createdAt": 1586339867.894,
        "errors": []
    }
}
```
大约等待30分钟，升级完成，再次查看托管节点组AMI发行版本

```
aws eks describe-nodegroup --cluster-name test --nodegroup-name test-nodegroup | grep releaseVersion
```
参考输出

```
"releaseVersion": "1.15.10-20200228",
```
托管节点组升级完成。

## 2.3 更新自行管理工作线程节点组

假定现有集群中已有名为test的1.14版本的自行管理工作线程节点组，接下来我们将采用" 迁移到新的工作线程节点组 "的方式进行升级。

查看集群中的所有托管节点组和自行管理节点组，

```
eksctl get nodegroups --cluster test
```
参考输出如下，NODEGROUP列下名为dev的节点组是提前创建的自行管理节点组

```
CLUSTER	NODEGROUP	CREATED			MIN SIZE	MAX SIZE	DESIRED CAPACITY	INSTANCE TYPE	IMAGE ID
test	dev		        2020-04-08T11:42:19Z	1		4		2			t3.medium	ami-047594a1f70bab3e2
test	test-nodegroup	2020-04-07T16:16:54Z	2		6		2			t3.micro
```
创建新的1.15版本的工作线程节点组

```
eksctl create nodegroup \
--cluster test \
--version 1.15 \
--name prod-1-15 \
--node-type t3.medium \
--node-ami auto \
--nodes 2 \
--nodes-min 1 \
--nodes-max 4
```
创建完成后，查看所有节点的状态是否都已经Ready

```
kubectl get nodes
```
参考输出

```
NAME                                                STATUS   ROLES    AGE   VERSION
ip-192-168-16-40.cn-northwest-1.compute.internal    Ready    <none>   47s   v1.15.10-eks-bac369
ip-192-168-17-102.cn-northwest-1.compute.internal   Ready    <none>   41m   v1.14.9-eks-1f0ca9
ip-192-168-55-90.cn-northwest-1.compute.internal    Ready    <none>   39m   v1.15.10-eks-bac369
ip-192-168-64-254.cn-northwest-1.compute.internal   Ready    <none>   39m   v1.15.10-eks-bac369
ip-192-168-75-131.cn-northwest-1.compute.internal   Ready    <none>   41m   v1.14.9-eks-1f0ca9
ip-192-168-91-145.cn-northwest-1.compute.internal   Ready    <none>   50s   v1.15.10-eks-bac369
```
查看工作线程节点组

```
eksctl get nodegroups --cluster test
```
参考输出

```
CLUSTER	NODEGROUP	CREATED			MIN SIZE	MAX SIZE	DESIRED CAPACITY	INSTANCE TYPE	IMAGE ID
test	dev		        2020-04-08T11:42:19Z	1		4		2			t3.medium	ami-047594a1f70bab3e2
test	prod-1-15	    2020-04-08T12:23:13Z	1		4		2			t3.medium	ami-039044adc28bd70e9
test	test-nodegroup	2020-04-07T16:16:54Z	2		6		2			t3.micro
```
使用如下命令，删除旧版本节点组

```
eksctl delete nodegroup --cluster test --name dev
```
等待数分钟后，删除成功，再次查看，旧节点组已从列表消失

```
eksctl get nodegroups --cluster test

CLUSTER	NODEGROUP	CREATED			MIN SIZE	MAX SIZE	DESIRED CAPACITY	INSTANCE TYPE	IMAGE ID
test	prod-1-15	    2020-04-08T12:23:13Z	1		4		2			t3.medium	ami-039044adc28bd70e9
test	test-nodegroup	2020-04-07T16:16:54Z	2		6		2			t3.micro
```


