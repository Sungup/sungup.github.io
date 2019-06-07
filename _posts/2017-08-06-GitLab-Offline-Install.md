---
layout: post
title: GitLab Offline Install on CentOS 7
author-id: sungup
tags: [centos, gitlab]
---

특정 개발망에서는 외부와의 네트워크 문제로 개발을 위한 Git 서버 구축의 어려움이
있습니다. 단, 이 경우 Offline Install이 필요하지만, 기본적인 Git 소스서버들의
경우 Django나 Rails와 같은 Web Framework를 사용하는 경우가 많은 관계로
Offline 설치가 가능한지 여부를 확인해야 할 필요가 있습니다. 이때 GitLab의 경우
RPM/DEB 버전을 통해서 AIO 패키지를 제공해서 설치방법이 매우 간편하고, 업데이트
시 발생하기 쉬운 Package 간의 충돌에 대해 상당히 자유로이 안정적으로 동작함을
확인했습니다.

## Install Environment

* OS: CentOS 7.3 Latest Update (2017/08/06)
* Package: GitLab Community Edition 9.4.3 (gitlab-ce-9.4.3-ce.0.el7.x86_64.rpm)

## Installation Process

### Install and configure the necessary dependencies

CentOS에서 아래의 명령으로 SSH 서버를 열고, HTTP 및 SSH로 접근이 가능하도록
Firewall에서 Port를 개방함.

```bash
sudo yum install -y curl policycoreutils openssh-server openssh-clients
sudo systemctl enable sshd
sudo systemctl start sshd
sudo yum install postfix
sudo systemctl enable postfix
sudo systemctl start postfix
sudo firewall-cmd --permanent --add-service=http
sudo systemctl reload firewalld
```

### Install the package

기본적으로 GitLab에서는 별도의 YUM Repository 서버를 통해 Update를 제공함. 단,
이번 Article은 외부 네트워크와 단절된 상황을 가정하였으므로, 해당 Repository에
직접적인 접근이 불가능하며 GitLab에서 제공하는 RPM 패키지를 다운받아 설치함.

* 설치파일 경로: https://packages.gitlab.com/gitlab/gitlab-ce/

상기 주소를 통해 접근이 가능하며, OS 별 RPM 및 DEB 패키니 파일들이 존재.

다운로드 받은 파일의 크기는 약 350MB 전후로, 다운로드 받은 파일은 아래 명령어로
GitLab을 설치. 이때 파일명의 XXX는 다운로드 받은 GitLab의 버전임.

```bash
sudo yum localinstall -y gitlab-ce-XXX.rpm
```

### Configure and start GitLab

하기의 경로로 가서 GitLab에 대한 설정을 확인한 다음 GitLab의 초기화 및 설치를
진행.

* 설장파일 경로: /etc/gitlab/gitlab.rb

```bash
sudo gitlab-ctl reconfigure
```

### Browse to the hostname and login

설치한 서버를 Web Browser에서 접근하면, 관리자의 암호 초기화 화면이 나오며.
해당 화면에서 원하는 관리자의 암호를 설정하면 다시 로그인 화면으로 이동.

이후 관리자 화면으로 들어가기 위해 **root** 계정에 대해 앞서 초기화 한 암호를
통해 로그인 하며, 이후 관리자의 이름을 원하는 형태로 변경할 수 있음.

## References

* [GitLab Installation](https://about.gitlab.com/installation/#centos-7 "GitLab Installation for CentOS 7")
