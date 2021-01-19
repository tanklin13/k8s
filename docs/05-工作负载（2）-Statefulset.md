# 工作负载（2）-Statefulset

除了无状态应用（Deployment）外，实践中还存在另一种不那么常见的应用：有状态应用。有状态应用通常有如下特点：

* 需要固定的网络标识符（通过Headless Service实现）。例如，Redis集群里面的多个Redis实例，相互之间需要固定的IP或hostname等网络标识进行访问。
* 需要稳定的存储（通过PV/PVC实现）。例如，Zookeeper集群中各个实例，需要单独且固定的存储。
* 实例可按固定顺序创建、删除、滚动升级（Statefulset部署的特性）

有状态应用对应K8s工作负载是Statefulset。对于Statefulset，“**固定的网络标识符**”是通过Headless Service和集群DNS机制实现；而“**稳定的存储**”是通过PV/PVC/VolumeClaimTemplates等机制实现。

Statefulset具有上述特点，使其尤其适合用来部署存储类、集群类应用。例如：

- [MongoDB](https://kubernetes.io/blog/2017/01/running-mongodb-on-kubernetes-with-statefulsets)
- [zookeeper部署示例1](https://jimmysong.io/kubernetes-handbook/guide/using-statefulset.html)、[zookeeper部署示例
2:Running ZooKeeper, A CP Distributed System](https://kubernetes.io/docs/tutorials/stateful-application/zookeeper/)
- [kafka](https://jimmysong.io/kubernetes-handbook/guide/using-statefulset.html)
- [Cassandra](https://kubernetes.io/docs/tutorials/stateful-application/cassandra/)
- [MySQL](https://kubernetes.io/docs/tasks/run-application/run-replicated-stateful-application/)
- [Redis](https://github.com/CommercialTribe/kube-redis)



### 1. Headless Service

Headless Service是一种特殊的K8s Service。从逻辑上，普通Service相当于反向代理，发给普通Service的流量会转发给Service后端的pod；而Headless Service仅提供类似DNS的功能，返回后端pod IP。

在YAML配置上，Headless Service与普通Service的区别在于，Headless Service的spec.clusterIP必须设置为``None``。

尝试创建如下Headless Service：

```
apiVersion: v1
kind: Service
metadata:
  annotations:
  labels:
    app: eureka
  name: eureka-headless
  namespace: pg-allen-ali   #【修改为各自namespace】
spec:
  clusterIP: None
  ports:
  - port: 8761
    protocol: TCP
    targetPort: 8761
  selector:
    app: eureka
  sessionAffinity: None
  type: ClusterIP
```

查看Headless Service信息：

```
# kubectl get svc eureka-headless -n pg-allen-ali 
NAME              TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
eureka-headless   ClusterIP   None         <none>        8761/TCP   150m
```



### 2. PV/PVC

Kubernetes存储资源控制的核心是PV（PersistentVolume）与PVC（PersistentVolumeClaim）。可参考[Kubernetes官方PV、PVC的说明文档](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) 来理解这两个概念。

PV是集群中与节点同级的资源，区别在于：节点是集群的计算资源，而PV是集群的存储资源。集群管理员预先部署好存储系统（例如NFS、CephFS等），然后定义一个PV即可将存储资源置于Kubernetes管理之下。

PVC与memory/cpu request有相似之处，区别在于：memory/cpu request是容器对计算资源需求的声明，而PVC是容器对存储资源需求的声明。

PV是集群层面的概念，与节点同级。PV属于集群，但不属于任何Namespace。 而PVC属于特定的Namespace，只能被同一个Namespace下的Pod所使用（可多个Pod共用一个PVC）。



### 3. Statefulset

创建如下statefulset

```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  labels:
    app: eureka
  name: eureka
  namespace: pg-allen-ali   #【修改为各自namespace】
spec:
  podManagementPolicy: Parallel
  replicas: 2
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: eureka
  serviceName: eureka-headless    # headless service名
  template:
    metadata:
      labels:
        app: eureka
        project: k8s-playground
    spec:
      containers:
      - env:
        - name: EUREKA_CLIENT_REGISTERWITHEUREKA
          value: "true"
        - name: EUREKA_CLIENT_FETCHREGISTRY
          value: "true"
        - name: replicas
          value: "2"
        - name: EUREKA_NAMESPACE
          value: pg-allen-ali    # 【修改为各自namespace】
        image: registry-vpc.ap-southeast-1.aliyuncs.com/allen-mo/proj.k8s-playground_app.eureka:20191227_01
        imagePullPolicy: Always
        livenessProbe:
          failureThreshold: 3
          initialDelaySeconds: 168
          periodSeconds: 20
          successThreshold: 1
          tcpSocket:
            port: 8761
          timeoutSeconds: 1
        name: eureka
        readinessProbe:
          failureThreshold: 3
          httpGet:
            path: /actuator/health
            port: 8761
            scheme: HTTP
          initialDelaySeconds: 11
          periodSeconds: 20
          successThreshold: 1
          timeoutSeconds: 1
        resources:
          limits:
            cpu: 500m
            memory: 768Mi
          requests:
            cpu: 100m
            memory: 512Mi
        volumeMounts:     # volume挂载
        - mountPath: /dianyi/data
          name: eureka-data
      dnsPolicy: ClusterFirst
      restartPolicy: Always
  updateStrategy:
    rollingUpdate:
      partition: 0      # 滚动升级的分区：分批进行，先升级序号>partition
    type: RollingUpdate
  volumeClaimTemplates:       # 动态PV声明
  - metadata:
      name: eureka-data
      namespace: pg-allen-ali
    spec:
      accessModes:
      - ReadWriteOnce
      - ReadOnlyMany
      - ReadWriteMany
      storageClassName: alicloud-nas
      resources:
        requests:
          storage: 1Gi
      volumeMode: Filesystem
```

若statefulset的pod未能正常启动，则可查看statefulset状态：

```
# kubectl -n pg-allen-ali describe statefulset eureka
```

查看pod状态：

```
# kubectl -n pg-allen-ali get po
```



**查看PVC**

```
# kubectl -n pg-allen-ali get pvc
```

PVC状态是否为``Bound``状态？



**查看PV**

```
# kubectl get pv
```

PV状态是否为``Bound``状态？



为了在集群外访问eureka服务，我们可以再搞一个普通Service：

```
apiVersion: v1
kind: Service
metadata:
  annotations:
    service.beta.kubernetes.io/alicloud-loadbalancer-address-type: intranet
  labels:
    app: eureka
  name: eureka
  namespace: pg-allen-ali   #【改成各自Service】
spec:
  externalTrafficPolicy: Cluster
  ports:
  - port: 8761
    protocol: TCP
    targetPort: 8761
  selector:
    app: eureka
  sessionAffinity: None
  type: LoadBalancer
```

稍等片刻，即可通过SLB地址访问Eureka。



进入``eureka-0``这个pod：

```
# kubectl -n pg-allen-ali exec -it eureka-0 bash
# 查看挂载的volume
# df -h 
Filesystem                                                                                                                    Size  Used Avail Use% Mounted on
7fcde4a719-bsa48.ap-southeast-1.nas.aliyuncs.com:/pg-allen-ali-eureka-data-eureka-0-pvc-6c4b31b9-7fa3-11ea-b7ec-00163e02d9d6   10P  166M   10P   1% /dianyi/data
... ...
# 往数据盘写个测试信息
# echo "in eureka-0" > /dianyi/data/data_test
# cat /dianyi/data/data_test 
in eureka-0

# 尝试通过headless service访问另一个eureka实例
# ping eureka-1.eureka-headless
PING eureka-1.eureka-headless.pg-allen-ali.svc.cluster.local (172.20.27.72) 56(84) bytes of data.
64 bytes from 172-20-27-72.eureka.pg-allen-ali.svc.cluster.local (172.20.27.72): icmp_seq=1 ttl=62 time=1.03 ms
64 bytes from 172-20-27-72.eureka.pg-allen-ali.svc.cluster.local (172.20.27.72): icmp_seq=2 ttl=62 time=0.983 ms
^C
```

如有时间，还可进入``eureka-1``中执行类似的操作。






Next：[06-工作负载（3）-Daemonset](06-工作负载（3）-Daemonset.md)

