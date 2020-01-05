---
layout: post
title: Install Kubernetes on Raspberry Pi 4
author-id: sungup
#feature-img: "assets/img/posts/2019-12-23-prometheus-and-grafana-banner.jpeg"
tags: [ubuntu, raspberry pi 4, kubernetes, cri-o, container]
date: 2019-12-26 00:00:00
---

## Create root certification file

First of all generate private key.

```bash
# Generate private key for root-cert
openssl genrsa -des3 -out my.domain.key 2048;

# Generate root-cert file
openssl req -x509 -new -nodes -key my.domain.key -sha256 -days 1825 -out my.domain.pem;
```

Install generated certification file into your system

## Create CA-Signed certification for the development sites

```bash
openssl genrsa -out k8s.sungup.io.key 2048;

openssl req -new -key k8s.sungup.io.key -out k8s.sungup.io.csr;

cat sungup.io.ext.template | sed "s/my.domain.com/k8s.sungup.io/" | sed "s/my.ip.com/10.0.1.61/" >> k8s.sungup.io.ext;

openssl x509 -req -in k8s.sungup.io.csr -CA sungup.io.pem -CAkey sungup.io.key \
    -CAcreateserial -days 1825 -sha256 -extfile k8s.sungup.io.ext \
    -out k8s.sungup.io.crt;
```

## Templates

### Subject Alternative Name Extention

```ini
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
subjectAltName = @alt_names

[alt_names]
DNS.1 = my.domain.com
DNS.2 = my.domain.dom.my.ip.com.xip.io
```

## Reference

[How to Create Your Own SSL Certificate Authority for Local HTTPS Development]: https://deliciousbrains.com/ssl-certificate-authority-for-local-https-development/
