---
title: "Dell Bios update 방법"
categories:
  - bios
tags:
  - Infra
  - idrac
  - dell

author: sungsoo
---

***

###### Environments
 - CentOS 7.6
 - Dell R720xd

***

Dell Server를 사용하다보면 원격으로 바이오스를 업데이트를 해야 할 필요성이 있다. 바이오스 업데이트는 다양한 방법이 있지만 서버를 운영하다보면 console로 접근하는 방법을 선호하게 된다.

다음은 일반적으로 사용하는 dell bios 업데이트 방법입니다. 아

먼저 repository를 설정합니다. 

```bash
wget -q -O - http://linux.dell.com/repo/hardware/dsu/bootstrap.cgi | bash
```
공개 키를 설치하기 전에는 동의가 필요합니다. 



저장소를 사용하여 dsu를 설치합니다. 

```bash
yum -y install dell-system-update
```



설치에 문제가 없다면 바이오스 업데이트를 진행합니다. 

```bash
dsu --inventory 
```

아니면 약자를 써서 이용합니다. 

```bash
dsu -i
```

다음과 같이 버전에 대한 정보가 나옵니다. 


```bash
[root@localhost ~]# dsu --inventory
DELL EMC System Update 1.7.0
Copyright (C) 2014 DELL EMC Proprietary.
Verifying catalog installation ...
Installing catalog from repository ...
Fetching dsucatalog ...
Reading the catalog ...
Verifying inventory collector installation ...
Getting System Inventory ...
warning: Inventory collector returned with partial failure.

1. Application, OpenManage Server Administrator  ( Version : 9.3.0 )

2. BIOS, BIOS  ( Version : 2.7.0 )

3. Application, Lifecycle Controller  ( Version : 2.63.60.62 )

4. Application, Dell 64 Bit uEFI Diagnostics, version 4247, 4247A1, 4247.2  ( Version : 4247A1 )

5. Firmware, Integrated Dell Remote Access Controller  ( Version : 2.63.60.62 )

6. Application, OS COLLECTOR 4.0 12-13G, A00  ( Version : 4.0 )

7. Firmware, System CPLD  ( Version : 1.0.3 )

8. Firmware, Power Supply  ( Version : 69.45.99 )

9. Firmware, Firmware for  - Disk 0 of PERC H710P Mini Controller 0    ( Version : 036C )

10. Firmware, PERC H710P Mini Controller 0 Firmware  ( Version : 21.3.5-0002 )

11. Firmware, 12G SEP Firmware   ( Version : 1.00 )

12. Firmware, NetXtreme BCM5720 2-port Gigabit Ethernet PCIe (em3)  ( Version : 21.40.9 )

13. Firmware, NetXtreme BCM5720 2-port Gigabit Ethernet PCIe (em1)  ( Version : 21.40.9 )

14. Firmware, NetXtreme BCM5720 2-port Gigabit Ethernet PCIe (em4)  ( Version : 21.40.9 )

15. Firmware, NetXtreme BCM5720 2-port Gigabit Ethernet PCIe (em2)  ( Version : 21.40.9 )

16. Firmware, Intel(R) Ethernet Server Adapter X520-2  ( Version : 16.5.20 )

17. Firmware, Intel(R) Ethernet Server Adapter X520-2  ( Version : 16.5.20 )

18. Application, Inventory Collector  ( Version : 19.04.00 )


Exiting DSU!

```



서버에 장착된 장비에 따라서 정보가 달리 나옵니다.  

바이오스 업데이트를 진행합니다.

```bash
dsu
```

아래와 같이 현재 버전과 업데이트 버전에 대한 정보가 나옵니다. 

```bash
[root@localhost ~]# dsu
DELL EMC System Update 1.7.0
Copyright (C) 2014 DELL EMC Proprietary.
Verifying catalog installation ...
Installing catalog from repository ...
Fetching dsucatalog ...
Reading the catalog ...
Verifying inventory collector installation ...
Getting System Inventory ...
warning: Inventory collector returned with partial failure.
Determining Applicable Updates ...

|--------DELL EMC System Update-----------|
[ ] represents 'not selected'
[*] represents 'selected'
[-] represents 'Component already at repository version (can be selected only if -e option is used)'
Choose:  q - Quit without update, c to Commit, <number> - To Select/Deselect, a - Select All, n - Select None
[-]1 12G SEP Firmware
Current Version : 1.00 Same as : 1.00, Criticality : Recommended, Type : Firmware

[-]2 BIOS
Current Version : 2.7.0 Same as : 2.7.0, Criticality : Urgent, Type : BIOS

[-]3 Integrated Dell Remote Access Controller
Current Version : 2.63.60.62 Same as : 2.63.60.62, Criticality : Urgent, Type : Firmware

[-]4 NetXtreme BCM5720 2-port Gigabit Ethernet PCIe (em3)
Current Version : 21.40.9 Same as : 21.40.9, Criticality : Optional, Type : Firmware

[-]5 NetXtreme BCM5720 2-port Gigabit Ethernet PCIe (em1)
Current Version : 21.40.9 Same as : 21.40.9, Criticality : Optional, Type : Firmware

[-]6 NetXtreme BCM5720 2-port Gigabit Ethernet PCIe (em4)
Current Version : 21.40.9 Same as : 21.40.9, Criticality : Optional, Type : Firmware

[-]7 NetXtreme BCM5720 2-port Gigabit Ethernet PCIe (em2)
Current Version : 21.40.9 Same as : 21.40.9, Criticality : Optional, Type : Firmware

[-]8 PERC H710P Mini Controller 0 Firmware
Current Version : 21.3.5-0002 Same as : 21.3.5-0002, Criticality : Recommended, Type : Firmware

[-]9 Dell 64 Bit uEFI Diagnostics, version 4247, 4247A1, 4247.2
Current Version : 4247A1 Same as : 4247A1, Criticality : Recommended, Type : Application

[-]10 OS COLLECTOR 4.0 12-13G, A00
Current Version : 4.0 Same as : 4.0, Criticality : Recommended, Type : Application

Enter your choice :

```

버전이 동일하면 [-] 로 표시가 되고, 업데이트 대상은 [ ]로 표시가 됩니다. 업데이트 대상을 번호로 선택을 하거나 모두 적용을 하고 싶으면 a를 선택합니다. 선택되면 [*]가 됩니다. 

선택후에 적용하려면 c 를 입력하면 업데이트가 진행이 됩니다. 



만약 아래와 같이 에러가 나오면 key를 변경해 줘야 합니다.

```bash
[root@localhost ~]# dsu --inventory
DELL EMC System Update 1.7.0
Copyright (C) 2014 DELL EMC Proprietary.
Do you want to import public key(s) on the system (Y/N)? : y
Unable to read public file /usr/libexec/dell_dup/0x756ba70b1019ced6.asc
Exiting DSU!
[root@localhost ~]#
```

key를 받아서 적용을 합니다. 

```bash
curl -s https://linux.dell.com/repo/hardware/dsu/copygpgkeys.sh | bash
```



아래는 실행 결과 입니다. 

```bash
[root@localhost ~]# curl -s https://linux.dell.com/repo/hardware/dsu/copygpgkeys.sh | bash
0x756ba70b1019ced6.asc: Downloading GPG key.
                      : Copying GPG key to /usr/libexec/dell_dup/.
0x1285491434D8786F.asc: Downloading GPG key.
                      : Copying GPG key to /usr/libexec/dell_dup/.
0xca77951d23b66a9d.asc: Downloading GPG key.
                      : Copying GPG key to /usr/libexec/dell_dup/.
Done!
```

바이오스 업데이트는 최상 버전으로 관리하는 것을 권장합니다. 
