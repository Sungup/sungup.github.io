---
layout: post
title: Install Kubernetes on Raspberry Pi 4
author-id: sungup
#feature-img: "assets/img/posts/2019-12-23-prometheus-and-grafana-banner.jpeg"
tags: [ubuntu, raspberry pi 4, kubernetes, cri-o, container]
date: 2019-12-26 00:00:00
---

## Make example using buildah

```shell
# buildah already installed while installing cri-o. But, the path of runc is not
# in /usr/bin and /usr/local/bin
sudo ln -s /usr/lib/cri-o-runc/sbin/runc /usr/local/bin/;
```

```html
<!DOCTYPE html>
<html>
  <head>
    <title>Hello World!</title>
  </head>
  <body>
    <h1>Hello, Kubernetes!</h1>
  </body>
</html>
```

```dockerfile
FROM docker.io/nginx:alpine

MAINTAINER Sungup Moon

COPY index.html /usr/share/nginx/html
```

```shell
sudo buildah build-using-dockerfile -t registry.sungup.io/hello .;
sudo buildah push registry.sungup.io/hello;
```

you can check the container images using following command

```text
ubuntu@rbp4001:~/k8s-books/04.k8s-dashboard$ sudo crictl images;
IMAGE                                TAG                 IMAGE ID            SIZE
docker.io/library/nginx              alpine              c09b2914c3c12       26.7MB
k8s.gcr.io/coredns                   1.6.5               f96217e2532b6       39.5MB
k8s.gcr.io/etcd                      3.4.3-0             ab707b0a0ea33       365MB
k8s.gcr.io/kube-apiserver            v1.17.0             aca151bf3e907       167MB
k8s.gcr.io/kube-controller-manager   v1.17.0             7045158f92f86       158MB
k8s.gcr.io/kube-proxy                v1.17.0             ac19e9cffff5b       116MB
k8s.gcr.io/kube-scheduler            v1.17.0             0d5c120f87f3c       95.4MB
k8s.gcr.io/pause                     3.1                 6cf7c80fe4444       530kB
quay.io/coreos/flannel               v0.11.0-arm64       32ffa9fadfd79       56.3MB
registry.sungup.io/hello             latest              6f4a188d39819       26.7MB
```

after push that image to the private registry.

```shell
sudo crictl rmi ;
```

## Start example service with the new container

Now we start service with the new built image. Make the example service YAML
file `example-service.yaml` and apply that.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world-example
  labels:
    app: hello-world-example
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello-world-example
  template:
    metadata:
      labels:
        app: hello-world-example
    spec:
      containers:
      - name: hello-world-example
        image: registry.sungup.io/hello:latest
        ports:
        - name: http
          containerPort: 80

---

apiVersion: v1
kind: Service
metadata:
  name: hello-world-example
  labels:
    app: hello-world-example
spec:
  ports:
  - name: http
    port: 80
    targetPort: 80
  selector:
    app: hello-world-example
  type: ClusterIP

---

apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: hello-world-example
spec:
  rules:
  - host: "example.sungup.io"
    http:
      paths:
      - path: /
        backend:
          serviceName: hello-world-example
          servicePort: 80
```

```shell
kubectl apply -f example-service.yaml;
```

You can check the example service working fine using `curl` command like this:

```text
ubuntu@rbp4001:~/k8s-books/05.repository$ curl -D- http://example.sungup.io
HTTP/1.1 200 OK
Server: openresty/1.15.8.2
Date: Sun, 19 Jan 2020 04:41:18 GMT
Content-Type: text/html
Content-Length: 133
Connection: keep-alive
Last-Modified: Sun, 19 Jan 2020 02:34:50 GMT
ETag: "5e23c04a-85"
Accept-Ranges: bytes

<!DOCTYPE html>
<html>
  <head>
    <title>Hello World!</title>
  </head>
  <body>
    <h1>Hello, Kubernetes!</h1>
  </body>
</html>
```

## Reference

- [Docker Image를 활용한 Local Registry 구축]

[Docker Image를 활용한 Local Registry 구축]: https://waspro.tistory.com/532
[Volumes]: https://kubernetes.io/docs/concepts/storage/volumes/
[PersistentVolume / PersistentVolumeClaim / StorageClassについて]: https://cstoku.dev/posts/2018/k8sdojo-12/
[Kubernetes Control-Plane Node에 Pod 띄울수 있는 방법 (Taints)]: https://17billion.github.io/kubernetes/2019/04/24/kubernetes_control_plane_working.html

[buildah]:https://github.com/containers/buildah/blob/master/install.md
[Steps To Build Apache Web Server Docker Image]: https://medium.com/@vi1996ash/steps-to-build-apache-web-server-docker-image-1a2f21504a8e
