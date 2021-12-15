# 浅谈Kubernetes PV的回收策略



## PV回收策略种类

当用户不再使用存储卷时，可以将 PVC 对象删除，允许该资源对应的卷被回收再利用。 目前，数据卷包含三种回收策略： Retained（保留）、Recycled（回收）或 Deleted（删除）。

### 保留（Retain）

回收策略Retain使得用户可以手动回收资源。当 PersistentVolumeClaim 对象被删除时，PersistentVolume 卷仍然存在，对应的数据卷被视为"已释放（released）"。 由于卷上仍然存在着前一申领人的数据，该卷还不能用于其他申领。 如果需要彻底回收，需要额外步骤来手动回收该卷。

> 手动创建的PV默认使用该回收策略。

### 删除（Delete）

对于支持Delete回收策略的卷插件，删除动作会将 PersistentVolume 对象从 Kubernetes 中移除，同时也会从外部基础设施（如 AWS EBS、GCE PD、Azure Disk 或 Cinder 卷）中移除所关联的存储资产。 动态供应的卷会继承其 StorageClass 中设置的回收策略，该策略默认为Delete。 管理员需要根据用户的期望来配置 StorageClass；否则存储卷被创建之后必须可以修改回收策略。

> 动态创建的PV默认使用该回收策略。
>
> 级联删除PV时候，需要插件支持。

### 回收（Recycle）

静态PV模式下，删除PVC后， PV对应NAS里内容会删除。 

> NOTES: 回收策略Recycle已被废弃。建议方案是使用动态存储。
>



## 配置PV回收策略

### PV中定义回收策略

```
kind: PersistentVolume
apiVersion: v1
metadata:
  name: pv-with-recycle-policy
  labels:
    app: pv-with-recycle-policy
spec:
  capacity:
    storage: 100Mi
  volumeMode: Filesystem
  persistentVolumeReclaimPolicy: Retain
  accessModes:
    - ReadWriteOnce
  storageClassName: pv-with-recycle-policy
  mountOptions:
    - hard
    - nfsvers=4.1
  nfs:
    path: /tmp
    server: xx.xx.xx.xx
```

### StorageClass中定义回收策略

```
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: pv-recycle-retain
provisioner: kubernetes.io/xxxxx
parameters:
  type: gp2
  fsType: ext4 
reclaimPolicy: Retain
```

## 修改pv回收策略

对于动态配置的PersistentVolumes来说，默认回收策略为 “Delete”。这表示当用户删除对应的PersistentVolumeClaim时，动态配置的 volume 将被自动删除。如果 volume 包含重要数据时，这种自动行为可能是不合适的。这种情况下，更适合使用 “Retain” 策略。使用 “Retain” 时，如果用户删除PersistentVolumeClaim，对应的PersistentVolume不会被删除。相反，它将变为Released状态，表示所有的数据可以被手动恢复。

使用如下命令可以修改pv回收策略：

```
$ kubectl patch pv <your-pv-name> -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'
```

## 场景操作

下面以NFS存储为例演示pv回收策略及如何复用被保留的pv卷。

### 测试资源准备

1、创建回收策略为Retain的静态pv

```
# static-pv-with-retain-policy.yaml
kind: PersistentVolume
apiVersion: v1
metadata:
  name: pv-with-retain-policy
  labels:
    app: pv-with-retain-policy
spec:
  capacity:
    storage: 100Mi
  volumeMode: Filesystem
  persistentVolumeReclaimPolicy: Retain
  accessModes:
    - ReadWriteOnce
  storageClassName: pv-with-retain-policy
  mountOptions:
    - hard
    - nfsvers=4.1
  nfs:
    path: /tmp
    server: xx.xx.xx.xx
```

查看创建的pv，该pv状态为Available ，可用于pvc申请：

```
$ kubectl apply -f static-pv-with-retain-policy.yaml

$ kubectl get pv
NAME                     CAPACITY   ACCESS MODES   RECLAIM POLICY  STATUS     CLAIM   STORAGECLASS     REASON        AGE
pv-with-retain-policy    100Mi      RWO            Retain          Available          pv-with-retain-policy
              2s
```

2、创建pvc绑定到pv

```
# pvc-bind-to-pv-randomly.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-bind-to-pv-randomly
  namespace: default
spec:
  storageClassName: pv-with-retain-policy
  accessModes:
    - ReadWriteOnce          
  resources:
    requests:
      storage: 100Mi
```


查看创建的pvc，此时pvc已经随机获取到一个可用pv并与其绑定，查看pvc及pv都为Bound状态

```
$ kubectl apply -f pvc-bind-to-pv-randomly.yaml 

$ kubectl -n default get pvc
NAME                      STATUS   VOLUME                  CAPACITY   ACCESS MODES   STORAGECLASS         AGE
pvc-bind-to-pv-randomly   Bound    pv-with-retain-policy   100Mi      RWO            pv-with-retain-policy   3s

$ kubectl get pv
NAME                     CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                           STORAGECLASS                REASON        AGE
pv-with-retain-policy    100Mi      RWO            Retain           Bound    default/pvc-bind-to-pv-randomly pv-with-retain-policy                     2m58s

```

3、创建pod申请pvc

```
# pod-claim-to-pvc.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-claim-to-pvc
  labels:
    app: nginx 
spec:
  containers:
    - name: pod-claim-to-pvc
      image: nginx
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: pod-claim-to-pvc
  volumes:
    - name: pod-claim-to-pvc
      persistentVolumeClaim:
        claimName: pvc-bind-to-pv-randomly
```

查看创建的pod：

```
$ kubectl get pod
NAME               READY   STATUS    RESTARTS   AGE
pod-claim-to-pvc   1/1     Running   0          84s
```

向pod内写入测试数据，并登录到NFS Server查看持久化数据：

```
$ kubectl exec pod-claim-to-pvc -- sh -c "echo 'Hello, i'm pod-claim-to-pvc' > /usr/share/nginx/html/index.html"

$ cat /tmp/index.html 
Hello, i'm pod-claim-to-pvc

$ curl http://<pod-claim-to-pvc>:80
Hello, i'm pod-claim-to-pvc
```

### kubernetes pv回收策略

1、删除pod，该pvc及pv依然保留并处于绑定状态，因此删除pod对pvc及pv没有影响：

```
$ kubectl delete pods pod-claim-to-pvc 

$ kubectl -n foo get pvc
NAME                      STATUS   VOLUME                  CAPACITY   ACCESS MODES   STORAGECLASS         AGE
pvc-bind-to-pv-randomly   Bound    pv-with-retain-policy   100Mi      RWO            pv-with-retain-policy   3s

$ kubectl get pv
NAME                     CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                           STORAGECLASS                REASON        AGE
pv-with-retain-policy    100Mi      RWO            Retain           Bound    default/pvc-bind-to-pv-randomly pv-with-retain-policy                     2m58s

```

2、继续删除pvc，此时pv是否被删除取决于其回收策略。

如果回收策略为Delete，则pv也会被自动删除；回收策略为Retain，则pv会被保留

```
$ kubectl -n default delete pvc pvc-bind-to-pv-randomly

$ kubectl get pv
NAME                    CAPACITY   ACCESS MODES   RECLAIM POLICY  STATUS     CLAIM                       STORAGECLASS                REASON        AGE
pv-with-retain-policy   100Mi      RWO            Retain          Released   default/pvc-bind-to-pv-randomly pv-with-retain-policy                     2m58s
```

可以看到pv依然保留，并且状态由Bound变为Released，该状态pv无法被新的pvc申领，只有PV处于Available状态时才能被pvc申领。

查看PV的属性，被保留下来的PV在spec.claimRef字段记录着原来PVC的绑定信息:

```
$ kubectl get pv pv-with-retain-policy -o yaml
apiVersion: v1
kind: PersistentVolume
......
spec:
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 100Mi
  claimRef:
    apiVersion: v1
    kind: PersistentVolumeClaim
    name: pvc-bind-to-pv-randomly
    namespace: default
    resourceVersion: "792426"
    uid: 325b013f-0467-4078-88e9-0fc208c6993c
  nfs:
    path: /tmp
    server: 10.0.0.5
  persistentVolumeReclaimPolicy: Retain
  storageClassName: pv-with-retain-policy
  volumeMode: Filesystem
status:
  phase: Released
```

删除绑定信息中的resourceVersion和uid键，即可重新释放PV使其状态由Released变为Available

```
$ kubectl edit pv pv-with-retain-policy
......
spec:
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 100Mi
  claimRef:
    apiVersion: v1
    kind: PersistentVolumeClaim
    name: pvc-bind-to-pv-randomly
    namespace: default
......
```

再次查看pv，状态已经变为Available

```
$ kubectl get pv
NAME                     CAPACITY   ACCESS MODES   RECLAIM POLICY  STATUS     CLAIM   STORAGECLASS     REASON        AGE
pv-with-retain-policy    100Mi      RWO            Retain          Available          pv-with-retain-policy
              2m
```

由于该pv中spec.claimRef.name和spec.claimRef.namespace键不变，该pv依然指向原来的pvc名称，具有相应spec的新PVC能够准确绑定到该PV。现在， 我们重新恢复之前删除的pvc及pod：

```
$ kubectl apply -f pvc-bind-to-pv-randomly.yaml

$ kubectl apply -f pod-claim-to-pvc.yaml
```

查看pvc及pod状态

```
$ kubectl -n default get pvc
NAME                      STATUS   VOLUME                  CAPACITY   ACCESS MODES   STORAGECLASS         AGE
pvc-bind-to-pv-randomly   Bound    pv-with-retain-policy   100Mi      RWO            pv-with-retain-policy   3s

$ kubectl get pv
NAME                     CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                           STORAGECLASS                REASON        AGE
pv-with-retain-policy    100Mi      RWO            Retain           Bound    default/pvc-bind-to-pv-randomly pv-with-retain-policy                     2m58s

$ kubectl get pod
NAME               READY   STATUS    RESTARTS   AGE
pod-claim-to-pvc   1/1     Running   0          84s

$ curl http://<pod-claim-to-pvc>:80
Hello, i'm pod-claim-to-pvc
```

之前写入的数据依然存在，数据成功恢复。 