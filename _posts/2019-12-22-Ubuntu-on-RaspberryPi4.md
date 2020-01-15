---
layout: post
title: Ubuntu on Raspberry Pi 4
author-id: sungup
feature-img: "assets/img/posts/2019-12-22-ubuntu-on-rasbpi4.jpeg"
tags: [ubuntu, raspberry pi 4]
date: 2019-12-22 19:13:59
---

Raspberry Pi 4 + Ubuntu로 Cluster를 구축하면서 필요한 내용들을 정리하는 문서입니다. 사용하는 라즈베리파이는
**Raspberry Pi 4 4GB** 버전이며, 최신 Ubuntu Server 이미지를 활용, 네트워크는 WiFi 환경을 바탕으로 하고
있습니다. OS 이미지 설치 및 HDMI 활성화 부분의 경우 x86 서버와 동일한 구조이기 때문에 x86 서버에서도 활용 가능합니다.

기본 환경 구축은 MacBook 2015 Mid 에서 진행했습니다.

## Install OS Image

### Prerequisite

* Raspberry Pi 4 Board
* Micro SD Card (< 32GB) & Card Reader
* Ubuntu Server Image ([Ubuntu 19.10.1 Link](http://cdimage.ubuntu.com/releases/19.10.1/release/ubuntu-19.10.1-preinstalled-server-arm64+raspi3.img.xz?_ga=2.35675104.119130526.1577003050-1913019912.1577003050))
* Monitor & Micro HDMI Cable
* USB keyboard

### Install Sequence

1. Download Ubuntu Server Image
2. Insert SD Card using SD Card Reader.
3. Check the SD Card's drive address using `diskutil list`
4. Unmount SD Card using `diskutil unmountDisk <drive address>`
5. Copy server image to SD card using following command

```shell
sudo sh -c 'gunzip -c <image file path> | sudo dd of=<drive address> bs=32m'
```

Not finished!!!

### Enable HDMI

Re-insert SD card image. You can find `system-boot` drive on you mac/PC. Move
that drive using terminal, open `usercfg.txt` file and add following settings.

```text
hdmi_force_hotplug=1
hdmi_driver=2
```

If you want no more graphical output after full install, simply comment out
the two lines.

Anyway, unmount and insert SD card into Raspberry Pi 4 after setup HDMI output
complete. After boot the Raspberry Pi, you can login using `ubuntu/ubuntu`
account ID and password. At the first login, you should change the password
for the security reason.

## Change hostname

Basically ubuntu server image set the hostname automatically at boot time using
`cloud-init` process. So, if you want to change the hostname permanently,
change the config file in `/etc/cloud/cloud.cfg`.

```yaml
# preserve_hostname: false # Comment out or delete default value
preserve_hostname: true
```

Change that configuration value, reboot raspberry pi and change host name using
following command.

```shell
sudo hostnamectl set-hostname <host name you want>;
```

## Change timezone

After install os, basic timezone has been set to UTC. If you want to change
timezone, run the following command.

```shell
sudo timedatectl set-timezone <your time zone>;
```

In my case, I ran `sudo timedatectl set-timezone Asia/Seoul`.

## WIFI Setup (netplan)

I want to WIFI network as default network interface because my home router
don't have enough port. So, if you want connect WIFI at the boot-up time like
me, you should change the `netplan` configuration file.

First, open the `/etc/netplan/50-cloud-init.yaml` YAML file. Only ethernet
config is stored only in the original configuration file. So, add the `wifis`
field and setup the configuration values like this.

```yaml
network:
    version: 2
    ethernets:
        eth0:
            dhcp4: true
            optional: true

    # WIFI configurations on boot timing.
    wifis:
        wlan0:
            dhcp4: true
            optional: true
            nameservers:
                addresses: [<your DNS server list>]
            access-points:
                "<your WIFI SSID>":
                    password: "<your WIFI password>"
```

After store the YAML files, you should apply the settings using following
commands.

```shell
sudo netplan generate;
sudo netplan apply;
```

### Trouble shooting

#### WIFI channel issue

Sometimes, Raspberry Pi can't connect the WIFI network at all time. I don't
know the exact reason about that, but expect the WIFI's channel is one of that
problem from some articles.

In my case, I changed the management channel of the 2.4GHz and 5GHz to channel
3 and channel 36 in my ASUS RT-AC58U router. Currently this solution is good
enough for me, but is not best solution.

If I found the better solution for this channel issue, I'll add more detail
solution in this article.

#### Clear journalctl old logs

Raspberry Pi only support < 32GB SD card because of the boot restriction. So,
there is no enough volume space to run server. In my case, the `journalctl`
logs consume many space of root partition because of the network warning and
errors. So, I has set the periodic cleaning up journal logs every Sunday
night.

To register the cleaning job every Monday, run the `crontab` as root
priviliges.

```shell
sudo crontab -e;
```

And add the following entry using text editor. This command will erase old logs
older than 2 days.

```text
59 23 * * 0 journalctl -m --rotate --vacuum-time=2d
59 23 * * 0 find /var/log -name "*.gz" -type f -mtime +2 -exec rm -f {} \;
59 23 * * 0 find /var/log -name "*.log.[0-9]*" -type f -mtime +2 -exec rm -f {} \;
```

## Reference

* [Install Ubuntu Server on a Raspberry Pi 2, 3 or 4](https://ubuntu.com/download/raspberry-pi)
* [Raspberry Pi HDMI display not working, how to solve it?](https://howtoraspberrypi.com/raspberry-pi-hdmi-not-working/)
* [How to Change Hostname on Ubuntu 18.04](https://linuxize.com/post/how-to-change-hostname-on-ubuntu-18-04/)
* [Wifi channel 7-10 (between 6 and 11) don't work (RPI3B)](https://forum.openwrt.org/t/wifi-channels-7-10-between-6-and-11-dont-work-rpi3b/40342)
* [How to clear journalctl](https://unix.stackexchange.com/questions/139513/how-to-clear-journalctl)
