---
layout: post
title: Installing Ingress-Nginx on the Private Network
author-id: sungup
feature-img: "assets/img/posts/2020-01-08-k8s-dashboard-feature.jpeg"
tags: [kubernetes, ingress-nginx, metallb, load balancer]
date: 2020-01-07 00:00:00
---

I'll explain the next stage **"Installing Ingress on the Private Network"**. If
you want access the dashboard, you can choose a method from two way, (1) run the
`kube-proxy`, or (2) connect the dashboard to the ingress load-balancer. For
the first method, you always run the kube-proxy on the terminal, but you can
login from localhost only, or apply complex settings. If you want access the
dashboard from the remote PC, you connect the dashboard to the `Ingress-Nginx`
load-balancer.

*(Also, you can change the `ClusterIP` to `NodePort` directly for the dashboard.
In this case, the dahsboard occupy a port on the a server port. To share the
80/443 web port through all web service, using `Ingress-Nginx` is better
solution.)*

1. *[Creating Self-Signed Certification for Local HTTPS Environment]*
2. **[Installing Ingress-Nginx on the Private Network]**
3. *[Install and Access Kubernetes Dashboard]*

## Install Ingress Controller

To install the `Ingress` on our Raspberry Pi, download a yaml file with the
following command. *(The latest version at 2020/01/07 is **nginx-0.26.2**.)*

```shell
wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.26.2/deploy/static/mandatory.yaml
```

If you use the default **mandatory.yaml** file, the k8s downloads and run
**x86_64** images. So that, you must modify the image's name from
`nginx-ingress-controller` to `nginx-ingress-controller-arm64`.

```diff
--- mandatory.yaml      2020-01-07 22:02:46.797035276 +0900
+++ 01.mandatory.yaml   2020-01-04 10:00:30.490748187 +0900
@@ -217,7 +217,7 @@
         kubernetes.io/os: linux
       containers:
         - name: nginx-ingress-controller
-          image: quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.26.2
+          image: quay.io/kubernetes-ingress-controller/nginx-ingress-controller-arm64:0.26.2
           args:
             - /nginx-ingress-controller
             - --configmap=$(POD_NAMESPACE)/nginx-configuration
```

After that you can apply the modified file with `kubectl`.

```bash
kubectl apply -f mandatory.yaml;
```

## Open Service for Ingress-Nginx

After install the ingress-nginx, apply service to access through the L4 or L7
load- balancer. By default, Ingress-Nginx runs with the cloud load balancer in
the public cloud service. However, we don't have that load balancer in the
small bare-metal environment. To run the ingress in the bare-metal environment,
apply the pure software solution, like `MetalLB`, or export `NodePort` through
all service node. I'll describe the details about the ingress near future.

Anyway, I tried applying **MetalLB** in my home network. MetalLB also had been
installed successfully, but worked fine only 10 minutes. If some WIFI device
connected with my home network switch, the links, between the metallb
controller and home devices, has been reset and I couldn't connect through the
ingress-nginx. Finally, remove the metallb and simply export all NodePorts
currently.

To open the ingress load balancer service, you can download the service YAML
file from the following link. But, the linked file will not working with our
purpose, so we need to change the contents by the load-balancing environment.

<https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/baremetal/service-nodeport.yaml>

### Using NodePort

To open service with **NodePort**, make a YAML file and store following
contents. From default service YAML file, just add the `externalIPs` array to
open 80/443 ports for the ingress-nginx.

```diff
--- service-nodeport.yaml       2020-01-07 22:34:38.827830196 +0900
+++ 02.service-nodeport.yaml    2020-01-04 14:13:04.358401616 +0900
@@ -8,6 +8,10 @@
     app.kubernetes.io/part-of: ingress-nginx
 spec:
   type: NodePort
+  externalIPs:
+    - 10.0.1.62
+    - 10.0.1.63
+    - 10.0.1.64
   ports:
     - name: http
       port: 80
```

After saving the changes, apply the service YAML file.

```shell
kubectl apply -f service-nodeport.yaml;
```

### Using MetalLB

#### Install MetalLB

Same with other posts about the Raspberry Pi, download the manifests file and
add the repository URL of the all of `image` keys.

```shell
wget https://raw.githubusercontent.com/google/metallb/v0.8.3/manifests/metallb.yaml;
```

```diff
--- metallb.yaml     2020-01-04 19:07:04.536753309 +0900
+++ 01.metallb.yaml  2020-01-04 19:05:09.886822710 +0900
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

And apply the changed YAML file to install the metallb.

```bash
kubectl apply -f metallb.yaml;
```

#### Apply MetallLB config map

To build the load-balancer simply, make the config map YAML file and save the
following contents. The layer2 protocol is the simple method to build the load-
balancer. Also, you can set the address range using CIDR or specific address
range with `-`.

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

After save the config map and apply that.

```bash
kubectl apply -f metallb-configmap.yaml;
```

If you want setup more customized, please read the official documents,
"[Bare-metal considerations]".

#### Requesting Specific IP

Different from `NodeExport`, just change the type from `NodePort` to
`LoadBalancer`. Don't add `externalIP` field. If the metallb works fine,
metallb will assign the new IP to the ingress-nginx service.

```diff
--- service-nodeport.yaml       2020-01-07 22:34:38.827830196 +0900
+++ 02.service-metallb.yaml     2020-01-04 20:27:06.034182181 +0900
@@ -7,7 +7,7 @@
     app.kubernetes.io/name: ingress-nginx
     app.kubernetes.io/part-of: ingress-nginx
 spec:
-  type: NodePort
+  type: LoadBalancer
   ports:
     - name: http
       port: 80
```

## Add cert file

If you want access the k8s web app through the ingress with HTTPS, you must
register the default certification file to the k8s. I already explain making
the certification file at the previous post. To use the key and cert file,
register the secret file with following command. The cert and key file will be
stored at the default namespace.

```shell
# Register the key and cert file
kubectl create secret tls {secret-name} --key {cert file}.key --cert {cert file}.crt;

# Query the stored secret information.
kubectl describe secret {secret-name};
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

[Creating Self-Signed Certification for Local HTTPS Environment]: /2020/01/06/Creating-Self-Signed-Certification-for-Local-HTTPS-Environment.html
[Installing Ingress-Nginx on the Private Network]: /2020/01/07/Installing-Ingress-Nginx-on-the-Private-Network.html
[Install and Access Kubernetes Dashboard]: /2020/01/08/Install-and-Access-Kubernetes-Dashboard.html
