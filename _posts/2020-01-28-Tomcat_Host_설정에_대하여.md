---
title: "Tomcat의 Host 설정에 관하여"
categories:
  - Fault
tags:
  - CentOS
  - Java
  - Tomcat
author: inpyo

---


담당하고 있는 Shortener 서비스의 CPU 사용률이 다소 높다는 이야기를 Infra 팀으로부터 전달받고, 사실을 확인하게 되었습니다.

먼저 top 명령어로 CPU 및 메모리 사용률을 확인하였습니다.

~~~
[root@server]# top
~~~



![top](/images/2020-01-28-Tomcat_Host/tomcat_host_01.png)



확인해보니, 실제로 동작하는 서비스의 CPU와 메모리 사용률이 높게 보이고 있었습니다. 그래서 아래의 명령어를 통해, 프로세스 내 어떤 Thread가 자원을 소모하고 있는지 확인해봤습니다.

~~~
[root@server] ps -mo %cpu,lwp {PID}
~~~



![ps](/images/2020-01-28-Tomcat_Host/tomcat_host_02.png)



대부분의 Thread는 0~0.5% 내외의 CPU 사용률을 보이고 있지만, 유독 몇몇 쓰레드들은 CPU 자원을 계속 소모하고 있었습니다. 트래픽 유입이 없는 시간대임에도 말이죠.
이에 부하의 원인일 수 있는 Thread들에 대해 조금 더 알아보기로 하였습니다.


~~~
[root@server] jstack {PID} > {파일명}
~~~



![jstack](/images/2020-01-28-Tomcat_Host/tomcat_host_03.png)



위 명령어를 통해 Thread Dump를 확인한 모습이며, GC task thread가 CPU 자원을 계속 소모하고 있는 사실을 알 수 있습니다.
여기서 어떻게 그 내용을 알 수 있느냐라는 물음이 있을 수 있는데, 이는 방금 전 ps 명령어로 조사했던 LWP(Light Weight Process)의 값을 통해 확인할 수 있습니다. LWP의 값은 10진수이며, 이를 헥사(16진수)로 변환 후 Thread Dump에서 nid의 값을 확인하면 됩니다.

이해를 돕기위해, 예를 들어 설명하자면
CPU 사용률이 높게 나왔던 Thread 중, "13361"은 10진수 값이고, 이를 헥사(16진수)로 변환하면 "0x3431"입니다.
nid가 "0x3431"인 Thread는, 스크린 샷에서 보이는 바와 같이 GC task입니다.

Lock이 걸려있는 것도 아니고, Runnable 상태인 것으로 미루어 보았을 때 계속 GC가 이루어지고 있다는 사실을 짐작할 수 있었습니다.
그래서 아래의 명령어를 통해 GC 현황을 살펴보기로 했습니다.   


~~~
jstat -gc {PID}
~~~



![jstat](/images/2020-01-28-Tomcat_Host/tomcat_host_04.png)



당시, 서비스를 내렸다가 올린 지 하루 정도 뿐이 되지 않았는데도 저렇게 빈번하게 Full-GC가 수행되는 점은 분명 문제가 있는 것이라 판단이 됩니다.
더구나 Old 영역 사용률이 100%로 고정되어있는 점도 의심스러운 부분입니다.
조금 더 상세히 알아보기 위해 아래 명령어를 이용하여 Heap 힙스토그램을 추출하고, MAT(Memory Analyzer)를 이용하여 분석하였습니다.  

~~~
jmap -dump:format=b,file=heapdump.hprof {PID}
~~~

MAT로 열어보니, 가장 눈에 띄면서 이상한 점이 발견되었습니다.



![mat](/images/2020-01-28-Tomcat_Host/tomcat_host_05.jpg)



같은 클래스가 5개씩 올라와 있는 모습이 쉽게 관찰됩니다.  
Hibernate SessionFactory, EntityLoader 등 유일하게 떠 있어야 하는 내용들이 중복으로 올라와있습니다.


동일한 클래스가 중복으로 떠있다는 것은 애플리케이션이 여러 개 떠있다는 내용일 수 있지만, 프로세스는 tomcat 하나가 떠있습니다.
그러하다면 Tomcat이 여러 애플리케이션을 띄우고 있을 수 있을까? 라는 물음이 들었고, 이에 대해서 검색을 통해 확인할 수 있었습니다.


[https://tomcat.apache.org/tomcat-8.0-doc/class-loader-howto.html](https://tomcat.apache.org/tomcat-8.0-doc/class-loader-howto.html)  
Tomcat 공식 Reference를 보면,

>"A class loader is created for each web application that is deployed in a single Tomcat instance."  
"클래스 로더는 톰캣 인스턴스에 디플로이된 웹 애플리케이션 각각에 대해 생성된다."

라는 문구가 있습니다. 이를 통해, 현재 톰캣 안에서 여러 애플리케이션이 떠 있을 수 있다라는 것을 의심해보게 되었습니다.
여러 개로 나뉘어서 분기처리 되고 있는 것이 무엇일까 찾아보던 중, Host에 대한 부분일 수 있겠다는 생각이 스쳤습니다.

현재 트럼피아 Shortener 시스템의 Host 설정은,

~~~
<Host name="A Host">
    <Alias>A's Sub Host</Alias>
    <Alias>A's Sub Host</Alias>
    <Alias>A's Sub Host</Alias>
    <Alias>A's Sub Host</Alias>
    <Alias>A's Sub Host</Alias>
</Host>

<Host name="B Host">
    <Alias>B's Sub Host</Alias>
    <Alias>B's Sub Host</Alias>
    <Alias>B's Sub Host</Alias>
    <Alias>B's Sub Host</Alias>
    <Alias>B's Sub Host</Alias>
</Host>

<Host name="C Host">
    <Alias>C's Sub Host</Alias>
    <Alias>C's Sub Host</Alias>
    <Alias>C's Sub Host</Alias>
    <Alias>C's Sub Host</Alias>
    <Alias>C's Sub Host</Alias>
</Host>

<Host name="D Host">
    <Alias>D's Sub Host</Alias>
    <Alias>D's Sub Host</Alias>
    <Alias>D's Sub Host</Alias>
    <Alias>D's Sub Host</Alias>
    <Alias>D's Sub Host</Alias>
</Host>

<Host name="E Host">
    <Alias>E's Sub Host</Alias>
    <Alias>E's Sub Host</Alias>
    <Alias>E's Sub Host</Alias>
    <Alias>E's Sub Host</Alias>
    <Alias>E's Sub Host</Alias>
</Host>
~~~

이와 같은 형태로 서비스 도메인 당 호스트가 하나씩 설정이 되어있었습니다.  
이를 아래와 같은 형태로 Host를 변경해 보기로 했습니다.

~~~
<Host name="A Host">

    <Alias>A's Sub Host</Alias>
    <Alias>A's Sub Host</Alias>
    <Alias>A's Sub Host</Alias>
    <Alias>A's Sub Host</Alias>
    <Alias>A's Sub Host</Alias>

    <Alias>B Host</Alias>
    <Alias>B's Sub Host</Alias>
    <Alias>B's Sub Host</Alias>
    <Alias>B's Sub Host</Alias>
    <Alias>B's Sub Host</Alias>
    <Alias>B's Sub Host</Alias>

    <Alias>C Host</Alias>
    <Alias>C's Sub Host</Alias>
    <Alias>C's Sub Host</Alias>
    <Alias>C's Sub Host</Alias>
    <Alias>C's Sub Host</Alias>
    <Alias>C's Sub Host</Alias>

    <Alias>D Host</Alias>
    <Alias>D's Sub Host</Alias>
    <Alias>D's Sub Host</Alias>
    <Alias>D's Sub Host</Alias>
    <Alias>D's Sub Host</Alias>
    <Alias>D's Sub Host</Alias>

    <Alias>E Host</Alias>
    <Alias>E's Sub Host</Alias>
    <Alias>E's Sub Host</Alias>
    <Alias>E's Sub Host</Alias>
    <Alias>E's Sub Host</Alias>
    <Alias>E's Sub Host</Alias>

 </Host>
~~~

이 결과, 아래 화면처럼 heap dump 상에서의 클래스의 중복 로드 현상이 사라진 것을 확인할 수 있었습니다.



![heapdump](/images/2020-01-28-Tomcat_Host/tomcat_host_06.jpg)



[https://www.jvmhost.com/tomcat-hosting](https://www.jvmhost.com/tomcat-hosting)  
위 포스팅 글을 살펴보면,

>"You can have multiple hosts and each can be mapped to a separate web application using Context.  
"복수의 Host를 가질 수 있으며 이는 컨텍스트를 사용하는 각각의 분리된 웹 애플리케이션과 매핑될 수 있다."

이렇듯 Tomcat에서는 Host별로 애플리케이션을 지정하여 각각 따로 띄울 수 있습니다.  
현재 Shortener 서비스는 동일한 애플리케이션을 Host만 달리해서 중복으로 띄우고 있었고, 이 때문에 시스템 자원을 불필요하게 많이 소모하고 있던 것입니다.

아래는 이를 실제 상용환경에서 적용한 결과입니다.



![applied](/images/2020-01-28-Tomcat_Host/tomcat_host_07.png)



CPU와 메모리 사용량이 현저히 줄어든 것이 확인됩니다.  
성능 또한 적잖은 차이를 보였는데, 아래 스크린샷은 JMeter를 활용한 수정 전과 후의 성능 비교입니다.
1,000건 테스트의 결과인데, Throughput의 확연한 차이를 볼 수 있었습니다.

  
![performance1](/images/2020-01-28-Tomcat_Host/tomcat_host_08.png)  
<center>[수정 전]</center>

-----

![performance2](/images/2020-01-28-Tomcat_Host/tomcat_host_09.png)  
<center>[수정 후]</center>