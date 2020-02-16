---
layout: post
title: Install Loki and Promtail
author-id: sungup
feature-img: "assets/img/posts/2019-12-23-prometheus-and-grafana-banner.jpeg"
tags: [ubuntu, centos, monitoring, prometheus, grafana]
date: 2020-02-16 14:35:41
---

## Install Loki

### Download and Install loki

Download and unzip Loki. Latest version at 2020/02/16 is v1.3.0.

```bash
# 1. Download and unzip
wget https://github.com/grafana/loki/releases/download/v1.3.0/loki-linux-amd64.zip;
unzip loki-linux-amd64.zip;
chmod a+x loki-linux-amd64;

# 2. Install /usr/local/bin
mv loki-linux-amd64 /usr/local/bin/loki;

# 3. Make data store at /var/lib/loki
mkdir /var/lib/loki;
chown -R root:root /var/lib/loki

# 4. Make config store at /etc/loki
mkdir /etc/loki;
chown -R root:root /etc/loki;
```

### Make Loki config file

Store following config at `/etc/loki/config.yaml`.

```yaml
auth_enabled: false

server:
  http_listen_port: 3100

ingester:
  lifecycler:
    address: 127.0.0.1
    ring:
      kvstore:
        store: inmemory
      replication_factor: 1
    final_sleep: 0s
  chunk_idle_period: 5m
  chunk_retain_period: 30s

schema_config:
  configs:
  - from: 2018-04-15
    store: boltdb
    object_store: filesystem
    schema: v9
    index:
      prefix: index_
      period: 168h

storage_config:
  boltdb:
    directory: /var/lib/loki/index

  filesystem:
    directory: /var/lib/loki/chunks

limits_config:
  enforce_metric_name: false
  reject_old_samples: true
  reject_old_samples_max_age: 168h

chunk_store_config:
  max_look_back_period: 0

table_manager:
  chunk_tables_provisioning:
    inactive_read_throughput: 0
    inactive_write_throughput: 0
    provisioned_read_throughput: 0
    provisioned_write_throughput: 0
  index_tables_provisioning:
    inactive_read_throughput: 0
    inactive_write_throughput: 0
    provisioned_read_throughput: 0
    provisioned_write_throughput: 0
  retention_deletes_enabled: false
  retention_period: 0
```

Test loki with following command.

```bash
/usr/local/bin/loki -config.file /etc/loki/config.yaml
```

And connect `http://{loki server address}:3100/metrics` through a browser.

### Make service

Make `/etc/systemd/system/loki.service` file with following contents.

```ini
[Unit]
Description=Loki service
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/loki -config.file /etc/loki/config.yaml

[Install]
WantedBy=multi-user.target
```

## Install Promtail

### Download and install Promtail

Similar with Loki download promtail from github

```bash
wget https://github.com/grafana/loki/releases/download/v1.3.0/promtail-linux-amd64.zip;

unzip promtail-linux-amd64.zip;
chmod a+x promtail-linux-amd64;

# 2. Install /usr/local/bin
mv promtail-linux-amd64 /usr/local/bin/promtail;

# 3. Make data store at /var/lib/promtail
mkdir /var/lib/promtail;
chown -R root:root /var/lib/promtail

# 4. Make config store at /etc/promtail
mkdir /etc/promtail;
chown -R root:root /etc/promtail;
```

### Make Promtail config file

Defailt configuration

```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /var/lib/promtail/positions.yaml

# Loki's api address to push
clients:
  - url: http://{loki server address}:3100/loki/api/v1/push

scrape_configs:
- job_name: system
  static_configs:
  - targets:
      - localhost
    labels:
      job: varlogs
      __path__: /var/log/*log
```

For the journal

```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /var/lib/promtail/positions.yaml

# Loki's api address to push
clients:
  - url: http://{loki server address}:3100/loki/api/v1/push

scrape_configs:
  - job_name: journal
    journal:
      max_age: 12h
      path: {journal directory path}
        # CentOS: /run/log/journal
        # Ubuntu: /var/log/journal
      labels:
        job: systemd-journal
    relabel_configs:
      - source_labels: ['__journal__systemd_unit']
        target_label: 'unit'
```

#### CentOS case

journal path is `/run/log/journal` not `/var/log/journal`!!!

### Make Promgail Service

Make `/etc/systemd/system/promtail.service` file with following contents.

```ini
[Unit]
Description=Promtail service
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/promtail -config.file /etc/promtail/promtail.yml

[Install]
WantedBy=multi-user.target
```

## Reference

- [Grafana Logs Panel with Loki and Promtail]

[Grafana Logs Panel with Loki and Promtail]: https://medium.com/@sean_bradley/setup-grafana-logs-panel-with-loki-and-promtail-3bd89cf40c31
