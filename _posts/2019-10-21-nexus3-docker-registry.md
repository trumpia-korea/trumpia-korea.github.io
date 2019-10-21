---
title: "Sonatype Nexus3 Docker Registry"
categories:
  - Manual
tags:
  - Infra
  - Sonatype Nexus
  - Docker
  - Docker Registry
author: kyedon
---

Gitlab CI를 통한 지속적인 배포를 위해 도커 레지스트리가 필요하여 [Docker Registry Official Image](https://hub.docker.com/_/registry)를 사용해보았으나 관리가 쉽지 않다.
기본적으로 웹어드민 페이지 같은 것을 제공하는 것도 아니고 접근 권한도 체계적으로 관리하기 어렵다.

인터넷에서 좀 더 찾아보니 [Sonatype Nexus](https://www.sonatype.com/nexus-repository-oss)가 도커 레지스트리를 지원한다.
별도로 실행중인 도커 레지스트리를 프록시로 연결하여 캐시처럼 사용할수도 있고 호스팅할 수도 있다.
복잡하게 프록시를 사용하지 않고 넥서스 자체에서 호스팅하여 넥서스3를 구성했다.

## Sonatype Nexus Repository Manager

넥서스는 여러가지 포맷을 지원하는 아티팩트 저장소이다.
Docker, YUM, APT, Maven, Composer 등 다양한 포맷에 대한 저장소를 지원한다.
Nexus3에서 Nexus2에 비해 더 다양한 저장소 포맷들이 추가되었으며 Raw 포맷에 대한 관리도 더 편해졌다.


## Installation

***

###### Environments
 - CentOS 7.7
 - Java 1.8
 - Nexus 3.19.1-01

***

OpenJDK 8을 설치하고 넥서스를 다운로드 받는다.

```shell
sudo yum install -y java-1.8.0-openjdk.x86_64 wget

sudo mkdir /usr/local/nexus
cd /usr/local/nexus
sudo wget -O nexus.tar.gz https://download.sonatype.com/nexus/3/latest-unix.tar.gz
sudo tar -xzvf nexus.tar.gz
sudo rm nexus.tar.gz
```

넥서스를 실행할 계정을 하나 추가해준다.

```shell
sudo useradd nexus
```

압축파일으르 풀면 별도의 루트 디렉토리 없이 nexus(install_dir) 디렉토리와 sonatype-work(data_dir) 디렉토리가 생긴다.
편하게 사용하기 위해 심볼릭 링크를 하나 추가해주고 임시로 환경변수를 추가한다.

```console
[kyedon@nexus-server nexus]# cd /usr/local/nexus
[kyedon@nexus-server nexus]# sudo ln -s nexus-3.19.1-01 nexus
[kyedon@nexus-server nexus]# sudo chown -R nexus. /usr/local/nexus
[kyedon@nexus-server nexus]# ls -l /usr/local/nexus
total 0
lrwxrwxrwx. 1 nexus nexus  16 Oct 15 00:04 nexus -> nexus-3.19.1-01/
drwxr-xr-x. 9 nexus nexus 163 Oct 15 00:04 nexus-3.19.1-01
drwxr-xr-x. 3 nexus nexus  20 Oct 15 00:04 sonatype-work
[kyedon@nexus-server nexus]$ export install_dir=/usr/local/nexus/nexus
[kyedon@nexus-server nexus]$ export data_dir=/usr/local/nexus/sonatype-work/nexus3/
```

nexus.rc 파일을 변경하여 nexus 계정으로 실행하도록 설정을 수정한다.

```shell
sudo sed -i 's/#run_as_user=""/run_as_user="nexus"/g' $install_dir/bin/nexus.rc
```

systemd 서비스를 생성한다.

```
sudo bash -c "cat >/etc/systemd/system/nexus.service" <<'EOF'
[Unit]
Description=nexus service
After=network.target

[Service]
Type=forking
LimitNOFILE=65536
User=nexus
Group=nexus
ExecStart=/usr/local/nexus/nexus/bin/nexus start
ExecStop=/usr/local/nexus/nexus/bin/nexus stop
User=nexus
Restart=on-abort

[Install]
WantedBy=multi-user.target
EOF
```

이제 systemd를 사용해 서비스를 켜고 끌 수 있다.
넥서스 서비스는 기본적으로 0.0.0.0:8081(http) 포트를 사용한다.

최초 로그인시 계정은 `admin`이고 패스워드는 `$data_dir/admin.password`에서 확인하면 된다.

## SSL Configuration

SSL 설정을 따로 하지 않아도 바로 사용 가능하며 docker registry 또한 동작하긴 하지만
SSL 설정을 하지 않는 것은 보안상 좋지 않으며 별도로 insecure-registry 옵션을 주어야 한다.
레지스트리를 안전하게 사용하기 위해 SSL을 설정하는 것이 좋다.

우선 기존 인증서를 사용해 Java Keystroke File을 생성한다.  
패스워드의 경우 이후 설정파일에 평문으로 입력해야하기 때문에 서버에 접속 가능한 다른 유저들에게 보여도 상관 없도록 설정한다.

```shell
openssl pkcs12 -inkey private.key -in cert.crt -certfile chain.crt -export -out cert.pfx -name "cert"
keytool -importkeystore -srckeystore cert.pfx -srcstoretype pkcs12 -destkeystore keystore.jks -deststoretype pkcs12  -alias "cert"
```

```console
[kyedon@nexus-server ~]$ ll
total 12
-rw-r--r--. 1 kyedon kyedon 1856 Oct 20 23:15 cert.crt
-rw-r--r--. 1 kyedon kyedon 1674 Oct 20 23:15 chain.crt
-rw-r--r--. 1 kyedon kyedon 1675 Oct 20 23:15 private.key
[kyedon@nexus-server ~]$ openssl pkcs12 -inkey private.key -in cert.crt -certfile chain.crt -export -out cert.pfx -name "cert"
Enter Export Password: test1234
Verifying - Enter Export Password: test1234
[kyedon@nexus-server ~]$ keytool -importkeystore -srckeystore cert.pfx -srcstoretype pkcs12 -destkeystore keystore.jks -deststoretype pkcs12  -alias "cert"
Importing keystore cert.pfx to keystore.jks...
Enter destination keystore password: test1234
Re-enter new password: test1234
Enter source keystore password: test1234
[kyedon@nexus-server ~]$ ll
total 28
-rw-r--r--. 1 kyedon kyedon 1856 Oct 20 23:15 cert.crt
-rw-rw-r--. 1 kyedon kyedon 4339 Oct 20 23:25 keystore.jks
-rw-rw-r--. 1 kyedon kyedon 4182 Oct 20 23:24 cert.pfx
-rw-r--r--. 1 kyedon kyedon 1674 Oct 20 23:15 chain.crt
-rw-r--r--. 1 kyedon kyedon 1675 Oct 20 23:15 private.key
```

생성된 키파일을 `$data_dir/etc/ssl/keystore.jks` 경로에 넣어준다.

```shell
sudo mkdir $data_dir/etc/ssl/
sudo cp keystore.jks $data_dir/etc/ssl/
sudo chown -R nexus. $data_dir/etc/ssl/
```

`$data_dir/etc/nexus.properties` 파일을 수정한다.

```
application-host=0.0.0.0
application-port=8080
application-port-ssl=8443
nexus-args=${jetty.etc}/jetty.xml,${jetty.etc}/jetty-http.xml,${jetty.etc}/jetty-requestlog.xml,${jetty.etc}/jetty-https.xml
ssl.etc=${karaf.data}/etc/ssl
```

`nexus-args` 옵션에 `${jetty.etc}/jetty-https.xml` 값을 추가한다.
편의상 `application-port`를 변경했으며, `application-port-ssl` 옵션을 추가했다.


jetty-https 설정 파일에서 패스워드를 수정해준다.

```
sudo sed -i 's/password/test1234/g' $install_dir/etc/jetty/jetty-https.xml
 ```
  
이제 8443 포트를 통해 https로 접근할 수 있다.

## Docker Registry(hosted)

이제 도커 레지스트리를 생성한다.

![create repository](/images/2019-10-21-nexus3-docker-registry/create-repository-1.png)  

![create repository](/images/2019-10-21-nexus3-docker-registry/create-repository-2.png)  

적당한 저장소 이름을 설정하고 HTTPS, 익명접근을 활성화환다.

![create repository](/images/2019-10-21-nexus3-docker-registry/create-repository-3.png)  

저장소가 생성되었으면 해당 저장소에 대한 롤을 추가해준다.

![create role](/images/2019-10-21-nexus3-docker-registry/create-role-1.png)  

![create role](/images/2019-10-21-nexus3-docker-registry/create-role-2.png)  

유저를 생성하여 이전에 생성한 도커 저장소 룰을 적용한다.

![create user](/images/2019-10-21-nexus3-docker-registry/create-user.png)  

인증을 할 수 있도록 Docker Bearer Token Realm을 활성화시킨다.

![activate security realm](/images/2019-10-21-nexus3-docker-registry/activate-security-realm.png)  

이전에 생성한 계정으로 로그인 시도 및 push 결과 정상적으로 동작하는 것을 알 수 있다.
익명 pull을 허용했으므로 pull 할때에는 로그인이 필요하지 않다.

```console
[kyedon@desktop ~]$ docker login kd-test-nexus-server.trumpia.com:5000
Username: kyedon
Password:
Login Succeeded
[kyedon@desktop ~]$ docker pull hello-world:latest
latest: Pulling from library/hello-world
1b930d010525: Pull complete
Digest: sha256:c3b4ada4687bbaa170745b3e4dd8ac3f194ca95b2d0518b417fb47e5879d9b5f
Status: Downloaded newer image for hello-world:latest
docker.io/library/hello-world:latest
[kyedon@desktop ~]$ docker tag hello-world:latest kd-test-nexus-server.trumpia.com:5000/hello-world:latest
[kyedon@desktop ~]$ docker tag hello-world:latest kd-test-nexus-server.trumpia.com:5000/hello-world:v0.1
[kyedon@desktop ~]$ docker push kd-test-nexus-server.trumpia.com:5000/hello-world:latest
The push refers to repository [kd-test-nexus-server.trumpia.com:5000/hello-world]
af0b15c8625b: Pushed
latest: digest: sha256:92c7f9c92844bbbb5d0a101b22f7c2a7949e40f8ea90c8b3bc396879d95e899a size: 524
[kyedon@desktop ~]$ docker push kd-test-nexus-server.trumpia.com:5000/hello-world:v0.1
The push refers to repository [kd-test-nexus-server.trumpia.com:5000/hello-world]
af0b15c8625b: Layer already exists
v0.1: digest: sha256:92c7f9c92844bbbb5d0a101b22f7c2a7949e40f8ea90c8b3bc396879d95e899a size: 524
```

![check image](/images/2019-10-21-nexus3-docker-registry/check-image.png)  

## Conclusion

이전부터 Ansible에서 사용할 중앙 저장소가 필요했는데 Nexus가 그 역할로 제격이다. Raw Repository와 YUM Repository를 통해 일부 바이너리나 라이브러리,
RPM 등을 한곳에서 관리할 수 있기 때문에 상당히 편하다.
LDAP을 지원하므로 LDAP을 통한 계정 관리 및 그룹에 따른 권한 관리도 용이하다.

처음 셋업했을 때는 어렵게 생각했으나 생각보다 쉽게 설정하고 관리할 수 있다.
프라이빗 도커 레지스트리가 생겼으니 이제 GitLab CI를 통한 컨테이너 배포를 본격적으로 시도해볼 수 있을 것으로 보인다.
