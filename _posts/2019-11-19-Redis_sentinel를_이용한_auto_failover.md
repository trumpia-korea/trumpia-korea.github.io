---
title: "Redis sentinel를 이용한 auto failover"
categories:
  - Management
tags:
  - Infra
  - DB
  - Redis 
  - Sentinel
author: sungsoo kim
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

## sentinel.conf 설정

redis-slave01 의 /etc/redis/sentinel.conf
```bash
bind 192.168.3.42  
port 26379  
sentinel monitor redis-cluster 192.168.3.41 6379 2  
sentinel down-after-milliseconds redis-cluster 5000  
sentinel parallel-syncs redis-cluster 1  
sentinel failover-timeout redis-cluster 10000
sentinel auth-pass redis-master  <master password>
daemonize yes  
pidfile "/var/run/redis/sentinel.pid"  
dir "/var/redis/"
```

redis-slave02 의 /etc/redis/sentinel.conf
```bash
bind 192.168.3.43  
port 26379  
sentinel monitor redis-cluster 192.168.3.41 6379 2  
sentinel down-after-milliseconds redis-cluster 5000  
sentinel parallel-syncs redis-cluster 1  
sentinel failover-timeout redis-cluster 10000
sentinel auth-pass redis-master  <master password>
daemonize yes  
pidfile "/var/run/redis/sentinel.pid"  
dir "/var/redis/"
```
 
redis-slave03 의 /etc/redis/sentinel.conf
```bash
bind 192.168.3.44  
port 26379  
sentinel monitor redis-cluster 192.168.3.41 6379 2  
sentinel down-after-milliseconds redis-cluster 5000  
sentinel parallel-syncs redis-cluster 1  
sentinel failover-timeout redis-cluster 10000
sentinel auth-pass redis-master <master password>
daemonize yes  
pidfile "/var/run/redis/sentinel.pid"  
dir "/var/redis/"
```

설정을 보면 bind ip만 다르고, 모두 동일한 옵션을 사용한다.
**bind** : Sentinel이 사용할 IP
```bash
bind 192.168.3.42
```
**port** : Sentinel이 실행 될 포트이며 기본은 26379이고 각각 Sentinel마다 다르게 설정을 해도 된다. 사용할 포트에 대한 방화벽 설정도 같이 이루어져야 한다.

```bash 
port 26379
```
  
**sentinel monitor  \<cluster name > \<redis master host> \<redis master port> \<quorum>** :

최초 모니터를 할 master redis 서버를 설정 부분이다. \<cluster name>은 이후 설정에서 사용할 내용이고 여기서는 redis-cluster로 지정하였다. \<redis master host>는 master redis의 현재 IP를 넣으면 되고, \<redis master port > master가 사용하는 포트를 기록하면 된다. \<quorum>은 말 그대로 정족수를 의미하며, 여기 2는 2개의 이상의 sentinel에서 확인하면 객관적으로 확인한다는 의미이다.   
```bash
sentinel monitor redis-cluster 192.168.3.41 6379 2
```

**sentinel  down-after-milliseconds \<cluster name > \<milliseconds>** :

다운으로 인식하는 최소 시간을 설정이며, sentinelCheckSubjectivelyDown에서 사용하는 마스터의 ‘down_after_period’ 이다. 5000이면 5초를 뜻한다.
```bash
sentinel down-after-milliseconds redis-cluster 5000
```
  
**sentinel parallel-syncs \<cluster name > <Parameter >** :

새로운 마스터 승격후에 몇개의 슬레이브가 싱크해야 하는지 설정한다. 값이 1이면 slave는 한대씩 Master와 동기화를 진행한다. 값이 클수록 Master에 부하가 가중이 된다.
 
```bash
sentinel parallel-syncs redis-cluster 1
```
 
**sentinel failover-timeout  \<cluster name >  \<milliseconds >** :

페일오버 작업 시간의 시간 설정  
  
```bash
sentinel failover-timeout redis-cluster 10000
```
  

**sentinel auth-pass  \<cluster name > \<master password>** :

Sentinel이 Master 접속하기 위한 패스워드 설정
  
```bash
sentinel auth-pass redis-master <master password>


# 4. Auto failover test

  

마스터에서 redis 서버를 내렸을 때 sentinal 로그에 다음과 같이 기록이 된다.

  
[그림]
  

로그에 보면 sdown, odown 이 나오는데 두가지 상태 모두 장애를 인식하는 것이다.

sdown은 Subjectively down으로 주관적인 판단으로 서버한대가 장애가 인식했다는 것이며, odown은 Objectively down으로 객관적인 판단으로 장애를 인식했다는 것인데 failover 정족수 설정에 의해서 판단을 하게 된다.
  

아래 로그기록은 master가 192.168.3.44(port 6379) 에서 192.168.3.42(port 6379)로 변경이 되었다는 것을 의미한다.
```bash
+switch-master redis-cluster 192.168.3.44 6379 192.168.3.42 6379
```
    

sentinel 한 서버에서 redis master 서버 한대가 주관적 판단으로 서버가 다운이 된것을 확인한것으로 나타낸다.
```bash
+sdown master redis-cluster 192.168.3.42 6379
```
  

객관적 판단을 하기 위해서 정족수 3명중 2명 이상이 판단을 하여 객관적으로 down이 된것을 나타낸다.
```bash
+odown master redis-cluster 192.168.3.42 6379 #quorum 3/2
```
  

아래 master가 192.168.3.42에서 192.168.3.44로 변경이 되었다고 알려준다.
```bash
+config-update-from sentinel ad9d421b5715f995579c952ce3368516c4c5a391 192.168.3.42 26379 @ redis-cluster 192.168.3.42 6379  
+switch-master redis-cluster 192.168.3.42 6379 192.168.3.44 6379
```
  
이상으로 Redis Sentinel를 설정하고 테스트를 해보았다.


