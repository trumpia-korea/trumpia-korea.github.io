
---
title: "CentOS 8 특징 및 설치하기 "
categories:
  - Manual
tags:
  - Infra
  - Linux
  - CentOS
author: sungsoo
---

CentOS 8 버전이 신규로 업데이트가 되었다. RHEL 8 버전(2019-05-07)에서 아래 패키지만 제거가 되었고, 동일한 패키지로 되었다고 합니다. 


## 제거 패키지 
 * insights-client
 * Red_Hat_Enterprise_Linux-Release_Notes-8-*
 * redhat-access-gui
 * redhat-bookmarks
 * redhat-indexhtml
 * redhat-logos
 * redhat-release-*
 * subscription-manager-migration
 * subscription-manager-migration-data


## 버전 8에 적용이 되는 특징 (RedHat 8 특징) 
  * 파일시스템 및 스토리지 관련해서 XFS에서 COW(copy-on-write) 데이터 범위 기능을 지원한다. COW는 두 개 이상의 파일에 대해 공통 데이터 블록 세트를  공유하는 기능이다. LUKS2 기능을 지원합니다. 
  * Cockpit 적용 : Cockpit에서  Web 베이스로 시스템을 관리 할 수 있도록 해주는 기능이며, 가상머신도 생성 관리를 할 수 있다. 
  * 데이터베이스 :  MySQL 8.0, MariaDB 10.3, PostgreSQL 9.6 and PostgreSQL 10, Redis 4
  * python 3.6 지원
  * Ruby 2.5
  * PHP 7.2
  * Perl 5.26
  * Apache HTTP Server 2.4.35
  * OpenSSL 1.1.1 & TLS 1.3
  * GNOME 3.28
  * iptables 대신에 nftables 적용
  *  qemu–kvm 2.12
  * 컨테이너 docker 대신 Podman,buildah 지원
  
  
 
## CentOS 8 설치
전반적으로 설치는 버전 7과 크게 다르지 않습니다. 

  
### 첫 화면에서 인스톨를 선택합니다. 

![centos8_1](/images/2019-10-04-CentOS-8-install/centos_1.png)


### 언어 설정

![centos8_2](/images/2019-10-04-CentOS-8-install/centos_2.png)

### 세부 설정 

![centos8_3](/images/2019-10-04-CentOS-8-install/centos_3.png)




