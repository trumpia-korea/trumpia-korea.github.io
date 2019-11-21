---
title: "Redis-Stats 을 이용한 redis 서버 모니터링"
categories:
  - Management
tags:
  - Infra
  - DB
  - Redis 
  - monitoring
author: sungsoo
---

***

> 해당 문서는 CentOS 7 Version을 기반으로 작성되었습니다.
>
> 모든 명령은 관리자(root) 권한으로 실행한다고 가정합니다.

***

# Redis-Stats 을 이용한 redis 서버 모니터링

redis-stat는 ruby로 만들어 졌기 때문에 ruby를 먼저 설치를 해야 합니다.


## redis-stat설치
 
```bash
yum -y install ruby-devel gcc gcc-c++  
gem install redis-stat
```
  

## redis-stat 사용

  
콘솔로 확인이 가능하며 명령어는 아래와 같다.
```
redis-stat [HOST[:PORT] ...] [INTERVAL [COUNT]]
```
옵션은 --help 명령어로 보면 간단하게 나온다.
```
-a, --auth=PASSWORD Password  
-v, --verbose Show more info  
--style=STYLE Output style: unicode|ascii  
--no-color Suppress ANSI color codes  
--csv[=CSV_FILE] Print or save the result in CSV  
--es=ELASTICSEARCH_URL Send results to ElasticSearch: [http://]HOST[:PORT][/INDEX]  
  
--server[=PORT] Launch redis-stat web server (default port: 63790)  
--daemon Daemonize redis-stat. Must be used with --server option.  
  
--version Show version  
--help Show this message
```
-a : redis 연결시 requirepass password가 필요할때 쓰는 옵션
-v : 세부적인 정보를 출력해 준다.

--csv : csv 파일로 저장한다.
--es : 엘라스틱서치로 보내는 옵션
--server : 콘솔이 아닌 웹으로 정보를 확인 port를 지정하지 않으면 기본 63790 포트를 사용한다.
--daemon : 백그라운드로 프로세스 실행


다음과 같이 실행을 하면..
```bash
redis-stat 192.168.3.41:6379 1 -a <password>
```
  

192.1683.41에 있는 포트 6379의 redis 서버를 연결하며 패스워드를 통해서 접근을 한다. 1초 마다 정보를 출력한다.

  
![Redis_state_l](/images/2019-11-21-Redis_Stats/Inkedredis_stat1.jpg)

  

### 1초 간격으로 10회 출력
```bash
redis-stat 192.168.3.41:6379 1 10 -a \<password>
 ```

![Redis_state_2](/images/2019-11-21-Redis_Stats/Inkedredis_stat2.jpg)

  
  

### --verbose 옵션으로 추가 정보 보여주기

redis-stat 192.168.3.41:6379 1 10 -a \<password> --verbose

![Redis_state_3](/images/2019-11-21-Redis_Stats/Inkedredis_stat3.jpg)

### 여러 대의 서버 동시에 보여주기

  
```bash
redis-stat 192.168.3.45:6380 192.168.3.46:6381 192.168.3.47:6382 1 10
```
  
![Redis_state_4](/images/2019-11-21-Redis_Stats/redis_stat4.png)  

### csv파일 형태로 출력하기
```bash
redis-stat 192.168.3.45:6380 1 10 --csv=/tmp/redis-stat-log.csv
```
로그기록을 따로 하지 않기 때문에 csv로 같이 기록하는 것을 권장한다.

```bash
[root@redis_master01 tmp]# cat redis-stat-log.csv  
time,us,sy,cl,bcl,mem,rss,keys,cmd/s,exp/s,evt/s,hit%/s,hit/s,mis/s,aofcs  
1574319765.1957128,,,36,0,3062992.0,8990720.0,8,,,,,,,106068.0  
1574319766.1960983,0,0,36,0,2989280.0,8990720.0,8,4.998072778123265,0.0,0.0,100.0,1.999229111249306,0.0,106068.0  
1574319767.1961405,0,0,36,0,3026136.0,8990720.0,8,4.9997888339186,0.0,0.0,100.0,1.9999155335674397,0.0,106068.0  
1574319768.196125,0,1,36,0,2989280.0,8990720.0,8,10.000156972463998,0.0,0.0,100.0,7.000109880724798,0.0,106068.0  
1574319769.196192,1,0,36,0,2989280.0,8990720.0,8,5.999597564994133,0.0,0.0,100.0,2.9997987824970664,0.0,106068.0  
1574319770.1961706,0,0,36,0,3026136.0,8990720.0,8,4.000085513828115,0.0,0.0,100.0,1.0000213784570287,0.0,106068.0  
1574319771.1962042,0,1,36,0,3062992.0,8990720.0,8,4.999831995645282,0.0,0.0,,0.0,0.0,106068.0  
1574319772.196141,0,1,36,0,3026136.0,8990720.0,8,7.000441370827989,0.0,0.0,100.0,1.000063052975427,0.0,106068.0  
1574319773.196138,0,1,36,0,3026136.0,8990720.0,8,9.00002835908936,0.0,0.0,100.0,6.000018906059573,0.0,106068.0  
1574319774.1960073,0,0,36,0,2989280.0,8990720.0,8,3.0003928504370863,0.0,0.0,,0.0,0.0,106068.0  
``` 

  
  
  

### Web 페이지로 상태를 보여주기

  
```bash
redis-stat 192.168.3.45:6380 192.168.3.46:6381 192.168.3.47:6382 5 --daemon --server=7000
```
  

프로세스 상태 확인
```bash
28192 ? Sl 0:00 redis-stat-daemon
```
  
  

7000포트에 대한 방화벽 설정을 끝나고 웹 페이지로 연결

![Redis_state_5](/images/2019-11-21-Redis_Stats/redis_stat5.png)


각 서버 별로 확인도 가능

![Redis_state_6](/images/2019-11-21-Redis_Stats/redis_stat6.png)


## 프로그램 종료
```bash
killall -9 redis-stat-daemon

