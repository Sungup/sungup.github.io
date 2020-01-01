---
layout: post
title: Install Kubernetes on Raspberry Pi 4
author-id: sungup
#feature-img: "assets/img/posts/2019-12-23-prometheus-and-grafana-banner.jpeg"
tags: [ubuntu, raspberry pi 4, kubernetes, cri-o, container]
date: 2019-12-26 00:00:00
---

Raspberry Pi 4 기반 HW 구성에 Kubernetes(이하 K8s)를 설치하는 과정을 정리했습니다. Raspberry Pi 4 +
Ubuntu 19.10 환경을 기반으로 설치를 진행했으며, 다양한 방법 중에서 K8s Official Site에서 설명된 설치방법을
통해서 진행한 방법으로 정리되어 있습니다.

## Environments

- **Hardware**: Raspberry Pi 4 with 4GB
  (currently 2 board, will be extended to 4~5 board)
- **OS**: Ubuntu 19.10 64bit for Raspberry Pi 3/4
- **Container Engine**: CRI-O

## Update boot command on Raspberry Pi 4

Default Ubuntu 19.10 image for Raspberry Pi doesn't support cgroup memory at
boot time. So, if you run `kubeadm init` command without changes, initializing
will be failed with *"missing cgroups: memory"*.

To avoid this problem, you should modify the `/boot/firmware/nobtcmd.txt` file.
Open the `nobtcmd.txt` and append the following boot command options.

```text
cgroup_enable=cpuset cgroup_enable=memory cgroup_memory=1
```

If you change that file on the Raspberry Pi, reboot and check command line
options using following command.

```shell
cat /proc/cmdline;
```

## Setup CRI-O

### Install prerequisites

Before installing CRI-O, setup the `overlay` and `br_netfilter` module for the
K8s's network environments. Also, create `/etc/sysctl.d/99-kubernetes-cri.conf`
to apply sysctl parameters on boot time.

```shell
sudo modprobe overlay;
sudo modprobe br_netfilter;

# Make append on-boot module list on /etc/modules-load.d/crio.conf
sudo tee -a /etc/modules-load.d/crio.conf << EOF
overlay
br_netfilter
EOF

# Make and append sysctl network parameters on /etc/sysctl.d/99-kubernetes-cri.conf
sudo tee -a /etc/sysctl.d/99-kubernetes-cri.conf << EOF
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sudo sysctl --system;
```

### Install CRI-O

Use the following commands to install CRI-O.

```shell
# Install prerequisites
sudo apt update && sudo apt install software-properties-common;
sudo add-apt-repository ppa:projectatomic/ppa;
```

However, CRI-O only supports stable ubuntu version (XX.04) only. To install
CRI-O on the Ubuntu 18.10(cosmic)/19.10(eoan), you must modify the repository
config file at `/etc/apt/sources.list.d/projectatomic-ubuntu-ppa-eoan.list`.
Open the repository config file and change release version from `eoan` to
`disco`.

```ini
# The cosmic and eoan doesn't support the CRI-O packages. To install CRI-O,
# fake the release version changing from eoan to disco. If you would upgrade
# to the 20.04 "Focal Fossa", you can sync the release versions between Ubuntu
# and CRI-O.
deb http://ppa.launchpad.net/projectatomic/ppa/ubuntu disco main
# deb http://ppa.launchpad.net/projectatomic/ppa/ubuntu eoan main
# deb-src http://ppa.launchpad.net/projectatomic/ppa/ubuntu eoan main
```

After fix target source list file, update and install CRI-O 1.15 (latest
version at Dec. 25).

```shell
sudo apt update && sudo apt install cri-o-1.15;

sudo systemctl daemon-reload;
sudo systemctl start crio;
sudo systemctl enable crio;
```

### Check cgroup_driver

In the default CRI-O configuration, `cgroup_manager` has been set with the
**systemd**. This is the CRI-O official recommended value. But, K8s uses
**cgroupfs** as the default cgroup manager. So, you must check and sync that
value between **kubelet** and **cri-o** services.

To check running configs, run following command. (If jq is not installed,
you can install that first.)

```shell
sudo curl -v --unix-socket /var/run/crio/crio.sock http://localhost/info | jq;
```

If you want to change different driver, modify `/etc/crio/crio.conf`.

## Install kubeadm, kubelet and kubectl

### Ensure iptables tooling does not use the nftables backend

Modern linux distributions use `nftables` instead of `iptables` (RedHat 8,
Ubuntu 19.04, Fedora 29, etc...). But, the current `kubeadm` doesn't
compatible with `nftables`, it causes duplicated firewall rules and break
`kube-proxy`. To avoid this problem, you need to change the `iptables` tooling
to **legacy** mode.

```shell
sudo update-alternatives --set iptables /usr/sbin/iptables-legacy
sudo update-alternatives --set ip6tables /usr/sbin/ip6tables-legacy
sudo update-alternatives --set arptables /usr/sbin/arptables-legacy
sudo update-alternatives --set ebtables /usr/sbin/ebtables-legacy
```

### Open firewall port

If you run the firewall on Ubuntu, must open some network ports to communicate
control and worker nodes.

#### Control-plane node(s)

```shell
sudo ufw allow 6443/tcp;        # Kubernetes API server
sudo ufw allow 2379:2380/tcp;   # etcd server client API
sudo ufw allow 10250/tcp;       # Kubelet API
sudo ufw allow 10251/tcp;       # kube-scheduler
sudo ufw allow 10252/tcp;       # kube-controller-manager
```

#### Worker node(s)

```shell
sudo ufw allow 10250/tcp;       # Kubelet API
sudo ufw allow 30000:32767/tcp; # NodePort Services
```

### Install packages

Before setting-up the K8s cluter, you should install the kubectl from the
Official Kubernetes repository. For this, register the kubernetes repository
and install kubeadm, kubectl and kubelet.

- **kubeadm**: the command to bootstrap the cluster.
- **kubelet**: the component that runs on all of the machines in your cluster
               and does things like starting pods and containers.
- **kubectl**: the command line util to talk to your cluster.

```shell
sudo apt update && sudo apt install -y apt-transport-https curl;
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -;
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee -a /etc/apt/sources.list.d/kubernetes.list
sudo apt update;
sudo apt install kubectl kubeadm kubelet;
```

### Configure cgroup driver userd by kubelet on control-plane node

***Caution! This configuration has been deprecated. I'll change this setting to right
way soon.***

In this example, we use CRI-O as a default CRI, so we must modify file
`/etc/default/kubelet` with CRI's `cgroup-driver`. So, you must check the
cgroup driver option value in `/etc/crio/crio.conf` and sync with kubelet's
arguments value.

To change the default kubelet's argument, modify `/etc/default/kubelet`, like
this:

```ini
KUBELET_EXTRA_ARGS=--cgroup-driver=<value>
```

After changing the values, reload and restarting kubelet.

```shell
sudo systemctl daemon-reload;
sudo systemctl restart kubelet;
```

But, if the CRI's cgroup driver already set to `cgroupfs`, you don't need to
add `--cgroup-driver` option.

## Creating a single control-plane cluster

To initialize kubernetes, run `kubeadm init` with pod's CIDR and api server ip.
Also `kubeadm config images pull` to verify connectivity to gcr.io registries.

In this example, I use `10.244.0.0/16` for the pod network CIDR.

```shell
export API_SERVER_IP=XXX.XXX.XXX.XXX;
export POD_NET_CIDR=10.244.0.0/16;

# (Optional) pull all kube images before setup
sudo kubeadm config images pull;

# Run kubeadm init with api-server-advertise-address and
# pod-network-cidr options.
sudo kubeadm init --pod-network-cidr=${POD_NET_CIDR} \
                  --apiserver-advertise-address=${API_SERVER_IP};
```

If initializing is succeed, you can see the following text at the end of
terminal. Please save this text into a file because the last `kubeadm join`
command will be used on the other worker nodes.

```text
........
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[kubelet-check] Initial timeout of 40s passed.
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join <control plane ip>:6443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash>
```

Also, you must run 3 commands following logs.

```shell
mkdir -p $HOME/.kube;
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config;
sudo chown $(id -u):$(id -g) $HOME/.kube/config;
```

## Installing a pod network add-on with Calico

You should install a pod network add-on for the communication with each other.
In this article, I explane installation using the `Calico` add-on. For the
other add-ons, please read the Kubernetes official documents.

### Download Calico YAML

First of all, download `calico.yaml` file from the official site. The official
document recommends the 3.8 version calico, but that version **pod2daemon**
image has a Makefile bug and not working on the ARM64 architecture. So that
reason, I use the 3.9, 3.10 or 3.11 calico YAML file. But, **DON'T RUN
`kubectl apply` DIRECTLY!**

```shell
# The stable version 3.8 at Dec. 29, 2019, but we use 3.11 image to run on
# Raspberry Pi 4.
wget https://docs.projectcalico.org/v3.11/manifests/calico.yaml;
```

### Modify YAML to run Calico on Raspberry Pi

Before apply the `calico.yaml` file, we should change the following 4 items.

1. Change **image** fields from `calico` to `docker.io/calico`.
2. Change `CALICO_IPV4POOL_CIDR` values to the our CIDR `10.244.0.0/16`.
3. Add `IP_AUTODETECTION_METHOD` environment value with `interface=wlan0`.
4. Add `FELIX_IGNORELOOSERPF` environment value with `true`.

#### Change image fields

Different from default Docker environment, our ubuntu + cri-o environment
doesn't have default registry value in `/etc/crio/crio.conf`. So that, we
should add the registry address at the image path. If you uncomment the
default registry in the configuration file , you shoudn't set the registry
to `quay.io`, because the calico images from that registry support only the
amd64/x86 architecture. We need the arm64 images to run on the Raspberry Pi,
add the `docker.io` to download arm64 based images.

#### Change CALICO_IPV4POOL_CIDR

We already set the CIDR with `10.244.0.0/16`, but the default pod-network for
the calico is `192.168.0.0/16`. So that, we should match the CIDR between
`kubeadm init` and `calico.yaml`.

#### Add IP_AUTODETECTION_METHOD (Only for the WLAN interface)

Through the all post for Raspberry Pi 4, I had set the main network interface
to the `wlan0` not `eth0` because of lack of the home network switch`s port.
But, it raises some fault to find the pod's network interface.

To solve this problem, I must set the `IP_AUTODETECTION_METHOD` to make the
`calico-node` point to the `wlan0` as the default network interface. If you
use the wlan0 as the default network like me, please check the YAML configs
the end of this calico sub-article.

#### Add FELIX_IGNORELOOSERPF (Optional)

Calico's node pods will not be READY to `1/1` in spite of `Running` state. I
saw an article for Calico from [Creating a Kind Cluster With Calico Networking]
by *Alexander Brand*. But, he doesn't recommend this method while setting up
pods network directly.

You can see the RPF setting value in `/proc/sys/net/ipv4/conf/all/rp_filter`
may be set with 2 as **loose** RPF. ~~I didn't set that value to 1, but guess
this method is better than changing the `FELIX_IGNORELOOSERPF` directly.~~

#### Differences between original and modified YAML

Following **diff** contents is the changed values from the original
`calico.yaml`.

```diff
--- calico-original.yaml        2020-01-01 15:31:04.530374465 +0900
+++ calico.yaml 2020-01-01 15:28:16.620898686 +0900
@@ -516,7 +516,7 @@
         # It can be deleted if this is a fresh installation, or if you have already
         # upgraded to use calico-ipam.
         - name: upgrade-ipam
-          image: calico/cni:v3.11.1
+          image: docker.io/calico/cni:v3.11.1
           command: ["/opt/cni/bin/calico-ipam", "-upgrade"]
           env:
             - name: KUBERNETES_NODE_NAME
@@ -538,7 +538,7 @@
         # This container installs the CNI binaries
         # and CNI network config file on each node.
         - name: install-cni
-          image: calico/cni:v3.11.1
+          image: docker.io/calico/cni:v3.11.1
           command: ["/install-cni.sh"]
           env:
             # Name of the CNI config file to create.
@@ -574,7 +574,7 @@
         # Adds a Flex Volume Driver that creates a per-pod Unix Domain Socket to allow Dikastes
         # to communicate with Felix over the Policy Sync API.
         - name: flexvol-driver
-          image: calico/pod2daemon-flexvol:v3.11.1
+          image: docker.io/calico/pod2daemon-flexvol:v3.11.1
           volumeMounts:
           - name: flexvol-driver-host
             mountPath: /host/driver
@@ -585,7 +585,7 @@
         # container programs network policy and routes on each
         # host.
         - name: calico-node
-          image: calico/node:v3.11.1
+          image: docker.io/calico/node:v3.11.1
           env:
             # Use Kubernetes API as the backing datastore.
             - name: DATASTORE_TYPE
@@ -610,6 +610,10 @@
             # Auto-detect the BGP IP address.
             - name: IP
               value: "autodetect"
+            - name: FELIX_IGNORELOOSERPF
+              value: "true"
+            - name: IP_AUTODETECTION_METHOD
+              value: "interface=wlan0"
             # Enable IPIP
             - name: CALICO_IPV4POOL_IPIP
               value: "Always"
@@ -760,7 +764,7 @@
       priorityClassName: system-cluster-critical
       containers:
         - name: calico-kube-controllers
-          image: calico/kube-controllers:v3.11.1
+          image: docker.io/calico/kube-controllers:v3.11.1
           env:
             # Choose which controllers to run.
             - name: ENABLED_CONTROLLERS
```

### Apply calico.yaml

```shell
# Open firewall port for Calico
sudo ufw allow 179/tcp;

# Apply calico`s pods
kubectl apply -f calico.yaml;
```

Once a pod network has been installed, you can confirm that it is working by
checking that pod is Running in the out of `kubectl get pods --all-namespaces`.

## Add worker node

Same with the creating control plane, simply run following command. The *token*
and *hash* values are notified when installing `kube init`.

```shell
kubeadm join <control plane ip>:6443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash>
```

After running `kubeadm join`, you can check name, status, and roles. But the
default role name is empty(`<none>`), so you can set the new role name to the
new node.

```text
ubuntu@rbp4001:~$ kubectl get nodes
NAME                STATUS   ROLES           AGE     VERSION
rbp4001.sungup.io   Ready    master          3h41m   v1.17.0
rbp4002.sungup.io   Ready    <none>          20m     v1.17.0
```

To add the role name on the node, you can run this command.

```shell
kubectl label node <node name> node-role.kubernetes.io/<role name>=<any name>;
```

Also, you can remove the role name using this.

```shell
kubectl label node <node name> node-role.kubernetes.io/<role name>-;
```

## Trouble shooting

### Failed to get kubelet cgroup

If you check `systemctl status kubelet`, you can get an error text at the end
of console.

```text
Failed to get kubelets cgroup: cpu and memory cgroup hierarchy not unified. Cpu:/, memory: /system.slice/kubelet.service.
```

This fail had been issued and already fixed, but not merged in the kubelet
packages in gcr.io(?). To solve this problem, you can modify the
`/etc/systemd/system/multi-user.target.wants/kubelet.service` file.

Open that service file and add `CPUAccounting=true` and `MemoryAccounting=true`
at the `[Service]` section.

```ini
[Service]
CPUAccounting=true
MemoryAccounting=true
ExecStart=/usr/bin/kubelet
Restart=always
StartLimitInterval=0
RestartSec=10
```

### Systemd generates many spam messages

If you installed the **Calico** successfully, you can see a lot of log entries
in `/var/log/systemd`. Every 10 seconds, **systemd** generate the mount success
logs for `calico-node-<hash>` and `calico-kube-controllers-<hash>-<hash>` pods.

```text
Jan  1 20:23:41 rbp4001 systemd[5312]: run-runc-3020876e4502948946dc68223d2eb57cc1effa900e8eacffe12582b5608c03b0-runc.sirEMa.mount: Succeeded.
Jan  1 20:23:41 rbp4001 systemd[1]: run-runc-3020876e4502948946dc68223d2eb57cc1effa900e8eacffe12582b5608c03b0-runc.sirEMa.mount: Succeeded.
Jan  1 20:23:45 rbp4001 systemd[5312]: run-runc-3020876e4502948946dc68223d2eb57cc1effa900e8eacffe12582b5608c03b0-runc.176ckI.mount: Succeeded.
Jan  1 20:23:45 rbp4001 systemd[1]: run-runc-3020876e4502948946dc68223d2eb57cc1effa900e8eacffe12582b5608c03b0-runc.176ckI.mount: Succeeded.
Jan  1 20:23:51 rbp4001 systemd[5312]: run-runc-3020876e4502948946dc68223d2eb57cc1effa900e8eacffe12582b5608c03b0-runc.uQmLJA.mount: Succeeded.
Jan  1 20:23:51 rbp4001 systemd[1]: run-runc-3020876e4502948946dc68223d2eb57cc1effa900e8eacffe12582b5608c03b0-runc.uQmLJA.mount: Succeeded.
```

From the [systemd logs filled with mount unit entries if healtcheck is enabled]
article, it is very common problem running lots of Kubernetes pods. Some peoples
guess that reason for the readyness/liveness probing mechanism, also this is the
similar problem in my Raspberry Pi cluster.

I can't find the right solutions, but can apply helpful method from a comment
by *gertjanklein*. Create and open file `/etc/rsyslog.d/01-blocklist.conf` and
add following options.

```text
if $msg contains "run-runc-" and $msg contains ".mount: Succeeded." then {
    stop
}
```

After apply this config, restart rsyslog daemon.

```shell
sudo systemctl restart rsyslog;
```

After this, no more mounting logs will be sotred at `/var/log/syslog`. But,
**systemd** will store that in the journal log continuously. There is no
way to avoid storing that log. So, you should clear the journal log
periodically using **crontab**. Following string is the clearing log with
**journalctl**. **Crontab** will clear journal logs every Sunday 23:59.

```text
59 23 * * 0 journalctl -m --rotate --vacuum-time=2d
```

## Reference

- [Raspberry Pi 4 Ubuntu 19.10 cannot enable cgroup memory at bootstrap]
- [Container runtimes]
- [Installing kubeadm]
- [Creating a single control-plane cluster with kubeadm]
- [Creating a Kind Cluster With Calico Networking]
- [pod calico-node on worker nodes with 'CrashLoopBackOff']
- [Failed to get kubelets cgroup]
- [arm64 docker container contains x86_64 flexvol driver]
- [systemd logs filled with mount unit entries if healtcheck is enabled]

[Raspberry Pi 4 Ubuntu 19.10 cannot enable cgroup memory at bootstrap]: https://askubuntu.com/questions/1189480/raspberry-pi-4-ubuntu-19-10-cannot-enable-cgroup-memory-at-boostrap
[Container runtimes]: https://kubernetes.io/docs/setup/production-environment/container-runtimes
[Installing kubeadm]: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
[Creating a single control-plane cluster with kubeadm]: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/
[Creating a Kind Cluster With Calico Networking]: https://alexbrand.dev/post/creating-a-kind-cluster-with-calico-networking/
[pod calico-node on worker nodes with 'CrashLoopBackOff']: https://github.com/projectcalico/calico/issues/2720
[Failed to get kubelets cgroup]: https://stackoverflow.com/questions/57456667/failed-to-get-kubelets-cgroup
[arm64 docker container contains x86_64 flexvol driver]: https://github.com/projectcalico/pod2daemon/issues/17
[systemd logs filled with mount unit entries if healtcheck is enabled]: https://github.com/docker/for-linux/issues/679
