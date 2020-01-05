---
layout: post
title: Install Kubernetes on Raspberry Pi 4
author-id: sungup
#feature-img: "assets/img/posts/2019-12-23-prometheus-and-grafana-banner.jpeg"
tags: [ubuntu, raspberry pi 4, kubernetes, cri-o, container]
date: 2019-12-26 00:00:00
---

## Install

```shell
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/master/aio/deploy/recommended.yaml;
```

latest yaml receipe use metrics-scraper v1.0.1, but that version has some errors on arm64. Change the version tag to v1.0.2;

```diff
--- k8s-dashboard-origin.yaml 2019-12-30 00:21:50.261535225 +0900
+++ k8s-dashboard.yaml 2019-12-30 00:21:31.909853184 +0900
@@ -187,7 +187,7 @@
     spec:
       containers:
         - name: kubernetes-dashboard
-          image: kubernetesui/dashboard:v2.0.0-beta8
+          image: docker.io/kubernetesui/dashboard:v2.0.0-beta8
           imagePullPolicy: Always
           ports:
             - containerPort: 8443
@@ -271,7 +271,7 @@
     spec:
       containers:
         - name: dashboard-metrics-scraper
-          image: kubernetesui/metrics-scraper:v1.0.1
+          image: docker.io/kubernetesui/metrics-scraper:v1.0.2
           ports:
             - containerPort: 8000
               protocol: TCP

```

## Ingress Add

To convert HTTPS to HTTP on Ingress, the certification file must be registered.

```shell
kubectl -n kube-system create secret tls tls-ingress --key k8s.sungup.io.key --cert k8s.sungup.io.crt;
```

and Make ingress resource and apply that

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: kubernetes-dashboard-ingress
  namespace: kubernetes-dashboard
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
spec:
  tls:
  - hosts:
    - "k8s.sungup.io"
    secretName: tls-ingress
  rules:
  - host: "k8s.sungup.io"
    http:
      paths:
      - path: /
        backend:
          serviceName: kubernetes-dashboard
          servicePort: 443
```

## Create Account

### Create service account

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
```

### Create ClusterRoldBinding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
```

### Retreaving Token

```shell
kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}');
```

## Reference

[Creating sample user]: https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/creating-sample-user.md
[Web UI (Dashboard)]: https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/
[kubernetes-metrics-scraper on ARM on 2.0beta1: Unable to initialize database tables: Binary was compiled with 'CGO_ENABLED=0]: https://github.com/kubernetes/dashboard/issues/4029
[Deploy the Kubernetes Web UI Dashboard]: https://xuri.me/2019/01/23/deploy-the-kubernetes-web-ui-dashboard.html

[Kubernetes Dashboard에 Ingress를 통해 접속하기]: https://medium.com/@essem_dev/kubernetes-dashboard에-ingress를-통해-접속하기-d78c0d5bc615
[Creating sample user]: https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/creating-sample-user.md
