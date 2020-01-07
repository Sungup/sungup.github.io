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

## Build Web UI

Clone web ui repository from `https://github.com/Joxit/docker-registry-ui.git`

```bash
sudo crictl pull docker.io/nginx:alpine
```
