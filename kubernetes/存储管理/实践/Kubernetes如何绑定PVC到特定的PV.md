# Kubernetes如何绑定PVC到特定的PV



正常模式下， PVC会随机选择满足条件的PV, 可使用如下两种常用方式将PVC绑定到特定的PV上。



## 在PVC中直接指定PV的名称

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-bind-with-pv-name
  namespace: foo
spec:
  storageClassName: "xxxxx"            // 需要与PV一样。或者都为空
  volumeName: "pv-name-select-by-hand"
  accessModes:
    - ReadWriteOnce          
  resources:
    requests:
      storage: 100Mi
```

> 注意：accessModes、resources需要与PV匹配。storageClassName可以都不设置，如果设置了，但要相同。

如果其他PersistentVolumeClaims可以使用您指定的PV，则首先需要保留该存储卷。claimRef在PV字段中指定相关的PersistentVolumeClaim，以便其他PVC无法与其绑定。

```
kind: PersistentVolume
apiVersion: v1
metadata:
  name: pv-name-select-by-hand
spec:
  storageClassName: "xxxxx" 
  capacity:
    storage: 100Mi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/data/pv-name-select-by-hand"
  claimRef:
    name: pvc-bind-with-pv-name
    namespace: foo
```



## 在PVC中指定特定PV的标签

```
// 创建带有标签的PV
kind: PersistentVolume
apiVersion: v1
metadata:
  name: pv-with-labels
  labels:
    app: pv-with-labels
spec:
  capacity:
    storage: 100Mi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/data/pv-with-labels"
    
// 创建PVC, 使用matchLabels关联特定标签的PV
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-bind-with-pv-labels
  namespace: foo
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Mi
  selector:
    matchLabels:
      app: pv-with-labels
```

