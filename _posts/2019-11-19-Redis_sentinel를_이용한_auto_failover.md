---
title: "Redis sentinel를 이용한 auto failover"
categories:
  - Management
tags:
  - Infra
  - DB
  - Redis 
  - Sentinel
author: sskim
---

***

> 해당 문서는 CentOS 7 Version을 기반으로 작성되었습니다.
>
> 모든 명령은 관리자(root) 권한으로 실행한다고 가정합니다.

***

# Redis sentinel를 이용한 auto failover
환경 : CentOS 7

호스트 이름 및 IP

  
|hostname|ip  |
|---|---|
| redis_master01 | 192.168.3.41 |
|redis_slave01|192.168.3.42|
|redis_slave02|192.168.3.43|
|redis_slave03|192.168.3.44|


## 구성 다이어그램


[그림]

**현재 2019.11 시점에서 redis는 5 버전까지 나와 있지만 소스코드가 아닌 yum 설치를 할 계획이라서 yum rpm 버전은 3.2.12-2 으로 설치를 할 계획이다.**


## 1. Redis Master 설정
redis 설치는 CentOS Yum 패키지로 설치를 진행한다. 

```bash
yum install redis
```
설치를 하면 아래와 같이 나온다.
```bash
#yum install redis
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
epel/x86_64/metalink                                                                                                           | 9.1 kB  00:00:00
 * base: mirror.kakao.com
 * epel: ftp.jaist.ac.jp
 * extras: mirror.kakao.com
 * updates: ftp.jaist.ac.jp
base                                                                                                                           | 3.6 kB  00:00:00
epel                                                                                                                           | 5.3 kB  00:00:00
extras                                                                                                                         | 2.9 kB  00:00:00
updates                                                                                                                        | 2.9 kB  00:00:00
(1/3): epel/x86_64/updateinfo                                                                                                  | 1.0 MB  00:00:00
(2/3): updates/7/x86_64/primary_db                                                                                             | 4.2 MB  00:00:01
(3/3): epel/x86_64/primary_db                                                                                                  | 6.9 MB  00:00:01
Resolving Dependencies
--> Running transaction check
---> Package redis.x86_64 0:3.2.12-2.el7 will be installed
--> Processing Dependency: libjemalloc.so.1()(64bit) for package: redis-3.2.12-2.el7.x86_64
--> Running transaction check
---> Package jemalloc.x86_64 0:3.6.0-1.el7 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

======================================================================================================================================================
 Package                             Arch                              Version                                  Repository                       Size
======================================================================================================================================================
Installing:
 redis                               x86_64                            3.2.12-2.el7                             epel                            544 k
Installing for dependencies:
 jemalloc                            x86_64                            3.6.0-1.el7                              epel                            105 k

Transaction Summary
======================================================================================================================================================
Install  1 Package (+1 Dependent package)

Total download size: 648 k
Installed size: 1.7 M
Is this ok [y/d/N]: y
Downloading packages:
(1/2): jemalloc-3.6.0-1.el7.x86_64.rpm                                                                                         | 105 kB  00:00:00
(2/2): redis-3.2.12-2.el7.x86_64.rpm                                                                                           | 544 kB  00:00:00
------------------------------------------------------------------------------------------------------------------------------------------------------
Total                                                                                                                 651 kB/s | 648 kB  00:00:00
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  Installing : jemalloc-3.6.0-1.el7.x86_64                                                                                                        1/2
  Installing : redis-3.2.12-2.el7.x86_64                                                                                                          2/2
  Verifying  : redis-3.2.12-2.el7.x86_64                                                                                                          1/2
  Verifying  : jemalloc-3.6.0-1.el7.x86_64                                                                                                        2/2

Installed:
  redis.x86_64 0:3.2.12-2.el7

Dependency Installed:
  jemalloc.x86_64 0:3.6.0-1.el7

Complete!
[root@redis_master01 ~]#
```

## 설치후 설정 redis.conf 파일 조정

```bash
bind 0.0.0.0
패스워드 설정
requirepass   #password  ← 패스워드 설정 slave에서도 사용함.
```

아래와 같이 설정 된 것을 0.0.0.0으로 변경을 한다. 
bind 127.0.0.1   →  bind 0.0.0.0

패스워드 인증을 위한 설정, 주석 제거하고 사용할 패스워드를 설정한다. 
requirepass foobared   →  requirepass  <password>


##설정 저장 후 redis 서버 재시작 및 자동 재시작 설정
```bash
systemctl  start redis
systemctl  enable redis
```

## 방화벽 설정

iptable 설정 방법
```bash
 # vi /etc/sysconfig/iptables
-A INPUT -p tcp -m state --state NEW -m tcp --dport 6379 -j ACCEPT
 # service iptables reload
 ```

firewalld(netfilter) 설정 방법
```bash
firewall-cmd --add-port=6379/tcp --permanen
firewall-cmd --reload
```

redis 프로세스 확인
```bash
 /usr/bin/redis-server 0.0.0.0:6379
```

잘 동작하는지 직접 키를 입력해서 확인해 본다.

```bash
[root@redis_master01 ~]# redis-cli -h 192.168.3.41 -p 6379
192.168.3.41:6379> auth dusruffpeltm
OK
192.168.3.41:6379> set mykey test1
OK
192.168.3.41:6379> get mykey
"test1"
192.168.3.41:6379> set mykey test2
OK
192.168.3.41:6379> get mykey
"test2"
192.168.3.41:6379> exit
[root@redis_master01 ~]#
```

정상적으로 동작을 하면 redis master 설정은 끝이 났다. 

# Redis slave 설정
redis를 설치는 master와 동일하고 설정 파일에 몇가지만 추가하면 된다. 
redis.conf를 다음과 같이 설정한다

```bash
#bind 127.0.0.1 ---> bind 0.0.0.0
bind 0.0.0.0

#master server ip 
slaveof  192.168.3.41 6379

#마스터 server에서 설정한 패스워드
masterauth <master password>  

#패스워드 설정
requirepass <master passsword>

#아래 내용은 주석만 제거 
repl-ping-slave-period 10
repl-timeout 60
```
방화벽과 재시작 설정을 master 서버와 동일하게 진행한다. 

## 테스트

master에서 정보를 잘 받아 오는지 확인 한다. 

```bash
[root@redis_slave03 ~]# redis-cli -h 192.168.3.44  
192.168.3.44:6379> get mykey  
"test2"
```
```bash
[root@redis_slave02 ~]# redis-cli -h 192.168.3.43  
192.168.3.43:6379> get mykey  
"test2"
```
master에서 기존의 key를 변경 후 slave에 잘 반영이 되는지 확인한다. 
master에서 mykey값을 “test55” 로 변경한다. 

```bash
192.168.3.41:6379> set mykey test55  
OK
```
slave에서 값이 변경이 되었는지 확인을 한다. “test55”값을 가져오는 것을 알 수 있다. 

```bash
192.168.3.43:6379> get mykey
"test55"
192.168.3.43:6379>
```
