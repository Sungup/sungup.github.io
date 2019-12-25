---
layout: post
title: Install Kubernetes on Raspberry Pi 4
author-id: sungup
#feature-img: "assets/img/posts/2019-12-23-prometheus-and-grafana-banner.jpeg"
tags: [ubuntu, raspberry pi 4, kubernetes, docker]
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
CRI-O on the Ubuntu 18.10(cosmit)/19.10(eoan), you must modify the repository
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
ubuntu@rbp4001:~$ kubectl get nodes -o wide
NAME                STATUS   ROLES    AGE     VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE       KERNEL-VERSION      CONTAINER-RUNTIME
<node name 1>       Ready    master   3h23m   v1.17.0   <internal ip> <none>        Ubuntu 19.10   5.3.0-1014-raspi2   cri-o://1.15.3-dev
<node name 2>       Ready    <none>   3m      v1.17.0   <internal ip> <none>        Ubuntu 19.10   5.3.0-1014-raspi2   cri-o://1.15.3-dev
```

To add the role name on the node, you can run following command.

```shell
kubectl label node <node name> node-role.kubernetes.io/<role name>=<any name>;
```

Also, you can remove role name using this.

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

## Reference

- [Raspberry Pi 4 Ubuntu 19.10 cannot enable cgroup memory at bootstrap](https://askubuntu.com/questions/1189480/raspberry-pi-4-ubuntu-19-10-cannot-enable-cgroup-memory-at-boostrap)
- [Container runtimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes)
- [Installing kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
- [Failed to get kubelets cgroup](https://stackoverflow.com/questions/57456667/failed-to-get-kubelets-cgroup)
