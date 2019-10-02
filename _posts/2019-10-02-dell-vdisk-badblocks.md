---
title: "Dell Vdisk Virtual Disk Bad Blocks 클리어"
categories:
  - Fault
tags:
  - Infra
  - IDRAC
  - Dell
  - BIOS
author: sungsoo
---

***

###### Environments
 - CentOS 7.6
 - Dell R810

***

Dell 장비를 운영하다가 아래와 같이 omreport에서 장애 에러가 나오게 되어 이것을 처리 하기 위한 방법을 기록하였습니다.
특정 디스크가 에러가 아니라서 원인츨 찾기 힘들었다. 

```bash
[dell]Storage;CRITICAL;SOFT;3;Logical Drive '/dev/sda' [RAID-10, 1,862.00 GB] is Ready
```

처음에는 서버 리부팅을 통해서 레이드콘트롤러로 확인을 했지만 특별히 이상이 보이지 않았다. 
그래서 여러 사이트를 찾고, omreport로 분석을 하다가 Virtual Disk Bad Blocks에 Yes가 표시가 된것을 발견하게 되었다. 

```bash
[root@virtsrv700 ~]# omreport storage vdisk
List of Virtual Disks in the System

Controller PERC H710P Mini (Embedded)
ID                                : 0
Status                            : Critical
Name                              : Virtual Disk0
State                             : Ready
Hot Spare Policy violated         : Not Assigned
Virtual Disk Bad Blocks           : Yes
Encrypted                         : No
Layout                            : RAID-10
Size                              : 1,862.00 GB (1999307276288 bytes)
T10 Protection Information Status : No
Associated Fluid Cache State      : Not Applicable
Device Name                       : /dev/sda
Bus Protocol                      : SATA
Media                             : SSD
Read Policy                       : Adaptive Read Ahead
Write Policy                      : Write Through
Cache Policy                      : Not Applicable
Stripe Element Size               : 64 KB
Disk Cache Policy                 : Enabled

[root@virtsrv700 ~]#
```

블량 블록을 지우기 위해서 Clear Virtual Disk Bad Blocks 작업을 싱행한다. 
```bash
omconfig storage vdisk action=clearvdbadblocks controller=id vdisk=id
```

cli 명령어를 실행 후 다시 한번 omreport로 확인하였습니다.
```bash
[root@virtsrv700 ~]# omconfig storage vdisk action=clearvdbadblocks controller=0 vdisk=0
Command successful!
```

다음과 같이 확인을 하면 Virtual Disk Bad Blocks이 없어 졌다. 

```bash
[root@virtsrv700 ~]# omreport storage vdisk
List of Virtual Disks in the System

Controller PERC H710P Mini (Embedded)
ID                                : 0
Status                            : Ok
Name                              : Virtual Disk0
State                             : Ready
Hot Spare Policy violated         : Not Assigned
Encrypted                         : No
Layout                            : RAID-10
Size                              : 1,862.00 GB (1999307276288 bytes)
T10 Protection Information Status : No
Associated Fluid Cache State      : Not Applicable
Device Name                       : /dev/sda
Bus Protocol                      : SATA
Media                             : SSD
Read Policy                       : Adaptive Read Ahead
Write Policy                      : Write Through
Cache Policy                      : Not Applicable
Stripe Element Size               : 64 KB
Disk Cache Policy                 : Enabled

```
이렇게 해서 문제를 해결 하였다. 

참고 사이트 : https://www.dell.com/support/article/us/en/19/sln111146/how-to-handle-puncturing-bad-blocks-on-virtual-disks-for-poweredge-servers?lang=en





