---
layout: post
title: How to Build Container with Buildah
author-id: sungup
#feature-img: "assets/img/posts/2019-12-23-prometheus-and-grafana-banner.jpeg"
tags: [ubuntu, raspberry pi 4, kubernetes, cri-o, container, buildah]
date: 2020-01-27 00:00:00
---

In this post, I'll explain how to build a container using `buildah` and upload
that into the private registry. We can set the default registry in the crio's
config file, but this method is not good because some images doesn't matched
for the arm64 environment system even if that had been provided from quay or
docker.io. So that reason, we keep the blank default registry settings.

- *[How to Setup the NFS on Ubuntu]*
- *[Build the Private Registry with the NFS Volume]*
- **[How to Build Container with Buildah]**

## Install buildah

Different from Docker, the running tools and build tools is divided into the
`crictl` and the `buildah`, so we need to install the `buildah`. That package
is provided from the project atomic repository, already installed for the crio,
simply run following command.

```bash
sudo apt install -y buildah;
```

Even if you installed `buildah` successfully, some Dockerfile script can't
build a container image. For example, some **Dockerfile** script should run
package install command like **dnf** or **apt** for the pre-built libraries.
However if you ran that **Dockerfile** script, `buildah` try to search and
run the `runc`, but can't find that in the default **PATH** environment and
will be failed. The `runc` is the core engine to run container in the CRI-O,
so `buildah` should use that to run some commands in Dockerfile while building
the custom images. Following **Dockerfile** is that example file. If you run
this Dockerfile without runc, you can see the following error logs.

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

To reference the `runc`, you need to run following command. Fortunately, the
runc had been installed at `/usr/lib/cri-o-runc/sbin/runc` when you installed
**CRI-O**, so you can make link that file into `/usr/local/bin` simply.

```shell
sudo ln -s /usr/lib/cri-o-runc/sbin/runc /usr/local/bin/runc;
```

However, if you don't have a `RUN` section in your Dockerfile for the custom
image, you can skip this making link step! :-)

## Make example using buildah

Now, we build an example container image to test building and pushing container
into our private registry. In this post, we make a very very simple web page
using **nginx** image. Make a new directory and make `index.html` and
`Dockerfile` with following contents.

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

After making two files, run following commands to build and push our image.
The `buildah` can run in any user account, so you don't need to run with
`sudo` to get root privileges.

```shell
buildah build-using-dockerfile -t {registry address}/hello .;
buildah push {registry address}/hello;
```

You can check the container images using following command. Also, you can
remove images already uploaded from the local registry.

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

Now we start service with the new image. Make the deploy and service YAML file
and apply that.

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

- [buildah]
- [nginx: Docker Official Images]

[buildah]:https://github.com/containers/buildah/blob/master/install.md
[nginx: Docker Official Images]: https://hub.docker.com/_/nginx

[How to Setup the NFS on Ubuntu]: /2020/01/15/How-to-Setup-the-NFS-on-Ubuntu.html
[Build the Private Registry with the NFS Volume]: /2020/01/19/Build-the-Private-Registry-with-the-NFS-Volume.html
[How to Build Container with Buildah]: /2020/01/27/How-to-Build-Container-with-Buildah.html
