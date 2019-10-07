---
title: "SMPP Protocol의 이해 - 1"
categories:
  - Knowledge
tags:
  - SMPP
  - Protocol
  - SMS
  - Network
author: seolmin
---

우리나라에서는 SMPP(Short Message Peer-to-Peer) 프로토콜이 생소할지도 모르겠습니다. Trumpia에서는 미국 시장을 대상으로 자동화된 다중 채널 마켓팅 툴을 제공하고 있으며 채널 중 SMS 메시지를 수발신 하기 위한 규격으로 SMPP 프로토콜이 일반적으로 사용됩니다. 때문에 SMS 채널을 위한 응용 프로그램을 개발, 운영하는 개발자라면 필히 숙지해야 하는 프로토콜이 되었습니다.

SMPP 프로토콜은 개방형 산업 표준 프로토콜로 짧은 메시지를 빠르게 주고 받을 수 있는 통신 규격을 제공합니다. Peer-to-Peer 라는 이름과는 다르게 클라이언트-서버 모델을 사용하는데  PDU(Protocol Data Units)라는 작은 패킷을 통해 데이터를 주고 받으며 Bind 동작을 통해 클라이언트가 서버에 인증을 한 후 양방향으로 데이터를 주고 받을 수 있는 것이 특징입니다.

이 글에서는 SMPP 프로토콜의 기본 동작 방식에 대해 살펴 보겠습니다. 가장 많이 쓰이는 SMPP 3.4 version을 기준으로 작성하겠습니다.



## SMSC와 ESME

미국에는 수백가지의 이동 통신사가 존재하며 이 중 여러 이동 통신사와 관계를 구축하고 하나의 인터페이스로 이동 통신사와 사용자 중간에서 연결을 제공하는 업체를 Aggregator라 부릅니다. Trumpia에서는 여러 개의 Aggregator와 협업하여 SMS 마켓팅 서비스를 제공하고 있습니다. SMPP 프로토콜의 관점에서 Trumpia는 ESME, Aggregator는 SMSC로 볼 수 있습니다.

![SMSC_ESME](/images/2019-10-04-SMPP_Protocol_1/SMSC_ESME.png)

SMSC는 메시지를 저장, 변환, 전달하는 역할을 하는 서버이며 ESME는 SMSC에 연결되는 SMS 메시지를 송수신하기 위한 응용 프로그램을 말합니다. 이 때 SMSC와 ESME 사이에 데이터를 주고 받는 규격이 SMPP 프로토콜인 것입니다.



## PDU

PDU는 SMPP 프로토콜에서 주고받는 데이터 단위입니다. 요청과 응답의 쌍으로 이루어진 사용 목적에 따른 17가지 종류가 있습니다.  있으며 크게 Header, Body 그리고 특정 PDU는 TLV(Optional Parameter)를 포함할 수 있습니다. 기본적인 구성은 다음과 같습니다.

![PDU](/images/2019-10-04-SMPP_Protocol_1/PDU.png)

### Header

모든 PDU는 Header로 시작하고 그 뒤에 Body가 붙을 수 있습니다. 총 4가지 필드로 구성되며 PDU의 크기, 종류 등을 표현합니다.

* **command_length**: PDU의 전체 길이를 나타냅니다.
* **command_id**: PDU의 종류를 나타냅니다. 각 PDU별 다른 ID값을 가지며 최상위 비트가 0이면 요청PDU, 최상위 비트가 1이면 응답 PDU입니다.
* **command_status**: 특정 PDU의 경우 요청에 대한 상태를 나타내기 위해 응답 PDU에서 사용됩니다.
* **sequence_number**: 요청과 응답이 같은 값으로 설정되어 요청에 대한 응답을 Tracking하는데 사용됩니다. 주로 비동기 방식에서 유용하게 쓰입니다.

### Body

목적에 따라 PDU마다 서로 다른 필드들을 포함합니다. 만약 SMS 발송 요청을 위한 PDU라면 source_address, dest_address, short_message 등을 포함할 수 있습니다. 

### TLV

user_message_reference, language_indicator 등 선택적인 정보를 포함하기 위해 사용됩니다. SMSC는 자신의 SMSC에서만 사용되는 TLV를 자유롭게 정의하여 사용할 수 있습니다. 

![TLV](/images/2019-10-04-SMPP_Protocol_1/TLV.png)

* **TAG**: TLV의 ID로 어떤 종류의 Optional Parameter인지 판별하기 위해 사용됩니다.
* **LENGTH**: VALUE 값의 크기를 나타냅니다.
* **VALUE**: 실제 Optional Parameter의 값입니다. TLV에 따라 길이가 가변적입니다.



## Bind and Unbind

### Bind

Bind는 SMSC에 ESME의 인증정보를 전달하여 등록하고 메시지 수발신을 위해 세션을 요청하는 동작입니다. 이 또한 Bind PDU로 요청함으로써 이루어지며 세션 요청시마다 이루어집니다. 총 가지 Type의 세션을 요청할 수 있으며 SMSC에 따라 요청이 수락되거나 거절될 수 있습니다.

Bind 요청에는 다음과 같은 Body가 포함됩니다.

| FIELD             | SIZE         | DESCRIPTION                                                  |
| ----------------- | ------------ | ------------------------------------------------------------ |
| system_id         | 16 bytes max | SMSC에 인증하기 위한 ESME의 ID입니다.                        |
| password          | 9 bytes max  | SMSC에 인증하기 위한 ESME의 Password입니다.                  |
| system_type       | 13 bytes max | transmitter, receiver, transceiver 총 3가지의 요청을 나타낼 수 있습니다. |
| interface_version | 1 byte       | ESME에서 지원 가능한 SMPP 버전을 나타냅니다.                 |
| addr_ton          | 1 byte       | ESME에서 사용하는 주소 유형을 나타냅니다. Optional한 값입니다. 전화번호 표기 체계 관련한 내용으로 추후 알아보겠습니다. |
| addr_npi          | 1 byte       | ESME의 Numbering Plan Indicator를 나타냅니다. Optional한 값입니다. 전화번호 표기 체계 관련한 내용으로 추후 알아보겠습니다. |
| address_range     | 41 bytes max | ESME의 주소를 나타냅니다. Optional한 값입니다.               |



#### bind_transmitter

ESME가 발신 요청만 하고 수신은 하지 않는 세션을 요청합니다. 따라서 이 요청으로 연결된 세션은 일반적으로 ESME가 SMSC로부터 메시지를 수신하지 않고 발신만 하게 됩니다. system_type을 transmitter로 설정합니다.

![bind_transimitter](/images/2019-10-04-SMPP_Protocol_1/bind_transimitter.png)

#### bind_receiver

ESME가 수신만하고 발신 요청을 하지 않는 세션을 요청합니다. 따라서 이 요청으로 연결된 세션은 일반적으로 SMSC로부터 메시지를 수신 받기만 합니다. system_type을 receiver로 설정합니다.

![bind_receiver](/images/2019-10-04-SMPP_Protocol_1/bind_receiver.png)

#### bind_transceiver

ESME가 수발신 모두 요청하는 세션을 요청합니다. 따라서 이요청으로 연결된 세션은 메시지 발송 요청도 하고 메시지를 수신하기도 합니다. system_type을 transceiver로 설정합니다.

![bind_transeiver](/images/2019-10-04-SMPP_Protocol_1/bind_transeiver.png)



### Unbind

Unbind는 SMSC에서 ESME 등록을 취소하고 ESME가 더 이상의 메시지를 전달하거나 수신하지 않겠다는 것을 알리는 요청입니다. 즉, 세션을 닫기 위한 동작입니다.

요청과 응답 모두 Header만을 포함하며 요청이 완료된 이후 세션은 종료됩니다.



## Conclusion

SMPP 프로토콜의 기본 개념과 PDU, 세션 요청과 종류에 대해 알아보았습니다. 

SMS를 수발신 하기 위해 사용되는 만큼 우리나라에서는 일반적인 경우에는 접할 일이 없으나 만약 필요한 개발자 분이 계신다면 꼭 도움이 되었으면 좋겠습니다. 다음 글에서는 실제로 SMS를 수발신 하기 위한 submit_sm과 deliver_sm에 대해 알아보겠습니다. 

아자아자 화이팅!

