---
title: "SMPP Protocol의 이해 - 2"
categories:
  - Knowledge
tags:
  - SMPP
  - Protocol
  - SMS
  - Network
author: seolmin
---



[저번 포스팅](/knowledge/SMPP_Protocol_1)에서 SMPP 프로토콜의 기본 개념을 살펴 보았습니다. 이번 포스팅에서는 SMPP 프로토콜을 이용하여 Bind 이후 SMS 수발신을 하는 과정에 대해 살펴보겠습니다.



## SMS MT, MO

SMPP 프로토콜을 통해 SMS 수발신을 하는 방식에 대해 설명하기에 앞서 SMS를 기술적으로 구분하여 이해할 필요가 있습니다. 수신과 발신이 서로 다른 PDU를 이용하기 때문입니다.

SMS는 MT(Mobile Terminated)와 MO(Mobile Originated)로 구분할 수 있습니다. 여기서 Mobile은 단말기를 뜻하는 것으로 MT는 단말기에서 종료되는 메시지로써 응용 프로그램 입장에서는 발송 메시지를 의미하고 MO의 경우 단말기에서 발생한 메시지로써 응용 프로그램 입장에서는 수신 메시지를 의미합니다.

DR(Delivery Receipt)는 말 그대로 배달 영수증을 뜻합니다. DR은 발송한 메시지가 단말까지 정상적으로 전송됬는지 등의 정보를 포함하여 메시지가 성공했는지 실패했는지 나타냅니다. 일반적인 경우 단말기에서 발생하여 이동 통신사와 Aggregator를 거쳐 수신할 수 있습니다.  Aggregator나 이동 통신사에 따라 MT를 발송한 후 하나의 MT에 대해 여러개의 DR을 받을 수도 있고 하나도 받지 못하는 경우도 있습니다.

MT, DR, MO 대해 간략히 표현하면 다음처럼 표현 할 수 있습니다.

![MO_MT](/images/2019-10-07-SMPP_Protocol_2/MO_MT.png)

MT는 응용 프로그램에서 생성되어 단말까지 전송되고 MO, DR은 단말에서 생성되어(DR은 Aggregator 또는 이동 통신사에서 생성될 수도 있습니다) Aggregator를 거쳐 응용 프로그램으로 전달됩니다.  

이 때, SMPPP 프로토콜에서 MT는 submit_sm이라는 PDU를 이용하고 MO와 DR은 deliver_sm이라는 PDU를 이용합니다. 즉, 응용 프로그램(ESME)이 수신하는 데이터는 deliver_sm, 발신하는 데이터는 submit_sm을 이용하는 것입니다.



## SUBMIT_SM

submit_sm은 ESME에서 SMSC로 메시지를 전달하기 위해 사용되는 PDU 입니다. 따라서 SMS 발송 요청이라고 볼 수 있겠습니다. Body에 다음과 같은 필드들이 포함합니다.

| FIELD                  | SIZE          | DESCRIPTION                                                  |
| ---------------------- | ------------- | ------------------------------------------------------------ |
| service_type           | 6 bytes max   | 메시지와 관련된 응용 프로그램 서비스를 나타내는데 사용될 수 있습니다. 일반적으로 특정 고급 메시징 서비스를 사용 시 지정되며 SMSC를 통한 메시징 서비스라면 `NULL`을 할당하면 됩니다. |
| source_addr_ton        | 1 byte        | 발신 주소에 대한 주소 형식을 지정합니다.                     |
| source_addr_npi        | 1 byte        | 발신 주소에 대한 번호 규칙을 지정합니다.                     |
| source_addr            | 21 bytes max  | 발신 주소를 지정합니다.                                      |
| dest_addr_ton          | 1 byte        | 수신 주소에 대한 주소 형식을 지정합니다.                     |
| dest_addr_npi          | 1 byte        | 수신 주소에 대한 번호 규칙을 지정합니다.                     |
| destination_addr       | 21 bytes max  | 수신 주소를 지정합니다.                                      |
| esm_class              | 1 byte        | 메시지 모드나 형식을 지정할 수 있습니다.                     |
| protocol_id            | 1 byte        | 프로토콜 식별자로 일반적으로 지정하지 않습니다.              |
| priority_flag          | 1 byte        | 메시지의 우선순위를 지정할 수 있습니다. SMSC가 지원하는 경우 사용 가능합니다. |
| schedule_delivery_time | 1 or 17 bytes | 메시지를 특정 시간에 발송하도록 예약하기 위해 사용됩니다. SMSC가 지원하는 경우 사용가능하며 즉시 발송하려면 `NULL`을 할당합니다. |
| validity_period        | 1 or 17 bytes | 메시지가 유효 시간을 설정합니다. SMSC가 지원하는 경우 일정 시간 내에 발송하지 못할 시 메시지를 폐기 시킬 수 있습니다. |
| registered_delivery    | 1 byte        | DR을 필요로 하는지 지정할 수 있습니다. 값에 따라 특정 경우의 DR만 수신하도록 할 수 있습니다. |
| data_coding            | 1 byte        | 메시지의 encoding을 표현합니다.                              |
| sm_default_msg_id      | 1 byte        | SMSC에 미리 정의된 메시지 목록에서 메시지를 선택해서 발송하도록 요청할 수 있습니다. SMSC에서 지원하는 경우 사용 가능합니다. |
| sm_length              | 1 byte        | short_message의 길이를 나타냅니다.                           |
| short_message          | 254 bytes     | 발송하고자 하는 메시지를 data_coding에 따라 encoding된 byte 배열로 지정합니다. |

이 필드들 이외에도 미리 지정된 TLV(Optional prameters)들이 정의되어 있습니다. TLV에 대한 추가적인 설명은 이 글에서는 제외하도록 하겠습니다.

각 필드들의 설명만으로는 정확히 어떤 목적으로 사용되는지 이해하기 힘들 수 있습니다. 좀 더 이해를 돕기 위해 필드들에 대해 좀 더 자세히 살펴보겠습니다.



### TON과 NPI

source_addr_ton과 dest_addr_ton, source_addr_npi과 dest_addr_npi 모두 수발신에 사용되는 주소의 형식과 규격을 정의하기 위해 사용됩니다. 전화번호 표기 형식을 지정하거나 특수한 주소(SMSC에 등록된)를 사용할 경우 그것을 표현할 수 있습니다. 이 값들은 submit_sm 뿐만 아니라 SMPP 프로토콜에서 주소 지정 시 동일한 규칙을 따릅니다.

**TON**의 경우 다음과 같은 값들이 올 수 있습니다.

* `0`: Unknown
* `1`: International
* `2`: National
* `3`: Network Specific
* `4`: Subscriber Number
* `5`: Alphanumeric
* `6`: Abbreviated

예를 들어 전화번호가 '1234567890'이고 값이 `1` 이라면 '+11234567890' 처럼 국제 전화번호 형식으로 표현하고 `2`라면 '1234567890'로 '+'와 국가 코드를 제외하고 표현할 수 있습니다. 이는 SMSC나 이동 통신사에 의해 정의된 표현만을 사용해야 합니다.

**NPI**의 경우 다음과 같은 값들이 올 수 있습니다.

- `0`: Unknown
- `1`: ISDN/telephone numbering plan (E163/E164) 
- `3`: Data numbering plan (X.121) 
- `4`: Telex numbering plan (F.69) 
- `6`: Land Mobile (E.212)
- `8`: National numbering plan
- `9`: Private numbering plan
- `10`: ERMES numbering plan (ETSI DE/PS 3 01-3) 
- `13`: Internet (IP) 
- `18`: WAP Client Id (to be defined by WAP Forum) 

모두 번호 규칙에 대한 규격들입니다. 예를 들어 E164 규격의 번호 규칙이라면 `1`로 설정하는데 '국가코드는 최대 3자, 전화번호는 최대 12자' 같은 규칙을 포함하게 됩니다. 이 또한 SMSC나 이동 통신사에 의해 정의된 표현만을 사용해야 합니다.

일반적으로 SMSC를 통해 SMS를 발송 할 때, Short code(5자의 짧은 코드), Long code(10자리 전화번호)를 사용하여 발송합니다. 대부분의 SMSC는 일반적으로 다음과 같은 기본 값을 할당하여 발송 요청하기를 요구하고 있습니다.

**Short code**가 '55555' 일 때, TON=`3`, NPI=`0`로 지정하고 source_addr은 '55555'로 지정합니다.

**Long code**가 '1234567890' 일 때, TON=`1`, NPI=`1`로 지정하고 source_addr은 '+11234567890'로 지정합니다.



### ESM_CLASS

일반적으로 submit_sm에서 사용되는 값들은 다음과 같습니다.

* `0x03`: Store and Forward
* `0x00`: Default
* `0x02`: Transaction
* `0x40`: UDHI Mask

예를 들어 `0x03`의 경우 메시지를 발송하면 SMSC에 안전하게 저장된 후 발송되도록 할 수 있고, `0x40`의 경우 메시지에 User Data Header 포함하여 특수한 기능을 이용할 수 있습니다. 당연히 이 기능들은 모두 SMSC가 지원하는 경우에만 사용 가능합니다.



### REGISTERED_DELIVERY

registered_delivery는 각 bit가 활성화 되었는지에 따라 특정 경우에 따라 메시지에 대한 DR을 수신할 것인지 지정할 수 있습니다. 

첫번째와 두번째 비트는 SMSC로부터의 메시지 상태를 DR로 수신할 것인지 표현 하는데 쓰입니다.

* `xxxxxx00`: DR을 요청하지 않습니다.
* `xxxxxx01`: SMSC에서 메시지 전송이 성공 또는 실패한 경우의 DR을 요청합니다.
* `xxxxxx10`: SMSC에서 메시지 전송이 실패한 경우의 DR을 요청합니다.

세번째와 네번째 비트는 단말로부터의 승인 상태를 DR로 수신할 것인지 표현 하는데 쓰입니다.

* `xxxx00xx`: 단말로부터 DR을 요청하지 않습니다.

* `xxxx01xx`: 단말로부터의 승인을 DR을 요청합니다.

* `xxxx10xx`: 사용자가 직접 단말에서 승인을 한 경우 DR을 요청합니다.

* `xxxx11xx`: 두 가지 모두 DR로 요청합니다.

다섯번째 비트는 중간 경로마다 알림을 DR로 수신할 것인지 표현하는데 쓰입니다.

* `xxx0xxxxx`: 중간 경로마다 알림을 DR로 요청하지 않습니다.
* `xxx1xxxxx`: 중간 경로마다 알림을 DR로 요청합니다.

이 값들은 요청한다고해서 SMSC가 수용하는 것은 아닙니다. 어떤 방식으로 DR을 넘겨줄 것인지는 SMSC의 구현에 따라 다릅니다.



### DATA_CODING

short_message의 encoding을 지정합니다. HTTP에서 Content-encoding Header와 동일한 역할을 한다고 볼 수 있습니다.

일반적으로 사용할 수 있는 값은 다음과 같습니다.

* `0x00`: Default - 일반적으로 GSM-7 encoding 인 경우가 많습니다
* `0x01`: IA5
* `0x03`: ISO-8859-1
* `0x08`: UCS-2

조금은 생소한 encoding들 일 수 있습니다. SMS의 경우 최대 140 bytes까지의 문자만 첨부할 수 있기 때문에 크기가 커질 수 있는 UTF-8 같은 encoding은 비효율적입니다. 때문에 최대한 많은 문자열을 포함하기 위해 GSM-7 encoding을 주로 사용하며 한 글자에 7bit가 할당됩니다. 이 경우 140(byte) * 8/7 = 160 character 까지 첨부 가능합니다.



## SUBMIT_SM_RESP

submit_sm에 대한 응답으로 submit_sm_resp를 받습니다. body에 메시지를 식별하기 위한 mesase_id를 반환합니다. 이 값을 통해 DR을 수신하였을 때 어떤 메시지에 대한 DR인지 식별 할 수 있습니다.

| FIELD      | SIZE         | DESCRIPTION                               |
| ---------- | ------------ | ----------------------------------------- |
| message_id | 65 bytes max | SMSC가 생성한 메시지에 대한 식별자입니다. |



## DELIVER_SM

deliver_sm은 SMSC로부터 ESME로 메시지를 전달하기 위해 사용되는 PDU입니다. MO와 DR같이 ESME가 수신을 받는 경우 사용할 수 있습니다. Body는 submit_sm과 동일하지만 몇가지는 사용되지 않습니다.

| FIELD                  | SIZE          | DESCRIPTION                                                  |
| ---------------------- | ------------- | ------------------------------------------------------------ |
| service_type           | 6 bytes max   | 메시지와 관련된 응용 프로그램 서비스를 나타내는데 사용될 수 있습니다. |
| source_addr_ton        | 1 byte        | 발신 주소에 대한 주소 형식을 지정합니다.                     |
| source_addr_npi        | 1 byte        | 발신 주소에 대한 번호 규칙을 지정합니다.                     |
| source_addr            | 21 bytes max  | 발신 주소를 지정합니다.                                      |
| dest_addr_ton          | 1 byte        | 수신 주소에 대한 주소 형식을 지정합니다.                     |
| dest_addr_npi          | 1 byte        | 수신 주소에 대한 번호 규칙을 지정합니다.                     |
| destination_addr       | 21 bytes max  | 수신 주소를 지정합니다.                                      |
| esm_class              | 1 byte        | 메시지 모드나 형식을 지정할 수 있습니다.                     |
| protocol_id            | 1 byte        | 프로토콜 식별자로 일반적으로 지정하지 않습니다.              |
| priority_flag          | 1 byte        | 메시지의 우선순위를 지정할 수 있습니다.                      |
| schedule_delivery_time | 1 or 17 bytes | 사용되지 않습니다.                                           |
| validity_period        | 1 or 17 bytes | 사용되지 않습니다.                                           |
| registered_delivery    | 1 byte        | 일반적으로 사용되지 않습니다.                                |
| data_coding            | 1 byte        | 메시지의 encoding을 표현합니다.                              |
| sm_default_msg_id      | 1 byte        | 사용되지 않습니다.                                           |
| sm_length              | 1 byte        | short_message의 길이를 나타냅니다.                           |
| short_message          | 254 bytes     | 발송하고자 하는 메시지를 data_coding에 따라 encoding된 byte 배열로 지정합니다. |



### MO

MO는 submit_sm과 거의 동일한 foramt으로 수신되기에 발신 할 때 사용한 submit_sm의 각 필드들에 대한 이해만 있다면 충분히 파싱 가능합니다. 하지만 MO와 DR을 어떻게 구분은 어떻게 해야 할까요? deliver_sm에서는 esm_class가 submit_sm과 다른 값 범위를 가질 수 있습니다.

* `0x00`: Default
* `0x04`: SMSC DR
* `0x08`: ESME DR
* `0x10`: MANUAL USER ACT
* `0x40`: UDHI Mask

`0x04`, `0x08`, `0x10` 이 세 값들은 deliver_sm이 DR임을 표현합니다. 따라서 특정 deliver_sm이 MO임을 확인하기 위해서는 세번째, 네번째, 다섯번째 비트가 set되어 있지 않음을 확인하면 됩니다. `0x1C`와 esm_class를 and 연산한 결과가 0보다 큰지 확인하면 되겠네요.



### DR

deliver_sm에 DR을 표현하기 위한 필드가 없기에 어떤 방식으로 DR이 구성되는지 의아하실 수도 있습니다. DR은  일반적으로 short_message에 특정한 format으로 표현되고 예외적으로 TLV를 이용하는 경우도 있습니다. TLV를 이용하는 경우 SMSC에서 정의한 방식으로 처리해야 합니다. 이 글에서는 일반적인 경우의 format만 확인해보도록 하겠습니다.

다음과 같은 필드를 포함하는 것이 일반적입니다.

| FIELD       | TYPE     | DESCRIPTION                                                  |
| ----------- | -------- | ------------------------------------------------------------ |
| id          | `string` | 메시지의 식별자입니다. submit_sm_resp의 message_id 입니다.   |
| sub         | `number` | sub id로 거의 사용되지 않습니다.                             |
| dlvrd       | `number` | 메시지의 완료 상태를 표현합니다. 1이면 완료되었음을 0이면 발송 중임을 나타냅니다. |
| submit date | `date`   | 발송 시간을 나타냅니다.                                      |
| done date   | `date`   | 메시지가 완료된 상태라면 그 시간을 나타냅니다.               |
| stat        | `string` | 메시지 상태를 나타냅니다. 거의 사용되지 않습니다.            |
| err         | `string` | SMSC에서 정의한 상태 코드를 포함하는 것이 일반적입니다. 이 값에 포함된 코드에 따라 메시지의 상태를 파악할 수 있습니다. |
| text        | `string` | 원본 메시지의 일부를 포함하거나 에러 메시지를 포함하는데 사용되는 것이 일반적입니다. |

예로 다음 처럼 short_message에 DR이 표현됩니다. 이와 같은 short_message를 파싱해서 DR의 정보를 추출합니다.

```
id=d1ad9f6e-7143-4912-b10a-e51e0ef7a8c0 sub=1 dlvrd=1 submitDate=2019-10-07T07:03:00.000-07:00 doneDate=2019-10-07T07:03:00.000-07:00 err=000 text=[Success]
```



## DELIVER_SM_RESP

deliver_sm에 대한 응답으로 deliver_sm_resp를 사용합니다. 만약 deliver_sm_resp 헤더의 상태 정보가 성공이 아니라면 SMSC정의에 따라 retry 처리가 가능한 경우도 있습니다. 

| FIELD      | SIZE         | DESCRIPTION                                                  |
| ---------- | ------------ | ------------------------------------------------------------ |
| message_id | 65 bytes max | deliver_sm에 대한 식별자를 반환합니다. ESME가 특별한 기능을 구현하지 않는 이상 NULL이 포함됩니다. |



## Conclusion

SMS 수발신 과정과 그에 참여하는 주요 PDU에 대해 살펴보았습니다. 

이 글에 작성된 내용들은 일반적인 범위 내에서 작성된 것이며 SMSC에 따라 값의 사용이나 처리 방식이 다른 경우도 있습니다. 때문에 만약 ESME를 구성하신다면 꼭 SMSC에서 제공하는 구현 가이드를 충족시켜야 합니다. 하지만 위 내용들이 표현하고자 하는 의미를 정확히 이해 하셨다면 처리 방식이 다른 SMSC와도 충분히 메시지를 주고 받을 수 있는 응용 프로그램을 작성할 수 있을 것입니다.

다음 포스팅에서는 User Data Header를 이용하여 SMS의 추가 기능을 사용하는 방법에 대해 알아보겠습니다.

Trumpia 기술 블로그를 방문해 주셔서 감사합니다. 아자아자 화이팅!
