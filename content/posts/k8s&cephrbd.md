---
title: k8s & rbd
tags:
- k8s
date: 2020-10-02 01:09:00
---

1. 创建 secret

```
kubectl create secret generic ceph-secret --type="kubernetes.io/rbd" \
  --from-literal=key='AQDbfnVfkbqeLxAAiPhSJvt1hVZrM9ntL7JGNQ==' \
  --namespace=kube-system
```

2. 创建存储类

```shell
cat > storage_class.yml <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rbd
  annotations:
    "storageclass.kubernetes.io/is-default-class": "true"
provisioner: kubernetes.io/rbd
parameters:
  monitors: 192.168.1.31:6789,192.168.1.32:6789,192.168.1.33:6789
  adminId: admin
  adminSecretName: ceph-secret
  adminSecretNamespace: kube-system
  pool: rbd-k8s1
  userId: admin
  userSecretName: ceph-secret
  userSecretNamespace: kube-system
  fsType: ext4
  imageFormat: "2"
  imageFeatures: "layering"
EOF

kubectl apply -f storage_class.yml
```

3. 创建pvc
4. 应用绑定pvc
