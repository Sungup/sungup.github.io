---
layout: post
title: Install Kubernetes on Raspberry Pi 4
author-id: sungup
#feature-img: "assets/img/posts/2019-12-23-prometheus-and-grafana-banner.jpeg"
tags: [ubuntu, raspberry pi 4, kubernetes, cri-o, container]
date: 2019-12-26 00:00:00
---

## Pull private registry image

```bash
sudo crictl pull docker.io/registry:latest;
```

## Make PV volume

Persistent Volume Claim

Run registry with VolumeClaim

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: kubernetes-registry

---

apiVersion: v1
kind: ServiceAccount
metadata:
  name: kubernetes-registry
  namespace: kubernetes-registry
  labels:
    k8s-app: kubernetes-registry
```

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-registry
  namespace: kubernetes-registry
  labels:
    k8s-app: kubernetes-registry
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 200Gi
  storageClassName: sc-registry

---

apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-registry
  namespace: kubernetes-registry
  labels:
    k8s-app: kubernetes-registry
spec:
  capacity:
    storage: 200Gi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: sc-registry
  local:
    path: /vol/registry/data
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - rbp4001.sungup.io
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kubernetes-registry
  namespace: kubernetes-registry
  labels:
    k8s-app: kubernetes-registry
spec:
  ports:
    - port: 5000
      protocol: TCP
      name: http
  selector:
    k8s-app: kubernetes-registry
  type: ClusterIP

---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: kubernetes-registry
  namespace: kubernetes-registry
  labels:
    k8s-app: kubernetes-registry
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: kubernetes-registry
  template:
    metadata:
      labels:
        k8s-app: kubernetes-registry
    spec:
      containers:
      - name: kubernetes-registry
        image: docker.io/nginx:alpine
        imagePullPolicy: Always
        ports:
        - containerPort: 5000
        volumeMounts:
        - name: vol-registry
          mountPath: /var/lib/registry/docker/registry/v2
      volumes:
      - name: vol-registry
        persistentVolumeClaim:
          claimName: pvc-registry
      nodeSelector:
        "kubernetes.io/hostname": "rbp4001.sungup.io"

---

```

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: kubernetes-registry-ingress
  namespace: kubernetes-registry
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/backend-protocol: "HTTP"
spec:
  tls:
  - hosts:
    - "registry.sungup.io"
    secretName: tls.registry.sungup.io
  rules:
  - host: "registry.sungup.io"
    http:
      paths:
      - path: /
        backend:
          serviceName: kubernetes-registry
          servicePort: 5000
```

## Run on master (Optional)

```shell
kubectl taint nodes rbp4001.sungup.io node-role.kubernetes.io/master-;

kubectl apply -f registry;

kubectl taint nodes rbp4001.sungup.io node-role.kubernetes.io/master=:NoSchedule
```

## Apply cert to registry

```shell
kubectl -n kubernetes-registry create secret tls tls.registry.sungup.io --key registry.sungup.io.key --cert registry.sungup.io.crt
```

## Make example using buildah

```shell
# buildah already installed while installing cri-o. But, the path of runc is not
# in /usr/bin and /usr/local/bin
sudo ln -s /usr/lib/cri-o-runc/sbin/runc /usr/local/bin/;
```

```html
```

```dockerfile
```

```shell
sudo buildah build-using-dockerfile -t registry.sungup.io/helloworld .
```

## Build Web UI

Clone web ui repository from `https://github.com/Joxit/docker-registry-ui.git`

```bash
sudo crictl pull docker.io/nginx:alpine
```

## Reference

- [Docker Image를 활용한 Local Registry 구축]

[Docker Image를 활용한 Local Registry 구축]: https://waspro.tistory.com/532
[Volumes]: https://kubernetes.io/docs/concepts/storage/volumes/
[PersistentVolume / PersistentVolumeClaim / StorageClassについて]: https://cstoku.dev/posts/2018/k8sdojo-12/
[Kubernetes Control-Plane Node에 Pod 띄울수 있는 방법 (Taints)]: https://17billion.github.io/kubernetes/2019/04/24/kubernetes_control_plane_working.html

[buildah]:https://github.com/containers/buildah/blob/master/install.md
[Steps To Build Apache Web Server Docker Image]: https://medium.com/@vi1996ash/steps-to-build-apache-web-server-docker-image-1a2f21504a8e
