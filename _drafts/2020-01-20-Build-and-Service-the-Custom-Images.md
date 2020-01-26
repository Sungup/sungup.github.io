---
layout: post
title: Install Kubernetes on Raspberry Pi 4
author-id: sungup
#feature-img: "assets/img/posts/2019-12-23-prometheus-and-grafana-banner.jpeg"
tags: [ubuntu, raspberry pi 4, kubernetes, cri-o, container]
date: 2019-12-26 00:00:00
---

이번에는 crio 환경에서 컨테이너 이미지를 빌드를 하고 이를 private repository에 올리는 방법을 정리합니다. crio의
기본설정에서 private registry를 설정해 둘 수 있지만, 라즈베리 파이에서 quay, docker.io와 함께 사용하기 위해서는
선택적으로 활용해야 하기 때문에 crio의 설정은 기본 설정으로 놔 둔 상태이서 진행합니다. 

단, crio의 환경은 일반 docker와 달리 컨테이너를 실행하는 `crictl`과 빌드를 담당하는 `buildah`가 분리되어 있어
별도의 패키지를 설치해야 합니다.

*관련 포스트 링크*

# Install buildah

앞서 **crio** 를 설치한 repository를 통해 buiildah를 설치합니다.

```bash
sudo apt install -y buildah;
```

**buildah**의 설치에 성공하였더라도 일부 Dockerfile로는 컨테이너 이미지를 만들 수 없습니다. 한 예로, 아래와 같이
컨테이너 생성을 위해 일부 명령어의 실행이 필요한 경우, ubuntu환경의 buildah는 컨테이너 실행을 위한 **runc**를 찾을
수 없어 이미지 생성에 실패하게 됩니다.

```dockerfile
FROM docker.io/fedora:latest

MAINTAINER Sungup Moon

RUN dnf install cmake make clang;
```

```text
$ buildah build-using-dockerfile .
STEP 1: FROM docker.io/fedora:latest
Getting image source signatures
Copying blob 5da54debb5a9 done
Copying config d0b2459d46 done
Writing manifest to image destination
Storing signatures
STEP 2: MAINTAINER Sungup Moon
STEP 3: RUN dnf install cmake make clang;
error running container: error creating container for [/bin/sh -c dnf install cmake make clang;]: : exec: "runc": executable file not found in $PATH
error building at STEP "RUN dnf install cmake make clang;": error while running runtime: exit status 1
ERRO[0095] exit status 1                                
```

따라서, **buildah**는 실행 시 **runc**를 반드시 참조해야 합니다 **runc**는 앞서 설치한 **crio**설치시
`/usr/lib/cri-o-runc/sbin/runc`에 이미 설치되어 있어, 해당 파일을 아래 명령어로 `/usr/local/bin`에
링크로 연결합니다.

```shell
sudo ln -s /usr/lib/cri-o-runc/sbin/runc /usr/local/bin/runc;
```

## Make example using buildah

**buildah**를 설치한 다음 새 디렉토리를 만들어 `index.html`와 `Dockerfile`을 만들고 아래와 같은 파일을
준비합니다.

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

파일 생성 후 아래 명령어로 컨테이너 이미지를 생성후 업로드 합니다. buildah는 사용자 계정별로 이미지를 생성할 수 있기 때문에,
`sudo`와 같은 명령어는 필요로 하지 않습니다.

```shell
buildah build-using-dockerfile -t {registry address}/hello .;
buildah push {registry address}/hello;
```

You can check the container images using following command. Also, you can
remove images already uploaded on the private registry.

```text
$ buildah images
REPOSITORY                  TAG      IMAGE ID       CREATED         SIZE
{registry address}/hello2   latest   6c30e001557f   2 minutes ago   24.1 MB
docker.io/library/nginx     alpine   c2efcb34277e   2 days ago      24.1 MB          
$ buildah rmi {registry address}/hello2 docker.io/library/nginx:alpine
untagged: {registry address}/hello2:latest
6c30e001557fdb7b812d514026e4d8cc3ccb5c7365071c179e5534c13c4b9a67
untagged: docker.io/library/nginx:alpine
c2efcb34277e5f88b5775b976153d5e1b6be87499e2b1f4e30110db5aab48ef4
```

## Start example service with the new container

Now we start service with the new built image. Make the deploy and service YAML
file and apply that.

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
        image: {registry address}/hello:latest
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
  - host: "{example domain name}"
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
$ curl -D- http://{example domain name}
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
