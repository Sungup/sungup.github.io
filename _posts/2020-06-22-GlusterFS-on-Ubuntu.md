---
layout: post
title: GlusterFS on Ubuntu 20.04
author-id: sungup
#feature-img: "assets/img/posts/2019-12-23-prometheus-and-grafana-banner.jpeg"
tags: [ubuntu, glusterfs, raspberry pi 4]
date: 2020-06-22 00:00:00
---

Today, I'll explain how to setup GlusterFS volume storage on ubuntu server.
Currently, I'm ready to re-build k8s cluster on the volume storage service like
Ceph or GlusterFS. However, Raspberry Pi is little bit sufficient to run the
Ceph and K8s clusters on the same board. So, I'll use the GlusterFS as a
backend volume service of my K8s cluster on 4 Raspi servers.

## Make ready empty volume for GlusterFS

Before install and build storage pool, we must prepare the empty volume for
the GlusrerFS storage pool. Through the same way of the previous post, find USB
block device on the `/dev/sda1` and format that to ext4 file system. After that,
make the new mounting directory `/vol/gv01`, and mount USB device on that. For
the automatic mounting on the boot process, register mounting information at
`/etc/fstab`.

```bash
# Format partition and assign label.
sudo mkfs.ext4 /dev/sda1;
sudo e2label /dev/sda1 gv01;

# Mount USB device to /vol/gv01
sudo mount LABEL=gv01 /vol/gv01;
sudo mkdir /vol/gv01/k8s

# Add fstab entry for automatic mounting
sudo su - -c "echo LABEL=gv01 /vol/gv01 ext4 defaults 0 2 >> /etc/fstab";
```

## Install

Now, we install the GlusterFS server and client package on all Raspi servers.

```bash
sudo apt install -y glusterfs-server glusterfs-client;
```

After install package, we connect only 1 server, any server is OK, and probing
other servers to make peering each other. In this example, we use 4 Raspi
servers naming with `rbp4001.example.io ~ rbp4004.example.io`, and make probing
and building volume on the `rbp4001.example.io`.

```bash
# Add other 3 peers
sudo gluster peer probe rbp4002.example.io
sudo gluster peer probe rbp4003.example.io
sudo gluster peer probe rbp4004.example.io

# Check peer status
sudo gluster peer status
```

If you run `gluster peer status`, you can see the connection status like this:

```text
Number of Peers: 3

Hostname: rbp4002.example.io
Uuid: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
State: Peer in Cluster (Connected)

Hostname: rbp4003.example.io
Uuid: 1e223213-1d32-4b11-a7d0-0289dbc99ec6
Uuid: yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy
State: Peer in Cluster (Connected)

Hostname: rbp4004.example.io
Uuid: zzzzzzzz-zzzz-zzzz-zzzz-zzzzzzzzzzzz
State: Peer in Cluster (Connected)
```

## Build storage volume

Currently, I'm using 4 Raspberry Pi boards with 256GB USB memory device. In
this environment, I can't build up volume with Distributed Replicated or
Dispersed volume. To build the Distributed Replicated volume, total storage
capacity is not enough further server related usage. And also, 4 Raspberry-Pis
cannot serve the Dispersed volume with best performance because of not aligned
to the power of 2 size.

So, I build the clustered volume with Distributed only in this article. If you
have enough space on your VMs or physical servers, don't use this Distributed
only volume. Please read this document before build volume:

[Setting Up Volumes](https://docs.gluster.org/en/latest/Administrator%20Guide/Setting%20Up%20Volumes/)

Anyway, we can build shared volume for k8s with `gluster volume` command.

```bash
# Create volume
sudo gluster volume create k8s-vol rbp400{1..4}:/vol/gv01/k8s;

# Start volume
sudo gluster volume start k8s-vol;

# Check volume information
sudo gluster volume info;
```

And, we can check volume structure and configuration with `gluster volume info`
command.

```text
Volume Name: k8s-vol
Type: Distribute
Volume ID: aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa
Status: Created
Snapshot Count: 0
Number of Bricks: 4
Transport-type: tcp
Bricks:
Brick1: rbp4001:/vol/gv01/k8s
Brick2: rbp4002:/vol/gv01/k8s
Brick3: rbp4003:/vol/gv01/k8s
Brick4: rbp4004:/vol/gv01/k8s
Options Reconfigured:
transport.address-family: inet
storage.fips-mode-rchecksum: on
nfs.disable: on
```
