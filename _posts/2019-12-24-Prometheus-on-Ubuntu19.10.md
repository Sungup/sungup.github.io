---
layout: post
title: Prometheus on Ubuntu 19.10
author-id: sungup
feature-img: "assets/img/posts/2019-12-23-prometheus-and-grafana-banner.webp"
tags: [ubuntu, monitoring, prometheus, grafana]
date: 2019-12-24 14:08:41
---

Ubuntu 환경에서 Prometheus 설치하는 방법을 정리하였습니다. Ubuntu 환경에서는 Prometheus의 DEB 패키지가
존재하기 때문에 `apt install`을 통해 설치가 가능합니다. 또한 node_exporter 만 필요한 경우 이전 등록했던
[Prometheus on CentOS 8](/2019/12/23/Prometheus-on-CentOS8.html)을 참고하시면 됩니다.

## Prerequisite

Before install Prometheus, run `sudo apt update && sudo apt dist-upgrade -y`.
Ubuntu supports the Prometheus DEB packages, don't need to change other
options.

## Install Prometheus

Make user account for prometheus and install packages using following commands.

```shell
sudo useradd --no-create-home --shell /usr/sbin/nologin prometheus;

sudo apt install -y prometheus prometheus-node-exporter;

# Start and register services
sudo systemctl start prometheus prometheus-node-exporter;
sudo systemctl enable prometheus prometheus-node-exporter;
```

If you need node_exporter only, run following command.

```shell
sudo apt install -y prometheus-node-exporter;

# Start and register node_exporter service
sudo systemctl start prometheus-node-exporter;
sudo systemctl enable prometheus-node-exporter;
```

## Register node_exporter node

If you installed node_exporter only on remote server, register node_exporter
server on the main prometheus server.

```yaml
scrape_configs:
  - job_name: 'node'
  # If prometheus-node-exporter is installed, grab stats about the local
  # machine by default.
  static_configs:
    - targets: ['localhost:9100', '<other node_exporter node>:9100']
```

After register other node_exporter, restart prometheus service on the main
prometheus server.

```shell
sudo systemctl restart prometheus;
```

## Reference

### English

- [How to Monitor an Ubuntu Server with Grafana & Prometheus](https://oastic.com/how-to-monitor-an-ubuntu-server-with-grafana-prometheus/)

### Korean

- [Ubuntu 서버에 Prometheus와 Grafana 설치](https://sarc.io/index.php/cloud/1807-ubuntu-server-prometheus-grafana)
