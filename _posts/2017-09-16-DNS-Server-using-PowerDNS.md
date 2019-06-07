---
layout: post
title: DNS Server Using PowerDNS
author-id: sungup
tags: [centos, DNS, powerdns]
---

현재 업무를 하고 있는 환경에서는 보안관련 이슈 때문에 30명 전후의 인원이
인터넷이 되지 않는 Local 개발환경에서 개발을 진행하는데요. 이때 사용하는
GitLab 이나 Redmine, 아니면 이보다 더 간단한 파일 공유 서버들을 사용하는데
있어서 IP 주소나 PC 이름으로 작업해야 하는건 여간 골치아픈 일이 아닙니다.
이때 쓸만한 방법이 내부 네트워크에서 DNS 서버를 구축해서 일괄적으로 관리하면,
OS 환경에 따라 PC 명을 못찾는 상황이나, IP를 일일히 외울 필요가 없어집니다.

단, 가장 기본적인 방법의 경우 Bind 를 사용하면 되는데, bind 의 경우 DNS를
설정할 때 일일히 서버에 접속하여 설정파일을 고쳐줘야 하는 불편함이 매우 큽니다.
이럴 때 mariadb 및 PHP 서버를 활용하여 구축하는데 요긴한게 Power DNS 라는
서비스 프로그램으로 구축시에는 관리용 Web UI를 통해 간편하게 도메인명을 관리할
수 있습니다.

## Background

우선 설치를 진행하기 전에 환경을 정의합니다.

* DNS 서버 OS: CentOS 7.X
* 내부 네트워크 주소 대역: 192.168.11.0/24
* DNS를 올릴 서버의 주소: 192.168.11.70
* DNS를 통해 연결할 서버들의 주소
  * domain: 192.168.11.70
  * www: 192.168.11.101
  * virt: 192.168.11.102
* 내부 네트워크 이름: yourdomain.com

## Install PowerDNS & Pre-requisit

우선 아래의 명령어로 순차적으로 epel, mariadb, pdns (Power DNS 패키지)를
설치합니다. 또한 DNS 서버에 대해 내부 네트워크 망의 타 서버/PC에서 접근할 수
있도록 firewalld 에서 DNS 포트를 개방합니다.

```bash
sudo yum install -y epel-release && sudo yum update -y;
sudo yum install -y mariadb mariadb-server;
sudo yum install -y pdns pdns-backend-mysql pdns-tools;
sudo firewall-cmd --zone=public --permanent --add-service=dns;
sudo firewall-cmd --reload;
```

## Setup PowerDNS

Power DNS 설치를 위해 mariadb 초기화, PowerDNS 용 DB 구축, PowerDNS의 설정 파일
수정의 순서로 셋팅을 진행합니다.

### Initialize mariadb

우선 아래의 명령으로 mariadb 서비스를 등록합니다.

```bash
systemctl enable mariadb.service
systemctl start mariadb.service
```

이후 mariadb의 DB 구성정보를 아래 명령어로 초기화 합니다. 아래 방법은 mariadb의
일반적인 초기화 방법으로 mariadb 설치 후 반드시 진행해야 합니다.

```bash
mysql_secure_installation
```

위 명령시 아래와 같은 7개의 항목에 대해 질의를 하며, 아래 내용에 따라 입력값을
설정해야 합니다.

1. **Enter current password for root**: mariadb 의 root 계정 암호 입력.
    * 초기에는 아무런 암호가 설정되어 있지 않으므로 공란으로 엔터 입력.
1. **Set root password?**: root 계정 암호 리셋 여부 확인
1. **New Password / Re-enter new password**: root 계정의 암호 입력
1. **Remove anonymous users?**: DB 상 불필요한 익명 계정들 삭제
1. **Disallow root login remotely?**: root 계정의 원격 접속 허용 여부
1. **Remove test database and access to it?**: 임시 데이터베이스 삭제 여부
1. **Reload privilege table now?**: 권한정보 런타임 갱신 여부

실행 시 아래와 같은 내용이 나오며 위의 질의내용에 맞춰 설정값을 입력합니다.

```text
# mysql_secure_installation

NOTE: RUNNING ALL PARTS OF THIS SCRIPT IS RECOMMENDED FOR ALL MariaDB
      SERVERS IN PRODUCTION USE!  PLEASE READ EACH STEP CAREFULLY!

In order to log into MariaDB to secure it, we'll need the current
password for the root user.  If you've just installed MariaDB, and
you haven't set the root password yet, the password will be blank,
so you should just press enter here.

Enter current password for root (enter for none):
OK, successfully used password, moving on...

Setting the root password ensures that nobody can log into the MariaDB
root user without the proper authorisation.

Set root password? [Y/n] y
New password:
Re-enter new password:
Password updated successfully!
Reloading privilege tables..
 ... Success!


By default, a MariaDB installation has an anonymous user, allowing anyone
to log into MariaDB without having to have a user account created for
them.  This is intended only for testing, and to make the installation
go a bit smoother.  You should remove them before moving into a
production environment.

Remove anonymous users? [Y/n] y
 ... Success!

Normally, root should only be allowed to connect from 'localhost'.  This
ensures that someone cannot guess at the root password from the network.

Disallow root login remotely? [Y/n] y
 ... Success!

By default, MariaDB comes with a database named 'test' that anyone can
access.  This is also intended only for testing, and should be removed
before moving into a production environment.

Remove test database and access to it? [Y/n] y
 - Dropping test database...
 ... Success!
 - Removing privileges on test database...
 ... Success!

Reloading the privilege tables will ensure that all changes made so far
will take effect immediately.

Reload privilege tables now? [Y/n] y
 ... Success!

Cleaning up...

All done!  If you've completed all of the above steps, your MariaDB
installation should now be secure.

Thanks for using MariaDB!
```

### Create DB and tables for PowerDNS

데이터 베이스 생성을 위해 아래 링크의 SQL 스크립트 파일을 다운받습니다.
(해당 내용은 Hatena 블로그의 sig9 유저님의 스크립트를 활용했습니다.)

* [pdns_db_create.sql]({{ site.baseurl }}/downloads/2017-09-16-PowerDNS/pdns_db_create.sql)

이후 PowerDNS를 위한 DB 생성을 위하 mariadb 에 로그인하고, 아래 설정에 맞춰
DB 를 생성하고 앞서 받은 SQL 파일을 사용하여 테이블들을 생성합니다. 단, DB
Name/User/Password 는 입맛에 맞게 **반드시** 수정해서 사용합니다.

```bash
mysql -u root -p mysql
```

#### 생성할 PowerDNS DB의 정보

|Name    |Value   |
|:------:|:------:|
|DB Name |powerdns|
|DB User |powerdns|
|Password|password|

#### 실행 SQL 명령어

```sql
CREATE DATABASE powerdns;
GRANT ALL ON powerdns.* TO 'powerdns'@'localhost' IDENTIFIED BY 'password';
FLUSH PRIVILEGES;
SOURCE pdns_db_create.sql;
exit;
```

### Configure PowerDNS

PowerDNS 에서 앞서 구축한 DB를 활용할 수 있도록, /etc/pdns/pdns.conf 파일을
아래와 같이 수정합니다. 아래 내용은 수정 전후의 diff 내용입니다.

```diff
diff -urN a/pdns.conf b/pdns.conf
--- a/pdns.conf	2017-09-16 13:18:29.841720869 +0900
+++ b/pdns.conf	2017-09-16 13:21:48.370662061 +0900
@@ -21,6 +21,7 @@
 # allow-recursion	List of subnets that are allowed to recurse
 #
 # allow-recursion=0.0.0.0/0
+allow-recursion=0.0.0.0/0

 #################################
 # also-notify	When notifying a domain, also notify these nameservers
@@ -216,6 +217,12 @@
 # launch	Which backends to launch and order to query them in
 #
 # launch=
+launch=gmysql
+gmysql-host=localhost
+gmysql-user=powerdns
+gmysql-password=password
+gmysql-dbname=powerdns
+gmysql-dnssec=yes

 #################################
 # load-modules	Load this module - supply absolute or relative path
@@ -251,21 +258,25 @@
 # log-dns-details	If PDNS should log DNS non-erroneous details
 #
 # log-dns-details=no
+log-dns-details=on

 #################################
 # log-dns-queries	If PDNS should log all incoming DNS queries
 #
 # log-dns-queries=no
+log-dns-queries=yes

 #################################
 # logging-facility	Log under a specific facility
 #
 # logging-facility=
+logging-facility=0

 #################################
 # loglevel	Amount of logging. Higher is more. Do not set below 3
 #
 # loglevel=4
+loglevel=10

 #################################
 # lua-prequery-script	Lua script with prequery handler
```

위와 같이 PowerDNS 설정이 완료된 경우 PowerDNS 서비스를 등록하고 시작합니다.

```bash
systemctl enable pdns.service
systemctl start pdns.service
```

## Install PowerAdmin

위와 같이 셋업한 경우 DB에 등록하는 방식으로 DNS를 설정할 수 있습니다. 단, 이
경우에는 매번 mariadb client 툴을 사용해서 SQL 문으로 등록해야 하므로, 좀더
편한 방법으로 관리할 수 있도록 PowerAdmin 이라고 하는 웹기반 관리툴을
설치합니다.

PowerAdmin 은 PHP로 만들어진 PowerDNS용 관리툴로 본 Article 에서는 APM
(Apache + PHP + MySQL) 환경을 기반으로 설정합니다.

### Install Prerequisit for PowerAdmin

PowerAdmin 설치를 위해 wget 과 unzip, 웹서비스 기동을 위한 http, php,
php-mysql 을 설치하고, 해당 서비스들을 기동합니다. 또한 관리자가 웹서버에
접근할 수 있도록 firewalld 에서 http 포트를 개방합니다.

```bash
sudo yum install -y httpd php php-mysql php-mcrypt wget unzip;
sudo systemctl enable httpd;
sudo systemctl start httpd;
sudo firewall-cmd --zone=public --permanent --add-service=http;
sudo firewall-cmd --reload;
```

### Install PowerAdmin

아래의 GitHub의 링크에서 master 버전의 파일을 다운받습니다. 다운로드 받은
파일의 압축을 분 다음, httpd 의 웹 홈디렉토리인 /var/www/html에 해당 파일들을
설치합니다.

* [PowerAdmin GitHub package](https://github.com/poweradmin/poweradmin/archive/master.zip "Master version PowerAdmin GitHub link")

> **주의**: CentOS 7 에서 해당 파일을 복사가 아니라 이동을 한 경우 httpd 에서 권한의 문제로 파일을 열 수 없는 문제가 발생할 수 있습니다. 내부의 Hidden Permission 권한에 의해 발생하는 문제로 예상되며, 이를 해결하기 위해서는 mv 가 아니라 cp 를 통해 파일을 <u>**복사**</u> 해야 올바로 웹페이지에 접근할 수 있는 권한이 생깁니다.

```bash
wget https://github.com/poweradmin/poweradmin/archive/master.zip;
sudo cp -r poweradmin-master /var/www/html/poweradmin;
```

### PowerAdmin 설정

PowerAdmin의 설정은 웹페이지에서 진행되며, 내용 입력시 복잡한 사항이 많지 않아
입력해야할 내용들을 아래에 순서대로 설명합니다.

우선 PowerAdmin의 설정을 위해 웹브라우저에서
http://<웹서버주소>/poweradmin/install/ 에 접근합니다.

#### Installation Step 1

설치시 진행하는 언어를 선택합니다. 언어는 영어, 네덜란드어, 독일어, 일본어,
폴란드어, 프랑스어, 노르웨이어를 지원합니다.

#### Installation Step 2

설치를 위한 사전 설명이 있습니다. 다음단계로 이동합니다.

#### Installation Step 3

PowerAdmin에서 mariadb 서버를 연결하기 위한 정보를 입력합니다.

1. **UserAdmin**: PowerDNS DB 에 접근하기 위한 계정 이름
1. **Password**: PowerDNS DB 에 접근하기 위한 계정의 암호
1. **Database type**: PowerDNS DB 가 설치된 데이터베이스 엔진
  - MySQL (mariadb), PostgreSQL, SQLite 를 선택할 수 있으며, 이 경우에는
    MySQL 을 선택합니다.
1. **Hostname**: PowerDNS DB 가 설치된 서버의 주소
  - 같은 서버내에 있기 때문에 localhost 입력합니다..
1. **DB Port**: 앞서 선택한 mariadb 와 통신하기 위한 네트워크 Port (3306)
1. **Database**: PowerDNS 의 DB 명
1. **DB charset**: DB 에 사용되는 characterset. 공란의 경우 DB 기본값
1. **DB collation**: DB 에서 character 를 비교하기 위한 rule. 공란의 경우 DB 기본값
1. **Poweradmin administrator password**: PowerAdmin의 기본 관리자 계정에 대한 암호

#### Installation Step 4

PowerAdmin에서 앞선 PowerDNS 의 접근정보와 다른 PowerAdmin 관리 툴이 DB 에
접근하기 위한 정보를 입력합니다. PowerDNS 가 아닌 PowerAdmin 의 불필요한 DB
접근을 방지하기 위해 PowerDNS 서비스와 다른 PowerAdmin 의 독립적인 DB 계정을
입력할 수 있습니다.

1. **Username**: PowerAdmin 에서 DB에 접근하기 위한 DB 계정 명.
1. **Password**: PowerAdmin 에서 DB에 접근하기 위한 DB 계정의 암호.
1. **Hostmaster**: SOA 레코드를 만들때 없는 경우 기본값으로 선택되는 값.
    * 일반적으로 "hostmaster.yourdomain.com" 과 같은 방식으로 선택됩니다.
1. **Primary namesaerver**: 템플릿을 만들때 주 네임서버로 설정되는 값.
1. **Secondary nameserver**: 템플릿을 만들때 보조 네임서버로 설정되는값.
    * Primary/Secondary nameserver 는 일반적으로 아래와 같이 설정됩니다.
        * **Primary nameserver**: ns1.yourdomain.com
        * **Secondary nameserver**: ns2.yourdomain.com

#### Installation Step 5

앞선 단계에서 PowerAdmin을 위한 계정을 DB 상에서 생성하고 권한 할당에 대한
작업사항을 알려줍니다. 서버에서 mariadb에 접근한 다음 화면에 표시된 SQL 문을
입력하여 권한을 설정합니다. 단, PowerDNS 와 동일한 계정을 Step 4에 설정한
경우 권한 설정을 위한 SQL문 작업이 필요 없습니다.

#### Installation Step 6

PowerAdmin에서 설정을 위한 값이 활용될 수 있도록, PowerAdmin의 inc 디렉토리에
있는 config-me.inc.php 파일을 config.inc.php로 복사하여 설정값을 수정합니다.

```bash
cd /var/www/html/poweradmin/inc;
cp config-me.inc.php config.inc.php;
```

변경해야 하는 내용은 아래의 Diff 내용을 참고하셔서 수정합니다.

```diff
--- config-me.inc.php   2017-09-17 10:16:03.568970503 +0900
+++ config.inc.php      2017-09-17 10:50:16.573658249 +0900
@@ -15,12 +15,12 @@
 // Better description of available configuration settings you can find here:
 // <https://github.com/poweradmin/poweradmin/wiki/Configuration-File>
 // Database settings
-$db_host = '';
-$db_port = '';
-$db_user = '';
-$db_pass = '';
-$db_name = '';
-$db_type = '';
+$db_host = 'localhost';
+$db_port = '3306';
+$db_user = '<Step 4 에서 입력한 DB 계정 명>';
+$db_pass = '<Step 4 에서 입력한 DB 계정 암호>';
+$db_name = 'powerdns';
+$db_type = 'mysql';
 //$db_charset = 'latin1'; // or utf8
 //$db_file             = '';           # used only for SQLite, provide full path to database file
 //$db_debug            = false;        # show all SQL queries
@@ -28,7 +28,7 @@

 // Security settings
 // This should be changed upon install
-$session_key = 'p0w3r4dm1n';
+$session_key = '<웹 화면상에 나온 Session Key 정보>';
 $password_encryption = 'md5'; // md5, md5salt or bcrypt
 //$password_encryption_cost = 12; // needed for bcrypt

@@ -42,9 +42,9 @@
 $iface_add_reverse_record = true;

 // Predefined DNS settings
-$dns_hostmaster = '';
-$dns_ns1 = '';
-$dns_ns2 = '';
+$dns_hostmaster = 'hostmaster.yourdomain.com';
+$dns_ns1 = 'ns1.yourdomain.com';
+$dns_ns2 = 'ns2.yourdomain.com';
 $dns_ttl = 86400;
 $dns_fancy = false;
 $dns_strict_tld_check = false;
```

#### Installation Step 7

최종적으로 PowerAdmin 설정을 종료하기 위한 마지막 작업을 진행합니다. 아래와
같이 2가지 작업을 진행합니다.

1. install/htaccess.dist 파일을 poweradmin 의 루트 디렉토리에 .htaccess 파일로 복사
1. powerdns 의 install 디렉토리를 삭제 하여 이후 추가 설정하는 것을 방지.

```bash
cd /var/www/html/poweradmin;
sudo cp install/htaccess.dist .htaccess;
sudo rm -rf install;
```

기본적으로 CentOS 7은 mod-rewrite가 활성화 되어 있지만 다른 설정의 문제로 비
활성화 되어있을 가능성도 있습니다. 따라서 httpd 서버에서 mod\_rewrite 가 활성화
안된 경우 mod\_rewrite를 활성화 합니다. 따라서 CentOS 7 기준으로 
/etc/httpd/conf.modules.d/00-base.conf에서 아래 정보가 있는 줄을 확인하고,
주석처리가 된 경우 주석을 해제합니다.

* **설정 줄 내용**: LoadModule reqrite_module modules/mod_rewrite.so

## DNS 환경 설정

PowerAdmin에서 DNS 환경을 구축합니다. 기본적인 관리자 계정의 아이디는 admin
이며, step 3에서 설정한 암호를 통해 로그인 할 수 있습니다.

우선, 로그인한 첫 화면이나 탑메뉴의 **Add master zone** 을 클릭하여 master zone
정보를 추가합니다. 이때, 사용하고자 하는 도메인을 **"Zone name"** 항목에 입력한
다음 Add zone 을 클릭합니다.

이후 탑메뉴에서 List zones 를 선택한 다음 추가된 도메인의 리스트를 확인합니다.
이때 리스트의 왼쪽 메뉴에서 수정 아이콘을 선택하며, 이동한 페이지에서 앞서
추가한 도메인에 대한 정보를 조회/수정이 가능하며, zone 내에서 활용할 레코드들을
추가할 수 있습니다.

가장 앞에서 가정한 www 와 virt 의 A recorde 에 대해 중간에 위치한 레코드
입력창에 아래 항목에 맞춰 입력합니다. 단, 한번에 하나씩 입력이 가능하며, zone
화면에서 입력한 다음 추가적인 레코드는 동일한 방법으로 계속 입력합니다.

* **www**
  * **Name**: www
  * **Type**: A
  * **Content**: 192.168.11.101 *(IP Address)*
* **virt**
  * **Name**: virt
  * **Type**: A
  * **Content**: 192.168.11.102 *(IP Address)*

## DNS 서버 테스트

### Install bind-utils and settings

DNS 설정을 확인을 위한 dig 프로그램 설치를 위해 bind-utils 를 설치합니다.

```bash
sudo yum install -y bind-utils;
```

만약 신규 설정한 DNS 서버를 기본 DNS 서버로 설정하지 않은 경우 /etc/resonv.conf
파일에 `nameserver <DNS 서버주소>` 를 추가하여 앞서 설치한 PowerDNS를 기본 DNS
서버로 사용하도록 변경합니다.

설정이 완료되면, 아래와 같은 명령으로 설정한 Domain 주소의 정보를 조회합니다.
아래는 성공적으로 셋팅이 된경우 나오는 내용입니다.

```text
# dig www.yourdomain.com A @127.0.0.1

; <<>> DiG 9.9.4-RedHat-9.9.4-51.el7 <<>> www.yourdomain.com A @127.0.0.1
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 19025
;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1680
;; QUESTION SECTION:
;www.yourdomain.com.		IN	A

;; ANSWER SECTION:
www.yourdomain.com.	86400	IN	A	192.168.11.101

;; Query time: 1 msec
;; SERVER: 127.0.0.1#53(127.0.0.1)
;; WHEN: Sun Sep 17 11:37:47 KST 2017
;; MSG SIZE  rcvd: 63

```

## References

* [CentOS7 に PowerDNS をインストールする by sig9](http://sig9.hatenablog.com/entry/2016/12/20/120000 "CentOS7 に PowerDNS をインストールする")
* [[install] powerdns 설치하기 by wapj2000](http://gyus.me/?p=560 "[install] powerdns 설치하기")
