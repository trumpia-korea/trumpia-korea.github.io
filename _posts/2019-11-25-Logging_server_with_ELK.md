---
title: "Multitenancy in hibernate"
categories:
  - knowledge
tags:
  - hibernate
  - multitenancy
  - java
author: seolmin
---



# Logging server with ELK

서비스의 가동률을 높이기 위해 서버를 다중화하는 경우 로그를 각 서버의 파일에 기록하면 디버깅이나 로그 관리에 피곤함을 느끼기 마련입니다. 디버깅을 위해 각 서버에 접근하여야 하거나 로그 관리 정책 변경 시 각각의 서버에 모두 적용을 해야 하기 때문입니다. 



성장을 거듭하며 고객의 서비스 사용율이 높아졌고 이에따라 스케일 아웃을 통해 트래픽 처리량을 늘렸습니다. 점점 관리하여야 하는 서버는 늘어났고 기존의 파일에 로그를 쓰는 방식에 불편함을 느끼게 되어 통합 로그 서버를 필요로 하게 되었습니다.



오늘은 ELK Stack을 이용해 통합 로그 관리 서버를 구축하는 방법에 대해 기본적인 방법에 대해 알아보고자 합니다. 구성은 다음과 같습니다.

* **Client**
  * **Java 1.8**
  * **Spring boot framework 2.0.5**
* **Server**
  * **Elastic search 7.4.2** : 로그  적재 및 분석
  * **Logstash 7.4.2**: 로그 수집
  * **Kibana 7.4.2**:  Visualization 
  * **Erlang 21.3.8.11**
  * **RabbitMQ 3.8.1**: 로그 전송을 위한 Queue

![structure](images/2019-11-25-Logging_server_with_ELK/structure.png)

각 Client는 Rabbit MQ로 로그를 발행하고, Logstash는 Rabbit MQ로부터 로그를 소비하여 Elasticsearch에 적재합니다. 이렇게 적재된 로그를 Kibana는 읽어 UI에 뿌려줍니다.



## Rabbit MQ config

로그를 클라이언트와 Logstash 사이에 주고 받기 위한 Rabbit MQ부터 설정해 보겠습니다.



**1. 설치**

Rabbit MQ 공식 사이트에서 운영체제에 맞는 패키지를 다운받아 설치하시면 됩니다. Rabbit MQ는 Erlang을 필요로 합니다.



**2. 방화벽 포트열기**

Rabbit MQ는 기본적으로 `5672`, `15672`, `35197`, `4369` 포트를 사용하는데 이 중 `5672`는 필수로 사용되며 `15672`는 Management plugin에서, `35197`, `4369` 는 Clustering에 사용됩니다. 여기서는 Clustering은 다루지 않을 것이므로 `5672`와 `15672` 만 개방하도록 하겠습니다.

```shell
-A INPUT -p tcp -m state --state NEW -m tcp --dport 5672 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 15672 -j ACCEPT
```



**3. Enable Management plugin**

좀 더 편하게 설정하기 위해 Management plugin을 활성화 합니다. 다음 명령어를 사용하면 plugin을 활성화 할 수 있습니다

```
rabbitmq-plugins enable rabbitmq_management
```



**4. User 생성**

Client와 Logstash에서 접근하기 위한 User를 생성하고 권한을 부여해 줍니다.

```
rabbitmqctl add_user admin trumpia123
rabbitmqctl set_user_tags admin  administrator
rabbitmqctl set_permissions -p / admin ".*" ".*" ".*"
```

저는 admin 권한을 포함하여 모든 권한을 부여했습니다.



**5. virtual host 생성**

logs라는 이름의 로그를 위한 virtual host를 생성합니다. 서버의 15672 포트로 접근하여 web ui를 통해 설정이 가능합니다.

![add_virtual_host](images/2019-11-25-Logging_server_with_ELK/add_virtual_host.png)



**6. exchange 생성**

exchange는 logs라는 이름의 topic type으로 생성하도록 하겠습니다. 각 프로세스 혹은 모듈별로 큐를 구분하고, 각 큐에 맞는 Routing key로 발행할 것입니다.

![add_exchange](images/2019-11-25-Logging_server_with_ELK/add_exchange.png)



**7. queue 생성**

test라는 이름의 queue를 생성합니다.

![add_queue](images/2019-11-25-Logging_server_with_ELK/add_queue.png)



**8. Binding**

앞서 생성한 exchange와 queue를 binding 해줍니다. 여기에 사용된 Routing key를 후에 Client에서 로그를 발행할 때 사용할 것입니다.

![binding](images/2019-11-25-Logging_server_with_ELK/binding.png)



## Client config

파일 로그도 유지하되 로그를 Elasticsearch에도 적재하는 경우에는 파일 로그를 기록하고 filebeat이나 Logstash를 이용해 로그를 읽어 Queue로 발행하고 ELK 서버의 Logstash가 읽어가기도 합니다. 하지만 이번에는 Client 프로세스에서 바로 RabbitMQ로 발행하도록 하여 ELK 서버에만 적재하도록 할 것입니다.



`spring-boot-starter-amqp`에는 로그를 Rabbit MQ에 발행할 수 있는 appender 구현체가 존재합니다. 이를 이용해 쉽게 설정을 할 수 있습니다.



**pom.xml**





## ELK  config



**1. 설치**

로그를 수집하고 적재 및 분석을 위한 ELK Stack을 설치합니다. 마찬가지로 공식 사이트에서 운영체제에 맞는 패키지를 다운받아 설치하면 됩니다.



**2. Elasticsearch 설정**

로그 서버를 위한 Elasticsearch 설정에는 index 관리를 위한 life cycle 설정이 필요합니다만 오늘은 깊에 진행하지 않고 로그를 Elasticsearch에 적재하고 Kibana로 Visualization하는 동작만을 확인하겠습니다. 따라서 포트만 개방해주면 되는데 `9200`, `9300` 포트를 개방하면 됩니다. 만약 로그의 적재는 Logstash를 이용해서만 하고, Kibana를 통해 확인한다고 했을 때 ELK 모두 같은 서버에 있다면 굳이 포트를 개방할 필요는 없습니다.

```shell
-A INPUT -p tcp -m state --state NEW -m tcp --dport 9200 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 9300 -j ACCEPT
```



**3. Logstash 설정**

Rabbit MQ를 통해 Logstash를 가져와 Elasticsearch에 적재하기 위해 conf 파일을 생성하여야 합니다. 일반적으로 `/etc/logstash/conf.d` 에 생성한 `*.conf` 파일을 읽어 Logstash가 실행되는데 테스트를 위한 해당 경로에 `test.conf` 라는 이름의 설정을 추가하겠습니다.

```
input {
    rabbitmq {
      host => "192.168.3.142"
      queue => "test"
      user => "admin"
      password => "trumpia123"
      threads => 50
      vhost => "logs"
      passive => true
   }
}
output {
    elasticsearch {
        hosts => "localhost:9200"
    }
}
```

`input` 항목에는 Rabbit MQ에서 로그를 가져오기 위한 설정을 추가합니다. 여기서 `passive`라는 설정이 활성화 한다면 Logstash가 시작하며 queue를 생성하지 않습니다. 우리는 이전에 manually하게 queue를 생성했으므로 `true`로 설정합니다. 만약 그렇지 않다면 미리 생성한 queue와 Logstash가 생성하려 하는 queue의 설정이 충돌하여 실행되지 않을 수 있습니다. 



**Kibana 설정**

