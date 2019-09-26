---
title: "Post: MegaRAID Disk Fault"
categories:
  - Fault
tags:
  - Infra
  - Raid Controller
  - Megaraid
  - Megacli
  - Baremetal
author: kyedon
---

***

###### Environments
 - CentOS 7.6
 - MegaRAID

***

먼저 pciutils 를 설치하여 어떤 레이드 컨트롤러를 사용하는지 확인한다.

```shell
yum install lshw pciutils
lspci | grep -i raid
```

```console
[root@image-storage ~]# yum install lshw pciutils
[root@image-storage ~]# lspci | grep -i raid
01:00.0 RAID bus controller: LSI Logic / Symbios Logic MegaRAID SAS-3 3108 [Invader] (rev 02)
```

LSI MegaRAID는 MegaCli라는 커맨드라인 툴이 있다.
[broadcom 사이트에서 다운로드](https://www.broadcom.com/docs/12351587?_ga=2.81670295.1654370925.1557214314-814067834.1557214314)
받아 RPM을 통해 설치한다.

```shell
wget https://docs.broadcom.com/docs-and-downloads/raid-controllers/raid-controllers-common-files/8-07-14_MegaCLI.zip
unzip 8-07-14_MegaCLI.zip
cd Linux
yum install MegaCli-8.07.14-1.noarch.rpm
```

설치 후에는 아래 명령어로 간단하게 알람을 비활성화할 수 있다.

```shell
/opt/MegaRAID/MegaCli/MegaCli64 -AdpSetProp AlarmSilence -a0
```

```console
[root@image-storage ~]# /opt/MegaRAID/MegaCli/MegaCli64 -AdpSetProp AlarmSilence -a0

Adapter 0: Set alarm to Silenced success.

Exit Code: 0x00
```

이제 디스크를 리빌드해야한다.
디스크를 PC에 연결하여 모든 파티션을 제거한 뒤 다시 서버에 부착한다.

자동으로 리빌드가 된다면 좋았겠지만 리빌드가 되지 않았다.

아래 명령어로 논리드라이브의 상태를 볼 수 있다.

```shell
/opt/MegaRAID/MegaCli/MegaCli64 -LDInfo -Lall -aAll
```

```console
[root@image-storage ~]# /opt/MegaRAID/MegaCli/MegaCli64 -LDInfo -Lall -aAll
Adapter 0 -- Virtual Drive Information:
Virtual Drive: 0 (Target Id: 0)
Name                :
RAID Level          : Primary-1, Secondary-0, RAID Level Qualifier-0
Size                : 930.390 GB
Sector Size         : 512
Is VD emulated      : No
Mirror Data         : 930.390 GB
State               : Degraded
Strip Size          : 256 KB
Number Of Drives    : 2
Span Depth          : 1
Default Cache Policy: WriteBack, ReadAhead, Direct, No Write Cache if Bad BBU
Current Cache Policy: WriteThrough, ReadAhead, Direct, No Write Cache if Bad BBU
Default Access Policy: Read/Write
Current Access Policy: Read/Write
Disk Cache Policy   : Disk's Default
Encryption Type     : None
PI type: No PI

Is VD Cached: No
```
**State: Degraded** 상태로 되어있다.

아래 명령어로 모든 물리드라이브의 정보를 확인한다.

```shell
/opt/MegaRAID/MegaCli/MegaCli64 -PDList -aAll
```

```console
[root@image-storage ~]# /opt/MegaRAID/MegaCli/MegaCli64 -PDList -aAll

Enclosure Device ID: 252
Slot Number: 1
Enclosure position: N/A
Device Id: 9
WWN: ***
Sequence Number: 11
Media Error Count: 0
Other Error Count: 0
Predictive Failure Count: 0
Last Predictive Failure Event Seq Number: 0
PD Type: SATA

Raw Size: 931.512 GB [0x74706db0 Sectors]
Non Coerced Size: 931.012 GB [0x74606db0 Sectors]
Coerced Size: 930.390 GB [0x744c8000 Sectors]
Sector Size:  512
Logical Sector Size:  512
Physical Sector Size:  512
Firmware state: Unconfigured(bad)
Device Firmware Level: ***
Shield Counter: 0
Successful diagnostics completion on :  N/A
SAS Address(0): ***
Connected Port Number: 2(path0)
Inquiry Data: ATA     ***
FDE Capable: Not Capable
FDE Enable: Disable
Secured: Unsecured
Locked: Unlocked
Needs EKM Attention: No
Foreign State: None
Device Speed: 6.0Gb/s
Link Speed: 6.0Gb/s
Media Type: Solid State Device
Drive:  Not Certified
Drive Temperature :35C (95.00 F)
PI Eligibility:  No
Drive is formatted for PI information:  No
PI: No PI
Drive's NCQ setting : Enabled
Port-0 :
Port status: Active
Port's Linkspeed: 6.0Gb/s
Drive has flagged a S.M.A.R.T alert : No
```

**Firmware state: Unconfigured(bad)** 상태이다.

good 상태로 만들어준다.

```shell
/opt/MegaRAID/MegaCli/MegaCli64 -pdmakegood -physdrv [252:1] -a0
```

```console
[root@image-storage ~]# /opt/MegaRAID/MegaCli/MegaCli64 -pdmakegood -physdrv [252:1] -a0
Adapter: 0: EnclId-252 SlotId-1 state changed to Unconfigured-Good.

Exit Code: 0x00
```

아래 명령어로 리빌드 시도했으나 실패했다.

```shell
/opt/MegaRAID/MegaCli/MegaCli64  -pdhsp -set -physdrv [252:1] -a0
```

물리드라이브의 상태를 다시 확인해보니 **Foreign State: Foreign**으로 되어있었다.
아래 명령어로 foreign clear 시킨다.

```shell
/opt/MegaRAID/MegaCli/MegaCli64 -cfgforeign -clear -a0
```

```console
[root@image-storage ~]# /opt/MegaRAID/MegaCli/MegaCli64 -cfgforeign -clear -a0

Foreign configuration 0 is cleared on controller 0.

Exit Code: 0x00

foreign clear
```

다시 시도하자 정상적으로 핫스페어로 등록되었다.

```console
[root@image-storage ~]# /opt/MegaRAID/MegaCli/MegaCli64 -pdhsp -set -physdrv [252:1] -a0

Adapter: 0: Set Physical Drive at EnclId-252 SlotId-1 as Hot Spare Success.

Exit Code: 0x00
```

아래 명령어로 현재 리빌드 상태를 확인할 수 있다.

```shell
/opt/MegaRAID/MegaCli/MegaCli64 -PDRbld -ShowProg -PhysDrv[252:1] -a0
```

```console
[root@image-storage ~]# /opt/MegaRAID/MegaCli/MegaCli64 -PDRbld -ShowProg -PhysDrv[252:1] -a0

Rebuild Progress on Device at Enclosure 252, Slot 1 Completed 0% in 0 Minutes.

Exit Code: 0x00
```

이후 LD info 에서 드라이브 숫자와 상태를 확인하면 된다.

리빌드 이후에도 알람이 한 번 더 오기 때문에 한번 더 알람을 꺼줘야한다.

## 참고
[스마일서브 공식 블로그 - RAID에 대한 기본적 정의부터 구성까지(LSI MegaRAID SAS 9261-8i)](https://idchowto.com/?p=35534)  
[스마일서브 공식 블로그 - MegaCli 를 통한 Raid Rebuilding](https://idchowto.com/?p=40327)

