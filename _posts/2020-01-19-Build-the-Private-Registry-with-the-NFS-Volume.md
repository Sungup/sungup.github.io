---
layout: post
title: Build the Private Registry with the NFS Volume
author-id: sungup
#feature-img: "assets/img/posts/2019-12-23-prometheus-and-grafana-banner.jpeg"
tags: [ubuntu, raspberry pi 4, kubernetes, registry]
date: 2020-01-19 00:00:00
---

Now we build the private registry. The private registry is very helpful, if you
run the kubernetes cluster in the dedicated network or test your built
micro-service before publishing. But, the default deploy config of the
docker-registry uses **emptyDir** and it cannot keep the registry volumes. To
keep the uploaded images permanently, the image data should be stored on the
persistent volumes like Ceph, NFS, iSCSI, and etc...

In this article, setup the PV *(persistent volume)* and PVC *(persistent volume
claim)* and attatch that volumes to the registry service.

- *[How to Setup the NFS on Ubuntu]*
- **[Build the Private Registry with the NFS Volume]**
- *Build and push image in the private registry*

## Make PV and PVC for registry

PV*(persistent volume)* is a piece of storage in the cluster. That has been
provisioned by the **Administrator** or dynamically using Storage Classes. It
means PV is also a cluster resource like a **node**, so that PVs have a
lifecycle independent of any Pod that uses the PV.

PVC*(persistent volume claim)* is a request for storage by a cluster user. Like
Pod is a request about the CPU resource from a node, PVC is a request about the
PV resource with specific size and access modes.

Anyway, we will define the PV and allocate the PVC for the registry pods. To
distinquish it from other services, we use a specific namespace of
**kubernetes-registry**. Also, we set the size of the **pv-registry** PV to
100GiB and assign all of space of the **pv-registry** to the **pvc-registry**
PVC. These PV and PVC will be used by the private registry only, so we set the
**accessModes** with **ReadWriteOnce**. Make `registry-volume.yaml` file and
save with following configurations.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-registry
  labels:
    k8s-app: kubernetes-registry
spec:
  capacity:
    storage: 100Gi
  accessModes:
  - ReadWriteOnce
  nfs:
    path: /vol/nfs/registry
    server: {NFS domain or IP address}
  persistentVolumeReclaimPolicy: Retain
  storageClassName: sc-registry

---

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
  resources:
    requests:
      storage: 100Gi
  storageClassName: sc-registry
```

Run following command to apply PV and PVC.

```shell
kubectl apply -f registry-volume.yaml;
```

You can check the build PV and PVC using `kubectl get pv,pvc` like this.

```text
$ kubectl get pv
NAME          CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                              STORAGECLASS   REASON   AGE
pv-registry   100Gi      RWO            Retain           Bound    kubernetes-registry/pvc-registry   sc-registry             4d7h
$ kubectl get pvc -n kubernetes-registry
NAME           STATUS   VOLUME        CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pvc-registry   Bound    pv-registry   100Gi      RWO            sc-registry    4d7h
```

## Define deployment and service

Now, we define the deployment and service for the registry pods. I has
referenced a document **[Docker Registry]** article in the example of **NGINX
Ingress Controller**. The default values of `REGISTRY_HTTP_ADDR` and
`REGISTRY_STORAGE_FILESYSTEM_ROOTDIRECTORY` are `/var/lib/registry` and
`:5000`, so you can erase the **env** field of follwing deployment section.
But, you should sync the port number between and `REGISTRY_HTTP_ADDR`
**spec.template.spec.containers.ports.containerPort**.

To mount the PVC to pods, define **spec.template.spec.volumes** with the PVC's
**claimName**. K8s will mount the volume, which has claim name with
**pvc-registry**, to the defined path at
**spec.template.spec.containers.volumeMounts.mountPath**.

Default docker-registry image uses 5000 port with http protocol so we define
service with default 5000 port of http.

Also make `registry-service.yaml` file, save following contents and apply the
YAML file to make registry Pod.

```yaml
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
          image: docker.io/registry:latest
          env:
            - name: REGISTRY_HTTP_ADDR
              value: ":5000"
            - name: REGISTRY_STORAGE_FILESYSTEM_ROOTDIRECTORY
              value: "/var/lib/registry"
          ports:
          - name: http
            containerPort: 5000
          volumeMounts:
          - name: vol-registry
            mountPath: "/var/lib/registry"
      volumes:
        - name: vol-registry
          persistentVolumeClaim:
            claimName: pvc-registry

---

apiVersion: v1
kind: Service
metadata:
  name: kubernetes-registry
  namespace: kubernetes-registry
  labels:
    k8s-app: kubernetes-registry
spec:
  ports:
  - name: http
    port: 5000
    targetPort: 5000
  selector:
    k8s-app: kubernetes-registry
  type: ClusterIP
```

```shell
kubectl apply -f registry-service.yaml;
```

## Export registry service through ingress

At the final step, we export the registry service through the ingress service.
Before applying the ingress config, we should make the self-signed cert file
and register that to the **kubernetes-registry** namespace. Please read the
following articles for the detail procedure.

- [Creating Self-Signed Certification for Local HTTPS Environment]:
- [Installing Ingress-Nginx on the Private Network]:

If you generate the cert key and file successfully, apply that in the namespace
using following command.

```shell
kubectl -n kubernetes-registry create secret tls {registry tls secret} --key {registry key} --cert {registry cert file}
```

Also, make `registry-ingress.yaml` file, save with following conents, and apply
that YAML file.

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: kubernetes-registry
  namespace: kubernetes-registry
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "0"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "600"
    kubernetes.io/tls-acme: "true"
spec:
  tls:
  - hosts:
    - "{registry domain or ip}"
    secretName: {registry tls secret}
  rules:
  - host: "{registry domain or ip}"
    http:
      paths:
      - path: /
        backend:
          serviceName: kubernetes-registry
          servicePort: 5000
```

```shell
kubectl apply -f registry-ingress.yaml;
```

## Reference

### Kubernetes Documents

- [Volumes]
- [Persistent Volumes]

### Other references

- [Docker Image를 활용한 Local Registry 구축]
- [PersistentVolume / PersistentVolumeClaim / StorageClassについて]
- [Kubernetes Control-Plane Node에 Pod 띄울수 있는 방법 (Taints)]
- [Docker Registry]

[Volumes]: https://kubernetes.io/docs/concepts/storage/volumes/
[Persistent Volumes]: https://kubernetes.io/docs/concepts/storage/persistent-volumes/
[Docker Image를 활용한 Local Registry 구축]: https://waspro.tistory.com/532
[PersistentVolume / PersistentVolumeClaim / StorageClassについて]: https://cstoku.dev/posts/2018/k8sdojo-12/
[Kubernetes Control-Plane Node에 Pod 띄울수 있는 방법 (Taints)]: https://17billion.github.io/kubernetes/2019/04/24/kubernetes_control_plane_working.html
[Docker Registry]: https://kubernetes.github.io/ingress-nginx/examples/docker-registry/

[How to Setup the NFS on Ubuntu]: /2020/01/15/How-to-Setup-the-NFS-on-Ubuntu.html
[Build the Private Registry with the NFS Volume]: /2020/01/19/Build-the-Private-Registry-with-the-NFS-Volume.html

[Creating Self-Signed Certification for Local HTTPS Environment]: /2020/01/06/Creating-Self-Signed-Certification-for-Local-HTTPS-Environment.html
[Installing Ingress-Nginx on the Private Network]: /2020/01/07/Installing-Ingress-Nginx-on-the-Private-Network.html
