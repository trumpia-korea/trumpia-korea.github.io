---
title: "Cent OS 7에 MongoDB 3 Version 설치 및 Replica Set 구성"
categories:
  - Management
tags:
  - Infra
  - NoSQL
  - MongoDB
  - ReplicaSet
author: joseph
---

***

> 이 문서에서는 CentOS 7에 MongoDB 3.0.15 version을 설치합니다.
>
> RPM Package를 사용한 설치를 권장합니다.

***

## MongoDB 3 version 설치

### Using Tar-ball

```bash
yum install libcurl openssl
wget https://fastdl.mongodb.org/linux/mongodb-linux-x86_64-rhel70-3.0.15.tgz
tar -zxvf mongodb-linux-*-3.0.15.tgz
cp mongodb-linux*/bin/* /usr/local/bin/
useradd mongod
mkdir -p /var/lib/mongo
mkdir -p /var/log/mongodb
chown mongod:mongod /var/lib/mongo
chown mongod:mongod /var/log/mongodb
```



### Using RPM Package

```bash
cd /etc/yum.repos.d/
wget https://repo.mongodb.org/yum/redhat/mongodb-org-3.0.repo
yum install -y mongodb-org
systemctl start mongod
systemctl enable mongod
```

만약 위 과정에서 `... /var/run/mongodb/mongodb.pid exist`와 같은 에러 발생 시,

```bash
rm /var/run/mongodb/mongodb.pid
chown mongod:mongod /tmp/mongodb-27017.sock
```

위 코드를 수행하고 다시 시도해본다.

해당 파일은 정상적으로 실행되는 시점에 생성되며, 종료 시에는 사라지는 파일이다.



***

## MongoDB Replica Set 구성

### Replica Set이란

하나의 MongoDB instance를 사용할 경우, 해당 host나 MongoDB instance가 down되는 상황과 같은 문제가 발생할 수 있다.

이를 대비하여 MongoDB는 다수의 복제 구성을 통한 DB HA(High Availability) 기능을 제공한다. 이러한 복제 구성의 그룹을 Replica Set이라 부른다.



### Replica Set(RS) 구성요소

Replica Set은 최소 3개의 member로 구성이 되며, 각 member는 아래 3가지 role 중 하나의 역할을 맡는다.

> **RS Role**
>
> * Primary
> * Secondary
> * Arbiter

**Primary**

> Master DB 역할을 수행한다.

**Secondary**

> Master DB의 replication. 필요에 따라 Primary가 될 수 있다.

**Arbiter**

> DB replication을 통한 storage 기능은 하지 않으며, Primary와 Secondary의 상태를 체크하여 Primary로 작동시킬 member를 선정하고 관리한다.



Replica Set의 구성은 다양하게 할 수 있으나, 주로 3개의 member에 대해 P-S-A(Primary - Secondary - Arbiter) 또는 P-S-S(Primary - Secondary - Secondary) 구성을 사용한다.

![MongoDB_ReplicaSet_type](/images/2019-11-25-Mongodb3/mongodb_rs_type.png "MongoDB Replica Set Types")



본 문서에서는 P-S-A 구성을 다룬다.



### MongoDB Replica Set 구성하기

#### 구성 절차

1. 3대의 CentOS 7 서버에서 위 [MongoDB 설치](#MongoDB 3 version 설치) 과정을 통해 MongoDB를 설치한다.



2. 각 서버의 daemon config 파일에서 해당하는 아래 부분을 설정한다.

   ```yaml
   systemLog:
     destination: file
     path: "/home/mongodb/db/log/mongod.log"
     logAppend: true
     logRotate: rename
   
   processManagement:
     fork: true
     pidFilePath: "/var/run/mongodb/mongod.pid"
   
   net:
     port: 27017
   #  bindIP 부분은 주석처리
   
   replication:
     oplogSizeMB: 10240
     replSetName: "rs01"
   setParameter:
     failIndexKeyTooLong: false
   ```

   * 기본 port인 27017을 유지해도 되고, 변경해도 된다.
   * "replication" 항목 아래 "replSetName"은 모두 동일해야 한다.



3. 모든 서버의 mongod daemon을 실행한다.

   ```bash
   systemctl restart mongod
   ```



4. 모든 서버에서 해당 port를 iptables에서 열어준다.

   ```
   vi /etc/sysconfig/iptables
   ...
   -A INPUT -p tcp -m state --state NEW -m tcp --dport 27017 -j ACCEPT
   ...
   
   :wq
   ```

   이후 iptables 서비스 재시작 및 확인

   ```bash
   systemctl reload iptables
   iptables -nL
   ```



5. 모든 서버의 /etc/hosts 파일에 각 서버의 IP와 hostname을 저장한다.

   ```
   192.168.3.27	mongodb-arbiter
   192.168.3.28	mongodb01
   192.168.3.29	mongodb02
   ```

   

6. Primary DB로 사용할 서버에서 mongo shell로 admin db에 접속한다.

   ```bash
   mongo localhost:27017/admin
   ```

   

7. 아래 명령 형식을 통해 Replica Set을 설정한다.

   ```
   > rs.initiate(
              {
         _id: "ReplicaSet명",
         version: 1,
         members: [
            { _id: 0, host : "<primaryIP>:<PORT>" },
            { _id: 1, host : "<SecondaryIP>:<PORT>" }
         ]
      }
   )
   ```

   이 문서의 환경에서는 다음과 같이 입력한다.

   ```
   > rs.initiate(
              {
         _id: "rs01",
         version: 1,
         members: [
            { _id: 0, host : "mongodb01:27017" },
            { _id: 1, host : "mongodb02:27017" }
         ]
      }
   )
   ```

   'OK' result 확인 후, prompt가 primary나 secondary로 변경되는지 확인한다.

   ```
   "rs01:PRIMARY>" or "rs02:SECONDARY>"
   ```

   

8. Primary 또는 Secondary 상태 확인 후, Arbiter를 추가한다.

   ```
   rs.addArb("mongodb-arbiter:27017")
   or
   rs.add( { host:"mongodb-arbiter:27017", arbiterOnly: true} )
   ```

   

9. 구성 상태를 확인한다.

   ```
   rs01:PRIMARY> rs.status()
   ```



#### Primary와 Secondary 확인

##### Primary 접속 후

```
use test
db.testcol.insert("seq" : 1)
```

##### Secondary 접속 후

```
use test
rs.slaveOk()
db.testcol.find({})
```



### Root 권한을 가진 관리 계정 생성

> admin DB에서 수행한다.

#### 사용자 생성

```
use admin
db.createUser(
   {
     user: "<계정명>",
     pwd: "<패스워드>",
     roles: ["root"]
   }
)
```



## MongoDB Replica Set Test

### Test Case & Result

![MongoDB_RS_Test](/images/2019-11-25-Mongodb3/mongodb_rs_test.png "MongoDB Replica Set Test")



## 다른 서버에서 Primary 접속하기

> 접속하는 환경에 mongo client가 설치되어 있어야 한다.

원격 MongoDB Replica Set의 Primary에 접속하는 명령은 다음과 같은 형식을 가진다.

```bash
mongo --host <RS_name>/<DB_Hostname>
```



RS_name은 "rs01", 각 역할을 하는 서버의 hostname은 아래 표와 같다고 가정한다.

| Role                     | HostName  |
| ------------------------ | --------- |
| **Primary** or Secondary | mongodb01 |
| Primary or **Secondary** | mongodb02 |
| **Arbiter** | mongodb-arbiter |

※ **참고**: **Primary** 및 **Secondary**는 상황에 따라 role이 서로 바뀔 수 있다.[^role 변경 예시]



위 상황에서 실제 원격 MongoDB Replica Set의 Primary에 접속하는 명령은 다음과 같다.

```bash
mongo --host rs01/mongodb01
mongo --host rs01/mongodb02
mongo --host rs01/mongodb-arbiter
```

위와 같이 RS의 member에 접속하면 해당 시점의 **Primary** 서버로 접속할 수 있다.



***

[^role 변경 예시]: Primary 접속이 끊기거나 종료된 경우 등에 **arbiter**의 판단에 따라 **Secondary**가 **Primary**로 변경된다.

