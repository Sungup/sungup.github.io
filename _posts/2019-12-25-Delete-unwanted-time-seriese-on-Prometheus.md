---
layout: post
title: Delete unwanted timeseriese on Prometheus
author-id: sungup
feature-img: "assets/img/posts/2019-12-23-prometheus-and-grafana-banner.jpeg"
tags: [monitoring, prometheus]
date: 2019-12-25 16:03:41
---

Prometheus 설정 후 서버 추가/변경 시 발생하는 불필요한 데이터들을 삭제하는 방법을 정리합니다. Prometheus는 외부에서
별도로 접근할 수 있는 DB 인터페이스를 제공하지 않기 때문에 Prometheus Server의 WEB API를 통해서 관련 처리를 진행해야
합니다.

## Enable admin

To enable admin api, pass the `--web.enable-admin-api` argument to Prometheus
service. Prometheus doesn't support admin api as default option.

### CentOS

Before update service script file, please read the previous article
[Prometheus on CentOS 8](/2019/12/23/Prometheus-on-CentOS8.html).

To enable admin api, modify service script file at
`/etc/systemd/system/prometheus.service`. Open and add
`--web.enable-admin-api` argument at the end of `ExecStart` value.

```ini
...
[Service]
...
ExecStart=/usr/local/bin/prometheus \
    --config.file=/etc/prometheus/prometheus.yml \
    --storage.tsdb.path=/var/lib/prometheus/ \
    --web.console.templates=/etc/prometheus/consoles \
    --web.console.libraries=/etc/prometheus/console_libraries \
    --web.listen-address=0.0.0.0:9090 \
    --web.enable-admin-api
...
```

After save the service script file, you can run following commands to apply
the changes.

```shell
sudo systemctl daemon-reload;
sudo systemctl restart prometheus;
```

### Ubuntu

Ubuntu default packages support the default running config file on
`/etc/default/prometheus`. Open that file and append or create `ARG` variable
with `--web.enable-admin-api`.

```ini
ARGS="--web.enable-admin-api"
```

Also same with CentOS, run following command to apply the changes.

```shell
sudo systemctl restart prometheus;
```

## Delete unwanted datas

To run the admin-api, you should call http address using `curl`. So, if `curl`
isn't installed in your system, you must install that.

```shell
# Ubuntu
sudo apt install -y curl;

# Old CentOS, Fedora or RedHat
sudo yum install -y curl;

# Latest CentOS, Fedora or RedHat
sudo dnf install -y curl;
```

## Delete unwanted job

```shell
curl -X POST -g 'http://<prometheus server>:9090/api/v1/admin/tsdb/delete_series?match[]={job="<job name>"}'
```

## Delete unwanded instance

```shell
curl -X POST -g 'http://<prometheus server>:9090/api/v1/admin/tsdb/delete_series?match[]={instance="<instance target>"}'
```

## Reference

- [Prometheus: Delete Time Series Metrics](https://www.shellhacks.com/prometheus-delete-time-series-metrics/)
