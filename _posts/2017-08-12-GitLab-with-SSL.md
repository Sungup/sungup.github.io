---
layout: post
title: GitLab with SSL on CentOS 7
author-id: sungup
tags: [centos, gitlab]
---

앞선 CentOS 7의 설치법에서 추가로 확장된 버전이며, SSL 을 쓰기 위해서는 별도의
인증서를 받아서 관리하는 시스템이 필요한데 이때 사용되는게, Let's encrypt
입니다. Open Source 로 관리되는 Certification 툴로, 리눅스 제단에서 협력해서
진행하고 있는 프로젝트 입니다. 이번에는 GitLab RPM 설치 후 let's encrypt 를
활용해서 SSL 로 프로젝트를 관리하는 환경을 소개합니다.

## Install Environment

* OS: CentOS 7.3 Latest Update (2017/08/06)
* Package: GitLab Community Edition 9.4.3 (gitlab-ce-9.4.3-ce.0.el7.x86_64.rpm)

## GitLab Installation Process

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

```bash
curl -sS https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.rpm.sh | sudo bash
```

만약 Pipe를 활용한 스크립트 설치가 불가능 한 경우 설치에 필요한 RPM 파일들을
다운로드 받아서 설치할 수 있음. GitLab 설치를 위한 다운로드 경로는 아래와
같으며, 각 OS 별 설치에 필요한 RPM 및 DEB 파일들이 존재함.

* 설치파일 경로: https://packages.gitlab.com/gitlab/gitlab-ce/

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

## GitLab + Let's Encrypt

GitLab의 보안을 위해 SSL을 사용할 필요가 있는 경우 이를 위해 인증서를 발급받아
사용해야 하는데, 일반적인 방법인 Self Signing 기법으로는 브라우저 상에서 인증
실패로 인해 접근이 차단됨. 보통 인증기관에서 발행하는 인증서의 경우 고가의 인증
비용을 제공해야 하지만, 일부 무료 인증기관도 존재하기 때문에 이를 통해서
HTTPS를 통한 웹 서비스 구축이 가능해짐.

### Install EPEL and certbot

Let's Encrypt를 통핸 인증 시스템 구축의 경우 certbot 이라는 프로그램을 통해
구축함. 이때 기본 CentOS의 패키지에서는 지원하지 않지만, EPEL에서는 certbot
패키지를 제공하기 때문에 EPEL을 설치한 다음 certbot을 설치.

```bash
sudo yum install -y epel-release
sudo yum install -y certbot
```

### Prepare GitLab for Let's Encrypt

E-Mail을 통한 인증처리 대신, Let's Encrypt는 파일을 사용한 인증처리를 함.
Certbot은 인증 파일을 서버상에 추가 한다음, Let's Encrypt에서 해당 파일을
원격지에서 접근하며, 이후 인증 파일에 대한 검증이 완료되면, 인증서 파일을
생성해 전송받아 설치함. certbot은 이러한 작업을 자동으로 처리해주지만, GitLab을
활용한 경우 일반적인 Let's Encrypt를 활용한 경우와 다르기 때문에 일부 작업을
수작업으로 처리할 필요가 존재.

우선, Let's Encrypt에서 처리해야 할 인증 파일을 저장할 디렉토리와 GitLab에서
이를 연동하기 위한 추가 설정파일을 저장할 디렉토리를 생성.

```bash
sudo mkdir -p /opt/letsencrypt/var/www
sudo mkdir -p /opt/letsencrypt/etc
```

다음, Let's Encrypt 시스템에서 /.well-known에 대한 요청을 처리하기 위해
GitLab의 Nginx에서 해당 경로에 대한 리다이렉션 처리에 대한 설정을 추가. 우선
앞서 만든 /opt/letsencrypt/etc 디렉토리에 nginx.conf 파일을 생성한 다음 아래
내용을 추가함.

```text
location ^~ /.well-known {
  root /opt/letsencrypt/var/www;
}
```

생성한 파일에 대해 Nginx에서 설정을 불러올수 있도록 /etc/gitlab/gitlab.rb에서
nginx['custom_gitlab_server_config'] 항목을 찾은 다음, 주석을 풀고 아래와 같이
변경함.

```ruby
nginx['custom_gitlab_server_config'] = "include /opt/letsencrypt/etc/nginx.conf;"
```

이후 GitLab의 Nginx에서 변경사항 적용을 위해 아래의 커맨드로 다시 재설정.

```bash
sudo gitlab-ctl reconfigure
```

### Request the Let's Encrypt certificate

아래의 명령어를 통해 인증처리를 진행. 명령어 처리시 발생하는 입력정보와 Terms
of Service 에 대해 입력.

```bash
sudo certbot certonly --webroot --web-path=/opt/letsencrypt/var/www -d yourdomain.com -m your@email.com
```

단, 인증처리 중 처리 실패가 있을 수 있는데,

1. 최초 인증처리이기 때문에, E-Mail을 통한 사용자 확인을 해야 하는 경우.
2. hostname과 domain 이 일치해서 Internet IP가 아닌 Local IP로 접근한 경우.
3. 과도한 인증 요청으로 Let's Encrypt에서 인증처리를 막아버린 경우.

등이 있음. 1번은 앞서 certbot 실행시 입력한 E-Mail 주소로 사용자 확인 메일이
오며 해당 메일로 사용자 확인을 진행함. 2번은 `/etc/hosts` 파일에 해당 도메인이
입력된 경우 해당 부분 삭제. 3번은 1~2 시간 이후에 다시 진행.

이후 인증처리가 완료 시 Let's Encrypt를 통해 인증서 발급 작업을 마칠 수 있음.

### Configure GitLab with the new certificates

이제 인증서는 /etc/letsencrypt/liv/yourdomain.com에 저장되어 있으며, 이를
GitLab에서 활용하도록 /etc/gitlab/gitlab.rb 내부의 설정을 변경. 우선 GitLab의
기본 접속을 HTTPS로 하기 위해 external_url의 설정값을 변경.

```ruby
external_url 'https://yourdomain.com'
```

HTTP 프로토콜 요청을 HTTPS로 리다이렉션 하기 위해
nginx['redirect_http_to_https']의 주석을 풀고 설정값을 true로 변경.

```ruby
nginx['redirect_http_to_https'] = true
```

마지막으로 Let's Encrypt에서 받은 인증서 파일을 연결하기 위해
nginx['ssl_certificate'] 및 nginx['ssl_certificate_key']의 주석을 풀고 아래와
같이 변경.

```ruby
nginx['ssl_certificate'] = "/etc/letsencrypt/live/yourdomain.com/fullchain.pem"
nginx['ssl_certificate_key'] = "/etc/letsencrypt/live/yourdomain.com/privkey.pem"
```

### Update the firewall and restart GitLab

이제 HTTPS로 접근이 가능하도록 443번 포트를 개방.

```bash
sudo firewall-cmd --permanent --add-service=https
sudo systemctl reload firewalld
sudo gitlab-ctl reconfigure
```

### Certificate renewal

Let's Encrypt에서 발행한 인증서는 최대 90일만 유효하며, 인증서 유효기간이
지나게 된 경우 certbot이 재인증 하도록 root 계정의 crontab으로 자동화 함.

```bash
sudo crontab -u root -e
```

crontab의 입력/수정 창이 열리면 아래와 같이 1달에 한번 Renew를 하도록 변경.

```text
0 2 1 * * /usr/bin/certbot renew --quiet --renew-hook "/usr/bin/gitlab-ctl restart nginx"
```

## References

* [GitLab Installation](https://about.gitlab.com/installation/#centos-7 "GitLab Installation for CentOS 7")
* [How to secure Gitlab with Let's Encrypt on CentOS 7](https://kb.yourwebhoster.eu/knowledge-base/secure-gitlab-with-lets-encrypt-on-centos-7/ "How to secure Gitlab with Let's Encript on CentOS 7")
* [GitLab Omnibus package の SSL 証明書を Let's Encrypt で取得する](http://qiita.com/yuuAn/items/09a434d3f6cffa31101e "GitLab Omnibus package の SSL 証明書を Let's Encrypt で取得する")