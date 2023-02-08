---
title: 【ClickHouse系列】使用clickhouse-operator搭建clickhouse集群
author: Chen
date: 2020-03-18 10:25:51 +0800
categories: [博客, 教程]
tags: [ClickHouse]
pin: true
---

### clickhouse-operator特性

- 根据提供的自定义资源规范创建ClickHouse群集
- 自定义存储资源调配（VolumeClaim模板）
- 自定义pod模板
- 自定义service模板
- ClickHouse配置和设置（包括Zookeeper集成）
- 柔性模板
- ClickHouse集群自动扩展
- ClickHouse版本升级
- ClickHouse指标导出Prometheus

要求：Kubernetes 1.12.6+

### 搭建方法

#### 自建一套Kubernetes集群

方法参考：[CentOS7 Kubernetes 1.14.1安装、启动、验证及踩坑](https://blog.csdn.net/weixin_39992480/article/details/90239458)

#### 下载源码

```shell
git clone https://github.com/Altinity/clickhouse-operator.git
```

#### 安装clickhouse-operator

执行`clickhouse-operator/deploy/operator/clickhouse-operator-install.sh`这个脚本

```
[root@k8s-master operator]# pwd
/home/ch/clickhouse-operator/deploy/operator
[root@k8s-master operator]# ./clickhouse-operator-install.sh 
Setup ClickHouse Operator into kube-system namespace
1. Build manifest
2. No need to create kube-system namespace
3. Install operator into kube-system namespace
customresourcedefinition.apiextensions.k8s.io/clickhouseinstallations.clickhouse.altinity.com created
customresourcedefinition.apiextensions.k8s.io/clickhouseinstallationtemplates.clickhouse.altinity.com created
customresourcedefinition.apiextensions.k8s.io/clickhouseoperatorconfigurations.clickhouse.altinity.com created
serviceaccount/clickhouse-operator created
clusterrolebinding.rbac.authorization.k8s.io/clickhouse-operator created
configmap/etc-clickhouse-operator-files created
configmap/etc-clickhouse-operator-confd-files created
configmap/etc-clickhouse-operator-configd-files created
configmap/etc-clickhouse-operator-templatesd-files created
configmap/etc-clickhouse-operator-usersd-files created
deployment.apps/clickhouse-operator created
service/clickhouse-operator-metrics created
```

查看状态，直到`clickhouse-operator-75695fddb8-9rwbc`为`Running`，这样operator即安装成功

```shell
[root@k8s-master chi-examples]# kubectl get pod -A -l app=clickhouse-operator
NAMESPACE     NAME                                   READY   STATUS    RESTARTS   AGE
kube-system   clickhouse-operator-75695fddb8-9rwbc   2/2     Running   2          73m
```

使用官方小例子验证是否operator可用，使用这个例子`01-simple-layout-02-1shard-2repl.yaml`

```shell
[root@k8s-master chi-examples]# pwd
/home/ch/clickhouse-operator/docs/chi-examples
[root@k8s-master chi-examples]# kubectl apply -f 01-simple-layout-02-1shard-2repl.yaml 
clickhouseinstallation.clickhouse.altinity.com/simple-02 created
```

第一次创建会有点慢，因为会取拉取clickhouse-server官方镜像查询详情，会发现`Pulling image "yandex/clickhouse-server:latest"`

```shell
[root@k8s-master chi-examples]# kubectl describe pod chi-simple-02-shard1-repl1-0-0-0
Name:               chi-simple-02-shard1-repl1-0-0-0
Namespace:          default
Priority:           0
PriorityClassName:  <none>
Node:               k8s-node1/192.168.80.134
Start Time:         Tue, 17 Mar 2020 14:57:05 +0800
Labels:             clickhouse.altinity.com/app=chop
                    clickhouse.altinity.com/chi=simple-02
                    clickhouse.altinity.com/chiScopeCycleIndex=0
                    clickhouse.altinity.com/chiScopeCycleOffset=0
                    clickhouse.altinity.com/chiScopeCycleSize=0
                    clickhouse.altinity.com/chiScopeIndex=0
                    clickhouse.altinity.com/chop=0.9.2
                    clickhouse.altinity.com/cluster=shard1-repl1
                    clickhouse.altinity.com/clusterScopeCycleIndex=0
                    clickhouse.altinity.com/clusterScopeCycleOffset=0
                    clickhouse.altinity.com/clusterScopeCycleSize=0
                    clickhouse.altinity.com/clusterScopeIndex=0
                    clickhouse.altinity.com/namespace=default
                    clickhouse.altinity.com/replica=0
                    clickhouse.altinity.com/replicaScopeIndex=0
                    clickhouse.altinity.com/settings-version=73d42865206a76de2371ae93f33a21c234c7dca7
                    clickhouse.altinity.com/shard=0
                    clickhouse.altinity.com/shardScopeIndex=0
                    clickhouse.altinity.com/zookeeper-version=cea9bfc4a19073e068023bb6a2f873b66cd1cb9d
                    controller-revision-hash=chi-simple-02-shard1-repl1-0-0-6695d584b
                    statefulset.kubernetes.io/pod-name=chi-simple-02-shard1-repl1-0-0-0
Annotations:        <none>
Status:             Pending
IP:                 
Controlled By:      StatefulSet/chi-simple-02-shard1-repl1-0-0
Containers:
  clickhouse:
    Container ID:   
    Image:          yandex/clickhouse-server:latest
    Image ID:       
    Ports:          8123/TCP, 9000/TCP, 9009/TCP
    Host Ports:     0/TCP, 0/TCP, 0/TCP
    State:          Waiting
      Reason:       ContainerCreating
    Ready:          False
    Restart Count:  0
    Readiness:      http-get http://:http/ping delay=10s timeout=1s period=10s #success=1 #failure=3
    Environment:    <none>
    Mounts:
      /etc/clickhouse-server/conf.d/ from chi-simple-02-deploy-confd-shard1-repl1-0-0 (rw)
      /etc/clickhouse-server/config.d/ from chi-simple-02-common-configd (rw)
      /etc/clickhouse-server/users.d/ from chi-simple-02-common-usersd (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-pcvlq (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             False 
  ContainersReady   False 
  PodScheduled      True 
Volumes:
  chi-simple-02-common-configd:
    Type:      ConfigMap (a volume populated by a ConfigMap)
    Name:      chi-simple-02-common-configd
    Optional:  false
  chi-simple-02-common-usersd:
    Type:      ConfigMap (a volume populated by a ConfigMap)
    Name:      chi-simple-02-common-usersd
    Optional:  false
  chi-simple-02-deploy-confd-shard1-repl1-0-0:
    Type:      ConfigMap (a volume populated by a ConfigMap)
    Name:      chi-simple-02-deploy-confd-shard1-repl1-0-0
    Optional:  false
  default-token-pcvlq:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-pcvlq
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type    Reason     Age   From                Message
  ----    ------     ----  ----                -------
  Normal  Scheduled  105s  default-scheduler   Successfully assigned default/chi-simple-02-shard1-repl1-0-0-0 to k8s-node1
  Normal  Pulling    104s  kubelet, k8s-node1  Pulling image "yandex/clickhouse-server:latest"
```

直到两个pod实例都变为`Running`，即operator功能正常

```shell
[root@k8s-master chi-examples]# kubectl get pod
NAME                               READY   STATUS    RESTARTS   AGE
chi-simple-02-shard1-repl1-0-0-0   1/1     Running   0          4m31s
chi-simple-02-shard1-repl1-0-1-0   1/1     Running   0          2m22s
```

### clickhouse-operator yaml模板

`Kind:ClickHouseInstallation`：一个自定义的controller，集成了一些自定义模板（如`podTemplates`、`serviceTemplates`、`dataVolumeClaimTemplate`、`logVolumeClaimTemplate`等）及一些自定义参数（如`clusters`、`zookeeper`、`users`、`profiles`等），方便创建ch集群

#### yaml例子

```yaml
apiVersion: "clickhouse.altinity.com/v1"
kind: "ClickHouseInstallation"
metadata:
  # ch集群名称，创建的pod名称例如chi-demo-{clusterName}-0-0-0，{}中下面会说明
  name: "demo"
spec:
  defaults:
    templates:
      # service模板
      serviceTemplate: service-template
      # pod模板，实际上内部使用的是StatefulSet
      podTemplate: pod-template
      # 数据文件pvc模板
      dataVolumeClaimTemplate: volume-claim
      # 日志文件pvc模板
      logVolumeClaimTemplate: volume-claim
  configuration:
    # zk主机名（域名）及端口信息，可配置多个
    zookeeper:
      nodes:
        - host: zookeeper-0
          port: 2181 
    clusters:
      # ch集群的clusterName，名字会填充上面的{clusterName}部分
      - name: "cluster-name"
        layout:
          # ch集群的分片数和每个分片的副本数
          shardsCount: 2
          replicasCount: 2

  templates:
    serviceTemplates:
      - name: service-template
        spec:
          ports:
            - name: http
              port: 8123
            - name: tcp
              port: 9000
          type: LoadBalancer

    podTemplates:
      - name: pod-template
        spec:
          containers:
            - name: clickhouse
              imagePullPolicy: Always
              image: yandex/clickhouse-server:latest
              volumeMounts:
                # 挂载数据文件路径
                - name: volume-claim
                  mountPath: /var/lib/clickhouse
                # 挂载数据文件路径
                - name: volume-claim
                  mountPath: /var/log/clickhouse-server
              resources:
                # 配置cpu和内存大小
                limits:
                  memory: "64Mi"
                  cpu: "0.2"
                requests:
                  memory: "64Mi"
                  cpu: "0.2"

    volumeClaimTemplates:
      - name: volume-claim
        spec:
          accessModes:
            - ReadWriteOnce
          resources:
            # pv的存储大小
            requests:
              storage: 1Gi
```

这里没有配置`StorageClass`，因为operator的一些用法是需要aws支持的，aws的k8s环境一半都会提供default StorageClass，所以不用配置也可以创建成功，如果是自建的Kubernetes集群，是不能成功的，会发现报错

```shell
[root@k8s-master yaml]# kubectl describe pod chi-demo-cluster-name-0-0-0
Name:               chi-demo-cluster-name-0-0-0
......
Events:
  Type     Reason            Age                From               Message
  ----     ------            ----               ----               -------
  Warning  FailedScheduling  79s (x2 over 79s)  default-scheduler  pod has unbound immediate PersistentVolumeClaims (repeated 2 times)
[root@k8s-master yaml]# kubectl get pvc
NAME                                       STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
volume-claim-chi-demo-cluster-name-0-0-0   Pending                                                     2m35s
[root@k8s-master yaml]# kubectl describe pvc volume-claim-chi-demo-cluster-name-0-0-0
Name:          volume-claim-chi-demo-cluster-name-0-0-0
......
Events:
  Type       Reason         Age                 From                         Message
  ----       ------         ----                ----                         -------
  Normal     FailedBinding  4s (x15 over 3m6s)  persistentvolume-controller  no persistent volumes available for this claim and no storage class is set
Mounted By:  chi-demo-cluster-name-0-0-0

```

提示没有可用的持久化卷，其实这里可以自行创建pvc，创建pv来时间pod的挂载。

#### 如何使用StorageClass

这里为了趋向于aws的方式，自行搭建基于nfs的`StorageClass`用于pod挂载，一般类似aws，都会有一个provisioner来提供pvc与pv的动态绑定，所以先要创建provisioner，这里用直接托管给k8s的方式来创建，但是前提是有一套可用的nfs，所以要先找一台机器搭建nfs

使用Deployment方式来创建，但是前提是有一套可用的nfs，所以要先找一台机器搭建nfs

##### 搭建nfs

```shell
yum install -y nfs-utils

#保证相关网段的机器都可以挂载
vim /etc/exports
/data 10.244.0.0/16(rw,sync)
/data 172.17.0.0/16(rw,sync)
/data 192.168.80.0/16(rw,sync)

systemctl enable rpcbind.service
systemctl enable nfs-server.service
systemctl start rpcbind.service
systemctl start nfs-server.service

#使配置生效
exportfs -r

#查看配置
exportfs
/data         	10.244.0.0/16
/data         	172.17.0.0/16
/data         	192.168.80.0/16

```

##### 创建provisioner

`provisioner.yaml`文件如下，包含认证及创建

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: provisioner
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["list", "watch", "create", "update", "patch"]
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["create", "delete", "get", "list", "watch", "patch", "update"]

---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-provisioner
subjects:
  - kind: ServiceAccount
    name: provisioner
    namespace: default
roleRef:
  kind: ClusterRole
  name: provisioner-runner
  apiGroup: rbac.authorization.k8s.io
---
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: provisioner
spec:
  replicas: 1
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: provisioner
    spec:
      serviceAccountName: provisioner
      containers:
        - name: provisioner
          image: xiedeyantu/nfs-client-provisioner
          volumeMounts:
            - name: root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: demo/nfs
            - name: NFS_SERVER
              value: 10.244.1.1
            - name: NFS_PATH
              value: /data
      volumes:
        - name: root
          nfs:
            # server ip是一台搭建了nfs的机器在10.244网段的ip
            server: 10.244.1.1
            path: /data
```

直接创建，直至`provisioner-5749684665-62q2p`为`Running`

```shell
[root@k8s-master yaml]# kubectl apply -f provisioner.yaml 
serviceaccount/provisioner created
clusterrole.rbac.authorization.k8s.io/provisioner-runner created
clusterrolebinding.rbac.authorization.k8s.io/run-provisioner created
deployment.extensions/provisioner created
[root@k8s-master yaml]# kubectl get pod
NAME                           READY   STATUS    RESTARTS   AGE
provisioner-5749684665-62q2p   1/1     Running   0          17m
```

最后就可以创建`StorageClass`了，yaml如下

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: sc-nfs
# 同上面的PROVISIONER_NAME
provisioner: demo/nfs
```

创建并设置默认sc

```shell
[root@k8s-master yaml]# kubectl apply -f sc.yaml 
storageclass.storage.k8s.io/sc-nfs created
[root@k8s-master yaml]# kubectl patch storageclass sc-nfs -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
storageclass.storage.k8s.io/sc-nfs patched
[root@k8s-master yaml]# kubectl get sc
NAME                 PROVISIONER      AGE
sc-nfs (default)     demo/nfs         7s
```

一系列复杂过程结束后，就可以使用operator的那个yaml模板创建了

### ch集群创建

创建集群

```shell
[root@k8s-master yaml]# kubectl apply -f demo.yaml 
clickhouseinstallation.clickhouse.altinity.com/demo created
[root@k8s-master yaml]# kubectl get pod
NAME                           READY   STATUS             RESTARTS   AGE
chi-demo-cluster-name-0-0-0    2/2     Running            0          4m12s
chi-demo-cluster-name-0-1-0    2/2     Running            0          3m7s
chi-demo-cluster-name-1-0-0    2/2     Running            0          3m2s
chi-demo-cluster-name-1-1-0    2/2     Running            0          62s
provisioner-5749684665-62q2p   1/1     Running            0          38m
```

查看service

```shell
root@ubuntu:/opt/code# kubectl get svc 
NAME                    TYPE           CLUSTER-IP       EXTERNAL-IP                    PORT(S)                         AGE
chi-demo-cluster-name-0-0   ClusterIP      None             <none>                         8123/TCP,9000/TCP,9009/TCP      12m
chi-demo-cluster-name-0-1   ClusterIP      None             <none>                         8123/TCP,9000/TCP,9009/TCP      11m
chi-demo-cluster-name-1-0   ClusterIP      None             <none>                         8123/TCP,9000/TCP,9009/TCP      9m
chi-demo-cluster-name-1-1   ClusterIP      None             <none>                         8123/TCP,9000/TCP,9009/TCP      5m
kubernetes                  ClusterIP      10.96.0.1        <none>                         443/TCP                         4h52m
service-demo                LoadBalancer   10.105.81.230    <pending>                      8123:32268/TCP,9000:30966/TCP   2m56s
```

到此可以使用CLUSTER-IP：10.105.81.230登陆ch集群，本地搭建的不支持EXTERNAL-IP
