---
layout: post
title: Ceph Self Study 01
author-id: sungup
#feature-img: "assets/img/posts/2019-12-23-prometheus-and-grafana-banner.jpeg"
tags: [Ceph, Object Storage]
date: 2020-02-02 00:00:00
---

I'm sorry, these contents are Korean only!!

K8s 관련 셀프 스터디를 진행중이었지만, 현재 업무 관련 Follow-up에 관련 배경지식이 부족한 부분이 있어 Ceph관련 배경지식
확보를 위해 Ceph 관련 스터디 내용을 먼저 진행 합니다.

## Introducing Ceph Storage

### Intro

#### Ceph의 특징

Open, Scalable, Distributed Nature

- 최신의 Scalable, scale-out infrastructure에 잘 어울림.
- Multi-tenancy 같은 특징으로 IaaS와 PaaS에 가장 잘 어울림
  - OpenStak의 60%에서 Storage IaaS로 사용

#### 장점

- 모든 Component들이 Scalable
- Single-point error에 강함
- Software-based, Opensource, adaptable
- Vendor Lock-in Free, Run on the Commodity Hardware
- self-manageable

막강한 Scalablity와 Flexibility로 Storage Silo에서 이전하는데 유리. Block, File, Object
Storage를 다양한 Back-end를 통해 제공 가능.

Object 기반으로 Storage가 구성이 되므로, 물리적인 Dependency에서 자유로울 수 있으며, 성능과 용량에 있어 선형 증가가
가능한 구성을 갖고 있음.

### The history and evolution of Ceph

History

- @Lawrence Livermore National Laboraroty (2003~2007)
  - 2003년: Sage Weil에 의해 개발, (40k lines)
  - 2006년: LGPL 라이선스로 공개됨.
- @DreamHost (2007~2011)
  - Staibility & Reliability 부분에서 개선.
- @InkTank (2012~2013)
  - 영역 확장을 위해 다양한 기능 및 툴 부분 개선
- @RedHat (2014~)

문어 관련 동물들의 이름으로 Code Name이 배정. Ceph는 UCSC의 마스코드인 Sammy와 연결되도록 이름이 선택

추가로 Ceph는 약자가 아니기 때문에 ~~CEPH~~로 표현하면 안됨

### Ceph release

Relase Rule은 리눅스 기반 다른 솔루션들과 유사한 방법으로 버전명을 배정함

- 기본적으로 Numeric Version으로 관리되며, 각 메이저 버전은 두족류 생물의 알파벳 첫글자 순으로 이름이 배정
- Luminous ~ Mimic
  - Luminous 이후 Ceph Community는 1년에 두번 Major 버전을 Tagging 함. (LTS, Stable 버전)
  - 최근 두버전의 LTS 버전에 대해서는 공식 지원을 하지만, Stable 버전은 최시 1개만 지원하게 됨
- Mimic ~ 현재
  - Major Release의 번호별 LTS와 Stable 구분이 사라짐
  - Release 9개월 이후 LTS 버전으로 변경됨

버전 Rule: `v{Major Release}.{Release Type}.{Point Release}`

- Major Number: 코드명의 알파벳 순서와 일치 *(Luminous: v12, Mimic: v13, Nautilus: v14)*
- Release Type: 아래 3가지만 존재
  - 0: Early Version, Pre-release상태의 Code
  - 1: Release Candidate, Cluster Test용이거나 사전 테스트용
  - 2: General-Available, Production-Ready

### New since the first edition

기능적인 차이점은 여기서 확인: <https://docs.ceph.com/docs/master/releases/#active-releases>

## Reference

- [Learning Ceph - Second Edition]

[Learning Ceph - Second Edition]: https://www.packtpub.com/virtualization-and-cloud/learning-ceph-second-edition
