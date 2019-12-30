# Install

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

[Creating sample user]: https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/creating-sample-user.md
[Web UI (Dashboard)]: https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/
[kubernetes-metrics-scraper on ARM on 2.0beta1: Unable to initialize database tables: Binary was compiled with 'CGO_ENABLED=0]: https://github.com/kubernetes/dashboard/issues/4029
[Deploy the Kubernetes Web UI Dashboard]: https://xuri.me/2019/01/23/deploy-the-kubernetes-web-ui-dashboard.html
