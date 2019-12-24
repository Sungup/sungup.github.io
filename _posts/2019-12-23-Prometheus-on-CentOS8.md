---
layout: post
title: Prometheus on CentOS 8
author-id: sungup
feature-img: "assets/img/post/20191223-prometheus-and-grafana-banner.jpeg"
tags: [centos, monitoring, prometheus, grafana]
date: 2019-12-23 09:41:18
---

CentOS 환경에서 Prometheus 설치하는 방법을 정리하였습니다. CentOS환경에서는 Ubuntu와 달리 Prometheus가
기본 Repository에 포함되어 있지 않기 때문에 구축을 위해서는 Official Site에서 바이너리를 다운로드 받아 수동으로
설치를 진행해야 합니다. 일부 서비스에 대해서만 모니터링 기능을 확장하는 경우 node_exporter 설치 방법만 참고하셔서 확장하면
됩니다.

## Prerequisite

- Update CentOS 8

Before install Prometheus on CentOS 8, run `sudo dnf update` and change selinux
configuration from `enforcing` to `disabled` or `permissive`. Open
`/etc/sysconfig/seinux` and change like following config.

```ini
#SELINUX=enforcing
SELINUX=permissive
```

- Install required packages

```bash
sudo dnf install -y wget tar;
```

## Install Prometheus

### Make user account for prometheus

Make prometheus and node_exporter user account and related directory.

```shell
sudo useradd --no-create-home --shell /bin/false prometheus;
sudo useradd --no-create-home --shell /bin/false nodeusr;
sudo mkdir /etc/prometheus /var/lib/prometheus;
sudo chown prometheus:prometheus /etc/prometheus /var/lib/prometheus;
```

### Download prometheus and node_exporter

Download prometheus and node_exporter from Prometheus Official Download Page.
([Link](https://prometheus.io/download/))

Latest version at 2019 Dec. 23:

- **prometheus**: 2.14.0
- **node_exporter**: 0.18.1

### Install prometheus files

First of all, extract achived prometheus file and copy binaries and libraries
using followind commands.

```bash
# Extract archived file
tar -xzvf prometheus-2.14.0.linux-amd64.tar.gz;

# Copy binaries into /usr/local/bin
sudo cp prometheus-2.14.0.linux-amd64/prometheus /usr/local/bin;
sudo cp prometheus-2.14.0.linux-amd64/promtool /usr/local/bin;

# Copy prometheus default configuration file into /etc/prometheus
sudo cp prometheus-2.14.0.linux-amd64/prometheus.yml /etc/prometheus;

# Copy console related files into /etc/prometheus
sudo cp -r prometheus-2.14.0.linux-amd64/consoles /etc/prometheus;
sudo cp -r prometheus-2.14.0.linux-amd64/console_libraries /etc/prometheus;

# Change ownership binaries and console libraries
sudo chown prometheus:prometheus /usr/local/bin/prometheus /usr/local/bin/promtool;
sudo chown prometheus:prometheus /etc/prometheus/prometheus.yml;
sudo chown -R prometheus:prometheus /etc/prometheus/consoles /etc/prometheus/console_libraries;
```

### Change configuration value

Following YAML config is the default configs of prometheus. If you want to
change the hostname or add other host, change the
`scrape_configs -> static_configs -> targets' list`.

```yaml
# my global config
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'

    # Override the global default and scrape targets from this job every 5 seconds.
    scrape_interval: 5s
    scrape_timeout: 5s

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
      - targets: ['localhost:9090']
```

### Make and register service file

To run prometheus as a service, make and edit
`/etc/systemd/system/prometheus.service`.

```ini
[Unit]
Description=Prometheus
Documentation=https://prometheus.io/docs/introduction/overview/
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecReload=/bin/kill -HUP $MAINPID
ExecStart=/usr/local/bin/prometheus \
    --config.file=/etc/prometheus/prometheus.yml \
    --storage.tsdb.path=/var/lib/prometheus/ \
    --web.console.templates=/etc/prometheus/consoles \
    --web.console.libraries=/etc/prometheus/console_libraries \
    --web.listen-address=0.0.0.0:9090

SyslogIdentifier=prometheus
Restart=always

[Install]
WantedBy=multi-user.target
```

After save and exit the config file, reload and start prometheus service.

```shell
sudo systemctl daemon-reload;
sudo systemctl start prometheus;
sudo systemctl enable prometheus;
```

If prometheus service couldn't run, please check using following commands.

```shell
sudo systemctl status prometheus;
sudo journalctl -xe;
```

## Install Node Exporter

If you already installed prometheus on other server, you can install
node_exporter only on the target server. In this case, install the
node_exporter without prometheus and register IP or URI address in the
prometheus config the main server already installed prometheus.

### Install node_exporter and register as a service

Extract achived node_exporter and install that `/usr/local/bin`

```shell
tar -xzvf node_exporter-0.18.1.linux-amd64.tar.gz;
sudo cp node_exporter-0.18.1.linux-amd64/node_exporter /usr/local/bin;
```

Make and edit the following contents at
`/etc/systemd/system/node_exporter.service`.

```ini
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=nodeusr
Group=nodeusr
Type=simple
ExecReload=/bin/kill -HUP $MAINPID
ExecStart=/usr/local/bin/node_exporter

SyslogIdentifier=node_exporter
Restart=always

[Install]
WantedBy=multi-user.target
```

Similar with prometheus, reload and start daemon.

```shell
sudo systemctl daemon-reload;
sudo systemctl start node_exporter;
sudo systemctl enable node_exporter;
```

### Add node_exporter configs in prometheus.yml

Add following config lines under the `scrape_configs`.

```yaml
  # Add this line after 'prometheus' job
  - job_name: 'node'
    # If prometheus-node-exporter is installed, grab stats about the local
    # machine by default.
    static_configs:
      - targets: ['localhost:9100']
```

Save the `prometheus.yml` and restart prometheus service.

```shell
sudo systemctl restart prometheus;
```

Access `http://<prometheus server>:9090/targets` and check the targets are up.

![prometheus-target](/assets/img/posts/20191223-prometheus-01.png)

## Open filrewall for prometheus and node_exporter

Add firewall rules for prometheus and reload firewall.

```shell
sudo firewall-cmd --zone=public --add-port=9090/tcp --permanent; # for prometheus
sudo firewall-cmd --zone=public --add-port=9100/tcp --permanent; # for node_exporter
sudo systemctl reload firewalld;
```

## Reference

- [How to install and configure Prometheus on CentOS 7](https://www.fosslinux.com/10398/how-to-install-and-configure-prometheus-on-centos-7.htm)
- [Instal Prometheus Server on CentOS 7 / Ubuntu 18.04](https://computingforgeeks.com/install-prometheus-server-on-centos-7/)
- [How to install and configure Grafana on CentOS 7](https://www.fosslinux.com/8328/how-to-install-and-configure-grafana-on-centos-7.htm)
