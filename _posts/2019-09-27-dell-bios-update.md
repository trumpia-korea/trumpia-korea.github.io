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
 - Dell R720xd

***

Dell Server를 사용하다보면 원격으로 바이오스를 업데이트를 해야 할 필요성이 있다. 바이오스 업데이트는 다양한 방법이 있지만 서버를 운영하다보면 console로 접근하는 방법을 선호하게 된다.

다음은 일반적으로 사용하는 dell bios 업데이트 방법입니다. 아

먼저 repository를 설정합니다. 

```bash
wget -q -O - http://linux.dell.com/repo/hardware/dsu/bootstrap.cgi | bash
```
