---
layout: post
title: Creating Self-Signed Certification for Local HTTPS Environment
author-id: sungup
#feature-img: "assets/img/posts/2019-12-23-prometheus-and-grafana-banner.jpeg"
tags: [certification, self-signed, HTTPS]
date: 2020-01-06 00:00:00
---

Before start build-up the K8s dashboard  with Ingress, we need to make the
self-signed certification file to connect the K8s dashboard. Making root
certification is better solution to reduce making and registering each
certification file of the other app example.

## Create root certification file

First of all generate the root private key and certification file. You must
enter the passphrase while generating the root key file. *(**my.domain** is the
example domain. Please change that to your domain name.)*

```bash
# Generate private key for root-cert
openssl genrsa -des3 -out my.domain.key 2048;

# Generate root-cert file
openssl req -x509 -new -nodes -key my.domain.key \
            -sha256 -days 1825 -out my.domain.pem;
```

After generating the root certification file, you need the that cert file on
you system. Copy the `my.domain.pem` file on your PC. In my case, I use
`Keychain Access` on my MacBook and register certification file on the system
directory.

## Create CA-Signed certification for the local sites

Next we generate the local site certification file. I'll use that certification
file for the K8s dashboard, and use the **k8s.my.domain** for the example
sub-domain name.

```bash
# Generate private key for the sub-domain
openssl genrsa -out k8s.my.domain.key 2048;

# Generate the sub-domain's CSR file
openssl req -new -key k8s.my.domain.key -out k8s.my.domain.csr;
```

While generating the CSR file, you must sync the input information with the
root certification file without `Common Name`.

After that, we should define the Subject Alternative Name (SAN) extension
(Please read the [Brad Touesnard's article]). Following contents is the
template file of SAN extension.

```ini
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
subjectAltName = @alt_names

[alt_names]
DNS.1 = example.com
DNS.2 = example.com.example.ip.xip.io
```

Store this conents as `example.com.ext.template` and generate the sub-domain's
SAN extension file `k8s.my.domain.ext` using the `cat` and `sed` command. Using
the SAN extension file, generate the sub-domain's certification file **(.crt)**
for the k8s dashboard site.

```bash
# Generate the SAN extension file
cat example.com.ext.template \
    | sed "s/example.com/k8s.my.domain/" \
    | sed "s/example.ip/XX.XX.XX.XX/" > k8s.my.domain.ext;

# Generate the certification file for the sub-domain
openssl x509 -req -in k8s.my.domain.csr \
    -CA my.domain.pem -CAkey my.domain.key -CAcreateserial \
    -days 1825 -sha256 -extfile k8s.my.domain.ext \
    -out k8s.my.domain.crt;
```

## Reference

- [How to Create Your Own SSL Certificate Authority for Local HTTPS Development]
- [How to Set Up HTTPS Locally Without Getting Annoying Browser Privacy Errors]

[How to Create Your Own SSL Certificate Authority for Local HTTPS Development]: https://deliciousbrains.com/ssl-certificate-authority-for-local-https-development/
[How to Set Up HTTPS Locally Without Getting Annoying Browser Privacy Errors]: https://deliciousbrains.com/https-locally-without-browser-privacy-errors/
[Brad Touesnard's article]: https://deliciousbrains.com/https-locally-without-browser-privacy-errors/#creating-self-signed-certificate
