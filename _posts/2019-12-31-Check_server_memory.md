---
title: "Check server memory"
categories:
  - knowledge
tags:
  - Server
  - CentOS
author: sungsoo
---


Server에서 지원되는 최대 Memory 용량을 알아보기.
현재 메모리량은 Free로 같은 명령어로 알수 있지만, 얼마나 Server가 메모리를 확장 할 수 있는지 확인이 불가능하다.
메모리가 부족할때 서버가 얼마나 확장이 가능한지 알아보는 방법에 대해서 알아보자.
dmidecode를 이용해서 알아 볼 수 있다. dmidecode은 하드웨어 정보를 확인할 수 있는데, 이것을 이용해서 메모리 용량을 알아 볼 수 있다. 

```bash
#dmidecode -t 16
```

아래와 같이 결과가 나온다. 

```bash
# dmidecode 2.12
SMBIOS 2.7 present.

Handle 0x1000, DMI type 16, 23 bytes
Physical Memory Array
        Location: System Board Or Motherboard
        Use: System Memory
        Error Correction Type: Multi-bit ECC
        Maximum Capacity: 1536 GB
        Error Information Handle: Not Provided
        Number Of Devices: 24
```

결과를 보면 Maximum Capacity 항목에서 192GB 용량이 나오는데, 이 서버의 최대 메모리 용량이 1536GB라는 것을 알 수 있다. 
그리고, 또 하나 봐야 할 부분은 서버보드에서 제공하는 메모리 슬롯 개수이다.  "Number Of Devices"를 확인하면 24개 메모리 슬롯을 가지고 있는 서버라는 것을 알 수 있다.

현재 어떻게 슬롯을 쓰고 있는지 확인해야 하는데 -t 17로 보면 좀 더 자세한 것을 볼 수 있다.

```bash
#dmidecode -t 16
```

결과는 아래와 같다. 

```bash
# dmidecode 2.12
SMBIOS 2.7 present.

Handle 0x1100, DMI type 17, 34 bytes
Memory Device
        Array Handle: 0x1000
        Error Information Handle: Not Provided
        Total Width: 72 bits
        Data Width: 64 bits
        Size: 16384 MB
        Form Factor: DIMM
        Set: 1
        Locator: DIMM_A1
        Bank Locator: Not Specified
        Type: DDR3
        Type Detail: Synchronous Registered (Buffered)
        Speed: 1333 MHz
        Manufacturer: 019804B30000
        Serial Number: AF29B613
        Asset Tag: 04132521
        Part Number: 9965516-024.A00LF
        Rank: 2
        Configured Clock Speed: 1333 MHz
```

메모리 슬롯 만큼 나오게 된다. 위 정보를 보면 메모리에 대한 자세한 정보를 알 수 있따. 
"Locator: DIMM_A1 " : A1 슬롯에 있는 메모리 정보
"Type: DDR3 " : DDR3 타입의 메모리
"Part Number: 9965516-024.A00LF  : 메모리 회사에 대한 내용을 알 수 있다. 이 번호를 이용해서 검색을 하면 메모리에 대한 정확안 내용을 알 수 있다. 

9965516-024.A00LF 정보를 가지고 찾아보면 Kingston 제품인것을 알 수 있다. 여기서 봐야 할 것은  PC3-10600 ECC 부분이다. 메모리 추가 할 때 최대한 같은 제품을 쓰면 좋지만
없을 때는 같은 spec에 메모리를 쓰면 된다. 그때 확인해야 할 정보를 알 수 있다. 

```
9965516-024.A00LF - Kingston 16GB DDR3-1333MHz PC3-10600 ECC Registered CL9 240-Pin DIMM Dual Rank x4 Memory Module (Intel Certified)
```

dmidecode 은 하드웨어 정보를 알아 볼 때 유용한 명령어이다. 









