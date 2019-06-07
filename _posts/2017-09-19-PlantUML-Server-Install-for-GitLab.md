---
layout: post
title: Plant UML + GitLab 연동
author-id: sungup
tags: [git, gitlab, uml]
---

GitLab 내에서 UML을 통한 의견 공유가 필요한 경우 이를 간단히 표현하기 위한
인터페이스가 필요했습니다. 이때, PlantUML 이라는 Java 기반의 Component를
발견했고, 이를 WebApp형태로 올리면 URL을 통해서 GitLab과 PlantUML을 연결할
수 있는 GitLab의 내부 기능을 확인했습니다.

기본 환경의 경우 이전 올린 내용을 바탕으로 CentOS 7을 기반으로 하여 구성했습니다.

## Install and configure the necessary dependencies

PlantUML을 설치해서 진행하는데 있어 기본적으로 아래와 같은 패키지들을 설치

```bash
sudo yum install -y graphviz tomcat tomcat-webapps;
```

이때 사용자가 직접 git repository에서 clone을 해서 컴파일 하는 경우 아래
패키지들을 추가 설치

```bash
sudo yum install -y git java-1.8.0-openjdk-devel maven;
```

## Package Download

### Git clone & user build

GitHub 에 있는 PlantUML의 server source를 다운받은 다음, Maven을 활용해
컴파일 진행

```bash
git clone https://github.com/plantuml/plantuml-server.git;
cd plantuml-server;
mvn package;
```

이때 생성된 plantuml.war는 target/plantuml.war 에 저장됨

### Download J2EE war package

아래 Link 에서 plantuml의 J2EE 패키지를 다운로드해서 저장

* [PlantUML J2EE Package](http://sourceforge.net/projects/plantuml/files/plantuml.war/download)

### Copy J2EE package and start tomcat

생성 혹은 다운로드 받은 plantuml.war 파일을 tomcat의 webapps 디렉토리에 복사한
다음 tomcat 서버를 등록하여 실행

```bash
sudo cp plantuml.war /var/lib/tomcat/webapps/plantuml.war;
sudo chown tomcat:tomcat /var/lib/tomcat/webapps/plantuml.war;
sudo systemctl enable tomcat;
sudo systemctl start tomcat;
```

이때 외부에서 접근하는데 있어서 firewall의 문제로 접근이 안되는 경우
firewall에서 tomcat의 접속 포트를 개방해야 함. 이때 tomcat의 기본 접속포트는
8080고, 해당 포트번호는 /etc/tomcat/server.xml에서 변경 가능

```bash
sudo firewall-cmd --permanent --add-port=8080/tcp;
sudo firewall-cmd --reload;
```

상기와 같이 설정을 마친경우 **http://localhost:8080/plantuml** 로 접근하여
동작여부를 확인

## GitLab Settings

GitLab에서 admin 권한이 있을 시에만 적용이 가능. GitLab 의 admin 권한이 있으면,
**"Admin area -> Settings"** 에 PlantUML 항목이 존재하며, 여기서 **"Enable
PlantUML"** 을 활성화 하고 **"PlantUML URL"** 을 위의 웹 주소 (단, localhost
가 아닌 실제 서버의 도메인 혹은 IP 주소) 로 설정하여 저장하면 활용이 가능하다.

## References

* [PlantUML Homepage](http://plantuml.com "PlantUML")
* [PlantUML & GitLab](https://gitlab.com/gitlab-org/gitlab-ce/blob/master/doc/administration/integration/plantuml.md "PlantUML & GitLab")
