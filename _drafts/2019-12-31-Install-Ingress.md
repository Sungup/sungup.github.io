---
layout: post
title: Install Kubernetes on Raspberry Pi 4
author-id: sungup
#feature-img: "assets/img/posts/2019-12-23-prometheus-and-grafana-banner.jpeg"
tags: [ubuntu, raspberry pi 4, kubernetes, cri-o, container]
date: 2019-12-26 00:00:00
---

## Install Ingress Controller

```shell
wget https://raw.githubusercontent.com/kuberntes/ingress-nginx/master/deploy/static/mandatory.yaml
```

add arm64 at the end of image name

```yaml
      containers:
        - name: nginx-ingress-controller
          image: quay.io/kubernetes-ingress-controller/nginx-ingress-controller-arm64:0.26.2
          args:
            - /nginx-ingress-controller
            - --configmap=$(POD_NAMESPACE)/nginx-configuration
            - --tcp-services-configmap=$(POD_NAMESPACE)/tcp-services
            - --udp-services-configmap=$(POD_NAMESPACE)/udp-services
            - --publish-service=$(POD_NAMESPACE)/ingress-nginx
            - --annotations-prefix=nginx.ingress.kubernetes.io
```

### Use nodeport

after install ingress-mandatory apply the cloud provider specific yaml files.
normal cloud providers support the **load balancer** on there cloud environments,
but, there is no load balancer on-premise environment. so that we must use nodeport
for the ingress' load balancing.

I'll add the images of the concept about Ingress near future.

Anyway, we apply the nodeport service.

```shell
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/baremetal/service-nodeport.yaml;
```

### Use MetalLB

#### Install MetalLB

```shell
wget https://raw.githubusercontent.com/google/metallb/v0.8.3/manifests/metallb.yaml;
```

```diff
--- 01.metallb-origin.yaml 2020-01-04 19:07:04.536753309 +0900
+++ 01.metallb.yaml 2020-01-04 19:05:09.886822710 +0900
@@ -212,7 +212,7 @@
           valueFrom:
             fieldRef:
               fieldPath: status.hostIP
-        image: metallb/speaker:v0.8.2
+        image: docker.io/metallb/speaker:v0.8.2
         imagePullPolicy: IfNotPresent
         name: speaker
         ports:
@@ -268,7 +268,7 @@
       - args:
         - --port=7472
         - --config=config
-        image: metallb/controller:v0.8.2
+        image: docker.io/metallb/controller:v0.8.2
         imagePullPolicy: IfNotPresent
         name: controller
         ports:
```

```bash
kubectl apply -f metallb.yaml;
```

#### Apply MetallLB config map

MetalLB's config map

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: metallb-ip-space
      protocol: layer2
      addresses:
      - 10.0.1.80-10.0.1.99
```

```bash
kubectl apply -f metallb-configmap.yaml;
```

#### Requesting Specific IP

Make ingress service file. you can reference original nodeport configuration file but you should change spec.type to
NodePort to LoadBalancer.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: ingress-nginx
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
spec:
  type: LoadBalancer
  ports:
    - name: http
      port: 80
      targetPort: 80
      protocol: TCP
    - name: https
      port: 443
      targetPort: 443
      protocol: TCP
  selector:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---

```

## For TLS

```shell
kubectl create secret tls tls-ingress --key k8s.sungup.io.key --cert k8s.sungup.io.crt;
kubectl describe secret tls-ingress;
```

## Reference

### Ingress Official

- [Installation Guide]
- [Bare-metal consideration]

### Other Site

- [Hi, can I run ingress under arm64 machines?]
- [쿠버네티스 Ingress 개념 및 사용 방법, 온-프레미스 환경에서 Ingress 구축하기]
- [Kubernetes Metal LB for On-Prem / BareMetal Cluster in 10 minutes]

[Hi, can I run ingress under arm64 machines?]: https://github.com/kubernetes/ingress-nginx/issues/1051
[쿠버네티스 Ingress 개념 및 사용 방법, 온-프레미스 환경에서 Ingress 구축하기]: https://m.blog.naver.com/PostView.nhn?blogId=alice_k106&logNo=221502890249&categoryNo=20&proxyReferer=&proxyReferer=https%3A%2F%2Fwww.google.com%2F
[Installation Guide]: https://kubernetes.github.io/ingress-nginx/deploy/
[Bare-metal considerations]: https://kubernetes.github.io/ingress-nginx/deploy/baremetal/
[Kubernetes Metal LB for On-Prem / BareMetal Cluster in 10 minutes]: https://medium.com/@JockDaRock/kubernetes-metal-lb-for-on-prem-baremetal-cluster-in-10-minutes-c2eaeb3fe813
